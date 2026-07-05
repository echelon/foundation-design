# Providers (committed integrations)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). The providers we KNOW we're integrating and must
write client crates for, each mapped to the models it supplies. Rule: **no aggregators (Fal,
Replicate, etc.) to satisfy coverage — we compete with them directly.** Candidates for
future waves: `../reports/other-providers.md`. Capacity figures are best-verified estimates;
items marked TBV get confirmed during integration.

---

## 1. Provider table

**Sign-off:** _unsigned_

| Provider    | Crate               | Supplies                          | Billing (theirs)                | API type                              | Webhooks? |
| ----------- | ------------------- | --------------------------------- | ------------------------------- | -------------------------------------- | --------- |
| OpenAI      | `clients/openai`    | gpt-image, Sora video, LLMs       | Prepaid credits (self-serve)    | Images sync; video create→poll         | Yes (video events) |
| Google      | `clients/google`    | Nano Banana, Veo, Gemini LLMs     | Postpaid (Cloud billing)        | Image sync; Veo LRO create→poll        | No        |
| ByteDance   | `clients/bytedance` | Seedance video, Seedream image    | Prepaid account balance         | Video create→poll; image sync          | No        |
| Kling       | `clients/kling`     | Kling video, lipsync, avatar      | Prepaid resource packages       | Create→poll                            | Yes (`callback_url`) |
| BFL         | `clients/bfl`       | FLUX family (all)                 | Prepaid credits (1cr = $0.01)   | Submit→poll returned `polling_url`     | Yes (config TBV) |
| xAI         | `clients/xai`       | Grok Imagine image+video, LLMs    | Prepaid credits                 | Image sync; video deferred→poll        | No        |
| Anthropic   | `clients/anthropic` | Claude LLMs                       | Prepaid credits / invoicing     | Held-open POST + SSE                   | n/a       |
| Vidu        | `clients/vidu`      | Vidu Q video                      | Prepaid credits                 | Create→poll                            | Yes (`callback_url`) |
| World Labs  | `clients/worldlabs` | Marble worlds/splats              | Prepaid credits (never expire)  | LRO create→poll (polling free)         | No (TBV)  |

Preference note: we favor poll- or webhook-based partners; the sync-only providers here
(gpt-image, Nano Banana, Seedream, Grok image) are fine because *fast image* sync fits
`dispatch → Completed(outputs)` in one worker pass. Callbacks, where offered, are an
optimization — we always also poll.

## 2. Per-provider detail

**Sign-off:** _unsigned_

### OpenAI

1. **Models:** `openai_gpt_image_v1/_mini/_v1_5/_v2`, `openai_sora_v2/_pro` (⚠ Videos API
   shutdown 2026-09-24 — build the adapter expecting a successor), LLMs
   (`openai_gpt_v5_5` family, `openai_gpt_oss_120b` via others).
2. **Billing:** self-serve prepaid credit balance; enterprise invoicing. Auto-recharge
   available on their side (we still treat quota exhaustion as `provider_capacity`).
3. **API type:** images = synchronous POST (b64 body; multipart for edits). Video = create →
   poll `GET /v1/videos/{id}` + webhook events (`video.completed/failed`) — we poll AND
   subscribe. LLM = held-open + SSE pass-through.
4. **Capacity:** usage-tier system (tiers rise with cumulative spend); per-model RPM/TPM
   published per tier — image RPM at mid tiers is comfortably thousands/min. Highest-capacity
   provider on this list. Exact tier targets TBV at integration.

### Google (Gemini API; Vertex later if quotas demand)

1. **Models:** `google_nano_banana*` (image, incl. Interactions API migration),
   `google_veo_v3*` (video), `google_gemini_*` (LLM), later `google_lyria_v3` (music),
   `google_gemini_v3_1_flash_tts`.
2. **Billing:** postpaid pay-as-you-go on Cloud Billing (the one postpaid launch provider —
   different cash-flow shape; budget alerts + our own spend caps required).
3. **API type:** image = sync `generateContent`/Interactions (inline b64 — WE upload to R2).
   Veo = `:predictLongRunning` → poll operation until `done`; output URI download authed,
   2-day retention (ingest immediately). No webhooks anywhere.
4. **Capacity:** tiered by billing account history; Veo preview quotas are LOW (single-digit
   concurrent ops initially — TBV); Vertex AI route exists for quota increases at the cost of
   OAuth + `:fetchPredictOperation` differences. Plan Veo capacity escalation early.

### ByteDance (BytePlus ModelArk intl + Volcano Engine CN)

1. **Models:** `bytedance_seedance_*` (video), `bytedance_seedream_*` (image); later Seed
   Audio family.
