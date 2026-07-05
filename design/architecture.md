# Architecture

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document. Split out of `../2026-07/2026-07-04-initial-build.md` on 2026-07-04.
Siblings: `api.md`, `schema.md`, `models.md`, `enums.md`, `agents.md`, `logging.md`, `mcp.md`,
`end-users.md`, `rankings.md`.

---

## 1. Decided technology stack

**Sign-off:** _unsigned_

| Choice          | Decision                              | Notes                                                        |
| --------------- | ------------------------------------- | ------------------------------------------------------------ |
| Web framework   | **Axum** (DECIDED 2026-07-04)         | Work-stealing Tokio runtime, tower middleware, utoipa OpenAPI |
| Database        | **Postgres** (DECIDED 2026-07-04)     | Managed on DigitalOcean; gotcha checklist in §9               |
| Cache / hot path| Redis (DO managed Valkey/Redis)       | Rate limiting, session cache                                  |
| Object storage  | **Cloudflare R2** (DECIDED)           | Zero egress — we host media forever (§8)                      |
| Frontend hosting| **Netlify** (DECIDED)                 | All three Nx apps                                             |
| Compute hosting | **DigitalOcean** (DECIDED)            | No AWS. Droplets or App Platform for api-server + worker      |
| Job queue       | Postgres (`FOR UPDATE SKIP LOCKED`)   | Swappable later (NATS JetStream is the leading candidate)     |
| Billing         | Stripe via `async-stripe`             |                                                               |
| Frontend        | Nx + TypeScript + React + Tailwind + Vite |                                                           |

DigitalOcean managed Postgres: confirmed available — automated daily backups, point-in-time
recovery, standby nodes for HA, built-in PgBouncer connection pooling, private VPC networking.

## 2. Catalog as code (DECIDED 2026-07-04)

**Sign-off:** _unsigned_

The model catalog — models, modalities, parameter schemas, provider routing, prices — is
**declared in Rust, not stored in the database**. Code is versioned, reviewable, testable; the
catalog IS the product. This replaces the earlier draft's `providers` / `models` /
`model_endpoints` / `model_aliases` / `model_routes` / `prices` tables, all deleted.

- `crates/core/catalog` holds the registry: one Rust module per model, registered into a
  static table (compile-time checked: unique slugs, unique prefixes, every model has ≥1
  modality, every modality has a price).
- Per (model, modality): typed parameter struct → JSON Schema derived from the Rust types
  (schemars or utoipa), defaults, coercion rules for the omni layer, price definition
  (unit + nano-USD rate + modifiers), provider route(s) in priority order.
- Aliases (short forms, renames, competitor-ID migration) are code too: a static
  `alias → canonical` map with a uniqueness test (see `models.md` §aliases).
- Price changes, new models, deprecations = normal PRs + deploys. Jobs snapshot the price they
  were charged onto the job row (see `schema.md`), so history survives code changes and no
  historical price table is needed.
- The docs/console frontends consume a **generated JSON artifact** (`catalog.json`) emitted by
  a build step from the same registry — single source of truth, no DB round trip.
- Provider credentials are config/environment (per-deployment secrets with named pools, e.g.
  ByteDance `volcengine` / `byteplus` / `byteplus_ultra`), not DB rows.
- Escape hatch (future, only if needed): a tiny runtime kill-switch (env/flag service) to
  disable a model without a deploy. Not in v1.

## 3. Identifiers (REVISED 2026-07-04)

**Sign-off:** _unsigned_

One ID per row: **the prefixed string is the primary key**. No internal numeric IDs.
(Full integer-PK analysis: `../reports/identifiers.md`.)

- Format: `{prefix}_{26-char Crockford base32 of 128 bits}`, e.g. `job_01j0qs3fw9x7k2m5n8p1r4t7v0`.
- Layout is ULID-style: 48-bit millisecond timestamp + 80 random bits. Time-ordered high bits
  keep B-tree inserts local (right-leaning appends, not random splits), and make
  `ORDER BY id` chronological.
- Used everywhere: PKs, FKs, joins, logs, API responses. A bare ID in a log tells you what it
  is — that's the point.
- Cost accepted: TEXT keys make PK/FK indexes roughly 3–4× larger than bigint and joins
  slightly slower. At our scale this is noise; debuggability wins.
- Collision math (128-bit, 80 random bits per millisecond): across n randomly generated IDs,
  collision probability ≈ n²/2¹²⁹. 10⁹ IDs (hundreds of millions) → ~10⁻²¹. 10¹² (trillions)
  → ~10⁻¹⁵. Within a single millisecond, 10,000 IDs collide with p ≈ 4×10⁻¹⁷. And every PK has
  a unique constraint — a collision is a retried insert, not corruption. 128 bits is enough for
  trillions.
- Prefix registry lives in `crates/core/ids` as one enum with a compile-time uniqueness test.
  Current prefixes: see `schema.md` (one per table) plus secrets: `foundation_api_` (API keys)
  and `foundation_whsec_` (webhook signing secrets) — service-declaring by design; the
  `foundation` half swaps when the product is named.

