# Auth (authentication & authorization)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Owns the session-token design, login flows,
session management, and the authorization model. The account/user/membership **tables** and
the signup/invite/linking **flows** live in `accounts.md` (cross-referenced throughout);
audit events in `audit_logs.md`; API-key auth for the product API in `api_overview.md` §1 and
`microservices.md`.

---

## 1. Session tokens: a signed JWT, online-checked (DECIDED 2026-07-14)

**Sign-off:** _unsigned_

bt's question — "what do we need in `sessions.token_hash`? Can't we just HMAC the session id
or store it in a JWT?" — answer: **yes; `token_hash` is dropped.** The cookie is a compact
**signed JWT** used as a *rollup of state*, not as a stateless bearer:

```
cookie = JWT{ sid: "sess_01j0...", uid: "user_01j0...", sv: 7, iat, exp }   (EdDSA, kid header)
```

- **Signature replaces the random token.** With an unforgeable signature there is nothing
  secret to hash into the DB: the `sessions` row is looked up by `sid`, and authenticity
  comes from the signature. (The old design stored `token_hash = sha256(random cookie)` —
  strictly more moving parts for the same guarantee.)
- **Still revocable — every request is online-checked.** Unlike ordinary JWTs, the JWT here
  is NOT trusted alone: middleware verifies the signature, then loads `sessions[sid]`
  (Redis read-through keyed by session id, Postgres the source of truth). Revoked or
  expired row → 401 immediately. Minted JWTs are therefore fully revocable.
- **`sv` = the user's `session_version`** — the *enforced* logout vector clock on the users
  row (`accounts.md` §2). If `jwt.sv != users.session_version` the session is treated as
  revoked: this is how "log out my other/all sessions" works in O(1) without touching each
  session row. **Two separate clocks (REVISED 2026-07-17):** `password_version` records
  password changes but is **no longer in the cookie and no longer forces logout** — changing a
  password does NOT end other sessions unless the user opts in, and that opt-in bumps
  `session_version`, not the password clock. A password change and a session purge are now
  independent choices (§3).
- Impersonation is modeled on the `sessions` row (`impersonator_user_id`, §6), not as a JWT
  claim; the rule stands regardless: claims are cached hints, the DB check is the authority.
- Signing keys live in config with `kid` rotation: new logins sign with the new key, old
  keys stay in the verify set until their sessions age out.
- Cookie attributes: `HttpOnly; Secure; SameSite=Lax`; CSRF defense for cookie-authed
  mutating requests per `microservices.md`.

## 2. Login flows

**Sign-off:** _unsigned_

All flows end the same way: create a `sessions` row + mint the JWT cookie.

