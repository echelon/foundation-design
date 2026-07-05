# Model ID Catalog (hypothetical seed list)

**Sign-off:** _unsigned_
<!-- bt: replace "_unsigned_" with initials + date (e.g. "bt 2026-07-05") when approved.
     Assistants: reset to _unsigned_ on any substantive edit. See ../CLAUDE.md. -->

**Status:** living document (2026-07-04). The naming rules are decided; this list is
illustrative — drawn from Fal, Replicate, and OpenRouter catalogs plus the direct providers —
and will be pruned to the actual launch set. Reference IDs shown are the competitor/upstream
spellings for comparison.

Rules recap (see architecture.md):

- Charset `[a-z0-9_]` only — every ID double-click-copyable on every platform.
- Shape: `{owner}_{family}[_v{major}[_{minor}]][_{variant}...]`. Dots in versions become
  underscores after a literal `v` (`2.5` → `v2_5`); trailing `.0` drops (`3.0` → `v3`).
- Versionless market names stay versionless (`google_nano_banana_pro`).
- Media type is part of model identity when the upstream splits it that way
  (`xai_grok_imagine_image` vs `xai_grok_imagine_video`); *modality* (t2v vs i2v) never is.
- OSS utility models with no meaningful owner may omit the owner prefix (`birefnet_v2`).
- Aliases (declared in code) absorb short forms and competitor spellings.

---

## Video

| Our ID                            | Display name              | Reference ID (theirs)                        |
| --------------------------------- | ------------------------- | -------------------------------------------- |
| `kling_video_v1_6_standard`       | Kling 1.6 Standard        | `fal-ai/kling-video/v1.6/standard/...`       |
| `kling_video_v1_6_pro`            | Kling 1.6 Pro             | `fal-ai/kling-video/v1.6/pro/...`            |
| `kling_video_v2_1_standard`       | Kling 2.1 Standard        | `fal-ai/kling-video/v2.1/standard/...`       |
| `kling_video_v2_1_pro`            | Kling 2.1 Pro             | `fal-ai/kling-video/v2.1/pro/...`            |
| `kling_video_v2_1_master`         | Kling 2.1 Master          | `fal-ai/kling-video/v2.1/master/...`         |
| `kling_video_v2_5_turbo_pro`      | Kling 2.5 Turbo Pro       | `fal-ai/kling-video/v2.5-turbo/pro/...`      |
| `kling_video_v2_6_pro`            | Kling 2.6 Pro             | `kling-2.6/...` (Kie)                        |
| `kling_video_v3_standard`         | Kling 3.0 Standard        | `fal-ai/kling-video/v3/standard/...`         |
| `kling_video_v3_pro`              | Kling 3.0 Pro             | `fal-ai/kling-video/v3/pro/...`              |
| `bytedance_seedance_v1_lite`      | Seedance 1.0 Lite         | `bytedance/seedance-1-lite` (Replicate)      |
| `bytedance_seedance_v1_pro`       | Seedance 1.0 Pro          | `bytedance/seedance-1-pro`                   |
| `bytedance_seedance_v1_5_pro`     | Seedance 1.5 Pro          | `fal-ai/bytedance/seedance/v1.5/pro`         |
| `bytedance_seedance_v2`           | Seedance 2.0              | `dreamina-seedance-2-0-260128` (BytePlus)    |
| `bytedance_seedance_v2_fast`      | Seedance 2.0 Fast         | `dreamina-seedance-2-0-fast-260128`          |
| `bytedance_seedance_v2_mini`      | Seedance 2.0 Mini         | `doubao-seedance-2-0-mini-260615` (Volc)     |
| `google_veo_v2`                   | Veo 2                     | `veo-2.0-generate-001`                       |
| `google_veo_v3`                   | Veo 3                     | `veo-3.0-generate-001`                       |
| `google_veo_v3_fast`              | Veo 3 Fast                | `veo-3.0-fast-generate-001`                  |
| `google_veo_v3_1`                 | Veo 3.1                   | `veo-3.1-generate-preview`                   |
| `google_veo_v3_1_fast`            | Veo 3.1 Fast              | `veo-3.1-fast-generate-preview`              |
| `google_veo_v3_1_lite`            | Veo 3.1 Lite              | `veo-3.1-lite-generate-preview`              |
| `openai_sora_v2`                  | Sora 2                    | `sora-2`                                     |
| `openai_sora_v2_pro`              | Sora 2 Pro                | `sora-2-pro`                                 |
| `xai_grok_imagine_video`          | Grok Imagine Video        | `grok-imagine-video`                         |
| `xai_grok_imagine_video_v1_5`     | Grok Imagine Video 1.5    | `grok-imagine-video-1.5`                     |
| `vidu_q1`                         | Vidu Q1                   | `viduq1`                                     |
| `vidu_q1_classic`                 | Vidu Q1 Classic           | `viduq1-classic`                             |
| `vidu_q2`                         | Vidu Q2                   | `viduq2`                                     |
| `vidu_q2_pro`                     | Vidu Q2 Pro               | `viduq2-pro`                                 |
| `vidu_q3_turbo`                   | Vidu Q3 Turbo             | `viduq3-turbo`                               |
| `vidu_q3_pro`                     | Vidu Q3 Pro               | `viduq3-pro`                                 |
| `alibaba_wan_v2_2`                | Wan 2.2                   | `wan-video/wan-2.2-t2v-fast` (Replicate)     |
| `alibaba_wan_v2_5`                | Wan 2.5                   | `fal-ai/wan-25-preview/...`                  |
| `alibaba_wan_v2_6`                | Wan 2.6                   | Comet `wan-2.6`                              |
| `minimax_hailuo_v2_3`             | Hailuo 2.3                | `fal-ai/minimax/hailuo-2.3/...`              |
| `minimax_hailuo_v2_3_fast`        | Hailuo 2.3 Fast           | `fal-ai/minimax/hailuo-2.3-fast/...`         |
| `runway_gen_v4`                   | Runway Gen-4              | `gen4_turbo` (Runway API)                    |
| `runway_gen_v4_turbo`             | Runway Gen-4 Turbo        | `gen4_turbo`                                 |
| `luma_ray_v2`                     | Luma Ray 2                | `luma/ray-2` (Replicate)                     |
| `pixverse_v5`                     | PixVerse V5               | `fal-ai/pixverse/v5/...`                     |
| `lightricks_ltx_v2`               | LTX-2                     | `fal-ai/ltx-2/...`                           |
| `tencent_hunyuan_video`           | Hunyuan Video             | `tencent/hunyuan-video`                      |
| `genmo_mochi_v1`                  | Mochi 1                   | `genmoai/mochi-1`                            |
| `midjourney_video_v1`             | Midjourney Video V1       | (no official API — via aggregators)          |

