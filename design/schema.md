# Database Schema

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Postgres. Siblings: `architecture.md`, `api.md`,
`models.md`, `enums.md`, `logging.md`, `end-users.md`, `rankings.md`.

Conventions:

- Every `id` is `TEXT PRIMARY KEY`, format `{prefix}_{26-char Crockford of 128 bits}`,
  ULID-style time-ordered. The prefix is listed with each table. No numeric IDs anywhere
  (analysis: `../reports/identifiers.md`).
- All timestamps are `timestamptz`. `created_at` defaults to `now()`. Money is `BIGINT`
  micro-USD unless noted (`_nanos` = nano-USD unit rates).
- **No catalog tables.** Models, modalities, schemas, routes, prices live in Rust
  (`crates/core/catalog`). Jobs snapshot pricing at enqueue. Provider credentials are
  deployment config, not rows.
- **No native Postgres enums (DECIDED 2026-07-05).** Enum-ish columns are `TEXT` + CHECK
  constraint; the closed sets are owned by `crates/core/enums` and the CHECK lists are
  generated from Rust.
- High-volume log tables live in `logging.md`; public-rankings rollup tables live in
  `rankings.md`; the pointer index is §7.

---

## 1. Identity & access

**Sign-off:** _unsigned_

**Moved to `accounts.md` (2026-07-05):** accounts, users, **user_identities** (new — SSO:
Google + GitHub), account_members, invitations (now code + magic link), sessions, api_keys,
email_verifications, password_resets — plus the signup, invite, and identity-linking flows.
What remains in this section:

### end_users  (`enduser_`) — our customers' customers (DECIDED 2026-07-05; see `end-users.md`)

| column        | type        | notes                                       |
| ------------- | ----------- | ------------------------------------------- |
| id            | text PK     | `enduser_...`                               |
| account_id    | text FK     | → accounts                                  |
| external_id   | text        | the customer's own id for this user, ≤256 chars |
| first_seen_at | timestamptz |                                             |
| last_seen_at  | timestamptz | throttled updates                           |
| metadata      | jsonb       | customer-supplied, opaque to us             |
|               |             | UNIQUE (account_id, external_id)            |

Always optional on requests; auto-upserted on first sight. Never PII by our instruction —
customers send their own stable IDs or hashes.

---

## 2. Jobs & artifacts

**Sign-off:** _unsigned_

### jobs  (`job_`)

| column               | type        | notes                                                        |
| -------------------- | ----------- | ------------------------------------------------------------ |
| id                   | text PK     | `job_...`                                                     |
| account_id           | text FK     | → accounts                                                    |
| api_key_id           | text FK     | → api_keys; **null = dashboard session**                      |
| user_id              | text FK     | → users; the human, when known                                |
| end_user_id          | text FK     | → end_users; null — the customer's customer, when supplied    |
| source               | text        | `api \| dashboard \| mcp`                                     |
| media_type           | text        | `image \| video \| audio \| mesh \| splat \| text`            |
| model                | text        | canonical catalog slug, e.g. `kling_video_v3_pro`             |
| modality             | text        | e.g. `image_to_video` (resolved, not necessarily requested)   |
| api_shape            | text        | `granular \| unified \| any \| chat`                          |
| api_version          | date        | version the request came in on                                |
| idempotency_key      | text        | null; UNIQUE (account_id, idempotency_key)                    |
| request_params       | jsonb       | canonicalized post-coercion params                            |
| param_adjustments    | jsonb       | what the Any layer pruned/coerced                             |
| status               | text        | `queued \| dispatched \| running \| succeeded \| failed \| canceled \| expired` |
| failure_code         | text        | null; from the failure-code enum (`api.md` §2)                |
| failure_message      | text        | null; human-readable                                          |
| provider             | text        | catalog provider slug, e.g. `bytedance`                       |
| provider_account     | text        | credential pool name from config, e.g. `byteplus_ultra`       |
| provider_job_id      | text        | upstream's task/request id; indexed                           |
| price_unit           | text        | snapshot: `per_video_second`, `per_image`, …                  |
| unit_amount_nanos    | bigint      | snapshot: nano-USD rate charged                               |
| quantity             | numeric     | snapshot: units billed (e.g. seconds)                         |
| charged_micros       | bigint      | fixed at enqueue; **never changes** (LLM jobs: set at completion) |
| refunded_micros      | bigint      | 0 unless failed/canceled → refunded                           |
| webhook_url          | text        | null                                                          |
| webhook_events       | text[]      |                                                               |
| created_at           | timestamptz | = enqueue + charge moment                                     |
| dispatched_at        | timestamptz | null                                                          |
| started_at           | timestamptz | null                                                          |
| completed_at         | timestamptz | null                                                          |
| expires_at           | timestamptz | give-up deadline                                              |

