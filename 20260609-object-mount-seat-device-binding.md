tags: ["satellite", "object-mount", "licensing", "console", "payments"]
---

# Object Mount Seat Device Binding

## Essentials

Object Mount (OM) seats are currently user-level licenses. This document proposes a first
implementation for binding those seats to desktop app installations, so that one OM seat authorizes
one active OM app device.

The design keeps billing unchanged. Customers still buy a quantity of OM seats per tier, free seats
remain part of the same user-level seat pool, and the Stripe subscription quantity remains the
source of truth for paid seat count. Device binding is an authorization layer on top of the existing
seat pool.

The proposed managed-device release uses one durable server-side table for device activations,
treated as the live source of truth: a row exists if and only if a device is active and holding a
seat. Short-lived license leases are stateless signed tokens that are never persisted server-side.
A separate lease/events history table can be added later if product or audit requirements justify
it.

### Header

Date: 2026-06-09

Owner: VitaliiShpital

Accountable:
- Console team

Consulted:
- ---

Informed:
- ---

### Context

OM seats are sold per user but, today, any use of Object Mount on the account simply consumes from a
seat pool — there is nothing that ties a seat to a specific machine. To make seat counts meaningful
(one seat → one authorized device) we need a way to bind seats to desktop installations.

This is constrained by how the OM desktop app works:

- The app has **no interactive authentication flow** — it should not ask the user to sign in and
  must not be trusted to submit a `user_id` directly.
- The app operates with **S3 credentials created in a project**; that project has an owner.
- For the **Any Cloud** product the app mounts a third-party backend (GCP, AWS, MinIO, …) and Storj
  is nowhere on the data path — there the Storj S3 credential is purely a licensing anchor.
- The app must behave reasonably **offline** for bounded windows.

### Goals

- Limit OM usage so one active seat maps to one authorized desktop app installation.
- Allow users to deactivate old devices and activate new ones.
- Avoid relying on raw hardware IDs as the primary license identity.
- Work with the current OM model where the desktop app has no interactive authentication.
- Support reasonable offline behavior for a desktop mount app.
- Include a concrete managed-device experience that can ship with the first release.
- Preserve the existing billing model.

### Approach / Design

#### Core idea

Each OM desktop app installation creates a local cryptographic identity: a private key stored on the
device and a public key sent to Satellite. Satellite stores the public key and related device
metadata as a **device activation**. Before the app mounts storage, it asks Satellite for a
short-lived OM license lease. Satellite grants the lease only if the activation is valid and the
entitlement system says the project owner for the OM S3 credentials still has enough seats for their
active devices.

In short:

1. User launches the OM desktop app.
2. App generates a device key pair.
3. App sends the public key and device metadata along with its existing S3 credential context.
4. Satellite derives the project owner from those S3 credentials.
5. Satellite creates an activation if that project owner has an available OM seat.
6. App periodically requests a signed OM lease.
7. Satellite renews the lease while the activation remains valid.
8. Mounting is allowed only while the app has a valid lease.

The primary identity is a locally generated private key; the app proves possession of it when
activating or renewing. Hardware-derived signals are hashed and stored only as secondary metadata
for abuse detection, support, and device-change heuristics.

#### Data model

One table is the live source of truth for device activations. A row exists if and only if the device
is active and holding a seat.

```text
om_device_activations
- id
- user_id
- product_id
- public_key
- device_fingerprint_hash
- device_name
- platform
- app_version
- last_seen_at
- created_at
- updated_at
```

Suggested constraints:

```text
primary key (id)
unique (user_id, product_id, public_key)
```

The unique constraint keeps a device key from being activated twice for the same user and product,
and as a btree it also serves the seat-count query (a prefix lookup on `user_id, product_id`). The
hot renewal path looks up by `id` and is covered by the primary key. The stale-cleanup job scans by
`last_seen_at`; it runs rarely against a small table, so a dedicated index can be added later if the
table grows enough to need one.

Every row is active by definition, so there is no stored `status` column. Revocation deletes the
row, which frees the seat. Reactivation inserts a fresh row. `stale` is a derived condition — any
row whose `last_seen_at` is older than the staleness threshold — not a stored value.

Field semantics:

- `created_at`: when the activation row was created (i.e. when the device started consuming a seat).
- `last_seen_at`: updated on every successful lease renewal; drives staleness and the UI "last seen"
  column.

#### Seat counting

For each user and OM tier:

```text
available seats = active seat entitlement - count(activation rows for user_id, product_id)
```

Every row in the table is an active device activation, so the count is simply the number of rows.
This intentionally counts activated devices, not currently running app processes. A seat remains
assigned to a device until the user deactivates that device (deleting the row), or until an
automatic cleanup policy deletes a long-stale row.