## Image

| Our ID                            | Display name              | Reference ID (theirs)                        |
| --------------------------------- | ------------------------- | -------------------------------------------- |
| `bfl_flux_v1_schnell`             | FLUX.1 [schnell]          | `fal-ai/flux/schnell`                        |
| `bfl_flux_v1_dev`                 | FLUX.1 [dev]              | `fal-ai/flux/dev`                            |
| `bfl_flux_pro_v1_1`               | FLUX 1.1 Pro              | `black-forest-labs/flux-1.1-pro`             |
| `bfl_flux_pro_v1_1_ultra`         | FLUX 1.1 Pro Ultra        | `black-forest-labs/flux-1.1-pro-ultra`       |
| `bfl_flux_kontext_pro`            | FLUX Kontext Pro          | `black-forest-labs/flux-kontext-pro`         |
| `bfl_flux_kontext_max`            | FLUX Kontext Max          | `black-forest-labs/flux-kontext-max`         |
| `bfl_flux_v2`                     | FLUX 2                    | `fal-ai/flux-2`                              |
| `openai_gpt_image_v1`             | GPT Image 1               | `gpt-image-1`                                |
| `openai_gpt_image_v1_mini`        | GPT Image 1 Mini          | `gpt-image-1-mini`                           |
| `openai_gpt_image_v1_5`           | GPT Image 1.5             | `gpt-image-1.5`                              |
| `openai_gpt_image_v2`             | GPT Image 2               | `gpt-image-2`                                |
| `google_nano_banana`              | Nano Banana               | `gemini-2.5-flash-image`                     |
| `google_nano_banana_v2`           | Nano Banana 2             | `gemini-3.1-flash-image`                     |
| `google_nano_banana_v2_lite`      | Nano Banana 2 Lite        | `gemini-3.1-flash-lite-image`                |
| `google_nano_banana_pro`          | Nano Banana Pro           | `gemini-3-pro-image`                         |
| `google_imagen_v4`                | Imagen 4                  | `imagen-4.0-generate-001` (dies 2026-08-17)  |
| `bytedance_seedream_v3`           | Seedream 3.0              | `seedream-3-0-t2i-250415`                    |
| `bytedance_seedream_v4`           | Seedream 4.0              | `bytedance/seedream-4` (Replicate)           |
| `bytedance_seedream_v4_5`         | Seedream 4.5              | `fal-ai/bytedance/seedream/v4.5`             |
| `bytedance_seedream_v5_lite`      | Seedream 5.0 Lite         | (AtlasCloud "Seedream v5.0")                 |
| `xai_grok_imagine_image`          | Grok Imagine Image        | `grok-imagine-image`                         |
| `xai_grok_imagine_image_quality`  | Grok Imagine Image Q      | `grok-imagine-image-quality`                 |
| `midjourney_v7`                   | Midjourney V7             | (no official API — via aggregators)          |
| `midjourney_v7_niji`              | Midjourney V7 Niji        |                                              |
| `midjourney_v8`                   | Midjourney V8             |                                              |
| `stability_sd_v3_5_large`         | SD 3.5 Large              | `stability-ai/stable-diffusion-3.5-large`    |
| `stability_sd_v3_5_large_turbo`   | SD 3.5 Large Turbo        | `stability-ai/stable-diffusion-3.5-large-turbo` |
| `recraft_v3`                      | Recraft V3                | `recraft-ai/recraft-v3`                      |
| `ideogram_v3`                     | Ideogram 3.0              | `ideogram-ai/ideogram-v3-quality`            |
| `ideogram_v3_turbo`               | Ideogram 3.0 Turbo        | `ideogram-ai/ideogram-v3-turbo`              |
| `alibaba_qwen_image`              | Qwen Image                | `qwen/qwen-image`                            |
| `alibaba_qwen_image_edit`         | Qwen Image Edit           | `qwen/qwen-image-edit`                       |
| `hidream_i1_full`                 | HiDream-I1 Full           | `hidream-ai/hidream-i1-full`                 |
| `luma_photon`                     | Luma Photon               | `luma/photon`                                |
| `luma_photon_flash`               | Luma Photon Flash         | `luma/photon-flash`                          |
| `bria_rmbg_v2`                    | Bria Background Removal   | `bria/rmbg` (utility)                        |
| `birefnet_v2`                     | BiRefNet v2               | `fal-ai/birefnet/v2` (OSS utility, no owner) |
| `esrgan`                          | Real-ESRGAN Upscaler      | `fal-ai/esrgan` (OSS utility)                |

