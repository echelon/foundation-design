# Model Catalog (IDs, modalities, aliases, snapshots)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Expanded from live research of fal.ai, replicate.com,
worldlabs.ai, and OpenRouter (2026-07). This is the seed list for `crates/core/catalog` —
curated-comprehensive, to be pruned to the launch set. **This document is canonical for model
slugs; `enums.md` model enums are generated/checked against it.** Reference IDs show the
upstream/competitor spelling.

---

## 1. Naming rules

**Sign-off:** _unsigned_

- Charset `[a-z0-9_]` only — every ID double-click-copyable on every platform.
- Shape: `{owner}_{family}[_v{major}[_{minor}]][_{variant}...]`. Dots become underscores after
  a literal `v` (`2.5` → `v2_5`); trailing `.0` drops (`3.0` → `v3`).
- Versionless market names stay versionless (`google_nano_banana_pro`); fused market names
  stay fused (`vidu_q3`, `moonshot_kimi_k2_6`, `tencent_hy3`).
- **Owner = model owner** (who made it), not the hosting provider. The dispatch provider is a
  separate catalog field (`Provider` enum) — e.g. `bfl_flux_v2_pro` may dispatch via fal.
- OSS utility models with no meaningful owner may omit the owner prefix (`birefnet_v2`,
  `esrgan`, `kokoro`).
- Modality/task is NEVER in the model ID — it's the endpoint (granular API) or inferred. When
  an upstream splits image/video into genuinely different models (`grok-imagine-image` vs
  `grok-imagine-video`), those are different models and the media word stays.
- Optional trailing date token pins a **snapshot** (§13).

