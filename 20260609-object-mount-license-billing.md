tags: ["satellite", "payments", "object-mount", "billing"]
---

# Object Mount License Billing

## Essentials

Object Mount (OM) lets users mount cloud storage as a local drive. Access is sold per "seat" — one
seat licenses one account to use OM — billed monthly. This document covers how seats are priced,
billed, prorated, and cancelled, and how the two tiers are licensed.

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

OM is moving to a paid, seat-based model, effective **July 1, 2026**, with two tiers:

- **Storj OS** — $29/seat/month. Includes 500 GB of free storage per seat per month.
- **Any Cloud** — $39/seat/month. No included storage.

Both are **billed** per seat through Stripe, but they are **licensed differently**, because the app
only reaches Storj for Storj OS:

- **Storj OS** mounts Storj buckets — the app is on the Storj data path. Licensing is an enforced
  satellite-side **entitlement** (the seat-count model below); device-level enforcement is layered
  on separately (device binding).
- **Any Cloud** mounts third-party clouds (AWS, GCP, MinIO, …) with Storj nowhere on the data path.
  Licensing is a self-contained **offline cunoFS key** imported into the app; the satellite is never
  contacted at mount time. Seats are a **billing quantity only** — not technically enforced.

### Goals

- One invoice per customer per tier, regardless of seat count.
- Charge fairly for mid-period additions (prorated); keep access through the paid period on
  cancellation.
- Predictable, abuse-resistant included storage for Storj OS.
- An Any Cloud free trial that cannot be chained across accounts.
- 2 free seats per user, granted automatically.
- Self-serve purchase of either tier in the UI, no admin in the loop.
- OM remains **usable with no internet** (e.g. an AWS VPC with no egress), even if licensing is
  delivered manually — the Any Cloud key must work fully offline.

### Approach / Design

#### Billing: one subscription per tier, seats as quantity

Each customer has one Stripe subscription per tier; adding seats raises its `quantity` rather than
creating new subscriptions. So a 1000-seat customer gets one invoice, not 1000.

- **Adding seats** mid-period is charged immediately, prorated for the remaining days (no free seats
  between the add and the next renewal). *E.g. renewal on the 3rd, add a seat on the 21st with 13
  days left → ~\$12 now, then $29 at renewal.*
- **Cancelling seats** keeps access until the end of the paid period; no refund. The count drops at
  the next boundary.

#### Storj OS entitlement: seat changes over time

Storj OS access is enforced satellite-side. Seat entitlement is stored per user as a list of
`AccountLicense` rows (`Type` `OM`, `ProductID`, `Count`, validity). The active seat count at any
instant `T` is the **sum of `Count` over rows whose validity interval contains `T`** (device binding
later subtracts active devices from this).

A license today has only an **end** boundary (`ExpiresAt`/`RevokedAt`), which can express "active
now, ends later" but not "recorded now, effective later" — exactly what a deferred change needs. We
add a **`StartsAt`** boundary so each row is a half-open interval `[StartsAt, ExpiresAt)`, counted
only while `T` is inside it.

Every change is a **close-and-open**, never an in-place edit of `Count`, so past values survive as
history:

- **Increase** (immediate, prorated): close the active row at *now*, open a higher-count row at
  `StartsAt = now`.
- **Decrease** (deferred): cap the active row at the period end, open a lower-count row at
  `StartsAt = period end` — stored now, dormant until the boundary.

Because intervals carry both a start and an end, they never overlap in counting, so a query for any
instant — past or present — returns the right number, and the interval sequence *is* the history:

> *Drop 3 → 2 on Jun 9 (period ends Jun 30): stored `{3, [.., Jul 1)}` + `{2, [Jul 1, ..)}`.
> Jun 15 → 3, Jul 2 → 2 — never 5. Then increase 2 → 4 on Jul 5: a Jun-15 query still returns 3.*

Two consequences: without `StartsAt`, the lower row would count immediately and a 3 → 2 drop would
transiently read **5**; and a row may be deleted/collapsed **only while its interval was never live**
(e.g. a pending decrease reversed before its boundary) — an interval that has been in effect is
history and must not be rewritten.

#### Included storage (Storj OS only)

Each active Storj OS seat includes 500 GB/month, fixed at the **start of the calendar month**
(midnight UTC on the 1st); mid-month changes don't affect that month. This needs no separate
snapshot — it's just a point-in-time query of the intervals above as of the 1st.

