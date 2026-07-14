# First-Class Enums

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Siblings: `architecture.md`, `api.md`, `schema.md`,
`models.md`, `agents.md`. Practice review: `../reports/enums-alternatives.md`.

---

## Where they live

**Sign-off:** _unsigned_

| Layer      | Location                              | Rule                                                                                                                                                                |
| ---------- | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rust       | `crates/core/enums`                   | Pure identifiers: enum + explicit `#[serde(rename)]` per variant + exhaustive serialization tests (ArtCraft style). No behavior beyond conversions                  |
| Rust       | `crates/core/catalog`                 | Attaches metadata to the model/provider enums: display names, modalities, schemas, prices, routes. Registry is keyed BY the enums; compile-time completeness checks |
| Database   | `text` columns + CHECK constraints    | Never native Postgres enums (see `../reports/postgresql-column-types.md`); CHECK lists are generated from the Rust enums by a migration-lint test                   |
| TypeScript | `frontend/libs/api-client/src/enums/` | **Generated, never hand-written** — codegen from the utoipa OpenAPI spec (enums come out as string-literal unions). A CI diff check keeps it fresh                  |
| Agent docs | `generated/llms/`                     | Rendered from the same catalog registry (see `agents.md`)                                                                                                           |

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
xai            vidu           bfl            worldlabs      elevenlabs
cartesia       minimax        deepgram       assemblyai     stability
runway         luma           tripo          meshy          hyper3d
tencent
```

- Launch subset (= `providers.md` §1's committed crates, FIXED 2026-07-14 — previously
  omitted `bfl`/`worldlabs`): `openai google bytedance kling bfl xai anthropic vidu
  worldlabs`.
- **No aggregator variants (FIXED 2026-07-14):** `fal` / `replicate` removed — providers.md
  and `future-roadmap.md` §7 rule: we never route through aggregators, we compete with
  them. A variant here means "we hold credentials and wrote a client crate," and no such
  crate will exist. (Model slugs never encoded routing anyway, so a future direct
  integration of a long-tail model is purely additive.)

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

Audited against the downstream docs 2026-07-14 (QueueStage renamed, LedgerEntryType +2,
QueueTaskType +2, OutputKind −thumbnail, Modality deferred to models.md, five enums added).
The **Authority** column names the doc that owns the value set; this table mirrors, never
leads.

| Enum                    | Values                                                                                                                                                           | Authority                       |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| `MediaType`             | `image, video, audio, mesh, splat, text`                                                                                                                         | `models.md` §2–§9               |
| `Modality`              | 24 values — not mirrored here (too many to keep in sync)                                                                                                         | `models.md` §11                 |
| `JobStatus`             | `queued, running, succeeded, failed, canceled, expired` (internal = external, decided)                                                                           | `jobs.md` §2                    |
| `QueueStage`            | `at_our_gateway, at_provider` — naming OPEN                                                                                                                      | `jobs.md` §2                    |
| `CanceledBy`            | `user, system, provider`                                                                                                                                         | `jobs.md` §4                    |
| `JobEventType`          | `created, charged, dispatch_attempted, provider_accepted, poll_tick, provider_completed, output_ingested, webhook_sent, refunded, failed, canceled, expired`     | `jobs.md` §4                    |
| `ApiShape`              | `granular, unified, any, chat`                                                                                                                                   | `api.md` §3                     |
| `RequestSource`         | `api_key, dashboard, mcp` (`api` renamed `api_key`, 2026-07-14)                                                                                                  | `jobs.md` §4                    |
| `LedgerEntryType`       | `topup, job_charge, job_refund, adjustment, promo, auto_refill, chargeback, chargeback_reversal`                                                                 | `payments.md` §2                |
| `PriceUnit`             | `per_call, per_image, per_video_second, per_audio_second, per_megapixel, per_second_compute, per_million_tokens_in, per_million_tokens_out, per_mesh, per_splat` | catalog (`crates/core/catalog`) |
| `QueueTaskType`         | `dispatch_job, poll_provider, ingest_outputs, deliver_webhook, settle_refund, auto_refill, rollup_usage, process_stripe_event`                                   | `schema.md` §3                  |
| `QueueTaskStatus`       | `pending, running, done, dead`                                                                                                                                   | `schema.md` §3                  |
| `WebhookEvent`          | `job_completed, job_failed` (wire form `job.completed`, `job.failed`)                                                                                            | `api.md` §10                    |
| `AccountStatus`         | `active, suspended, closed`                                                                                                                                      | `accounts.md` §2                |
| `MemberRole`            | `admin, member`                                                                                                                                                  | `accounts.md` §2                |
| `OutputKind`            | `image, video, audio, mesh, splat, text` (`thumbnail` REMOVED 2026-07-14 — thumbnails are derived files, not outputs)                                            | `files.md` §2                   |
| `StorageClass`          | `hot, cold`                                                                                                                                                      | `files.md` §2                   |
| `FilePurpose`           | `input, output, avatar` (+ `thumbnail` when files.md §4 lands)                                                                                                   | `files.md` §2                   |
| `FileOrigin`            | `upload, mirrored_url, generated`                                                                                                                                | `files.md` §2                   |
| `PaymentMethodStatus`   | `active, detached`                                                                                                                                               | `payments.md` §2                |
| `SpendLimitWindow`      | `weekly, monthly`                                                                                                                                                | `payments.md` §2                |
| `ProviderCallDirection` | `request, response, poll_request, poll_response, callback_in`                                                                                                    | `logging.md` §3                 |
| `FailureCode`           | 14 values, not mirrored here (bt-editable) — hand-written enum with `category()`, `refunds_in_full()`, `is_customer_actionable()`                                | `api.md` §2                     |
| `FailureCategory`       | `moderation, bad_input, provider, lifecycle, internal`                                                                                                           | `api.md` §2                     |
| `AuditTargetType`       | singular entity slugs (`api_key, member, invitation, …`) — one per schema.md §7 entity, generated from the prefix registry                                       | `schema.md` §5                  |
