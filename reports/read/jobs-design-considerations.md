# Jobs as a first-class primitive — design considerations

Report, 2026-07-05. To be promoted into `design/jobs.md` after your feedback.

## 1. The claim: everything is a job

Every generation attempt — API key, dashboard sandbox, MCP — creates exactly one job row, for
every media type **and** for LLMs. This is already the direction in schema.md; this report
stress-tests it, especially the LLM case, and redesigns the status machine.

What a job buys us per request: identity (`job_...`), the charge/refund anchor, the log
timeline anchor, the spend list row, the rankings rollup row, the webhook subject, the
idempotency subject. Anything that skipped jobs would need parallel plumbing for all seven.

## 2. Does the primitive survive LLMs? (OpenRouter's model, examined)

How OpenRouter models it: each `POST /chat/completions` is one **generation** — an id you can
look up afterward at `GET /v1/generation?id=...` for tokens/cost/latency. That's it. There is
NO server-side conversation: clients resend the full message history every call; a
"conversation" is purely a client-side construct. So the industry's biggest LLM gateway
already runs on exactly our shape — one request = one accountable unit — they just call it a
generation and never give it a lifecycle (it's born terminal).

Contrast: OpenAI's Responses API has optional server-side state (`previous_response_id`
chaining, background mode); Anthropic/Gemini are stateless like OpenRouter.

**Conclusion: the primitive holds.** An LLM job is a job that is born `running` and reaches a
terminal status seconds later, with token quantities instead of seconds/megapixels. What does
NOT belong in v1 is a **conversation/thread primitive** — the stateless model is the industry
default, costs us nothing, and a `threads` resource can be added later (as its own table
referencing jobs) if we ever want Responses-style server state. Punt explicitly.

**Continuations (your question):** two different things, keep them distinct —

1. **Data lineage** (media): extend-video / remix / upscale take a prior output as input.
   That's already modeled: job B's `request_params` reference a `file_`/output id produced by
   job A. Provenance = follow the file references. No job nesting needed.
2. **Causal lineage** (nice-to-have): an optional `parent_job_id` column for "this job was
   created FROM that job" (console "extend" button, Sora-style remix). One nullable
   self-referencing FK; powers a lineage view in the console. Recommend adding it — cheap now.

LLM "conversations" are neither: no parent pointer (the client resends history; there's no
authoritative chain), just jobs that share nothing but an account.

## 3. Status machine (the double queue)

### Inputs

**ArtCraft `JobStatusPlus`** (`crates/schema/public/enums/src/common/job_status_plus.rs`):
`pending, started, complete_success, complete_failure, attempt_failed, dead,
cancelled_by_user, cancelled_by_system`. Lessons worth stealing: `attempt_failed` (retryable
failure distinct from terminal), `dead` (retries exhausted ≠ ordinary failure), cancellation
split by actor. Lesson to fix (as you noted): single queue — no state distinguishing "queued
with us" from "queued at the provider".

**The field:**

| Platform      | Statuses                                                             |
| ------------- | --------------------------------------------------------------------- |
| Fal           | `IN_QUEUE → IN_PROGRESS → COMPLETED` (failure discovered in the result) |
| Replicate     | `starting → processing → succeeded / failed / canceled`               |
| Kie           | `waiting → queuing → generating → success / fail`                     |
| OpenAI video  | `queued → in_progress → completed / failed`                           |
| Kling         | `submitted → processing → succeed / failed`                           |
| Vidu          | `created → queueing → processing → success / failed`                  |
| OpenRouter    | (LLM: none — born terminal; video jobs: submitted → ready)            |

Note Kie and Vidu already surface a **two-stage queue** (`waiting` vs `queuing`) — precedent
for exactly your 429 concern.

### Proposed status enum (customer-visible)

| Status            | Meaning                                                                  |
| ----------------- | -------------------------------------------------------------------------- |
| `queued`          | Accepted + charged; in OUR queue. Not yet accepted upstream. Covers 429-from-provider backoff: job stays `queued`, retry metadata visible (`dispatch_attempts`, `next_dispatch_at`), events show each attempt |
| `provider_queued` | Submitted; sitting in the PROVIDER's queue (their task id exists)           |
| `running`         | Provider reports active generation                                          |
| `succeeded`       | Terminal. Set only AFTER outputs are ingested into our storage              |
| `failed`          | Terminal; `failure_code` says why; auto-refunded                            |
| `canceled`        | Terminal; `canceled_by: user \| system` as a field, not two statuses        |
| `expired`         | Terminal; passed `expires_at` before completion; auto-refunded              |

Changes vs current schema.md: `dispatched` → **`provider_queued`** (the honest name);
`canceled_by` column added; everything else survives. Mapping notes:

- `attempt_failed` (ArtCraft) = NOT a status: a `job_events` entry + `queued`/
  `provider_queued` retained. Customers see a stable state with a visible retry trail rather
  than a flapping status.
- `dead` (ArtCraft) = `failed` with `failure_code` reflecting exhaustion (e.g.
  `provider_capacity` after max attempts). No separate terminal state.
- Output ingestion is not a customer status (an `output_ingested` event between provider
  completion and `succeeded`) — customers care that succeeded ⇒ URLs are durable, ours.
- LLM jobs: `running → succeeded/failed` only; the queue states never occur.
- Sync providers (gpt-image): may skip `provider_queued` entirely (`queued → running →
  succeeded` within one worker pass). Statuses are a lattice each provider walks a path
  through, not mandatory stops.

### State transitions owned in exactly one place

`services/jobs` exposes the only transition function (`transition(job, event) -> status`),
with an exhaustive match and an invalid-transition error — every worker goes through it;
`job_events` is written in the same transaction as the status change.

## 4. Naming: "jobs", "tasks", or something else?

| Candidate     | Verdict                                                                     |
| ------------- | ----------------------------------------------------------------------------- |
| **jobs**      | RECOMMENDED. Honest for heavy async work (the core product), tolerable for the born-terminal LLM case, universally understood, no collisions in our own vocabulary |
| tasks         | Collides fatally with `queue_tasks` (internal work items ≠ customer requests); Kie uses it, but our split matters |
| runs          | Nice and short (OpenAI "runs"), but implies process-execution more than request-fulfillment; weaker for "list my video jobs" |
| generations   | OpenRouter's word; great for LLM/media gen, wrong the moment we do upscales/STT/rigging/fine-tunes ("a transcription generation"?) |
| predictions   | Replicate's word; ML-paper jargon that's aging badly (nobody calls a video a prediction) |
| requests      | Hopelessly overloaded with HTTP requests (we log those separately!)            |
| operations    | Google LRO flavor; bureaucratic; fine but charmless                            |
| invocations   | Serverless flavor; long                                                        |

Recommendation: **jobs**, everywhere, including LLMs. The ephemerality objection ("an LLM call
isn't really a *job*") is real but cosmetic — and the alternative (two primitives) costs a
second spend list, a second log anchor, and a second rankings pipeline. One caveat to
docs-writing: in LLM docs we can *say* "request" casually while the resource stays a job.

## 5. Open questions for you

1. Bless `provider_queued` (rename of `dispatched`)? And `canceled_by` as a field?
2. Add `parent_job_id` (causal lineage) to schema.md now?
3. Punt conversations/threads confirmed?
4. Should `queued` split visible sub-state for 429-backoff (`queued` + `throttled`)? My vote:
   no — one state + retry metadata; fewer states customers must handle.
5. Promote this to `design/jobs.md` once settled?
