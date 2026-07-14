# Accounts, Identity & Auth

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Peeled from `schema.md` §1; this doc now owns all
account/login tables plus the signup, invite, and SSO-linking flows. SSO providers:
**Google and GitHub** (ideal pair; more can be added later — the `user_sso_connections`
table is provider-agnostic).

---

## 1. Model overview

**Sign-off:** _unsigned_

- A **user** is a human with 1+ **login methods**: an email/password credential and/or
  linked SSO connections (Google, GitHub). Password is optional (SSO-only users exist).
- An **account** is the billing/team entity. Sign-up creates user + account with that user as
  admin. All resources hang off `account_id`.
- **`user_id` is the join key across login methods** — `user_sso_connections`,
  `account_members`, `sessions`, etc. all FK to `users.id`. One user row per human, however many ways they log
  in. Email is **never** a join key or FK: SSO logins resolve on the stable
  `(provider, provider_user_id)` pair, passwords resolve on the `users` row itself. Email's
  only structural role is the signup-time collision check (§3B: "an account with this email
  exists — log in to link") — a routing hint, deliberately frail-proof because matching email
  alone never logs you in or merges anything.
- No email verification at launch for email/password signup (decided) — but the column and
  token table exist so enabling it later is a flag, not a migration.

## 2. Tables

**Sign-off:** _unsigned_

### accounts  (`acct_`)

| column                  | type        | null? | notes                                    |
| ----------------------- | ----------- | ----- | ---------------------------------------- |
| sequence_id             | bigint      | no    | identity; never exposed (ids.md §4)      |
| id                      | text PK     | no    | `acct_...`                               |
| name                    | text        | no    | display name                             |
| billing_email           | text        | yes   | falls back to the admin's email          |
| billing_phone           | text        | yes   | optional; stored as entered              |
| billing_address_line_1  | text        | yes   | optional                                 |
| billing_address_line_2  | text        | yes   |                                          |
| billing_city            | text        | yes   |                                          |
| billing_region          | text        | yes   | state / province / prefecture            |
| billing_postal_code     | text        | yes   |                                          |
| billing_country         | text        | yes   | ISO 3166-1 alpha-2 (`US`, `JP`)          |
| api_version             | date        | yes   | Stripe-style pin, set at first API use   |
| status                  | text        | no    | `active \| suspended \| closed`          |
| created_at              | timestamptz | no    |                                          |
| updated_at              | timestamptz | no    |                                          |
| deleted_at              | timestamptz | yes   | soft delete                              |

- **`slug` nuked (2026-07-14):** console URLs and the account switcher use `accounts.id`
  directly (Stripe-style). A vanity slug buys nothing at launch and brings uniqueness,
  rename, and squatting baggage; trivially additive later if a product need appears.
- **No `inactive` status (2026-07-14):** `status` encodes how we *treat* the account's
  interactions with the platform — `active`, `suspended` (our action), `closed`
  (self-closure or ban) — not activity level. Dormancy is derivable from usage; an
  `inactive` value would add transitions with no policy meaning.
- Billing address fields are deliberately loose free-text (international formats vary too
  much for strict per-field validation); `billing_country` is the only coded field. Stripe
  stays the validator of record where tax/receipts care.

### users  (`user_`)

| column            | type        | null? | notes                                                                                                   |
| ----------------- | ----------- | ----- | ------------------------------------------------------------------------------------------------------- |
| sequence_id       | bigint      | no    | identity; never exposed (ids.md §4)                                                                     |
| id                | text PK     | no    | `user_...`                                                                                              |
| email             | text UNIQUE | no    | **canonicalized** (trim + lowercase); the login/collision key                                           |
| email_original    | text        | no    | exactly as the user typed it (or as the SSO provider reported it); **non-unique**, display/support only |
| email_verified_at | timestamptz | yes   | set by verification OR by trusted SSO                                                                   |
| password_hash     | text        | yes   | **null for SSO-only users**; argon2id when set                                                          |
| display_name      | text        | yes   |                                                                                                         |
| avatar_url        | text        | yes   | from SSO profile when available                                                                         |
| totp_secret       | text        | yes   | encrypted at rest (true MFA, later)                                                                     |
| created_at        | timestamptz | no    |                                                                                                         |
| updated_at        | timestamptz | no    |                                                                                                         |
| deleted_at        | timestamptz | yes   | soft delete                                                                                             |

Canonicalization is conservative: trim whitespace + Unicode-lowercase, nothing else. No
dot-stripping or plus-tag-stripping — those are provider-specific ownership semantics we
must not guess at. All lookups and uniqueness run against `email`; `email_original` is what
we show back and what support sees.

### user_sso_connections  (`sso_`) — SSO links (renamed from `user_identities`, 2026-07-14)

| column            | type        | null? | notes                                                               |
| ----------------- | ----------- | ----- | ------------------------------------------------------------------- |
| sequence_id       | bigint      | no    | identity; never exposed (ids.md §4)                                 |
| id                | text PK     | no    | `sso_...`                                                           |
| user_id           | text FK     | no    | → users                                                             |
| provider          | text        | no    | `google \| github`                                                  |
| provider_user_id  | text        | no    | Google `sub` / GitHub numeric id — the STABLE key (never the email) |
| provider_email    | text        | no    | email as reported by the provider at link time                      |
| provider_login    | text        | yes   | GitHub username etc., display only                                  |
| created_at        | timestamptz | no    |                                                                     |
| last_used_at      | timestamptz | no    | set at link, touched on each login                                  |
|                   |             |       | UNIQUE (provider, provider_user_id)                                 |
|                   |             |       | UNIQUE (user_id, provider) — one connection per provider per user   |

Naming: "identity" is too large a word for what this is — a link to an external login
provider. Considered `user_sso_identities`, `oauth_accounts`, `external_logins`,
`sso_logins`; **`user_sso_connections`** wins because it matches the console verbs exactly
("Connect Google" / disconnect) and stays scoped to SSO.

No OAuth access/refresh tokens stored — we use OAuth strictly for authentication (profile +
email scopes), not to act on the user's behalf. Nothing to leak.

### account_members  (`mem_`)

| column      | type        | null? | notes                               |
| ----------- | ----------- | ----- | ----------------------------------- |
| sequence_id | bigint      | no    | identity; never exposed (ids.md §4) |
| id          | text PK     | no    | `mem_...`                           |
| account_id  | text FK     | no    | → accounts                          |
| user_id     | text FK     | no    | → users                             |
| role        | text        | no    | `admin \| member`                   |
| created_at  | timestamptz | no    |                                     |
|             |             |       | UNIQUE (account_id, user_id)        |

### invitations  (`inv_`) — now supports code AND magic link

| column              | type        | null? | notes                                               |
| ------------------- | ----------- | ----- | --------------------------------------------------- |
| sequence_id         | bigint      | no    | identity; never exposed (ids.md §4)                 |
| id                  | text PK     | no    | `inv_...`                                           |
| account_id          | text FK     | no    | → accounts                                          |
| email               | text        | no    | invitee (invite is bound to this email)             |
| role                | text        | no    | `admin \| member`                                   |
| code                | text UNIQUE | no    | short human-typeable code (e.g. `7F3K-QT9M`)        |
| token_hash          | text        | no    | sha256 of the magic-link token (long, unguessable)  |
| invited_by_user_id  | text FK     | no    | → users                                             |
| accepted_by_user_id | text FK     | yes   | → users                                             |
| created_at          | timestamptz | no    |                                                     |
| expires_at          | timestamptz | no    | default 7 days                                      |
| accepted_at         | timestamptz | yes   | null until accepted                                 |
| revoked_at          | timestamptz | yes   |                                                     |

Code and link are two doors to the same invitation: the email contains the magic link; the
console shows the code for copy/paste into "join with code."

### sessions  (`sess_`)

| column      | type        | null? | notes                               |
| ----------- | ----------- | ----- | ----------------------------------- |
| sequence_id | bigint      | no    | identity; never exposed (ids.md §4) |
| id          | text PK     | no    | `sess_...`                          |
| user_id     | text FK     | no    | → users                             |
| token_hash  | text        | no    | sha256 of cookie value              |
| ip          | inet        | yes   |                                     |
| user_agent  | text        | yes   |                                     |
| created_at  | timestamptz | no    |                                     |
| expires_at  | timestamptz | no    |                                     |
| revoked_at  | timestamptz | yes   |                                     |

Sessions are read on every console request, so they **can/will be Redis-cached** (the
session-cache slot already reserved in `architecture.md` §1): keyed by token hash, TTL from
`expires_at`, entry deleted on revoke/logout. Postgres stays the source of truth; the cache
is a read-through.

### api_keys  (`key_`; secret `foundation_api_...`, shown once)

| column              | type        | null? | notes                                         |
| ------------------- | ----------- | ----- | --------------------------------------------- |
| sequence_id         | bigint      | no    | identity; never exposed (ids.md §4)           |
| id                  | text PK     | no    | `key_...`                                     |
| account_id          | text FK     | no    | → accounts                                    |
| created_by_user_id  | text FK     | no    | → users                                       |
| title               | text        | no    | **immutable after creation**                  |
| description         | text        | yes   | editable                                      |
| key_prefix          | text        | no    | `foundation_api_` + first 6 payload chars     |
| key_hash            | text        | no    | sha256; secret never stored                   |
| created_at          | timestamptz | no    |                                               |
| last_used_at        | timestamptz | yes   | throttled updates                             |
| expires_at          | timestamptz | yes   | null = no expiry                              |
| revoked_at          | timestamptz | yes   |                                               |

### email_verifications  (`emailver_`)

| column      | type        | null? | notes                               |
| ----------- | ----------- | ----- | ----------------------------------- |
| sequence_id | bigint      | no    | identity; never exposed (ids.md §4) |
| id          | text PK     | no    | `emailver_...`                      |
| user_id     | text FK     | no    | → users                             |
| token_hash  | text        | no    | sha256 of emailed token             |
| created_at  | timestamptz | no    |                                     |
| expires_at  | timestamptz | no    |                                     |
| used_at     | timestamptz | yes   |                                     |

Dormant at launch (no verification requirement yet); the table exists so enabling
verification later is a feature flag, not a migration.

### password_resets  (`pwreset_`)

| column      | type        | null? | notes                               |
| ----------- | ----------- | ----- | ----------------------------------- |
| sequence_id | bigint      | no    | identity; never exposed (ids.md §4) |
| id          | text PK     | no    | `pwreset_...`                       |
| user_id     | text FK     | no    | → users                             |
| token_hash  | text        | no    | sha256 of emailed token             |
| created_at  | timestamptz | no    |                                     |
| expires_at  | timestamptz | no    | short (~1h)                         |
| used_at     | timestamptz | yes   | single-use                          |

Live at launch (password reset works day one). Reset consumption revokes all existing
sessions for the user.

## 3. Initial signup flows

**Sign-off:** _unsigned_

All three converge on the same final step: create `users` row → create `accounts` row →
create `account_members(admin)` → create session → land in console onboarding.

**That final step is one database transaction.** User, account, membership, and session rows
(plus the `user_sso_connections` row for SSO signups) commit atomically; any failure rolls back
everything. No orphan users, no memberless accounts, no half-signups to reap.

**A. Email + password**

1. `POST /v1/auth/signup` `{email, password, display_name?}`.
2. Reject if email already exists (→ "log in instead", with SSO hints if that email has
   linked SSO connections).
3. Create user (`password_hash` set, `email_verified_at` NULL — no verification at launch;
   when we enable it later, unverified users get a nag + eventual feature gates, not a
   hard signup block).

**B. Google**

1. `GET /v1/auth/oauth/google/start` → redirect to Google (scopes: openid, email, profile;
   PKCE; `state` cookie).
2. Callback: verify `state`, exchange code, read `sub` + email + `email_verified`.
3. Lookup `user_sso_connections(google, sub)`:
   - exists → log in (update `last_used_at`).
   - not found, email matches an existing user → **do NOT silently merge**: show "an account
     with this email exists — log in to link" (flow D2 below handles the link). Prevents
     account takeover via a Google account created later for the same address.
   - genuinely new → create user (`password_hash` NULL, `email_verified_at` = now, since
     Google asserts the email) + connection row + account.

**C. GitHub**

Same as B with GitHub OAuth (scopes: `read:user`, `user:email`); use the **numeric user id**
as `provider_user_id` (logins are mutable); pick the primary **verified** email from
`/user/emails`. GitHub accounts with no verified email → block with a clear message.

## 4. Invite signup flows

**Sign-off:** _unsigned_

Entry: magic link (`/join?token=...`) or code entry (`/join` + `7F3K-QT9M`). Both resolve the
invitation (valid, unexpired, unrevoked), then branch on the visitor's state:

1. **Not signed in, no existing user for invite email** → choose signup method:
   - **Email/password**: email field pre-filled AND LOCKED to the invite email; set password
     → user created → membership created → invite marked accepted.
   - **Google**: OAuth as flow B; the invite is accepted for the resulting user. If the
     Google email ≠ invite email: allow, but membership binds to the authenticated user and
     the mismatch is recorded on the invitation (`accepted_by_user_id` tells the truth; admins see
     "invited x@a, joined as y@b" in the audit log).
   - **GitHub**: same as Google.
2. **Not signed in, user exists for that email** → login screen (any method they have) →
   accept.
3. **Already signed in** → confirmation screen ("join {account} as {role}?") → accept.
   Signed in as a different email than invited → show both, let them proceed or switch user.

Invites never create accounts — only memberships. A user can belong to many accounts; the
console gets an account switcher from day one.

## 5. Adding login methods (linking)

**Sign-off:** _unsigned_

All linking happens **while authenticated**, from console → Security settings. The three
flows you listed:

**1. Email/password user adds Google or GitHub**

Settings → "Connect Google" → OAuth round-trip (with a `link` intent in `state`, bound to the
current session) → on callback, attach `user_sso_connections` row to the CURRENT user.
- If that Google/GitHub connection is already linked to a *different* user → hard error
  ("already connected to another account") — connections are never moved implicitly.
- Provider email ≠ user email → allowed (people have many emails); shown in settings.

**2. SSO-only user adds a password**

Settings → "Set password" → since SSO-only users have no current password to confirm, require
a **fresh SSO re-auth** (re-run the OAuth flow, max age enforced) → set `password_hash`.
Email/password login now works alongside SSO.

**3. SSO user adds another SSO provider**

Same as flow 1 — Gmail-first user connects GitHub (or vice versa). The connections list in
settings shows all connected methods with `last_used_at` and a disconnect button.

**Unlink rule:** a user must always retain ≥1 login method — the last remaining connection (or
the password, if it's the only method) cannot be removed. Disconnecting a method requires a
fresh re-auth with any remaining method.

**Login resolution (any time):** try `user_sso_connections` by (provider, sub) for SSO; by email +
password check for passwords. The email-collision rule from §3B applies universally: matching
email alone NEVER logs you into or merges with an existing user — it routes you to log in
with an existing method first, then link.

## 6. API endpoints (delta to api.md §9)

**Sign-off:** _unsigned_

| Endpoint                                | Method | Purpose                                    |
| --------------------------------------- | ------ | ------------------------------------------- |
| `/v1/auth/oauth/{provider}/start`       | GET    | Begin OAuth (google \| github); `intent=login\|signup\|link` |
| `/v1/auth/oauth/{provider}/callback`    | GET    | Complete OAuth per flows above              |
| `/v1/auth/sso_connections`              | GET    | List connected SSO providers                |
| `/v1/auth/sso_connections/{id}`         | DELETE | Disconnect (guard: ≥1 login method remains) |
| `/v1/auth/password`                     | PUT    | Set/change password (fresh re-auth required)|
| `/v1/invitations/{code-or-token}`       | GET    | Resolve invite for the join page            |
| `/v1/invitations/{id}/accept`           | POST   | Accept (authenticated)                      |
