# API Surface

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Endpoints are split across TWO gateway binaries — `api-gateway` (the product) and `dashboard-gateway` (the console) — per `microservices.md`.
Base URL placeholder: `https://api.example.dev`. Siblings: `architecture.md`, `schema.md`,
`models.md`, `enums.md`, `logging.md`, `mcp.md`, `end-users.md`, `rankings.md` — and the
generation planes: `api_any.md`, `api_model_specific.md`, `apis_llm.md`.

---

## 1. Conventions

**Sign-off:** _unsigned_

| Concern        | Convention                                                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Auth (API)     | Canonical: `Authorization: Bearer foundation_api_...`, `source=api_key` — three more accepted forms below                                                                      |
| Auth (console) | Session cookie; same endpoints, `source=dashboard` recorded on jobs                                                                                                            |
| Auth (MCP)     | Same API keys via the MCP transport; `source=mcp` recorded (see `mcp.md`)                                                                                                      |
| Versioning     | Date version pinned per account; override header `Foundation-Version: 2026-07-04`                                                                                              |
| Idempotency    | `Idempotency-Key` header on POSTs; 24h replay window per account                                                                                                               |
| Sync mode      | **None for media** (DECIDED 2026-07-05) — polling + webhooks only; see `../reports/sync-apis.md`. LLM chat is inherently held-open (that's not a "sync mode", it's the medium) |
| Errors         | Uniform envelope, §2                                                                                                                                                           |
| Pagination     | Cursor-based: `?limit=&after=`                                                                                                                                                 |
| Content        | JSON everywhere; file upload is multipart                                                                                                                                      |

**Accepted auth forms** (all resolve to the same key; canonical form is what docs teach):

| Form                                                                   | Why we accept it                                                |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- |
| `Authorization: Bearer foundation_api_...`                             | Canonical; what OpenAI/Anthropic/OpenRouter trained everyone on |
| `X-API-Key: foundation_api_...`                                        | Common in gateway/enterprise tooling (AWS-style)                |
| `Authorization: Key foundation_api_...`                                | Fal's scheme — eases migration of Fal integrations              |
| `curl -u foundation_api_...:` (Basic, key as username, empty password) | Stripe's curl ergonomics                                        |

Note: You cannot use multiple forms at once, or the server will reject it.

Query-param keys (`?api_key=`) are **rejected** — they leak into logs and referrers.

**API key format:** `foundation_api_{40-char Crockford}` — service-declaring on sight
(GitHub-style), double-click-copyable, greppable in leaked-secret scanners. No test/live split:
everything costs real money, so a Stripe-style sandbox axis would be a lie.

**The prefix is config, not contract (2026-07-14):** verification accepts a configured set of
prefixes — one canonical (used to mint) plus legacy ones kept for compatibility — and keys
resolve by hash, with the prefix serving only recognition. Renaming the service means adding
a new canonical prefix and demoting `foundation_api_` to legacy; no key is ever invalidated
by a rename, and docs/scanner patterns list all accepted prefixes. (Storage side:
`accounts.md` §2 api_keys.)

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

### `FailureCode` — the generation-failure enum (user-facing; on failed jobs, not HTTP errors)

`jobs.failure_code` — why a generation failed after acceptance. A proper first-class enum
(PROMOTED 2026-07-14), hand-written in `crates/core/enums` with the usual explicit serde
renames and golden tests (`enums.md`). **bt: edit the value list freely** — this table stays
the authority; the enum mirrors it.

Structure follows ArtCraft's `GenerateError`/`*Reason` pattern — categorized, with the policy
questions answered *on the enum* instead of scattered through worker code — but flattened to
one wire level, since this is a DB/API value, not an internal error type:

```rust
pub enum FailureCode {
  #[serde(rename = "content_policy_input")]
  ContentPolicyInput,
  // ... one variant per row below
}

pub enum FailureCategory { Moderation, BadInput, Provider, Lifecycle, Internal }

impl FailureCode {
  pub const fn category(self) -> FailureCategory { /* exhaustive match */ }
  pub const fn refunds_in_full(self) -> bool     { /* exhaustive match */ }
  pub const fn is_customer_actionable(self) -> bool { /* can changing the request fix it? */ }
}
```

