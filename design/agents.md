# Agent-Facing Documentation (llms.txt & friends)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->
<!-- Assistant note 2026-07-05: you signed this doc; restructuring to per-section sign-offs
     preserved your initials on unchanged sections. Only §1 changed (MCP row → mcp.md;
     removed the retired `Prefer: wait` mention), so §1 and this top line are re-unsigned. -->

**Status:** living document (2026-07-05). Goal: a customer's coding agent (Claude Code,
Cursor, etc.) should be able to integrate any model or API of ours in one shot, from documents
we generate — the way Fal publishes `llms.txt` corpora, but generated from our catalog crate so
they can never drift from reality.

---

## 1. The artifacts

**Sign-off:** _unsigned_

All generated from `crates/core/catalog` (the same single source that produces `catalog.json`,
the OpenAPI spec, and the docs pages). Following the llmstxt.org convention:

| Artifact                             | Content                                                              |
| ------------------------------------ | -------------------------------------------------------------------- |
| `/llms.txt`                          | Index: one-paragraph product overview, auth in 3 lines, link list to everything below |
| `/llms-full.txt`                     | The whole corpus concatenated (for agents that slurp one file)        |
| `/apis/llms.txt`                     | Platform guide: auth, the three API shapes, cost twins, job lifecycle, polling vs webhooks (no sync mode), idempotency, errors incl. the 402 body, versioning |
| `/apis/generate/{media}/llms.txt`    | Any-API guide per media type: the superset schema, param coercion rules, model comparison table (params × price × speed) |
| `/models/{model_id}/llms.txt`        | Everything for ONE model: canonical ID + aliases, modalities, per-modality JSON Schema rendered as a param table, pricing, limits, curl + TypeScript + Python examples (submit → poll → download, and webhook variant), model-specific gotchas |
| `/openapi.json`                      | Already emitted by utoipa; linked from the index                      |
| MCP server                           | `generate`/`cost`/`get_job`/`list_models` as tools — the catalog gives us the tool schemas for free. Now designed: see `mcp.md` |

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
   (`docs.example.dev/llms.txt`, `docs.example.dev/models/{id}/llms.txt`). CDN-cached, zero
   backend load — right place for high-volume agent slurping.
2. **Convenience: api-server.** Agents habitually probe the API host itself. Axum serves
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
docs app build copies → Netlify CDN;  api-server redirects point here
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
- Include the failure modes agents must handle: 402 body shape, poll cadence guidance, job
  `expires_at`, webhook signature verification snippet.
- Keep each per-model file self-contained (paste-safe) even though that means repetition —
  repetition is free for generated files.