**Terminology note ("modality" vs "task"):** the research community calls text/image/audio
*modalities* (the medium) and text-to-video/upscale/lipsync *tasks* (HuggingFace's term). Our
`media_type` already names the medium, so our "modality" is really a task, and omni-modal
models will keep blurring the word. Recommendation on file: rename the concept **task**; docs
keep "modality" pending bt's ruling (architecture.md open question #5).

## 2. Image generation models

**Sign-off:** _unsigned_

| Our ID                           | Display name           | Reference (theirs)                              |
| -------------------------------- | ---------------------- | ----------------------------------------------- |
| `bfl_flux_v1_schnell`            | FLUX.1 [schnell]       | `fal-ai/flux/schnell`                           |
| `bfl_flux_v1_dev`                | FLUX.1 [dev]           | `black-forest-labs/flux-dev`                    |
| `bfl_flux_v1_krea_dev`           | FLUX.1 Krea [dev]      | `black-forest-labs/flux-krea-dev`               |
| `bfl_flux_pro_v1_1`              | FLUX 1.1 [pro]         | `black-forest-labs/flux-1.1-pro`                |
| `bfl_flux_pro_v1_1_ultra`        | FLUX 1.1 [pro] ultra   | `black-forest-labs/flux-1.1-pro-ultra`          |
| `bfl_flux_v2_dev`                | FLUX.2 [dev]           | `black-forest-labs/flux-2-dev`                  |
| `bfl_flux_v2_pro`                | FLUX.2 [pro]           | `black-forest-labs/flux-2-pro`                  |
| `bfl_flux_v2_flex`               | FLUX.2 [flex]          | `black-forest-labs/flux-2-flex`                 |
| `bfl_flux_v2_max`                | FLUX.2 [max]           | `black-forest-labs/flux-2-max`                  |
| `bfl_flux_v2_klein_9b`           | FLUX.2 [klein] 9B      | `black-forest-labs/flux-2-klein-9b`             |
| `bfl_flux_v2_klein_4b`           | FLUX.2 [klein] 4B      | `black-forest-labs/flux-2-klein-4b`             |
| `openai_gpt_image_v1`            | GPT Image 1            | `gpt-image-1`                                   |
| `openai_gpt_image_v1_mini`       | GPT Image 1 Mini       | `gpt-image-1-mini`                              |
| `openai_gpt_image_v1_5`          | GPT Image 1.5          | `gpt-image-1.5`                                 |
| `openai_gpt_image_v2`            | GPT Image 2            | `gpt-image-2`                                   |
| `google_nano_banana`             | Nano Banana            | `gemini-2.5-flash-image`                        |
| `google_nano_banana_v2`          | Nano Banana 2          | `gemini-3.1-flash-image`                        |
| `google_nano_banana_v2_lite`     | Nano Banana 2 Lite     | `gemini-3.1-flash-lite-image`                   |
| `google_nano_banana_pro`         | Nano Banana Pro        | `gemini-3-pro-image`                            |
| `google_imagen_v4`               | Imagen 4 (+fast/ultra) | `google/imagen-4` — **dies 2026-08-17**         |
| `bytedance_seedream_v3`          | Seedream 3.0           | `bytedance/seedream-3`                          |
| `bytedance_seedream_v4`          | Seedream 4.0           | `bytedance/seedream-4`                          |
| `bytedance_seedream_v4_5`        | Seedream 4.5           | `bytedance/seedream-4.5`                        |
| `bytedance_seedream_v5_lite`     | Seedream 5.0 Lite      | `bytedance/seedream-5-lite`                     |
| `bytedance_dreamina_v3_1`        | Dreamina 3.1           | `bytedance/dreamina-3.1`                        |
| `bytedance_bagel`                | Bagel 7B               | `bytedance/bagel` (OSS)                         |
| `stability_sd_v3_5_large`        | SD 3.5 Large           | `stability-ai/stable-diffusion-3.5-large`       |
| `stability_sd_v3_5_large_turbo`  | SD 3.5 Large Turbo     | `stability-ai/stable-diffusion-3.5-large-turbo` |
| `stability_sd_v3_5_medium`       | SD 3.5 Medium          | `stability-ai/stable-diffusion-3.5-medium`      |
| `recraft_v3`                     | Recraft V3             | `recraft-ai/recraft-v3`                         |
| `recraft_v4`                     | Recraft V4             | `recraft-ai/recraft-v4`                         |
| `recraft_v4_pro`                 | Recraft V4 Pro         | `recraft-ai/recraft-v4-pro`                     |
| `ideogram_v3`                    | Ideogram 3.0           | `ideogram-ai/ideogram-v3-quality`               |
| `ideogram_v4`                    | Ideogram 4.0           | `ideogram-ai/ideogram-v4-quality`               |
| `ideogram_character`             | Ideogram Character     | `ideogram-ai/ideogram-character`                |
| `alibaba_qwen_image`             | Qwen-Image             | `qwen/qwen-image`                               |
| `alibaba_qwen_image_v2`          | Qwen-Image 2           | `qwen/qwen-image-2`                             |
| `alibaba_qwen_image_v2_pro`      | Qwen-Image 2 Pro       | `qwen/qwen-image-2-pro`                         |
| `alibaba_z_image_turbo`          | Z-Image Turbo          | `fal-ai/z-image/turbo`                          |
| `xai_grok_imagine_image`         | Grok Imagine Image     | `xai/grok-imagine-image`                        |
| `xai_grok_imagine_image_quality` | Grok Imagine Image Q   | `grok-imagine-image-quality`                    |
| `hidream_i1_full`                | HiDream-I1 Full        | `fal-ai/hidream-i1-full`                        |
| `luma_photon`                    | Luma Photon            | `luma/photon`                                   |
| `luma_photon_flash`              | Luma Photon Flash      | `luma/photon-flash`                             |
| `nvidia_sana`                    | Sana                   | `fal-ai/sana`                                   |
| `kwai_kolors`                    | Kolors                 | `fal-ai/kolors`                                 |
| `bria_v3_2`                      | Bria 3.2               | `bria/image-3.2`                                |
| `bria_fibo`                      | Bria FIBO              | `bria/fibo`                                     |
| `krea_v2_turbo`                  | Krea 2 Turbo           | `fal-ai/krea-2/turbo`                           |
| `krea_v2_large`                  | Krea 2 Large           | `krea/v2/large/text-to-image`                   |
| `midjourney_v7`                  | Midjourney V7          | no official API (aggregator/consumer only)      |
| `midjourney_v8`                  | Midjourney V8          | no official API                                 |

## 3. Image editing & utility models

**Sign-off:** _unsigned_

Note: where *editing* is the same model as generation (Nano Banana edit, Seedream edit,
Ideogram remix, FLUX.2 edit), editing is a **modality of that model** (§10), not a separate
ID. This section is models whose whole identity is editing/utility.

| Our ID                         | Display name         | Reference (theirs)                   |
| ------------------------------ | -------------------- | ------------------------------------ |
| `bfl_flux_kontext_dev`         | FLUX.1 Kontext [dev] | `black-forest-labs/flux-kontext-dev` |
| `bfl_flux_kontext_pro`         | FLUX.1 Kontext [pro] | `black-forest-labs/flux-kontext-pro` |
| `bfl_flux_kontext_max`         | FLUX.1 Kontext [max] | `black-forest-labs/flux-kontext-max` |
| `bfl_flux_fill_pro`            | FLUX.1 Fill [pro]    | `black-forest-labs/flux-fill-pro`    |
| `alibaba_qwen_image_edit`      | Qwen-Image-Edit      | `qwen/qwen-image-edit`               |
| `alibaba_qwen_image_edit_plus` | Qwen-Image-Edit Plus | `qwen/qwen-image-edit-plus`          |
| `omnigen_v2`                   | OmniGen V2           | `fal-ai/omnigen-v2` (OSS)            |
| `bria_eraser`                  | Bria Eraser          | `fal-ai/bria/eraser`                 |
| `bria_genfill`                 | Bria GenFill         | `fal-ai/bria/genfill`                |
| `bria_rmbg_v2`                 | Bria RMBG 2.0        | `bria/remove-background`             |
| `birefnet_v2`                  | BiRefNet v2          | `fal-ai/birefnet/v2` (OSS)           |
| `rembg`                        | rembg                | `fal-ai/imageutils/rembg` (OSS)      |
| `esrgan`                       | Real-ESRGAN          | `nightmareai/real-esrgan` (OSS)      |
| `clarity_upscaler`             | Clarity Upscaler     | `philz1337x/clarity-upscaler`        |
| `topaz_image_upscale`          | Topaz Image Upscale  | `fal-ai/topaz/upscale/image`         |
| `aura_sr`                      | AuraSR 4x            | `fal-ai/aura-sr` (OSS)               |
| `supir`                        | SUPIR                | `cjwbw/supir` (OSS)                  |
| `codeformer`                   | CodeFormer           | `sczhou/codeformer` (OSS)            |
| `gfpgan`                       | GFPGAN               | `tencentarc/gfpgan` (OSS)            |
| `ddcolor`                      | DDColor              | `piddnad/ddcolor` (OSS)              |
| `lama`                         | LaMa                 | `allenhooo/lama` (OSS)               |
| `ic_light_v2`                  | IC-Light v2          | `fal-ai/iclight-v2` (OSS)            |
| `idm_vton`                     | IDM-VTON             | `cuuupid/idm-vton` (OSS)             |
| `fashn_tryon_v1_6`             | FASHN Try-On 1.6     | `fal-ai/fashn/tryon/v1.6`            |

## 4. Video generation models

**Sign-off:** _unsigned_

| Our ID                        | Display name              | Reference (theirs)                        |
| ----------------------------- | ------------------------- | ----------------------------------------- |
| `kling_video_v1_6_standard`   | Kling 1.6 Standard        | `fal-ai/kling-video/v1.6/standard/...`    |
| `kling_video_v1_6_pro`        | Kling 1.6 Pro             | `fal-ai/kling-video/v1.6/pro/...`         |
| `kling_video_v2_1_standard`   | Kling 2.1 Standard        | `kwaivgi/kling-v2.1`                      |
| `kling_video_v2_1_pro`        | Kling 2.1 Pro             | `fal-ai/kling-video/v2.1/pro/...`         |
| `kling_video_v2_1_master`     | Kling 2.1 Master          | `fal-ai/kling-video/v2.1/master/...`      |
| `kling_video_v2_5_turbo_pro`  | Kling 2.5 Turbo Pro       | `fal-ai/kling-video/v2.5-turbo/pro/...`   |
| `kling_video_v2_6_pro`        | Kling 2.6 Pro             | `fal-ai/kling-video/v2.6/pro/...`         |
| `kling_video_o1`              | Kling O1                  | `fal-ai/kling-video/o1/...`               |
| `kling_video_o3_standard`     | Kling O3 Standard         | `fal-ai/kling-video/o3/...`               |
| `kling_video_o3_pro`          | Kling O3 Pro              | `fal-ai/kling-video/o3/pro/...`           |
| `kling_video_v3_standard`     | Kling 3.0 Standard        | `fal-ai/kling-video/v3/...`               |
| `kling_video_v3_pro`          | Kling 3.0 Pro             | `fal-ai/kling-video/v3/pro/...`           |
| `kling_video_v3_turbo`        | Kling 3.0 Turbo           | `fal-ai/kling-video/v3/turbo/...`         |
| `google_veo_v2`               | Veo 2                     | `veo-2.0-generate-001`                    |
| `google_veo_v3`               | Veo 3                     | `google/veo-3`                            |
| `google_veo_v3_fast`          | Veo 3 Fast                | `fal-ai/veo3/fast`                        |
| `google_veo_v3_1`             | Veo 3.1                   | `veo-3.1-generate-preview`                |
| `google_veo_v3_1_fast`        | Veo 3.1 Fast              | `veo-3.1-fast-generate-preview`           |
| `google_veo_v3_1_lite`        | Veo 3.1 Lite              | `veo-3.1-lite-generate-preview`           |
| `google_gemini_omni_flash`    | Gemini Omni Flash (video) | `google/gemini-omni-flash`                |
| `openai_sora_v2`              | Sora 2                    | `openai/sora-2` — **API dies 2026-09-24** |
| `openai_sora_v2_pro`          | Sora 2 Pro                | `sora-2-pro` — same shutdown              |
| `bytedance_seedance_v1_lite`  | Seedance 1.0 Lite         | `bytedance/seedance-1-lite`               |
| `bytedance_seedance_v1_pro`   | Seedance 1.0 Pro          | `bytedance/seedance-1-pro`                |
| `bytedance_seedance_v1_5_pro` | Seedance 1.5 Pro          | `fal-ai/bytedance/seedance/v1.5/pro/...`  |
| `bytedance_seedance_v2`       | Seedance 2.0              | `dreamina-seedance-2-0-260128` (BytePlus) |
| `bytedance_seedance_v2_fast`  | Seedance 2.0 Fast         | `bytedance/seedance-2.0/fast/...`         |
| `bytedance_seedance_v2_mini`  | Seedance 2.0 Mini         | `doubao-seedance-2-0-mini-260615` (Volc)  |
| `alibaba_wan_v2_2`            | Wan 2.2 A14B              | `fal-ai/wan/v2.2-a14b/...`                |
| `alibaba_wan_v2_5`            | Wan 2.5                   | `fal-ai/wan-25-preview/...`               |
| `alibaba_wan_v2_6`            | Wan 2.6                   | `wan/v2.6/...`                            |
| `alibaba_wan_v2_7`            | Wan 2.7                   | `fal-ai/wan/v2.7/...`                     |
| `alibaba_wan_vace_14b`        | Wan VACE 14B              | `fal-ai/wan-vace-14b` (controllable v2v)  |
| `alibaba_happy_horse`         | Happy Horse               | `alibaba/happy-horse`                     |
| `alibaba_happy_horse_v1_1`    | Happy Horse 1.1           | `alibaba/happy-horse/v1.1`                |
| `minimax_hailuo_v1`           | Hailuo video-01           | `fal-ai/minimax/video-01`                 |
| `minimax_hailuo_v2`           | Hailuo 02                 | `minimax/hailuo-02`                       |
| `minimax_hailuo_v2_3`         | Hailuo 2.3                | `fal-ai/minimax/hailuo-2.3/...`           |
| `minimax_hailuo_v2_3_fast`    | Hailuo 2.3 Fast           | `fal-ai/minimax/hailuo-2.3-fast/...`      |
| `vidu_v2`                     | Vidu 2.0                  | `fal-ai/vidu/...`                         |
| `vidu_q1`                     | Vidu Q1                   | `viduq1`                                  |
| `vidu_q2`                     | Vidu Q2                   | `viduq2`                                  |
| `vidu_q3`                     | Vidu Q3                   | `viduq3`                                  |
| `vidu_q3_turbo`               | Vidu Q3 Turbo             | `viduq3-turbo`                            |
| `pika_v2_2`                   | Pika 2.2                  | `fal-ai/pika/v2.2/...`                    |
| `pixverse_v5`                 | PixVerse V5               | `fal-ai/pixverse/v5/...`                  |
| `pixverse_v6`                 | PixVerse V6               | `fal-ai/pixverse/v6/...`                  |
| `pixverse_c1`                 | PixVerse C1               | `fal-ai/pixverse/c1/...`                  |
| `luma_ray_v2`                 | Luma Ray 2                | `fal-ai/luma-dream-machine/ray-2`         |
| `luma_ray_v2_flash`           | Luma Ray 2 Flash          | `.../ray-2-flash`                         |
| `luma_ray_v3_2`               | Luma Ray 3.2              | `luma/agent/ray/v3.2/...`                 |
| `runway_gen_v4_turbo`         | Runway Gen-4 Turbo        | `runwayml/gen4-turbo`                     |
| `runway_gen_v4_aleph`         | Runway Gen-4 Aleph        | `runwayml/gen4-aleph` (v2v editing)       |
| `lightricks_ltx_v2`           | LTX-2 (+Fast)             | `fal-ai/ltx-2/...`                        |
| `lightricks_ltx_v2_3`         | LTX 2.3 (+Fast/22B)       | `fal-ai/ltx-2.3/...`                      |
| `tencent_hunyuan_video`       | Hunyuan Video             | `tencent/hunyuan-video` (OSS)             |
| `tencent_hunyuan_video_v1_5`  | Hunyuan Video 1.5         | `fal-ai/hunyuan-video-v1.5/...`           |
| `xai_grok_imagine_video`      | Grok Imagine Video        | `xai/grok-imagine-video/...`              |
| `xai_grok_imagine_video_v1_5` | Grok Imagine Video 1.5    | `xai/grok-imagine-video/v1.5/...`         |
| `moonvalley_marey_v1_5`       | Marey Realism 1.5         | `moonvalley/marey/...`                    |
| `genmo_mochi_v1`              | Mochi 1                   | `genmoai/mochi-1` (OSS)                   |
| `zhipu_cogvideox_5b`          | CogVideoX-5B              | `fal-ai/cogvideox-5b` (OSS)               |
| `sandai_magi`                 | MAGI-1                    | `fal-ai/magi`                             |
| `meituan_longcat_video`       | LongCat Video             | `fal-ai/longcat-video/...` (long-form)    |
| `sber_kandinsky_v5`           | Kandinsky 5 (+Pro)        | `fal-ai/kandinsky5/...`                   |
| `characterai_ovi`             | Ovi (joint audio+video)   | `fal-ai/ovi`                              |
| `decart_lucy_5b`              | Lucy 5B                   | `fal-ai/decart/lucy-5b/...`               |
| `decart_lucy_edit`            | Lucy Edit                 | `decart/lucy-edit/pro`                    |

## 5. Video utility, lipsync & avatar models

**Sign-off:** _unsigned_

| Our ID                     | Display name          | Reference (theirs)                       |
| -------------------------- | --------------------- | ---------------------------------------- |
| `topaz_video_upscale`      | Topaz Video Upscale   | `topazlabs/video-upscale`                |
| `bytedance_seedvr_v2`      | SeedVR2               | `fal-ai/seedvr/upscale/video`            |
| `flashvsr`                 | FlashVSR              | `fal-ai/flashvsr/upscale/video` (OSS)    |
| `rife`                     | RIFE                  | `fal-ai/rife/video` (OSS, interpolation) |
| `bytedance_latentsync`     | LatentSync            | `bytedance/latentsync`                   |
| `sync_lipsync_v2`          | sync.so Lipsync 2     | `sync/lipsync-2`                         |
| `sync_lipsync_v2_pro`      | sync.so Lipsync 2 Pro | `sync/lipsync-2-pro`                     |
| `kling_lipsync`            | Kling LipSync         | `fal-ai/kling-video/lipsync/...`         |
| `tencent_musetalk`         | MuseTalk              | `fal-ai/musetalk` (real-time)            |
| `bytedance_omnihuman`      | OmniHuman 1.0         | `fal-ai/bytedance/omnihuman`             |
| `bytedance_omnihuman_v1_5` | OmniHuman 1.5         | `.../omnihuman/v1.5`                     |
| `kling_avatar_v2_pro`      | Kling AI Avatar 2 Pro | `fal-ai/kling-video/ai-avatar/v2/pro`    |
| `heygen_avatar_v4`         | HeyGen Avatar IV      | `fal-ai/heygen/avatar4/...`              |
| `kwai_liveportrait`        | LivePortrait          | `fal-ai/live-portrait` (OSS)             |
| `meigen_multitalk`         | MultiTalk             | `fal-ai/ai-avatar/multi` (multi-person)  |
| `meigen_infinitetalk`      | InfiniteTalk          | `fal-ai/infinitalk` (long-form)          |
| `alibaba_wan_animate`      | Wan 2.2 Animate       | `fal-ai/wan/v2.2-14b/animate/...`        |

## 6. Audio models

**Sign-off:** _unsigned_

No-API callouts: **Suno** (v5.5 — no public API; "exploring" one as of 2026-07-03), **Udio**
(UMG walled garden, no API), **PlayAI/PlayHT** (defunct — Meta acquired, API sunset
2025-12-31). Everything below has a real API path.

**Taxonomy (DECIDED 2026-07-05, from `../reports/read/audio-taxonomy.md`):** audio stays ONE
media type, split by task modalities (§11) plus a **lifecycle** axis:

- Catalog entries carry `lifecycle: sync | async_job | session`. `session` models (realtime
  TTS/STT, S2S dialogue, Lyria RealTime — websocket-native) are **listed in the catalog but
  not invocable via the generation planes (`api_any.md` etc.)**; docs point to the future realtime plane
  (`future_roadmap.md` §2). In the table below: `elevenlabs_scribe_v2_realtime` and
  `deepgram_flux` are `session`.
- **VOICE is a future platform resource** (`voice_` prefix; designed / instant clone /
  professional clone / licensed library), not a task — v1 ships provider-preset voices only;
  the resource lands with cloning (`future_roadmap.md` §3).
- Rankings/jobs need no changes — batch audio flows through the existing pipeline untouched.
- Rebrand note: Sonauto → **Treblo** (2026-06, `api.treblo.com`); slug stays `sonauto_v2`
  until they rename the model line itself.

| Our ID                            | Display name            | Reference (theirs)                              |
| --------------------------------- | ----------------------- | ----------------------------------------------- |
| `elevenlabs_v3`                   | Eleven v3               | `eleven_v3`                                     |
| `elevenlabs_multilingual_v2`      | Multilingual v2         | `eleven_multilingual_v2`                        |
| `elevenlabs_flash_v2_5`           | Flash v2.5              | `eleven_flash_v2_5` (Turbo line deprecated)     |
| `elevenlabs_scribe_v2`            | Scribe v2 (STT)         | `scribe_v2`                                     |
| `elevenlabs_scribe_v2_realtime`   | Scribe v2 Realtime      | `scribe_v2_realtime`                            |
| `elevenlabs_music_v2`             | Eleven Music v2         | `music_v2`                                      |
| `elevenlabs_sfx_v2`               | Text to Sound v2        | `eleven_text_to_sound_v2`                       |
| `cartesia_sonic_v3_5`             | Sonic 3.5               | `sonic-3.5` (snapshot `sonic-3.5-2026-05-04`)   |
| `cartesia_ink_v2`                 | Ink 2 (STT)             | `ink-2`                                         |
| `minimax_speech_v2_8_hd`          | Speech 2.8 HD           | `speech-2.8-hd`                                 |
| `minimax_speech_v2_8_turbo`       | Speech 2.8 Turbo        | `speech-2.8-turbo`                              |
| `minimax_voice_clone`             | MiniMax Voice Clone     | `fal-ai/minimax/voice-clone`                    |
| `minimax_music_v2_6`              | MiniMax Music 2.6       | `music-2.6`                                     |
| `openai_gpt_4o_mini_tts`          | GPT-4o mini TTS         | `gpt-4o-mini-tts`                               |
| `openai_gpt_4o_transcribe`        | GPT-4o Transcribe       | `gpt-4o-transcribe`                             |
| `openai_whisper_large_v3`         | Whisper large-v3        | `openai/whisper` / `whisper-1`                  |
| `google_gemini_v3_1_flash_tts`    | Gemini 3.1 Flash TTS    | `gemini-3.1-flash-tts-preview`                  |
| `google_chirp_v3`                 | Chirp 3 (TTS/STT)       | Cloud TTS `Chirp3-HD` voices                    |
| `google_lyria_v3`                 | Lyria 3                 | `lyria-3-clip-preview`                          |
| `google_lyria_v3_pro`             | Lyria 3 Pro             | `lyria-3-pro-preview`                           |
| `deepgram_nova_v3`                | Nova-3 (STT)            | `nova-3`                                        |
| `deepgram_flux`                   | Flux (STT, agents)      | `flux-general-en`                               |
| `deepgram_aura_v2`                | Aura-2 (TTS)            | `aura-2-<voice>-<lang>`                         |
| `assemblyai_universal_v3_5_pro`   | Universal-3.5 Pro (STT) | `universal-3-5-pro`                             |
| `stability_stable_audio_v3`       | Stable Audio 3.0        | `stable-audio-3`                                |
| `stability_stable_audio_v2_5`     | Stable Audio 2.5        | `stable-audio-2.5`                              |
| `xai_grok_voice_tts_v1`           | Grok Voice TTS 1.0      | `x-ai/grok-voice-tts-1.0`                       |
| `mistral_voxtral_mini_tts`        | Voxtral Mini TTS        | `mistralai/voxtral-mini-tts-2603`               |
| `mistral_voxtral_mini_transcribe` | Voxtral Mini Transcribe | `mistralai/voxtral-mini-transcribe-2602`        |
| `kokoro`                          | Kokoro 82M              | `fal-ai/kokoro` (OSS)                           |
| `narilabs_dia`                    | Dia 1.6B                | `fal-ai/dia-tts` (OSS, dialogue)                |
| `f5_tts`                          | F5-TTS                  | `fal-ai/f5-tts` (OSS, clone)                    |
| `canopy_orpheus`                  | Orpheus TTS             | `fal-ai/orpheus-tts` (OSS)                      |
| `resemble_chatterbox`             | Chatterbox              | `resemble-ai/chatterbox` (OSS)                  |
| `sesame_csm_1b`                   | CSM-1B                  | `fal-ai/csm-1b` (OSS, conversational)           |
| `microsoft_vibevoice`             | VibeVoice               | `microsoft/vibevoice` (OSS, long multi-speaker) |
| `alibaba_qwen3_tts`               | Qwen3-TTS               | `qwen/qwen3-tts` (OSS, clone)                   |
| `mmaudio_v2`                      | MMAudio v2              | `fal-ai/mmaudio-v2` (OSS, video-to-audio)       |
| `ace_step`                        | ACE-Step                | `fal-ai/ace-step` (OSS, music)                  |
| `sonauto_v2`                      | Sonauto v2              | `sonauto/v2/text-to-music`                      |

## 7. Mesh / 3D models

**Sign-off:** _unsigned_

| Our ID                         | Display name         | Reference (theirs)                                                                 |
| ------------------------------ | -------------------- | ---------------------------------------------------------------------------------- |
| `tencent_hunyuan3d_v2_1`       | Hunyuan3D 2.1        | `tencent/hunyuan3d-2.1`                                                            |
| `tencent_hunyuan3d_v3`         | Hunyuan3D 3.0        | `fal-ai/hunyuan3d-v3/image-to-3d`                                                  |
| `tencent_hunyuan3d_v3_1_pro`   | Hunyuan 3D 3.1 Pro   | `fal-ai/hunyuan-3d/v3.1/pro/...`                                                   |
| `tencent_hunyuan3d_v3_1_rapid` | Hunyuan 3D 3.1 Rapid | `.../v3.1/rapid/...`                                                               |
| `microsoft_trellis`            | TRELLIS              | `microsoft/trellis` (OSS)                                                          |
| `microsoft_trellis_v2`         | TRELLIS.2            | `fal-ai/trellis-2` (PBR)                                                           |
| `tripo_v2_5`                   | Tripo v2.5           | `tripo3d/tripo/v2.5/image-to-3d`                                                   |
| `tripo_h3_1`                   | Tripo H3.1           | `tripo3d/h3.1/...`                                                                 |
| `hyper3d_rodin_v2_5`           | Rodin V2.5           | `fal-ai/hyper3d/rodin/v2.5`                                                        |
| `meshy_v6`                     | Meshy 6              | `fal-ai/meshy/v6/...` — fullest 3D-utility suite (rig/animate/remesh/UV/retexture) |
| `triposr`                      | TripoSR              | `fal-ai/triposr` (OSS)                                                             |
| `meta_sam3d_objects`           | SAM 3D Objects       | `fal-ai/sam-3/3d-objects`                                                          |
| `anything_world_animate`       | Animate Anything     | Anything World API (experimental)                                                  |

## 8. World & splat models

**Sign-off:** _unsigned_

World Labs Marble is NOT on fal — access is their own World API
(`POST api.worldlabs.ai/marble/v1/worlds:generate`, launched 2026-01-21). RTFM is a research
demo, no API. ArtCraft already has `worldlabs_api_client` bindings to adapt.

| Our ID                       | Display name               | Reference (theirs)                     |
| ---------------------------- | -------------------------- | -------------------------------------- |
| `worldlabs_marble_v1`        | Marble 1.0 (+Draft)        | Marble app tiers (GA 2025-11-12)       |
| `worldlabs_marble_v1_1`      | Marble 1.1                 | `marble-1.1` (World API, default)      |
| `worldlabs_marble_v1_1_plus` | Marble 1.1 Plus            | `marble-1.1-plus` (larger world scale) |
| `tencent_hunyuan_world_v1`   | HunyuanWorld 1.0           | `fal-ai/hunyuan_world/image-to-world`  |
| `spaitial_echo_v1`           | SpAItial Echo-1            | Echo API — hosted, exports SPZ/PLY/SOG |
| `spaitial_echo_v2`           | SpAItial Echo-2            | Echo API + semantic world editing      |
| `dreamgaussian`              | DreamGaussian              | `mareksagan/dreamgaussian` (OSS)       |
| `lgm`                        | LGM (Large Gaussian Model) | OSS weights (hosted API TBD)           |

## 9. Language models

**Sign-off:** _unsigned_

Seeded from OpenRouter's live 2026-07 weekly-usage rankings (their page brands it "LLM
Rankings"). Rough usage order. The 2026 surprises: Xiaomi MiMo at #2, Tencent Hy3 at #4,
StepFun and Poolside in the top 15, NVIDIA Nemotron 3 as a top-15 lab.

| Our ID                          | Display name             | Reference (OpenRouter slug)               |
| ------------------------------- | ------------------------ | ----------------------------------------- |
| `deepseek_v4_flash`             | DeepSeek V4 Flash        | `deepseek/deepseek-v4-flash` (#1)         |
| `deepseek_v4_pro`               | DeepSeek V4 Pro          | `deepseek/deepseek-v4-pro`                |
| `deepseek_v3_2`                 | DeepSeek V3.2            | `deepseek/deepseek-v3.2`                  |
| `xiaomi_mimo_v2_5`              | MiMo-V2.5                | `xiaomi/mimo-v2.5` (#2)                   |
| `xiaomi_mimo_v2_5_pro`          | MiMo-V2.5-Pro            | `xiaomi/mimo-v2.5-pro`                    |
| `minimax_m3`                    | MiniMax M3               | `minimax/minimax-m3` (#3)                 |
| `tencent_hy3`                   | Tencent Hy3              | `tencent/hy3-preview` (#4)                |
| `zai_glm_v5_2`                  | GLM 5.2                  | `z-ai/glm-5.2` (#5)                       |
| `zai_glm_v5_1`                  | GLM 5.1                  | `z-ai/glm-5.1`                            |
| `zai_glm_v5`                    | GLM 5                    | `z-ai/glm-5`                              |
| `anthropic_claude_opus_v4_8`    | Claude Opus 4.8          | `anthropic/claude-opus-4.8` (#7)          |
| `anthropic_claude_sonnet_v5`    | Claude Sonnet 5          | `anthropic/claude-sonnet-5`               |
| `anthropic_claude_sonnet_v4_6`  | Claude Sonnet 4.6        | `anthropic/claude-sonnet-4.6`             |
| `anthropic_claude_haiku_v4_5`   | Claude Haiku 4.5         | `anthropic/claude-haiku-4.5`              |
| `anthropic_claude_fable_v5`     | Claude Fable 5           | `anthropic/claude-fable-5`                |
| `stepfun_step_v3_7_flash`       | Step 3.7 Flash           | `stepfun/step-3.7-flash` (#9)             |
| `openai_gpt_v5_5`               | GPT-5.5                  | `openai/gpt-5.5` (#12)                    |
| `openai_gpt_v5_5_pro`           | GPT-5.5 Pro              | `openai/gpt-5.5-pro`                      |
| `openai_gpt_v5_4`               | GPT-5.4 (+mini/nano)     | `openai/gpt-5.4`                          |
| `openai_gpt_v5_3_codex`         | GPT-5.3 Codex            | `openai/gpt-5.3-codex`                    |
| `openai_gpt_oss_120b`           | gpt-oss-120b             | `openai/gpt-oss-120b` (open weights)      |
| `openai_o4_mini`                | o4-mini (legacy tail)    | `openai/o4-mini`                          |
| `nvidia_nemotron_v3_ultra`      | Nemotron 3 Ultra 550B    | `nvidia/nemotron-3-ultra-550b-a55b` (#13) |
| `nvidia_nemotron_v3_super`      | Nemotron 3 Super 120B    | `nvidia/nemotron-3-super-120b-a12b`       |
| `google_gemini_v3_flash`        | Gemini 3 Flash           | `google/gemini-3-flash-preview` (#14)     |
| `google_gemini_v3_1_pro`        | Gemini 3.1 Pro           | `google/gemini-3.1-pro-preview`           |
| `google_gemini_v3_1_flash_lite` | Gemini 3.1 Flash Lite    | `google/gemini-3.1-flash-lite`            |
| `google_gemini_v3_5_flash`      | Gemini 3.5 Flash         | `google/gemini-3.5-flash`                 |
| `google_gemini_v2_5_flash`      | Gemini 2.5 Flash (+Lite) | `google/gemini-2.5-flash`                 |
| `google_gemma_v4_31b`           | Gemma 4 31B              | `google/gemma-4-31b-it` (open weights)    |
| `poolside_laguna_m1`            | Laguna M.1               | `poolside/laguna-m.1` (#15)               |
| `moonshot_kimi_k2_6`            | Kimi K2.6                | `moonshotai/kimi-k2.6`                    |
| `moonshot_kimi_k2_7_code`       | Kimi K2.7 Code           | `moonshotai/kimi-k2.7-code`               |
| `alibaba_qwen_v3_7_max`         | Qwen3.7 Max              | `qwen/qwen3.7-max`                        |
| `alibaba_qwen_v3_7_plus`        | Qwen3.7 Plus             | `qwen/qwen3.7-plus`                       |
| `alibaba_qwen_v3_coder_next`    | Qwen3 Coder Next         | `qwen/qwen3-coder-next` (open weights)    |
| `xai_grok_v4_3`                 | Grok 4.3                 | `x-ai/grok-4.3`                           |
| `xai_grok_v4_20`                | Grok 4.20                | `x-ai/grok-4.20`                          |
| `xai_grok_build_v0_1`           | Grok Build 0.1           | `x-ai/grok-build-0.1` (agentic/coding)    |
| `mistral_medium_v3_5`           | Mistral Medium 3.5       | `mistralai/mistral-medium-3-5`            |
| `mistral_large_v3`              | Mistral Large 3          | `mistralai/mistral-large-2512`            |
| `mistral_devstral_v2`           | Devstral 2               | `mistralai/devstral-2512`                 |
| `meta_llama_v4_maverick`        | Llama 4 Maverick         | `meta-llama/llama-4-maverick`             |
| `meta_llama_v4_scout`           | Llama 4 Scout            | `meta-llama/llama-4-scout`                |
| `meta_llama_v3_3_70b`           | Llama 3.3 70B            | `meta-llama/llama-3.3-70b-instruct`       |
| `amazon_nova_v2_lite`           | Nova 2 Lite              | `amazon/nova-2-lite-v1`                   |
| `amazon_nova_premier`           | Nova Premier             | `amazon/nova-premier-v1`                  |
| `cohere_command_a`              | Command A                | `cohere/command-a`                        |
| `microsoft_phi_v4`              | Phi 4                    | `microsoft/phi-4` (open weights)          |

## 10. Model modalities table

**Sign-off:** _unsigned_

Per your spec: model ID → comma-separated modalities, for all models. This IS the catalog's
per-model task declaration (each modality here = one granular endpoint + one JSON Schema).
Definitions in §11.

### Image models

| Model                            | Modalities                                             |
| -------------------------------- | ------------------------------------------------------ |
| `bfl_flux_v1_schnell`            | text_to_image                                          |
| `bfl_flux_v1_dev`                | text_to_image, image_to_image                          |
| `bfl_flux_v1_krea_dev`           | text_to_image                                          |
| `bfl_flux_pro_v1_1`              | text_to_image                                          |
| `bfl_flux_pro_v1_1_ultra`        | text_to_image                                          |
| `bfl_flux_v2_dev`                | text_to_image, image_edit                              |
| `bfl_flux_v2_pro`                | text_to_image, image_edit                              |
| `bfl_flux_v2_flex`               | text_to_image, image_edit                              |
| `bfl_flux_v2_max`                | text_to_image, image_edit                              |
| `bfl_flux_v2_klein_9b`           | text_to_image, image_edit                              |
| `bfl_flux_v2_klein_4b`           | text_to_image, image_edit                              |
| `openai_gpt_image_v1`            | text_to_image, image_edit                              |
| `openai_gpt_image_v1_mini`       | text_to_image                                          |
| `openai_gpt_image_v1_5`          | text_to_image, image_edit                              |
| `openai_gpt_image_v2`            | text_to_image, image_edit                              |
| `google_nano_banana`             | text_to_image, image_edit                              |
| `google_nano_banana_v2`          | text_to_image, image_edit                              |
| `google_nano_banana_v2_lite`     | text_to_image                                          |
| `google_nano_banana_pro`         | text_to_image, image_edit                              |
| `google_imagen_v4`               | text_to_image                                          |
| `bytedance_seedream_v3`          | text_to_image                                          |
| `bytedance_seedream_v4`          | text_to_image, image_edit                              |
| `bytedance_seedream_v4_5`        | text_to_image, image_edit                              |
| `bytedance_seedream_v5_lite`     | text_to_image                                          |
| `bytedance_dreamina_v3_1`        | text_to_image                                          |
| `bytedance_bagel`                | text_to_image, image_edit                              |
| `stability_sd_v3_5_large`        | text_to_image                                          |
| `stability_sd_v3_5_large_turbo`  | text_to_image                                          |
| `stability_sd_v3_5_medium`       | text_to_image                                          |
| `recraft_v3`                     | text_to_image, image_to_image, text_to_svg, upscale    |
| `recraft_v4`                     | text_to_image, text_to_svg                             |
| `recraft_v4_pro`                 | text_to_image, text_to_svg                             |
| `ideogram_v3`                    | text_to_image, image_edit, inpaint, background_replace |
| `ideogram_v4`                    | text_to_image, image_to_image                          |
| `ideogram_character`             | image_to_image                                         |
| `alibaba_qwen_image`             | text_to_image                                          |
| `alibaba_qwen_image_v2`          | text_to_image                                          |
| `alibaba_qwen_image_v2_pro`      | text_to_image                                          |
| `alibaba_z_image_turbo`          | text_to_image                                          |
| `xai_grok_imagine_image`         | text_to_image, image_edit                              |
| `xai_grok_imagine_image_quality` | text_to_image, image_edit                              |
| `hidream_i1_full`                | text_to_image, image_to_image                          |
| `luma_photon`                    | text_to_image                                          |
| `luma_photon_flash`              | text_to_image                                          |
| `nvidia_sana`                    | text_to_image                                          |
| `kwai_kolors`                    | text_to_image                                          |
| `bria_v3_2`                      | text_to_image                                          |
| `bria_fibo`                      | text_to_image, image_edit, relight                     |
| `krea_v2_turbo`                  | text_to_image                                          |
| `krea_v2_large`                  | text_to_image                                          |
| `midjourney_v7`                  | text_to_image                                          |
| `midjourney_v8`                  | text_to_image                                          |
| `bfl_flux_kontext_dev`           | image_edit                                             |
| `bfl_flux_kontext_pro`           | image_edit                                             |
| `bfl_flux_kontext_max`           | image_edit                                             |
| `bfl_flux_fill_pro`              | inpaint, outpaint                                      |
| `alibaba_qwen_image_edit`        | image_edit                                             |
| `alibaba_qwen_image_edit_plus`   | image_edit                                             |
| `omnigen_v2`                     | text_to_image, image_edit                              |
| `bria_eraser`                    | object_removal                                         |
| `bria_genfill`                   | inpaint, outpaint                                      |
| `bria_rmbg_v2`                   | background_removal                                     |
| `birefnet_v2`                    | background_removal                                     |
| `rembg`                          | background_removal                                     |
| `esrgan`                         | upscale                                                |
| `clarity_upscaler`               | upscale                                                |
| `topaz_image_upscale`            | upscale                                                |
| `aura_sr`                        | upscale                                                |
| `supir`                          | upscale, restoration                                   |
| `codeformer`                     | face_restore                                           |
| `gfpgan`                         | face_restore                                           |
| `ddcolor`                        | colorize                                               |
| `lama`                           | object_removal, inpaint                                |
| `ic_light_v2`                    | relight, background_replace                            |
| `idm_vton`                       | try_on                                                 |
| `fashn_tryon_v1_6`               | try_on                                                 |

### Video models

| Model                         | Modalities                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------ |
| `kling_video_v1_6_standard`   | text_to_video, image_to_video, reference_to_video, effects                           |
| `kling_video_v1_6_pro`        | text_to_video, image_to_video, reference_to_video, effects                           |
| `kling_video_v2_1_standard`   | image_to_video                                                                       |
| `kling_video_v2_1_pro`        | image_to_video                                                                       |
| `kling_video_v2_1_master`     | text_to_video, image_to_video                                                        |
| `kling_video_v2_5_turbo_pro`  | text_to_video, image_to_video                                                        |
| `kling_video_v2_6_pro`        | text_to_video, image_to_video, motion_control                                        |
| `kling_video_o1`              | first_last_to_video, reference_to_video, video_edit                                  |
| `kling_video_o3_standard`     | text_to_video, image_to_video, reference_to_video, video_edit                        |
| `kling_video_o3_pro`          | text_to_video, image_to_video, reference_to_video, video_edit                        |
| `kling_video_v3_standard`     | text_to_video, image_to_video, motion_control                                        |
| `kling_video_v3_pro`          | text_to_video, image_to_video, motion_control                                        |
| `kling_video_v3_turbo`        | text_to_video, image_to_video                                                        |
| `google_veo_v2`               | text_to_video, image_to_video                                                        |
| `google_veo_v3`               | text_to_video, image_to_video                                                        |
| `google_veo_v3_fast`          | text_to_video, image_to_video                                                        |
| `google_veo_v3_1`             | text_to_video, image_to_video, reference_to_video, first_last_to_video, extend_video |
| `google_veo_v3_1_fast`        | text_to_video, image_to_video, first_last_to_video, extend_video                     |
| `google_veo_v3_1_lite`        | text_to_video, image_to_video                                                        |
| `google_gemini_omni_flash`    | text_to_video, image_to_video, reference_to_video, video_edit                        |
| `openai_sora_v2`              | text_to_video, image_to_video, video_remix                                           |
| `openai_sora_v2_pro`          | text_to_video, image_to_video                                                        |
| `bytedance_seedance_v1_lite`  | text_to_video, image_to_video, reference_to_video                                    |
| `bytedance_seedance_v1_pro`   | text_to_video, image_to_video                                                        |
| `bytedance_seedance_v1_5_pro` | text_to_video, image_to_video                                                        |
| `bytedance_seedance_v2`       | text_to_video, image_to_video, reference_to_video                                    |
| `bytedance_seedance_v2_fast`  | text_to_video, image_to_video, reference_to_video                                    |
| `bytedance_seedance_v2_mini`  | text_to_video, image_to_video                                                        |
| `alibaba_wan_v2_2`            | text_to_video, image_to_video, video_to_video                                        |
| `alibaba_wan_v2_5`            | text_to_video, image_to_video                                                        |
| `alibaba_wan_v2_6`            | text_to_video, image_to_video, reference_to_video                                    |
| `alibaba_wan_v2_7`            | text_to_video, image_to_video, reference_to_video, video_edit                        |
| `alibaba_wan_vace_14b`        | video_to_video, inpaint, outpaint, pose_control                                      |
| `alibaba_happy_horse`         | text_to_video, image_to_video                                                        |
| `alibaba_happy_horse_v1_1`    | text_to_video, image_to_video                                                        |
| `minimax_hailuo_v1`           | text_to_video, image_to_video, reference_to_video                                    |
| `minimax_hailuo_v2`           | text_to_video, image_to_video                                                        |
| `minimax_hailuo_v2_3`         | text_to_video, image_to_video                                                        |
| `minimax_hailuo_v2_3_fast`    | text_to_video, image_to_video                                                        |
| `vidu_v2`                     | image_to_video, reference_to_video, first_last_to_video                              |
| `vidu_q1`                     | text_to_video, image_to_video, reference_to_video, first_last_to_video               |
| `vidu_q2`                     | text_to_video, image_to_video, reference_to_video, extend_video                      |
| `vidu_q3`                     | text_to_video, image_to_video, reference_to_video                                    |
| `vidu_q3_turbo`               | text_to_video, image_to_video                                                        |
| `pika_v2_2`                   | text_to_video, image_to_video, first_last_to_video, scene_composition                |
| `pixverse_v5`                 | text_to_video, image_to_video, effects, transition                                   |
| `pixverse_v6`                 | text_to_video, image_to_video, transition, extend_video                              |
| `pixverse_c1`                 | text_to_video, image_to_video, reference_to_video, transition                        |
| `luma_ray_v2`                 | text_to_video, image_to_video, video_to_video                                        |
| `luma_ray_v2_flash`           | text_to_video, image_to_video, video_to_video                                        |
| `luma_ray_v3_2`               | text_to_video, image_to_video, video_to_video                                        |
| `runway_gen_v4_turbo`         | image_to_video                                                                       |
| `runway_gen_v4_aleph`         | video_edit                                                                           |
| `lightricks_ltx_v2`           | text_to_video, image_to_video, audio_to_video, extend_video                          |
| `lightricks_ltx_v2_3`         | text_to_video, image_to_video, audio_to_video, extend_video, video_to_video          |
| `tencent_hunyuan_video`       | text_to_video, image_to_video, video_to_video                                        |
| `tencent_hunyuan_video_v1_5`  | text_to_video, image_to_video                                                        |
| `xai_grok_imagine_video`      | text_to_video, image_to_video, reference_to_video, video_edit, extend_video          |
| `xai_grok_imagine_video_v1_5` | image_to_video                                                                       |
| `moonvalley_marey_v1_5`       | text_to_video, image_to_video, motion_transfer                                       |
| `genmo_mochi_v1`              | text_to_video                                                                        |
| `zhipu_cogvideox_5b`          | text_to_video, image_to_video, video_to_video                                        |
| `sandai_magi`                 | text_to_video, image_to_video, extend_video                                          |
| `meituan_longcat_video`       | text_to_video, image_to_video                                                        |
| `sber_kandinsky_v5`           | text_to_video, image_to_video                                                        |
| `characterai_ovi`             | text_to_video, image_to_video                                                        |
| `decart_lucy_5b`              | image_to_video                                                                       |
| `decart_lucy_edit`            | video_edit                                                                           |
| `topaz_video_upscale`         | video_upscale, frame_interpolation                                                   |
| `bytedance_seedvr_v2`         | video_upscale                                                                        |
| `flashvsr`                    | video_upscale                                                                        |
| `rife`                        | frame_interpolation                                                                  |
| `bytedance_latentsync`        | lipsync                                                                              |
| `sync_lipsync_v2`             | lipsync                                                                              |
| `sync_lipsync_v2_pro`         | lipsync                                                                              |
| `kling_lipsync`               | lipsync                                                                              |
| `tencent_musetalk`            | lipsync                                                                              |
| `bytedance_omnihuman`         | avatar                                                                               |
| `bytedance_omnihuman_v1_5`    | avatar                                                                               |
| `kling_avatar_v2_pro`         | avatar                                                                               |
| `heygen_avatar_v4`            | avatar                                                                               |
| `kwai_liveportrait`           | image_animation, motion_transfer                                                     |
| `meigen_multitalk`            | avatar                                                                               |
| `meigen_infinitetalk`         | avatar                                                                               |
| `alibaba_wan_animate`         | motion_transfer                                                                      |

### Audio models

| Model                             | Modalities                                             |
| --------------------------------- | ------------------------------------------------------ |
| `elevenlabs_v3`                   | text_to_speech, dialogue_to_speech                     |
| `elevenlabs_multilingual_v2`      | text_to_speech                                         |
| `elevenlabs_flash_v2_5`           | text_to_speech                                         |
| `elevenlabs_scribe_v2`            | speech_to_text                                         |
| `elevenlabs_scribe_v2_realtime`   | speech_to_text (lifecycle: session)                    |
| `elevenlabs_music_v2`             | text_to_music, inpaint_music                           |
| `elevenlabs_sfx_v2`               | text_to_sfx                                            |
| `cartesia_sonic_v3_5`             | text_to_speech                                         |
| `cartesia_ink_v2`                 | speech_to_text                                         |
| `minimax_speech_v2_8_hd`          | text_to_speech                                         |
| `minimax_speech_v2_8_turbo`       | text_to_speech                                         |
| `minimax_voice_clone`             | voice_clone                                            |
| `minimax_music_v2_6`              | text_to_music, cover_music                             |
| `openai_gpt_4o_mini_tts`          | text_to_speech                                         |
| `openai_gpt_4o_transcribe`        | speech_to_text                                         |
| `openai_whisper_large_v3`         | speech_to_text                                         |
| `google_gemini_v3_1_flash_tts`    | text_to_speech                                         |
| `google_chirp_v3`                 | text_to_speech, speech_to_text                         |
| `google_lyria_v3`                 | text_to_music                                          |
| `google_lyria_v3_pro`             | text_to_music                                          |
| `deepgram_nova_v3`                | speech_to_text                                         |
| `deepgram_flux`                   | speech_to_text (lifecycle: session)                    |
| `deepgram_aura_v2`                | text_to_speech                                         |
| `assemblyai_universal_v3_5_pro`   | speech_to_text                                         |
| `stability_stable_audio_v3`       | text_to_music, text_to_sfx, inpaint_music, cover_music |
| `stability_stable_audio_v2_5`     | text_to_music, text_to_sfx, inpaint_music, cover_music |
| `xai_grok_voice_tts_v1`           | text_to_speech                                         |
| `mistral_voxtral_mini_tts`        | text_to_speech                                         |
| `mistral_voxtral_mini_transcribe` | speech_to_text                                         |
| `kokoro`                          | text_to_speech                                         |
| `narilabs_dia`                    | text_to_speech, dialogue_to_speech, voice_clone        |
| `f5_tts`                          | text_to_speech, voice_clone                            |
| `canopy_orpheus`                  | text_to_speech                                         |
| `resemble_chatterbox`             | text_to_speech, voice_clone                            |
| `sesame_csm_1b`                   | text_to_speech                                         |
| `microsoft_vibevoice`             | text_to_speech, dialogue_to_speech                     |
| `alibaba_qwen3_tts`               | text_to_speech, voice_clone                            |
| `mmaudio_v2`                      | video_to_audio, text_to_sfx                            |
| `ace_step`                        | text_to_music                                          |
| `sonauto_v2`                      | text_to_music, extend_music, inpaint_music             |

### Mesh models

| Model                          | Modalities                                                                        |
| ------------------------------ | --------------------------------------------------------------------------------- |
| `tencent_hunyuan3d_v2_1`       | image_to_mesh, retexture                                                          |
| `tencent_hunyuan3d_v3`         | image_to_mesh                                                                     |
| `tencent_hunyuan3d_v3_1_pro`   | text_to_mesh, image_to_mesh, retopology                                           |
| `tencent_hunyuan3d_v3_1_rapid` | text_to_mesh, image_to_mesh                                                       |
| `microsoft_trellis`            | image_to_mesh                                                                     |
| `microsoft_trellis_v2`         | image_to_mesh, pbr_generation                                                     |
| `tripo_v2_5`                   | image_to_mesh                                                                     |
| `tripo_h3_1`                   | text_to_mesh, image_to_mesh, rigging, animation, retexture, retopology            |
| `hyper3d_rodin_v2_5`           | text_to_mesh, image_to_mesh, retexture, pbr_generation                            |
| `meshy_v6`                     | text_to_mesh, image_to_mesh, retexture, rigging, animation, retopology, uv_unwrap |
| `triposr`                      | image_to_mesh                                                                     |
| `meta_sam3d_objects`           | image_to_mesh                                                                     |
| `anything_world_animate`       | rigging, animation (only non-humanoid rig/animate API; experimental)              |

### Splat / world models (SPLIT from mesh 2026-07-05 — different media type)

Marble outputs Gaussian splats (`.spz`) + meshes; splats are their own media type, never
"mesh". Additional splat generators + the animation/rigging/texturing/PBR utility sweep are
under research (`../reports/other-providers.md` when it lands); known OSS splat models added
below.

| Model                        | Modalities                                                       |
| ---------------------------- | ---------------------------------------------------------------- |
| `worldlabs_marble_v1`        | text_to_world, image_to_world, video_to_world, panorama_to_world |
| `worldlabs_marble_v1_1`      | text_to_world, image_to_world, video_to_world, panorama_to_world |
| `worldlabs_marble_v1_1_plus` | text_to_world, image_to_world, video_to_world, panorama_to_world |
| `tencent_hunyuan_world_v1`   | image_to_world                                                   |
| `spaitial_echo_v1`           | text_to_world, image_to_world                                    |
| `spaitial_echo_v2`           | text_to_world, image_to_world, world_edit                        |
| `dreamgaussian`              | image_to_splat (OSS)                                             |
| `lgm`                        | image_to_splat, text_to_splat (OSS, Large Gaussian Model)        |

### Language models

All models in §9: `chat` (tool calling / vision / reasoning are catalog capability flags, not
modalities).

## 11. Modality definitions

**Sign-off:** _unsigned_

| Modality              | Meaning                                                          |
| --------------------- | ---------------------------------------------------------------- |
| `text_to_image`       | Prompt → image(s)                                                |
| `image_to_image`      | Image + prompt → restyled/varied image                           |
| `image_edit`          | Instruction-driven edit of input image(s), incl. multi-reference |
| `inpaint`             | Masked-region regeneration                                       |
| `outpaint`            | Extend the canvas beyond original borders                        |
| `object_removal`      | Erase an object, fill plausibly                                  |
| `background_removal`  | Cut out subject; alpha output                                    |
| `background_replace`  | Swap the background, keep subject                                |
| `upscale`             | Increase resolution (may hallucinate detail)                     |
| `restoration`         | Fix degraded/old images                                          |
| `face_restore`        | Repair faces specifically                                        |
| `colorize`            | B&W → color                                                      |
| `relight`             | Change lighting/environment of a subject                         |
| `try_on`              | Garment transfer onto a person                                   |
| `text_to_svg`         | Prompt → vector graphics                                         |
| `text_to_video`       | Prompt → video                                                   |
| `image_to_video`      | Start image (+prompt) → video                                    |
| `first_last_to_video` | Start + end keyframes → video between them                       |
| `reference_to_video`  | Subject/style reference images → video featuring them            |
| `video_to_video`      | Video + prompt → transformed video (restyle/controllable)        |
| `video_edit`          | Instruction-driven edit of a video                               |
| `video_remix`         | Regenerate a video with variations (Sora remix)                  |
| `extend_video`        | Continue a video in time                                         |
| `motion_control`      | Explicit camera/subject motion directives                        |
| `motion_transfer`     | Apply motion from a source video to a target subject             |
| `image_animation`     | Animate a still (expressions/pose)                               |
| `effects`             | Template effects applied to input media                          |
| `transition`          | Generate a transition between two inputs                         |
| `scene_composition`   | Multi-image ingredients composed into one video scene            |
| `pose_control`        | Pose/depth-guided video generation                               |
| `lipsync`             | Retarget mouth to audio/text                                     |
| `avatar`              | Audio/text → talking-person video                                |
| `video_upscale`       | Video resolution/quality increase                                |
| `frame_interpolation` | Increase FPS / smooth motion                                     |
| `audio_to_video`      | Audio input drives video generation                              |
| `video_to_audio`      | Generate soundtrack/foley from video                             |
| `text_to_speech`      | Text → voice audio                                               |
| `dialogue_to_speech`  | Multi-speaker script → conversation audio                        |
| `speech_to_text`      | Audio → transcript                                               |
| `voice_clone`         | Reference voice sample → reusable voice                          |
| `voice_conversion`    | Audio in one voice → same performance in another voice           |
| `text_to_music`       | Prompt → music                                                   |
| `extend_music`        | Continue a music clip in time (either end)                       |
| `inpaint_music`       | Regenerate/replace a section inside a music clip                 |
| `cover_music`         | Style-transfer an existing song (reference-audio cover)          |
| `remaster`            | Quality-uplift an existing audio recording                       |
| `stem_split`          | Song → separated instrument/vocal stems                          |
| `audio_isolation`     | Extract speech/vocals from noisy audio                           |
| `forced_alignment`    | Audio + transcript → word/phoneme timestamps                     |
| `text_to_sfx`         | Prompt → sound effect                                            |
| `text_to_mesh`        | Prompt → 3D asset                                                |
| `image_to_mesh`       | Image(s) → 3D asset                                              |
| `retexture`           | New textures on an existing mesh                                 |
| `retopology`          | Clean topology of a mesh                                         |
| `rigging`             | Skeleton/animation rig for a mesh                                |
| `text_to_splat`       | Prompt → Gaussian splat asset (.spz/.ply)                        |
| `image_to_splat`      | Image → Gaussian splat asset                                     |
| `text_to_world`       | Prompt → explorable 3D world (3DGS/mesh)                         |
| `image_to_world`      | Image → explorable world                                         |
| `video_to_world`      | Video → explorable world                                         |
| `panorama_to_world`   | 360° panorama → world                                            |
| `pbr_generation`      | PBR material maps for a mesh                                     |
| `animation`           | Animate a rigged/riggable 3D asset                               |
| `uv_unwrap`           | UV coordinates for a mesh                                        |
| `world_edit`          | Semantic edit of a generated world (SpAItial Echo-2)             |
| `chat`                | LLM chat completion                                              |

## 12. Model aliases

**Sign-off:** _unsigned_

A code-level `alias → canonical` map in the catalog (no DB table — catalog-as-code). Policy:
**we launch with zero aliases** and add them only when needed. When we do, the kinds:

| Alias kind   | Example                                                  | Purpose                      |
| ------------ | -------------------------------------------------------- | ---------------------------- |
| `short`      | `seedance_v2` → `bytedance_seedance_v2`                  | Ergonomics                   |
| `rename`     | old slug → new slug after a model rename                 | Never break integrations     |
| `competitor` | `fal-ai/flux/schnell` → `bfl_flux_v1_schnell`            | Migration from Fal/Replicate |
| `rolling`    | `kling_video_latest_pro` → current best (probably never) | Dangerous; avoid             |

Rules: aliases resolve at request admission; jobs always record the canonical slug; an alias
can never collide with a canonical slug (compile-time test); each model's aliases appear in
its agent docs.

## 13. Snapshot pinning (provider sub-versions) — DECIDED 2026-07-05

**Sign-off:** _unsigned_

Decision: **`_snap_{version}`** marks a pinned snapshot; **`_latest`** is also supported as an
explicit rolling alias. (Analysis retained in `../reports/read/model-versioning.md`; enum
representation of snapshots: `../reports/model-enums-expanded-considerations.md`.)

| Form                                       | Meaning                                                                                           |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `anthropic_claude_sonnet_v5`               | Default: the provider's current/rolling build                                                     |
| `anthropic_claude_sonnet_v5_latest`        | Explicit rolling alias — same resolution as bare, for callers who want the intent visible in code |
| `anthropic_claude_sonnet_v5_snap_20260630` | Pinned to the dated snapshot                                                                      |
| `bfl_flux_v1_dev_snap_a1b2c3d`             | Pinned to a non-date upstream version (hash/tag) — the marker makes non-date pins expressible     |

Rules:

- `snap` and `latest` are **reserved tokens** (slug lint: `snap` must be followed by exactly
  one payload token and terminate the ID; `latest` must terminate the ID; neither may appear
  in family/variant names).
- Snapshots are first-class catalog entries (own pricing, own deprecation status) — never
  string-parsed at runtime. `_latest` resolves through the alias map to the canonical bare
  slug; jobs always record the canonical slug that actually ran.
- Upstream retires a snapshot → catalog marks it `retired`, requests fail
  `provider_deprecated` with the migration target in the error details.

**v1 scope note:** we may or may not ship any pinned snapshots (or the `_latest` alias) in
version one — most launch providers work fine on rolling builds. The point of deciding now is
to keep the design space wide open: the lint rules, alias machinery, and catalog shape all
accommodate snapshots from day one, so adding the first one is a catalog entry, not a naming
crisis.