| Code                     | Category   | Meaning                                                    |
| ------------------------ | ---------- | ---------------------------------------------------------- |
| `content_policy_input`   | moderation | Prompt or input media rejected by provider moderation      |
| `content_policy_output`  | moderation | Result generated but blocked by provider output moderation |
| `face_or_person_blocked` | moderation | Provider refused due to real-person/face/celebrity policy  |
| `copyright_blocked`      | moderation | Provider refused due to IP/copyrighted-character policy    |
| `invalid_input`          | bad_input  | Provider rejected params we couldn't pre-validate          |
| `input_fetch_failed`     | bad_input  | Input URL/file couldn't be retrieved                       |
| `input_unsupported`      | bad_input  | Input format/resolution/duration outside provider limits   |
| `provider_error`         | provider   | Upstream 5xx / opaque failure                              |
| `provider_timeout`       | provider   | Upstream exceeded our deadline                             |
| `provider_capacity`      | provider   | Upstream throttled or over capacity (429)                  |
| `provider_deprecated`    | provider   | Model retired upstream mid-flight                          |
| `canceled`               | lifecycle  | User canceled                                              |
| `expired`                | lifecycle  | Job passed `expires_at` before completion                  |
| `internal`               | internal   | Our bug — auto-refunded and alarmed                        |

- `category()` drives grouping in the console and docs (moderation vs bad_input is exactly
  the "was this my fault?" question customers ask); `moderation` + `bad_input` are
  customer-actionable, `provider` + `internal` are not.
- `refunds_in_full()` is true for every code except `canceled` after dispatch (policy TBD
  per provider — most upstreams charge us anyway).
- The mapping FROM client errors: each provider client's `{X}SpecificApiError` bucket
  (`coding_conventions.md` §3) maps onto these codes in `services/jobs` — classification is
  a client concern, `FailureCode` assignment is policy and lives in one exhaustive match.

## 3. Generation — shared conventions; the planes live in their own docs

**Sign-off:** _unsigned_

Every generate endpoint has a **cost twin** with the identical request body (`.../cost`), which
returns the exact fixed price the matching generate call would charge — same validation, same
coercion, no job, no charge. (Media prices are deterministic at submission and never change
after enqueue.)

**The shapes moved to their own docs (2026-07-14), each with its own path namespace:**

| Plane                | Namespace                                           | Doc                     |
| -------------------- | --------------------------------------------------- | ----------------------- |
| Any                  | `/v1/any/{media_type}` (+ `/cost`)                  | `api_any.md`            |
| Unified + Granular   | `/v1/models/{model}/generate[...]` (+ `/cost[...]`) | `api_model_specific.md` |
| Chat (OpenAI-compat) | `/v1/chat/completions`                              | `apis_llm.md`           |

(The previous `/v1/generate/*` + `/v1/cost/*` paths are retired; alternatives-considered
tables live in the plane docs. Any branding: DECIDED 2026-07-05, replacing "omni" —
`../reports/read/omni-api-naming.md`.)

Shared across every plane:

- All planes produce the same Job resource with `api_shape` recording the door used. Media
  is always async (polling or webhook — never held-open); text/chat is synchronous.
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

**Moved to `jobs.md` (2026-07-05)** — endpoints, the status machine (internal + external),
cancel/refund semantics, and the retry policy all live there. Jobs are the universal
primitive; every generate call in §3 (and every LLM call in §5) produces one.

## 5. LLM

**Sign-off:** _unsigned_

**Moved to `apis_llm.md` (2026-07-14):** the OpenAI-compatible `/v1/chat/completions` plane —
param pass-through rules, response deltas (job ID as completion `id`), streaming, metered
billing, and the reserved LLM paths.

## 6. Catalog (read-only; served from the code catalog)

**Sign-off:** _unsigned_

