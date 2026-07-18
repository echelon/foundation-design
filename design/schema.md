# Database Schema

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Postgres. Siblings: `architecture.md`, `api_overview.md`, `api_misc.md`,
`models.md`, `enums.md`, `jobs.md`, `files.md`, `logging.md`, `end_users.md`, `rankings.md`.

This doc owns only the tables no sibling claims. Every table defined in a sibling doc appears
here **only** as a pointer (§7) — never duplicated.

Conventions:

- Every `id` is `TEXT PRIMARY KEY`, format `{prefix}_{26-char Crockford ULID}`, time-ordered.
  The prefix is listed with each table. Every table opens with
  `sid BIGINT GENERATED ALWAYS AS IDENTITY` (never exposed, never joined on,
  `ORDER BY` only), listed explicitly in every table here and in sibling docs, with `id`
  declared right after it. Full spec + decisions: `ids.md`.
- All timestamps are `timestamptz`. `created_at` defaults to `now()` and always sorts above
  the other timestamp columns in a declaration. Money is `BIGINT`
  micro-USD unless noted (`_nanos` = nano-USD unit rates).
- **No catalog tables.** Models, modalities, schemas, routes, prices live in Rust
  (`crates/core/catalog`). Jobs snapshot pricing at enqueue. Provider credentials are
  deployment config, not rows.
- **No native Postgres enums (DECIDED 2026-07-05).** Enum-ish columns are `TEXT` + CHECK
  constraint; the closed sets are owned by `crates/core/enums` and the CHECK lists are
  generated from Rust.
- High-volume log tables live in `logging.md`; public-rankings rollup tables live in
  `rankings.md`; the pointer index is §7.
- **`FK` in these docs is a logical reference — no actual FK constraints, no triggers**
  (DECIDED 2026-07-17); integrity lives in code. Full DB conventions:
  `conventions_database.md`.

---

## 1. Identity & access

**Sign-off:** _unsigned_

**Moved to `accounts.md` (2026-07-05):** accounts, users, **user_sso_connections** (new — SSO:
Google + GitHub), account_members, invitations (now code + magic link), sessions, api_keys,
email_verifications, password_resets — plus the signup, invite, and identity-linking flows.
What remains in this section:

### end_users  (`enduser_`) — our customers' customers (DECIDED 2026-07-05; see `end_users.md`)

**Sign-off:** _unsigned_

| column        | type        | notes                                           |
| ------------- | ----------- | ----------------------------------------------- |
| sid           | bigint      | identity; never exposed (ids.md §4)             |
| id            | text PK     | `enduser_...`                                   |
| account_id    | text FK     | → accounts                                      |
| external_id   | text        | the customer's own id for this user, ≤256 chars |
| first_seen_at | timestamptz |                                                 |
| last_seen_at  | timestamptz | throttled updates                               |
| metadata      | jsonb       | customer-supplied, opaque to us                 |
|               |             | UNIQUE (account_id, external_id)                |

Always optional on requests; auto-upserted on first sight. Never PII by our instruction —
customers send their own stable IDs or hashes.

---

## 2. Jobs & artifacts

**Sign-off:** _unsigned_

**Moved to `jobs.md` (2026-07-05):** jobs, job_events — plus the status machine
(internal + external state tables), the jobs API endpoints, and the retry/cancel policy. Jobs
are the universal primitive (every generation — API, dashboard, MCP, LLM — is a job row);
the full design lives there.

**Moved to `files.md` (2026-07-14):** files (the storage primitive — uploads AND generated
bytes) and outputs — the two-table storage/provenance split. Nothing remains in this
section.

---

## 3. Queue

**Sign-off:** _unsigned_

### queue_tasks  (`task_`)

**Sign-off:** _unsigned_

| column       | type        | notes                                                                                                                             |
| ------------ | ----------- | --------------------------------------------------------------------------------------------------------------------------------- |
| sid          | bigint      | identity; never exposed (ids.md §4)                                                                                               |
| id           | text PK     | `task_...`                                                                                                                        |
| task_type    | text        | `dispatch_job, poll_provider, ingest_outputs, deliver_webhook, settle_refund, auto_refill, rollup_usage, process_stripe_event, …` |
| payload      | jsonb       | usually `{ "job_id": ... }`                                                                                                       |
| created_at   | timestamptz |                                                                                                                                   |
| run_at       | timestamptz | scheduling + backoff; runnable at-or-after this time                                                                              |
| priority     | int         | lower = sooner                                                                                                                    |
| attempts     | int         |                                                                                                                                   |
| max_attempts | int         |                                                                                                                                   |
| locked_by    | text        | worker identity; null                                                                                                             |
| locked_at    | timestamptz | visibility timeout base; null                                                                                                     |
| status       | text        | `pending, running, done, dead`                                                                                                    |
| last_error   | text        | null                                                                                                                              |

Worker loop: `SELECT ... WHERE status='pending' AND run_at <= now() ORDER BY priority,
run_at FOR UPDATE SKIP LOCKED LIMIT n`. Lowered fillfactor + aggressive autovacuum.

