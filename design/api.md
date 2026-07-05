# API Surface

**Sign-off:** _unsigned_
<!-- bt: replace "_unsigned_" with initials + date (e.g. "bt 2026-07-05") when approved.
     Assistants: reset to _unsigned_ on any substantive edit. See ../CLAUDE.md. -->

**Status:** living document (2026-07-04). Every endpoint served by `api-server`.
Base URL placeholder: `https://api.example.dev`. Siblings: `architecture.md`, `schema.md`, `models.md`.

---

## 1. Conventions

| Concern         | Convention                                                                     |
| --------------- | ------------------------------------------------------------------------------ |
| Auth (API)      | `Authorization: Bearer sk_live_...`                                            |
| Auth (console)  | Session cookie; same endpoints, `source=dashboard` recorded on jobs             |
| Versioning      | Date version pinned per account; override header `Foundation-Version: 2026-07-04` |
| Idempotency     | `Idempotency-Key` header on POSTs; 24h replay window per account                |
| Sync mode       | `Prefer: wait[=N]` on generate endpoints — hold up to N s, else return queued job |
| Errors          | `{ "error": { "type", "code", "message", "param" } }`; 402 body includes balance, cost, top-up URL |
| Pagination      | Cursor-based: `?limit=&after=`                                                  |
| Content         | JSON everywhere; file upload is multipart                                       |

## 2. Generation — three shapes × two verbs

Every generate endpoint has a **cost twin** with the identical request body. `POST /v1/cost/...`
returns the exact fixed price the matching generate call would charge — same validation, same
coercion, no job, no charge. (Media prices are deterministic at submission and never change
after enqueue.)

| Shape                | Generate                                       | Cost                                       |
| -------------------- | ---------------------------------------------- | ------------------------------------------ |
| Omni (per media)     | `POST /v1/generate/video`                      | `POST /v1/cost/video`                      |
|                      | `POST /v1/generate/image`                      | `POST /v1/cost/image`                      |
|                      | `POST /v1/generate/audio`                      | `POST /v1/cost/audio`                      |
|                      | `POST /v1/generate/mesh`                       | `POST /v1/cost/mesh`                       |
|                      | `POST /v1/generate/splat`                      | `POST /v1/cost/splat`                      |
| Unified (per model)  | `POST /v1/generate/models/{model}`             | `POST /v1/cost/models/{model}`             |
| Granular             | `POST /v1/generate/models/{model}/{modality}`  | `POST /v1/cost/models/{model}/{modality}`  |

- Omni body: `{ "model": "kling_video_v3_pro", ...superset params... }` — irrelevant params
  pruned/coerced; adjustments reported in the response and job record.
- Unified body: model's own param union; modality inferred from present params, echoed back as
  `resolved_modality`.
- Granular body: exactly that (model, modality)'s schema.
- All three produce the same Job resource.

**Cost response:**

```json
{
  "currency":       "usd",
  "amount_micros":  150000,
  "amount_display": "$0.15",
  "model":          "kling_video_v3_pro",
  "modality":       "image_to_video",
  "line_items":     [ { "unit": "per_video_second", "unit_nanos": 30000000, "quantity": 5 } ],
  "param_adjustments": []
}
```

**Generate response (201):**

```json
{
  "id":                "job_01j0qs3fw9x7k2m5n8p1r4t7v0",
  "status":            "queued",
  "model":             "kling_video_v3_pro",
  "modality":          "image_to_video",
  "charged_micros":    150000,
  "created_at":        "2026-07-04T20:11:00Z",
  "urls":              { "get": "...", "cancel": "..." }
}
```

**Webhook registration (any generate call):**

```json
"webhook": { "url": "https://customer.example/hook", "events": ["job.completed", "job.failed"] }
```

## 3. Jobs

| Endpoint                       | Method | Purpose                                              |
| ------------------------------ | ------ | ---------------------------------------------------- |
| `/v1/jobs/{job_id}`            | GET    | Full job: status, params echo, outputs, cost         |
| `/v1/jobs/{job_id}/status`     | GET    | Lightweight status-only (cheap poll)                 |
| `/v1/jobs`                     | GET    | List; filters: status, model, media_type, source, api_key, time range |
| `/v1/jobs/{job_id}/cancel`     | POST   | Cancel if not terminal (refund if never dispatched)  |
| `/v1/jobs/{job_id}/logs`       | GET    | Merged debug timeline (the logs product)             |