## Audio

| Our ID                            | Display name              | Reference ID (theirs)                        |
| --------------------------------- | ------------------------- | -------------------------------------------- |
| `elevenlabs_multilingual_v2`      | ElevenLabs Multilingual v2| `eleven_multilingual_v2`                     |
| `elevenlabs_turbo_v2_5`           | ElevenLabs Turbo 2.5      | `eleven_turbo_v2_5`                          |
| `elevenlabs_v3`                   | ElevenLabs v3             | `eleven_v3`                                  |
| `minimax_speech_v2_5`             | MiniMax Speech 2.5        | `minimax/speech-02-hd` family                |
| `stability_stable_audio_v2`       | Stable Audio 2            | `stability-ai/stable-audio-2`                |
| `mmaudio_v2`                      | MMAudio v2                | `fal-ai/mmaudio-v2` (OSS utility)            |
| `openai_whisper_large_v3`         | Whisper Large v3 (STT)    | `openai/whisper-large-v3`                    |
| `cartesia_sonic_v2`               | Cartesia Sonic 2          | `sonic-2`                                    |

## Mesh

| Our ID                            | Display name              | Reference ID (theirs)                        |
| --------------------------------- | ------------------------- | -------------------------------------------- |
| `tencent_hunyuan3d_v2_1`          | Hunyuan3D 2.1             | `tencent/hunyuan3d-2`                        |
| `tencent_hunyuan3d_v3`            | Hunyuan3D 3.0             | `fal-ai/hunyuan3d/v3`                        |
| `microsoft_trellis`               | TRELLIS                   | `microsoft/trellis`                          |
| `tripo_v2_5`                      | Tripo v2.5                | `tripo3d/tripo-v2.5`                         |
| `hyper3d_rodin_v1_5`              | Rodin 1.5                 | `fal-ai/hyper3d/rodin`                       |