| Endpoint                               | Method | Purpose                                        |
| -------------------------------------- | ------ | ---------------------------------------------- |
| `/v1/models`                           | GET    | Filter by media_type, owner, modality, status  |
| `/v1/models/{model}`                   | GET    | Metadata, modalities, prices, deprecation info |
| `/v1/models/{model}/schema/{modality}` | GET    | JSON Schema for that endpoint                  |

## 7. Files

**Sign-off:** _unsigned_

| Endpoint         | Method | Purpose                                                |
| ---------------- | ------ | ------------------------------------------------------ |
| `/v1/files`      | POST   | Multipart upload → `file_...` usable in any input slot |
| `/v1/files/{id}` | GET    | Metadata + CDN URL                                     |

Inputs may also be plain HTTPS URLs (we mirror them to R2 before dispatch).
Tables (`files` + `outputs`) and the storage/provenance split: **`files.md`**.
(Naming analysis — files vs media vs artifacts: `../reports/read/files-naming.md`.)

## 8. Billing & spend

**Sign-off:** _unsigned_

| Endpoint                      | Method          | Purpose                                                   |
| ----------------------------- | --------------- | --------------------------------------------------------- |
| `/v1/billing/balance`         | GET             | Current balance (micros + display)                        |
| `/v1/billing/ledger`          | GET             | Ledger entries: topups, charges, refunds — the spend list |
| `/v1/billing/usage`           | GET             | Rollups: spend by day / model / api key / end_user        |
| `/v1/billing/topup`           | POST            | Create Stripe payment for a chosen amount                 |
| `/v1/billing/payment-methods` | GET/POST/DELETE | Card on file management                                   |
| `/v1/billing/auto-refill`     | GET/PUT         | Threshold, refill amount, daily max, enable/disable       |
| `/v1/billing/spend-limits`    | GET/PUT         | Weekly/monthly caps (behavior: block)                     |

"Show users what they've spent": `GET /v1/jobs` (each row: job, status, model, cost) is the
per-job view; `/v1/billing/ledger` is the money view; `/v1/billing/usage` is the chart view.

## 9. Account, team, keys

**Sign-off:** _unsigned_

| Endpoint                         | Method          | Purpose                                                                                                                         |
| -------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `/v1/account`                    | GET/PATCH       | Account settings                                                                                                                |
| `/v1/account/members`            | GET             | Team list                                                                                                                       |
| `/v1/account/members/{id}`       | PATCH/DELETE    | Role change, removal                                                                                                            |
| `/v1/account/invitations`        | GET/POST/DELETE | Invite teammates                                                                                                                |
| `/v1/api-keys`                   | GET/POST        | Create: `title` (immutable), `description`, `expires_at?` — `foundation_api_...` secret shown once                              |
| `/v1/api-keys/{id}`              | PATCH/DELETE    | PATCH edits description only; DELETE = revoke                                                                                   |
| `/v1/webhook-secret`             | GET/POST        | View/rotate `foundation_whsec_...` (two active during rotation)                                                                 |
| `/v1/auth/{signup,login,logout}` | POST            | Console session auth (`auth.md`; full auth surface: `accounts.md` §6 — login_methods, sessions, audit logs, ownership transfer) |
| `/v1/auth/password-reset`        | POST            | Token flow                                                                                                                      |

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

`/mcp` — streamable-HTTP MCP endpoint mounted in api-gateway; tools mirror this API
(generate/cost/jobs/models). Full design: `mcp.md`.

## 12. Public (unauthenticated)

**Sign-off:** _unsigned_

| Endpoint              | Method | Purpose                                              |
| --------------------- | ------ | ---------------------------------------------------- |
| `/v1/public/rankings` | GET    | Aggregated model usage stats (see `rankings.md`)     |
| `/llms.txt`           | GET    | Redirect to docs-domain agent docs (see `agents.md`) |

## 13. Reserved, not built

**Sign-off:** _unsigned_

`/v1/embeddings`, `/v1/batch`, realtime voice. Nothing else may squat on these paths.