Indexes: (account_id, created_at desc), (account_id, status), (provider, provider_job_id),
(status, expires_at) for the reaper, (account_id, end_user_id, created_at) partial where
end_user_id is not null.

This table IS the customer spend list: job + status + cost in one row.

### job_events  (`jevt_`)

| column      | type        | notes                                                              |
| ----------- | ----------- | ------------------------------------------------------------------ |
| id          | text PK     | `jevt_...`                                                          |
| job_id      | text FK     | → jobs                                                              |
| event_type  | text        | `created \| charged \| dispatched \| provider_accepted \| poll_tick \| provider_completed \| output_ingested \| webhook_sent \| refunded \| failed \| …` |
| data        | jsonb       | small payload                                                       |
| created_at  | timestamptz |                                                                     |

Append-only state-machine trail; merged into the customer debug timeline (`logging.md`).

### job_outputs  (`out_`)

| column           | type        | notes                                        |
| ---------------- | ----------- | -------------------------------------------- |
| id               | text PK     | `out_...`                                     |
| job_id           | text FK     | → jobs                                        |
| output_index     | int         | 0-based within the job                        |
| kind             | text        | `image \| video \| audio \| mesh \| splat \| thumbnail \| text` |
| storage_key      | text        | R2 object key                                 |
| cdn_url          | text        | durable, ours                                 |
| mime             | text        |                                               |
| bytes            | bigint      |                                               |
| width            | int         | null                                          |
| height           | int         | null                                          |
| duration_ms      | int         | null                                          |
| checksum_sha256  | text        |                                               |
| provider_url     | text        | original (expiring) upstream URL, for debugging |
| storage_class    | text        | `hot \| cold` — forever-retention with cold migration |
| created_at       | timestamptz |                                               |

### files  (`file_`) — customer inputs

| column           | type        | notes                                |
| ---------------- | ----------- | ------------------------------------ |
| id               | text PK     | `file_...`                           |
| account_id       | text FK     | → accounts                           |
| purpose          | text        | `input \| avatar \| …`               |
| storage_key      | text        | R2                                   |
| cdn_url          | text        |                                      |
| mime             | text        |                                      |
| bytes            | bigint      |                                      |
| checksum_sha256  | text        |                                      |
| origin           | text        | `upload \| mirrored_url`             |
| origin_url       | text        | null; when mirrored                  |
| created_at       | timestamptz |                                      |

---

## 3. Queue

**Sign-off:** _unsigned_

### queue_tasks  (`task_`)

| column       | type        | notes                                                          |
| ------------ | ----------- | --------------------------------------------------------------- |
| id           | text PK     | `task_...`                                                       |
| task_type    | text        | `dispatch_job \| poll_provider \| ingest_outputs \| deliver_webhook \| settle_refund \| auto_refill \| rollup_usage \| …` |
| payload      | jsonb       | usually `{ "job_id": ... }`                                      |
| run_after    | timestamptz | scheduling + backoff                                             |
| priority     | int         | lower = sooner                                                   |
| attempts     | int         |                                                                  |
| max_attempts | int         |                                                                  |
| locked_by    | text        | worker identity; null                                            |
| locked_at    | timestamptz | visibility timeout base; null                                    |
| status       | text        | `pending \| running \| done \| dead`                             |
| last_error   | text        | null                                                             |
| created_at   | timestamptz |                                                                  |

