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

### users  (`user_`)

| column            | type        | notes                                      |
| ----------------- | ----------- | ------------------------------------------ |
| id                | text PK     | `user_...`                                 |
| email             | text UNIQUE | citext behavior via lower-index            |
| email_verified_at | timestamptz | null = unverified                          |
| password_hash     | text        | argon2id                                   |
| display_name      | text        |                                            |
| totp_secret       | text        | null; encrypted at rest when set           |
| created_at        | timestamptz |                                            |
| updated_at        | timestamptz |                                            |
| deleted_at        | timestamptz | soft delete                                |

### account_members  (`mem_`)

| column      | type        | notes                                  |
| ----------- | ----------- | -------------------------------------- |
| id          | text PK     | `mem_...`                              |
| account_id  | text FK     | → accounts                             |
| user_id     | text FK     | → users                                |
| role        | text        | `admin \| member`                      |
| created_at  | timestamptz |                                        |
|             |             | UNIQUE (account_id, user_id)           |

Sign-up creates a user AND an account with that user as admin. Everything hangs off
`account_id`, never bare `user_id` — teams are day one.

### invitations  (`inv_`)

| column       | type        | notes                                 |
| ------------ | ----------- | ------------------------------------- |
| id           | text PK     | `inv_...`                             |
| account_id   | text FK     | → accounts                            |
| email        | text        | invitee                               |
| role         | text        | `admin \| member`                     |
| token_hash   | text        | sha256 of emailed token               |
| invited_by   | text FK     | → users                               |
| expires_at   | timestamptz |                                       |
| accepted_at  | timestamptz | null until accepted                   |
| accepted_by  | text FK     | → users, null                         |
| created_at   | timestamptz |                                       |

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

### api_keys  (`key_`; secret shape `foundation_api_...`, shown once at creation)

| column       | type        | notes                                              |
| ------------ | ----------- | -------------------------------------------------- |
| id           | text PK     | `key_...`                                          |
| account_id   | text FK     | → accounts                                         |
| created_by   | text FK     | → users — who made it                              |
| title        | text        | **immutable after creation**                       |
| description  | text        | null; editable any time                            |
| key_prefix   | text        | `foundation_api_` + first 6 payload chars, for console display |
| key_hash     | text        | sha256 of full secret; secret never stored         |
| last_used_at | timestamptz | coarse-grained update (throttled)                  |
| expires_at   | timestamptz | **null = no expiry**                               |
| revoked_at   | timestamptz | null                                               |
| created_at   | timestamptz |                                                    |

Unscoped for now. Future usage limits: additive nullable columns or a `key_limits` side
table — nothing reserved now, nothing blocks it.

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

### email_verifications / password_resets  (`everif_` / `pwreset_`)

| column      | type        | notes                          |
| ----------- | ----------- | ------------------------------ |
| id          | text PK     |                                |
| user_id     | text FK     | → users                        |
| token_hash  | text        |                                |
| expires_at  | timestamptz |                                |
| used_at     | timestamptz | null                           |
| created_at  | timestamptz |                                |

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
| api_shape            | text        | `granular \| unified \| omni \| chat`                         |
| api_version          | date        | version the request came in on                                |
| idempotency_key      | text        | null; UNIQUE (account_id, idempotency_key)                    |
| request_params       | jsonb       | canonicalized post-coercion params                            |
| param_adjustments    | jsonb       | what omni pruned/coerced                                      |
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

Policy (decided): **charge at enqueue, price fixed for the job's lifetime, hard 402, no
negative balances.** Failure/cancel-before-dispatch → full refund entry.

### stripe_customers  (`cust_`)

| column              | type        | notes                     |
| ------------------- | ----------- | ------------------------- |
| id                  | text PK     | `cust_...`                |
| account_id          | text FK     | UNIQUE → accounts         |
| stripe_customer_id  | text UNIQUE | `cus_...` at Stripe       |
| created_at          | timestamptz |                           |

### payment_methods  (`pm_`)

| column                    | type        | notes                        |
| ------------------------- | ----------- | ---------------------------- |
| id                        | text PK     | `pm_...` (ours)              |
| account_id                | text FK     | → accounts                   |
| stripe_payment_method_id  | text        | theirs                       |
| brand                     | text        | `visa`, …                    |
| last4                     | text        |                              |
| exp_month                 | int         |                              |
| exp_year                  | int         |                              |
| is_default                | bool        |                              |
| status                    | text        | `active \| detached`         |
| created_at                | timestamptz |                              |

### ledger_entries  (`txn_`) — APPEND-ONLY, the source of truth for money

