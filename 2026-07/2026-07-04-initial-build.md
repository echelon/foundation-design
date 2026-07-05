# Foundation — Initial Build Plan

**Status:** Living document. First draft 2026-07-04. This describes the data model, code
organization, tech choices, and API shape for an open-source model gateway (Fal / Replicate /
OpenRouter competitor). Iterate here.

> Naming note: "Foundation" is used as the project codename throughout. `api.example.dev` is a
> placeholder base URL.

---

## 1. Product surface (recap)

- Uniform API access to image / video / audio / mesh / splat models (and later LLMs) across many
  upstream providers (ByteDance, Google, OpenAI, Kling, xAI, Vidu, …).
- Three API shapes: granular (model+modality), unified per-model, and omni (per media type,
  model as a parameter).
- Polling and webhooks for job delivery. Stripe-style API versioning. Dollar-denominated billing
  (no credits). Full request/response debug logs exposed to customers as a product.

---

## 2. Model ID convention

### Requirements

1. Double-click-selects-the-whole-thing on Mac/Linux/Windows → charset must be `[a-z0-9_]` only.
   (Slashes, dashes, dots, and colons all break word-selection.)
2. Readable versions — avoid the `1p5` "p" hack.
3. Namespace/owner support without an ugly `__` separator.
4. Stable across modalities (modality is *not* part of the model ID; it's part of the endpoint
   or inferred from parameters).

### The scheme

```
{owner}_{family}[_v{major}[_{minor}]][_{variant}...]
```

- **Charset:** `^[a-z0-9_]+$`, single `_` as the only separator. The whole ID is always
  double-click copyable.
- **Version rule:** versions are introduced by a literal `v` and dots become underscores:
  `v3`, `v1_5`, `v3_1`. The `v` makes the version token self-identifying, so `kling_v2_5_turbo`
  is unambiguous (version `2.5`, variant `turbo`) without a parser needing to guess whether
  `2_5` is two qualifiers.
- **Models whose market name embeds no version** simply omit it: `nano_banana_pro`.
- **Owner-first is the canonical form.** The owner is the *model* owner, not the hosting
  provider (matching what Fal should have done):

| Canonical ID | Display name |
|---|---|
| `kling_video_v3_pro` | Kling 3.0 Pro |
| `kling_video_v2_5_turbo_pro` | Kling 2.5 Turbo Pro |
| `bytedance_seedance_v2` | Seedance 2.0 |
| `bytedance_seedance_v2_fast` | Seedance 2.0 Fast |
| `bytedance_seedream_v4_5` | Seedream 4.5 |
| `google_veo_v3_1_fast` | Veo 3.1 Fast |
| `google_nano_banana_pro` | Nano Banana Pro |
| `openai_gpt_image_v1_5` | GPT Image 1.5 |
| `xai_grok_imagine_v1_5` | Grok Imagine 1.5 |
| `vidu_q_v2` | Vidu Q2 |

- **Aliases are first-class.** An `model_aliases` table maps alternate slugs to the canonical
  model, so we can:
  - accept short forms (`seedance_v2` → `bytedance_seedance_v2`),
  - accept competitors' IDs during migration (`fal-ai/kling-video/v3/pro` → ours),
  - rename models without breaking customers (old slug becomes an alias).
  Jobs always record the *canonical* model ID they resolved to.
- **Structure lives in the catalog, not the string.** The DB stores `owner`, `family`,
  `version`, `variant` as separate columns; the slug is just the join of them. Nothing ever
  parses the slug at runtime.

### Rejected alternatives

- `owner/family/version` (Fal): not click-copyable; namespace convention decays anyway.
- `owner.family-v1.5`: dots/dashes break selection.
- `owner__family` double underscore: ugly, and people will typo one underscore.
- `1p5` for 1.5: works but reads badly; `v1_5` is no wider and reads as a version.

---

## 3. API design

### 3.1 Base URLs and auth

- `https://api.example.dev` — public API, auth via `Authorization: Bearer {api_key}`.
- Dashboard traffic hits the same API with cookie/session auth instead of a key (same endpoints,
  same job pipeline, `source=dashboard` recorded on the job).
- API keys: `sk_live_{crockford}` (and `sk_test_` if we ever do test mode).

### 3.2 The three API shapes

