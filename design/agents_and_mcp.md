# Agents & MCP (the agent surface)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->
<!-- Assistant note 2026-07-17: MERGED from agents.md + mcp.md at bt's direction. Sections
     moved verbatim keep their existing sign-offs (same precedent as the 2026-07-05
     restructure); only the top-of-doc line and edited sections reset. §1–§5 = old agents.md
     §1–§5; §6–§9 = old mcp.md §1–§4. -->

**Status:** living document (2026-07-17). MERGED `agents.md` + `mcp.md` (2026-07-17): one doc
for the whole agent surface — what agents **read** (§1–§5: the generated llms.txt corpus) and
what agents **call** (§6–§9: the MCP server). Both halves are generated from the same
`crates/core/catalog` source, so they can never drift from reality or from each other. Goal:
a customer's coding agent (Claude Code, Cursor, etc.) integrates any model or API of ours in
one shot — reads the docs, checks the cost, generates, polls, self-diagnoses failures.

---

## 1. The artifacts

**Sign-off:** bt 2026-07-14

All generated from `crates/core/catalog` (the same single source that produces `catalog.json`,
the OpenAPI spec, and the docs pages). Following the llmstxt.org convention:

| Artifact                          | Content                                                                                                                                                                                                                                        |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/llms.txt`                       | Index: one-paragraph product overview, auth in 3 lines, link list to everything below                                                                                                                                                          |
| `/llms-full.txt`                  | The whole corpus concatenated (for agents that slurp one file)                                                                                                                                                                                 |
| `/apis/llms.txt`                  | Platform guide: auth, the three API shapes, cost twins, job lifecycle, polling vs webhooks (no sync mode), idempotency, errors incl. the 402 body, versioning                                                                                  |
| `/apis/generate/{media}/llms.txt` | Any-API guide per media type: the superset schema, param coercion rules, model comparison table (params × price × speed)                                                                                                                       |
| `/models/{model_id}/llms.txt`     | Everything for ONE model: canonical ID + aliases, modalities, per-modality JSON Schema rendered as a param table, pricing, limits, curl + TypeScript + Python examples (submit → poll → download, and webhook variant), model-specific gotchas |
| `/openapi.json`                   | Already emitted by utoipa; linked from the index                                                                                                                                                                                               |
| MCP server                        | `generate`/`cost`/`get_job`/`list_models` as tools — the catalog gives us the tool schemas for free. Designed below: §6–§9                                                                                                                     |

Every human docs page also carries a "Copy for LLM" affordance pointing at its llms.txt
sibling (the thing that makes Fal's pages agent-usable despite the SEO sprawl).

## 2. Where they live in the source tree

**Sign-off:** bt 2026-07-05

```
foundation/
├── crates/
│   └── tools/
│       └── docgen/                 # the generator crate (binary)
│           ├── src/
│           └── templates/          # askama/minijinja templates per artifact type
├── generated/
│   └── llms/                       # CHECKED-IN output — the reviewable artifact
│       ├── llms.txt
│       ├── llms-full.txt
│       ├── apis/
│       │   ├── llms.txt
│       │   └── generate/{image,video,audio,mesh,splat}/llms.txt
│       └── models/{model_id}/llms.txt
└── frontend/apps/docs/             # copies generated/llms/ into its publish dir at build
```

- **Checked in** (your requirement): diffs to agent docs are visible in PRs — a price change or
  schema change shows up as a reviewable docs diff, which is itself a correctness check.
- **Regenerated** by `cargo run -p docgen` (also wired as `admin-cli gen agent-docs`).
- **CI staleness gate:** CI reruns the generator and fails if `generated/llms/` differs from
  what's committed — the same pattern as sqlx offline data. Docs can never silently drift from
  the catalog.
- Deterministic output: prose fields (descriptions, tips, gotchas) are *authored data in the
  catalog registry*, not LLM output at generation time. When we want LLM help writing them, the
  LLM edits the catalog source in a PR — generation itself stays reproducible.

## 3. How they're served (your Axum vs Netlify question)

**Sign-off:** bt 2026-07-05

Both, cheaply, same files:

1. **Primary: Netlify (docs app).** The Nx docs app's build step copies `generated/llms/` into
   its static publish directory. Canonical URLs live on the docs domain
   (`docs.example.com/llms.txt`, `docs.example.com/models/{id}/llms.txt`). CDN-cached, zero
   backend load — right place for high-volume agent slurping.
2. **Convenience: api-gateway.** Agents habitually probe the API host itself. Axum serves
   `GET /llms.txt` and `GET /v1/llms.txt` as 308 redirects to the docs-domain canonical (one
   tiny handler, no file embedding, no second deploy artifact to keep fresh).

No third deployable. The repo artifact is the single source; Netlify is the delivery vehicle;
Axum is a signpost.

## 4. Generation pipeline (concretely)

**Sign-off:** bt 2026-07-05

```
catalog registry (Rust)
   │  docgen iterates models × modalities × api shapes
   ▼
