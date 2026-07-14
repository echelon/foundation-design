# Naming Conventions

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Captures bt's conventions as practiced in ArtCraft
(`~/dev/storyteller/artcraft`, `~/dev/storyteller/artcraft-dashboard`) plus the decisions
already made in this design tree. Where Foundation deliberately diverges from ArtCraft, the
divergence is called out. Sibling: `coding_conventions.md` (style/structure; this doc is
names only).

---

## 1. The canonical slug

**Sign-off:** _unsigned_

One identifier grammar rules everything serialized: **`^[a-z0-9_]+$`** — lowercase
snake_case, no hyphens, no uppercase, no dots. The same slug is the value on the wire, in
the DB column, in the TS union, and in the docs (`enums.md`).

| Applies to                 | Example                                    |
| -------------------------- | ------------------------------------------ |
| Enum serialized values     | `queued`, `image_to_video`, `auto_refill`  |
| Model IDs                  | `kling_video_v3_pro`                       |
| JSON field names           | `charged_micros`, `queue_stage`            |
| Query params               | `?after=`, `?media_type=`                  |
| Ledger/queue/event types   | `job_charge`, `dispatch_job`               |

The one exception: **webhook event names on the wire use dots** (`job.completed`,
`job.failed`) — industry convention (Stripe-style); the internal enum variants stay
snake_case (`job_completed`).

## 2. Rust

**Sign-off:** _unsigned_

| Thing                | Convention                                                       |
| -------------------- | ----------------------------------------------------------------- |
| Types/traits/variants| `PascalCase` — variant name = PascalCase of its slug (`KlingVideoV3Pro`) |
| Functions/vars/mods  | `snake_case`; module file name mirrors the item it holds (`task_status.rs` → `TaskStatus`) |
| Constants            | `SCREAMING_SNAKE_CASE`, declared at top of file after imports     |
| Library crates       | `snake_case` (ArtCraft: `grok_api_client`, `mysql_queries`)       |
| Binary/CLI crates    | `kebab-case` (ArtCraft: `db-backfill`; here: `api-server`, `worker`, `admin-cli`) |
| ID newtypes          | `{Entity}Id` (`JobId`, `AccountId`) — DIVERGES from ArtCraft's `{Entity}Token`; this project says "id" everywhere (`crates/core/ids`), so the types do too |
| Money newtypes       | Unit-bearing names: `Micros`, `NanoUsd` (`crates/core/money`)     |
| Constructors         | `new`, `new_from_str`                                             |
| Accessors/conversions| `as_str`, `to_str`, `from_str`, `all_variants`                    |
| Error enums          | Four-bucket taxonomy names: `{X}Error` (top), `{X}ClientError`, `{X}SpecificApiError`, `{X}GenericApiError`, plus a `classify_{x}_http_error` free fn (see `coding_conventions.md` §4) |
| HTTP handlers        | `{verb}_{noun}_handler.rs` under `endpoints/{domain}/` (ArtCraft: `get_session_credits_handler.rs`) |

Serde: **explicit `#[serde(rename = "...")]` on every variant** (DECIDED — `enums.md`).
This deliberately diverges from ArtCraft's `#[serde(rename_all = "snake_case")]`: the
explicit form makes the wire string reviewable at the variant and immune to a rename_all
refactor silently changing the API.

## 3. Model & catalog naming

**Sign-off:** _unsigned_

Authoritative grammar lives in `models.md` (§1 naming rules, §13 versioning). Recap:

- Model slug: `{owner}_{name}_{version}` — `bytedance_seedance_v1_pro`. Owner prefix is
  catalog metadata, not the dispatch provider.
- Version tokens: `v3`, dots become underscores (`v2_5`), qualifiers trail (`v3_pro`).
- Reserved suffixes: `_latest` (explicit rolling alias) and `_snap_{token}` (frozen build) —
  never usable as ordinary name parts.
- Slugs are globally unique across media types (compile-time test).
- `Provider` enum = who we actually call; lowercase single words (`bytedance`, `fal`).

## 4. Database schema

**Sign-off:** _unsigned_

