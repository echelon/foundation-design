# Future roadmap

**Sign-off:** bt 2026-07-06

<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Roadmap items collected from `reports/` —
**none of these are v1**, but the v1 design must cohesively allow each to be built without
rework. Every entry names the seam in today's design that keeps it buildable. When an item is
started, it graduates to its own design doc and gets struck here.

---

## 1. LLM plane

**Sign-off:** bt 2026-07-06

| Item                                  | Notes                                                              | Design seam today                                       | Source |
| ------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------------- | ------ |
| Server-side web search tool           | OpenRouter's server-side web search is the one gateway-native tool pattern with traction — a server tool that works across all models | Stream plane already proxies `tools`; a server tool is an interception layer there | `reports/later/after-v1/code-models.md` §5 |
| `/v1/completions` (raw text mode)     | RP/power users need instruct-template control; OpenRouter keeps both | Path reserved; same job/billing shape as chat            | `reports/read/role-play.md` |
| WebSocket text mode                   | Watch-item: OpenAI Responses WS mode is the first crack in "WS is voice-only"; if agentic loops normalize it, the stream plane grows a WS listener | Plane split is exactly the seam it slots into            | `reports/llm-existing-synchronous-apis.md` §5 |
| Threads/conversations resource        | Server-held conversation state (OpenAI Conversations, Gemini Interactions are drifting stateful) | A new resource referencing jobs; job primitive unchanged. Don't squat on `/v1/conversations` | `reports/jobs-design-considerations-2.md` §5 |
| `/v1/embeddings`, `/v1/batch`         | Reserved, not built                                                  | Paths reserved in `api.md` §13                            | `api.md` |

## 2. Media planes

**Sign-off:** bt 2026-07-06

| Item                                  | Notes                                                              | Design seam today                                       | Source |
| ------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------------- | ------ |
| Media sync API (hold plane)           | `Prefer: wait`-style held-open generate for fast models; explicitly NOT v1 (no-media-sync decision) | Additive: same Job envelope, no schema change; ships as a fourth deploy config with long LB timeouts + per-request cost caps | `reports/sync-and-async-apis.md` §C, `reports/sync-apis.md` |
| SSE job stream (`GET /v1/jobs/{id}/stream`) | Push for browsers/CLIs/agents waiting on one job; Fal ships exactly this. Near-term candidate — cheapest of these items | Served by the accept plane; plain HTTP; complements polling + webhooks | `reports/sync-and-async-apis.md` §B |
| Realtime session plane (WS)           | Realtime TTS/STT, S2S dialogue, Lyria RealTime — sessions can't be job-queued; needs ephemeral-token minting + WS proxy/broker | Catalog `lifecycle: session` marks these models as listed-but-not-invocable via `/v1/generate`; "session plane" named in the plane vocabulary | `reports/read/audio-taxonomy.md` §6, `reports/sync-and-async-apis.md` §D |
| `parent_job_id` (causal lineage)      | Console "extend"/"remix" lineage view; OpenAI Sora's `remixed_from_video_id` is the precedent | One nullable self-FK, additive migration; data provenance already recoverable via file references | `reports/jobs-design-considerations-2.md` §4 |
| Dubbing modality                      | Composite (STT+translate+TTS), always async — natural jobs fit, later | Just a modality + models when we carry one               | `reports/read/audio-taxonomy.md` |

## 3. Audio: the voices resource

**Sign-off:** bt 2026-07-06

v1 ships TTS with provider-preset voices only. The moment we ship cloning, **voice becomes a
platform resource** (every serious vendor converges here — ElevenLabs, ByteDance `S_xxx`
speaker ids, Suno ownership-verified Voices, Kits):

- `voices` table, `voice_` prefix, per-account.
- Kinds: `designed | instant_clone | professional_clone | licensed_library`.
- Professional clones are **async training jobs** (poll states) — the existing job machinery.
- Referenced by `voice_id` from tts, dialogue, voice_conversion, and music (singing voice).
- Cautionary tale on licensing: Uberduck's rights-holder purge — unlicensed voice catalogs
  are legal glass.

Seam today: `voice_clone`/`voice_conversion` modalities exist in `models.md` §11; the jobs
pipeline handles async training unchanged. Source: `reports/read/audio-taxonomy.md` §7
(schema sketch on request).

## 4. BYOM / compute lane

**Sign-off:** bt 2026-07-06

