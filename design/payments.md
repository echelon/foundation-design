# Payments & Billing

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Peeled from `schema.md` §4/§6; this doc owns all
money: the prepaid dollar balance, cards on file, auto-refill, spend limits, Stripe Checkout,
Stripe webhooks, and the admission/refund transactions.

**Vocabulary rule:** we do NOT sell "credits." Customers **fund their account with US
dollars** and hold a **prepaid dollar balance**. Everything is denominated in USD
(micro-USD internally); the console shows dollars. The word "credits" never appears in
product copy (ledger entry types like `promo` are internal terms).

---

## 1. The money model

**Sign-off:** _unsigned_

- One **prepaid balance** per account (`i64` micro-USD). Jobs charge it at enqueue; failures
  refund it. Hard `402` when balance < cost. No negative balances (architecture.md §4).
- The **ledger is the source of truth** (append-only); the balance is a cache updated in the
  same transaction under `FOR UPDATE`; a reconciliation task asserts
  `balance == SUM(ledger)` per account.
- **One bucket at launch.** Promotional funds (sign-up grants, make-goods) are ordinary
  ledger entries (`entry_type = promo`) in the same balance. If we ever need real bucket
  separation (promo-spends-first, expiring grants, non-refundable vs refundable funds), the
  ledger design extends to a `bucket` column + per-bucket balance rows without breaking
  history — deliberately kept open, not built.

## 2. Tables (moved from schema.md §4)

**Sign-off:** _unsigned_

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

### ledger_entries  (`txn_`) — APPEND-ONLY

| column                    | type        | notes                                                |
| ------------------------- | ----------- | ---------------------------------------------------- |
| id                        | text PK     | `txn_...`                                             |
| account_id                | text FK     | → accounts                                            |
| amount_micros             | bigint      | signed: + credit, − debit                             |
| entry_type                | text        | `topup \| job_charge \| job_refund \| adjustment \| promo \| auto_refill \| chargeback \| chargeback_reversal` |
| job_id                    | text FK     | → jobs; null for non-job entries                      |
| stripe_payment_intent_id  | text        | null; topups/auto-refills/chargebacks                 |
| balance_after_micros      | bigint      | written under the balance row lock                    |
| memo                      | text        | null                                                  |
| idempotency_key           | text        | UNIQUE, null (e.g. `topup:{checkout_session_id}`)     |
| created_at                | timestamptz |                                                       |

### account_balances

| column          | type        | notes                                                  |
| --------------- | ----------- | ------------------------------------------------------- |
| account_id      | text PK/FK  | → accounts                                               |
| balance_micros  | bigint      | cache; mutated only with a ledger insert, under `FOR UPDATE` |
| updated_at      | timestamptz |                                                          |

### auto_refill_rules  (`refill_`)

| column               | type        | notes                                       |
| -------------------- | ----------- | -------------------------------------------- |
| id                   | text PK     | `refill_...`                                 |
| account_id           | text FK     | UNIQUE → accounts (one rule)                 |
| enabled              | bool        |                                              |
| threshold_micros     | bigint      | trigger when balance drops below             |
| refill_amount_micros | bigint      | amount to charge                             |
| payment_method_id    | text FK     | → payment_methods                            |
| daily_max_micros     | bigint      | runaway-card safety                          |
| consecutive_failures | int         | rule auto-disables at 3 (see §6)             |
| last_triggered_at    | timestamptz | null                                         |
| created_at           | timestamptz |                                              |
| updated_at           | timestamptz |                                              |

### spend_limits  (`limit_`)

| column        | type        | notes                                |
| ------------- | ----------- | ------------------------------------ |
| id                  | text PK     | `limit_...`                          |
| account_id          | text FK     | → accounts                           |
| window              | text        | `weekly \| monthly`                  |
| limit_micros        | bigint      |                                      |
| behavior            | text        | `block`                              |
| created_by_user_id  | text FK     | → users                              |
| created_at          | timestamptz |                                      |
| updated_at          | timestamptz |                                      |

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

## 3. Funding the balance: Stripe Checkout sessions

**Sign-off:** _unsigned_

Top-ups use **Stripe Checkout** (hosted page — we never touch card data; SCA/3DS handled by
Stripe):

1. Console → "Add funds" → amount (presets + custom, min $5, max $10k self-serve) →
   `POST /v1/billing/topup {amount_micros}`.
2. Server creates a Checkout Session: `mode=payment`, our `stripe_customer_id`, line item =
   the amount, `metadata: {account_id, kind: topup}`,
   `payment_intent_data.setup_future_usage=off_session` **if** the user ticked "save this
   card" (this is how cards get on file), plus success/cancel URLs back to the console.
