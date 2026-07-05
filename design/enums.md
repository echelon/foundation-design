# First-Class Enums

**Sign-off:** _unsigned_
<!-- bt: replace "_unsigned_" with initials + date (e.g. "bt 2026-07-05") when approved.
     Assistants: reset to _unsigned_ on any substantive edit. See ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Siblings: `architecture.md`, `api.md`, `schema.md`,
`models.md`, `agents.md`.

---

## Where they live

| Layer      | Location                                  | Rule                                                        |
| ---------- | ----------------------------------------- | ----------------------------------------------------------- |
| Rust       | `crates/core/enums`                       | Pure identifiers: enum + explicit `#[serde(rename)]` per variant + exhaustive serialization tests (ArtCraft style). No behavior beyond conversions |
| Rust       | `crates/core/catalog`                     | Attaches metadata to the model/provider enums: display names, modalities, schemas, prices, routes. Registry is keyed BY the enums; compile-time completeness checks |
| Database   | `text` columns + CHECK constraints        | Never native Postgres enums (see `../reports/postgresql-column-types.md`); CHECK lists are generated from the Rust enums by a migration-lint test |
| TypeScript | `frontend/libs/api-client/src/enums/`     | **Generated, never hand-written** — codegen from the utoipa OpenAPI spec (enums come out as string-literal unions). A CI diff check keeps it fresh |
| Agent docs | `generated/llms/`                         | Rendered from the same catalog registry (see `agents.md`)   |

Serialized form of every variant = the canonical slug (`[a-z0-9_]`), identical across wire,
DB, TS, and docs.

## Style (carried from ArtCraft)

```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq, Hash, Serialize, Deserialize, ToSchema, EnumIter)]
pub enum VideoModel {
  #[serde(rename = "kling_video_v3_pro")]
  KlingVideoV3Pro,
  // ...
}
```

- Explicit `#[serde(rename = "...")]` on every variant — never rely on `rename_all` alone.
- Mechanical tests per enum: serde round-trip, `^[a-z0-9_]+$` slug lint, variant count,
  exhaustive `assert_serialization` list.
- Model enums additionally get: `media_type()`, `provider()` (via catalog), and a
  `FromStr` that consults the alias map.

## The model enums

Variants below mirror `models.md` (the seed list — prune together). Slugs shown are the serde
renames; Rust variant names are the CamelCase of the slug.

### `Provider`

```
bytedance      google         openai         anthropic      kling
xai            vidu           minimax        runway         luma
stability      tencent        worldlabs      moonshot       deepseek
meta           mistral        alibaba        elevenlabs     cartesia
bfl            midjourney     recraft        ideogram       hidream
pixverse       lightricks     genmo          microsoft      tripo
hyper3d        bria
```

Launch subset: `bytedance google openai anthropic kling xai vidu`. The enum ships with all
known upstreams; only launch providers get client crates + catalog routes.

### `ImageModel`

```
bfl_flux_v1_schnell            bfl_flux_v1_dev                bfl_flux_pro_v1_1
bfl_flux_pro_v1_1_ultra        bfl_flux_kontext_pro           bfl_flux_kontext_max
bfl_flux_v2                    openai_gpt_image_v1            openai_gpt_image_v1_mini
openai_gpt_image_v1_5          openai_gpt_image_v2            google_nano_banana
google_nano_banana_v2          google_nano_banana_v2_lite     google_nano_banana_pro
google_imagen_v4               bytedance_seedream_v3          bytedance_seedream_v4
bytedance_seedream_v4_5        bytedance_seedream_v5_lite     xai_grok_imagine_image
xai_grok_imagine_image_quality midjourney_v7                  midjourney_v7_niji
midjourney_v8                  stability_sd_v3_5_large        stability_sd_v3_5_large_turbo
recraft_v3                     ideogram_v3                    ideogram_v3_turbo
alibaba_qwen_image             alibaba_qwen_image_edit        hidream_i1_full
luma_photon                    luma_photon_flash              bria_rmbg_v2
birefnet_v2                    esrgan
```

### `VideoModel`