"Bring your own fine-tune or open-weights model; we spin it up and serve it through the same
gateway key, billing, and logs." The **(a)+(d) empty quadrant** — no gateway offers this
today; every substitute forces a second vendor relationship. Timing caveat: Cloudflare buying
Replicate says the gap won't stay open for years.

- Recommended posture: RunPod Serverless as primary spin-up backend (REST-created vLLM/SGLang
  endpoints, S3-staged weights); DeepInfra Custom / HF Endpoints as second source;
  Fireworks/Together dedicated as premium SLA tier; LoRAX-style multi-adapter serving so
  small fine-tunes share GPUs.
- Flat-rate/concurrency pricing (Featherless's mechanic) is how the long tail becomes
  economical — if we serve long-tail open weights, copy it or don't compete there.
- Seam today: **nothing structural needed.** A customer model = a catalog entry whose
  `Provider` is a compute backend (`runpod`) — jobs, billing (per-second price units exist),
  logging, rankings (private models excluded) all apply unchanged. New nouns later:
  `custom_models` resource + compute-backend adapters.

Sources: `reports/other-service-offerings.md` §4–5, `reports/read/role-play.md`.

## 5. Role-play / verticals (decision recorded)

**Sign-off:** bt 2026-07-06

Decision from `reports/read/role-play.md`, recorded verbatim (bt, 2026-07-05):

> **Recommendation:** don't chase RP as a launch vertical — the volume follows cheap
> permissive models (which we'll carry anyway: DeepSeek, Gemma, GLM via LLM providers) and
> BYOK frontends will find us if our chat API + sampler pass-through is solid. Concretely for
> v1: (1) sampler pass-through parity with OpenRouter, (2) reserve `/v1/completions`, (3)
> carry the DeepSeek/GLM/Gemma tier at good prices. The deeper RP plays — long-tail fine-tune
> hosting, persona LoRA serving, zero-log tiers — all live in the BYOM/compute lane and the
> moderation-policy decision, both already on the roadmap as their own decisions.

Roadmap items this pins:

| Item                          | Notes                                                                  |
| ----------------------------- | ------------------------------------------------------------------------ |
| Long-tail fine-tune hosting   | Featherless's moat; lives in the BYOM lane (§4)                           |
| Persona LoRA serving          | Multi-LoRA serving in the BYOM lane                                       |
| Zero-log tier per key         | **Flagged business/policy decision, not made** — ArliAI/Infermatic sell moderation posture; our logs-as-product default is the opposite posture |
| DRY/XTC samplers              | Local-only industry-wide; only closable on self-hosted lanes where we control the inference stack |
| Flat-rate pricing tier        | Tension with prepaid-USD metering; only relevant if we enter long-tail hosting |

## 6. Catalog & platform

**Sign-off:** bt 2026-07-06

| Item                          | Notes                                                       | Design seam today                             | Source |
| ----------------------------- | ------------------------------------------------------------ | ----------------------------------------------- | ------ |
| Model aliases                 | Launch with zero; add `short`/`rename`/`competitor` kinds when needed | Code-level alias map + rules already specified | `models.md` §12 |
| Pinned snapshots / `_latest`  | May not ship in v1; lint rules + catalog shape ready day one  | `models.md` §13 (DECIDED)                       | `models.md` |
| `end_user_limits`             | Per-end-user spend caps (customers ask for this eventually)   | Table sketched, prefix reserved (`eulimit_`)    | `end-users.md` |
| Email verification            | Dormant at launch; enabling = feature flag, not migration     | `email_verifications` table exists              | `accounts.md` |
| `job.started` webhook event   | Later event type alongside completed/failed                   | Events list is additive                         | `api.md` §10 |

## 7. Explicitly NOT on the roadmap

**Sign-off:** bt 2026-07-06

Recorded so they don't creep back in:

| Non-goal                      | Why                                                                     |
| ----------------------------- | ------------------------------------------------------------------------- |
| Agent execution/sandboxes     | Different business (E2B/Daytona/Modal); our MCP server is the right-sized agent play — let agents *call* us, don't run them (`reports/later/after-v1/code-models.md` §4) |
| Splitting code models from LLMs | Same media type, same wire shape; the split is a `capabilities` metadata flag (`coding_tuned`), not a category |
| Voice **agents** (stateful)   | Agent CRUD + duplex sessions + telephony is a product, not generation (`reports/read/audio-taxonomy.md`) |
| Evals platform                | Adjacent (model-migration pain), not core (`reports/evergreen/competitors.md` §4) |
| Aggregators as upstreams      | Never route through Fal/Replicate as providers — we compete with them (`providers.md`) |
