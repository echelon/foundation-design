# Design Feedback & Answers — 2026-07-04

Companion to `2026-07-04-initial-build.md` (which holds the actual decisions). This doc answers
the questions you asked and records the competitive research that informed the decisions.
Research was verified against official docs on 2026-07-04; items competitors' docs blocked or
render client-side are flagged inline.

---

## 1. Competitor API previews

### FAL

- **Model identity:** hierarchical endpoint IDs in the URL path: `{owner}/{model}[/{version}][/{variant}][/{modality}]`, e.g. `fal-ai/kling-video/v1.5/pro/image-to-video`. Modalities are separate endpoints with separate schemas, queues, and prices.
- **Sync:** `POST https://fal.run/{endpoint_id}` — held-open response, no retries, prototyping only.
- **Queue (the real API):** `POST https://queue.fal.run/{endpoint_id}` → `{request_id, response_url, status_url, cancel_url, queue_position}`. Poll `GET .../requests/{id}/status` (`IN_QUEUE`/`IN_PROGRESS`/`COMPLETED`, `?logs=1` for logs), SSE variant at `.../status/stream`, result at `GET .../requests/{id}`, cancel via `PUT .../cancel`. Queue is per-endpoint and **shared across all customers**.
- **Webhooks:** per-request query param `?fal_webhook=https://...`. Signed with **ED25519** against a JWKS endpoint (not HMAC). 15s delivery timeout, 10 retries over 2 hours.
- **Auth:** `Authorization: Key $FAL_KEY` (literal scheme `Key`). Key scopes: API vs ADMIN.
- **Files:** outputs on `fal.media` CDN; default retention ≥7 days, configurable per-request via `X-Fal-Object-Lifecycle-Preference` header. Upload flow: token from `rest.alpha.fal.ai` then POST to CDN.
- **Versioning:** none on the inference plane — no `/v1/`, no version header. Version changes = new endpoint ID. Their management-plane APIs (`api.fal.ai/v1/models/pricing`) are versioned.
- **Model pages:** `fal.ai/models/{endpoint_id}` with Playground / API / Examples tabs, generated from the endpoint's schema; pricing rendered on-page.

### Replicate

