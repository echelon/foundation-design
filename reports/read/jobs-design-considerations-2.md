# Jobs design, round 2 — queue naming, retries, lineage, threads

Report, 2026-07-05. Follow-ups from `read/jobs-design-considerations.md` after bt feedback.
Third-party claims verified against official docs today (Fal, Replicate, OpenRouter, Runway,
Luma, OpenAI); the settled parts are recorded in `../design/jobs.md`.

## 1. Naming the two-stage queue

The problem: `queued` vs `provider_queued` (or Kie's `waiting` vs `queuing`) are two states
whose names are near-synonyms because — from the customer's chair — they ARE near-synonyms:
"not running yet, be patient." Candidates:

| Scheme                          | Verdict                                                            |
| ------------------------------- | -------------------------------------------------------------------- |
| `queued` → `provider_queued`    | Honest but sibling-named; reads like a typo'd duplicate in a dashboard filter |
| `queued` → `submitted`          | Kling/Vidu-ish precedent, but backwards-feeling: the CUSTOMER submitted the job at creation; "submitted" meaning "we submitted it upstream" needs a docs footnote |
| `queued` → `dispatched`         | Current schema.md word. Says "sent," not "accepted"; a dispatch that upstream 429'd is not meaningfully dispatched |
| `waiting` → `queued`            | Kie's pair — the confusion we're escaping                             |
| `pending` → `queued`            | "Pending" is the vaguest word in the industry (Replicate uses `starting` for worker spin-up, Runway `PENDING` for enqueued…). No |
| **One `queued` + `queue_stage` field** | **RECOMMENDED** — see below                                    |

**Recommendation: stop trying to name two statuses and admit it's one status with a detail.**
`status = queued`, plus `queue_stage: gateway | provider` (and `provider_job_id` non-null as
the machine-readable equivalent). Reasons:

1. **Nobody in the field exposes a two-tier queue as statuses.** Verified today: Fal collapses
   everything pre-runner into `IN_QUEUE`; Replicate `starting`; Runway `PENDING` (its
   `THROTTLED` is a different thing — see §5); Luma `queued`; OpenAI `queued`. We'd be
   inventing a distinction every SDK switch-statement must handle but almost no customer
   wants to branch on.
2. It keeps the customer state machine at six values — every extra state is a case in every
   customer's code forever.
3. The 429 story becomes trivial: throttled dispatch is just `queued`/`gateway` with retry
   metadata. No flapping, no `throttled` state.
4. The stage detail stays available for the ones who care (dashboards, our console, debug
   timeline) without burdening the ones who don't.

Field-name candidates if blessed: `queue_stage` (recommended), `queue`, `queued_in`,
`stage`. Value candidates: `gateway | provider` (recommended — "gateway" is what we are),
`ours | theirs` (cute, unserious), `local | upstream` (fine; "upstream" leaks proxy jargon).

## 2. Internal vs external status enums — same enum

Should internal and customer-facing status lists differ? **No — one enum.** We're open
source: a private status vocabulary would be visible in the repo anyway, and a second enum
guarantees drift + a mapping layer that exists only to lie slightly. The things that feel
"internal" are not statuses:

| "Internal state" instinct       | Actual home                                             |
| ------------------------------- | --------------------------------------------------------- |
| attempt_failed / retrying       | `job_events` (`dispatch_attempted`) + `dispatch_attempts`, `next_dispatch_at` columns |
| dead (retries exhausted)        | `failed` + `failure_code: provider_capacity` (etc.)        |
| ingest-in-progress              | `job_events` (`provider_completed` → `output_ingested`); customers just see `running` until durable |
| worker/queue bookkeeping        | `queue_tasks.status` (`pending \| running \| done \| dead`) — a different table with its own lifecycle; never customer-visible |

Precedent: Fal's public queue enum is three values while their internals obviously track far
more; the public enum is a projection. Ours is the same projection — we just publish the
projector too.

## 3. Retry in the field — verified

| Platform    | Customer-facing retry?                                                        |
| ----------- | ------------------------------------------------------------------------------ |
| Fal         | No endpoint, but **auto-retries infra failures** (503/504/conn) up to 10×, opt-out header `X-Fal-No-Retry`. No attempt counter, no retry status — invisible except the job re-enters queue |
| Replicate   | **None documented.** Error-code docs push retry onto the customer ("retry your prediction"); only webhook delivery is auto-retried |
| OpenRouter  | **Provider fallback inside a single request** (`allow_fallbacks: true` default) — a routing feature, not a job retry; only the final serving provider is visible |
| Runway      | No auto-retry; docs tell customers when re-submitting is sensible, keyed on `failureCode` (retry `INTERNAL`, never retry `SAFETY.*`) |
| Luma/OpenAI | Fail → resubmit                                                                 |

**Conclusion: your instinct is right — `attempt_failed`/`dead` as customer states is over the
top, and a retry *endpoint* is nonexistent in the field.** Making retry uniform across
providers is genuinely weird (each upstream bills, moderates, and retries differently; a
"retry" of a content-policy failure is a re-offense, of a capacity failure a re-queue, of an
invalid-input failure a bug). The honest uniform semantic is the one everyone converged on:
**terminal failure + refund + customer re-enqueues.** What we keep from Fal's playbook is
pre-acceptance infra retries (free to make invisible, since nothing was charged upstream) —
recorded in `../design/jobs.md` §3.

## 4. `parent_job_id` — who has it, what it'd be for

Field survey (verified today):

| Platform         | Lineage mechanism                                                        |
| ---------------- | -------------------------------------------------------------------------- |
| OpenAI Sora      | **The one real precedent:** `POST /videos/{id}/remix` → child job carries `remixed_from_video_id` |
| Luma             | Extend/interpolate reference the prior generation **inside the input** (`keyframes.frame0 = {type: "generation", id: ...}`) — no dedicated parent field |
| Fal / Replicate / Runway | Nothing — inputs are plain URLs/asset refs                          |
| Gemini           | `previous_interaction_id` — conversation threading, not media lineage       |

What would use ours: a console "extend"/"remix" button, a lineage view in job details,
provenance analytics. What wouldn't: the API's actual function — the child job's inputs
already reference the parent's outputs (`file_`/output ids in `request_params`), so data
provenance is recoverable by following references, Luma-style.

**Answer to "would this be wasted in 99% of cases?": yes.** ~100% of LLM jobs, all
first-generation media, everything API-driven without our console — realistically >99% NULL.
The storage cost is nothing (nullable text in Postgres); the real cost is API surface: a
documented field implies query filters, webhook payload presence, docs, and a semantics
argument (is an upscale-of-a-frame-of-a-video a child?).

**Recommendation: skip it for v1.** It's purely additive later — one nullable
self-referencing FK + backfill-free migration the week the console grows an "extend" button.
Nothing in today's design blocks it; deferring costs zero.

## 5. Threads/conversations vs `parent_job_id` — different concepts, both punted

Your question: is OpenAI's `previous_response_id` a `parent_job_id`? **No — it's server-side
context threading.** Semantics (verified): the server reconstructs the prior conversation so
the client doesn't resend history; billing is unchanged ("all previous input tokens for
responses in the chain are billed as input tokens") — it's a convenience pointer, not state
compression. OpenAI now also has a durable Conversations object; Gemini's new Interactions
API stores by default with `previous_interaction_id`; **Anthropic remains fully stateless**,
and OpenRouter has no server-side state at all.

So there are two distinct concepts, and we're punting both:

| Concept          | What it is                          | Verdict                              |
| ---------------- | ------------------------------------ | ------------------------------------- |
| `parent_job_id`  | Causal media lineage (remix-of)      | Skip v1 (§4); additive later          |
| Threads          | Server-held conversation state       | Punt — stateless chat is the industry default and what OpenRouter-compatible clients expect; a `threads` table referencing jobs can be added later without touching the job primitive |

"Maybe this whole thing is bad and we should just abandon the concept. Simplicity wins?" —
agreed for v1, with the caveat that the market is drifting statefull-ward (OpenAI and Google
both shipped server state in the last year). If that becomes table-stakes for agentic
workloads, it arrives as a NEW resource (threads/conversations) that references jobs; the job
primitive doesn't change either way. Nothing to reserve except not squatting on
`/v1/conversations`.

## 6. The 429/throttle story, spelled out

Your reading is correct. Sequence when a provider throttles us:

1. Job is charged and `queued` (stage `gateway`). Worker picks the dispatch task, calls the
   provider, gets 429.
2. **Nothing customer-visible changes**: status stays `queued`. The dispatch task reschedules
   with backoff (`next_dispatch_at`), `dispatch_attempts` increments, a `dispatch_attempted`
   event with the 429 detail lands in `job_events` (visible in `/v1/jobs/{id}/logs`).
3. Retry succeeds → provider accepts → stage `provider` (their queue), then `running` as
   normal.
4. Backoff budget exhausted (or `expires_at` passes first) → `failed` with
   `provider_capacity` (or `expired`), auto-refund.

Why not a visible `throttled` status: it flaps (429 → accepted → 429 across attempts), every
customer integration must handle it, and it discloses nothing actionable — the customer can't
un-throttle Kling. Runway's `THROTTLED` looks like precedent but is a different mechanism:
it's *their own* per-org concurrency hold ("stored but not enqueued"), i.e. admission
control, not upstream backoff. Our equivalent of that concept is the 402/spend-limit path at
admission, not a job state.

With the §1 recommendation the nomenclature confusion dissolves: there is no
"waiting vs queuing vs pending" triple — there is `queued`, and a `queue_stage` detail saying
whose queue, and metadata saying why it's still there.
