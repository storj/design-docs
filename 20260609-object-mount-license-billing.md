tags: ["satellite", "payments", "object-mount", "billing"]
---

# Object Mount License Billing

## Essentials

Object Mount (OM) lets users mount Storj storage as a local drive on their computer. Access is sold
per "seat" — one seat is one license that allows one account to use Object Mount, billed as a
monthly subscription. This document describes how seats are priced, billed, prorated, and cancelled,
and the design decisions behind that billing model.

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

Object Mount is moving to a paid, seat-based model. We need a billing design that:

- works for both individual users (1–2 seats) and enterprises (hundreds or thousands of seats),
- handles mid-period seat additions and cancellations fairly,
- supports two product tiers (one with included storage, one without),
- offers a free trial without opening the door to trial abuse,
- and grants a small number of free seats to drive adoption.

New pricing becomes effective **July 1, 2026**. There are two tiers:

- **Storj OS** — $29/seat/month. Includes 500 GB of free storage per seat per month.
- **Any Cloud** — $39/seat/month. No included storage.

### Goals

- Bill OM seats per customer with a single invoice per tier, regardless of seat count.
- Charge fairly for mid-period seat additions (prorated) and avoid giving seats away for free.
- Let customers who cancel keep access through the period they have already paid for.
- Grant included storage for the Storj OS tier in a way that is predictable and abuse-resistant.
- Offer an Any Cloud free trial that cannot be chained indefinitely across accounts.
- Grant every user 2 free OM seats automatically.

### Approach / Design

#### One subscription per tier, seat count as quantity

Each customer has a single Stripe subscription per product tier. Adding seats increases the
`quantity` on that subscription rather than creating a new subscription. A customer with 1000 seats
therefore receives **one** invoice and **one** email per month, not 1000.

#### Adding seats mid-period (prorated, charged immediately)

When a customer adds seats mid-period, they are charged immediately for the remaining days of the
current billing period. The customer only pays for the time they actually use, and Storj does not
give seats away for free between the add date and the next renewal.

> *Example: subscription renews on the 3rd of each month. Customer adds a seat on July 21st (13 days
> left in the period). They pay approximately \$12 immediately, then $29 at the next renewal.*

#### Cancelling seats (access until period end, no refund)

When a customer cancels seats, they keep access until the end of the billing period they have
already paid for. No refund is issued. At the end of the period the seat count drops to the new
lower number, and future invoices reflect the reduced count.

The seat-count transition is tracked precisely in our system so the invoicing process always sees
the correct seat count for any given point in time — both historically and going forward.

#### How seat changes are represented over time

Seat entitlement is stored per user as a list of license entries (see the entitlements service —
`AccountLicense`), each carrying a `Type` (`OM`), a `ProductID` (Storj OS vs Any Cloud), a seat
`Count`, and a validity window. The active seat count for a tier at any instant `T` is the **sum of
`Count` over the licenses whose validity window contains `T`** (device-level seat enforcement, which
subtracts active devices from this number, is designed separately).

A license today has only an **end** boundary (`ExpiresAt` / `RevokedAt`). That is enough to express
"active now, ends later", but it cannot express "recorded now, takes effect later" — and that
distinction is exactly what a deferred seat change needs. We therefore add a **`StartsAt` /
`EffectiveFrom`** boundary so each license describes a half-open interval `[StartsAt, ExpiresAt)`. A
license is counted only while `T` falls inside its interval.

A seat change is **never an in-place edit of an existing interval's `Count`**. It always *closes the
current interval and opens a new one* (a "close-and-open"), so the previous count survives as
history. The only difference between the two directions is where the new interval starts and whether
it charges:

- **Increasing seats (immediate, prorated).** Close the active row at *now* and open a new,
  higher-count row with `StartsAt = now`. The customer gets the new seats immediately, matching the
  prorated charge.
- **Decreasing seats (deferred to period end).** Cap the current row at the period boundary
  (`ExpiresAt = period end`) and record a new, lower-count row with `StartsAt = period end`. The
  lower count is stored *now* but is dormant until the boundary, so it is never counted early.

Because each interval has both a start and an end, intervals never overlap in *counting* even when
several exist in storage at once — the system reports the correct number at any instant, historically
and going forward, with no double-counting.

##### Point-in-time history

Because a change closes one interval and opens another rather than overwriting a count, the sequence
of `[StartsAt, ExpiresAt)` intervals *is* the seat-count history. A query for any instant returns the
count of the interval covering it.

> *Example: customer has 3 seats from June 1, then increases to 4 on June 9.*
> *Stored: `{Count: 3, [Jun 1, Jun 9)}` and `{Count: 4, [Jun 9, ...)}`.*
> *Queried on June 1 → still **3**. Queried now → **4**.* If we had instead bumped the existing
> row's `Count` from 3 to 4 in place, the June 1 query would wrongly read 4 — the original value
> would be gone.

This is why an increase must split the interval rather than mutate `Count`: included storage and any
other point-in-time question depend on the past value remaining queryable.

