# APIs: LLM (the chat plane)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Peeled from `api.md` §5. One of the three
generation planes: this doc (chat), `api_all.md` (the Any API), `api_model_specific.md`
(unified + granular). Shared conventions (auth, errors, idempotency, cost twins, `end_user`):
`api.md` §1–§3.

---

## 1. Namespace & positioning

**Sign-off:** _unsigned_

**Namespace: `/v1/chat/*` — the path IS the compatibility contract.** This plane is strictly
OpenAI-wire-compatible; its value is that existing OpenAI/OpenRouter code works by changing
the base URL and key. So the namespace isn't ours to invent: `POST /v1/chat/completions`,
verbatim. Anything we add to the LLM plane lives beside it under paths the ecosystem already
expects (`/v1/completions`, `/v1/embeddings`, `/v1/batch` — reserved, §6).

| Endpoint                | Method | Status    | Purpose                                        |
| ----------------------- | ------ | --------- | ---------------------------------------------- |
| `/v1/chat/completions`  | POST   | v1        | OpenAI-compatible chat; SSE via `stream: true` |
| `/v1/completions`       | POST   | RESERVED  | Legacy text-completion (RP vertical; `future-roadmap.md` §5) |
| `/v1/embeddings`        | POST   | RESERVED  | (`future-roadmap.md` §1)                        |
| `/v1/batch`             | POST   | RESERVED  | (`future-roadmap.md` §1)                        |

Execution model: **synchronous pass-through** — held-open HTTP (or SSE), bypasses the job
queue entirely (no polling, no webhooks), but writes a `jobs` row (`media_type = text`,
`api_shape = chat`, born `running`, terminal seconds later) so billing, spend history, logs,
and rankings are uniform with media (`architecture.md` §7).

## 2. Request: OpenAI schema + pass-through

**Sign-off:** _unsigned_

`model` = our catalog slug (`deepseek_v4`, `openai_gpt_v5_1_snap_20251113`, `..._latest` —
full `ModelRef` grammar, `models.md` §13). Everything else is the OpenAI chat schema:

| Param class                     | Handling                                                       |
| ------------------------------- | --------------------------------------------------------------- |
| Core (`messages`, `max_tokens`/`max_completion_tokens`, `stream`, `stop`, `n=1`) | Honored everywhere |
| Standard samplers (`temperature`, `top_p`, `seed`, `presence_penalty`, `frequency_penalty`, `logit_bias`, `logprobs`) | **Passed through** to the provider; never reinterpreted by us |
| Extended samplers (`top_k`, `min_p`, `repetition_penalty`; later DRY/XTC) | **Pass-through when the provider supports them**, pruned otherwise — the sampler pass-through commitment from the RP analysis (`future-roadmap.md` §5) |
| Tools (`tools`, `tool_choice`, parallel calls) | Translated per provider (well-trodden adapter territory)  |
| `response_format` (json_object / json_schema) | Translated where supported; pruned + reported where not   |
| `user`                          | Mapped to our `end_user` (`end-users.md`); `end_user` also accepted directly |
| Unknown params                  | Pruned, recorded as `param_adjustments` on the job + debug logs; never a 400 (compat: OpenAI clients send vendor params freely) |

## 3. Response

**Sign-off:** _unsigned_

OpenAI response shape, with deliberate deltas:

- **`id` is our job ID** (`job_...`), not a `chatcmpl-...` string — opaque to compat clients,
  and it makes every completion retrievable: `GET /v1/jobs/{id}`, `/v1/jobs/{id}/logs`.
- `usage` always present (prompt/completion/total tokens); on streams, sent in the final
  chunk (`stream_options.include_usage` honored, and defaulted on).
- Additive vendor block `"foundation": { "provider": "...", "charged_micros": ...,
  "param_adjustments": [...] }` — additive fields don't break OpenAI clients; name swaps
  with the product name.

Errors on this plane use the standard envelope (`api.md` §2), which is field-compatible with
OpenAI's (`error.message/type/param` present; we add `details`/`request_id`).

## 4. Billing (metered mode)

**Sign-off:** _unsigned_

Token counts are known only after completion → **metered** billing (`architecture.md` §4):
admission-check against a conservative worst-case (`max_tokens` × output rate + prompt
estimate × input rate); block with `402` if the worst case exceeds the balance; single
`job_charge` written at completion with actuals. Cost twin: `POST /v1/chat/completions/cost`
returns the worst-case estimate + the rates (same body, no call, no charge).

## 5. What this plane deliberately isn't

**Sign-off:** _unsigned_

- No conversation/thread state (`previous_response_id`-style) — stateless, industry default;
  punted (`../reports/jobs-design-considerations-2.md` §5).
- No async/queued LLM jobs in v1 — chat is inherently held-open; `/v1/batch` is the future
  async lane.
- No Anthropic-native `/v1/messages` compat endpoint in v1 — the OpenAI dialect is the
  lingua franca; revisit on demand.
- Server-side web search / tool execution: roadmap (`future-roadmap.md` §1).

LLMs are also reachable through the other two planes — `POST /v1/any/text` (`api_all.md` §4)
and `POST /v1/models/{model}/generate/chat` (`api_model_specific.md` §4) — both synchronous,
both speaking our **canonical** chat schema rather than the OpenAI wire dialect. This plane
exists for compatibility; those exist for uniformity.