| column                    | type        | notes                                                |
| ------------------------- | ----------- | ---------------------------------------------------- |
| id                        | text PK     | `txn_...`                                             |
| account_id                | text FK     | → accounts                                            |
| amount_micros             | bigint      | signed: + credit, − debit                             |
| entry_type                | text        | `topup \| job_charge \| job_refund \| adjustment \| promo \| auto_refill` |
| job_id                    | text FK     | → jobs; null for non-job entries                      |
| stripe_payment_intent_id  | text        | null; for topups/auto-refills                         |
| balance_after_micros      | bigint      | written under the account balance row lock            |
| memo                      | text        | null                                                  |
| idempotency_key           | text        | UNIQUE, null                                          |
| created_at                | timestamptz |                                                       |

Never UPDATE/DELETE. Invariant (reconciliation task asserts): `account_balances.balance_micros
== SUM(amount_micros)` per account.

### account_balances

| column          | type        | notes                                                  |
| --------------- | ----------- | ------------------------------------------------------- |
| account_id      | text PK/FK  | → accounts                                               |
| balance_micros  | bigint      | cache; mutated only in the same txn as a ledger insert, under `SELECT ... FOR UPDATE` |
| updated_at      | timestamptz |                                                          |

(No `reserved_micros` — holds were removed with the charge-at-enqueue decision.)

### auto_refill_rules  (`refill_`)

| column               | type        | notes                             |
| -------------------- | ----------- | --------------------------------- |
| id                   | text PK     | `refill_...`                      |
| account_id           | text FK     | UNIQUE → accounts (one rule)      |
| enabled              | bool        |                                   |
| threshold_micros     | bigint      | trigger when balance drops below  |
| refill_amount_micros | bigint      | amount to charge the card         |
| payment_method_id    | text FK     | → payment_methods                 |
| daily_max_micros     | bigint      | runaway-card safety               |
| last_triggered_at    | timestamptz | null                              |
| created_at           | timestamptz |                                   |
| updated_at           | timestamptz |                                   |

### spend_limits  (`limit_`)

| column        | type        | notes                                |
| ------------- | ----------- | ------------------------------------ |
| id            | text PK     | `limit_...`                          |
| account_id    | text FK     | → accounts                           |
| window        | text        | `weekly \| monthly`                  |
| limit_micros  | bigint      |                                      |
| behavior      | text        | `block` (only option for now)        |
| created_by    | text FK     | → users                              |
| created_at    | timestamptz |                                      |
| updated_at    | timestamptz |                                      |

Enforced at admission: rolling-window SUM of `job_charge` entries (indexed on
(account_id, entry_type, created_at)); rollup cache if/when this gets hot.

### stripe_events  (`sevt_`)

| column           | type        | notes                        |
| ---------------- | ----------- | ---------------------------- |
| id               | text PK     | `sevt_...`                   |
| stripe_event_id  | text UNIQUE | dedupe                       |
| event_type       | text        |                              |
| payload          | jsonb       |                              |
| processed_at     | timestamptz | null until handled           |
| error            | text        | null                         |
| created_at       | timestamptz |                              |

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

## 6. The admission transaction (the part to scrutinize)

**Sign-off:** _unsigned_

```
BEGIN;
  SELECT balance_micros FROM account_balances WHERE account_id = $1 FOR UPDATE;
  -- balance < cost  →  ROLLBACK, return 402
  -- spend-limit window sum + cost > limit  →  ROLLBACK, return 402 (limit variant)
  INSERT INTO ledger_entries (…, amount_micros = -cost, entry_type = 'job_charge', job_id, balance_after_micros);
  UPDATE account_balances SET balance_micros = balance_micros - cost;
  INSERT INTO jobs (…, charged_micros = cost, status = 'queued');
  INSERT INTO queue_tasks (task_type = 'dispatch_job', payload = {job_id});
COMMIT;
```

Refund path (failure/expiry/cancel-before-dispatch) is the mirror image with
`entry_type = 'job_refund'` and `jobs.refunded_micros` set. Both paths are idempotent per job
(ledger idempotency_key = `charge:{job_id}` / `refund:{job_id}`).

---

## 7. Tables owned by sibling design docs

**Sign-off:** _unsigned_

| Table                  | Prefix     | Defined in     |
| ---------------------- | ---------- | -------------- |
| request_logs           | `rlog_`    | `logging.md`   |
| provider_calls         | `pcall_`   | `logging.md`   |
| webhook_deliveries     | `whd_`     | `logging.md`   |
| usage_rollups          | `roll_`    | `rankings.md`  |
| end_user_limits (future) | `eulimit_` | `end-users.md` |