This model is easier to reason about than concurrent session licensing:

- A laptop consumes a seat even when the app is not currently open.
- Starting and stopping the app does not free and reclaim seats repeatedly.
- Users can see which devices are assigned.
- License enforcement does not depend on perfect real-time presence.

#### Entitlement integration

The OM app does not authenticate interactively and is not trusted to submit `user_id` or
`project_id`. Instead, `user_id` is resolved from the project used by the OM app's S3 credentials,
whose owner is the licensed user whose seat entitlement is checked.

Seat entitlement is an **account-level** property of that user, not of any particular credential or
project. The S3 credential is used only **once, at activation, to bootstrap the user's identity**;
the resolved `user_id` is then stored on the activation row. Because identity is captured at
activation, renewal does not re-resolve it from the credential. Deleting the bootstrap credential,
or even the project it lived in, therefore does not break or orphan an activation — the device keeps
its seat because the user still owns the seat, and the seat is freed only by revoking the device.

The rule:

```text
desktop app supplies device identity
server resolves the user once, at activation, from the S3 credentials
that user supplies the account-level seat entitlement
```

If existing background scripts already read the entitlements table to decide whether OM is enabled,
device binding should be added at that decision point. The entitlement check becomes:

```text
project owner has OM entitlement
and this install has an active device activation, or can create one with an available seat
and project owner's active device activations do not exceed active seat entitlement
```

#### Activation flow

**First launch:**

1. User launches the OM desktop app.
2. App checks local secure storage for an OM device private key.
3. If no key exists, the app generates one.
4. App collects non-sensitive device metadata: user-visible device name, platform, app version,
   hashed fingerprint (if available).
5. App calls Satellite through the existing OM S3 credential context to create an activation.
6. Satellite resolves the project from the S3 credentials, then `user_id` from the project owner.
7. Satellite checks whether the project owner has an available OM seat for the requested tier.
8. If a seat is available, Satellite stores the activation and returns an initial lease.
9. If not, Satellite returns a typed error the app surfaces as "no available Object Mount seats".

**Existing activation (renewal):**

1. App loads its private key and activation ID from secure local storage.
2. App asks Satellite for a challenge or submits a signed renewal request.
3. Satellite verifies the signature using the stored public key.
4. Satellite re-checks entitlement directly against the `user_id` stored on the activation row — it
   does **not** re-resolve the user from the S3 credentials, so a deleted bootstrap credential does
   not break renewal.
5. Satellite confirms the activation row still exists and that this user still has enough seats for
   their active devices.
6. Satellite updates `last_seen_at` and returns a new signed lease.

#### Lease design

The lease is short-lived and signed by Satellite (a JWT or another compact signed token already used
by the codebase). The lease is **stateless**: the server signs it and hands it to the client but
does not store it; its expiry lives inside the signed token. The lease exists only to let the app
keep mounting offline for a bounded window without a network round-trip.

Suggested claims:

```text
user_id
activation_id
product_id
issued_at
expires_at
license_generation
```

`license_generation` is an optional but useful claim. It is a monotonically increasing version
number on the user's overall license state, stored once per user server-side and stamped into every
lease the user is issued. Anything that changes that user's licensing — seat count up or down,
revoke, tier switch, license-key rotation — bumps the counter.

Its purpose is to compensate for leases being **stateless and never stored**: the server keeps no
list of outstanding leases, so it has nothing to revoke directly. Bumping one counter instead marks
every previously-issued lease as belonging to an older license snapshot. On any server contact, a
lease whose `license_generation` is behind the user's current value is known to predate a change and
can be refused or forced to re-issue — revocation-like behavior without per-lease tracking.

It is genuinely optional because, for the common case (a seat reduction), it is redundant: the app
renews roughly hourly and renewal already re-checks entitlement against the database, so a downgrade
or revoke is caught within the renewal interval regardless. Where it adds value is for changes a
seat-count re-check would *not* catch — a tier switch, a scope change, a license-key rotation, or a
deliberate "invalidate everything now" support/security action — and for letting a client recognize
its still-unexpired lease is stale and drop it immediately rather than coasting to expiry. The cost
is maintaining the per-user counter and bumping it on every user-level license change.

Suggested default durations (exact values are product decisions):

```text
online renewal interval: 1 hour
lease lifetime: 24 hours
offline grace period: 3 to 7 days
```

**What happens when a lease expires:** nothing on the server — lease expiry triggers no state change
to the activation row or the seat. Behavior is entirely client-side, revalidated on renewal:

