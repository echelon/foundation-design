# Accounts, Identity & Auth

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Peeled from `schema.md` §1; this doc now owns all
account/identity tables plus the signup, invite, and identity-linking flows. SSO providers:
**Google and GitHub** (ideal pair; more can be added later — the `user_identities` table is
provider-agnostic).

---

## 1. Model overview

**Sign-off:** _unsigned_

- A **user** is a human with 1+ **login identities**: an email/password credential and/or
  linked SSO identities (Google, GitHub). Password is optional (SSO-only users exist).
- An **account** is the billing/team entity. Sign-up creates user + account with that user as
  admin. All resources hang off `account_id`.
- Email is the join key across identities: one user row per human, however many ways they
  log in.
- No email verification at launch for email/password signup (decided) — but the column and
  token table exist so enabling it later is a flag, not a migration.

## 2. Tables

**Sign-off:** _unsigned_

### accounts  (`acct_`)

| column          | type        | notes                                        |
| --------------- | ----------- | -------------------------------------------- |
| id              | text PK     | `acct_...`                                   |
| name            | text        | display name                                 |
| slug            | text UNIQUE | for console URLs                             |
| billing_email   | text        |                                              |
| api_version     | date        | Stripe-style pin, set at first API use       |
| status          | text        | `active \| suspended \| closed`              |
| created_at      | timestamptz |                                              |
| updated_at      | timestamptz |                                              |
| deleted_at      | timestamptz | null; soft delete                            |

### users  (`user_`)

| column            | type        | notes                                              |
| ----------------- | ----------- | --------------------------------------------------- |
| id                | text PK     | `user_...`                                          |
| email             | text UNIQUE | canonical, lower-indexed                            |
| email_verified_at | timestamptz | null; set by verification OR by trusted SSO         |
| password_hash     | text        | **null for SSO-only users**; argon2id when set      |
| display_name      | text        |                                                     |
| avatar_url        | text        | null; from SSO profile when available               |
| totp_secret       | text        | null; encrypted at rest (true MFA, later)           |
| created_at        | timestamptz |                                                     |
| updated_at        | timestamptz |                                                     |
| deleted_at        | timestamptz | soft delete                                         |

### user_identities  (`ident_`) — NEW: SSO links

| column            | type        | notes                                              |
| ----------------- | ----------- | --------------------------------------------------- |
| id                | text PK     | `ident_...`                                         |
| user_id           | text FK     | → users                                             |
| provider          | text        | `google \| github`                                  |
| provider_user_id  | text        | Google `sub` / GitHub numeric id — the STABLE key (never the email) |
| provider_email    | text        | email as reported by the provider at link time      |
| provider_login    | text        | null; GitHub username etc., display only            |
| created_at        | timestamptz |                                                     |
| last_used_at      | timestamptz |                                                     |
|                   |             | UNIQUE (provider, provider_user_id)                 |
|                   |             | UNIQUE (user_id, provider) — one identity per provider per user |

No OAuth access/refresh tokens stored — we use OAuth strictly for authentication (profile +
email scopes), not to act on the user's behalf. Nothing to leak.

### account_members  (`mem_`)

| column      | type        | notes                                  |
| ----------- | ----------- | -------------------------------------- |
| id          | text PK     | `mem_...`                              |
| account_id  | text FK     | → accounts                             |
| user_id     | text FK     | → users                                |
| role        | text        | `admin \| member`                      |
| created_at  | timestamptz |                                        |
|             |             | UNIQUE (account_id, user_id)           |

### invitations  (`inv_`) — now supports code AND magic link

| column       | type        | notes                                             |
| ------------ | ----------- | -------------------------------------------------- |
| id           | text PK     | `inv_...`                                          |
| account_id   | text FK     | → accounts                                         |
| email        | text        | invitee (invite is bound to this email)            |
| role         | text        | `admin \| member`                                  |
| code         | text UNIQUE | short human-typeable code (e.g. `7F3K-QT9M`)       |
| token_hash   | text        | sha256 of the magic-link token (long, unguessable) |
| invited_by_user_id   | text FK     | → users                                            |
| expires_at   | timestamptz | default 7 days                                     |
| accepted_at  | timestamptz | null until accepted                                |
| accepted_by_user_id  | text FK     | → users, null                                      |
| revoked_at   | timestamptz | null                                               |
| created_at   | timestamptz |                                                    |

Code and link are two doors to the same invitation: the email contains the magic link; the
console shows the code for copy/paste into "join with code."

### sessions  (`sess_`)

| column      | type        | notes                              |
| ----------- | ----------- | ---------------------------------- |
| id          | text PK     | `sess_...`                         |
| user_id     | text FK     | → users                            |
| token_hash  | text        | sha256 of cookie value             |
| ip          | inet        |                                    |
| user_agent  | text        |                                    |
| expires_at  | timestamptz |                                    |
| revoked_at  | timestamptz | null                               |
| created_at  | timestamptz |                                    |

### api_keys  (`key_`; secret `foundation_api_...`, shown once)

