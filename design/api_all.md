# APIs: the Any API (the "all" shape)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Peeled from `api.md` §3. One of the three
generation planes: this doc (Any), `apis_llm.md` (chat), `api_model_specific.md`
(unified + granular). Shared conventions: `api.md` §1–§3.

**Naming note:** bt calls this the "all"-shaped API; the DECIDED product brand (2026-07-05)
is **the Any API** ("omni" expunged — `../reports/read/omni-api-naming.md`). This doc keeps
"Any" for the brand and paths. If "all" is meant as a rebrand, say the word and I'll sweep.

---

## 1. The shape

**Sign-off:** _unsigned_

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

**Sign-off:** _unsigned_

**`POST /v1/any/{media_type}`** — reads as what it is ("run this on any video model"), names
the brand in the path, and gives the plane its own namespace instead of sharing a generic
`/v1/generate` with the other shapes.

| Endpoint               | Method | Purpose                                    |
| ---------------------- | ------ | ------------------------------------------- |
| `/v1/any/image`        | POST   | Any image model                             |
| `/v1/any/video`        | POST   | Any video model                             |
| `/v1/any/audio`        | POST   | Any audio model                             |
| `/v1/any/mesh`         | POST   | Any mesh model                              |
| `/v1/any/splat`        | POST   | Any splat model                             |
| `/v1/any/text`         | POST   | Any LLM — **asynchronous** (§4)             |
| `/v1/any/text/sync`    | POST   | Any LLM — **synchronous** (§4)              |
| `/v1/any/{media}/cost` | POST   | Cost twin: identical body, exact price (worst-case estimate for text), no job, no charge |

Alternatives considered:

| Namespace                   | Verdict                                                        |
| --------------------------- | --------------------------------------------------------------- |
| `/v1/generate/{media}`      | Previous design (`api.md` §3) — RETIRED: generic verb, doesn't say which shape, and the other planes squatted under it too |
| `/v1/all/{media}`           | Close, but the brand is Any (decided); "all" reads as "every model at once" |
| **`/v1/any/{media}`**       | **RECOMMENDED** — brand in the path, one namespace per plane    |

## 3. Request & response (media)

**Sign-off:** _unsigned_

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

## 4. Text through Any — synchronous

**Sign-off:** _unsigned_

`POST /v1/any/text` is the LLM member of the family and it is **synchronous** (held-open,
optional SSE via `"stream": true`) — same execution model as the chat plane, same metered
billing, same job row (`api_shape = any`, `media_type = text`).

- Body is our **canonical text schema** (`model`, `messages`, `max_tokens`, samplers…) — the
  Any-style canonicalized superset, NOT the OpenAI wire dialect. OpenAI compat lives at
  `/v1/chat/completions` (`apis_llm.md`); this endpoint exists so the Any plane covers all
  media types uniformly.
- Response: `200` with the completion inline + the job fields (`id`, `charged_micros`,
  `usage`, `param_adjustments`).

## 5. Fit with the other planes

**Sign-off:** _unsigned_

| Plane          | You know…            | Schema                    | Doc                     |
| -------------- | -------------------- | ------------------------- | ------------------------ |
| **Any**        | the media type       | superset, pruned/coerced  | this doc                 |
| Unified        | the model            | that model's param union  | `api_model_specific.md`  |
| Granular       | the model + modality | exact typed schema        | `api_model_specific.md`  |
| Chat           | OpenAI's dialect     | OpenAI wire format        | `apis_llm.md`            |

All four produce the same Job resource and identical billing/logging. `api_shape` on the job
records which door was used.
