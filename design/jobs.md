# Jobs

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Promoted from
`../reports/read/jobs-design-considerations.md` per bt; the still-open items are analyzed in
`../reports/jobs-design-considerations-2.md`. Owns the jobs tables (moved from `schema.md`
§2) and the jobs API (moved from `api.md` §4). Siblings: `schema.md`, `api.md`,
`payments.md`, `logging.md`.

Decided so far (bt, 2026-07-05):

- The name is **"jobs"** — everywhere, including LLMs. Not tasks (collides with
  `queue_tasks`), not generations, not predictions.
- **`canceled_by` is a field**, not a family of statuses.
- `attempt_failed` / `dead` (ArtCraft-style retry states) are **not statuses** — retries are
  events + metadata under a stable state.
- Open: the exact naming of the our-queue/provider-queue distinction (§3 presents the
  recommendation; alternatives in `../reports/jobs-design-considerations-2.md`).
- Open: `parent_job_id` (leaning no for v1 — same report).

---

## 1. Jobs are the universal primitive

**Sign-off:** _unsigned_

Every generation attempt — API key, dashboard sandbox, MCP, media **and** LLM — creates
exactly one job row. What the job buys per request: identity (`job_...`), the charge/refund
anchor, the log-timeline anchor, the spend-list row, the rankings-rollup row, the webhook
subject, the idempotency subject. Anything that skipped jobs would need parallel plumbing for
all seven.

The LLM case holds (OpenRouter runs the same shape: one request = one "generation" you can
look up afterward). An LLM job is simply born `running` and reaches a terminal status seconds
later, with token quantities instead of seconds/megapixels. **No conversation/thread
primitive in v1** — stateless is the industry default (Anthropic, Gemini's classic API,
OpenRouter); punted explicitly (`../reports/jobs-design-considerations-2.md` §4).

## 2. Status machine

**Sign-off:** _unsigned_

### External (customer-facing) statuses

| Status      | Terminal | Meaning                                                                |
| ----------- | -------- | ----------------------------------------------------------------------- |
| `queued`    | no       | Accepted + charged; not yet running. Covers BOTH our queue and the provider's queue (see below); also covers provider-429 backoff — the job stays `queued` with visible retry metadata |
| `running`   | no       | Provider reports active generation                                       |
| `succeeded` | yes      | Set only AFTER outputs are ingested into our storage — succeeded ⇒ URLs are durable, ours |
| `failed`    | yes      | `failure_code` says why; auto-refunded                                   |
| `canceled`  | yes      | `canceled_by` says who                                                   |
| `expired`   | yes      | Passed `expires_at` before completion; auto-refunded                     |

### The two-stage queue (naming OPEN)

There are genuinely two queue residencies — ours (charged, not yet accepted upstream) and the
provider's (their task id exists). The **recommendation** (full analysis + rejected
alternatives in `../reports/jobs-design-considerations-2.md` §1) is to expose **one `queued`
status plus a `queue_stage` detail field** rather than two similar-sounding statuses:

| `queue_stage`    | Meaning                                                          |
| ---------------- | ---------------------------------------------------------------- |
| `at_our_gateway` | In OUR queue: charged, dispatch pending (incl. 429 backoff)      |
| `at_provider`    | Accepted upstream; sitting in the PROVIDER's queue               |

This matches the field (Fal collapses everything pre-runner into `IN_QUEUE`; nobody exposes a
two-tier queue as statuses), keeps the customer state machine at six values, and dissolves
the `waiting`-vs-`queuing` confusion by admitting they're one customer-visible condition.
`provider_job_id` being non-null is the same signal, machine-readably.

### Internal statuses — same enum, deliberately

Internal and external status vocabularies are **identical** (we're open source; nothing to
hide, and a private second enum would drift). Internal granularity lives in three places that
are not statuses:

