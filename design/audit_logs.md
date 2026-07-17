# Audit Logs

**Sign-off:** bt 2026-07-15

<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Replaces the single `audit_logs` table that lived
in `schema.md` §5 with **two** audit logs: user-scoped and account-scoped. Siblings:
`accounts.md`, `auth.md`, `schema.md`. These are the *security/change* trails — the
request/debug logs product is `logging.md`, a different thing.

---

## 1. Two logs, deliberately

**Sign-off:** bt 2026-07-14

| Log                  | Scope                                                                              | Who reads it                                        |
| -------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------- |
| `user_audit_logs`    | Events about a **user's identity**: password, email, SSO connections, sessions     | The user themself (`GET /v1/user/audit_logs`)       |
| `account_audit_logs` | Events about an **account's resources**: membership, keys, billing profile, limits | Account admins/owner (`GET /v1/account/audit_logs`) |

Why two: a user belongs to zero-or-more accounts, so identity events have no single home
account — and org admins legitimately need to see security-relevant events about their
members without seeing the member's other accounts. **Dual-write rule:** identity events are
written to the user's log AND fanned out to the account log of every account the user is a
member of at event time (the org admin's view of "our member changed their password").
Account-resource events go to the account log only, plus the affected user's log when the
event is about a specific member.

Both tables are **append-only, kept forever** (low volume; these answer "who did what,
when" months later).

## 2. Tables

**Sign-off:** bt 2026-07-14

### user_audit_logs  (`uaudit_`)

**Sign-off:** bt 2026-07-14

| column        | type        | null? | notes                                                                       |
| ------------- | ----------- | ----- | --------------------------------------------------------------------------- |
| sid           | bigint      | no    | identity; never exposed (ids.md §4)                                         |
| id            | text PK     | no    | `uaudit_...`                                                                |
| user_id       | text FK     | no    | → users; whose log this is                                                  |
| actor_user_id | text FK     | yes   | → users; who did it (usually = user_id; null = system)                      |
| event         | text        | no    | enum `AuditEvent` (§3)                                                      |
| target_type   | text        | yes   | enum `AuditTargetType`: singular entity slug (`session, sso_connection, …`) |
| target_id     | text        | yes   | the affected row's id (self-describing prefix!)                             |
| data          | jsonb       | yes   | event-specific detail (e.g. `{provider: "google"}`)                         |
| ip            | inet        | yes   | request origin, when there was a request                                    |
| created_at    | timestamptz | no    | date of event                                                               |

### account_audit_logs  (`aaudit_`)

**Sign-off:** bt 2026-07-14

| column        | type        | null? | notes                                                           |
| ------------- | ----------- | ----- | --------------------------------------------------------------- |
| sid           | bigint      | no    | identity; never exposed (ids.md §4)                             |
| id            | text PK     | no    | `aaudit_...`                                                    |
| account_id    | text FK     | no    | → accounts; whose log this is                                   |
| actor_user_id | text FK     | yes   | → users; who did it (null = system/webhook)                     |
| user_id       | text FK     | yes   | → users; the member the event is ABOUT, when applicable         |
| event         | text        | no    | enum `AuditEvent` (§3)                                          |
| target_type   | text        | yes   | enum `AuditTargetType`                                          |
| target_id     | text        | yes   | the affected row's id                                           |
| data          | jsonb       | yes   | event-specific detail (e.g. old/new role, masked address delta) |
| ip            | inet        | yes   |                                                                 |
| created_at    | timestamptz | no    | date of event                                                   |

Indexes: `(user_id, created_at desc)` / `(account_id, created_at desc)` — the two read paths.

## 3. `AuditEvent` — the event enum, with the dual-write matrix

**Sign-off:** bt 2026-07-14

Hand-written enum in `crates/core/enums` (explicit serde renames, golden tests). The audit
write happens **in the same transaction** as the mutation it records. U = user log,
A = account log(s).

| Event                     | U   | A   | Written when                                                         |
| ------------------------- | --- | --- | -------------------------------------------------------------------- |
| `password_changed`        | ✓   | ✓   | User sets/changes password (fan out to ALL member accounts)          |
| `password_reset`          | ✓   | ✓   | Reset token consumed (fan out)                                       |
| `email_changed`           | ✓   | ✓   | Email change (fan out; also an `email_changes` row — accounts.md §2) |
| `email_verified`          | ✓   | ✓   | Verification token validated (fan out)                               |
| `sso_connected`           | ✓   | ✓   | SSO provider linked (fan out)                                        |
| `sso_disconnected`        | ✓   | ✓   | SSO provider unlinked (fan out)                                      |
| `session_revoked`         | ✓   |     | A session killed from the sessions list (auth.md §3)                 |
| `account_created`         | ✓   | ✓   | Signup or additional-account creation; creator = owner               |
| `member_added`            | ✓   | ✓   | Invitation accepted / membership created (user_id = the new member)  |
| `member_role_changed`     | ✓   | ✓   | Role change (data: `{from, to}`)                                     |
| `member_removed`          | ✓   | ✓   | Membership revoked or member left                                    |
| `ownership_transferred`   | ✓   | ✓   | Owner swap (both old and new owner get user-log rows)                |
| `invitation_sent`         |     | ✓   |                                                                      |
| `invitation_revoked`      |     | ✓   |                                                                      |
| `invitation_accepted`     |     | ✓   | Alongside `member_added`                                             |
| `api_key_created`         |     | ✓   |                                                                      |
| `api_key_revoked`         |     | ✓   |                                                                      |
| `billing_address_changed` |     | ✓   | ANY change to the accounts billing address/phone/email fields        |
| `spend_limit_changed`     |     | ✓   |                                                                      |
| `auto_refill_changed`     |     | ✓   |                                                                      |
| `webhook_secret_rotated`  |     | ✓   |                                                                      |

The list is additive-forever: new events append to the enum; old rows never reinterpret.
(`AuditTargetType` values are the singular entity slugs, one per `schema.md` §7 entity,
generated from the prefix registry.)

## 4. Visibility & API

**Sign-off:** bt 2026-07-14

| Endpoint                     | Who            | Returns                                       |
| ---------------------------- | -------------- | --------------------------------------------- |
| `GET /v1/user/audit_logs`    | any user       | **their own** `user_audit_logs`, newest first |
| `GET /v1/account/audit_logs` | admin or owner | the **entire account's** `account_audit_logs` |

Cursor pagination (`?limit=&after=`), filter params `?event=&user_id=&since=&until=`.
Members who are not admins do NOT see the account log. Rendered in the console under
Settings → Security (user) and Account → Audit log (admins).
