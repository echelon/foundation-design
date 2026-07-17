# Audit Logs

**Sign-off:** _unsigned_

<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). Replaces the single `audit_logs` table that lived
in `schema.md` §5 with **two** audit logs: user-scoped and account-scoped. Siblings:
`accounts.md`, `auth.md`, `schema.md`. These are the *security/change* trails — the
request/debug logs product is `logging.md`, a different thing. **REVISED 2026-07-17:**
identity events are no longer fanned out to every member account's log — they're written once
and tagged `is_org_visible`, and orgs read a member's org-visible subset through a new
per-user endpoint (§4). Billing audit events added (§3).

---

## 1. Two logs, deliberately

**Sign-off:** _unsigned_

| Log                  | Scope                                                                              | Who reads it                                                                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `user_audit_logs`    | Events about a **user's identity**: password, email, SSO connections, sessions     | The user themself (`GET /v1/user/audit_logs`); a shared org for the **org-visible** subset (`GET /v1/account/users/{user_id}/audit_logs`, §4) |
| `account_audit_logs` | Events about an **account's resources**: membership, keys, billing profile, limits | Account admins/owner (`GET /v1/account/audit_logs`)                                                                                           |

Why two: a user belongs to zero-or-more accounts, so identity events have no single home
account — and org admins legitimately need to see security-relevant events about their
members without seeing the member's other accounts.

**Single-write + `is_org_visible` (REVISED 2026-07-17 — replaces the old fan-out):** an
identity event is written **once** to the user's log, tagged `is_org_visible` from the event's
category (§3). We no longer duplicate it into the account log of every member account. An org
admin who wants "did our member change their password?" reads the member's org-visible events
through the per-user endpoint (§4), authorized against **current** membership. This drops the
previous extra row-per-member-account-per-event and its event-time membership snapshot.

Account-resource events go to the account log only, plus the affected user's log when the
event is about a specific member (so the member sees "my role changed" in their own trail).

