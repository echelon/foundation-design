# APIs: model-specific (unified + granular)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Peeled from `api_overview.md` §3. One of the three
generation planes: this doc (unified + granular), `api_any.md` (Any), `apis_llm.md` (chat).
Shared conventions: `api_overview.md` §1–§3.

---

## 1. Two sub-shapes, one namespace

**Sign-off:** _unsigned_

Both sub-shapes address a **named model** — the Fal-style integration where the customer has
already chosen their model and wants its real parameters, not a superset:

| Sub-shape    | You specify          | Body schema                                                                                  | `api_shape` |
| ------------ | -------------------- | -------------------------------------------------------------------------------------------- | ----------- |
| **Unified**  | the model            | that model's param **union**; modality inferred, echoed as `resolved_modality`               | `unified`   |
| **Granular** | the model + modality | **exactly** that (model, modality)'s typed schema — the SDK-codegen and docs-sandbox surface | `granular`  |

Covers every media type: image, video, audio, mesh, splat, and text (LLMs via the `chat`
modality — synchronous, §4).

**Strictness (the deliberate contrast with Any):** no pruning, no coercion. Unknown or
out-of-range params → `400 invalid_request` naming the `param`. On this plane you said
exactly what you meant; silently adjusting it would be a lie. (Any is the forgiving door:
`api_any.md` §1.)

## 2. Namespace: under the model resource

**Sign-off:** _unsigned_

Model-specific invocation lives **on the model resource**, next to the catalog reads that
describe it (`api_misc.md` §3: `GET /v1/models/{model}`, `GET /v1/models/{model}/schema/{modality}`)
— read the schema and call the endpoint at the same path root:

| Endpoint                                 | Method | Purpose                                                                              |
| ---------------------------------------- | ------ | ------------------------------------------------------------------------------------ |
| `/v1/models/{model}/generate`            | POST   | Unified: param union, modality inferred                                              |
| `/v1/models/{model}/generate/{modality}` | POST   | Granular: exact schema, e.g. `/v1/models/kling_video_v3_pro/generate/image_to_video` |
| `/v1/models/{model}/cost`                | POST   | Cost twin of unified (identical body)                                                |
| `/v1/models/{model}/cost/{modality}`     | POST   | Cost twin of granular                                                                |

`{model}` accepts the full ID grammar: canonical slug, alias, `_latest`, `_snap_{ver}`
(`models.md` §13); the job records the canonical slug that ran.

**Model IDs:** the complete enumerated catalog — every slug, grouped per media type — is
**`models.md` §2–§9** (also served live at `GET /v1/models`). The naming scheme/convention
(`{owner}_{name}_{version}`, version tokens like `v2_5`, the reserved `_latest` /
`_snap_{ver}` suffixes) is `models.md` §1 + §13, summarized in `conventions_naming.md` §3.

Alternatives considered:

| Namespace                                  | Verdict                                                                                                                     |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `/v1/generate/models/{model}[/{modality}]` | Previous design (`api_overview.md` §3) — RETIRED: buries the model resource under a verb; catalog reads then live elsewhere |
| `/v1/run/{model}` (Fal-style)              | Terse, but `run` says nothing about cost twins, and Fal-flavored                                                            |
| `/v1/predictions` (Replicate-style)        | Wrong noun for us (jobs), and model-in-body defeats the point                                                               |
| **`/v1/models/{model}/generate[...]`**     | **RECOMMENDED** — invocation and introspection co-located on one resource                                                   |

## 3. Request & response (media)

**Sign-off:** 2026-07-14

```json
POST /v1/models/kling_video_v3_pro/generate/image_to_video
{
  "prompt": "...",
  "image_url": "file_01j0...",
  "duration_seconds": 5,
  "end_user": "cust-123",
  "webhook": { "url": "https://...", "events": ["job.completed"] }
}
```

No `model` field in the body — it's in the path (a body `model` that contradicts the path is
a `400`). Media responses are always async: `201` + Job resource, then polling or webhook
(`api_overview.md` §3). Granular schemas come straight from the catalog's typed param structs — the
same source that generates the docs pages, sandbox forms, and SDK types (`models.md`,
`agents_and_mcp.md`): one schema, four surfaces, zero drift.

## 4. LLMs on this plane — synchronous

**Sign-off:** _unsigned_

- Granular: `POST /v1/models/{model}/generate/chat` — held-open (optional SSE), our
  **canonical** chat schema (same one as `/v1/any/text`, NOT the OpenAI dialect — that's
  `apis_llm.md`). Response: `200`, completion inline + job fields.
- Unified on an LLM (`POST /v1/models/deepseek_v4/generate`) resolves modality `chat` and
  behaves identically.
- Billing is metered (worst-case admission, actuals at completion — `architecture.md` §4);
  the cost twins return the worst-case estimate + rates.

## 5. Who this plane is for

**Sign-off:** _unsigned_

- **Granular** is the surface SDKs and typed integrations codegen against, the target of
  every docs-page "try it" sandbox, and what per-model example snippets teach. Per-model
  modality declarations (`models.md` §10) define exactly which granular endpoints exist.
- **Unified** is the middle door: model-committed customers who don't want to route on
  modality themselves (one endpoint per model regardless of input mix).
- Choosing across models at runtime? That's the Any plane (`api_any.md`). OpenAI-compat
  drop-in? Chat plane (`apis_llm.md`). All doors produce the same Job resource.