> *3 seats on Aug 1, +2 on Aug 15 → August stays 1.5 TB; September (5 seats) → 2.5 TB.*

Edge case: a first seat bought mid-month gets no included storage that month (no interval covers the
1st → count 0). Communicate this clearly in the UI.

#### Free trial (Any Cloud) and free seats

- **Free trial**: not charged during the trial; billing starts automatically at trial end. **A card
  is required** (otherwise trials chain indefinitely across accounts). Cancelling during the trial
  ends access immediately with no charge.
- **Free seats**: every user gets 2, automatically on signup; existing users via a one-time bulk
  grant on July 1, 2026. They grant OM access only — no included storage.

#### Any Cloud licensing (offline cunoFS key)

Any Cloud is **decoupled from the seat-enforcement machinery** above. The app is gated by a
self-contained **cunoFS license key** validated **offline**; the satellite is never contacted at
mount time, so a "seat" here is a billing quantity, not an enforced limit.

**What the key is** (from `storj/cunofs-keygen`, backed by `storj/cunofs-licensing`): a base64 blob
of version, `license_type` (`personal`/`pro`/`ent`/`*_eval`/`edu_*`), a UUID, start, expiry, and a
64-bit feature bitmask, plus an **RSA-PSS/SHA-256 signature**. The OM client verifies it against an
embedded RSA public key (separate signing keys per platform) and writes it locally. The format is
dictated by the cunoFS engine, so we **generate** it — replacing it isn't an option.

**Issuing it (self-serve)**: today `cunofs-keygen` produces the key out of band (offline with the
RSA private key, or an AWS Lambda `GenerateLicense2` that holds it) and an admin hands it into the
grant API, which stores it; the user obtains it and imports it. For self-serve, the satellite mints
it inline on purchase (vendor `cunofs-keygen`/`cunofs-licensing`, or a small signing service),
mapping the tier/term to `license_type`/validity/features (trial → `*_eval`). The user then
**downloads it from the UI and imports it** — the import step is unchanged. We vendor
`cunofs-keygen` in the satellite and supply the RSA signing key (per platform) through **config that
defaults empty**: the keys live in **OpenBao** and infra populates the config at deploy time, so
self-serve Any Cloud minting stays off until the secret is present. The admin "supply a `Key`" path
can stay for backwards compatibility.

**Offline / air-gapped (supported)**: the key is generated once, **delivered manually** if needed,
and validated offline forever after (cunoFS supports perpetual expiry). Any online mechanism (key
refresh, future enforcement) must stay **optional** so disconnected deployments still work.

**Enforcement limits**: a customer holds a **single key**, not one per seat — it encodes no seat or
device count, so changing seats never generates or modifies a key (it's reissued only on
renewal/expiry or a feature change). The key is a **bearer token** (no embedded identity): usable on
any number of machines until expiry, and not revocable (validated locally). So Any Cloud seat count
is **purely contractual**, enforced by issuance + expiry + contract — short expiries with reissue for
connected customers, long/perpetual keys for air-gapped ones (where there is effectively no technical
enforcement, by design).

#### Summary of key decisions

| Decision | Reason |
|---|---|
| One subscription per tier, quantity for seats | One invoice for enterprise customers |
| Prorated immediate charge when adding seats | Fair both ways; no free seats |
| No refund on cancellation, access to period end | Simplicity; no value lost |
| Included storage fixed at month start | Prevents end-of-month abuse; predictable |
| Card required for free trial | Prevents chained trials across accounts |
| 2 free seats, no included storage | Adoption without giving away storage |
| Time-bounded intervals (`[StartsAt, ExpiresAt)`) | Correct count at any instant; clean history |
| Any Cloud: satellite mints the cunoFS key for self-serve | No admin in the loop; format dictated by cunoFS |
| Any Cloud decoupled from seat enforcement; offline + single key | Validated offline, no satellite contact, must work air-gapped |
| cunoFS signing keys in OpenBao, injected via empty-default config | Keys never committed; feature off until infra populates; rotation owned by infra |

## Disclaimers

### Anti-goals

- **No proration/refund on cancellation.** Day-level credits add system and invoice complexity for
  no lost value (access runs to period end).
- **No mid-month increase of included storage.** Otherwise a customer could add seats on the last
  day, claim a full month of storage cheaply, then cancel. Month-start locking removes this.
- **No technical seat/device enforcement for Any Cloud.** Offline validation (and the air-gap
  requirement) make it impossible; closing the gap would mean making the app online. Accepted.