**Visibility-timeout reclaim (ADDED 2026-07-14** — `locked_at` was named "visibility timeout
base" with no reaper specified): a sweeper task runs every ~30s:
`UPDATE queue_tasks SET status='pending', locked_by=NULL, locked_at=NULL,
attempts=attempts+1 WHERE status='running' AND locked_at < now() - {visibility_timeout}`
(timeout per task_type; dispatch ~2 min, ingest ~10 min). Rows reaching `max_attempts` go
`dead` + alarm instead. Without this, any worker crash/deploy kill strands its in-flight
tasks forever and the charged job silently expires. Task handlers must therefore be
idempotent under redelivery (dispatch checks `provider_job_id` before re-submitting
upstream).

---

## 4. Billing

**Sign-off:** _unsigned_

**Moved to `payments.md` (2026-07-05):** stripe_customers, payment_methods, ledger_entries,
account_balances, auto_refill_rules, spend_limits, stripe_events — plus the money model
(prepaid dollar balance, no "credits"), Stripe Checkout flows, webhook handling, and
auto-refill mechanics. Policy recap only: charge at enqueue by default (metered models
admission-check a worst-case estimate, charge actuals at completion — architecture.md §4),
hard 402, no negative balances.

---

## 5. Security & bookkeeping

**Sign-off:** _unsigned_

The three high-volume log tables (`request_logs`, `provider_calls`, `webhook_deliveries`)
moved to **`logging.md`** (2026-07-05), which owns the logs product: truncation
(`is_truncated`), redaction, partitioning, downsampling. What remains here:

### webhook_secrets  (`whs_` row id; the secret itself is `foundation_whsec_...`)

**Sign-off:** _unsigned_

| column     | type        | notes                                     |
| ---------- | ----------- | ----------------------------------------- |
| sid        | bigint      | identity; never exposed (ids.md §4)       |
| id         | text PK     | `whs_...`                                 |
| account_id | text FK     | → accounts                                |
| secret     | text        | `foundation_whsec_...`, encrypted at rest |
| created_at | timestamptz |                                           |
| revoked_at | timestamptz | null; rotation = two active at once       |

### audit_logs — moved to `audit_logs.md` (2026-07-14)

Split into TWO tables: `user_audit_logs` (`uaudit_`, identity events, user-visible) and
`account_audit_logs` (`aaudit_`, resource/membership events, admin-visible) — event enum,
dual-write matrix, and API in **`audit_logs.md`**.

### idempotency_keys  (`idem_`)

**Sign-off:** _unsigned_

| column            | type        | notes                               |
| ----------------- | ----------- | ----------------------------------- |
| sid               | bigint      | identity; never exposed (ids.md §4) |
| id                | text PK     | `idem_...`                          |
| account_id        | text FK     | → accounts                          |
| key               | text        | UNIQUE (account_id, key)            |
| request_hash      | text        | reject reuse with changed body      |
| response_snapshot | jsonb       | replayed on repeat                  |
| created_at        | timestamptz |                                     |
| expires_at        | timestamptz | 24h window                          |

---

## 6. The admission transaction

**Sign-off:** _unsigned_

Moved to `payments.md` §6 (with the refund mirror and LLM exception).

---

## 7. All tables — the complete index

**Sign-off:** _unsigned_

Every table in the system, wherever it's defined — sibling doc or this one. This is THE
outline; a table missing here doesn't exist.

| Domain            | Table                    | Prefix      | Defined in                    |
| ----------------- | ------------------------ | ----------- | ----------------------------- |
| Identity & access | accounts                 | `acct_`     | `accounts.md` §2              |
|                   | users                    | `user_`     | `accounts.md` §2              |
|                   | user_sso_connections     | `sso_`      | `accounts.md` §2              |
|                   | account_members          | `mem_`      | `accounts.md` §2              |
|                   | invitations              | `inv_`      | `accounts.md` §2              |
|                   | sessions                 | `sess_`     | `accounts.md` §2              |
|                   | api_keys                 | `key_`      | `accounts.md` §2              |
|                   | email_verifications      | `emailver_` | `accounts.md` §2              |
|                   | password_resets          | `pwreset_`  | `accounts.md` §2              |
|                   | email_changes            | `emailchg_` | `accounts.md` §2              |
|                   | user_avatars             | `avatar_`   | `accounts.md` §2              |
|                   | user_avatar_files        | `uaf_`      | `accounts.md` §2              |
|                   | user_sso_disconnections  | `ssodisc_`  | `accounts.md` §2              |
|                   | end_users                | `enduser_`  | **this doc** §1               |
| Jobs              | jobs                     | `job_`      | `jobs.md` §4                  |
|                   | job_events               | `jevt_`     | `jobs.md` §4                  |
| Files & outputs   | files                    | `file_`     | `files.md` §2                 |
|                   | outputs                  | `out_`      | `files.md` §2                 |
| Queue             | queue_tasks              | `task_`     | **this doc** §3               |
| Money             | stripe_customers         | `cust_`     | `payments.md` §2              |
|                   | payment_methods          | `pm_`       | `payments.md` §2              |
|                   | ledger_entries           | `txn_`      | `payments.md` §2              |
|                   | account_balances         | —           | `payments.md` §2              |
|                   | auto_refill_rules        | `refill_`   | `payments.md` §2              |
|                   | spend_limits             | `limit_`    | `payments.md` §2              |
|                   | stripe_events            | `sevt_`     | `payments.md` §2              |
| Logs              | request_logs             | `rlog_`     | `logging.md` §3               |
|                   | provider_calls           | `pcall_`    | `logging.md` §3               |
|                   | webhook_deliveries       | `whd_`      | `logging.md` §3               |
| Security          | webhook_secrets          | `whs_`      | **this doc** §5               |
|                   | idempotency_keys         | `idem_`     | **this doc** §5               |
| Audit             | user_audit_logs          | `uaudit_`   | `audit_logs.md` §2            |
|                   | account_audit_logs       | `aaudit_`   | `audit_logs.md` §2            |
|                   | impersonation_audit_logs | `impaudit_` | `accounts.md` §2 (staff-only) |
| Rollups           | usage_rollups            | `roll_`     | `rankings.md` §2              |
| Future (reserved) | end_user_limits          | `eulimit_`  | `end_users.md` (not built)    |
