# First-Class Enums

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Siblings: `architecture.md`, `api.md`, `schema.md`,
`models.md`, `agents.md`. Practice review: `../reports/enums-alternatives.md`.

---

## Where they live

**Sign-off:** _unsigned_

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

**Sign-off:** _unsigned_

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
- **Model enums are generated, not hand-written (DECIDED 2026-07-05):** `cargo xtask codegen`
  parses the canonical `models.md` tables and emits plain, checked-in Rust (enum + serde
  renames + `slug()`/`display_name()`/`modalities()` + the tests above) under
  `crates/core/enums/src/generated/`, with a CI diff gate. No macros, no `build.rs` — see
  `../reports/enums-alternatives.md` "Recommendation v2" for the rationale and samples.
  Supporting enums (statuses etc.) stay hand-written — they're small and carry logic.

## The model enums

**Sign-off:** _unsigned_

Variants below mirror `models.md` (the seed list — prune together). Slugs shown are the serde
renames; Rust variant names are the CamelCase of the slug.

### `Provider` — dispatch targets, not owners

`Provider` = **who we actually call**: an API we hold credentials for and wrote a client
crate against. The `owner` prefix in a model slug (`bfl_`, `moonvalley_`, `pixverse_`, …) is
catalog metadata, NOT this enum — most owners are reached through an aggregator or a bigger
host.

```
bytedance      google         openai         anthropic      kling
xai            vidu           worldlabs      elevenlabs     cartesia
minimax        deepgram       assemblyai     stability      runway
luma           tripo          meshy          hyper3d        tencent
fal            replicate
```

- Launch subset: `bytedance google openai anthropic kling xai vidu` — plus `fal` as the
  aggregator route for long-tail models we haven't integrated directly.
- `fal` / `replicate` as Providers is deliberate: the catalog can route a model through an
  aggregator first and flip it to a direct integration later without changing the model's
  slug — routing is a catalog field, identity is not.

### Per-media model enums — canonical lists live in `models.md`

`ImageModel`, `VideoModel`, `AudioModel`, `MeshModel`, `SplatModel`, `LanguageModel` variant
lists are deliberately NOT duplicated here: **`models.md` §2–§9 is canonical**, and the enum
code is generated from / CI-checked against those tables (same staleness-gate pattern as the
agent docs). One review surface, zero drift. The per-model modality declarations
(`models.md` §10) likewise drive each model's granular endpoints.

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

**Sign-off:** _unsigned_

| Enum              | Variants                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------- |
| `MediaType`       | `image  video  audio  mesh  splat  text`                                                        |
| `Modality`        | `text_to_image  image_to_image  text_to_video  image_to_video  reference_to_video  video_to_video  keyframe_to_video  text_to_audio  audio_to_audio  speech_to_text  text_to_mesh  image_to_mesh  text_to_splat  image_to_splat  chat` |
| `JobStatus`       | `queued  running  succeeded  failed  canceled  expired` — authoritative in `jobs.md` §2 (internal = external, decided) |
| `QueueStage`      | `gateway  provider` — naming OPEN (`jobs.md` §2)                                                |
| `CanceledBy`      | `user  system  provider` (`jobs.md` §4)                                                         |
| `JobEventType`    | `created  charged  dispatch_attempted  provider_accepted  poll_tick  provider_completed  output_ingested  webhook_sent  refunded  failed  canceled  expired` — authoritative in `jobs.md` §4 |
| `ApiShape`        | `granular  unified  any  chat`                                                                  |
| `RequestSource`   | `api  dashboard  mcp`                                                                           |
| `LedgerEntryType` | `topup  job_charge  job_refund  adjustment  promo  auto_refill`                                 |
| `PriceUnit`       | `per_call  per_image  per_video_second  per_audio_second  per_megapixel  per_second_compute  per_million_tokens_in  per_million_tokens_out  per_mesh  per_splat` |
| `QueueTaskType`   | `dispatch_job  poll_provider  ingest_outputs  deliver_webhook  settle_refund  auto_refill`      |
| `QueueTaskStatus` | `pending  running  done  dead`                                                                  |
| `WebhookEvent`    | `job_completed  job_failed` (wire form `job.completed` / `job.failed`)                          |
| `AccountStatus`   | `active  suspended  closed`                                                                     |
| `MemberRole`      | `admin  member`                                                                                 |
| `OutputKind`      | `image  video  audio  mesh  splat  thumbnail  text`                                             |
| `StorageClass`    | `hot  cold`                                                                                     |
| `FailureCode`     | authoritative user-facing list lives in `api.md` §2 (bt-editable); this enum mirrors it exactly |
