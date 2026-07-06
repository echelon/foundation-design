# Evaluating the three-service sync/async split

Report, 2026-07-05. Your preliminary design:

1. One microservice accepting all model-API / Any-API requests — traditional async (poll a
   job id, or accept a push from us).
2. One microservice for LLM synchronous connections (OpenRouter-style).
3. One microservice for media-gen synchronous connections ("seems bad, but if we provide it,
   it's its own independently scaling service").

## Verdict up front

The **decomposition is right** — these three workloads genuinely have different connection
lifetimes, drain semantics, and scaling curves, and they should never share a process's
fate. Two amendments: (a) at launch, make them **three routers/listeners with three deploy
configs of one binary**, not three codebases — the split is real at the *deployment* layer,
free at the *code* layer; (b) service #3 should stay a design slot, not a launch commitment
(consistent with the no-media-sync decision — `reports/sync-apis.md`).

## Why the split is architecturally sound

| Plane            | Connection lifetime  | Deploy/drain story                          | Scaling driver                  |
| ---------------- | -------------------- | -------------------------------------------- | -------------------------------- |
| 1. Async accept  | Milliseconds         | Restart anytime; nothing held                | Request rate (cheap, CPU-light)  |
| 2. LLM sync      | Seconds–minutes (SSE)| Drain = stop accepting, let streams finish (bounded by max generation time) | Concurrent held streams (memory + fd) |
| 3. Media sync    | **Minutes** (video…) | Drain = minutes-long; deploys crawl; a crash mid-hold loses results the client can't recover without... polling | Held connections × job duration |

Mixing plane 2 or 3 into plane 1 poisons the cheap plane's deploys: you can't roll the job-
accept path in seconds if the same process is nursing 90-second video holds. Isolating them
also isolates failure (an SSE-stream memory spike can't take down job acceptance) and lets
timeouts/load-balancer settings differ per plane (LB idle timeouts must exceed generation
time on planes 2/3; plane 1 wants tight timeouts).

This matches how the incumbents actually run: Fal literally separates hosts — `queue.fal.run`
(async) vs `fal.run` (sync) — and OpenRouter is a single-plane product (only #2 exists),
which is why it's simple for them.

## Amendments

**A. One binary, three listeners — split deployments, not codebases.** All three planes share
auth, admission/billing, the catalog, logging, and the job model (~everything). Three
microservices with three repos/codepaths would triple the drift surface. Instead:
`api-server --plane=async|llm-sync|media-sync` (or env var) mounting only that plane's
routers; deploy it three times with independent scaling/drain configs. You get every
operational benefit of the split with none of the code duplication. When teams/scaling later
justify real separation, the seam is already cut. (This refines architecture.md's "one
api-server" decision: one *binary*, now up to three *deployments*.)

**B. Plane 1's "push from us": prefer SSE + webhooks over raw WebSockets.** You said "poll on
a job id or accept a websocket push from us." Three push options, evaluated:

| Mechanism                     | Verdict                                                              |
| ----------------------------- | ---------------------------------------------------------------------- |
| Webhooks (already designed)   | KEEP — the server-to-server standard; survives client restarts        |
| **SSE job stream** (`GET /v1/jobs/{id}/stream`) | **ADD** — half the reason people want "push": browsers, CLIs, agents waiting on one job. Plain HTTP (proxies love it), auto-reconnect built into EventSource, trivially served by the async plane. Fal ships exactly this (`/status/stream`) |
| WebSockets                    | SKIP for job events — bidirectionality buys nothing here (the client only listens), and WS costs you LB config, auth-on-upgrade, and heartbeat plumbing. Reserve WS for the future realtime-audio plane where it's actually required |

So plane 1 = poll + webhook + SSE-stream-per-job. That trio covers every consumer shape
without WS operational tax.

**C. Plane 3 stays parked.** Everything from `reports/sync-apis.md` §follow-up holds: media
sync is additive later (`Prefer: wait` semantics, same Job envelope), no schema change, and
the isolation design above is exactly how it would ship — a fourth deploy config with
aggressive per-request cost caps and long LB timeouts. Design space: open. Launch scope: no.

**D. Name the planes in the docs.** Suggested vocabulary for architecture.md when this is
blessed: **accept plane** (async jobs), **stream plane** (LLM sync), **hold plane** (media
sync, future), plus the already-punted **session plane** (realtime WS audio,
`reports/audio-taxonomy.md`). Four planes, one binary, N deployments.

## What OpenRouter's shape validates

OpenRouter is "100% synchronous" only because LLMs are their only medium — a held-open HTTP
POST with SSE *is* the right shape for seconds-scale text (full transport detail in
`reports/llm-existing-synchronous-apis.md`). The moment they added video they built a job
API. We're building both planes on day one with the media plane primary; your instinct to
keep the LLM plane OpenRouter-identical (plane 2) is exactly right and requires nothing
novel: it's a pass-through proxy with billing and logging, held open for the provider's
stream duration.

## Cost/risk notes

- Plane 2 capacity is cheap: parked SSE streams are ~KBs each; a single modest box holds
  tens of thousands (`reports/sync-apis.md` §Axum). The scaling knob is fd limits + memory,
  not CPU.
- The one genuinely new component for plane 2 is **drain-aware deploys** (SIGTERM → stop
  accepting → let streams finish with a hard cap). Kubernetes/DO App Platform both support
  this via termination grace periods; set it to max-generation-time + margin.
- Billing/admission stays identical across planes (same admission transaction; LLM actuals
  charged at completion) — no per-plane billing logic.