Statuses: `queued → dispatched → running → succeeded | failed | canceled | expired`.
Failed and expired jobs are auto-refunded (ledger `job_refund`).

## 4. LLM

| Endpoint                  | Method | Purpose                                                          |
| ------------------------- | ------ | ---------------------------------------------------------------- |
| `/v1/chat/completions`    | POST   | OpenAI-compatible; `model` = our slug; SSE via `"stream": true`   |

Synchronous pass-through (no job queue); still writes a job row + request/provider logs for
spend history and debugging. Admission-checked against balance using `max_tokens` estimate;
actuals charged at completion.

## 5. Catalog (read-only; served from the code catalog)

| Endpoint                                   | Method | Purpose                                        |
| ------------------------------------------ | ------ | ---------------------------------------------- |
| `/v1/models`                               | GET    | Filter by media_type, owner, modality, status  |
| `/v1/models/{model}`                       | GET    | Metadata, modalities, prices, deprecation info |
| `/v1/models/{model}/schema/{modality}`     | GET    | JSON Schema for that endpoint                  |

## 6. Files

| Endpoint            | Method | Purpose                                                      |
| ------------------- | ------ | ------------------------------------------------------------ |
| `/v1/files`         | POST   | Multipart upload → `file_...` usable in any input slot       |
| `/v1/files/{id}`    | GET    | Metadata + CDN URL                                            |

Inputs may also be plain HTTPS URLs (we mirror them to R2 before dispatch).

## 7. Billing & spend

| Endpoint                        | Method         | Purpose                                                  |
| ------------------------------- | -------------- | -------------------------------------------------------- |
| `/v1/billing/balance`           | GET            | Current balance (micros + display)                        |
| `/v1/billing/ledger`            | GET            | Ledger entries: topups, charges, refunds — the spend list |
| `/v1/billing/usage`             | GET            | Rollups: spend by day / model / api key over a range      |
| `/v1/billing/topup`             | POST           | Create Stripe payment for a chosen amount                 |
| `/v1/billing/payment-methods`   | GET/POST/DELETE| Card on file management                                   |
| `/v1/billing/auto-refill`       | GET/PUT        | Threshold, refill amount, daily max, enable/disable       |
| `/v1/billing/spend-limits`      | GET/PUT        | Weekly/monthly caps (behavior: block)                     |

"Show users what they've spent": `GET /v1/jobs` (each row: job, status, model, cost) is the
per-job view; `/v1/billing/ledger` is the money view; `/v1/billing/usage` is the chart view.

## 8. Account, team, keys

| Endpoint                          | Method           | Purpose                                            |
| --------------------------------- | ---------------- | -------------------------------------------------- |
| `/v1/account`                     | GET/PATCH        | Account settings                                    |
| `/v1/account/members`             | GET              | Team list                                           |
| `/v1/account/members/{id}`        | PATCH/DELETE     | Role change, removal                                |
| `/v1/account/invitations`         | GET/POST/DELETE  | Invite teammates                                    |
| `/v1/api-keys`                    | GET/POST         | Create: `title` (immutable), `description`, `expires_at?` — secret shown once |
| `/v1/api-keys/{id}`               | PATCH/DELETE     | PATCH edits description only; DELETE = revoke       |
| `/v1/webhook-secret`              | GET/POST         | View/rotate `whsec_...` (two active during rotation)|
| `/v1/auth/signup|login|logout`    | POST             | Console session auth                                |
| `/v1/auth/password-reset`         | POST             | Token flow                                          |

## 9. Outbound webhooks (us → customer)

- Events: `job.completed`, `job.failed` (later: `job.started`).
- Payload: the full Job resource + `event_type` + `event_id`.
- Signature header (Stripe-style HMAC): `Foundation-Signature: t=1720123456,v1=<hex hmac_sha256(whsec, "{t}.{body}")>`.
- Retries: exponential backoff, ~8 attempts over 24h; every attempt + customer response logged
  and visible in `/v1/jobs/{id}/logs`.

## 10. Reserved, not built

`/v1/chat/completions` streaming extensions (tools passthrough — in v1), `/v1/embeddings`,
`/v1/batch`, realtime voice. Nothing else may squat on these paths.
