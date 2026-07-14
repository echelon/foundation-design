# Other providers: filling the rest of the model catalog

Report, 2026-07-05. Companion to `../design/providers.md` (the committed nine). Goal: a
provider suggestion for **every** model in `design/models.md`, without using aggregators as a
crutch. Verified against official docs 2026-07-05; TBV = re-verify at integration.

## 1. Owner APIs, ranked by integration friendliness

### Webhook-native (our preferred shape)

| Owner        | Covers (our slugs)                          | Billing                       | Notes                                            |
| ------------ | -------------------------------------------- | ----------------------------- | ------------------------------------------------- |
| Ideogram     | `ideogram_v3/v4/character`                   | Prepaid + auto top-up         | Async endpoint with **Ed25519-signed webhooks** + poll fallback; 10 inflight default |
| Luma         | `luma_ray_v2/_flash/v3_2`, `luma_photon*`    | Prepaid API credits           | Poll + `callback_url` (3 retries); Build tier: Ray 10 concurrent, Photon 40; $5k/mo cap then Scale plan |
| PixVerse     | `pixverse_v5/v6/c1`                          | Prepaid packs + memberships   | Poll + webhooks; per-request `Ai-trace-id` required |
| MiniMax      | `minimax_hailuo_*`, `minimax_music_v2_6`, `minimax_speech_*`, `minimax_m3` LLM | Prepaid + token plans (two key systems!) | Video poll ~10s + webhooks; per-model RPM low (video ~5 RPM TBV) — capacity planning needed |
| sync.so      | `sync_lipsync_v2/_pro`                       | Subscription + per-frame      | **Webhook-first** (HMAC-signed, no retries — poll fallback mandatory); concurrency by plan (1→15) |
| HeyGen       | `heygen_avatar_v4`                           | Prepaid USD wallet            | Webhook-first + poll; ~10 concurrent              |
| Treblo       | `sonauto_v2` (rename to `treblo_*`?)         | Subscription + credit packs   | `webhook_url` + poll                              |

### Polling-only (fine — our poller is the primary path anyway)

| Owner        | Covers                                       | Billing                       | Notes                                            |
| ------------ | -------------------------------------------- | ----------------------------- | ------------------------------------------------- |
| Runway       | `runway_gen_v4_turbo/_aleph`                 | Prepaid credits + autobilling | Poll `GET /v1/tasks/{id}` ≥5s; **spend-tiered concurrency** (5 tiers by cumulative purchases: 20 concurrent at tier 1 → $100k/mo caps) |
| Stability    | `stability_sd_v3_5_*`, `stability_stable_audio_v3/v2_5` | Prepaid credits (1cr=$0.01) | Images sync, video async poll; 150 req/10s flat  |
| Lightricks   | `lightricks_ltx_v2/v2_3`                     | Prepaid credits (TBV)         | First-party LTX API + open weights; limits unpublished |
| Alibaba      | `alibaba_wan_v2_2..v2_7`, `alibaba_qwen_image*`, `alibaba_happy_horse*`, Qwen LLMs | **Postpaid** (Alibaba Cloud) | Model Studio intl (SG/US/DE); Wan async-only, poll ~15s; ⚠ endpoint migration to workspace-scoped hosts in progress; ~2 concurrent video tasks typical (TBV) |
| Tencent Cloud| `tencent_hunyuan3d_*` (3D ONLY)              | Cloud account billing         | SubmitJob→QueryJob; **TC3 request signing** (real integration cost); 3 concurrent default; intl API can't take user meshes. **Hunyuan VIDEO has no intl API** — self-host lane |
| Topaz        | `topaz_image_upscale`, `topaz_video_upscale` | PAYG credits + subs           | Fully async upload→poll→download                  |
| Tripo        | `tripo_v2_5/h3_1` (+rig/animate/retexture)   | Prepaid credits ($0.01/cr)    | Poll; webhooks TBV                                |
| Meshy        | `meshy_v6` (+remesh/UV/rig/animate suite)    | Subscription credits          | Poll; Pro 20 RPS; the most complete 3D-utility API |
| Hyper3D      | `hyper3d_rodin_v2_5`                         | **Business plan gate** ($120/mo) + credits | Poll ~5s; PBR + retexture endpoints         |
| Mureka       | (candidate music addition)                   | Prepaid credits (12-mo validity) | Poll + optional stream_url; extend/inpaint/stems — richest music API grammar |
| Kits.AI      | (voice conversion, future)                   | Subscription from $14.99      | Job poll; TTS removed from API — VC/blend/separation only |
| World-class audio: ElevenLabs / Cartesia / Deepgram / AssemblyAI | all our audio slugs | Subs+credits / PAYG | All self-serve; already assumed in models.md |
| SpAItial     | `spaitial_echo_v1/v2` (NEW — added to models.md) | TBV                       | The ONLY other hosted splat API (SPZ/PLY/SOG out, async, ships an MCP server); partially verified — site blocks fetchers |
| Recraft      | `recraft_v3/v4/_pro`                         | Prepaid "API Units" ($1/1000, never expire) | **Sync** (OpenAI-images-style); rate limits undocumented |