Both tables are **append-only, kept forever** (low volume; these answer "who did what,
when" months later).

**Read-time visibility tradeoff (OPEN):** org visibility is resolved at *read* time against
*current* membership, so an org loses visibility into a member's past identity events once that
member leaves — the old fan-out captured member accounts at event time. If retaining
post-departure history matters, add a `user_audit_log_shares(user_audit_log_id, account_id)`
join written in the event transaction; deliberately deferred, not built.

## 2. Tables

**Sign-off:** _unsigned_

### user_audit_logs  (`uaudit_`)

**Sign-off:** _unsigned_

| column         | type        | null? | notes                                                                                 |
| -------------- | ----------- | ----- | ------------------------------------------------------------------------------------- |
| sid            | bigint      | no    | identity; never exposed (ids.md §4)                                                   |
| id             | text PK     | no    | `uaudit_...`                                                                          |
| user_id        | text FK     | no    | → users; whose log this is                                                            |
| actor_user_id  | text FK     | yes   | → users; who did it (usually = user_id; null = system)                                |
| event          | text        | no    | enum `AuditEvent` (§3)                                                                |
| target_type    | text        | yes   | enum `AuditTargetType`: singular entity slug (`session, sso_connection, …`)           |
| target_id      | text        | yes   | the affected row's id (self-describing prefix!)                                       |
| data           | jsonb       | yes   | event-specific detail (e.g. `{provider: "google"}`)                                   |
| is_org_visible | bool        | no    | mirrors `AuditEvent::is_org_visible()` at write; the org read path (§4) filters on it |
| ip             | inet        | yes   | request origin, when there was a request                                              |
| created_at     | timestamptz | no    | date of event                                                                         |

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

Indexes: `(user_id, created_at desc)` / `(account_id, created_at desc)` — the self + account
read paths — plus a partial `(user_id, created_at desc) WHERE is_org_visible` for the per-user
org view (§4).

## 3. `AuditEvent` — the event enum & the write matrix

**Sign-off:** _unsigned_

Hand-written enum in `crates/core/enums` (explicit serde renames, golden tests). The audit
write happens **in the same transaction** as the mutation it records. **U** = a
`user_audit_logs` row; **A** = an `account_audit_logs` row; **Org** = that user-log row is
`is_org_visible` (a shared org may read it via §4). `AuditEvent::is_org_visible(self) -> bool`
is the categorization column — an exhaustive match, like `FailureCode::category()`, consulted
only when a `user_audit_logs` row is written. Identity events are **U-only** now (no fan-out);
their org reach is the `Org` flag, not a duplicated account-log row.

| Event                            | U   | A   | Org | Written when                                                        |
| -------------------------------- | --- | --- | --- | ------------------------------------------------------------------- |
| `password_changed`               | ✓   |     | ✓   | User sets/changes password                                          |
| `password_reset`                 | ✓   |     | ✓   | Reset token consumed                                                |
| `email_changed`                  | ✓   |     | ✓   | Email change (also an `email_changes` row — accounts.md §2)         |
| `email_verified`                 | ✓   |     | ✓   | Verification token validated                                        |
| `sso_connected`                  | ✓   |     | ✓   | SSO provider linked                                                 |
| `sso_disconnected`               | ✓   |     | ✓   | SSO provider unlinked                                               |
| `session_revoked`                | ✓   |     |     | A session killed from the sessions list (auth.md §3) — user-private |
| `account_created`                | ✓   | ✓   |     | Signup or additional-account creation; creator = owner              |
| `member_added`                   | ✓   | ✓   |     | Invitation accepted / membership created (user_id = the new member) |
| `member_role_changed`            | ✓   | ✓   |     | Role change (data: `{from, to}`)                                    |
| `member_removed`                 | ✓   | ✓   |     | Membership revoked or member left                                   |
| `ownership_transferred`          | ✓   | ✓   |     | Owner swap (both old and new owner get user-log rows)               |
| `invitation_sent`                |     | ✓   |     |                                                                     |
| `invitation_revoked`             |     | ✓   |     |                                                                     |
| `invitation_accepted`            |     | ✓   |     | Alongside `member_added`                                            |
| `api_key_created`                |     | ✓   |     |                                                                     |
| `api_key_revoked`                |     | ✓   |     |                                                                     |
| `payment_method_added`           |     | ✓   |     | Card/bank added to the billing profile (payments.md §2)             |
| `payment_method_removed`         |     | ✓   |     | Card/bank removed                                                   |
| `default_payment_method_changed` |     | ✓   |     | Which saved method is the default changed                           |
| `billing_address_changed`        |     | ✓   |     | ANY change to the account's billing address/phone/email fields      |
| `billing_link_added`             |     | ✓   |     | A billing link created (see note)                                   |
| `billing_link_removed`           |     | ✓   |     | A billing link removed (see note)                                   |
| `billing_paused`                 |     | ✓   |     | Account billing paused (see note)                                   |
| `billing_resumed`                |     | ✓   |     | Account billing resumed                                             |
| `spend_limit_changed`            |     | ✓   |     |                                                                     |
| `auto_refill_changed`            |     | ✓   |     |                                                                     |
| `webhook_secret_rotated`         |     | ✓   |     |                                                                     |

The list is additive-forever: new events append to the enum; old rows never reinterpret.
(`AuditTargetType` values are the singular entity slugs, one per `schema.md` §7 entity,
generated from the prefix registry.)

**Billing audit events (added 2026-07-17):** `payment_method_added` / `payment_method_removed`
/ `default_payment_method_changed` cover changing billing info (cards, bank details —
`payments.md` §2); `billing_link_*` and `billing_paused` / `billing_resumed` cover the
billing-link and billing-pause features. **OPEN — bt confirm semantics:** I read *billing
link* as a cross-account billing relationship (one account funding another) and *pause billing*
as voluntarily halting auto-refill/charges on an account. Both underlying features live in
`payments.md` (TBD, parked); only their audit events are reserved here. Rename if the features
mean something else.

## 4. Visibility & API

**Sign-off:** _unsigned_

| Endpoint                                     | Who            | Returns                                                                                                                                |
| -------------------------------------------- | -------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /v1/user/audit_logs`                    | any user       | **their own** `user_audit_logs`, newest first                                                                                          |
| `GET /v1/account/audit_logs`                 | admin or owner | the **entire account's** `account_audit_logs`                                                                                          |
| `GET /v1/account/users/{user_id}/audit_logs` | admin or owner | a member's **org-visible** `user_audit_logs` (`is_org_visible` only) — `403` unless `{user_id}` is a **current** member of the account |

Cursor pagination (`?limit=&after=`), filter params `?event=&user_id=&since=&until=`.

**The per-user org endpoint (added 2026-07-17)** is how an org sees a member's security events
now that identity events aren't fanned out (§1). It fixes `user_id` via the path and
**hard-filters `is_org_visible = true`** — a non-org-visible event (e.g. `session_revoked`) is
never returned even if named in `?event=`. Authorization is re-checked per request against
current membership: an ex-member's `user_id` returns `403`, not their history. Members who are
not admins do NOT see the account log or this per-user view. Rendered in the console under
Settings → Security (user), Account → Audit log (admins), and Account → Members → {member} →
Security (the per-user org view).