2. **Billing:** prepaid account balance per platform — and the platforms are **disjoint**
   (different hosts, keys, model IDs: `dreamina-*` vs `doubao-*`). Two credential pools
   minimum; ArtCraft ran three (Volcengine/BytePlus/Ultra) — we inherit that pattern.
3. **API type:** video = `POST /api/v3/contents/generations/tasks` → poll task id
   (`queued/running/succeeded/failed`); cancel via DELETE. Image = sync
   `/api/v3/images/generations`. No callbacks. Asset URLs ~24h (TBV) — ingest fast.
4. **Capacity:** per-model concurrent-task quotas per account; multiple platform accounts
   multiply capacity (the reason `provider_account` pools exist). Exact quotas negotiated —
   ArtCraft experience says this is the provider where account multiplexing matters most.

### Kling (Kuaishou)

1. **Models:** `kling_video_*` (all tiers/versions), `kling_lipsync`, `kling_avatar_v2_pro`,
   Kling image gen later.
2. **Billing:** prepaid **resource packages** (bundles bought per model family; concurrency
   entitlements come WITH the package tier).
3. **API type:** create → poll (`submitted/processing/succeed/failed`), optional
   `callback_url` push + `external_task_id` correlation (we set it to our `job_` id — free
   reconciliation). Auth is the odd one: **self-signed HS256 JWT** from AccessKey/SecretKey,
   30-min expiry → token cache in the client crate. Assets purged after 30 days.
4. **Capacity:** concurrency scales with purchased package size (single-digit → dozens of
   concurrent tasks; TBV per package). Docs portal blocks scrapers — verify numbers in
   console during onboarding.

### Black Forest Labs

1. **Models:** `bfl_flux_*` — the whole FLUX catalog directly from the lab (pro/ultra/
   kontext/fill/FLUX.2 line).
2. **Billing:** prepaid credits, 1 credit = $0.01, Stripe top-ups.
3. **API type:** submit → poll the **returned `polling_url`** (must use it verbatim — they
   shard result hosts; never reconstruct URLs). Webhooks supported (mechanics TBV). Regional
   + global endpoints.
4. **Capacity:** **fixed concurrency, not spend-tiered:** 24 concurrent tasks on most
   endpoints, 6 on `flux-kontext-max`; 429 + backoff beyond. Our scheduler must treat BFL as
   a fixed-width pipe per key (multiple keys TBV).

### xAI

1. **Models:** `xai_grok_imagine_image*` (sync), `xai_grok_imagine_video*` (deferred→poll),
   `xai_grok_v4_3`+ LLMs, later `xai_grok_voice_tts_v1`.
2. **Billing:** prepaid credits on the team account.
3. **API type:** image = synchronous OpenAI-style `/v1/images/generations` (+edits). Video =
   deferred: POST → `request_id` → `GET /v1/videos/{request_id}`
   (`pending/done/failed/expired`). Temp asset URLs — ingest promptly. LLM = held-open+SSE.
4. **Capacity:** per-model RPM/TPM per team, raised on request/spend (TBV). ArtCraft's
   `grok_api_client` + video poller adapt directly.

### Anthropic

1. **Models:** `anthropic_claude_*` (Opus 4.8, Sonnet 5, Haiku 4.5, Fable 5).
2. **Billing:** prepaid credits self-serve; monthly invoicing at scale.
3. **API type:** Messages API, held-open + SSE; our chat adapter translates from the
   OpenAI-compatible wire shape (system prompt placement, content blocks, tools).
4. **Capacity:** usage tiers (1–4 + custom) by spend history; per-model RPM/ITPM/OTPM.
   Comfortable for launch LLM volumes; enterprise tier when needed.

### Vidu

1. **Models:** `vidu_v2`, `vidu_q1/q2/q3*`.
2. **Billing:** prepaid credits; `off_peak` discount option (worth surfacing as a request
   flag — cheaper non-urgent generations).
3. **API type:** `POST /ent/v2/{task}` → `GET /ent/v2/tasks/{id}/creations`
   (`created/queueing/processing/success/failed`), optional `callback_url`. Auth scheme is
   the nonstandard `Token` header. Result URLs ~1h (TBV) — ingest immediately.
4. **Capacity:** concurrency tiered by plan/spend (TBV at onboarding).

### World Labs

1. **Models:** `worldlabs_marble_v1_1`, `worldlabs_marble_v1_1_plus` (splats/worlds).
2. **Billing:** prepaid credits, $1 = 1,250 credits, $5 minimum, **never expire**, their own
   auto-refill; world gen costs 150–1,500 credits; they 402 on exhaustion (kindred spirits).
3. **API type:** `POST /marble/v1/worlds:generate` → long-running operation → poll (polling
   is free). Webhooks not documented. Outputs SPZ/PLY + mesh — our splat media pipeline
   normalizes on PLY + SPZ.
