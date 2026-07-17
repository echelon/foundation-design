# Accounts, Identity & Auth

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Peeled from `schema.md` §1; this doc now owns all
account/login tables plus the signup, invite, and SSO-linking flows. SSO providers:
**Google and GitHub** (ideal pair; more can be added later — the `user_sso_connections`
table is provider-agnostic). Session-token design, login mechanics, and the authorization
model: **`auth.md`**. Audit trails: **`audit_logs.md`**.

---

## 1. Model overview

**Sign-off:** _unsigned_

- A **user** is a human with 1+ **login methods**: an email/password credential and/or
  linked SSO connections (Google, GitHub). Password is optional (SSO-only users exist).
- An **account** is the billing/team entity. All resources hang off `account_id`.
- **A user belongs to zero or more accounts**, with a role per membership (`owner, admin,
  member` — permissions differ per account; matrix in `auth.md` §4). The console puts the
  account id in the URL, Stripe-style (`/{acct_id}/…`), and switching accounts is switching
  URLs. **Any user can create additional accounts** at any time — sign-up creates user +
  first account with that user as **owner**, and "New account" from the switcher creates
  more (creator = owner).
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

**Sign-off:** _unsigned_

| column                 | type        | null? | notes                                  |
| ---------------------- | ----------- | ----- | -------------------------------------- |
| sid                    | bigint      | no    | identity; never exposed (ids.md §4)    |
| id                     | text PK     | no    | `acct_...`                             |
| name                   | text        | yes   | display name                           |
| billing_email          | text        | yes   | falls back to the admin's email        |
| billing_phone          | text        | yes   | optional; stored as entered            |
| billing_address_line_1 | text        | yes   | optional                               |
| billing_address_line_2 | text        | yes   |                                        |
| billing_country_code   | text        | yes   | ISO 3166-1 alpha-2 (`US`, `JP`)        |
| billing_region         | text        | yes   | state / province / prefecture          |
| billing_city           | text        | yes   |                                        |
| billing_postal_code    | text        | yes   |                                        |
| api_version            | date        | yes   | Stripe-style pin, set at first API use |
| status                 | text        | no    | `active, suspended, closed`            |
| created_at             | timestamptz | no    |                                        |
| updated_at             | timestamptz | no    |                                        |
| deleted_at             | timestamptz | yes   | soft delete                            |

- **`slug` nuked (2026-07-14):** console URLs and the account switcher use `accounts.id`
  directly (Stripe-style). A vanity slug buys nothing at launch and brings uniqueness,
  rename, and squatting baggage; trivially additive later if a product need appears.
- **No `inactive` status (2026-07-14):** `status` encodes how we *treat* the account's
  interactions with the platform — `active`, `suspended` (our action), `closed`
  (self-closure or ban) — not activity level. Dormancy is derivable from usage; an
  `inactive` value would add transitions with no policy meaning.
- Billing address fields are deliberately loose free-text (international formats vary too
  much for strict per-field validation); `billing_country_code` is the only coded field. Stripe
  stays the validator of record where tax/receipts care.
- **Every change to the billing address/phone/email fields writes a
  `billing_address_changed` event to the account audit log** (`audit_logs.md` §3), in the
  same transaction as the update.

### users  (`user_`)

**Sign-off:** _unsigned_

| column            | type        | null? | notes                                                                                                                                             |
| ----------------- | ----------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| sid               | bigint      | no    | identity; never exposed (ids.md §4)                                                                                                               |
| id                | text PK     | no    | `user_...`                                                                                                                                        |
| email             | text UNIQUE | no    | **canonicalized** (trim + lowercase); the login/collision key                                                                                     |
| email_original    | text        | no    | exactly as the user typed it (or as the SSO provider reported it); **non-unique**, display/support only                                           |
| email_clean       | text        | no    | **aggressively canonicalized** (gmail dot/plus stripping etc.); NOT unique, never a login key — abuse/duplicate-detection signal only             |
| email_verified_at | timestamptz | yes   | set by verification OR by trusted SSO                                                                                                             |
| password_hash     | text        | yes   | **null for SSO-only users**; argon2id when set                                                                                                    |
| password_version  | int         | no    | vector clock; **+1 on every password set/change/reset**; embedded in session JWTs (`auth.md` §1) so bulk session invalidation is one integer bump |
| display_name      | text        | yes   |                                                                                                                                                   |
| totp_secret       | text        | yes   | encrypted at rest (true MFA, later)                                                                                                               |
| created_at        | timestamptz | no    |                                                                                                                                                   |
| updated_at        | timestamptz | no    |                                                                                                                                                   |
| deleted_at        | timestamptz | yes   | soft delete                                                                                                                                       |