```
kling_video_v1_6_standard      kling_video_v1_6_pro           kling_video_v2_1_standard
kling_video_v2_1_pro           kling_video_v2_1_master        kling_video_v2_5_turbo_pro
kling_video_v2_6_pro           kling_video_v3_standard        kling_video_v3_pro
bytedance_seedance_v1_lite     bytedance_seedance_v1_pro      bytedance_seedance_v1_5_pro
bytedance_seedance_v2          bytedance_seedance_v2_fast     bytedance_seedance_v2_mini
google_veo_v2                  google_veo_v3                  google_veo_v3_fast
google_veo_v3_1                google_veo_v3_1_fast           google_veo_v3_1_lite
openai_sora_v2                 openai_sora_v2_pro             xai_grok_imagine_video
xai_grok_imagine_video_v1_5    vidu_q1                        vidu_q1_classic
vidu_q2                        vidu_q2_pro                    vidu_q3_turbo
vidu_q3_pro                    alibaba_wan_v2_2               alibaba_wan_v2_5
alibaba_wan_v2_6               minimax_hailuo_v2_3            minimax_hailuo_v2_3_fast
runway_gen_v4                  runway_gen_v4_turbo            luma_ray_v2
pixverse_v5                    lightricks_ltx_v2              tencent_hunyuan_video
genmo_mochi_v1                 midjourney_video_v1
```

### `LanguageModel`

```
anthropic_claude_opus_v4_8     anthropic_claude_sonnet_v5     anthropic_claude_haiku_v4_5
openai_gpt_v5                  openai_gpt_v5_1                openai_gpt_v5_mini
openai_o3                      google_gemini_v2_5_pro         google_gemini_v2_5_flash
google_gemini_v3_pro           google_gemini_v3_1_flash       xai_grok_v4
xai_grok_v4_1                  meta_llama_v4_maverick         meta_llama_v4_scout
meta_llama_v3_3_70b            deepseek_v3_2                  deepseek_r1
moonshot_kimi_k2               moonshot_kimi_k2_6             alibaba_qwen_v3_max
mistral_large_v2_1             mistral_medium_v3
```

### `AudioModel`  (you didn't list it — needed)

```
elevenlabs_multilingual_v2     elevenlabs_turbo_v2_5          elevenlabs_v3
minimax_speech_v2_5            stability_stable_audio_v2      mmaudio_v2
openai_whisper_large_v3        cartesia_sonic_v2
```

### `MeshModel`  (you didn't list it — needed)

```
tencent_hunyuan3d_v2_1         tencent_hunyuan3d_v3           microsoft_trellis
tripo_v2_5                     hyper3d_rodin_v1_5
```

### `SplatModel`

```
worldlabs_marble
```

### `Model` (umbrella)

A tagged union over the per-media enums so code can hold "any model":

```rust
pub enum Model {
  Image(ImageModel),
  Video(VideoModel),
  Language(LanguageModel),
  Audio(AudioModel),
  Mesh(MeshModel),
  Splat(SplatModel),
}
```

Serializes as the bare slug (slugs are globally unique across media types — enforced by a
compile-time test).

## Supporting enums (the ones you'd forgotten — all referenced by schema.md/api.md)

| Enum              | Variants                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------- |
| `MediaType`       | `image  video  audio  mesh  splat  text`                                                        |
| `Modality`        | `text_to_image  image_to_image  text_to_video  image_to_video  reference_to_video  video_to_video  keyframe_to_video  text_to_audio  audio_to_audio  speech_to_text  text_to_mesh  image_to_mesh  text_to_splat  image_to_splat  chat` |
| `JobStatus`       | `queued  dispatched  running  succeeded  failed  canceled  expired`                             |
| `JobEventType`    | `created  charged  dispatched  provider_accepted  poll_tick  provider_completed  output_ingested  webhook_sent  refunded  failed  canceled` |
| `ApiShape`        | `granular  unified  omni  chat`                                                                 |
| `RequestSource`   | `api  dashboard`                                                                                |
| `LedgerEntryType` | `topup  job_charge  job_refund  adjustment  promo  auto_refill`                                 |
| `PriceUnit`       | `per_call  per_image  per_video_second  per_audio_second  per_megapixel  per_second_compute  per_million_tokens_in  per_million_tokens_out  per_mesh  per_splat` |
| `QueueTaskType`   | `dispatch_job  poll_provider  ingest_outputs  deliver_webhook  settle_refund  auto_refill`      |
| `QueueTaskStatus` | `pending  running  done  dead`                                                                  |
| `WebhookEvent`    | `job_completed  job_failed` (wire form `job.completed` / `job.failed`)                          |
| `AccountStatus`   | `active  suspended  closed`                                                                     |
| `MemberRole`      | `admin  member`                                                                                 |
| `OutputKind`      | `image  video  audio  mesh  splat  thumbnail  text`                                             |
| `StorageClass`    | `hot  cold`                                                                                     |
| `FailureCode`     | closed enum + `other` escape: `provider_error  provider_timeout  provider_content_policy  invalid_input  upstream_deprecated  internal  other` — grows deliberately |