## Splat / World

| Our ID                            | Display name              | Reference ID (theirs)                        |
| --------------------------------- | ------------------------- | -------------------------------------------- |
| `worldlabs_marble`                | World Labs Marble         | (World Labs API — ArtCraft has a client)     |

## LLM (chat)

Naming stress-tests: `o3` (versionless family), `k2.6` (fused family+version), dated
Anthropic-style minors — the rules absorb all of them.

| Our ID                            | Display name              | Reference ID (OpenRouter-style)              |
| --------------------------------- | ------------------------- | -------------------------------------------- |
| `anthropic_claude_opus_v4_8`      | Claude Opus 4.8           | `anthropic/claude-opus-4.8`                  |
| `anthropic_claude_sonnet_v5`      | Claude Sonnet 5           | `anthropic/claude-sonnet-5`                  |
| `anthropic_claude_haiku_v4_5`     | Claude Haiku 4.5          | `anthropic/claude-haiku-4.5`                 |
| `openai_gpt_v5`                   | GPT-5                     | `openai/gpt-5`                               |
| `openai_gpt_v5_1`                 | GPT-5.1                   | `openai/gpt-5.1`                             |
| `openai_gpt_v5_mini`              | GPT-5 Mini                | `openai/gpt-5-mini`                          |
| `openai_o3`                       | o3                        | `openai/o3`                                  |
| `google_gemini_v2_5_pro`          | Gemini 2.5 Pro            | `google/gemini-2.5-pro`                      |
| `google_gemini_v2_5_flash`        | Gemini 2.5 Flash          | `google/gemini-2.5-flash`                    |
| `google_gemini_v3_pro`            | Gemini 3 Pro              | `google/gemini-3-pro`                        |
| `google_gemini_v3_1_flash`        | Gemini 3.1 Flash          | `google/gemini-3.1-flash`                    |
| `xai_grok_v4`                     | Grok 4                    | `x-ai/grok-4`                                |
| `xai_grok_v4_1`                   | Grok 4.1                  | `x-ai/grok-4.1`                              |
| `meta_llama_v4_maverick`          | Llama 4 Maverick          | `meta-llama/llama-4-maverick`                |
| `meta_llama_v4_scout`             | Llama 4 Scout             | `meta-llama/llama-4-scout`                   |
| `meta_llama_v3_3_70b`             | Llama 3.3 70B             | `meta-llama/llama-3.3-70b-instruct`          |
| `deepseek_v3_2`                   | DeepSeek V3.2             | `deepseek/deepseek-v3.2`                     |
| `deepseek_r1`                     | DeepSeek R1               | `deepseek/deepseek-r1`                       |
| `moonshot_kimi_k2`                | Kimi K2                   | `moonshotai/kimi-k2`                         |
| `moonshot_kimi_k2_6`              | Kimi K2.6                 | `moonshotai/kimi-k2.6` (AtlasCloud)          |
| `alibaba_qwen_v3_max`             | Qwen3 Max                 | `qwen/qwen3-max`                             |
| `mistral_large_v2_1`              | Mistral Large 2.1         | `mistralai/mistral-large-2411`               |
| `mistral_medium_v3`               | Mistral Medium 3          | `mistralai/mistral-medium-3`                 |

Caveat: the LLM rows (and some video rows) reflect my knowledge cutoff plus what today's
research surfaced; exact current model lineups should be re-verified per provider before the
catalog crate is seeded.
