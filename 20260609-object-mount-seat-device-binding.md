tags: ["satellite", "object-mount", "licensing", "console", "payments"]
---

# Object Mount Seat Device Binding

## Essentials

OM seats are currently user-level licenses with nothing tying a seat to a machine. This document
proposes binding seats to desktop app installations, so one OM seat authorizes one active device.

**Scope: Storj OS only.** Device binding needs the app to reach the satellite (to activate and renew
leases); Storj OS mounts Storj buckets, so the app is already on that path. **Any Cloud is out of
scope** — it is licensed by a self-contained offline cunoFS key the satellite never sees at mount
time, so there is nothing to bind to; its seats are billed but not device-enforced.

Billing is unchanged: customers still buy seats per tier, free seats stay in the same user-level
pool, and the Stripe quantity is the source of truth for paid seats. Device binding is an
authorization layer on top.

The proposed release uses **one durable table** for device activations as the live source of truth (a
row exists iff a device is active and holding a seat) and **stateless signed leases** that are never
persisted. A separate lease/events history table can be added later if needed.

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

To make seat counts meaningful (one seat → one authorized device) we must bind seats to
installations, constrained by how the OM desktop app works:

- It has **no interactive auth** — it must not be trusted to submit a `user_id`/`project_id`.
- It operates with **S3 credentials created in a project** (which has an owner) and is on the Storj
  data path, so the satellite is reachable.
- It must behave reasonably **offline** for bounded windows.

### Goals

- One active seat maps to one authorized installation; users can deactivate old devices and activate
  new ones.
- Don't rely on raw hardware IDs as the primary identity.
- Work with the no-interactive-auth model and tolerate bounded offline use.
- Ship a concrete managed-device experience in the first release, without changing billing.

### Approach / Design

#### Core idea

Each installation creates a local key pair; the private key stays on the device, the public key goes
to Satellite as a **device activation** (with device metadata). Before mounting, the app requests a
short-lived **lease**, granted only while the activation is valid and the licensed user still has a
free seat. The flow:

1. App generates/loads a device key pair and sends its public key + metadata over the existing S3
   credential context.
2. Satellite resolves the user (and project) from the access grant and, if a seat is available,
   stores the activation and returns an initial lease.
3. The app periodically renews the lease (proving possession of the private key); Satellite renews
   while the activation stays valid.
4. Mounting is allowed only while the app holds a valid lease.

The durable identity is the locally generated private key. Hardware signals are stored only as
**hashed** secondary metadata for abuse/support heuristics.

#### Data model

One table, live source of truth — a row exists iff the device is active and holding a seat:

```text
om_device_activations
- id (pk)
- user_id, product_id
- public_key
- device_fingerprint_hash, device_name, platform, app_version
- last_seen_at, created_at, updated_at

unique (user_id, product_id, public_key)
```

- No `status` column: every row is active by definition. **Revoke = delete the row** (frees the
  seat); reactivation inserts a fresh row; `stale` is *derived* from `last_seen_at`, not stored.
- The unique constraint prevents double-activation and (as a btree) serves the seat-count prefix
  query on `(user_id, product_id)`; renewal looks up by `id`. A `last_seen_at` index can be added
  later if cleanup needs it.
- `created_at` = when the device started consuming a seat; `last_seen_at` = updated on every renewal,
  driving staleness and the UI "last seen".

#### Seat counting

```text
available seats = active seat entitlement - count(activation rows for user_id, product_id)
```

This counts **activated devices, not running processes**: a seat stays assigned until the user
deactivates the device or a stale-cleanup job removes it. So a laptop holds its seat while closed,
starting/stopping the app doesn't churn seats, users see which devices are assigned, and enforcement
doesn't depend on real-time presence.

#### Entitlement integration

**Reaching an entitlement.** The app discovers its entitlement implicitly via the mount path: S3
credentials → access grant (embedding an API key) → edge asks Satellite to authorize → Satellite
verifies the API key, derives its **project**, then the **project owner** (the account that holds
the seats and billing), and looks up that owner's OM entitlements **filtered by the resolved project
and bucket**.

So the licensed identity is the **project owner**, not whoever created the key — a member who holds
no seats may have created it. (The code currently resolves `keyInfo.CreatedBy`, with a TODO; that
must change to the project owner — see open questions.)