| Thing            | Convention                                                              |
| ---------------- | ------------------------------------------------------------------------ |
| Table names      | plural `snake_case`: `jobs`, `ledger_entries`, `auto_refill_rules`       |
| Columns          | `snake_case`                                                              |
| Column order     | `sequence_id` first, `id` second (`ids.md` §4), payload columns, timestamps last |
| FK columns       | **include the referenced table's singular name**: `invited_by_user_id`, `created_by_user_id` — never bare roles (`invited_by`). `{table}_id` forms (`account_id`, `job_id`) already comply |
| Timestamps       | `{event}_at`, `timestamptz`: `created_at`, `canceled_at`, `last_triggered_at` |
| Money            | `{name}_micros` (i64 micro-USD); catalog unit rates `{name}_nanos` (nano-USD) |
| Booleans         | `is_{adj}` (`is_default`) or bare enable-words (`enabled`)                |
| Enum-ish columns | `text` + CHECK, values = canonical slugs; CHECK lists generated from Rust |
| Counts/limits    | descriptive suffixes: `daily_max_micros`, `consecutive_failures`          |

## 5. ID prefixes

**Sign-off:** _unsigned_

Registry and format rules: `ids.md` §5. Naming rules for new prefixes:

- Short, lowercase, letters only, **no underscores inside the prefix** — the underscore is
  the prefix/payload separator: `job_`, `acct_`, `txn_`, `enduser_`, `emailver_`, `jevt_`.
- Prefer terse abbreviations over full words when the table name is long (`sevt_` for
  stripe_events), full words when short and unambiguous (`job_`, `limit_`).
- Uniqueness enforced at compile time in `crates/core/ids`.
- Secrets are deliberately long and service-declaring: `foundation_api_`,
  `foundation_whsec_` (the `foundation` half swaps with the product name).

## 6. API surface

**Sign-off:** _unsigned_

| Thing           | Convention                                                             |
| --------------- | ----------------------------------------------------------------------- |
| Paths           | `/v1/{plural_resource}`: `/v1/jobs`, `/v1/chat/completions` (OpenAI-compat paths kept verbatim) |
| JSON fields     | `snake_case`, mirroring the Rust serde output exactly — TS DTOs do NOT camelize (ArtCraft practice: `banked_credits`, `user_token` straight through) |
| Headers         | `Title-Case` with product prefix: `Foundation-Version`, `Foundation-Signature` |
| Webhook events  | dot-form on the wire: `job.completed`, `job.failed`                     |
| Query params    | `snake_case`; cursor pagination param is `after`                        |
| Error codes     | snake_case slugs (`insufficient_balance`, `spend_limit_exceeded`) — registry in `api.md` §2 |

## 7. TypeScript / frontend

**Sign-off:** _unsigned_

| Thing               | Convention                                                          |
| ------------------- | -------------------------------------------------------------------- |
| Components/pages    | `PascalCase` files and named function exports: `MediaCard.tsx`       |
| Generated/vendored UI primitives | `kebab-case` (shadcn style): `components/ui/dropdown-menu.tsx` — generated names stay as generated |
| Hooks               | camelCase `use{Xxx}.ts`: `usePageTitle.ts`                           |
| Props types         | local `interface {Component}Props`                                   |
| Interfaces vs types | `interface` preferred; `type` reserved for unions and small aliases  |
| Enums               | **never TS `enum`** — string-literal unions, generated from the OpenAPI spec (`enums.md`) |
| DTO fields          | `snake_case` (mirror the wire; see §6)                               |
| Exports             | named exports only; no default exports                               |
| Path alias          | `@/` → `src/`                                                        |
| Nx app names        | lowercase single words: `www`, `docs`, `console`                     |

## 8. Files & repo layout

**Sign-off:** _unsigned_

- **One item per file** in Rust: each enum, error bucket, token, and handler gets its own
  file named after the item; `mod.rs` files are pure re-export lists.
- Design docs and reports: `kebab-case.md` (`future-roadmap.md`, `end-users.md`) — this doc
  and `coding_conventions.md` are grandfathered exceptions by explicit request.
- Generated artifacts live under clearly-generated paths: `crates/core/enums/src/generated/`,
  `generated/llms/`, `frontend/libs/api-client/` (generated), `catalog.json`.

## 9. Documentation & prose

**Sign-off:** _unsigned_

- Markdown tables: pad cells so pipes align; keep cells terse so padding stays maintainable
  (`../CLAUDE.md`).
- Dated, attributed inline notes in code and docs: `// NB(bt,2026-07-14): ...`,
  `// TODO(bt,2026-07-14): ...` (ArtCraft convention).
- Decision markers in design docs: `(DECIDED yyyy-mm-dd)`, `(REVISED yyyy-mm-dd)`, `OPEN`.
- No "credits" vocabulary anywhere in product copy (`payments.md`).