| Internal concern                | Where it lives                                              |
| ------------------------------- | ------------------------------------------------------------ |
| Retry/backoff progress          | `dispatch_attempts`, `next_dispatch_at` columns + `job_events` |
| Queue residency                 | `queue_stage` (also customer-visible)                         |
| Pipeline micro-steps            | `job_events` (`output_ingested`, `poll_tick`, …)              |
| Worker bookkeeping              | `queue_tasks` rows (own lifecycle: `pending \| running \| done \| dead`) — internal work items, never customer-visible |

### Lattice, not ladder

Statuses are a lattice each provider walks a path through, not mandatory stops: LLM jobs are
born `running`; sync-ish providers (gpt-image) may go `queued → running → succeeded` inside
one worker pass; heavy video walks every stage.

### Transitions owned in one place

`services/jobs` exposes the only transition function (`transition(job, event) -> status`)
with an exhaustive match and an invalid-transition error; every worker goes through it, and
`job_events` is written in the same transaction as the status change.

## 3. Retry policy

**Sign-off:** _unsigned_

**No customer-facing retry.** Failure is terminal + auto-refund; the customer re-enqueues (a
new job, a new charge). This matches the entire field — Replicate, Runway, Luma, OpenAI all
mark failed and tell you to resubmit; nobody exposes a retry endpoint or attempt-count states
(verified 2026-07-05, `../reports/jobs-design-considerations-2.md` §3).

What we DO retry, invisibly, under a stable status:

| Situation                                   | Behavior                                                  |
| ------------------------------------------- | ----------------------------------------------------------- |
| Provider 429 / capacity at dispatch          | Job stays `queued` (stage `gateway`); backoff via `next_dispatch_at`; each attempt is a `job_events` entry; exhaustion → `failed` with `provider_capacity` |
| Transient infra error at dispatch (conn/5xx before acceptance) | Same — safe to retry, nothing charged upstream yet |
| Provider accepted, then failed               | **Fail fast**: `failed` + refund. No cross-provider re-dispatch of an accepted job — uniform retry semantics across providers is a tar pit (each upstream bills/retries differently); the refund + re-enqueue path is honest and simple |

## 4. Tables

**Sign-off:** _unsigned_

### jobs  (`job_`)

| column               | type        | notes                                                        |
| -------------------- | ----------- | ------------------------------------------------------------ |
| id                   | text PK     | `job_...`                                                     |
| account_id           | text FK     | → accounts                                                    |
| api_key_id           | text FK     | → api_keys; **null = dashboard session**                      |
| user_id              | text FK     | → users; the human, when known                                |
| end_user_id          | text FK     | → end_users; null — the customer's customer, when supplied    |
| source               | text        | enum, see below                                               |
| media_type           | text        | enum, see below                                               |
| model                | text        | canonical catalog slug, e.g. `kling_video_v3_pro`             |
| modality             | text        | e.g. `image_to_video` (resolved, not necessarily requested); values = `models.md` §11 |
| api_shape            | text        | enum, see below                                               |
| api_version          | date        | version the request came in on                                |
| idempotency_key      | text        | null; UNIQUE (account_id, idempotency_key)                    |
| request_params       | jsonb       | canonicalized post-coercion params                            |
| param_adjustments    | jsonb       | what the Any layer pruned/coerced                             |
| status               | text        | enum, see §2                                                  |
| queue_stage          | text        | enum, see §2; null unless status = `queued` (naming OPEN)     |
| dispatch_attempts    | int         | dispatch tries so far (429/backoff visibility)                |
| failure_code         | text        | null; enum owned by `api.md` §2 (bt-editable list)            |
| failure_message      | text        | null; human-readable                                          |
| canceled_by          | text        | null; enum, see below                                         |
| provider             | text        | catalog provider slug, e.g. `bytedance` (`enums.md` Provider) |
| provider_account     | text        | credential pool name from config, e.g. `byteplus_ultra`       |
| provider_job_id      | text        | upstream's task/request id; indexed                           |
| price_unit           | text        | snapshot: `per_video_second`, `per_image`, `per_token_in/out`, … (catalog-owned set) |
| unit_amount_nanos    | bigint      | snapshot: nano-USD rate charged                               |
| quantity             | numeric     | snapshot: units billed (e.g. seconds)                         |
| charged_micros       | bigint      | fixed mode: set at enqueue, never changes. metered mode (LLM): set at completion (architecture.md §4) |
| refunded_micros      | bigint      | 0 unless failed/canceled/expired → refunded                   |
| webhook_url          | text        | null                                                          |
| webhook_events       | text[]      |                                                               |
| created_at           | timestamptz | = enqueue + charge moment                                     |
| next_dispatch_at     | timestamptz | null; when the next dispatch attempt is due (pairs with dispatch_attempts) |
| dispatched_at        | timestamptz | null; when the provider accepted (queue_stage → provider)     |
| started_at           | timestamptz | null; when generation began (status → running)                |
| completed_at         | timestamptz | null; terminal timestamp                                      |
| expires_at           | timestamptz | give-up deadline                                              |