**License scope.** Entitlements are stored per user but each entry is scoped via optional
project (`PublicID`) / `BucketName` filters (empty = wildcard): user-level enables everything,
project-level enables all buckets in a project, bucket-level enables one bucket. A mount is allowed
if an active license matches the resolved `(user, project, bucket)` — which is why "a license set
for a project allows all its buckets."

**Identity capture.** The resolved owner `user_id` is captured **once, at activation**, and stored on
the row; renewal re-checks entitlement against that stored owner, not the S3 credentials. For a
user-level license this means deleting the bootstrap credential — or even its project — does not
orphan the activation (the owner still holds a pooled, account-level seat); the seat frees only on
revoke. (A project-/bucket-scoped license is narrower: deleting its scope target removes the
entitlement — see open questions.) The combined check at the entitlement decision point:

```text
an active OM license matches the resolved (user, project, bucket)
AND this install has an activation, or can create one with a free seat
AND the user's active activations do not exceed their seat entitlement
```

#### Lease design

The lease is short-lived, signed by Satellite (e.g. a JWT), and **stateless** — never stored
server-side; its expiry lives in the token. It exists only to let the app keep mounting offline for a
bounded window. Suggested claims: `user_id`, `activation_id`, `product_id`, `issued_at`,
`expires_at`, `license_generation`.

`license_generation` is **optional**: a per-user, monotonically increasing license-state version,
stamped into every lease and bumped on any user-level change. Because leases aren't stored, the
server can't revoke them individually; bumping the counter marks all older leases as a stale snapshot,
so a server contact presenting an old generation can be refused — revocation-like behavior without
per-lease tracking. It's optional because hourly renewal already re-checks entitlement (catching
downgrades/revokes); its value is for changes a seat-count check misses (tier/scope change, key
rotation, "invalidate everything now") and for telling a client to drop a still-valid lease early.

Suggested durations (product decisions): renew hourly, ~24h lease lifetime, 3–7 day offline grace.

**On expiry, nothing happens server-side** — it's all client-side, revalidated on renewal: the app
renews well before expiry; offline, it keeps mounting on the valid token, then through the grace
period; after that it stops and surfaces "license needs attention". On the next successful renewal
Satellite re-checks the activation + entitlement and either re-issues or declines (downgrade/revoke).
The seat stays bound until the row is deleted or a stale-cleanup removes it — never because a lease
lapsed.

#### API sketch

Names illustrative. App endpoints run behind the existing OM S3 credential path (no `project_id`/
`user_id` in the body); console endpoints use the authenticated session.

```text
POST   /api/v0/om/activations                      create activation + initial lease
POST   /api/v0/om/activations/{activationID}/lease renew lease (challenge + signature)
GET    /api/v0/om/activations                       list the user's active devices
DELETE /api/v0/om/activations/{activationID}        revoke a device, free the seat

GET    /api/v0/om/devices                            console: list the user's OM devices
PATCH  /api/v0/om/devices/{activationID}             console: update metadata (e.g. name)
DELETE /api/v0/om/devices/{activationID}             console: revoke a device
```

#### Desktop storage & single-instance

Store the private key in OS-protected storage (macOS Keychain/Secure Enclave; Windows Credential
Manager/DPAPI/TPM; Linux Secret Service/keyring or a documented encrypted fallback), non-exportable
through normal UI, plus the activation ID, last lease, expiry, and product ID. Separately, prevent
multiple app instances using the same activation on one machine via a platform single-instance
mechanism (named mutex / lock file) — a local correctness feature, not a replacement for server
checks.

#### Managed device experience

A console **Object Mount devices** page for the licensed user: total / consumed / available seats,
and per device its name, platform, app version, product, derived state, activated and last-seen
times; actions to rename, revoke, and copy the activation ID for support. Derived states (no stored
status): `active` (recent `last_seen_at`), `stale` (older than threshold — still holds a seat until
cleanup), `revoked` (no row; shown only transiently). A "recently revoked" list, if needed, comes
from logs or a dedicated events table.

**Transfer/recovery** (new laptop, OS reinstall, lost device): revoke deletes the row and frees the
seat immediately; a revoked device can't renew (its token lapses after grace) but may re-activate if
a seat is free. Rate-limit repeated activate/revoke cycles (keyed off logged events / fingerprint).

#### Billing decreases and stale cleanup

