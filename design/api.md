# API Surface

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Every endpoint served by `api-server`.
Base URL placeholder: `https://api.example.dev`. Siblings: `architecture.md`, `schema.md`,
`models.md`, `enums.md`, `logging.md`, `mcp.md`, `end-users.md`, `rankings.md`.

---

## 1. Conventions

**Sign-off:** _unsigned_

| Concern         | Convention                                                                     |
| --------------- | ------------------------------------------------------------------------------ |
| Auth (API)      | Canonical: `Authorization: Bearer foundation_api_...` — three more accepted forms below |
| Auth (console)  | Session cookie; same endpoints, `source=dashboard` recorded on jobs             |
| Auth (MCP)      | Same API keys via the MCP transport; `source=mcp` recorded (see `mcp.md`)       |
| Versioning      | Date version pinned per account; override header `Foundation-Version: 2026-07-04` |
| Idempotency     | `Idempotency-Key` header on POSTs; 24h replay window per account                |
| Sync mode       | **None for media** (DECIDED 2026-07-05) — polling + webhooks only; see `../reports/sync-apis.md`. LLM chat is inherently held-open (that's not a "sync mode", it's the medium) |
| Errors          | Uniform envelope, §2                                                            |
| Pagination      | Cursor-based: `?limit=&after=`                                                  |
| Content         | JSON everywhere; file upload is multipart                                       |

**Accepted auth forms** (all resolve to the same key; canonical form is what docs teach):

| Form                                        | Why we accept it                                  |
| ------------------------------------------- | -------------------------------------------------- |
| `Authorization: Bearer foundation_api_...`  | Canonical; what OpenAI/Anthropic/OpenRouter trained everyone on |
| `X-API-Key: foundation_api_...`             | Common in gateway/enterprise tooling (AWS-style)    |
| `Authorization: Key foundation_api_...`     | Fal's scheme — eases migration of Fal integrations  |
| `curl -u foundation_api_...:` (Basic, key as username, empty password) | Stripe's curl ergonomics |

Query-param keys (`?api_key=`) are **rejected** — they leak into logs and referrers.

**API key format:** `foundation_api_{40-char Crockford}` — service-declaring on sight
(GitHub-style), double-click-copyable, greppable in leaked-secret scanners. No test/live split:
everything costs real money, so a Stripe-style sandbox axis would be a lie. (`foundation`
swaps with the real product name.)

## 2. Errors & failure codes

**Sign-off:** _unsigned_

One envelope for every non-2xx response:

```json
{
  "error": {
    "type":       "insufficient_balance",
    "message":    "This job costs $0.15 but your balance is $0.02.",
    "param":      null,
    "details":    { "balance_micros": 20000, "cost_micros": 150000, "topup_url": "https://..." },
    "request_id": "rlog_01j0qs..."
  }
}
```

- `type` — machine-readable, stable, from a closed enum (`invalid_request`, `authentication_error`,
  `permission_error`, `insufficient_balance`, `spend_limit_exceeded`, `not_found`,
  `idempotency_conflict`, `rate_limited`, `provider_unavailable`, `internal_error`).
- `param` — the offending field for validation errors (`"duration_seconds"`).
- `details` — type-specific structured payload; additive, never breaking.
- `request_id` — correlates to the logs product.

### Generation failure codes (user-facing; on failed jobs, not HTTP errors)

`jobs.failure_code` — why a generation failed after acceptance. **bt: edit this list freely.**