4. **Capacity:** no published tiers (TBV); volumes in this media type are low-rate/high-value
   — capacity unlikely to bind early.

## 3. Model coverage map

**Sign-off:** _unsigned_

| Provider    | Models covered (models.md slugs)                                                  |
| ----------- | ---------------------------------------------------------------------------------- |
| OpenAI      | `openai_gpt_image_v1`, `_v1_mini`, `_v1_5`, `_v2`, `openai_sora_v2`, `_pro`, `openai_gpt_v5_5*`, `openai_gpt_v5_4*`, `openai_gpt_v5_3_codex`, `openai_o4_mini`, `openai_gpt_4o_mini_tts`, `openai_gpt_4o_transcribe`, `openai_whisper_large_v3` |
| Google      | `google_nano_banana`, `_v2`, `_v2_lite`, `_pro`, `google_imagen_v4` (†dies 08-17), `google_veo_v2/v3/v3_fast/v3_1/v3_1_fast/v3_1_lite`, `google_gemini_omni_flash`, `google_gemini_*` LLMs, `google_gemma_v4_31b`, `google_lyria_v3*`, `google_gemini_v3_1_flash_tts`, `google_chirp_v3` |
| ByteDance   | `bytedance_seedance_v1_lite/v1_pro/v1_5_pro/v2/v2_fast/v2_mini`, `bytedance_seedream_v3/v4/v4_5/v5_lite`, `bytedance_dreamina_v3_1` (TBV on intl) |
| Kling       | `kling_video_*` (all 13), `kling_lipsync`, `kling_avatar_v2_pro`                    |
| BFL         | `bfl_flux_v1_schnell/dev/krea_dev`, `bfl_flux_pro_v1_1/_ultra`, `bfl_flux_v2_dev/pro/flex/max/klein_9b/klein_4b`, `bfl_flux_kontext_dev/pro/max`, `bfl_flux_fill_pro` |
| xAI         | `xai_grok_imagine_image`, `_quality`, `xai_grok_imagine_video`, `_v1_5`, `xai_grok_v4_3/v4_20/build_v0_1`, `xai_grok_voice_tts_v1` |
| Anthropic   | `anthropic_claude_opus_v4_8`, `_sonnet_v5`, `_sonnet_v4_6`, `_haiku_v4_5`, `_fable_v5` |
| Vidu        | `vidu_v2`, `vidu_q1`, `vidu_q2`, `vidu_q3`, `vidu_q3_turbo`                          |
| World Labs  | `worldlabs_marble_v1_1`, `worldlabs_marble_v1_1_plus`                                |

## 4. Models with NO committed provider

**Sign-off:** _unsigned_

Everything else in models.md. Three tiers (full candidate analysis:
`../reports/other-providers.md`):

**A. Owner has a self-serve API — integrate the owner directly (wave 2+):**
Recraft, Ideogram, Luma (Ray/Photon), Runway, PixVerse, MiniMax (video/music/speech + M3
LLM), Stability, Lightricks LTX, Alibaba (Wan + Qwen via Model Studio intl), Tencent (3D
only), Moonshot/DeepSeek/Mistral/Meta-hosts (LLMs via their platforms or compute), ElevenLabs,
Cartesia, Deepgram, AssemblyAI, Treblo, Mureka, Kits, sync.so, HeyGen, Topaz, Tripo, Meshy,
Hyper3D Rodin, SpAItial (Echo splats).

**B. Open weights, no owner API — self-host lane (RunPod/Modal, per
`../reports/other-service-offerings.md`) or defer:**
`birefnet_v2`, `rembg`, `esrgan`, `clarity_upscaler`, `aura_sr`, `supir`, `codeformer`,
`gfpgan`, `ddcolor`, `lama`, `ic_light_v2`, `idm_vton`, `omnigen_v2`, `kokoro`, `f5_tts`,
`mmaudio_v2`, `ace_step`, `genmo_mochi_v1`, `zhipu_cogvideox_5b`, `tencent_hunyuan_video*`,
`microsoft_trellis*`, `triposr`, `dreamgaussian`, `lgm`, `kwai_liveportrait`,
`meigen_multitalk/infinitetalk`, `flashvsr`, `rife`, `microsoft_vibevoice`,
`alibaba_qwen3_tts`, `sesame_csm_1b`, `narilabs_dia`, `canopy_orpheus`,
`resemble_chatterbox`.

**C. No path at all right now (catalog `status` handles it — list, mark unavailable, or
drop):** `midjourney_v7/v8` (no API, period), Pika (`pika_v2_2` — official page routes to
fal; partner-only), `moonvalley_marey_v1_5` (API waitlist), Suno/Udio (no APIs). These stay
in models.md as catalog knowledge; they don't ship as invocable at launch.
