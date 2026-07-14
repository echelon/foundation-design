# Logging (the debug-logs product)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Owns the user-facing logs product: what we capture,
the tables (moved here from `schema.md` §5), truncation/redaction, retention, and the API.

**North star: Stripe's developer-facing logging.** Stripe's Dashboard → Developers → Logs /
Events experience is the bar — every request visible with full request/response payloads,
filterable, linked from errors via request IDs, friendly enough that debugging rarely needs
support. Our users should be able to see *everything that happens* to their requests and
self-serve their debugging the same way.

---

## 1. What we capture (per job, correlated by `job_id`)

**Sign-off:** _unsigned_

1. Inbound customer request as received (redacted)          — `request_logs`
2. Canonicalized params + omni prune/coerce adjustments     — on the `jobs` row
3. Our outbound provider request (before send, so failures still capture the payload) — `provider_calls`
4. Provider response, every poll round-trip, inbound provider callbacks — `provider_calls`
5. Our webhook deliveries to the customer + their responses — `webhook_deliveries`
6. State transitions                                        — `job_events` (schema.md)

All of it is customer-visible. This is the differentiator: Fal/Replicate expose almost none of
this, and "why was my generation weird/slow/failed" is the #1 support question in this market.

## 2. Truncation & redaction (write-time, never read-time)

**Sign-off:** _unsigned_

- **Truncation:** any logged body larger than the cap (default **1 MB**; per-table constant)
  is cut at the cap with `is_truncated = true` and the original size recorded in `body_bytes`.
  Rationale: Postgres JSONB technically takes ~255 MB per value, but every oversized body is
  paid for again in backups, replicas, and reads (`../reports/postgresql-column-types.md`).
  Truncated bodies keep a valid JSON prefix envelope: we store
  `{ "_truncated": true, "prefix": "<first N bytes as text>" }` so the column stays queryable.
- **Redaction:** our provider credentials and customers' auth headers are stripped before
  write. Header capture is allowlist-only. Null bytes sanitized (JSONB rejects them).
- **Compression:** `lz4` column compression on all body columns.

## 3. Tables

**Sign-off:** _unsigned_

Monthly `PARTITION BY RANGE (created_at)`; minimal indexes: (account_id, created_at) +
(job_id); partition machinery from day one.

### request_logs  (`rlog_`)

**Sign-off:** _unsigned_

| column              | type        | notes                                                                                          |
| ------------------- | ----------- | ---------------------------------------------------------------------------------------------- |
| sid                 | bigint      | identity; never exposed (ids.md §4)                                                            |
| id                  | text PK     | `rlog_...`                                                                                     |
| account_id          | text FK     | → accounts                                                                                     |
| api_key_id          | text FK     | → api_keys; null when cookie-authed                                                            |
| session_id          | text FK     | → sessions; set for dashboard/console requests (exactly one of api_key_id / session_id is set) |
| job_id              | text FK     | → jobs; see note below                                                                         |
| end_user_id         | text FK     | → end_users; null                                                                              |
| source              | text        | `api_key, dashboard, mcp`                                                                      |
| method              | text        |                                                                                                |
| path                | text        |                                                                                                |
| api_version         | date        |                                                                                                |
| ip                  | inet        |                                                                                                |
| user_agent          | text        |                                                                                                |
| request_headers     | jsonb       | allowlisted only                                                                               |
| request_body        | jsonb       | redacted                                                                                       |
| request_truncated   | bool        | `is_truncated` for the request body                                                            |
| request_body_bytes  | bigint      | original size pre-truncation                                                                   |
| response_status     | int         |                                                                                                |
| response_body       | jsonb       | redacted                                                                                       |
| response_truncated  | bool        |                                                                                                |
| response_body_bytes | bigint      |                                                                                                |
| latency_ms          | int         |                                                                                                |
| created_at          | timestamptz | partition key                                                                                  |

**Why `job_id` is nullable (clarified 2026-07-05):** `request_logs` records EVERY API
request, not just generations — `GET /v1/jobs`, key management, balance checks, auth, catalog
reads. Those requests have no job by nature. The invariant is narrower and absolute: **every
generation/chat/cost-charged request — from API key, dashboard sandbox, or MCP alike — always
creates a job and its log row carries that job_id.** Dashboard/sandbox runs flow through the
identical job pipeline (same billing, same statuses); they differ only in auth, which is why
`session_id` (not `api_key_id`) identifies them here.

### provider_calls  (`pcall_`)

**Sign-off:** _unsigned_

| column              | type        | notes                                                         |
| ------------------- | ----------- | ------------------------------------------------------------- |
| sid                 | bigint      | identity; never exposed (ids.md §4)                           |
| id                  | text PK     | `pcall_...`                                                   |
| job_id              | text FK     | → jobs                                                        |
| provider            | text        | catalog slug                                                  |
| direction           | text        | `request, response, poll_request, poll_response, callback_in` |
| url                 | text        |                                                               |
| request_headers     | jsonb       | our provider keys stripped                                    |
| request_body        | jsonb       |                                                               |
| request_truncated   | bool        |                                                               |
| request_body_bytes  | bigint      |                                                               |
| response_status     | int         | null                                                          |
| response_body       | jsonb       |                                                               |
| response_truncated  | bool        |                                                               |
| response_body_bytes | bigint      |                                                               |
| latency_ms          | int         | null                                                          |
| created_at          | timestamptz | partition key                                                 |

