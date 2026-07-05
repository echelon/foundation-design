# Sync APIs: who offers them, and why we won't

Report, 2026-07-05. Decision recorded in `../design/api.md` §1: **no sync mode for media —
polling + webhooks only.** LLM chat stays held-open because that's the medium, not a "mode."

---

## Who offers synchronous generation (hold the connection, return the result)

| Provider        | Sync offering                                                       | Notes                                                        |
| --------------- | -------------------------------------------------------------------- | ------------------------------------------------------------ |
| Fal             | Yes — dedicated `fal.run` host                                        | Explicitly second-class in their own docs: no retries, no queue, "prototyping"; the queue API is the recommended path |
| Replicate       | Yes — `Prefer: wait[=N]` header (≤60s)                                | Degrades gracefully: on timeout returns the in-flight prediction and you poll. Not a separate API |
| Kie.ai          | No — createTask → poll/callback only                                  |                                                              |
| AtlasCloud      | No for media — submit → poll (~2s), deliberately no webhooks either   |                                                              |
| OpenRouter      | LLMs: held-open by nature. **Media: their new video API is async job-based** — even the sync-native LLM shop went async for video |    |

## Foundation labs (first-party APIs)

| Lab / API                       | Lifecycle                                                   |
| ------------------------------- | ------------------------------------------------------------ |
| OpenAI chat / Anthropic / Gemini text | Held-open POST (+SSE) — seconds-scale, inherently sync   |
| OpenAI gpt-image                | **Sync** (seconds; b64 body)                                  |
| Gemini image (Nano Banana)      | **Sync** (inline b64)                                         |
| xAI Grok image                  | **Sync**                                                      |
| ByteDance Seedream image        | **Sync**                                                      |
| OpenAI video (Sora)             | Async: create → poll (+ webhooks)                             |
| Google Veo                      | Async: long-running operation → poll                          |
| Kling / Vidu / Seedance video   | Async: create → poll (+ callbacks)                            |
| xAI Grok Imagine video          | Async: deferred request → poll                                |

Pattern: **light image gen is sync at the source; anything heavy (video, 3D, world) is async
everywhere, with no exceptions.** Nobody holds a connection for a 90-second video render.

## Why we skip sync even where upstream is sync

1. One lifecycle for customers: every media job is job-id → poll/webhook. Two delivery models
   double the docs, SDK surface, retry semantics, and failure modes — for a feature Fal
   itself flags as prototyping-grade.
2. Sync connections can't survive our retries/failover: held connections pin a request to a
   process; a deploy or worker death mid-hold is a dropped result the client can't recover
   without... polling. The async path IS the recovery path; offering both means building both.
3. The "sync feel" is recoverable client-side: our SDKs ship `generateAndWait()` sugar
   (submit + poll internally) — junior-friendly ergonomics without server-held state. MCP's
   `wait_for_job` does the same for agents.
4. Sync providers still fit fine internally: adapter `dispatch()` just returns
   `Completed(outputs)` and the job goes terminal in one worker pass. Sync upstream ≠ sync
   downstream.

## Your Axum question: holding many open connections

If we *did* hold lots of requests open (we still will for LLM SSE):

- Parked async connections are nearly free in both frameworks: a waiting request is a
  suspended future (~KB of state) + a socket; no thread is consumed. The binding limits are
  file descriptors, socket buffers, and memory — not the web framework.
- **Axum's work-stealing helps** the moment load is uneven: any worker thread can resume any
  ready connection, so one slow/busy core can't strand thousands of parked requests behind
  it. Actix's per-worker single-threaded runtimes pin each connection to the worker that
  accepted it — wake-ups queue behind whatever that worker is doing. Both are "fine"; Axum
  degrades more gracefully at the tail.
- Scale numbers (order of magnitude, single modest box): tens of thousands of concurrently
  held idle connections is routine for hyper/tokio; 100k+ is achievable with fd limits and
  ~2–4 GB of RAM for connection state. Our realistic held-open load (LLM streams, seconds
  each) is thousands at most before revenue says scale horizontally anyway.

Conclusion: the decision to drop media sync isn't capacity-driven — Axum could hold them — it
is a product-surface decision. Axum remains the right choice for the LLM streaming we do keep.