| Code                      | Meaning                                                        |
| ------------------------- | --------------------------------------------------------------- |
| `content_policy_input`    | Prompt or input media rejected by provider moderation           |
| `content_policy_output`   | Result generated but blocked by provider output moderation      |
| `face_or_person_blocked`  | Provider refused due to real-person/face/celebrity policy       |
| `copyright_blocked`       | Provider refused due to IP/copyrighted-character policy         |
| `invalid_input`           | Provider rejected params we couldn't pre-validate               |
| `input_fetch_failed`      | Input URL/file couldn't be retrieved                            |
| `input_unsupported`       | Input format/resolution/duration outside provider limits        |
| `provider_error`          | Upstream 5xx / opaque failure                                   |
| `provider_timeout`        | Upstream exceeded our deadline                                  |
| `provider_capacity`       | Upstream throttled or over capacity (429)                       |
| `provider_deprecated`     | Model retired upstream mid-flight                               |
| `canceled`                | User canceled                                                   |
| `expired`                 | Job passed `expires_at` before completion                       |
| `internal`                | Our bug — auto-refunded and alarmed                             |

Every failure code maps to refund behavior (all of the above refund in full except `canceled`
after dispatch, which is policy TBD per provider — most upstreams charge us anyway).

## 3. Generation — three shapes × two verbs

**Sign-off:** _unsigned_

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
- All three produce the same Job resource; results arrive by polling or webhook — never a
  held-open generate call.
- Optional on every shape: `"end_user": "<customer's own user id, ≤256 chars>"` — never
  required; see `end-users.md`.

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

## 4. Jobs

**Sign-off:** _unsigned_

| Endpoint                       | Method | Purpose                                              |
| ------------------------------ | ------ | ---------------------------------------------------- |
| `/v1/jobs/{job_id}`            | GET    | Full job: status, params echo, outputs, cost         |
| `/v1/jobs/{job_id}/status`     | GET    | Lightweight status-only (cheap poll)                 |
| `/v1/jobs`                     | GET    | List; filters: status, model, media_type, source, api_key, end_user, time range |
| `/v1/jobs/{job_id}/cancel`     | POST   | Cancel if not terminal (refund if never dispatched)  |
| `/v1/jobs/{job_id}/logs`       | GET    | Merged debug timeline — the logs product (`logging.md`) |

Statuses: `queued → dispatched → running → succeeded | failed | canceled | expired`.
Failed and expired jobs are auto-refunded (ledger `job_refund`).

## 5. LLM

**Sign-off:** _unsigned_

