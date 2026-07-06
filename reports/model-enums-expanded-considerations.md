# Model enums vs snapshots, `_latest`, and fine-tunes

Report, 2026-07-05. The question: `openai_gpt_v5_1_snap_20251113` is a valid request value —
should it be a Rust enum variant (`OpenaiGptV5p1Snap20251113`)? Follows
`read/enums-alternatives.md` (xtask codegen, Recommendation v2) and
`read/model-versioning.md` (`_snap_`/`_latest`, decided in `../design/models.md` §13).

## Answer up front

**No — that's crazy, and there's a saner structure.** The enum holds **one variant per
catalog model** (the canonical rolling slug — the parent). Snapshots, `_latest`, and future
fine-tunes are a thin **parse layer on top**: the request field deserializes into a
`ModelRef` (parent enum + pin), and whether a given snapshot *exists* is catalog **data**
validated at admission — not a variant. Customers still type
`openai_gpt_v5_1_snap_20251113`; the enum space just doesn't mirror it.

## Why variant-per-snapshot is wrong

1. **Unbounded multiplicative growth.** OpenAI alone mints dated snapshots monthly, per
   model. ~215 base models × N snapshots, forever — the enum stops describing *what our
   product offers* and starts mirroring upstream release calendars. Every upstream snapshot
   mint becomes a codegen diff + release.
2. **It destroys the one thing enums buy us.** The whole argument for enums (vs a string
   newtype) is exhaustive matching in adapters/routing/pricing. Snapshot variants behave
   *identically* to their parent everywhere except one lookup table, so every match would
   grow wildcard arms (`_ => parent_behavior()`) — and once matches are wildcarded, the
   compiler can no longer tell you what you forgot. Worst of both worlds.
3. **Fine-tunes break it outright.** A customer fine-tune ID is minted at **runtime** (a DB
   row), so it cannot be a compile-time variant, period. If the `model` request field must
   someday carry fine-tunes, the type behind that field must be parse-based. Build that seam
   now, cheaply, instead of migrating an enum-typed field later.
4. Churn tax on everything generated: `ALL`, `slug()`, round-trip tests, TS enums, docs —
   all rebloat per snapshot for zero behavioral content.

## The structure: `ModelRef` = parent enum + pin

```rust
// crates/core/enums (hand-written once; parser tests generated)
pub struct ModelRef {
  pub model: Model,      // the generated umbrella enum — one variant per models.md row
  pub pin:   Pin,
}

pub enum Pin {
  Rolling,               // bare slug: "openai_gpt_v5_1"
  Latest,                // "..._latest" — explicit rolling alias
  Snapshot(SnapToken),   // "..._snap_20251113" / "..._snap_a1b2c3d"
}
```

Parsing `openai_gpt_v5_1_snap_20251113`:

1. **Shape:** strip reserved trailing tokens (`latest`, `snap` + one payload token — the §13
   lint rules, applied in reverse); the remainder must parse via the generated
   `Model::from_str` (alias-aware).
2. **Existence:** catalog lookup — does `openai_gpt_v5_1` declare snapshot `20251113`, and
   with what status? Unknown snapshot → `invalid_request` naming the known ones; `retired` →
   `provider_deprecated`. Snapshots are first-class *catalog entries* (own upstream string,
   own status, own pricing when it differs) — exactly §13's rule, "never string-parsed at
   runtime" meaning: never *trusted* from parsing alone; the catalog is the authority.
3. `Pin::Latest` normalizes to `Rolling` at admission (alias semantics); jobs record the
   canonical slug that actually ran — unchanged from today.

What each layer matches on:

| Layer                         | Matches on                | Snapshot awareness                          |
| ----------------------------- | ------------------------- | -------------------------------------------- |
| Adapter dispatch, routing     | `Model` (exhaustive)      | None — receives the resolved upstream string  |
| Pricing, modalities, schemas  | `Model` (exhaustive)      | Override table per snapshot only when it differs |
| Catalog                       | data: `snapshots: &[Snap]` per model (token, upstream id, status) | The single source of truth |
| Wire / docs / job rows        | full canonical string     | Customers see exactly the IDs models.md promises |

**Rule of thumb this encodes: enum variants for things code *branches* on; data for things
only the catalog *maps*.** Model identity changes behavior (which adapter, which schema) —
enum. Snapshot changes only a lookup value (upstream string) — data.

## "Should we map to the parent model?" — yes, exactly

The snapshot ID *is* the parent plus a parameter. `ModelRef { model: OpenaiGptV5p1,
pin: Snapshot("20251113") }` — parent variant carries identity, pin carries the frozen-build
selection. Same answer for `_latest` (parent + explicit-rolling flag, erased at admission).

## Fine-tunes later — the seam is already cut

When BYOM/fine-tunes land (`../design/future-roadmap.md` §4), the request type grows one arm
without touching anything above:

```rust
pub enum ModelRef {
  Catalog { model: Model, pin: Pin },
  FineTune(FineTuneId),    // "ft_..." — a DB-resident resource, validated per-account
}
```

Fine-tune IDs live in their own prefix namespace (`ft_`-style), so parsing is unambiguous
against catalog slugs, and the catalog/enum machinery never learns about them — they resolve
to (base model for schema/pricing purposes, customer deployment for routing) via the
`custom_models` table.

## Options considered

| Design                                   | Verdict                                                        |
| ---------------------------------------- | ---------------------------------------------------------------- |
| Variant per snapshot                     | Rejected: unbounded churn, wildcarded matches, fine-tune-impossible (above) |
| String newtype everywhere (`ModelId(String)`) | Rejected already in `read/enums-alternatives.md` — loses exhaustive adapter dispatch entirely |
| **Parent enum + `Pin` + catalog data**   | **RECOMMENDED** — keeps exhaustiveness where it pays, makes the multiplicative axes data, gives fine-tunes a landing slot |
| Two enums (models + snapshot-models)     | Mirror-enum tax (the ArtCraft disease); no benefit over Pin       |

## Codegen impact (xtask)

- Generated: unchanged `Model`/per-media enums from models.md tables; **plus** a per-model
  snapshot data table (from catalog declarations) and round-trip parser tests for every
  declared snapshot and `_latest` form.
- Hand-written once: the `ModelRef` parser (~50 lines) and `Pin`. It changes only if the §13
  grammar changes.
- The §13 lint rules (`snap`/`latest` reserved, position constraints) become parser unit
  tests plus a models.md table lint in the same xtask.