A consequence: a row may be safely **collapsed or deleted only while its interval has never been
live** — e.g. a pending decrease that is reversed before its boundary never applied to any real time,
so dropping it loses nothing. An interval that has already been in effect is real history and must
never be deleted or rewritten.

> *Example: period ends June 30 (renews July 1); customer drops 3 → 2 on June 9.*
> *Stored from June 9 onward:*
> *`{Count: 3, [..., Jul 1)}` and `{Count: 2, [Jul 1, ...)}`.*
> *Queried on June 15 → only the first interval contains the date → **3 seats**.*
> *Queried on July 2 → only the second → **2 seats**. Never 5.*

Without the `StartsAt` boundary, the lower-count row would be active the moment it is inserted and
would be summed on top of the still-active higher-count row — a customer dropping 3 → 2 would
transiently read **5** seats until the old row expired. The alternative of *not* pre-recording the
change (creating the lower-count row only at the boundary, via the billing chore) avoids the extra
field but loses the "going forward" auditability; see Alternatives considered.

#### Included storage (Storj OS only)

Each active Storj OS seat includes 500 GB of free storage per month (3 seats → 1.5 TB, 10 seats →
5 TB).

Included storage is calculated from the number of seats the customer has at the **start of the
calendar month** (midnight UTC on the 1st). Seats added or cancelled during the month do not change
the included storage for that month.

Thanks to the interval model above, this needs no separate stored snapshot: the start-of-month seat
count is just a point-in-time query of the entitlement intervals as of midnight UTC on the 1st. A
mid-month increase opens a new interval starting later in the month, so it does not affect the value
the 1st-of-month query returns.

> *Example: customer has 3 seats on August 1st and adds 2 more on August 15th. Their included
> storage for August is 1.5 TB (based on the 3 seats at the start of the month). From September 1st,
> with 5 seats, they get 2.5 TB.*

New-customer edge case: a customer who buys their first seat on July 15th receives no included
storage for July, because no interval covers July 1st (the 1st-of-month query returns zero seats).
Their included storage begins on August 1st. This must be communicated clearly in the product UI so
customers are not surprised.

#### Free trial (Any Cloud)

The Any Cloud tier includes a free trial during which the customer is not charged. When the trial
ends, billing starts automatically. **A credit card is required to start the trial** — without it,
someone could create unlimited accounts to chain free trials together. If the customer cancels
during the trial, access ends immediately and no charge is made.

#### Free seats

Every user receives 2 free Object Mount seats, granted automatically on account creation. Existing
users receive them via a one-time bulk grant on July 1, 2026 (the new-pricing effective date). Free
seats grant Object Mount access only — they do **not** include any free storage.

#### License key issuance (legacy mechanism for Any Cloud)

The signed license `Key` on an OM license is a **legacy licensing mechanism that predates
entitlements**. It is how the OM desktop app authorizes access to **any S3 cloud other than Storj**
(the Any Cloud tier): for those backends Storj is nowhere on the data path, so this key — not a
Storj credential — is the licensing anchor. It is still how Any Cloud works today.

This key is **not generated by the satellite**. It is produced outside the satellite (in the OM
backend) and handed in: the admin grant API accepts a `Key`, and the satellite simply stores it on
the license and passes it back to the app on request. That works today because granting such a
license goes through an admin / back-office step where the externally-generated key can be supplied.

Self-serve, automatic granting removes that step. When a user buys (or is auto-granted) an Any Cloud
seat directly, there is no admin in the loop to supply a key. So either the legacy key-generation
and signing logic moves from the OM backend into the satellite (or a service the satellite calls) so
it can run inline on the self-serve grant path, or the Any Cloud flow is reworked so it no longer
depends on the legacy key. Whichever we choose:

- The signing secret/material the key depends on must be available to the satellite and managed
  accordingly (see Security / Privacy).
- The admin "supply a `Key`" path can remain for backwards compatibility, but the self-serve path
  must not depend on a human supplying the key.

The exact key format and signing scheme are owned by the OM backend today; porting them (or
replacing them for self-serve) is a prerequisite for self-serve Any Cloud licensing and should be
scoped with that team.

#### Summary of key decisions

| Decision | Reason |
|---|---|
| One subscription per tier, quantity for seat count | Avoids generating hundreds of invoices for enterprise customers |
| Charge immediately (prorated) when adding seats | Fair to both sides; prevents getting seats for free |
| No refund on cancellation, access until period end | Simplicity; customer loses no value |
| Included storage based on seat count at month start | Prevents abuse; simple and predictable |
| Credit card required for free trial | Prevents unlimited trial abuse via multiple accounts |
| 2 free seats for all users, no included storage | Encourages adoption without giving away storage credits |
| Time-bounded license intervals (`[StartsAt, ExpiresAt)`) | Correct seat count at any instant; no double-counting on deferred decreases |
| Self-serve Any Cloud must produce the legacy license key without an admin | The license key is a pre-entitlements mechanism for non-Storj S3 access, currently supplied by hand via the admin grant path |