When entitlement drops below active activations: keep existing leases until expiry, stop issuing new
leases while over the limit, and require the user to deactivate devices to fit (app shows "license
requires attention"). Manual resolution is preferred over automatic LRU revocation — more
predictable, better for trust.

Stale cleanup is conservative (machines may be off for long stretches), e.g. flag stale in the UI
after 60 days without renewal and auto-delete the row after 180. A product decision; the first
release may rely on manual deactivation only.

## Disclaimers

### Anti-goals

- Prevent all tampering by a user with full control of their machine — this raises the bar for casual
  sharing, not piracy.
- Track every historical lease event at launch.
- Use hardware fingerprinting as the source of truth.
- Change the billing model.

### Alternatives considered

- **Hardware ID as primary identity** — rejected: varies by OS/vendor, changes on
  repair/reinstall/VM migration, may need elevated perms, is spoofable, and raises privacy concerns.
  Kept only as hashed secondary metadata; a local private key is the durable identity.
- **Concurrent-session licensing** (count running processes) — rejected: depends on real-time
  presence and churns seats on start/stop. Counting devices is simpler.
- **Automatic LRU revocation on decrease** — simpler ops but worse for trust; manual resolution lets
  the user choose which device keeps access.
- **A separate leases/events table at launch** — deferred (`last_seen_at` + logs suffice).

### Open question

- The licensed user is the **project owner**, not the key creator (today's `keyInfo.CreatedBy` is
  wrong and must change). Residual: on **project ownership transfer** the seat moves between accounts
  — re-resolve/migrate the stored owner, or only at next activation? (Today it's frozen at
  activation.)
- How do **project-/bucket-scoped** licenses interact with per-`(user_id, product_id)` seat counting?
  Does a device consume a seat from the matching scoped license, and should the row record the scope?
- Seats bind to users — will future team/org ownership need a separate model?
- Should free vs paid seats differ for binding? What offline grace is acceptable? Should admins
  pre-authorize devices? What UI shows when no seats are available? Should a revoked device
  auto-reactivate when a seat frees?
- The device UI must make clear it controls only Storj OS activations, not any Any Cloud seats the
  customer also holds (those are offline-licensed and not represented here).

## Reminders

### Security / Privacy

- Sign leases server-side; verify renewal via the device private-key signature.
- Store private keys in OS-protected storage; never expose them through UI.
- Never store raw hardware IDs — only hashed fingerprints.
- Rate-limit activation and repeated revoke/reactivate cycles; consider an extra confirmation before
  revoking the current device.

### Observability

- Log activation, renewal-failure, and revocation events (with `user_id` + `activation_id`) as the
  source for support and abuse review.
- Track active activations per user/tier and seats consumed vs. available.

### Test plan

- Activation creates a row only when the **project owner** has a free seat (else a typed "no
  available seats" error); a non-owner member's key resolves to the owner's entitlement.
- The access grant resolves to the owner + scope once at activation; renewal re-checks the stored
  owner and a user-level license survives deletion of the bootstrap credential/project.
- Scope works: project-scope allows all its buckets, bucket-scope allows one, user-scope allows all.
- Seat count = number of activation rows for `(user_id, product_id)`.
- Renewal verifies the signature, updates `last_seen_at`, issues a fresh lease; a lapsed lease causes
  no server state change; revoke deletes the row and frees the seat; a revoked device can re-activate
  if a seat is free (rate-limited).
- When entitlement drops below active activations, no new leases issue until devices are deactivated.
- Stale rows are flagged and (if enabled) auto-deleted after the threshold.

### Rollout

- Ship behind the OM entitlement decision point so binding can be enabled per environment/tier; the
  first release may launch with manual deactivation only.

### Rollback

- Disable binding at the entitlement check to fall back to user-level seats; existing activation rows
  are harmless when enforcement is off.

## Out of scope

- **Any Cloud** — offline cunoFS key licensing, no satellite contact, so no device binding; its seats
  are billed but not device-enforced.
- **Air-gapped / no-internet deployments** — binding needs satellite reachability; these fall back to
  offline key licensing with no device-level enforcement.
- A separate `om_license_leases` / `om_device_events` table — add only for a concrete need (full
  audit history, "recently revoked" list, session dashboards, fraud analysis); until then
  `last_seen_at` + logs suffice.
- Team/organization seat ownership models, and deep platform-specific anti-tamper hardening.