Three email tiers, three jobs: `email` is conservatively canonicalized (trim + Unicode
lowercase, nothing else) — the ONLY login/uniqueness key. `email_original` is exactly as
typed — display/support. `email_clean` goes further — provider-aware normalization
(gmail-style dot removal and `+tag` stripping, etc.); it is deliberately NOT unique and
never authenticates anything (those transforms are ownership guesses), it exists to catch
duplicate-account/abuse patterns. All three update together on every email change.
`avatar_url` DROPPED (2026-07-14) — avatars are now the `user_avatars` table below.

### email_changes  (`emailchg_`) — NEW 2026-07-14

**Sign-off:** _unsigned_

| column     | type        | null? | notes                               |
| ---------- | ----------- | ----- | ----------------------------------- |
| sid        | bigint      | no    | identity; never exposed (ids.md §4) |
| id         | text PK     | no    | `emailchg_...`                      |
| user_id    | text FK     | no    | → users                             |
| old_email  | text        | no    | canonicalized; **not unique**       |
| new_email  | text        | no    | canonicalized; **not unique**       |
| created_at | timestamptz | no    |                                     |

Every email change, atomically with the `users` update: (1) this row; (2) `email_changed`
events in the user audit log AND every member account's audit log (`audit_logs.md` §3);
(3) `email`/`email_original`/`email_clean` re-derived; (4) all outstanding
`email_verifications` rows invalidated; (5) the avatar pipeline re-runs (gravatar hash +
letter, below).

### user_avatars  (`avatar_`) — NEW 2026-07-14

**Sign-off:** _unsigned_

| column              | type        | null? | notes                                                                                                                       |
| ------------------- | ----------- | ----- | --------------------------------------------------------------------------------------------------------------------------- |
| sid                 | bigint      | no    | identity; never exposed (ids.md §4)                                                                                         |
| id                  | text PK     | no    | `avatar_...`                                                                                                                |
| user_id             | text FK     | no    | → users; UNIQUE (one row per user)                                                                                          |
| source              | text        | no    | enum `AvatarSource`: `custom, gravatar, letter, generated` — what currently displays                                        |
| custom_file_id      | text FK     | yes   | → files; user-uploaded image (purpose = avatar)                                                                             |
| gravatar_file_id    | text FK     | yes   | → files; **our mirrored copy** of the gravatar — we never hotlink                                                           |
| gravatar_hash       | text        | yes   | sha256(trim + lowercase(email)); recomputed on email change                                                                 |
| gravatar_checked_at | timestamptz | yes   | last existence probe                                                                                                        |
| gravatar_dismissed  | bool        | no    | user disabled gravatars for their account                                                                                   |
| generated_seed      | text        | no    | random hash fixed at creation; color = seed mod N_COLORS, character = seed mod N_CHARS (14 × 14 default; both configurable) |
| letter              | text        | yes   | first letter of email; re-derived on every email change                                                                     |
| created_at          | timestamptz | no    |                                                                                                                             |
| updated_at          | timestamptz | no    |                                                                                                                             |

**Gravatar (verified 2026-07-14):** avatar fetches need no API key —
`https://gravatar.com/avatar/{sha256(trim+lowercase(email))}?s=512&d=404`; `d=404` returns
HTTP 404 when no gravatar exists (the existence probe), sizes up to 2048px. On signup and on
every email change we probe, and if one exists we download and store OUR copy as a `files`
row (`origin = mirrored_url`) — customers' consoles never call gravatar.com.