### LLM-only platforms (for the language catalog)

DeepSeek, Moonshot, Mistral, Z.AI, StepFun, Xiaomi, Alibaba, xAI, Google, Anthropic, OpenAI
all run self-serve platforms (prepaid balances, OpenAI-compatible or near). Meta Llama /
gpt-oss / Gemma / Nemotron open weights → serve via compute lane or a serverless-inference
partner (Fireworks/Together/DeepInfra — see `other-service-offerings.md`; these are
infrastructure vendors, not aggregator-competitors, and are legitimate to use).

## 2. The self-host lane (open weights, no owner API)

Per `other-service-offerings.md`: RunPod Serverless (REST-created endpoints, S3-staged
weights) or Modal for the OSS utility fleet — `birefnet_v2`, `rembg`, `esrgan`,
`clarity_upscaler`, `aura_sr`, `supir`, `codeformer`, `gfpgan`, `ddcolor`, `lama`,
`ic_light_v2`, `idm_vton`, `omnigen_v2`, `kokoro`, `f5_tts`, `narilabs_dia`,
`canopy_orpheus`, `resemble_chatterbox`, `sesame_csm_1b`, `microsoft_vibevoice`,
`alibaba_qwen3_tts`, `mmaudio_v2`, `ace_step`, `mochi`, `cogvideox`, `hunyuan_video`,
`trellis`, `triposr`, `dreamgaussian`, `lgm`, `liveportrait`, `multitalk`, `infinitetalk`,
`flashvsr`, `rife`. Economics: one L40S/A100 pool serves the whole image-utility set;
video/3D OSS models need H100-class and are lower priority. This lane doubles as the proving
ground for the future BYOM product.

## 3. No path (catalog-listed, not invocable)

| Model                   | Status                                                        |
| ----------------------- | -------------------------------------------------------------- |
| Midjourney v7/v8        | Still no API (2026-07); enterprise-API rumors contradicted. Wrappers violate ToS — not touching them |
| Pika 2.2                | Official page routes developers to fal; direct = partnership conversation |
| Moonvalley Marey        | API waitlisted; access via partners/enterprise only            |
| Suno / Udio             | No public APIs (Suno "exploring", partner-gated; Udio walled garden) |
| Genmo Mochi             | Weights only → self-host lane                                  |
| Tencent Hunyuan Video   | No intl API → self-host lane                                   |

## 4. Aggregators (noted, per your instruction — NOT for coverage)

Fal, Replicate (Cloudflare), Novita, Kie, AtlasCloud, RunPod Public Endpoints, HF Inference
Providers. We compete with the first group directly; using them as upstreams would put our
COGS and availability in a competitor's hands and leak our demand data to them. Two nuanced
exceptions worth acknowledging: (a) serverless-inference infra (Fireworks/Together/DeepInfra)
for open-weights LLMs is a supplier relationship, not aggregation-through-a-competitor;
(b) if a must-have model is EXCLUSIVE to an aggregator (Pika on fal today), the honest
choices are partnership with the owner or absence from our catalog — not quiet resale
through a rival.

## 5. Suggested wave plan

| Wave | Providers                                                            | Rationale                                  |
| ---- | --------------------------------------------------------------------- | ------------------------------------------- |
| 1    | The committed nine (`providers.md`)                                    | Core media + LLM breadth                    |
| 2    | MiniMax, Luma, Runway, Ideogram, Recraft, Stability                    | Big video/image names, self-serve, verified |
| 3    | ElevenLabs, Cartesia, Deepgram, AssemblyAI, Treblo, Mureka             | The audio buildout (per audio-taxonomy)     |
| 4    | Tripo, Meshy, Hyper3D, Tencent 3D, SpAItial                            | 3D/splat depth + utility suite              |
| 5    | Self-host lane (OSS utilities on RunPod)                               | Cheap utility models + BYOM proving ground  |
| —    | Alibaba, PixVerse, Lightricks, Topaz, sync.so, HeyGen, Kits            | Slot into waves 2–4 by demand signal        |