| Flow                | Mechanics                                                                                                                                                                                             |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Email + password    | `POST /v1/auth/login` — resolve user by canonicalized email, argon2id verify                                                                                                                          |
| SSO (Google/GitHub) | OAuth per `accounts.md` §3B/C — resolve on `(provider, provider_user_id)`, never on email                                                                                                             |
| Signup              | `accounts.md` §3 — including the **signup-as-login rule**: signup with an existing email + the CORRECT password just logs you in (no error; it's the same person proving the same thing login proves) |
| Invite              | `accounts.md` §4                                                                                                                                                                                      |

Failed logins are rate-limited per (email, IP) and never reveal whether the email exists.
Linking/unlinking methods: `accounts.md` §5 (fresh re-auth required; password can never be
removed once set; SSO removable only when a password exists).

## 3. Session management

**Sign-off:** _unsigned_

Users see and kill their live sessions from Settings → Security:

- `GET /v1/auth/sessions` — every non-revoked, non-expired session: created_at, creation
  `ip`, `user_agent`, parsed `device`, last-seen, and a "this device" marker. (Staff
  impersonation sessions are filtered out — they are internal artifacts, §6.)
- `DELETE /v1/auth/sessions/{id}` — kill one (writes `revoked_at`, deletes the cache entry;
  `session_revoked` user-audit event).
- `DELETE /v1/auth/sessions?others=true` — "log out all other sessions": bumps
  `users.session_version` AND writes `revoked_at` on every session except the current, then
  re-mints the current cookie at the new `sv`.

**A session dies exactly three ways** (more can be added later — e.g. a future admin-set
expiry bump):

1. **Expiry.** `expires_at` has passed — pure row state; the online check returns 401.
   Nothing writes `expires_at` after creation today except impersonation's 1h cap (§6); a
   user-driven extension is a possible future.
2. **Manual single kill.** The user clicks "kill" next to a device in the sessions list →
   `DELETE /v1/auth/sessions/{id}` writes that row's `revoked_at` and drops its cache entry.
   Only that one session dies.
3. **Kill-all — the `session_version` bump.** "Log out all sessions" bumps
   `users.session_version` (every outstanding JWT now carries a stale `sv` → revoked in O(1),
   no per-row read) **and** writes `revoked_at` on the rows so durable state and cache agree.
   The two together cover the fast path (stale `sv` rejected outright) and the slow path
   (`revoked_at` for anything that still re-reads its row).

**Password reset/change does NOT force-revoke other sessions** — the user is ASKED ("log out
my other devices?"): yes → path 3 minus the current session (bump `session_version`); no →
sessions live on. `password_version` is bumped either way as a record, but it is not the clock
that ends sessions (REVISED 2026-07-17 — that is `session_version`).

## 4. Authorization: roles & the account switcher

**Sign-off:** _unsigned_

A user belongs to **zero or more accounts** with a role per membership; the console puts the
account id in the URL (Stripe-style — `/{acct_id}/…`) and every dashboard request authorizes
as (session user, URL account): membership row must exist and its role must permit the
action. Any user can create additional accounts (becoming their owner).

| Capability                                    | owner | admin | member |
| --------------------------------------------- | ----- | ----- | ------ |
| Generate, view own jobs/logs                  | ✓     | ✓     | ✓      |
| View account jobs/spend/usage                 | ✓     | ✓     | ✓      |
| API keys, webhooks, spend limits, auto-refill | ✓     | ✓     |        |
| Billing profile, top-ups, cards               | ✓     | ✓     |        |
| Members: invite, role change, remove          | ✓     | ✓     |        |
| Account audit log                             | ✓     | ✓     |        |
| Transfer ownership, close account             | ✓     |       |        |

Rules (enforced in `services/auth`, all audited — `audit_logs.md` §3):

- **Exactly one owner per account** (partial unique index); the account **creator is the
  owner**; invitations can grant `admin` or `member`, never `owner`.
- **The owner cannot remove themselves** (or be removed, or demote themselves) — transfer
  ownership first. This also guarantees an account is never left admin-less.
- Ownership transfer: `POST /v1/account/transfer_ownership {user_id}` — owner-only, target
  must already be a member, fresh re-auth required; atomic swap (target → owner, old owner
  → admin); `ownership_transferred` audit events on the account and both users.
- Admins cannot touch the owner's membership or role.

## 5. Two front doors

**Sign-off:** _unsigned_

| Credential           | Accepted by                               | Identifies                                 |
| -------------------- | ----------------------------------------- | ------------------------------------------ |
| API key (`Bearer`)   | product API gateway (`microservices.md`)  | an account                                 |
| Session cookie (JWT) | dashboard gateway + product API (sandbox) | a user (+ URL account for dashboard authz) |
| MCP transport        | product API gateway, API-key-authed       | an account                                 |

Product-API requests made with a cookie record `source = dashboard` on the job and bill
identically (`architecture.md` §4). Auth middleware is one tower layer per gateway; both
resolve to the same internal `AuthContext { account, user?, source, api_key? }`.

## 6. Support impersonation (staff only)

**Sign-off:** _unsigned_

Support/ops staff log into a customer's account to debug — a **staff impersonation session**.
It is an ordinary `sessions` row (`accounts.md` §2) with three differences:

- **`impersonator_user_id` is set** to the staff user. Presence of that column IS the
  impersonation flag; it is written at creation and never mutated. `user_id` stays the
  customer being acted as, so every downstream authorization check and audit row already
  attributes actions to the right account — with the impersonator permanently recorded
  alongside. Impersonation is NOT a JWT claim (§1): the row is the authority.
- **Short-lived: `expires_at` ≤ `created_at + 1h`**, capped at mint, no extension. An
  impersonation session cannot outlive the debugging window; staff re-initiate if they need
  longer.
- **Every start and end writes `impersonation_audit_logs`** (`accounts.md` §2) — an internal,
  staff-only trail (`impaudit_...`; action `started`/`ended`; `reason` captured at start). It
  is NOT a customer/org audit log: it never appears in `user_audit_logs` or
  `account_audit_logs` (`audit_logs.md`), and no customer-facing endpoint reads it.

Guardrails: minted only through internal tooling (never a customer flow), gated on a support
role we never grant customers; the console shows a persistent "You are impersonating {user}"
banner; the session is filtered out of the customer's own sessions list (§3). Killing it is
the same as any session (paths 2/3 of §3), and it self-expires within the hour regardless.