**Display precedence:** `custom` (if uploaded) → `gravatar` (if mirrored AND not dismissed)
→ `letter` (default fallback: email's first letter on the seed-derived color) → `generated`
(seed color + seed character, for when there's no usable letter). Uploading a custom avatar
overrides but does NOT discard the gravatar state — remove the custom image and the
gravatar is right back. "Dismiss gravatar" sets `gravatar_dismissed` and falls through to
letter/generated; the mirrored file is kept (dismissal is reversible).

### user_sso_connections  (`sso_`) — SSO links (renamed from `user_identities`, 2026-07-14)

**Sign-off:** _unsigned_

| column           | type        | null? | notes                                                               |
| ---------------- | ----------- | ----- | ------------------------------------------------------------------- |
| sid              | bigint      | no    | identity; never exposed (ids.md §4)                                 |
| id               | text PK     | no    | `sso_...`                                                           |
| user_id          | text FK     | no    | → users                                                             |
| provider         | text        | no    | enum `SsoProvider`: `google, github`                                |
| provider_user_id | text        | no    | Google `sub` / GitHub numeric id — the STABLE key (never the email) |
| provider_email   | text        | no    | email as reported by the provider at link time                      |
| provider_name    | text        | yes   | the user's real/display name as reported by the provider            |
| provider_login   | text        | yes   | GitHub username etc., display only                                  |
| created_at       | timestamptz | no    |                                                                     |
| last_used_at     | timestamptz | no    | set at link, touched on each login                                  |
|                  |             |       | UNIQUE (provider, provider_user_id)                                 |
|                  |             |       | UNIQUE (user_id, provider) — one connection per provider per user   |

Naming: "identity" is too large a word for what this is — a link to an external login
provider. Considered `user_sso_identities`, `oauth_accounts`, `external_logins`,
`sso_logins`; **`user_sso_connections`** wins because it matches the console verbs exactly
("Connect Google" / disconnect) and stays scoped to SSO.

No OAuth access/refresh tokens stored — we use OAuth strictly for authentication (profile +
email scopes), not to act on the user's behalf. Nothing to leak.

### account_members  (`mem_`)

**Sign-off:** _unsigned_

| column     | type        | null? | notes                                                        |
| ---------- | ----------- | ----- | ------------------------------------------------------------ |
| sid        | bigint      | no    | identity; never exposed (ids.md §4)                          |
| id         | text PK     | no    | `mem_...`                                                    |
| account_id | text FK     | no    | → accounts                                                   |
| user_id    | text FK     | no    | → users                                                      |
| role       | text        | no    | enum `MemberRole`: `owner, admin, member`                    |
| created_at | timestamptz | no    |                                                              |
|            |             |       | UNIQUE (account_id, user_id)                                 |
|            |             |       | UNIQUE (account_id) WHERE role = 'owner' — exactly one owner |

Role rules (REVISED 2026-07-14 — `owner` added above `admin`; enforcement + permission
matrix in `auth.md` §4):

- The **account creator is the owner**; there is exactly ONE owner per account (partial
  unique index above). Invitations grant `admin` or `member`, never `owner`.
- **The owner cannot remove themselves** (or be removed/demoted) — ownership must be
  transferred first (`POST /v1/account/transfer_ownership`, owner-only, target must already
  be a member, fresh re-auth; atomic swap: target → owner, old owner → admin). Because the
  owner can never leave, an account can never be left admin-less.
- **Every membership mutation is audited** — `member_added`, `member_role_changed`,
  `member_removed`, `ownership_transferred` — to the account audit log AND the affected
  user's audit log, in the mutation's transaction (`audit_logs.md` §3).

### invitations  (`inv_`) — now supports code AND magic link

**Sign-off:** _unsigned_

| column              | type        | null? | notes                                              |
| ------------------- | ----------- | ----- | -------------------------------------------------- |
| sid                 | bigint      | no    | identity; never exposed (ids.md §4)                |
| id                  | text PK     | no    | `inv_...`                                          |
| account_id          | text FK     | no    | → accounts                                         |
| email               | text        | no    | invitee (invite is bound to this email)            |
| role                | text        | no    | `admin, member`                                    |
| code                | text UNIQUE | no    | short human-typeable code (e.g. `7F3K-QT9M`)       |
| token_hash          | text        | no    | sha256 of the magic-link token (long, unguessable) |
| invited_by_user_id  | text FK     | no    | → users                                            |
| accepted_by_user_id | text FK     | yes   | → users                                            |
| created_at          | timestamptz | no    |                                                    |
| expires_at          | timestamptz | no    | default 7 days                                     |
| accepted_at         | timestamptz | yes   | null until accepted                                |
| revoked_at          | timestamptz | yes   |                                                    |

Code and link are two doors to the same invitation: the email contains the magic link; the
console shows the code for copy/paste into "join with code."

**Controller rule (2026-07-14):** the invite endpoint REJECTS an invite to an email whose
user is already an accepted member of the account (`409` naming the member). But an
**unaccepted** invitee may be re-invited freely — re-sending revokes the outstanding
invitation and issues a fresh one (new code, new expiry).

### sessions  (`sess_`)

**Sign-off:** _unsigned_

| column     | type        | null? | notes                                                    |
| ---------- | ----------- | ----- | -------------------------------------------------------- |
| sid        | bigint      | no    | identity; never exposed (ids.md §4)                      |
| id         | text PK     | no    | `sess_...`                                               |
| user_id    | text FK     | no    | → users                                                  |
| ip         | inet        | yes   | creation IP                                              |
| user_agent | text        | yes   | raw UA at creation                                       |
| device     | text        | yes   | parsed from UA ("Chrome on macOS") for the sessions list |
| created_at | timestamptz | no    |                                                          |
| expires_at | timestamptz | no    |                                                          |
| revoked_at | timestamptz | yes   |                                                          |

**`token_hash` DROPPED (2026-07-14):** the cookie is a **signed JWT**
(`{sid, uid, pv, iat, exp}`) — the signature is the authenticity, the row is looked up by
`sid`, and every request is still online-checked so revocation works (full design:
`auth.md` §1; `pv` = `users.password_version`). Users can list their live sessions (device,
creation IP, UA) and kill any of them — `auth.md` §3. Sessions are read on every console
request, so they **can/will be Redis-cached** (the session-cache slot in `architecture.md`
§1): keyed by session id, TTL from `expires_at`, entry deleted on revoke/logout. Postgres
stays the source of truth; the cache is a read-through.

### api_keys  (`key_`; secret currently `foundation_api_...`, shown once)

**Sign-off:** bt 2026-07-14

| column             | type        | null? | notes                                                           |
| ------------------ | ----------- | ----- | --------------------------------------------------------------- |
| sid                | bigint      | no    | identity; never exposed (ids.md §4)                             |
| id                 | text PK     | no    | `key_...`                                                       |
| account_id         | text FK     | no    | → accounts                                                      |
| created_by_user_id | text FK     | no    | → users                                                         |
| title              | text        | no    | **immutable after creation**                                    |
| description        | text        | yes   | editable                                                        |
| key_prefix         | text        | no    | the key's OWN minted prefix + first 6 payload chars (see below) |
| key_hash           | text        | no    | sha256; secret never stored                                     |
| created_at         | timestamptz | no    |                                                                 |
| last_used_at       | timestamptz | yes   | throttled updates                                               |
| expires_at         | timestamptz | yes   | null = no expiry                                                |
| revoked_at         | timestamptz | yes   |                                                                 |

**The secret prefix is config, not schema (2026-07-14).** Nothing in this table hardcodes
`foundation_api_`: keys are looked up by `key_hash`, and the auth layer accepts a
**configured set** of secret prefixes — one *canonical* prefix used for minting new keys,
plus any number of *legacy* prefixes still accepted for verification. A service rename is
therefore: add the new canonical prefix, demote the old one to legacy — every existing key
keeps working, forever. `key_prefix` stores what each key was actually minted with, so the
console displays old keys truthfully after a rename. Same mechanism covers
`foundation_whsec_` webhook secrets. (Wire-side description: `api.md` §1.)

### email_verifications  (`emailver_`)

**Sign-off:** _unsigned_

| column       | type        | null? | notes                                                                 |
| ------------ | ----------- | ----- | --------------------------------------------------------------------- |
| sid          | bigint      | no    | identity; never exposed (ids.md §4)                                   |
| id           | text PK     | no    | `emailver_...`                                                        |
| user_id      | text FK     | no    | → users                                                               |
| email        | text        | no    | the canonicalized address this token verifies — snapshot at send time |
| token_hash   | text        | no    | sha256 of emailed token                                               |
| created_at   | timestamptz | no    |                                                                       |
| expires_at   | timestamptz | no    |                                                                       |
| validated_at | timestamptz | yes   | renamed from `used_at` (2026-07-14) — non-null IS the validity signal |

This is the table verification emails are sent against. **`email` is snapshotted on the row
(2026-07-14)** — double-defensive: outstanding rows are already invalidated on email change
(the first guard), and validation additionally requires `row.email == users.email` (the
second), so a token can never verify an address it wasn't minted for. Successful validation
sets `users.email_verified_at` and writes an `email_verified` user-audit event
(`audit_logs.md` §3). Dormant at launch (no verification requirement yet); the table exists
so enabling verification later is a feature flag, not a migration.

### password_resets  (`pwreset_`)

**Sign-off:** _unsigned_

| column     | type        | null? | notes                               |
| ---------- | ----------- | ----- | ----------------------------------- |
| sid        | bigint      | no    | identity; never exposed (ids.md §4) |
| id         | text PK     | no    | `pwreset_...`                       |
| user_id    | text FK     | no    | → users                             |
| token_hash | text        | no    | sha256 of emailed token             |
| created_at | timestamptz | no    |                                     |
| expires_at | timestamptz | no    | short (~1h)                         |
| used_at    | timestamptz | yes   | single-use                          |

Live at launch (password reset works day one). Reset consumption bumps
`users.password_version` and **ASKS the user whether to log out their other sessions**
(CORRECTED 2026-07-14 — not automatic; the choice is theirs, `auth.md` §3) and writes a
`password_reset` user-audit event (`audit_logs.md` §3).

## 3. Initial signup flows

**Sign-off:** _unsigned_

All three converge on the same final step: create `users` row → create `accounts` row →
create `account_members(owner)` → create session → land in console onboarding.

**That final step is one database transaction.** User, account, membership, and session rows
(plus the `user_sso_connections` row for SSO signups) commit atomically; any failure rolls back
everything. No orphan users, no memberless accounts, no half-signups to reap.

**A. Email + password**

1. `POST /v1/auth/signup` `{email, password, display_name?}`.
2. If the email already exists AND the supplied password is **correct** for that user →
   **just log them in** (2026-07-14): they proved exactly what login proves; a "you already
   have an account" error would be pure friction. Email exists with a wrong/absent password
   → reject ("log in instead", with SSO hints if that email has linked SSO connections).
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

**Unlink rules (REVISED 2026-07-14):**

- An SSO connection can be removed **only if the user has an email/password combo** — not
  merely another SSO provider. (Password is the anchor method.)
- **Password auth can NEVER be removed once added.** No exceptions in v1 (revisit only if
  corporate SSO someday demands passwordless-enforced accounts).
- Together these guarantee ≥1 login method always remains. Disconnecting requires a fresh
  re-auth; every connect/disconnect writes `sso_connected`/`sso_disconnected` audit events
  (`audit_logs.md` §3).

**Login resolution (any time):** try `user_sso_connections` by (provider, sub) for SSO; by email +
password check for passwords. The email-collision rule from §3B applies universally: matching
email alone NEVER logs you into or merges with an existing user — it routes you to log in
with an existing method first, then link.

## 6. API endpoints (delta to api.md §9; served by the dashboard gateway — `microservices.md`)

**Sign-off:** _unsigned_

| Endpoint                             | Method | Purpose                                                      |
| ------------------------------------ | ------ | ------------------------------------------------------------ |
| `/v1/auth/oauth/{provider}/start`    | GET    | Begin OAuth (google, github); `intent=login, signup, link`   |
| `/v1/auth/oauth/{provider}/callback` | GET    | Complete OAuth per flows above                               |
| `/v1/auth/login_methods`             | GET    | ALL login methods: the password entry + every SSO connection |
| `/v1/auth/login_methods/{sso_id}`    | DELETE | Disconnect an SSO connection (§5 rules)                      |
| `/v1/auth/password`                  | PUT    | Set/change password (fresh re-auth; no DELETE — §5)          |
| `/v1/auth/sessions`                  | GET    | Live sessions: device, creation IP, UA (`auth.md` §3)        |
| `/v1/auth/sessions/{id}`             | DELETE | Kill a session (`?others=true` kills all but current)        |
| `/v1/user/audit_logs`                | GET    | The user's own audit trail (`audit_logs.md` §4)              |
| `/v1/account/audit_logs`             | GET    | Whole-account audit trail — admin/owner only                 |
| `/v1/account/transfer_ownership`     | POST   | Owner-only; target must be a member (`auth.md` §4)           |
| `/v1/invitations/{code-or-token}`    | GET    | Resolve invite for the join page                             |
| `/v1/invitations/{id}/accept`        | POST   | Accept (authenticated)                                       |

**`sso_connections` → `login_methods` (RENAMED 2026-07-14):** "SSO connections" discarded
email/password auth. Decision: ONE inclusive resource — `GET /v1/auth/login_methods` returns
the password entry (`{kind: "password", enabled, created_at}` — never deletable, so it
exposes no id) alongside the SSO connections (each with its `sso_...` id, provider,
provider_email, provider_name, last_used_at). DELETE takes an `sso_...` id, so password
non-removability is enforced by construction — there is nothing to address. Alternative
considered: keeping `/v1/auth/sso_connections` + a separate password-status endpoint —
rejected; the settings page wants one list, and two endpoints invite drift.