1. The app renews well before expiry (~hourly against a ~24h lease), so brief outages are absorbed.
2. While offline, the app keeps mounting on its still-valid token until it expires, then continues
   through the offline grace period.
3. After the grace period with no successful renewal, the app stops mounting and surfaces a
   "license needs attention" state.
4. On the next successful renewal, Satellite re-checks the activation row and entitlement, then
   issues a fresh token. If the seat is gone (downgrade or revoke), it simply declines.

The seat stays bound to the device until the row is deleted (revoke) or a cleanup job removes a
long-stale row — never because a lease lapsed. This is why lease expiry is not stored on the
activation table.

#### API sketch

Names are illustrative. The OM app activation endpoints run behind the existing OM S3 credential
path; the request body intentionally does not contain `project_id` or `user_id`. Console management
endpoints use the normal authenticated console session.

```text
POST   /api/v0/om/activations                      create activation + initial lease
POST   /api/v0/om/activations/{activationID}/lease renew lease (challenge + signature)
GET    /api/v0/om/activations                       list the user's active devices
DELETE /api/v0/om/activations/{activationID}        revoke a device, freeing the seat

GET    /api/v0/om/devices                            console: list project owner's OM devices
PATCH  /api/v0/om/devices/{activationID}             console: update device metadata (e.g. name)
DELETE /api/v0/om/devices/{activationID}             console: revoke a device
```

Example activation request / response:

```json
{
  "productID": "storj_os",
  "publicKey": "...",
  "deviceName": "Vitalii's MacBook Pro",
  "platform": "darwin-arm64",
  "appVersion": "1.0.0",
  "deviceFingerprintHash": "..."
}
```

```json
{
  "activationId": "...",
  "lease": "...",
  "leaseExpiresAt": "2026-07-01T12:00:00Z"
}
```

#### Desktop storage

The app stores the private key in OS-protected storage:

- macOS: Keychain, Secure Enclave where practical.
- Windows: Credential Manager, DPAPI, or TPM-backed storage where practical.
- Linux: Secret Service, KWallet, GNOME Keyring, or a documented encrypted fallback.

It also stores the activation ID, last received lease, lease expiry, and selected OM product ID. The
private key must not be exportable through normal app UI.

#### Enforcing one app instance per device