## 4. Billing policy (DECIDED 2026-07-04)

**Sign-off:** _unsigned_

- **Payment at point of enqueue. The price of a job never changes after submission.**
- Admission: in one transaction — `SELECT ... FOR UPDATE` the balance row; if
  `balance_micros < cost` → `402 Payment Required` (machine-readable body with balance, cost,
  top-up link); else insert ledger debit, update balance, insert job + queue task, commit.
- Job failure → ledger refund entry (full amount). No holds table, no reserved balance, no
  partial capture — deleted from the earlier draft.
- No negative balances, no grace. (Reversible policy later; not now.)
- Money types: ledger/balances in `i64` micro-USD; catalog unit prices in `i64` nano-USD;
  one rounding function (half-up to micros) at quote time. Newtypes `Micros` / `NanoUsd` in
  `crates/core/money`.
- Dashboard sandbox runs bill **identically** to API runs: same ledger, same logs, same jobs
  table, same 402. The only difference is auth (cookie vs API key), recorded on the job as
  `source = dashboard | api | mcp`.
- Spend visibility is a first-class product surface: every job row carries its cost + status;
  `GET /v1/jobs` and the ledger endpoints power the console's spend views (see `api.md`).

## 5. Monorepo layout (REVISED: `providers` → `clients`; single api-server)

**Sign-off:** _unsigned_

```
foundation/
├── Cargo.toml                      # [workspace]
├── rust-toolchain.toml
├── crates/
│   ├── bin/
│   │   ├── api-server/             # ONE service: public API + console API + /mcp (axum)
│   │   ├── worker/                 # queue consumer: dispatch/poll/webhook/ingest/rollups
│   │   └── admin-cli/              # migrations, ops one-offs
│   ├── core/
│   │   ├── ids/                    # prefixed string ID types + prefix registry
│   │   ├── money/                  # Micros / NanoUsd newtypes
│   │   ├── enums/                  # job status, media type, modality, provider, …
│   │   ├── api-defs/               # request/response types, utoipa schemas, version transforms
│   │   ├── catalog/                # THE model registry (code-as-catalog, §2)
│   │   └── config/                 # env/config loading, credential pools
│   ├── db/
│   │   ├── migrations/
│   │   └── queries/                # one function per query, sqlx compile-checked
│   ├── clients/                    # third-party API clients — pure HTTP, no DB deps
│   │   ├── client-core/            # shared traits, reqwest factory, retry/backoff,
│   │   │                           #   four-bucket error taxonomy (from ArtCraft)
│   │   ├── bytedance/              # Seedance (poll) + Seedream (sync), BytePlus + Volcengine
│   │   ├── google/                 # Gemini image (sync) + Veo (LRO poll)
│   │   ├── openai/                 # gpt-image (sync), video (poll), chat completions
│   │   ├── anthropic/              # messages API (LLM)
│   │   ├── kling/                  # HS256 JWT auth, create+poll, callback
│   │   ├── xai/                    # Grok image (sync), Grok Imagine video (poll), chat
│   │   └── vidu/                   # create+poll, callback, `Token` auth scheme
│   ├── router/                     # omni param translation: canonical request → provider
│   │                               #   request; prune/coerce; modality inference
│   ├── services/                   # business logic, DB-aware
│   │   ├── jobs/                   # job state machine
│   │   ├── billing/                # ledger, auto-refill, spend limits, 402 admission
│   │   ├── auth/                   # sessions, api keys, invitations
│   │   ├── webhooks/               # outbound delivery + HMAC signing
│   │   └── logging/                # debug-log product write path (see logging.md)
│   ├── tools/
│   │   └── docgen/                 # agent-docs generator (see agents.md)
│   └── queue/                      # queue abstraction (Postgres impl now)
├── generated/
│   └── llms/                       # checked-in agent docs (see agents.md)
├── frontend/                       # Nx workspace → Netlify
│   ├── apps/
│   │   ├── www/                    # marketing (+ public rankings page, see rankings.md)
│   │   ├── docs/                   # model pages + sandbox + API docs (reads catalog.json)
│   │   └── console/                # keys, billing, jobs, logs, team
│   └── libs/
│       ├── ui/
│       ├── api-client/             # generated from our OpenAPI
│       └── model-forms/            # JSON Schema → form renderer
├── external/claude_pairing/        # planning docs (this tree)
├── docs/                           # internal engineering docs
└── infra/                          # docker-compose (pg, redis, minio-as-R2-stand-in), CI
```

**api-server vs dashboard-server:** one service (DECIDED, revisit if scaling demands). The
console calls the same endpoints as customers, with cookie auth instead of keys — a separate
dashboard service would duplicate the whole auth/billing/jobs stack for zero isolation benefit
now. Routers are separate axum `Router` modules mounted in one binary, so splitting later is a
deployment change, not a refactor. The `worker` is a separate deployable from day one (it has
different scaling and restart characteristics).

## 6. Provider adapter seam

**Sign-off:** _unsigned_