## Disclaimers

### Anti-goals

- **No mid-month proration on cancellation.** Partial refunds require day-level credit calculations
  and partial invoices, adding significant complexity for both the billing system and the customer
  reading their invoice. Since the customer retains full access through the period they paid for, no
  value is lost — the simplicity trade-off is worth it.
- **No mid-month increase of included storage.** Allowing mid-month seat additions to immediately
  increase included storage creates an abuse opportunity: a customer could add many seats on the
  last day of the month, gain a large storage credit for the entire month at a fraction of the cost,
  then cancel immediately. Locking the calculation to the start of the month eliminates this.

### Alternatives considered

- **One subscription per seat.** Rejected: an enterprise with 1000 seats would receive 1000 invoices
  and emails per month. Quantity-on-a-single-subscription gives the same revenue with one invoice.
- **Proration / partial refunds on cancellation.** Rejected for the complexity reasons above; the
  customer keeps access for the paid period instead.
- **Free trial without a credit card.** Rejected: trivially abused by creating new accounts to chain
  trials indefinitely.
- **Mutating seat count in place at the period boundary (no `StartsAt` field).** Instead of
  pre-recording the lower count, keep a single license row and let the billing chore change its
  `Count` at the period rollover. This avoids the schema change and cannot double-count, but it does
  not record the upcoming change ahead of time, so the system cannot answer "what will this
  customer's seat count be next period" from stored state — it loses the "going forward"
  auditability. Rejected in favor of the time-bounded interval model.

### Open question

- How should an existing Object Mount customer be migrated onto the new pricing on July 1, 2026 —
  in particular, how are their existing seats and any in-flight billing reconciled with the bulk
  free-seat grant?
- Should included storage that goes unused in a month be visibly surfaced to the customer, or only
  reflected silently in the invoice?
- The active-license conflict check currently keys on `Type` + `PublicID` + `BucketName` and ignores
  `ProductID`. For account-level OM licenses both tiers share an empty scope, so a customer holding
  both a Storj OS and an Any Cloud seat would collide. Does the conflict check need to include
  `ProductID` before per-tier seats ship?
- Where should the license-key signing material live, and who owns key rotation once generation moves
  into the satellite?

## Reminders

### Security / Privacy

- Credit-card collection for the free trial goes through Stripe; no raw card data is stored on our
  side.
- Seat grants and cancellations must be authorized as the account owner.
- Moving license-key generation into the satellite brings the signing secret/material into satellite
  scope. It must be stored securely (e.g. via the existing secret-management path), access-controlled,
  and rotatable without invalidating still-valid leases mid-period.

### Observability

- Track seat additions, cancellations, and the resulting prorated charges per customer.
- Track free-seat grants (per-account on signup, and the July 1, 2026 bulk grant as a one-time job
  with a count of accounts affected).
- Track free-trial starts, conversions, and cancellations.
- Surface the start-of-month seat snapshot used for included-storage calculation, for auditability.

### Test plan

Tests should confirm:

- A customer with N seats has one subscription per tier with `quantity = N`, and receives one
  invoice per period.
- Adding seats mid-period produces an immediate prorated charge for the remaining days.
- Cancelling seats keeps access through the paid period, issues no refund, and reduces the seat
  count only at the next period boundary.
- Included storage equals 500 GB × (seats at midnight UTC on the 1st) and does not change for
  mid-month additions/cancellations.
- A first seat bought mid-month yields zero included storage that month.
- The free trial requires a card, starts billing automatically at trial end, and ends immediately
  with no charge on cancellation during the trial.
- Every new account is granted 2 free seats with no included storage; the bulk grant applies once
  to existing users.
- The active seat count at an instant equals the sum of `Count` over licenses whose
  `[StartsAt, ExpiresAt)` interval contains that instant: a 3 → 2 decrease reads 3 before the period
  boundary and 2 after, never 5; a mid-period increase reads the higher count immediately.
- A mid-month increase (3 → 4 on the 9th) preserves history: a query as of the 1st still returns 3
  (and so included storage for the month is unchanged), while a query for "now" returns 4.
- A self-serve Any Cloud grant produces a valid legacy license key without any externally-supplied
  key (whether by the satellite generating it or by a reworked flow), and the key is accepted by the
  OM desktop app for non-Storj S3 access.

### Rollout

- New pricing and the bulk free-seat grant become effective July 1, 2026.
- The 2-free-seat grant for new accounts ships ahead of the effective date so signups are correct
  from day one.

### Rollback

- If billing issues arise, new seat purchases and trials can be gated behind a feature flag while
  existing subscriptions continue unchanged.

## Out of scope

- **Per-seat assignment (device/credential binding).** Today seats are account-level — any use of
  Object Mount on the account consumes from the seat pool. Binding individual seats to specific
  devices or credentials so a company can control exactly which machines are authorised is planned
  for a later phase. The billing model described here is designed to support it without changes.
- Minimum-retention exemption rules for Object Mount storage.