| column       | type        | notes                                              |
| ------------ | ----------- | -------------------------------------------------- |
| id           | text PK     | `key_...`                                          |
| account_id   | text FK     | → accounts                                         |
| created_by   | text FK     | → users                                            |
| title        | text        | **immutable after creation**                       |
| description  | text        | null; editable                                     |
| key_prefix   | text        | `foundation_api_` + first 6 payload chars          |
| key_hash     | text        | sha256; secret never stored                        |
| last_used_at | timestamptz | throttled updates                                  |
| expires_at   | timestamptz | null = no expiry                                   |
| revoked_at   | timestamptz | null                                               |
| created_at   | timestamptz |                                                    |

### email_verifications / password_resets  (`everif_` / `pwreset_`)

| column      | type        | notes                          |
| ----------- | ----------- | ------------------------------ |
| id          | text PK     |                                |
| user_id     | text FK     | → users                        |
| token_hash  | text        |                                |
| expires_at  | timestamptz |                                |
| used_at     | timestamptz | null                           |
| created_at  | timestamptz |                                |

## 3. Initial signup flows

**Sign-off:** _unsigned_

All three converge on the same final step: create `users` row → create `accounts` row →
create `account_members(admin)` → create session → land in console onboarding.

**A. Email + password**

1. `POST /v1/auth/signup` `{email, password, display_name?}`.
2. Reject if email already exists (→ "log in instead", with SSO hints if that email has
   linked identities).
3. Create user (`password_hash` set, `email_verified_at` NULL — no verification at launch;
   when we enable it later, unverified users get a nag + eventual feature gates, not a
   hard signup block).

**B. Google**

1. `GET /v1/auth/oauth/google/start` → redirect to Google (scopes: openid, email, profile;
   PKCE; `state` cookie).
2. Callback: verify `state`, exchange code, read `sub` + email + `email_verified`.
3. Lookup `user_identities(google, sub)`:
   - exists → log in (update `last_used_at`).
   - not found, email matches an existing user → **do NOT silently merge**: show "an account
     with this email exists — log in to link" (flow D2 below handles the link). Prevents
     account takeover via a Google account created later for the same address.
   - genuinely new → create user (`password_hash` NULL, `email_verified_at` = now, since
     Google asserts the email) + identity row + account.

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
     the mismatch is recorded on the invitation (`accepted_by` tells the truth; admins see
     "invited x@a, joined as y@b" in the audit log).
   - **GitHub**: same as Google.
2. **Not signed in, user exists for that email** → login screen (any method they have) →
   accept.
3. **Already signed in** → confirmation screen ("join {account} as {role}?") → accept.
   Signed in as a different email than invited → show both, let them proceed or switch user.

Invites never create accounts — only memberships. A user can belong to many accounts; the
console gets an account switcher from day one.

## 5. Adding login methods (identity linking)

**Sign-off:** _unsigned_

All linking happens **while authenticated**, from console → Security settings. The three
flows you listed:

**1. Email/password user adds Google or GitHub**

Settings → "Connect Google" → OAuth round-trip (with a `link` intent in `state`, bound to the
current session) → on callback, attach `user_identities` row to the CURRENT user.
- If that Google/GitHub identity is already linked to a *different* user → hard error
  ("already connected to another account") — identities are never moved implicitly.
- Provider email ≠ user email → allowed (people have many emails); shown in settings.

**2. SSO-only user adds a password**

Settings → "Set password" → since SSO-only users have no current password to confirm, require
a **fresh SSO re-auth** (re-run the OAuth flow, max age enforced) → set `password_hash`.
Email/password login now works alongside SSO.

**3. SSO user adds another SSO provider**

Same as flow 1 — Gmail-first user connects GitHub (or vice versa). The identity list in
settings shows all connected methods with `last_used_at` and a disconnect button.

**Unlink rule:** a user must always retain ≥1 login method — the last remaining identity (or
the password, if it's the only method) cannot be removed. Disconnecting a method requires a
fresh re-auth with any remaining method.

**Login resolution (any time):** try `user_identities` by (provider, sub) for SSO; by email +
password check for passwords. The email-collision rule from §3B applies universally: matching
email alone NEVER logs you into or merges with an existing user — it routes you to log in
with an existing method first, then link.

## 6. API endpoints (delta to api.md §9)

**Sign-off:** _unsigned_

| Endpoint                                | Method | Purpose                                    |
| --------------------------------------- | ------ | ------------------------------------------- |
| `/v1/auth/oauth/{provider}/start`       | GET    | Begin OAuth (google \| github); `intent=login\|signup\|link` |
| `/v1/auth/oauth/{provider}/callback`    | GET    | Complete OAuth per flows above              |
| `/v1/auth/identities`                   | GET    | List connected login methods                |
| `/v1/auth/identities/{id}`              | DELETE | Unlink (guard: ≥1 method must remain)       |
| `/v1/auth/password`                     | PUT    | Set/change password (fresh re-auth required)|
| `/v1/invitations/{code-or-token}`       | GET    | Resolve invite for the join page            |
| `/v1/invitations/{id}/accept`           | POST   | Accept (authenticated)                      |
