tags: [ "web/satellite", "edge/linksharing", "edge/authservice" ]
---

# Password Protected Linksharing

## Essentials

### Header

Date: 2026-05-05

Owner: wilfred-asomanii

Accountable:
- Console team

Consulted:
- mobyvb

### Context

The console ticket has previously worked on multiple UX improvements to Linksharing (see [this ticket](https://github.com/storj/storj-private/issues/1521)).
For this design/[ticket](https://github.com/storj/storj-private/issues/1539), we want to allow users to set a password on their share links. This will require
that anyone with the link knows the password to be able to access the data at the share link.

### Goal

- Allow users to generate share links which require passwords before allowing access to the data.

### Non-Goal

- This password will not be used to encrypt the data in any way. The encryption passphrase is what does this. This means anyone with means to access the data can
still bypass Linksharing to do that.

### Approach / Design

This feature depends on the existing `Record.UsageTags []string` slice on the auth record
([see](https://github.com/storj/edge/blob/a6d1a0d450da28657ff2276590c59f216f773276/pkg/auth/authdb/storage.go#L29)).
No schema, proto, or DBX changes are required: the column already round-trips through Badger and Spanner, and through the HTTP
registration path that the satellite UI uses.

The tag prefix to be used is `share-pwd=`. The value differs between what the satellite UI sends and what authservice persists:

| Direction                                                 | Tag value                                                              | Notes                                                                                                                                      |
|-----------------------------------------------------------|------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Inbound (satellite UI → authservice on `POST /v1/access`) | `share-pwd=<password>`                                                 | Plaintext password. Transient — never persisted in this form.                                                                              |
| Stored (authservice → linksharing)                        | `share-pwd=<base64url(t=<iters>,m=<KiB>,p=<lanes>,s=<salt>,h=<hash>)>` | Argon2id parameters, salt, and hash. The outer base64url alphabet is comma-free, so the existing usage_tags comma constraint is preserved. |

#### Satellite UI

This is where the process begins; a user creates an optionally password protected share link here.

- `web/satellite/src/components/dialogs/ShareDialog.vue` gets an optional password field inside the Advanced section.
  Validation: minimum length 8, no commas (commas are rejected by `processUsageTags` ([See](https://github.com/storj/edge/blob/a6d1a0d450da28657ff2276590c59f216f773276/pkg/auth/authdb/db.go#L217))).
- `generatePublicCredentials` in `web/satellite/src/composables/useLinksharing.ts` accepts an optional `password` and, when set, appends
  `"share-pwd=<password>"` to the `usage_tags` array passed into `agStore.getEdgeCredentials`.
- `web/satellite/src/api/accessGrants.ts` extends `getGatewayCredentials` to forward the `usage_tags` array on the
  `POST /v1/access` body.

#### Authservice

Access registration happens here. If a password is provided, it will be stored here.

- `processUsageTags` in `pkg/auth/authdb/db.go` will be updated to recognize the `share-pwd=` prefix. On registration:
  - Reject if `public=false` (share links are public).
  - Reject more than one `share-pwd=` entry on a single record to avoid mistaken ambiguity.
  - Generate a 16-byte random salt, derive Argon2id hash to be stored
- The plaintext password never reaches the database layer.
- `newAccess` in `pkg/auth/httpauth/resources.go` redacts `share-pwd=...` values from request logging or error wrapping.

#### Linksharing

This is what handles the flow when a user visits the share link.

- New logic will be added to parse the stored `share-pwd=` tag and verify submitted passwords with Argon2id.
  It will also set signed cookies so that users are not repeatedly prompted for the password.
- Per accessKeyID and/or IP ratelimiting will be added to mitigate brute-force attacks.
- `ServeHTTP` in `pkg/linksharing/sharing/handler.go` will be updated to first check whether the share has `share-pwd=` tag.
  If it does and an unlock cookie is not present, it would require the password. This has to be done before serving any data.
- A new route `POST /s/{accessKeyID}/__auth` (and the `/raw/` twin) accepts the form submission, verifies the password, and
  on success sets two cookies — one scoped to `/s/<id>/`, one to `/raw/<id>/` — so the recipient can switch between viewing and
  direct download without a second prompt. `middleware/preflight.go` is updated to allow `POST` only on these `__auth` paths.

## Disclaimers

### Anti-goals
- Server-side rendered features (distribution map, ZIP archive previews) should keep working once the recipient unlocks.
- The feature should not change behavior for share links without a password set.
- Plaintext passwords must not appear in any operator-visible output (request logs, eventkit, panic traces).

### Alternatives considered

**Embed the password hash as a macaroon caveat on the access grant.**
Caveats are not a designed to be extensions points for UI only features.

**Hash the password client-side in WASM before sending.** 
TLS already protects the password in transit, and adding a dependency on Argon2 increases wasm size.

### Open questions

- **CAPTCHA layer.** With the inclusion of per-accessKeyID and/IP rate limiting, we mitigate the risk factor of brute force
  attacks, but do we also want to include captcha requirements on password submissions?
- **Paid-tier gating.** Do we want this feature to be available to paid-users and others with paid privileges?
- **Raw downloads.** Do we want password protected shares to be downloadable?
- **Previews.** Do we want to support social previews to work for these shares? 

## Reminders

### Security / Privacy

- Unlock cookies are path-scoped to the originating share so not to mistakenly skip requiring password for other shares.
- The plaintext password only ever traverses TLS; it is never persisted by authservice, or logged anywhere.

### Observability

We should consider tracking these event's via eventkit:

- `password_required` — request hit a `share-pwd=` share and had no valid cookie.
- `password_attempt` — every POST to `__auth`, with a `success` boolean. Useful for spotting brute-force ramps before rate limit kicks in.
- `password_success` — successful unlock; cookie set.
- `password_locked` — rate limit threshold reached.

### Test plan

- Each changed component in the Linksharing flow will be unit tested
- Manual testing: on QA, we want to make sure that;
  - shared files are indeed password protected
  - rate limiting works as expected
  - password prompting does not happen after the user first unlocks the file
  - password will be required again eventually when the unlock cookie expires.

### Rollout

- This feature is significant and as such should be behind a feature flag that is disabled by default.
- The Authservice update will be deployed first. The Authservice may not need a flag because the changes there have no effect
  if the Satellite UI doesn't send the new tag.
- Linksharing changes deploy next, with its own feature flag; if disabled, password protected shares should not work anymore.
- The satellite UI changes are deployed next, also behind a feature flag.

### Rollback

All the changes being proposed to be made can easily be reverted. Except that password protected shares should not work
anymore once the changes are reverted. If not, we break the user's expectation that their share is password protected.

## Out of scope

- Password-protecting access through other access points where users with the right access can see the data.
- Password recovery / reset. Forgetting the password requires revoking and re-issuing the share.

## Tickets

Tickets follow the deployment order in [Rollout](#rollout).

### 1. Authservice: accept and hash share-pwd= usage tag

To support password protected Linksharing, we have to update the Authservice to store user provided share passwords.
Update `processUsageTags` in `pkg/auth/authdb/db.go` to recognize the `share-pwd=` prefix, derive an Argon2id hash on registration,
and store the encoded form. Redact the inbound tag from logs and error wrapping in `pkg/auth/httpauth/resources.go`.

**AC**:
- `share-pwd=<password>` on a `public=true` registration is rewritten to `share-pwd=<base64url(t,m,p,salt,hash)>` before persistence.
- Registration is rejected when `public=false` or when more than one `share-pwd=` entry is present.
- Plaintext password does not appear in any log output or wrapped error.

### 2. Linksharing: enforce the password gate

We want to update Linksharing to require password if a share access has a `share-pwd` usage tag.

**AC**:
- A share with a `share-pwd=` tag renders the password prompt before any data is served, on both `/s/` and `/raw/` paths.
- Correct password sets `/s/<id>/` and `/raw/<id>/` unlock cookies with reasonable expiry.
- New password submission endpoints (`/s/` and `/raw/` variants) with per-accessKeyID and/or IP rate limit are added.
- Eventkit emits `password_required`, `password_attempt`, `password_success`, `password_locked`.
- This implementation should be behind a feature flag.

### 3. Satellite UI: add the password field

To support password protected Linksharing, we need users to be able to optionally set passwords on their share links.

**AC**:
- A password field is added in the Advanced section, with min-length 8 and no-comma validation.
- The non-empty password is sent to authservice via a `share-pwd=<password>` entry in `usage_tags`.
- A new feature flag is added for this.
