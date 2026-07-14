# MCP Server

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Agents shouldn't just read our docs (`agents.md`) —
they should be able to *call* us and iterate: generate, check cost, poll, inspect failures,
retry with fixed params. That loop is the MCP server.

---

## 1. Shape & deployment

**Sign-off:** _unsigned_

- **Transport:** Streamable HTTP MCP endpoint at `https://api.example.dev/mcp`, mounted in
  `api-server` (a tower service alongside the REST routers — Rust MCP SDK `rmcp`). No separate
  deployable; splitting later is a routing change, same as the console-api argument.
- **Auth:** the same `foundation_api_...` keys, sent as `Authorization: Bearer` on the MCP
  connection. No special MCP identities; a revoked key kills MCP access identically.
- **Config for users:** one JSON block in Claude Code / Cursor / etc. docs; also listed on
  each model page ("Use this model from your agent").

## 2. Tools

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

## 3. Database schema impact: none

**Sign-off:** _unsigned_

Your question — does MCP need schema? **No new tables.** MCP calls authenticate with normal
API keys and create normal jobs/logs. For analytics we extend the existing `source` enum
(`api | dashboard | mcp`) on `jobs` and `request_logs` — better than a boolean because it's
one dimension, already indexed, already flowing into the usage rollups (`rankings.md`) — so
"what share of traffic is agentic?" is a one-liner, and a future fourth source doesn't need
another column.

## 4. Not in v1

**Sign-off:** _unsigned_

OAuth-based MCP auth (key-in-config is fine for launch), MCP resources/prompts (tools only),
stdio-transport local binary, exposing account-admin operations as tools (billing mutations
stay human-only).
