# Frontend: Docs

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). The `docs` Nx app on `docs.example.com` — everything
developer-facing that is NOT a model page: guides, API reference, integration recipes.
Model-specific documentation (params, examples, pricing, gallery, playground) lives on the
model pages (`frontend_playground.md`) — this app links there, never duplicates. Siblings:
`frontend_overview.md`, `agents.md` (agent-readable docs), `api_overview.md`, `api_misc.md`.

---

## 1. Content map

**Sign-off:** _unsigned_

| Route                    | Content                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `/quickstart`            | Key → first generation in 5 minutes (curl/TS/Python tabs)                                                     |
| `/guides/authentication` | API key formats, all accepted auth forms (`api_overview.md` §1)                                               |
| `/guides/jobs`           | The job lifecycle: statuses, polling, expiry, cancel/refund semantics (`jobs.md`)                             |
| `/guides/webhooks`       | Outbound webhook setup + signature verification, 5-line verify snippets per language (`api_overview.md` §4)   |
| `/guides/errors`         | Error envelope, error types, `FailureCode` taxonomy — "was this my fault?" (`api_overview.md` §2)             |
| `/guides/billing`        | Prepaid balance, cost twins, spend limits, auto-refill (`api_misc.md` §5)                                     |
| `/guides/files`          | Uploads, URL inputs, output hosting (`api_misc.md` §4, `files.md`)                                            |
| `/guides/any-api`        | The Any plane: cross-model requests, coercion (`api_any.md`)                                                  |
| `/guides/chat`           | OpenAI-compatible chat plane, migration notes (`apis_llm.md`)                                                 |
| `/guides/mcp`            | MCP endpoint + tool list (`mcp.md`)                                                                           |
| `/guides/end-users`      | The `end_user` attribution param (`end_users.md`)                                                             |
| `/reference/...`         | Generated API reference from the utoipa OpenAPI spec — one page per endpoint group                            |
| `/changelog`             | Platform changelog: API versions (date pins), new endpoint groups; model-level changelogs stay on model pages |

## 2. Conventions

**Sign-off:** _unsigned_

- **Reference is generated** from the same OpenAPI spec as the TS client — docs can't drift
  from the wire (`enums.md` pipeline). Guides are authored MDX.
- Every code sample compiles/runs in CI where feasible (snippets extracted and executed
  against a test account — the ArtCraft "docs that lie are worse than no docs" lesson).
- Agent docs (`llms.txt`, `llms-full.txt`) are generated siblings, same content pipeline
  (`agents.md`); the human docs and agent docs never diverge.
- 100% public, no login wall (same rule as model pages).
- Search: static index (pagefind-style) — no external search service dependency at launch.
