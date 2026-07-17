# APIs: the Any API

**Sign-off:** bt 2026-07-14

<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Peeled from `api.md` §3. One of the three
generation planes: this doc (Any), `apis_llm.md` (chat), `api_model_specific.md`
(unified + granular). Shared conventions: `api.md` §1–§3. Brand: **the Any API** — DECIDED
2026-07-05, "omni" expunged (`../reports/read/omni-api-naming.md`).

---

## 1. The shape

**Sign-off:** bt 2026-07-14

One endpoint per media type, **model as a parameter**, one superset request schema per media
type. Switching models is changing one string — no per-model integration work, ever. This is
the plane that covers *everything we serve*: image, video, audio, mesh, splat, **and text**.

- Params irrelevant to the chosen model are **pruned**; near-miss values are **coerced**
  (e.g. `duration_seconds: 6` → nearest supported 5); every adjustment is reported in the
  response AND recorded on the job (`param_adjustments`). We never fail a request our
  coercion rules can honestly satisfy — and never silently distort one either.
- Modality is **inferred** from the params present (image input + text → `image_to_video`),
  echoed back as `resolved_modality`.
- The superset schema per media type is generated from the catalog (union of all models'
  params, canonicalized names); documented as one page per media type.

## 2. Namespace: `/v1/any/*`

**Sign-off:** bt 2026-07-14

**`POST /v1/any/{media_type}`** — reads as what it is ("run this on any video model"), names
the brand in the path, and gives the plane its own namespace instead of sharing a generic
`/v1/generate` with the other shapes.

| Endpoint               | Method | Purpose                                                                                  |
| ---------------------- | ------ | ---------------------------------------------------------------------------------------- |
| `/v1/any/image`        | POST   | Any image model                                                                          |
| `/v1/any/video`        | POST   | Any video model                                                                          |
| `/v1/any/audio`        | POST   | Any audio model                                                                          |
| `/v1/any/mesh`         | POST   | Any mesh model                                                                           |
| `/v1/any/splat`        | POST   | Any splat model                                                                          |
| `/v1/any/text`         | POST   | Any LLM — **asynchronous** (§4)                                                          |
| `/v1/any/text/sync`    | POST   | Any LLM — **synchronous** (§4)                                                           |
| `/v1/any/{media}/cost` | POST   | Cost twin: identical body, exact price (worst-case estimate for text), no job, no charge |

## 3. Request & response (media)

**Sign-off:** 2026-07-14

```json
POST /v1/any/video
{
  "model": "kling_video_v3_pro",
  "prompt": "...",
  "image_url": "file_01j0... or https://...",
  "duration_seconds": 5,
  "end_user": "cust-123",
  "webhook": { "url": "https://...", "events": ["job.completed", "job.failed"] }
}
```

Media responses are **always async**: `201` + the Job resource (`api.md` §3), results by
polling `/v1/jobs/{id}` or webhook — never a held-open call (DECIDED 2026-07-05,
`../reports/sync-apis.md`). `param_adjustments` rides along in the 201 body.

The only exception is `/v1/any/text/sync` which is **synchronous**.

## 4. Text through Any — async by default, `/sync` variant

**Sign-off:** bt 2026-07-14

Text is the LLM member of the family, with two doors (matching the §2 table):

- **`POST /v1/any/text` — asynchronous**, uniform with every other media type: `201` + Job
  resource, dispatch through the queue, result by polling or webhook. The completion lands
  as an output of `kind = text` (inline or file spill — `files.md` §3). This is the door for
  batchy/agentic callers who don't want to hold a connection open.
- **`POST /v1/any/text/sync` — synchronous**: held-open, optional SSE via `"stream": true` —
  same execution model as the chat plane. Response: `200` with the completion inline + the
  job fields (`id`, `charged_micros`, `usage`, `param_adjustments`).

Both doors share the same body — our **canonical text schema** (`model`, `messages`,
`max_tokens`, samplers…), the Any-style canonicalized superset, NOT the OpenAI wire dialect
(OpenAI compat lives at `/v1/chat/completions`, `apis_llm.md`) — the same metered billing
(worst-case admission, actuals at completion), and the same job row shape
(`api_shape = any`, `media_type = text`; the async door walks `queued → running`, the sync
door is born `running`).

## 5. Fit with the other planes

**Sign-off:** bt 2026-07-14

| Plane    | You know…            | Schema                   | Doc                     |
| -------- | -------------------- | ------------------------ | ----------------------- |
| **Any**  | the media type       | superset, pruned/coerced | this doc                |
| Unified  | the model            | that model's param union | `api_model_specific.md` |
| Granular | the model + modality | exact typed schema       | `api_model_specific.md` |
| Chat     | OpenAI's dialect     | OpenAI wire format       | `apis_llm.md`           |

All four produce the same Job resource and identical billing/logging. `api_shape` on the job
records which door was used.