- **We don't prevent an Any Cloud key from being used to mount Storj.** A cunoFS key unlocks the
  *engine*, not a backend — "Storj OS" / "Any Cloud" are Storj's billing tiers, invisible to cunoFS.
  So one offline Any Cloud key can run OM against Storj (e.g. via Storj's S3 gateway, which looks
  like any S3 endpoint) on unlimited machines, bypassing Storj OS seats and device binding. This is
  inherent to the offline bearer-token model and not fixable client-side; pricing/packaging and the
  cunoFS question below are the only levers.

### Alternatives considered

- **One subscription per seat** — rejected: 1000 invoices for a 1000-seat customer.
- **Proration/refunds on cancellation** — rejected for complexity; access-to-period-end instead.
- **Free trial without a card** — rejected: trivially chained across accounts.
- **In-place `Count` edit at the boundary (no `StartsAt`)** — avoids a schema field and can't
  double-count, but can't pre-record the next-period count, losing "going forward" auditability.
- **Forcing Any Cloud online (entitlement + device binding)** — contradicts the offline import flow
  and breaks no-internet VPCs. Any Cloud stays offline.
- **Mandatory short-lived keys with online refresh for Any Cloud** — enables some enforcement but
  breaks air-gap; kept as an optional path for connected customers only.

### Open question

- Migrating existing OM customers onto the new pricing on July 1, 2026 — reconciling existing seats
  and in-flight billing with the bulk free-seat grant.
- Should unused included storage be surfaced to the customer, or only reflected in the invoice?
- The active-license conflict check keys on `Type`+`PublicID`+`BucketName` and ignores `ProductID`;
  account-level OM licenses for both tiers share an empty scope and would collide. Include
  `ProductID`?
- Any Cloud uses a single key per customer (decided). Residual: how do support/audit reconcile billed
  seats against usage, given the app never reports the key back?
- **cunoFS ask:** can a key be scoped by **backend type** (and can the engine identify Storj
  endpoints) so an Any Cloud key can be minted *without* the ability to mount Storj? If yes, it
  closes the casual cross-tier leak (an Any Cloud key substituting for Storj OS seats); if no, that
  leak is an accepted limitation. One for Nikos's team.

## Reminders

### Security / Privacy

- Trial card collection goes through Stripe; no raw card data stored here.
- Seat grants/cancellations must be authorized as the account owner.
- The cunoFS RSA signing keys live in **OpenBao** and reach the satellite via config that defaults
  empty (feature off until infra populates it) — keys are never committed. Rotation is an infra
  operation in OpenBao + redeploy, done only in lockstep with the client's embedded public key.
- A cunoFS key is a bearer token: prefer short expiries with reissue for connected customers.
  Air-gapped customers get long/perpetual keys, safeguarded only by issuance + contract.

### Observability

- Track seat add/cancel events and resulting prorated charges.
- Track free-seat grants (per-signup and the July 1, 2026 bulk job, with affected-account count).
- Track free-trial starts, conversions, cancellations.

### Test plan

- N seats → one subscription per tier with `quantity = N`, one invoice per period.
- Mid-period add → immediate prorated charge; cancel → access to period end, no refund, count drops
  only at the boundary.
- Active count = sum of `Count` over intervals covering the instant: 3 → 2 reads 3 then 2 (never 5);
  a mid-month 3 → 4 still returns 3 for a 1st-of-month query (included storage unchanged) and 4 for
  "now".
- Included storage = 500 GB × (seats at the 1st); a first seat bought mid-month gets none that month.
- Free trial requires a card, auto-bills at trial end, ends with no charge if cancelled in-trial.
- Every new account gets 2 free seats (no storage); bulk grant applies once to existing users.
- A self-serve Any Cloud purchase mints a valid cunoFS key (correct `license_type`/validity/features)
  that verifies against the client's embedded public key; it imports and works with **no network**;
  and a later downgrade does **not** invalidate an already-issued key (no clawback).

### Rollout

- New pricing and the bulk free-seat grant are effective July 1, 2026; the per-signup 2-free-seat
  grant ships ahead of that date.

### Rollback

- New seat purchases and trials can be gated behind a feature flag while existing subscriptions
  continue unchanged.

## Out of scope

- **Per-seat device/credential binding** (Storj OS) — planned for a later phase; this billing model
  supports it without changes.
- Minimum-retention exemption rules for OM storage.