Worker loop: `SELECT ... WHERE status='pending' AND run_after <= now() ORDER BY priority,
run_after FOR UPDATE SKIP LOCKED LIMIT n`. Lowered fillfactor + aggressive autovacuum.

---

## 4. Billing

**Sign-off:** _unsigned_

**Moved to `payments.md` (2026-07-05):** stripe_customers, payment_methods, ledger_entries,
account_balances, auto_refill_rules, spend_limits, stripe_events — plus the money model
(prepaid dollar balance, no "credits"), Stripe Checkout flows, webhook handling, and
auto-refill mechanics. Policy recap only: charge at enqueue, price fixed for the job's
lifetime, hard 402, no negative balances.

---

## 5. Security & bookkeeping

**Sign-off:** _unsigned_

The three high-volume log tables (`request_logs`, `provider_calls`, `webhook_deliveries`)
moved to **`logging.md`** (2026-07-05), which owns the logs product: truncation
(`is_truncated`), redaction, partitioning, downsampling. What remains here:

### webhook_secrets  (`whs_` row id; the secret itself is `foundation_whsec_...`)

| column      | type        | notes                                  |
| ----------- | ----------- | --------------------------------------- |
| id          | text PK     | `whs_...`                               |
| account_id  | text FK     | → accounts                              |
| secret      | text        | `foundation_whsec_...`, encrypted at rest |
| created_at  | timestamptz |                                         |
| revoked_at  | timestamptz | null; rotation = two active at once     |

### audit_logs  (`audit_`)

| column           | type        | notes                                            |
| ---------------- | ----------- | ------------------------------------------------ |
| id               | text PK     | `audit_...`                                       |
| account_id       | text FK     | → accounts                                        |
| actor_user_id    | text FK     | → users                                           |
| action           | text        | `key.created \| key.revoked \| member.invited \| limit.changed \| …` |
| target_id        | text        | the affected row's id (self-describing prefix!)   |
| data             | jsonb       |                                                   |
| ip               | inet        |                                                   |
| created_at       | timestamptz |                                                   |

### idempotency_keys  (`idem_`)

| column             | type        | notes                          |
| ------------------ | ----------- | ------------------------------ |
| id                 | text PK     | `idem_...`                     |
| account_id         | text FK     | → accounts                     |
| key                | text        | UNIQUE (account_id, key)       |
| request_hash       | text        | reject reuse with changed body |
| response_snapshot  | jsonb       | replayed on repeat             |
| expires_at         | timestamptz | 24h window                     |
| created_at         | timestamptz |                                |

---

## 6. The admission transaction

**Sign-off:** _unsigned_

Moved to `payments.md` §6 (with the refund mirror and LLM exception).

---

## 7. Tables owned by sibling design docs

**Sign-off:** _unsigned_

| Table                    | Prefix      | Defined in      |
| ------------------------ | ----------- | --------------- |
| accounts                 | `acct_`     | `accounts.md`   |
| users                    | `user_`     | `accounts.md`   |
| user_identities          | `ident_`    | `accounts.md`   |
| account_members          | `mem_`      | `accounts.md`   |
| invitations              | `inv_`      | `accounts.md`   |
| sessions                 | `sess_`     | `accounts.md`   |
| api_keys                 | `key_`      | `accounts.md`   |
| email_verifications      | `everif_`   | `accounts.md`   |
| password_resets          | `pwreset_`  | `accounts.md`   |
| stripe_customers         | `cust_`     | `payments.md`   |
| payment_methods          | `pm_`       | `payments.md`   |
| ledger_entries           | `txn_`      | `payments.md`   |
| account_balances         | —           | `payments.md`   |
| auto_refill_rules        | `refill_`   | `payments.md`   |
| spend_limits             | `limit_`    | `payments.md`   |
| stripe_events            | `sevt_`     | `payments.md`   |
| request_logs             | `rlog_`     | `logging.md`    |
| provider_calls           | `pcall_`    | `logging.md`    |
| webhook_deliveries       | `whd_`      | `logging.md`    |
| usage_rollups            | `roll_`     | `rankings.md`   |
| end_user_limits (future) | `eulimit_`  | `end-users.md`  |
