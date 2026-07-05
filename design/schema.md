# Database Schema

**Status:** living document (2026-07-04). Postgres. Siblings: `architecture.md`, `api.md`, `models.md`.

Conventions:

- Every `id` is `TEXT PRIMARY KEY`, format `{prefix}_{26-char Crockford of 128 bits}`,
  ULID-style time-ordered. The prefix is listed with each table. No numeric IDs anywhere.
- All timestamps are `timestamptz`. `created_at` defaults to `now()`. Money is `BIGINT`
  micro-USD unless noted (`_nanos` = nano-USD unit rates).
- **No catalog tables.** Models, modalities, schemas, routes, prices live in Rust
  (`crates/core/catalog`). Jobs snapshot pricing at enqueue. Provider credentials are
  deployment config, not rows.
- Enum-ish columns are `TEXT` + CHECK constraint (values owned by `crates/core/enums`).

---

## 1. Identity & access

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

### api_keys  (`key_`; secret shape `sk_live_...`, shown once at creation)

| column       | type        | notes                                              |
| ------------ | ----------- | -------------------------------------------------- |
| id           | text PK     | `key_...`                                          |
| account_id   | text FK     | → accounts                                         |
| created_by   | text FK     | → users — who made it                              |
| title        | text        | **immutable after creation**                       |
| description  | text        | null; editable any time                            |
| key_prefix   | text        | first 12 chars of secret, for console display      |
| key_hash     | text        | sha256 of full secret; secret never stored         |
| last_used_at | timestamptz | coarse-grained update (throttled)                  |
| expires_at   | timestamptz | **null = no expiry**                               |
| revoked_at   | timestamptz | null                                               |
| created_at   | timestamptz |                                                    |

Unscoped for now. Future usage limits: additive nullable columns or a `key_limits` side
table — nothing reserved now, nothing blocks it.

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

### jobs  (`job_`)

| column               | type        | notes                                                        |
| -------------------- | ----------- | ------------------------------------------------------------ |
| id                   | text PK     | `job_...`                                                     |
| account_id           | text FK     | → accounts                                                    |
| api_key_id           | text FK     | → api_keys; **null = dashboard session**                      |
| user_id              | text FK     | → users; the human, when known                                |
| source               | text        | `api \| dashboard`                                            |
| media_type           | text        | `image \| video \| audio \| mesh \| splat \| text`            |
| model                | text        | canonical catalog slug, e.g. `kling_video_v3_pro`             |
| modality             | text        | e.g. `image_to_video` (resolved, not necessarily requested)   |
| api_shape            | text        | `granular \| unified \| omni \| chat`                         |
| api_version          | date        | version the request came in on                                |
| idempotency_key      | text        | null; UNIQUE (account_id, idempotency_key)                    |
| request_params       | jsonb       | canonicalized post-coercion params                            |
| param_adjustments    | jsonb       | what omni pruned/coerced                                      |
| status               | text        | `queued \| dispatched \| running \| succeeded \| failed \| canceled \| expired` |
| failure_code         | text        | null; machine-readable                                        |
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
(status, expires_at) for the reaper.

This table IS the customer spend list: job + status + cost in one row.

### job_events  (`jevt_`)

| column      | type        | notes                                                              |
| ----------- | ----------- | ------------------------------------------------------------------ |
| id          | text PK     | `jevt_...`                                                          |
| job_id      | text FK     | → jobs                                                              |
| event_type  | text        | `created \| charged \| dispatched \| provider_accepted \| poll_tick \| provider_completed \| output_ingested \| webhook_sent \| refunded \| failed \| …` |
| data        | jsonb       | small payload                                                       |
| created_at  | timestamptz |                                                                     |

Append-only state-machine trail; merged into the customer debug timeline.

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

### queue_tasks  (`task_`)

| column       | type        | notes                                                          |
| ------------ | ----------- | --------------------------------------------------------------- |
| id           | text PK     | `task_...`                                                       |
| task_type    | text        | `dispatch_job \| poll_provider \| ingest_outputs \| deliver_webhook \| settle_refund \| auto_refill \| …` |
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

## 5. Logs product

High-volume tables: monthly `PARTITION BY RANGE (created_at)`, minimal indexes
((account_id, created_at) + (job_id)), secrets redacted at write time. Retention: indefinite,
with future downsampling (drop bodies, keep skeletons) — partition machinery from day one.

### request_logs  (`rlog_`)

| column           | type        | notes                                   |
| ---------------- | ----------- | ---------------------------------------- |
| id               | text PK     | `rlog_...`                               |
| account_id       | text FK     | → accounts                               |
| api_key_id       | text FK     | null (dashboard = null + source cookie)  |
| job_id           | text FK     | null for non-job endpoints               |
| source           | text        | `api \| dashboard`                       |
| method           | text        |                                          |
| path             | text        |                                          |
| api_version      | date        |                                          |
| ip               | inet        |                                          |
| user_agent       | text        |                                          |
| request_headers  | jsonb       | allowlisted only                         |
| request_body     | jsonb       | redacted                                 |
| response_status  | int         |                                          |
| response_body    | jsonb       | redacted; truncated over size cap        |
| latency_ms       | int         |                                          |
| created_at       | timestamptz | partition key                            |

### provider_calls  (`pcall_`)

| column           | type        | notes                                                    |
| ---------------- | ----------- | --------------------------------------------------------- |
| id               | text PK     | `pcall_...`                                                |
| job_id           | text FK     | → jobs                                                     |
| provider         | text        | catalog slug                                               |
| direction        | text        | `request \| response \| poll_request \| poll_response \| callback_in` |
| url              | text        |                                                            |
| request_headers  | jsonb       | our provider keys stripped                                 |
| request_body     | jsonb       |                                                            |
| response_status  | int         | null                                                       |
| response_body    | jsonb       |                                                            |
| latency_ms       | int         | null                                                       |
| created_at       | timestamptz | partition key                                              |

### webhook_deliveries  (`whd_`)

| column           | type        | notes                              |
| ---------------- | ----------- | ----------------------------------- |
| id               | text PK     | `whd_...`                           |
| job_id           | text FK     | → jobs                              |
| account_id       | text FK     | → accounts                          |
| url              | text        | customer's endpoint                 |
| event_type       | text        | `job.completed \| job.failed`       |
| payload          | jsonb       |                                     |
| attempt          | int         | 1-based                             |
| response_status  | int         | null                                |
| response_body    | text        | truncated                           |
| latency_ms       | int         | null                                |
| next_retry_at    | timestamptz | null                                |
| delivered_at     | timestamptz | null                                |
| gave_up_at       | timestamptz | null                                |
| created_at       | timestamptz |                                     |

### webhook_secrets  (`whsec_` — the secret itself carries the prefix)

| column      | type        | notes                                  |
| ----------- | ----------- | --------------------------------------- |
| id          | text PK     | row id `whs_...`                        |
| account_id  | text FK     | → accounts                              |
| secret      | text        | `whsec_...`, encrypted at rest          |
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