3. **The redirect is NOT the source of truth.** Funds are credited exclusively by the
   `checkout.session.completed` webhook (§5): ledger `topup` with
   `idempotency_key = topup:{session_id}` — replays and double-clicks are no-ops.
4. Success page polls balance briefly ("funds usually appear in seconds") to cover webhook
   lag.

Card-on-file without a purchase: `mode=setup` Checkout Session (SetupIntent under the hood) →
`payment_methods` row on the `setup_intent.succeeded`/`checkout.session.completed` webhook.

## 4. Auto-refill

**Sign-off:** _unsigned_

- Trigger check runs inside the admission path (cheap: after a charge, if
  `balance < threshold` and rule enabled → enqueue `auto_refill` task) and via a sweeper for
  balances that drift down between requests.
- The task charges **off-session**: PaymentIntent with the saved `payment_method`,
  `off_session: true`, `confirm: true`, idempotency key `refill:{rule_id}:{yyyymmddhh}` (also
  enforces at-most-hourly cadence per rule).
- Guards: `daily_max_micros` (sum of today's `auto_refill` entries), rule disabled ⇒ no-op,
  and **3 consecutive failures auto-disable the rule** + email the admins (expired cards
  shouldn't retry forever).
- SCA: off-session charges can fail with `authentication_required` — treated as a failure
  (email links the user to a Checkout session to re-authenticate the card). We never hold
  jobs waiting on a refill; a refill that lands simply makes the next request pass admission.

## 5. Stripe webhook handling

**Sign-off:** _unsigned_

One endpoint: `POST /stripe/webhook` (not under `/v1` — Stripe-facing, signature-authed).
Pipeline:

1. Verify `Stripe-Signature` (endpoint secret; 5-min tolerance).
2. Insert into `stripe_events` (`stripe_event_id` UNIQUE → duplicate deliveries dead-end
   here). Return `200` immediately.
3. A `process_stripe_event` queue task does the real work — webhook handlers stay fast and
   the processing is retryable/idempotent (every money mutation goes through ledger
   idempotency keys).

| Event                                  | Handling                                                  |
| -------------------------------------- | ---------------------------------------------------------- |
| `checkout.session.completed`           | mode=payment → ledger `topup` (idempotent by session id); save PM if setup_future_usage; mode=setup → register payment_method |
| `payment_intent.succeeded`             | auto-refill confirmations → ledger `auto_refill`            |
| `payment_intent.payment_failed`        | auto-refill failure bookkeeping (§4), notify                |
| `setup_intent.succeeded`               | payment_methods upsert                                      |
| `payment_method.detached` / `customer.updated` | mirror card list / default changes                   |
| `charge.refunded`                      | (we initiated) ledger `adjustment` with memo; support tooling |
| `charge.dispute.created`               | ledger `chargeback` (debit), **account → suspended** pending review — disputes on a prepaid API business are a fraud signal, not a customer-service signal |
| `charge.dispute.closed` (won)          | ledger `chargeback_reversal`, un-suspend                    |
| `radar.early_fraud_warning.created`    | alert ops; optional preemptive suspend                      |

Unhandled event types are stored (audit) and marked processed. A daily reconciliation task
cross-checks Stripe balance transactions vs our `topup`/`auto_refill` ledger entries.

## 6. Admission & refund transactions (moved from schema.md §6)

**Sign-off:** _unsigned_

```
BEGIN;
  SELECT balance_micros FROM account_balances WHERE account_id = $1 FOR UPDATE;
  -- balance < cost                       → ROLLBACK, 402 (insufficient_balance)
  -- window spend + cost > spend limit    → ROLLBACK, 402 (spend_limit_exceeded)
  INSERT INTO ledger_entries (amount_micros = -cost, entry_type = 'job_charge', job_id,
                              balance_after_micros, idempotency_key = 'charge:{job_id}');
  UPDATE account_balances SET balance_micros = balance_micros - cost;
  INSERT INTO jobs (charged_micros = cost, status = 'queued');
  INSERT INTO queue_tasks (task_type = 'dispatch_job');
COMMIT;
-- post-commit: if balance < refill threshold → enqueue auto_refill task
```

Refunds (job failed/expired/canceled-before-dispatch) mirror it with
`entry_type = 'job_refund'`, `idempotency_key = 'refund:{job_id}'`, and `jobs.refunded_micros`
set. LLM jobs: admission-check with the `max_tokens` estimate, single `job_charge` written at
completion with actuals (the one exception; architecture.md §7).

## 7. Console & API surface (already in api.md §8 — recap)

**Sign-off:** _unsigned_

Balance display, ledger (the money view), usage rollups, top-up flow, saved cards
(add/remove/default), auto-refill rule editor with the safety rails visible (daily max,
auto-disable), spend limits. Everything read-side comes from the ledger — no second
bookkeeping system.