All three shapes create the **same Job resource** and return the same envelope. They differ only
in how the (model, modality, params) triple is expressed.

**Shape 1 — Granular (model + modality), Fal-style:**

```
POST /v1/models/{model_id}/{modality}
POST /v1/models/kling_video_v3_pro/text_to_video
POST /v1/models/kling_video_v3_pro/image_to_video
POST /v1/models/bytedance_seedance_v2/reference_to_video
```

Body = exactly the parameters that (model, modality) supports, validated against its JSON
Schema. This is the "I read the docs page for this one model and integrated it" API.

**Shape 2 — Unified per-model:**

```
POST /v1/models/{model_id}
POST /v1/models/kling_video_v3_pro
```

Body = the union of the model's parameters across modalities; we infer the modality from what's
present (`start_frame` present → image_to_video; `reference_images` → reference_to_video; else
text_to_video). Inference rules are published per model. The response records the modality we
resolved (`"resolved_modality": "image_to_video"`).

**Shape 3 — Omni (per media type, model as parameter):**

```
POST /v1/video          { "model": "kling_video_v3_pro", "prompt": ..., ... }
POST /v1/image          { "model": "google_nano_banana_pro", ... }
POST /v1/audio
POST /v1/mesh
POST /v1/splat
```

Body = the superset of parameters across all models of that media type. Irrelevant parameters
are pruned or coerced (upgraded/downgraded) per model — the ArtCraft omni approach. The
response includes a `param_adjustments` list describing every pruning/coercion so callers can
see what we did (this feeds the debug-logs product too).

**LLMs (later):** reserve `POST /v1/chat/completions` (OpenAI-compatible, held-open response,
SSE streaming via `"stream": true`). This is the industry-standard wire shape (what OpenRouter
speaks); no websockets except future realtime voice. LLM calls are synchronous-only and do not
create pollable jobs (but they do get request logs and a `generation` record for cost
accounting). Punt on implementation, not on the path reservation.

### 3.3 Job lifecycle endpoints (shared by all three shapes)

```
201 → { "id": "job_01H...", "status": "queued", ... }        # submit response

GET    /v1/jobs/{job_id}            # full job: status, params echo, output, costs
GET    /v1/jobs/{job_id}/status     # lightweight status-only (cheap to poll)
GET    /v1/jobs                     # list, filterable: status, model, time range, api key
POST   /v1/jobs/{job_id}/cancel
GET    /v1/jobs/{job_id}/logs       # the debug timeline (see §8)
```

Job status enum: `queued → dispatched → running → succeeded | failed | canceled | expired`.

**Sync mode:** `Prefer: wait=55` request header (Replicate-style) holds the connection open up
to N seconds and returns the finished job if it completes in time, else returns the queued job.
Good for fast image models; the default is always async.

**Webhooks:** any submit may include:

```json
"webhook": { "url": "https://customer.example/hook", "events": ["job.completed", "job.failed"] }
```

Deliveries are signed (`Foundation-Signature: t=...,v1=hmac_sha256(...)` — Stripe scheme) with a
per-account signing secret (`whsec_...`), retried with exponential backoff, and every attempt +
the customer's response is logged and shown in the dashboard (§8).

**Idempotency:** `Idempotency-Key` header, unique per account, 24h window; replay returns the
original response.

### 3.4 Catalog endpoints

```
GET /v1/models                       # filter by media_type, owner, modality, status
GET /v1/models/{model_id}            # metadata, modalities, pricing, JSON Schemas per modality
GET /v1/models/{model_id}/schema/{modality}
```

The per-modality JSON Schema is the machine-readable source of truth that renders the docs
pages, powers the sandbox, and validates requests. (Replicate does this via Cog+OpenAPI; it's
the right call.)

### 3.5 File handling

- Inputs: accept HTTPS URLs everywhere; also `POST /v1/files` (multipart or presigned) →
  `file_...` ID usable in any input slot. We mirror remote inputs to our own storage before
  dispatch so provider fetch failures are ours to debug.
- Outputs: we always ingest provider outputs into our object storage (S3/R2) and serve CDN URLs.
  Provider URLs expire (Replicate's delivery URLs, Kling's, etc.); ours are durable subject to a
  retention policy per account tier.

### 3.6 API versioning (Stripe model)