- **Model identity:** `{owner}/{name}` plus immutable 64-hex **version IDs** minted on every `cog push`. Since 2025-08, `POST /v1/predictions` accepts `{owner}/{name}`, `{owner}/{name}:{version}`, or bare version — one endpoint runs everything.
- **Lifecycle:** create prediction → response has `id`, `status`, `urls: {get, cancel, stream, web}`. Poll `GET /v1/predictions/{id}`; statuses `starting/processing/succeeded/failed/canceled`; cancel endpoint.
- **Sync:** `Prefer: wait[=N]` header holds the response open up to N seconds; on timeout returns the incomplete prediction and you fall back to polling. (We're adopting this.)
- **Webhooks:** per-request body fields `webhook` + `webhook_events_filter` (`start`, `output`, `logs`, `completed`; output/logs throttled to 1/500ms). Signing is Svix-style HMAC-SHA256 (`webhook-id`/`webhook-timestamp`/`webhook-signature` headers, `whsec_` secret fetched from `GET /v1/webhooks/default/secret`).
- **Schemas:** every version carries a full **OpenAPI schema** (`components.schemas.Input/Output`, `x-order` for form ordering) — this generates their docs and playground. (We're adopting the concept via JSON Schema per model endpoint.)
- **Files:** outputs on `replicate.delivery`, **expire after 1 hour**; API predictions are deleted after ~1 hour. Input uploads via `POST /v1/files` expire in 24h.
- **Pricing:** per-second hardware for community models (CPU $0.0001/s … H100 $0.001525/s), per-output for official models (FLUX Dev $0.025/image, Wan 2.1 $0.09/video-second).
- **Deployments:** private named endpoints pinning a version + hardware + autoscaling.

### Kie.ai

- Aggregator/reseller (claims 30–50% below official prices), credits-based.
- **Unified jobs API** (their new pattern — closest to our omni shape): `POST /api/v1/jobs/createTask` with `{"model": "kling-2.6/text-to-video", "input": {...}, "callBackUrl": "..."}` → `taskId`; poll `GET /api/v1/jobs/recordInfo?taskId=...` → `state: waiting|queuing|generating|success|fail`, `resultJson` with result URLs. Optional webhook via `callBackUrl`.
- Legacy per-product endpoints still coexist (Veo, Suno) — a cautionary tale about API sprawl.
- Result URLs expire in 24h. Auth `Bearer`. Model IDs are loose `family/task` slugs, inconsistently applied.

### OpenRouter (the LLM gateway)

- Base `https://openrouter.ai/api/v1`. Core: `POST /chat/completions` (OpenAI-compatible), `POST /completions` (legacy), `GET /models` (catalog with pricing + supported params), `GET /generation?id=` (post-hoc cost/token stats per generation), `GET /key` + `/keys` (key info + provisioning API), `GET /credits`, OAuth PKCE endpoints so third-party apps can mint user-scoped keys.
- **OpenAI-compatible plus extensions:** `models` (fallback array), `route: "fallback"`, `provider` routing preferences (`order`, `only`, `ignore`, `sort: price|throughput|latency`, `require_parameters`, `max_price`, `zdr`, `data_collection: deny`…), `transforms: ["middle-out"]` prompt compression, `plugins`, extra sampling params (`top_k`, `min_p`, `repetition_penalty`, `top_a`).
- **Model naming:** `author/slug` (`anthropic/claude-3.5-sonnet`) + a permanent `canonical_slug` + auto-resolving aliases. Variants via `:suffix` — `:free`, `:nitro` (throughput-sorted), `:floor` (price-sorted), `:exacto`, `:extended`, `:thinking` (`:online` deprecated). Versions live inside the slug.
- **Delivery:** everything is HTTP POST with the response held open; streaming is SSE on the same POST (keep-alive comments, usage in final chunk, mid-stream errors as SSE events under an already-200 response). **No websockets, no webhooks anywhere.** One exception: their new **video generation API is job-based** (`POST /api/v1/videos` → job id → poll) — even the LLM gateway went async for media.
- **Billing:** prepaid credits; inference at provider passthrough price with **no markup** — the fee is 5.5% on credit purchase. BYOK supported at 5% of the would-be cost. Billed on native tokenizer counts.
- **Attribution:** `HTTP-Referer` + `X-OpenRouter-Title` headers power public app leaderboards — clever growth loop worth copying eventually.
- Now multi-modal: unified image API, video jobs API, STT/TTS endpoints.

### AtlasCloud

- "Full-modal" gateway (~400+ models: LLMs + image + video + audio + 3D) + serverless/reserved GPU tiers.
- Base `https://api.atlascloud.ai/v1`; chat is OpenAI-compatible (drop-in base-URL swap). Model IDs `provider/model` (`moonshotai/kimi-k2.6`).
- Media = async two-step: submit → prediction id → **poll only (~2s), deliberately no webhooks**.
- Notable: ships an MCP server + "Skills" so agents can call any hosted model — gateway-as-agent-tool positioning.
- Pricing: pay-as-you-go, images from ~$0.003, video $0.01–0.49/sec.

### Takeaways we baked into the design

1. Everyone converges on: opaque request/job ID, per-request webhook registration, poll + SSE, short-lived CDN URLs, schema-generated docs/playgrounds. Our plan follows the convergent shape and differentiates on logs + durable asset hosting + the omni API.
2. Fal's shared per-endpoint queues and 1-hour Replicate URL expiry are real pain points → our jobs are per-account and we ingest all outputs into our own storage.
3. Kie proves the omni/`createTask` pattern works commercially but their split-brain API (unified + legacy per-product) shows why we should launch all three shapes over ONE job pipeline from day one.
4. OpenRouter's `:variant` suffixes and passthrough-pricing-plus-purchase-fee model are both worth stealing when we do LLMs.

---

## 2. LLM APIs: how they actually deliver (your question)

**They are not websocket-based.** The industry standard:

- **Non-streaming:** HTTP POST, response held open until generation completes (seconds).
- **Streaming:** SSE over that same POST (`stream: true`, `data:` chunks, `[DONE]` sentinel). OpenAI, Anthropic, and OpenRouter all do exactly this.
- **WebSockets:** only for realtime voice/multimodal (OpenAI Realtime — WebRTC/WebSocket/SIP; Gemini Live — WebSocket). Not relevant to a v1 gateway.
- **Batch APIs:** fire-and-poll (Anthropic Message Batches; OpenAI Batch adds a completion webhook).
- **No polling, no webhooks** in normal LLM serving — the request/response cycle is short enough not to need job infrastructure.

**Recommendation:** punt on LLMs for launch, but (a) reserve `POST /v1/chat/completions` and never use that path for anything else, (b) when we build it, be strictly OpenAI-compatible like OpenRouter/AtlasCloud — it's table stakes (every SDK works via base-URL swap), and (c) LLM calls bypass the job queue entirely (synchronous pass-through with SSE piping) but still write request logs and a cost-accounting record. Media jobs and LLM generations are different enough that forcing them into one pipeline would hurt both.

For media delivery (your other half of the question): poll + webhook per job, as planned. Every competitor does per-request webhook registration rather than account-level endpoints; we support both a per-request `webhook.url` and (later) account-level default endpoints.

---

## 3. Postgres vs MySQL

**Recommendation: Postgres.** Reasons specific to this product:

1. **JSONB + GIN indexes** — the logs product stores every request/response body as JSON and needs to query into them. MySQL's JSON is serviceable; JSONB is better and the OSS ecosystem around it is deeper.
2. **Native table partitioning** that's actually pleasant — the log tables need monthly partitions from day one.
3. **`FOR UPDATE SKIP LOCKED`** — both DBs have it now (MySQL 8.0+ too), but the Postgres job-queue pattern is extremely well-trodden (this is how everyone builds SQL queues).
4. **Transactional DDL** — a failed migration rolls back instead of leaving half-applied DDL. Big deal for an OSS project where strangers run your migrations.
5. **Open-source expectations** — self-hosters expect Postgres; it's the default in every modern hosting stack (Supabase/Neon/RDS/Fly).

### Your MySQL knowledge that transfers directly

- `SELECT ... FOR UPDATE` works identically (plus `NOWAIT`, `SKIP LOCKED`).
- `FOR SHARE` ≈ `LOCK IN SHARE MODE`.
- Row-level locking via MVCC, no lock escalation, deadlock detection with automatic victim abort — all familiar.
- Upsert: `INSERT ... ON CONFLICT (col) DO UPDATE` — cleaner than `ON DUPLICATE KEY UPDATE` (you name the conflict target explicitly; no auto-increment burning quirk).

### Postgres gotchas (the list you asked for)

1. **VACUUM and bloat.** UPDATE writes a whole new row version; dead tuples pile up until (auto)vacuum reclaims them. Hot-update tables — our `queue_tasks`, `account_balances` — bloat if autovacuum can't keep up. Mitigations: per-table aggressive autovacuum settings, lower `fillfactor` (enables HOT updates that skip index maintenance), keep transactions short. This is THE operational difference from InnoDB (which does undo-log MVCC + purge instead).
2. **Long-running / idle-in-transaction sessions block vacuum** cluster-wide (vacuum can't remove tuples any open snapshot might see). A forgotten `BEGIN` in a psql session degrades the whole DB over hours. Set `idle_in_transaction_session_timeout`.
3. **XID wraparound.** Transaction IDs are 32-bit; anti-wraparound autovacuum ("freeze") MUST be allowed to run. Monitoring `datfrozenxid` age is standard ops hygiene. Neglecting it is the classic Postgres outage.
4. **Connections are processes**, not threads — heavier than MySQL. Hundreds of direct connections hurt. Plan on PgBouncer (transaction pooling) or keep sqlx pools modest per service. (sqlx + a few services = fine; just don't give every worker 100 connections.)
5. **No clustered/index-organized tables.** InnoDB stores rows in PK order; Postgres heaps don't. Range scans over the PK aren't automatically local on disk. Our time-ordered ULID-style IDs keep B-tree locality for inserts, which is most of what matters.
6. **Default isolation is READ COMMITTED** (MySQL: REPEATABLE READ). Fewer surprise gap-lock deadlocks (Postgres has no next-key locks at all), but you can hit write skew where InnoDB's gap locks would have saved you. For our money paths we don't rely on isolation subtleties: explicit `FOR UPDATE` on the balance row, always.
7. **Replication slots can eat your disk.** Physical streaming replication + logical replication both hold WAL for disconnected consumers; a dead replica with a slot pins WAL forever until the primary's disk fills. Set `max_slot_wal_keep_size`. Failover isn't built-in — you run Patroni or use a managed provider (recommendation: managed Postgres until it hurts).
8. **DDL takes ACCESS EXCLUSIVE locks and queues behind long queries** — and everything else then queues behind *it*. Always set `lock_timeout` in migrations. Also: `CREATE INDEX CONCURRENTLY` (can't run inside a transaction), `ADD COLUMN ... DEFAULT` is instant (PG11+), but `ALTER COLUMN TYPE` usually rewrites the table.
9. **No optimizer hints** (philosophical stance). Occasionally you restructure a query to coax the planner instead of dropping a hint. Prepared statements can flip to "generic plans" that are worse — sqlx users occasionally hit this; `plan_cache_mode` exists.
10. **`count(*)` is a full scan** (visibility must be checked per-row). Use `reltuples` estimates or counter caches for dashboard counts. (InnoDB is also O(n) but people expect this one.)
11. Small stuff: use `text` everywhere (`varchar(n)` buys nothing), use `timestamptz` never `timestamp`, unquoted identifiers fold to lowercase, sequences don't roll back (gaps — same as auto_increment), `NUMERIC` is arbitrary-precision but slow (we use i64 micros instead).

**Verdict:** the gotchas are real but they're all *operational* (vacuum, connections, WAL) rather than *semantic* — your locking/transaction instincts transfer. The features we gain (JSONB, partitioning, transactional DDL) map directly onto this product's needs.

---

## 4. Actix vs Axum

**Recommendation: Axum.**

**Threading model — the question you asked:**
- **Actix-web:** spawns N workers (default = cores), each a **single-threaded** Tokio runtime with its own accept loop and reactor. No work-stealing across workers: a slow task on worker 3 can't be picked up by idle worker 5. Upside: handlers/state don't need `Send`; slightly better tail latency in some microbenchmarks from cache locality.
- **Axum:** a thin router over hyper + tower running on the standard **multi-threaded, work-stealing** Tokio runtime. All connections/tasks share one scheduler; a blocked-ish worker's queue gets stolen. Handlers must be `Send + 'static`. For a workload like ours — lots of concurrent I/O-bound requests with occasional slow ones (provider calls, `Prefer: wait` holds) — work-stealing is the better default and removes a whole class of "one worker got unlucky" issues.
- Both are "multithreading" not "multiprocessing" — neither forks processes; you run multiple instances behind a load balancer for that (and we will anyway).

**Support/evolution:** Axum is maintained by the Tokio team, sits on tower (timeouts, rate limiting, tracing, compression as reusable `Layer`s shared with the entire ecosystem — tonic for gRPC uses the same middleware), and is where new-project momentum overwhelmingly is. Actix-web is mature, fast, and actively maintained, but evolves more slowly and its middleware/extractor ecosystem is its own island. `utoipa` (which ArtCraft uses for OpenAPI annotations) supports both, so the doc-generation approach ports.

**Which do I know better:** both well; Axum better on current idioms (it changes less and surprises less — Actix's `web::Data`/app-factory/runtime nuances are where subtle bugs live).

**The honest counterargument:** ArtCraft handlers are Actix, and you know Actix. If we were lifting whole handler files, Actix would save real time. But the handlers here will be substantially new (different auth, billing, job model) — the ported parts are the *clients* and *pollers*, which are framework-agnostic. Porting handler idioms Actix→Axum is mechanical (extractors map 1:1). Greenfield → Axum.

---

## 5. Money type (research-backed)

**Researched pricing floor:** Fal's published unit prices bottom out around **$0.000575/compute-second** (esrgan, photomaker) with per-MP prices at $0.001 (ddcolor) and $0.003 (flux/schnell); Replicate's cheapest is **$0.000025/second** (CPU-small). Both use 6 decimal places of a dollar for *unit rates*, and billed line items are rate × duration — routinely sub-cent, occasionally sub-millicent.

**Decision (details in build doc §5):**
- **Ledger, balances, charges: `i64` micro-USD** (10⁻⁶ dollars = 10⁻⁴ cents). Range ±$9.2 trillion — summing billions in revenue is 3 orders of magnitude inside the safe zone. Resolution one micro-dollar — every researched *charge* granularity fits.
- **Unit prices: `i64` nano-USD** (10⁻⁹ dollars) in the pricing table only, so rate×duration and per-token math is exact before rounding. $0.000025/s = 25,000 nano-USD/s. Rounding to micro-USD happens once, at charge settlement, half-up.
- **Why not one type?** nano-USD in i64 caps at ±$9.2 *billion* — uncomfortably close to your "sum revenues of billions" scenario for an accumulator. Micro everywhere money *moves*, nano where prices are *quoted*, and the seam is one well-tested rounding function.
- **Why signed (i64 not u64):** refunds, adjustments, negative balances (post-failure reconciliation) are first-class. Unsigned money forces every subtraction through a panic-or-wrap decision.

---

## 6. Provider integrations

**Yes — the specs are readable and clients are writable for all six.** Verified lifecycle per provider (full details + doc URLs in the research appendix of the build doc):

| Provider | Auth | Lifecycle | Watch out |
|---|---|---|---|
| ByteDance (BytePlus/Volcengine Ark) | Bearer key | video: create task → poll; Seedream image: **sync** | Intl vs CN deployments have different hosts AND model IDs (`dreamina-*` vs `doubao-*`); no callback support seen |
| Google Gemini API | `x-goog-api-key` | image: **sync**; Veo: `predictLongRunning` → poll operation | `gemini-3-pro-image` = Nano Banana Pro confirmed; Google is steering image gen to the new Interactions API; Veo assets live 2 days; Vertex differs (OAuth + `:fetchPredictOperation`) |
| OpenAI | Bearer key | images: **sync** (b64 only); Sora: create → poll + webhooks | **Sora-2/Videos API deprecated — shutdown 2026-09-24**; plan for successor |
| Kling | **HS256 JWT** self-signed from AK/SK, 30-min expiry | create → poll, optional `callback_url` | Token cache w/ TTL needed; assets purged after 30 days; docs portal blocks scrapers (verify field details in a browser) |
| xAI Grok | Bearer key | images: **sync** (punted per your call); video: deferred → poll `GET /v1/videos/{request_id}` | Grok Imagine video API is public and poll-based — cleaner than the consumer-surface mess; ArtCraft has bindings to adapt |
| Vidu | **`Token`** scheme (not Bearer) | create → poll `GET /ent/v2/tasks/{id}/creations`, optional callback | Result URLs ~1h expiry (unverified) — ingest immediately |

Design consequences (already in the build plan): one `ProviderAdapter` trait with `dispatch`/`poll`/`parse_callback`; callbacks are an optimization only — every async job is also polled (callbacks get lost); all outputs ingested into our storage immediately (expiries range 1h → 30 days).

**ArtCraft reuse** (inventoried from the monorepo): `crates/api_clients/` has adaptable clients for Fal (huge: queue + webhooks + polling subsystem), Grok (`grok_api_client` — official API, structured api/requests/traits/types), Comet (covers Seedance/Vidu/Wan — useful as reference even though we'll go direct), OpenAI Sora (consumer surface — we'll use the platform API instead), GMI Cloud, RunwayML, Midjourney, World Labs. The most valuable lift is **`artcraft_router`** (provider-agnostic request routing with per-provider adapter clients and mismatch-mitigation strategies — this is the omni param-translation layer) and the **poller-worker template** (`service/job/*_job` crates all share one skeleton: main loop, process_successful/failed_job, alert_on_error). The four-bucket error taxonomy (`Client` / `ApiBadRequest` / `ApiSpecific` / `ApiGeneric` with `is_having_downtime_issues()`) is worth carrying over as-is. Note the older clients use `reqwest`, newer consumer ones use `wreq` (TLS-fingerprint spoofing) — for official APIs we only need `reqwest`.

---

## 7. Job queue (the "later" suggestion)

Stay on Postgres `FOR UPDATE SKIP LOCKED` far longer than instinct says — it's transactional with the job writes (a job row and its queue task commit atomically; no dual-write problem), and it's fine into the thousands of tasks/sec with tuning. When it genuinely hurts, the candidates are **NATS JetStream** (lightweight, great Rust client, at-least-once with acks — my pick), Redis Streams (you already run Redis, but delivery semantics are more DIY), or staying in Postgres with `pgmq`. Avoid heavyweight brokers (Kafka/RabbitMQ) — wrong shape for this. The `queue` crate boundary in the build plan exists precisely so this swap is an implementation detail.

---

## 8. Startup / business

**Delaware C-corp:** Stripe Atlas — $500 flat (includes DE fees + first-year registered agent, $100/yr after), ~2 business days to incorporate, EIN handled, **files your 83(b) for you**, and the perk bundle is real: $2,500 Stripe credits + free processing on your first $100K volume + partner discounts. The main alternative worth considering is **Clerky** (~$427–819) — startup-lawyer-built, the gold standard for VC-track paperwork (SAFEs, option grants) — better if you expect a priced round soon and want bulletproof docs. Both produce standard DE C-corps law firms accept. Skip LegalZoom (upsell maze, generic docs). Given you want fast + easy: **Atlas**.

**Free credits (bootstrapped, entity + website only — all claimable now):**

| Program | Amount | Notes |
|---|---|---|
| **Cloudflare for Startups Tier 3** | **$10,000** | No funding required. Best fit for this product (Workers, R2 — zero egress for media, AI Gateway). 1-year validity. |
| Google Cloud Start tier | $2,000 | Self-serve, <5yrs old, no institutional funding. |
| AWS Activate Founders | $1,000 | Self-serve; **gotcha: taking it can reduce later Portfolio-tier ($100K) eligibility amounts** — sequence carefully. |
| Microsoft for Startups | up to $5,000 | Program restructured 2025; **trap: Azure credits do NOT cover marketplace-billed models (Anthropic etc.) in AI Foundry** — founders have eaten surprise invoices. |
| Modal | up to $25,000 | Early-stage AI startups, simple application, 12-mo expiry. |
| Supabase | 6 mo Team free | No VC needed. |
| **NVIDIA Inception** | No direct credits | It's a membership: DLI training, ~30% off DGX Cloud, discounts, and a portal to *request* partner credits (up to $100K AWS, $150K Nebius). Free, no equity, easy application (~2–4 wks). Join it, but don't count it as funding. |

The six-figure tiers everywhere (AWS $100K, GCP $200–350K AI tier, Cloudflare $100–350K, MSFT $100K) all effectively require VC/accelerator affiliation. Universal gotcha: 12–24 month expiry — don't activate big credits before your burn can consume them.

---

## 9. Website organization (reacting to your take)

Agreed that Fal's model pages are gross — but the *structure* (playground + generated API docs + schema tables per endpoint) is right; it's the execution (SEO-spam density, modality sprawl as separate near-identical pages) that's bad. Replicate's schema-driven pages are the better reference: one OpenAPI schema per version drives docs, playground form, and validation. Our version: JSON Schema per (model, modality) in the catalog DB drives all three, and the **modalities are tabs on ONE model page** instead of separate pages.

Your instinct on the coarse-grained pages is the differentiated bet: `/generate/video` as a first-class doc surface — "integrate video gen once, swap model IDs" — with a param-support/price/speed comparison matrix across models. Nobody does this well (Kie is closest and it's an afterthought). It also matches where the market is going: OpenRouter just shipped exactly this shape for video from the LLM side.

Three Nx apps (`www`, `docs`, `console`) sharing a UI lib — detailed in the build doc §9.

---

## 10. Small pushbacks / things to decide

1. **"Payment Required" hard-stop:** consider a small configurable negative-balance grace (e.g. −$5) so long-running video jobs that finish after the balance crosses zero don't strand the customer mid-workflow. The ledger design supports it; default can still be hard-stop. [Held for your call — not in the build doc.]
2. **Webhook signing:** I spec'd Stripe-style HMAC (`whsec_` secrets) rather than Fal's ED25519+JWKS. HMAC is what customers already know how to verify (Stripe/Replicate/Svix ecosystem); asymmetric adds ceremony without a threat-model win at our scale.
3. **Grok sync-image weirdness:** agreed on punting, but note the *official* xAI image API is a normal synchronous OpenAI-style endpoint — the grossness is the consumer surface ArtCraft wraps. The official one is actually one of the easiest integrations.
4. **API key display:** store only the hash, show `key_prefix` in the console, reveal full secret exactly once at creation (Stripe behavior). Worth deciding now because it constrains the console UX.
5. **Deprecation reality check:** two of our launch providers have shutdown cliffs inside 3 months of today (Sora-2 API 2026-09-24, Imagen 2026-08-17). The catalog's `status` field and alias machinery aren't nice-to-haves; model churn is the steady state of this business.

------------------------