Seat binding controls devices, not local processes. The app should separately prevent multiple app
instances from using the same activation on the same machine, using a platform-appropriate
single-instance mechanism (named mutex on Windows; lock file/launch service on macOS/Linux; or the
desktop framework's single-instance APIs). This is a local correctness feature; it does not replace
server-side activation checks.

#### Managed device experience

The "managed device experience" is the user-facing and server-side controls around OM device
activations — not a separate licensing model. It can be built for the first release using the same
`om_device_activations` table.

**Console UI** — an Object Mount devices page for the project owner showing total seats, active
devices consuming seats, and available seats. Each device row shows device name, platform, app
version, product ID, derived state (active/stale), activated time (`created_at`), and last seen time
(`last_seen_at`). Supported actions: rename device, revoke device, copy activation ID for support.

**Derived statuses** (the table has no stored status):

- `active`: a row exists and `last_seen_at` is recent — consumes one seat, can receive leases.
- `stale`: a row exists but `last_seen_at` is older than the staleness threshold — still consumes a
  seat until cleanup deletes it, but the UI can flag it.
- `revoked`: there is no row — shown only transiently (e.g. a success toast), since revoked rows are
  not retained. A "recently revoked devices" list, if needed, comes from logs or a dedicated events
  table.

**Transfer and recovery** (replace laptop, reinstall OS, lose a device, remove a device):

- Revoking an active device deletes the row and frees the seat immediately.
- A revoked device cannot renew leases; its current token lapses at the end of the grace period.
- A revoked device may request activation again, creating a fresh activation if a seat is available.
- Repeated activation/revocation is rate-limited (keyed off logged events / fingerprint hash).

#### Billing state changes

When seat entitlement decreases, existing activations may exceed the new seat count. Recommended
first-version behavior:

1. Keep existing leases valid until their expiry.
2. Stop issuing new leases once the user is over their active seat entitlement.
3. Require the user to deactivate devices until active activations fit within the seat count.
4. The app shows a clear "license requires attention" state when renewal fails.

Manual resolution is preferred over automatic LRU revocation because it is more predictable and
better for user trust (see Alternatives considered).

#### Stale activation cleanup

If a device has not renewed for a long time, the UI can flag it stale (derived from `last_seen_at`)
and a cleanup job can delete the row automatically. This should be conservative because OM may be
used on machines that are powered off for extended periods. Possible policy:

```text
flag stale in the UI after 60 days without renewal (derived, no write needed)
auto-delete the row after 180 days without renewal, freeing the seat
```

This is a product decision. The first release can omit automatic cleanup and rely on manual
deactivation, or include stale-device cleanup at launch.

## Disclaimers

### Anti-goals

- Prevent all possible tampering by a malicious user with full control of their machine. This design
  raises the bar for casual seat sharing but does not make piracy impossible.
- Track every historical license lease event in the launch release.
- Use hardware fingerprinting as the only source of truth.
- Change the billing model — seat purchasing, free seats, and Stripe subscription logic stay as-is.

### Alternatives considered

- **Hardware ID as the primary license identity.** Rejected: hardware IDs differ by OS/vendor, can
  change after repairs, OS reinstall, motherboard replacement, VM migration, or privacy-setting
  changes, may require elevated permissions or be unavailable in sandboxes, can be spoofed, and raise
  privacy concerns if stored raw. They are kept only as hashed secondary metadata. A locally
  generated private key is the durable identity instead.
- **Concurrent-session licensing** (count running app processes). Rejected: it makes enforcement
  depend on perfect real-time presence and causes seats to be freed/reclaimed as the app starts and
  stops. Counting activated devices is simpler to reason about.
- **Automatic LRU revocation when seat count decreases.** Simpler operationally but worse for user
  trust; manual resolution lets the user choose which device keeps access.
- **A separate `om_license_leases` / `om_device_events` table at launch.** Deferred (see Out of
  scope) — `last_seen_at` plus server logs are sufficient until a concrete requirement appears.

### Open question

- Should seats always bind to users, or will future team/organization ownership need a separate
  model?
- Should Any Cloud enrollment use a dedicated OM identity instead of a data-path-irrelevant Storj S3
  credential as the bootstrap?
- Should free seats and paid seats behave differently for device binding?
- What offline grace period is acceptable for OM users?
- Should delegated admins be able to name and pre-authorize devices before first activation?
- What exact UI should appear when the user has no available seats?
- Should a revoked device be able to reactivate automatically if seats are available?
- Should activation be product-specific if a customer owns both Storj OS and Any Cloud seats?

## Reminders

### Security / Privacy

This design raises the bar for casual seat sharing but does not make piracy impossible — a user with
full control of a machine can tamper with the app, memory, local files, or network calls. Recommended
safeguards:

- Sign license leases server-side.
- Verify activation renewal using a private-key signature.
- Store private keys in OS-protected storage; never expose them through normal app UI.
- Never store raw hardware identifiers — only hashed fingerprints.
- Rate-limit activation attempts and repeated revoke/reactivate cycles.
- Require an extra confirmation step before revoking the current device, if it can be detected.

### Observability

- Record activation, renewal-failure, and revocation events in server logs as the source for support
  and abuse review.
- Include `user_id` and `activation_id` in server logs for support.
- Track active activation counts per user/tier and seats available vs. consumed.
- A dedicated `om_device_events` table can be added later if detailed audit/fraud analysis is needed.

### Test plan

Tests should confirm:

- Activation creates a row only when the project owner has an available seat; otherwise returns the
  typed "no available seats" error.
- The S3 credential resolves to the project owner once at activation; renewal re-checks entitlement
  against the stored `user_id` and does not break when the bootstrap credential/project is deleted.
- Seat counting equals the number of activation rows for a `(user_id, product_id)`.
- Renewal verifies the signature, updates `last_seen_at`, and issues a fresh lease.
- A lapsed lease causes no server-side state change; revocation deletes the row and frees the seat.
- A revoked device cannot renew but can re-activate if a seat is available (rate-limited).
- When entitlement decreases below active activations, no new leases are issued until devices are
  deactivated.
- Stale rows are flagged in the UI and (if enabled) auto-deleted after the cleanup threshold.

### Rollout

- Ship behind the existing OM entitlement decision point so device binding can be enabled per
  environment/tier.
- The first release can launch with manual deactivation only (no automatic stale cleanup).

### Rollback

- Device binding can be disabled at the entitlement check, falling back to user-level seat behavior
  without device enforcement. Existing activation rows remain harmless if enforcement is off.

## Out of scope

- A separate `om_license_leases` or `om_device_events` table. Do not add one unless there is a
  concrete requirement: detailed historical audit of every lease/activation/revocation event, a
  "recently revoked devices" list, current-session dashboards, strict concurrent-session replacement
  semantics, fraud analysis over renewal patterns, or support tooling that must inspect past leases.
  Until then, `last_seen_at` on `om_device_activations` plus server logs are sufficient.
- Team/organization seat ownership models.
- Deep platform-specific anti-tamper hardening.