Indexes: (account_id, created_at desc), (account_id, status), (provider, provider_job_id),
(status, expires_at) for the reaper, (account_id, end_user_id, created_at) partial where
end_user_id is not null.

This table IS the customer spend list: job + status + cost in one row.

**Enum-valued fields** (all TEXT + CHECK, generated from `crates/core/enums`):

| Field         | Values                                                                  |
| ------------- | ------------------------------------------------------------------------ |
| `source`      | `api \| dashboard \| mcp`                                                 |
| `media_type`  | `image \| video \| audio \| mesh \| splat \| text`                        |
| `api_shape`   | `granular \| unified \| any \| chat`                                      |
| `status`      | `queued \| running \| succeeded \| failed \| canceled \| expired`         |
| `queue_stage` | `gateway \| provider` (naming OPEN with the two-stage question, §2)       |
| `canceled_by` | `user \| system \| provider` — user hit cancel; we canceled (suspension, policy, ops); upstream canceled it out from under us |
| `failure_code`| owned by `api.md` §2 (`content_policy_input`, `provider_capacity`, …) — bt-editable |
| `modality`    | owned by `models.md` §11                                                  |
| `provider`    | owned by `enums.md` (dispatch targets)                                    |

NOT included (open question, leaning no): `parent_job_id` — causal lineage analysis in
`../reports/jobs-design-considerations-2.md` §4.

### job_events  (`jevt_`)

| column      | type        | notes                                                              |
| ----------- | ----------- | ------------------------------------------------------------------ |
| id          | text PK     | `jevt_...`                                                          |
| job_id      | text FK     | → jobs                                                              |
| event_type  | text        | `created \| charged \| dispatch_attempted \| provider_accepted \| poll_tick \| provider_completed \| output_ingested \| webhook_sent \| refunded \| failed \| canceled \| expired \| …` |
| data        | jsonb       | small payload (e.g. `{attempt: 3, retry_in_ms: 8000}` on a 429)     |
| created_at  | timestamptz |                                                                     |

Append-only state-machine trail; merged into the customer debug timeline (`logging.md`).
Retries live HERE (as `dispatch_attempted` events), never as statuses.

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

## 5. API

**Sign-off:** _unsigned_

Moved from `api.md` §4 (2026-07-05). Conventions (auth, pagination, errors) stay in `api.md`.

| Endpoint                       | Method | Purpose                                              |
| ------------------------------ | ------ | ---------------------------------------------------- |
| `/v1/jobs/{job_id}`            | GET    | Full job: status, params echo, outputs, cost         |
| `/v1/jobs/{job_id}/status`     | GET    | Lightweight status-only (cheap poll)                 |
| `/v1/jobs`                     | GET    | List; filters: status, model, media_type, source, api_key, end_user, time range |
| `/v1/jobs/{job_id}/cancel`     | POST   | Cancel if not terminal (full refund if still in our queue; after provider acceptance, refund policy is per-provider — most upstreams charge us anyway) |
| `/v1/jobs/{job_id}/logs`       | GET    | Merged debug timeline — the logs product (`logging.md`) |

Failed and expired jobs are auto-refunded (ledger `job_refund`). No retry endpoint (§3).