```rust
trait ProviderAdapter {
    async fn dispatch(&self, req: CanonicalRequest, creds: &Creds) -> Result<DispatchOutcome>;
    //   DispatchOutcome = Receipt(provider_job_id) | Completed(outputs)   (sync providers)
    async fn poll(&self, receipt: &Receipt, creds: &Creds) -> Result<PollOutcome>;
    fn parse_callback(&self, body: &[u8], headers: &HeaderMap) -> Result<CallbackOutcome>;
}
```

Rules: callbacks (Kling, Vidu, OpenAI video) are an optimization — every async job is also
polled; outputs are ingested into R2 **before** the job is marked succeeded (upstream URL
expiries range from ~1 hour to 30 days); provider model IDs and credential pool selection come
from the code catalog (§2).

## 7. LLMs: in scope for v1 (DECIDED 2026-07-04, pending your final word)

**Sign-off:** _unsigned_

Congruence check result: LLM serving APIs are congruent enough to build now. The OpenAI
chat-completions wire format is the industry lingua franca — OpenRouter, AtlasCloud, xAI,
DeepSeek, Google (compat endpoint), and most others speak it natively; Anthropic and Gemini
native APIs differ but translate mechanically (system prompt placement, content blocks, tool
schemas — well-trodden adapter territory). Plan:

- `POST /v1/chat/completions`, strictly OpenAI-compatible, `model` = our slug.
- Synchronous pass-through with SSE piping — **bypasses the job queue** (no polling, no
  webhooks), but writes a `jobs` row (media_type `text`, terminal status immediately) so spend
  history, logs, and billing are uniform across media and LLMs.
- Billing nuance: token counts are known only after completion, so LLM calls are the one
  exception to fixed-cost-at-enqueue — we admission-check against balance with a conservative
  estimate (max_tokens × rate), then charge actuals at completion. Media jobs keep the strict
  price-fixed-at-enqueue rule.
- Punt: batch APIs, realtime/websocket voice, embeddings (paths reserved, nothing built).

## 8. Retention (DECIDED 2026-07-04)

**Sign-off:** _unsigned_

- **Media outputs: kept forever.** Hot in R2 initially; migrate cold objects to
  R2 Infrequent Access (or equivalent) by age/access policy later. Durable hosting is a
  product feature (and a possible future product line: managed result hosting for customers).
- **Logs: kept indefinitely, downsampled at scale.** Monthly partitions; when large, compact
  old partitions (drop full request/response bodies, keep the event skeleton + metadata).
  Partition-drop machinery exists from day one; the policy knob starts at "never".
  Details: `logging.md`.

## 9. Postgres gotcha checklist (operational guardrails I own)

**Sign-off:** _unsigned_

Carried from the feedback doc — these get encoded as code review rules / infra defaults:

1. Aggressive per-table autovacuum + lowered fillfactor on hot-update tables (`queue_tasks`, `account_balances`).
2. `idle_in_transaction_session_timeout` set globally; short transactions only around provider I/O (never hold a txn across an HTTP call — ArtCraft's PoolTimedOut lesson).
3. Monitor XID wraparound age (`datfrozenxid`) from day one.
4. Connection budget: modest sqlx pools; DO's built-in PgBouncer in transaction mode for anything bursty.
5. `max_slot_wal_keep_size` set so a dead replica/slot can't fill the disk.
6. Every migration sets `lock_timeout`; `CREATE INDEX CONCURRENTLY` for anything on a live table.
7. No `count(*)` on big tables in request paths — estimates or rollups.
8. `text` not `varchar(n)`; `timestamptz` not `timestamp`; explicit `FOR UPDATE` on money paths, never isolation-level cleverness.
9. **No native Postgres enums — DECIDED 2026-07-05.** Enum-shaped columns are `text` + CHECK,
   with the closed set owned by the Rust enums (`enums.md`); CHECK lists generated from Rust.

## 10. API versioning (unchanged)

**Sign-off:** _unsigned_

Stripe model: date versions (`2026-07-04`), pinned per account at first use, per-request
override header, edge transforms for old versions, `/v1/` URL constant. Header name blocked on
product naming (see open questions).

## 11. Open questions

**Sign-off:** _unsigned_

1. **Product name.** Several strings bake the name in permanently — the `Foundation-Version`
   and `Foundation-Signature` headers, the `foundation_api_` / `foundation_whsec_` secret
   prefixes, user-agent we send upstream, Rust crate namespace, domains. "Foundation" is a
   placeholder; once you pick the real name I'll sweep it through the docs.
2. Which DO compute primitive: App Platform (simpler) vs Droplets + docker (cheaper, more
   control). Leaning App Platform to start.
3. LLM launch set: which providers first? (OpenAI + Anthropic + xAI is the natural trio given
   we're already writing those clients for media.)
4. The "omni" product name — candidates and analysis in `../reports/omni-api-naming.md`;
   docs keep saying "omni" until you pick.
5. "Modality" vs "task" terminology — analysis in `models.md`; docs keep "modality" until you
   rule.