- Version = a date string: `2026-07-04`.
- Every account has a pinned `api_version` set at first API call/key creation.
- Per-request override: `Foundation-Version: 2026-09-01` header.
- Implementation: internally there is exactly ONE current request/response shape; older versions
  are implemented as **transform layers** (request upgraders / response downgraders) applied at
  the edge, exactly like Stripe. Version gates live in one module; each breaking change ships a
  dated transform + changelog entry.
- URL `/v1/` never changes unless the entire wire protocol is reinvented.

---

## 4. Technology choices

### 4.1 Postgres (recommended over MySQL)

Chosen for: `JSONB` + GIN indexes (the logs product is basically a JSONB firehose), native
partitioning (log tables), `FOR UPDATE SKIP LOCKED` (the SQL job queue), transactional DDL
(migrations can't half-apply), `ON CONFLICT` upserts, and — decisive for open source — it's what
self-hosters expect. Full MySQL-vs-Postgres notes and the gotchas list are in the companion
feedback doc (§ Postgres).

### 4.2 Axum (recommended over Actix)

Tokio-team maintained, tower middleware ecosystem, work-stealing multithreaded runtime,
`utoipa` works with it (so the ArtCraft-style OpenAPI annotations port). Handler porting from
Actix is mechanical. Full comparison in the feedback doc.

### 4.3 Queue: Postgres first

`queue_tasks` table worked with `FOR UPDATE SKIP LOCKED`, visibility timeouts, exponential
backoff. This comfortably handles thousands of jobs/sec with tuning (fillfactor, partial
indexes, aggressive autovacuum on the queue table). Swap candidates later: NATS JetStream
(preferred) or Redis Streams; the worker trait boundary (§6) keeps this swappable.

### 4.4 The rest

- Rust workspace (Cargo), sqlx (compile-checked queries) against Postgres, Redis for
  rate-limiting/caching/session hot path.
- Nx + TypeScript + React + Tailwind + Vite for all web apps.
- `async-stripe` for billing.
- Object storage: S3-compatible (R2 attractive: zero egress for a media-heavy product).

---

## 5. Money representation

**Ledger/balances: `i64` micro-USD** (1_000_000 micros = $1; 1 micro = 10⁻⁴ cent).

- Range: ±$9,223,372,036,854 (±$9.2 trillion) — billions in summed revenue is nowhere near the
  edge.
- Resolution: 10⁻⁴ cents covers the researched pricing floor. Verified examples: Fal
  `fal-ai/flux/schnell` $0.003/megapixel, `fal-ai/ddcolor` $0.001/MP, utility models
  ~$0.000575/compute-second; Replicate CPU-small $0.000025/second. Unit *rates* go to 6
  dollar-decimals; billed *charges* (rate × quantity) fit micro-USD after rounding.
- **Signed** (`i64`, not `u64`): refunds, adjustments, and transient negative balances must
  representable; unsigned money types force error-prone workarounds.

**Unit prices: `i64` nano-USD** (10⁻⁹ dollars) in the pricing table only. Rationale: per-unit
rates go far below a micro (e.g. an LLM at $0.15/1M tokens = 150 nano-USD/token; per-pixel or
per-token rates need headroom). Price math happens in nano-USD (or i128 intermediate), then the
**charge** is rounded half-up to micro-USD at settlement. Nano-USD i64 caps at ~$9.2B which is
fine for a *unit rate*, never for a balance.

Rust: newtypes `Micros(i64)` and `NanoUsd(i64)` with checked arithmetic; no raw i64s cross a
function boundary.

---

## 6. Monorepo layout

```
foundation/
├── Cargo.toml                      # [workspace]
├── rust-toolchain.toml
├── crates/
│   ├── bin/                        # deployable binaries (thin mains)
│   │   ├── api-server/             # public API + dashboard API (axum)
│   │   ├── worker/                 # queue consumer: dispatch/poll/webhook/ingest workers
│   │   └── admin-cli/              # migrations, catalog seeding, ops one-offs
│   ├── core/
│   │   ├── ids/                    # prefixed Crockford ID types (see §7.1)
│   │   ├── money/                  # Micros / NanoUsd newtypes
│   │   ├── enums/                  # job status, media type, modality, provider, …
│   │   ├── api-defs/               # request/response types, utoipa schemas, versioned transforms
│   │   ├── catalog/                # model registry domain types + JSON Schema handling
│   │   └── config/                 # env/config loading
│   ├── db/
│   │   ├── migrations/             # sqlx migrations
│   │   └── queries/                # one function per query, sqlx compile-checked
│   ├── providers/                  # third-party clients — pure HTTP, no DB deps
│   │   ├── provider-core/          # shared traits (see below), reqwest client factory,
│   │   │                           #   retry/backoff, error taxonomy
│   │   ├── bytedance/              # Seedance/Seedream via BytePlus + Volcengine
│   │   ├── google/                 # Gemini image (sync) + Veo (LRO poll)
│   │   ├── openai/                 # gpt-image (sync), Sora (poll)
│   │   ├── kling/                  # JWT auth, create+poll, callback
│   │   ├── xai/                    # Grok image (sync — punted), Grok Imagine video
│   │   └── vidu/                   # create+poll, callback
│   ├── router/                     # omni param translation: canonical request → provider
│   │   │                           #   request; prune/coerce rules; modality inference
│   ├── services/                   # business logic, DB-aware
│   │   ├── jobs/                   # job state machine
│   │   ├── billing/                # ledger, holds, auto-refill, spend limits
│   │   ├── auth/                   # sessions, api keys, invitations
│   │   ├── webhooks/               # outbound delivery + signing
│   │   └── logging/                # the debug-log product write path
│   └── queue/                      # queue_tasks abstraction (Postgres impl now, trait for later)
├── frontend/                       # Nx workspace
│   ├── apps/
│   │   ├── www/                    # marketing site
│   │   ├── docs/                   # docs + per-model pages + sandbox
│   │   └── console/                # dashboard: keys, billing, jobs, logs, team
│   └── libs/
│       ├── ui/                     # shared Tailwind components
│       ├── api-client/             # TS client generated from OpenAPI
│       └── model-forms/            # JSON Schema → form renderer (sandbox + docs share it)
├── external/plan/                  # these documents
├── docs/                           # internal engineering docs
└── infra/                          # docker-compose (pg, redis, minio), CI, deploy
```

**Provider trait sketch** (the seam everything hangs on):

```rust
trait ProviderAdapter {
    async fn dispatch(&self, req: CanonicalRequest, creds: &Creds) -> Result<DispatchReceipt>;   // returns provider job id (or inline result for sync providers)
    async fn poll(&self, receipt: &DispatchReceipt, creds: &Creds) -> Result<PollOutcome>;        // Pending | Succeeded(outputs) | Failed(err)
    fn parse_callback(&self, body: &[u8], headers: &HeaderMap) -> Result<CallbackOutcome>;        // for providers that push (Kling/Vidu callbacks)
}
```

Sync providers (OpenAI images, Gemini image) implement `dispatch` returning the finished result;
poll providers (Seedance, Kling, Veo, Sora, Vidu) return a receipt the poll worker chews on.
Grok's synchronous-image weirdness fits `dispatch`-returns-result too (punted for now).

Existing ArtCraft clients (Grok bindings etc.) get adapted into `crates/providers/*` — inventory
of what's reusable is being compiled into the feedback doc.

---

## 7. Database schema

Conventions first, then tables. Postgres. All timestamps `timestamptz`.

### 7.1 Identifiers

Every table gets **both**:

- `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY` — internal, used for all FKs and joins
  (compact indexes, friendly to replication tooling).
- `public_id TEXT NOT NULL UNIQUE` — `{prefix}_{26-char Crockford-encoded 128-bit}` — the ONLY
  identifier that ever leaves the system. Generated app-side (ULID-style: time-ordered high
  bits so they index well).

Prefix registry (single Rust enum, compile-time checked, one prefix per table):

```
acct_  account          user_  user             mem_    membership
inv_   invitation       sess_  session          key_    api key (secret: sk_live_...)
prov_  provider         model_ model            alias_  model alias
mep_   model endpoint   price_ price            cred_   provider credential
job_   job              jevt_  job event        out_    job output/artifact
file_  uploaded file    task_  queue task       rlog_   request log
pcall_ provider call    whd_   webhook delivery whsec_  webhook secret
cust_  stripe customer  txn_   ledger entry     hold_   billing hold
refill_ auto-refill rule limit_ spend limit     sevt_   stripe event
audit_ audit log        idem_  idempotency key
```

### 7.2 Identity & access

```
accounts            id, public_id, name, slug, billing_email,
                    api_version DATE NOT NULL,          -- Stripe-style pin
                    status (active|suspended|closed), created_at, updated_at

users               id, public_id, email UNIQUE, email_verified_at, password_hash (argon2id),
                    display_name, totp_secret NULL, created_at, updated_at, deleted_at

account_members     id, public_id, account_id FK, user_id FK, role (admin|member),
                    UNIQUE(account_id, user_id)

invitations         id, public_id, account_id FK, email, role, token_hash, invited_by FK users,
                    expires_at, accepted_at NULL, accepted_by NULL

sessions            id, public_id, user_id FK, token_hash, ip, user_agent, expires_at,
                    revoked_at NULL                      -- dashboard cookie auth

api_keys            id, public_id, account_id FK, created_by FK users, name,
                    key_prefix TEXT,                     -- first 12 chars, shown in UI
                    key_hash TEXT,                       -- sha256 of full secret
                    last_used_at, expires_at NULL,       -- NULL = no expiry
                    revoked_at NULL, created_at
                    -- scopes JSONB reserved for later; unscoped for now
```

Sign-up creates a user AND an account with that user as admin (solo user == account of one).
Every resource hangs off `account_id`, never `user_id` — teams are day-one, not a retrofit.

### 7.3 Catalog

```
providers           id, public_id, slug (bytedance|google|openai|kling|xai|vidu|…),
                    adapter (enum matching the Rust adapter), status

provider_credentials id, public_id, provider_id FK, name,
                    secret_ref TEXT,                     -- pointer into secret store/env, NOT the secret
                    weight INT, status (active|drained|disabled)
                    -- multiple upstream accounts per provider (ArtCraft's
                    -- Volcengine/BytePlus/Ultra split, generalized)

models              id, public_id, slug UNIQUE (canonical, §2), owner, family,
                    version_major NULL, version_minor NULL, variant NULL,
                    display_name, media_type (image|video|audio|mesh|splat|text),
                    description, status (preview|active|deprecated|retired),
                    metadata JSONB (sample media, docs blurbs), created_at, updated_at

model_aliases       id, alias_slug UNIQUE, model_id FK, kind (short|legacy|competitor|rename)

model_endpoints     id, public_id, model_id FK, modality
                    (text_to_image|image_to_image|text_to_video|image_to_video|
                     reference_to_video|video_to_video|text_to_audio|…),
                    input_schema JSONB,                  -- JSON Schema: validation + docs + sandbox forms
                    output_schema JSONB, defaults JSONB,
                    status, UNIQUE(model_id, modality)

model_routes        id, model_endpoint_id FK, provider_id FK, credential pool selector,
                    provider_model_id TEXT,              -- upstream's name for it
                    priority INT, enabled BOOL
                    -- one endpoint can route to multiple upstreams; router picks by
                    -- priority/health

prices              id, public_id, model_endpoint_id FK,
                    unit (per_call|per_image|per_video_second|per_megapixel|per_second_compute|
                          per_million_tokens_in|per_million_tokens_out|…),
                    unit_amount_nanos BIGINT,            -- nano-USD per unit (§5)
                    modifiers JSONB,                     -- e.g. resolution/duration multipliers
                    effective_at, superseded_at NULL     -- append-only; jobs reference the row
                                                         -- that priced them
```

### 7.4 Jobs & artifacts

```
jobs                id, public_id, account_id FK, api_key_id FK NULL,  -- NULL = dashboard session
                    user_id FK NULL,                     -- the human, when known
                    source (api|dashboard),
                    model_id FK, model_endpoint_id FK,   -- endpoint = resolved modality
                    api_shape (granular|unified|omni),   -- which front door was used
                    request_params JSONB,                -- canonicalized post-coercion params
                    param_adjustments JSONB,             -- what omni pruned/coerced (§3.2)
                    status (queued|dispatched|running|succeeded|failed|canceled|expired),
                    failure_code NULL, failure_message NULL,
                    provider_id FK NULL, provider_credential_id FK NULL,
                    provider_job_id TEXT NULL,           -- upstream's id, indexed
                    price_id FK, quoted_amount_micros BIGINT, final_amount_micros BIGINT NULL,
                    hold_id FK NULL,
                    idempotency_key TEXT NULL, UNIQUE(account_id, idempotency_key),
                    api_version DATE,                    -- version the request came in on
                    webhook_url TEXT NULL, webhook_events TEXT[],
                    debug_event_id,                      -- correlates all logs (§8)
                    created_at, dispatched_at, started_at, completed_at, expires_at

job_events          id, job_id FK, event_type (created|dispatched|provider_accepted|poll_tick|
                    provider_completed|output_ingested|webhook_sent|failed|…),
                    data JSONB, created_at
                    -- append-only state-machine trail; feeds both ops and the customer
                    -- debug timeline

job_outputs         id, public_id, job_id FK, output_index INT,
                    kind (image|video|audio|mesh|splat|thumbnail|text),
                    storage_key TEXT, cdn_url TEXT, mime, bytes BIGINT,
                    width NULL, height NULL, duration_ms NULL, checksum_sha256,
                    provider_url TEXT NULL,              -- original (expiring) upstream URL
                    created_at

files               id, public_id, account_id FK, purpose (input|avatar|…),
                    storage_key, cdn_url, mime, bytes, checksum_sha256,
                    origin (upload|mirrored_url), origin_url NULL, created_at
```

### 7.5 Queue (SQL-first)

```
queue_tasks         id, public_id, task_type (dispatch_job|poll_provider|ingest_outputs|
                    deliver_webhook|settle_billing|refill_balance|…),
                    payload JSONB, run_after timestamptz, priority INT,
                    attempts INT, max_attempts INT,
                    locked_by TEXT NULL, locked_at NULL,  -- visibility timeout
                    status (pending|running|done|dead), last_error NULL, created_at

-- worker loop:  SELECT … WHERE status='pending' AND run_after <= now()
--               ORDER BY priority, run_after
--               FOR UPDATE SKIP LOCKED LIMIT $n;
-- fillfactor lowered + aggressive autovacuum on this table.
```

### 7.6 Billing

```
stripe_customers    id, public_id, account_id FK UNIQUE, stripe_customer_id UNIQUE

payment_methods     id, public_id, account_id FK, stripe_payment_method_id,
                    brand, last4, exp, is_default, status, created_at

ledger_entries      id, public_id, account_id FK,
                    amount_micros BIGINT,                -- signed: + credit, − debit
                    entry_type (topup|job_charge|job_refund|adjustment|promo|auto_refill),
                    job_id FK NULL, stripe_payment_intent_id NULL,
                    balance_after_micros BIGINT,         -- written under the account row lock
                    memo, idempotency_key UNIQUE NULL, created_at
                    -- APPEND-ONLY. Never UPDATE/DELETE. The balance cache must always
                    -- equal SUM(amount_micros); a reconciliation task asserts this.

account_balances    account_id PK/FK, balance_micros BIGINT, reserved_micros BIGINT,
                    updated_at
                    -- mutated only in the same txn as a ledger insert / hold change,
                    -- via SELECT … FOR UPDATE on this row

holds               id, public_id, account_id FK, job_id FK, amount_micros,
                    status (held|captured|released), created_at, resolved_at
                    -- submit: hold quoted cost (402 if balance - reserved < quote)
                    -- success: capture (ledger job_charge for final, release remainder)
                    -- failure: release

auto_refill_rules   id, public_id, account_id FK, enabled,
                    threshold_micros, refill_amount_micros,
                    payment_method_id FK, daily_max_micros,   -- runaway-card safety
                    last_triggered_at

spend_limits        id, public_id, account_id FK, window (weekly|monthly),
                    limit_micros, behavior (block), created_by FK users
                    -- enforcement: rolling window SUM over ledger job_charges + active holds

stripe_events       id, stripe_event_id UNIQUE, event_type, payload JSONB,
                    processed_at NULL, error NULL        -- webhook dedupe + replay
```

**The 402 path:** admission check = `balance_micros − reserved_micros ≥ quote` under the
account-balance row lock; fail → `402 Payment Required` with a machine-readable body
(`insufficient_balance`, current balance, quote, top-up link).

### 7.7 The logs product (§8 has the product spec)

```
request_logs        id, public_id, account_id FK, api_key_id NULL, job_id NULL,
                    method, path, api_version, ip, user_agent,
                    request_headers JSONB (allowlisted), request_body JSONB (redacted),
                    response_status, response_body JSONB, latency_ms, created_at
                    -- PARTITIONED BY RANGE (created_at), monthly; retention policy per tier

provider_calls      id, public_id, job_id FK, provider_id FK,
                    direction (request|response|poll_request|poll_response|callback_in),
                    url, request_headers JSONB (secrets stripped), request_body JSONB,
                    response_status NULL, response_body JSONB, latency_ms, created_at
                    -- also partitioned monthly

webhook_deliveries  id, public_id, job_id FK, account_id FK, url, event_type,
                    payload JSONB, attempt INT, request_headers JSONB,
                    response_status NULL, response_body TEXT (truncated), latency_ms,
                    next_retry_at NULL, delivered_at NULL, gave_up_at NULL, created_at

webhook_secrets     id, public_id, account_id FK, secret (whsec_..., encrypted at rest),
                    created_at, revoked_at NULL          -- rotation = two active secrets

audit_logs          id, public_id, account_id FK, actor_user_id FK, action,
                    target_public_id, data JSONB, ip, created_at
                    -- key created/revoked, member invited/removed, limits changed, …

idempotency_keys    id, account_id FK, key, request_hash, response_snapshot JSONB,
                    created_at, expires_at, UNIQUE(account_id, key)
```

### 7.8 Schema notes

- **Everything the customer sees hangs off `account_id`** and every list query starts with it —
  the natural shard key if we ever need one.
- Log tables are the write-volume monsters: partitioned, minimal indexes,
  `(account_id, created_at)` + `(job_id)` only. Redaction happens at write time
  (auth headers, our provider keys), not read time.
- `job_events` vs `provider_calls`: events are the cheap state-machine trail; provider_calls
  hold full payloads. The customer debug timeline (§8) is a merge of both.
- The ledger is the only source of truth for money; `account_balances` is a cache with an
  invariant checker. This is the part I'd be most careful with — every mutation is
  ledger-insert + balance-update in one transaction under `FOR UPDATE`.

---

## 8. User-facing logs as a product

Everything correlated by `job_id` (and `debug_event_id` for pre-job failures):

1. inbound request (as received, redacted) — `request_logs`
2. canonicalized params + omni adjustments — on the job row
3. outbound provider request — `provider_calls(direction=request)`
4. provider response / every poll / inbound provider callback — `provider_calls`
5. our webhook deliveries to the customer + their responses — `webhook_deliveries`
6. state transitions — `job_events`

API: `GET /v1/jobs/{job_id}/logs` returns the merged, time-ordered timeline. Console renders it
as a request-trace view (the "why was my video weird / slow / failed" answer machine). This is
genuinely differentiating — Fal/Replicate expose almost none of this.

Privacy note for the OSS story: our *own* provider credentials are stripped before write;
customer payloads are their own data, retained per account policy.

---

## 9. Websites

Three Nx apps, one design system:

1. **`www`** — marketing. Landing, pricing, blog.
2. **`docs`** — the model catalog + docs + sandbox:
   - `/models/{model_id}` — one page per model: description, sample media, pricing table,
     per-modality tabs (rendered from `model_endpoints.input_schema`), runnable sandbox
     (cookie-auth, hits the same API), code samples (curl/TS/Python) generated from the schema.
   - `/generate/image`, `/generate/video`, … — the *media-type* docs: the omni API. This is
     the bet: engineers integrate "video gen" once and swap model IDs. These pages get a
     model-comparison matrix (params supported, price, speed) and the omni schema.
3. **`console`** — dashboard: API keys, billing (balance, top-up, auto-refill, limits), job
   history + debug timelines, team management, account settings.

Sandbox and docs forms are both rendered from the same JSON Schemas — build `model-forms` once.

---

## 10. Provider integration matrix

Verified against official docs 2026-07-04 (full endpoint/shape details in the feedback doc §6
and the research it cites). The lifecycle column drives which adapter methods each client
implements.

| Provider | Models | Auth | Lifecycle | Asset expiry | Notes |
|---|---|---|---|---|---|
| ByteDance — BytePlus/Volcengine Ark | Seedance video | `Bearer` key | `POST /api/v3/contents/generations/tasks` → poll | ~24h (unverified) | Intl vs CN are disjoint deployments with different hosts AND model IDs (`dreamina-*` vs `doubao-*`) → `provider_credentials` pools; no callback support |
| ByteDance — same platform | Seedream image | `Bearer` key | **sync** (`/api/v3/images/generations`, OpenAI-like) | — | |
| Google (Gemini API) | Nano Banana / Pro image (`gemini-3-pro-image` confirmed) | `x-goog-api-key` | **sync** | inline b64 | Google steering image gen to new Interactions API (`POST /v1beta/interactions`) — support both |
| Google (Gemini API) | Veo video | `x-goog-api-key` | `:predictLongRunning` → poll operation | **2 days** | Vertex AI variant differs: OAuth + `:fetchPredictOperation` POST |
| OpenAI | gpt-image | `Bearer` key | **sync** (b64 only) | — | edits via multipart `/v1/images/edits` |
| OpenAI | Sora video | `Bearer` key | `POST /v1/videos` → poll, + webhooks (`video.completed/failed`) | ~1–24h | **Deprecated: shutdown 2026-09-24** — build the adapter expecting a successor |
| Kling | Kling video + image | **self-signed HS256 JWT** from AK/SK, 30-min exp | create → poll, optional `callback_url`, `external_task_id` | **30 days** | JWT cache with TTL; envelope `{code,message,request_id,data}`; docs portal blocks scrapers — verify fields in browser |
| xAI | Grok image | `Bearer` key | **sync** (OpenAI-style) | temp URLs | punted initially, but official API is clean |
| xAI | Grok Imagine video | `Bearer` key | deferred: POST → `request_id` → `GET /v1/videos/{id}` (`pending/done/failed/expired`) | temp URLs | ArtCraft `grok_api_client` adaptable |
| Vidu | Vidu Q video | **`Token`** scheme (not Bearer) | `POST /ent/v2/{modality}` → `GET /ent/v2/tasks/{id}/creations`, optional callback | ~1h (unverified) | `off_peak` discount option worth exposing |

Rules baked into the adapter layer:

- Adapters implementing `parse_callback` (Kling, Vidu, OpenAI Sora) still ALSO get polled —
  callbacks are an optimization, never the only completion path (they get lost; we reconcile
  by poll).
- Asset expiry ranges from ~1 hour to 30 days → **ingest every output into our storage
  immediately on completion**, before marking the job succeeded.
- Model churn is the steady state (two launch-window providers have shutdown cliffs:
  Sora-2 API 2026-09-24, Google Imagen 2026-08-17) — catalog `status` + alias machinery are
  load-bearing, not nice-to-haves.

**ArtCraft reuse:** adapt `grok_api_client`, the Fal client's polling/webhook subsystems, the
`artcraft_router` param-translation patterns (→ our `router` crate), the poller-worker skeleton
from `service/job/*_job`, and the four-bucket error taxonomy
(`Client`/`ApiBadRequest`/`ApiSpecific`/`ApiGeneric` + `is_having_downtime_issues()`).
Official-API clients only need `reqwest` (the `wreq` TLS-fingerprint fork is only for the
consumer-surface clients we're not carrying over).

---

## 11. Build order (suggested)

1. **Skeleton**: workspace, `ids`, `money`, `enums`, migrations for identity + catalog; axum
   server with health + auth (sessions, api keys).
2. **One model end-to-end**: catalog seed, omni `POST /v1/image` → one sync provider (OpenAI
   gpt-image is simplest — no polling), job row, output ingestion, `GET /v1/jobs/{id}`.
   Billing stubbed as ledger writes with fake prices.
3. **Async pipeline**: queue_tasks + worker, one poll provider (Seedance or Kling), webhooks
   out, the logs timeline.
4. **Billing for real**: Stripe top-ups, holds, 402, auto-refill, limits.
5. **Granular + unified shapes** (they're routing sugar over the same pipeline) + JSON Schema
   validation + docs generation.
6. **Console + docs sites.** Then more providers.

---

## Open questions (for us to resolve in iteration)

- Project/product name (affects header names like `Foundation-Version`, key prefixes).
- R2 vs S3; single region to start?
- Retention windows per tier for logs and stored media (cost driver).
- Do dashboard sandbox runs bill identically to API runs? (Proposal: yes, same ledger, maybe
  small free monthly allowance per account.)
- LLM timeline: reserve paths now (done), implement after media APIs are stable?