templates (askama): model_page.md, api_guide.md, index.md
   │  + shared snippet renderers: json-schema → param table,
   │    example-request builder (curl/ts/python from the same schema)
   ▼
generated/llms/**   (committed)
   │  CI: regen + git diff --exit-code
   ▼
docs app build copies → Netlify CDN;  api-gateway redirects point here
```

The example-request builder is the piece worth doing well: it derives runnable curl/TS/Python
snippets from each modality's JSON Schema + defaults, so every model page's examples are
correct by construction — never hand-maintained. The docs *website* uses the same renderer
(via catalog.json) so humans and agents read the same truth.

## 5. Conventions inside the documents

**Sign-off:** bt 2026-07-05

- Front-load the contract: base URL, auth header, and a complete minimal request in the first
  20 lines (agents often read truncated files).
- Absolute URLs everywhere (files get copied into agent contexts detached from their origin).
- Every doc states the model's canonical ID early and verbatim — the string agents must not
  hallucinate.
- Include the failure modes agents must handle: 400, 401, 402 body shape, 429 rate limiting, 
  poll cadence guidance, job `expires_at`, webhook signature verification snippet.
- Keep each per-model file self-contained (paste-safe) even though that means repetition —
  repetition is free for generated files.

## 6. MCP: shape & deployment

**Sign-off:** _unsigned_

Agents shouldn't just read our docs (§1–§5) — they should be able to *call* us and iterate:
generate, check cost, poll, inspect failures, retry with fixed params. That loop is the MCP
server.

- **Transport:** Streamable HTTP MCP endpoint at `https://api.example.com/mcp`, mounted in
  `api-gateway` (a tower service alongside the REST routers — Rust MCP SDK `rmcp`). No separate
  deployable; splitting later is a routing change, same as the console-api argument.
- **Auth:** the same `foundation_api_...` keys, sent as `Authorization: Bearer` on the MCP
  connection. No special MCP identities; a revoked key kills MCP access identically.
- **Config for users:** one JSON block in Claude Code / Cursor / etc. docs; also listed on
  each model page ("Use this model from your agent").

## 7. MCP: tools

**Sign-off:** _unsigned_

Tool schemas are generated from the same catalog/api-defs source as everything else.

| Tool               | Maps to                             | Notes                                                                                                           |
| ------------------ | ----------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `list_models`      | `GET /v1/models`                    | Filters: media_type, modality, owner                                                                            |
| `get_model`        | `GET /v1/models/{model}`            | Metadata, prices, modalities                                                                                    |
| `get_model_schema` | `GET /v1/models/{model}/schema/{m}` | The JSON Schema agents need to build params                                                                     |
| `cost`             | `POST /v1/any/{media}/cost`         | Any shape; price before committing money                                                                        |
| `generate`         | `POST /v1/any/{media}`              | Any shape (model as param) — one tool per media type keeps schemas tight: `generate_image`, `generate_video`, … |
| `get_job`          | `GET /v1/jobs/{id}`                 | Status + outputs + failure code                                                                                 |
| `wait_for_job`     | poll loop server-side               | Long-poll up to N seconds so agents don't spin; returns terminal job or timeout-with-status                     |
| `get_job_logs`     | `GET /v1/jobs/{id}/logs`            | The debug timeline — agents can self-diagnose                                                                   |
| `get_balance`      | `GET /v1/billing/balance`           | Lets agents pre-check before generating                                                                         |

Guardrails: `generate_*` tool descriptions state the exact cost behavior (charged at enqueue)
and instruct agents to call `cost` first when the user hasn't approved a price; an optional
`max_cost_micros` tool argument hard-rejects anything above the caller's stated cap.

## 8. MCP: database schema impact — none

**Sign-off:** _unsigned_

Your question — does MCP need schema? **No new tables.** MCP calls authenticate with normal
API keys and create normal jobs/logs. For analytics we extend the existing `source` enum
(`api_key | dashboard | mcp`) on `jobs` and `request_logs` — better than a boolean because it's
one dimension, already indexed, already flowing into the usage rollups (`rankings.md`) — so
"what share of traffic is agentic?" is a one-liner, and a future fourth source doesn't need
another column.

## 9. MCP: not in v1

**Sign-off:** _unsigned_

OAuth-based MCP auth (key-in-config is fine for launch), MCP resources/prompts (tools only),
stdio-transport local binary, exposing account-admin operations as tools (billing mutations
stay human-only).
