# Auth (authentication & authorization)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Owns the session-token design, login flows,
session management, and the authorization model. The account/user/membership **tables** and
the signup/invite/linking **flows** live in `accounts.md` (cross-referenced throughout);
audit events in `audit_logs.md`; API-key auth for the product API in `api.md` §1 and
`microservices.md`.

---

## 1. Session tokens: a signed JWT, online-checked (DECIDED 2026-07-14)

**Sign-off:** _unsigned_

bt's question — "what do we need in `sessions.token_hash`? Can't we just HMAC the session id
or store it in a JWT?" — answer: **yes; `token_hash` is dropped.** The cookie is a compact
**signed JWT** used as a *rollup of state*, not as a stateless bearer:

```
cookie = JWT{ sid: "sess_01j0...", uid: "user_01j0...", pv: 3, iat, exp }   (EdDSA, kid header)
```

- **Signature replaces the random token.** With an unforgeable signature there is nothing
  secret to hash into the DB: the `sessions` row is looked up by `sid`, and authenticity
  comes from the signature. (The old design stored `token_hash = sha256(random cookie)` —
  strictly more moving parts for the same guarantee.)
- **Still revocable — every request is online-checked.** Unlike ordinary JWTs, the JWT here
  is NOT trusted alone: middleware verifies the signature, then loads `sessions[sid]`
  (Redis read-through keyed by session id, Postgres the source of truth). Revoked or
  expired row → 401 immediately. Minted JWTs are therefore fully revocable.
- **`pv` = the user's `password_version`** (vector clock on the users row, +1 on every
  password set/change/reset — `accounts.md` §2). If `jwt.pv != users.password_version` the
  session is treated as revoked — this is how "log out my other sessions" works in O(1)
  without touching each session row.
- Claims may grow (e.g. impersonation flags for support); the rule stands: claims are
  cached hints, the DB check is the authority.
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

Users can see and kill their live sessions:

- `GET /v1/auth/sessions` — every non-revoked, non-expired session: created_at, creation
  `ip`, `user_agent`, parsed `device`, last-seen, and a "this device" marker.
- `DELETE /v1/auth/sessions/{id}` — kill one (writes `revoked_at`, deletes the cache entry;
  `session_revoked` user-audit event).
- `DELETE /v1/auth/sessions?others=true` — kill everything except the current session.
- **Password reset/change does NOT force-revoke other sessions** — the user is ASKED
  ("log out my other devices?"): yes → bump `password_version` (invalidates every session's
  `pv` claim except the fresh one); no → sessions live on.

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