### webhook_deliveries  (`whd_`)

**Sign-off:** _unsigned_

| column          | type        | notes                               |
| --------------- | ----------- | ----------------------------------- |
| sid             | bigint      | identity; never exposed (ids.md §4) |
| id              | text PK     | `whd_...`                           |
| job_id          | text FK     | → jobs                              |
| account_id      | text FK     | → accounts                          |
| url             | text        | customer's endpoint                 |
| event_type      | text        | `job.completed, job.failed`         |
| payload         | jsonb       |                                     |
| attempt         | int         | 1-based                             |
| response_status | int         | null                                |
| response_body   | text        | truncated to 16 KB; `is_truncated`  |
| is_truncated    | bool        |                                     |
| latency_ms      | int         | null                                |
| created_at      | timestamptz |                                     |
| next_retry_at   | timestamptz | null                                |
| delivered_at    | timestamptz | null                                |
| gave_up_at      | timestamptz | null                                |

## 4. One timeline, many tables (DECIDED 2026-07-14)

**Sign-off:** _unsigned_

**Separate tables stay.** They log different shapes (an inbound HTTP request ≠ a provider
poll ≠ a webhook delivery), at different write rates, with different retention/downsampling
policies and partition schedules — one polymorphic mega-table would mean sparse columns or an
untyped jsonb blob, and per-type retention gets much harder. The read side is unified
instead.

**The unified read is a `UNION ALL` — and it's cheap, not ugly.** Every log table already
carries the same four spine columns (`id`, `job_id`, `account_id`, `created_at`), each with a
`(job_id)` index. One round trip fetches a complete job timeline:

```sql
SELECT log_type, id, created_at, summary FROM (
  SELECT 'request'        AS log_type, id, created_at,
         jsonb_build_object('method', method, 'path', path, 'status', response_status,
                            'latency_ms', latency_ms) AS summary
    FROM request_logs        WHERE job_id = $1
  UNION ALL
  SELECT 'provider_call',  id, created_at,
         jsonb_build_object('provider', provider, 'direction', direction,
                            'status', response_status, 'latency_ms', latency_ms)
    FROM provider_calls      WHERE job_id = $1
  UNION ALL
  SELECT 'webhook',        id, created_at,
         jsonb_build_object('url', url, 'event', event_type, 'attempt', attempt,
                            'status', response_status)
    FROM webhook_deliveries  WHERE job_id = $1
  UNION ALL
  SELECT 'job_event',      id, created_at,
         jsonb_build_object('event_type', event_type, 'data', data)
    FROM job_events          WHERE job_id = $1
) t
ORDER BY created_at, id;
```

Each branch is one index scan; the planner runs them independently and merges — for a single
job that's a handful of rows per branch, single-digit milliseconds. (Full payloads aren't
dragged through the union; the UI fetches a row's body lazily by `log_type` + `id` — the
self-describing prefix makes that trivial.)

**The envelope contract (how this scales to future log types):** any new log table MUST
carry the spine (`sid`, `id`, `job_id` nullable, `account_id`, `created_at`, a `(job_id)`
index) and register a summary projection. In code this is one Rust trait impl
(`TimelineSource`: table, summary projection, badge type) in `services/logging`; the timeline
query is assembled from the registry, so adding a log type never touches the endpoint.

**Grouping keys, in priority order:** `job_id` is THE correlation key (every generation
artifact carries it). `request_logs.id` (= the `request_id` in error envelopes) pins the
single inbound request; `account_id` + time range serves the log explorer. The API that
grabs everything at once is exactly `GET /v1/jobs/{id}/logs` (§6) — one call, all types,
merged and ordered; a future `GET /v1/logs?job_id=&types=` filter is additive.

## 5. Retention & downsampling

**Sign-off:** _unsigned_

Policy (decided 2026-07-04): **indefinite**, downsampled when large. Concretely:

- Phase 0 (launch): keep everything.
- Phase 1 (growth): a `downsample_partition` task rewrites partitions older than N months —
  drops `request_body`/`response_body` payloads, keeps the row skeleton (status, latency,
  sizes, truncation flags) so timelines still render.
- Media outputs are NOT logs — they're kept forever regardless (architecture.md §8).

## 6. API & console surface

**Sign-off:** _unsigned_

| Surface                      | What it shows                                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `GET /v1/jobs/{job_id}/logs` | Merged, time-ordered timeline: request → params → provider calls → polls → callbacks → webhook deliveries → events |
| Console job page             | The same timeline rendered as a request-trace view                                                                 |
| Console log explorer         | `request_logs` filterable by key, end_user, path, status, time                                                     |

Timeline entries carry `is_truncated` so the UI can badge "payload truncated at 1 MB
(original 4.2 MB)".