| Endpoint                  | Method | Purpose                                                          |
| ------------------------- | ------ | ---------------------------------------------------------------- |
| `/v1/chat/completions`    | POST   | OpenAI-compatible; `model` = our slug; SSE via `"stream": true`; optional `end_user` (maps onto OpenAI's `user` field convention) |

Synchronous pass-through (no job queue); still writes a job row + request/provider logs for
spend history and debugging. Admission-checked against balance using `max_tokens` estimate;
actuals charged at completion.

## 6. Catalog (read-only; served from the code catalog)

**Sign-off:** _unsigned_

| Endpoint                                   | Method | Purpose                                        |
| ------------------------------------------ | ------ | ---------------------------------------------- |
| `/v1/models`                               | GET    | Filter by media_type, owner, modality, status  |
| `/v1/models/{model}`                       | GET    | Metadata, modalities, prices, deprecation info |
| `/v1/models/{model}/schema/{modality}`     | GET    | JSON Schema for that endpoint                  |

## 7. Files

**Sign-off:** _unsigned_

| Endpoint            | Method | Purpose                                                      |
| ------------------- | ------ | ------------------------------------------------------------ |
| `/v1/files`         | POST   | Multipart upload → `file_...` usable in any input slot       |
| `/v1/files/{id}`    | GET    | Metadata + CDN URL                                            |

Inputs may also be plain HTTPS URLs (we mirror them to R2 before dispatch).
(Naming analysis — files vs media vs artifacts: `../reports/files-naming.md`.)

## 8. Billing & spend

**Sign-off:** _unsigned_

| Endpoint                        | Method         | Purpose                                                  |
| ------------------------------- | -------------- | -------------------------------------------------------- |
| `/v1/billing/balance`           | GET            | Current balance (micros + display)                        |
| `/v1/billing/ledger`            | GET            | Ledger entries: topups, charges, refunds — the spend list |
| `/v1/billing/usage`             | GET            | Rollups: spend by day / model / api key / end_user        |
| `/v1/billing/topup`             | POST           | Create Stripe payment for a chosen amount                 |
| `/v1/billing/payment-methods`   | GET/POST/DELETE| Card on file management                                   |
| `/v1/billing/auto-refill`       | GET/PUT        | Threshold, refill amount, daily max, enable/disable       |
| `/v1/billing/spend-limits`      | GET/PUT        | Weekly/monthly caps (behavior: block)                     |

"Show users what they've spent": `GET /v1/jobs` (each row: job, status, model, cost) is the
per-job view; `/v1/billing/ledger` is the money view; `/v1/billing/usage` is the chart view.

## 9. Account, team, keys

**Sign-off:** _unsigned_

| Endpoint                          | Method           | Purpose                                            |
| --------------------------------- | ---------------- | -------------------------------------------------- |
| `/v1/account`                     | GET/PATCH        | Account settings                                    |
| `/v1/account/members`             | GET              | Team list                                           |
| `/v1/account/members/{id}`        | PATCH/DELETE     | Role change, removal                                |
| `/v1/account/invitations`         | GET/POST/DELETE  | Invite teammates                                    |
| `/v1/api-keys`                    | GET/POST         | Create: `title` (immutable), `description`, `expires_at?` — `foundation_api_...` secret shown once |
| `/v1/api-keys/{id}`               | PATCH/DELETE     | PATCH edits description only; DELETE = revoke       |
| `/v1/webhook-secret`              | GET/POST         | View/rotate `foundation_whsec_...` (two active during rotation) |
| `/v1/auth/signup|login|logout`    | POST             | Console session auth                                |
| `/v1/auth/password-reset`         | POST             | Token flow                                          |

## 10. Outbound webhooks (us → customer)

**Sign-off:** _unsigned_

**Signing: per-account HMAC — DECIDED 2026-07-05.** We evaluated Stripe-style per-account
HMAC-SHA256 vs Fal-style global Ed25519+JWKS; Stripe's approach is superior for our shape:
per-account secrets make cross-tenant replay fail closed even for lazy verifiers, verification
is stdlib-only for customers, and revocation is per-account. (Comparison recorded in
`../2026-07/2026-07-04-design-feedback.md` §10 and the 2026-07-05 discussion.)

- Events: `job.completed`, `job.failed` (later: `job.started`).
- Payload: the full Job resource + `event_type` + `event_id`.
- Signature: `Foundation-Signature: t=1720123456,v1=<hex hmac_sha256(secret, "{t}.{body}")>`.
- Secret format: `foundation_whsec_{40-char Crockford}`. The `whsec` token is a Svix
  *convention*, not a standard — prefixing it breaks nothing except drop-in use of Svix SDK
  verifiers, which we don't promise; our docs ship 5-line verify snippets per language. Replay
  tolerance: reject if `|now - t|` > 5 min.
- Retries: exponential backoff, ~8 attempts over 24h; every attempt + customer response logged
  and visible in `/v1/jobs/{id}/logs`.

## 11. MCP

**Sign-off:** _unsigned_

`/mcp` — streamable-HTTP MCP endpoint mounted in api-server; tools mirror this API
(generate/cost/jobs/models). Full design: `mcp.md`.

## 12. Public (unauthenticated)

**Sign-off:** _unsigned_

| Endpoint                  | Method | Purpose                                              |
| ------------------------- | ------ | ----------------------------------------------------- |
| `/v1/public/rankings`     | GET    | Aggregated model usage stats (see `rankings.md`)      |
| `/llms.txt`               | GET    | Redirect to docs-domain agent docs (see `agents.md`)  |

## 13. Reserved, not built

**Sign-off:** _unsigned_

`/v1/embeddings`, `/v1/batch`, realtime voice. Nothing else may squat on these paths.
