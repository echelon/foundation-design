# Coding Conventions

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Style and structure rules for the Rust and
TypeScript codebases, mined from ArtCraft practice plus decisions in this tree. Sibling:
`naming_conventions.md` (names only; this doc is everything else). Test conventions:
`testing.md`.

---

## 1. Universal style

**Sign-off:** bt on 2026-07-14

- **Two-space indent, never four** — all languages, enforced by formatters. Favors laptops
  and side-by-side diffs/panes.
- **Small scope.** Shallow nesting; guard clauses and **early returns** instead of arrow-code and deep nesting.
  Handle the error/absent case and get out; the happy path reads straight down at low indent.
- Small functions, small files: **one item per file** in Rust (each enum, error bucket,
  handler in its own file; `mod.rs` = re-exports only).
- File layout, top to bottom ("importance ordering"): imports → constants → structs/enums
  (Request, then Response, then Error for API types) → impls (constructors → public methods →
  private helpers below their callers).
- Comments: literate code first — too much documentation is bad. `///` doc comments go on
  types, enum variants, and struct fields (every error variant says when it fires); function
  bodies stay mostly comment-free. Inline notes are dated and attributed:
  `// NB(bt,2026-07-14): ...`.
- Formatting is always tool-enforced (rustfmt / Prettier defaults), never hand-argued in
  review.

## 2. Rust: formatting & lints

**Sign-off:** _unsigned_

`rustfmt.toml` (carried from ArtCraft, deliberately re-adopted for greenfield):

```toml
tab_spaces = 2
newline_style = "Unix"
match_block_trailing_comma = true
use_try_shorthand = true
use_small_heuristics = "Off"   # all *_width knobs high: wrapping is author-controlled
```

- Imports hand-ordered and comment-grouped (`// Internal`, external below); rustfmt
  `reorder_imports = false`, `merge_derives = false`.
- Lints via **cargo-cranky** (curated, not inline attributes): deny `unsafe_code`,
  `needless_return` (no explicit trailing `return` — pairs with early-return style),
  `redundant_field_names`; allow `missing_docs`. Each crate's `lib.rs` opens with
  `#![forbid(unused_must_use, unused_imports, unused_mut, unused_variables, private_interfaces)]`.
- Edition 2024; every crate `publish = false`; one workspace, members list hand-grouped with
  comment banners.
- `println!`/`eprintln!` banned outside tests. Logging: ;
  **Foundation uses `tracing`** (it's the native citizen of the
  Axum/tower/Tokio stack we decided in `architecture.md` §1, and spans matter for a gateway
  that fans out to providers).

## 3. Rust: error handling

**Sign-off:** _unsigned_

Two regimes, by layer:

| Layer                          | Regime                                                       |
| ------------------------------ | ------------------------------------------------------------ |
| Services / handlers / workers  | `anyhow` internally; convert at the boundary to the typed web error (log-then-convert: `.map_err(|err| { error!(...); ApiError::from_error(err) })?`) |
| `crates/clients/*` (API clients)| Hand-rolled **four-bucket taxonomy** (below) — typed, no anyhow |
| `unwrap`/`expect`              | Never in production paths; fine in tests, generators, and provably-infallible spots (`expect("static json")`) |

The four-bucket client taxonomy (one file per bucket under `src/error/`):

1. `{X}ClientError` — failures before/independent of the HTTP call: missing key, request
   serialization, pre-flight validation.
2. `{X}SpecificApiError` — classified server responses, one variant per meaningful status
   (`Unauthorized`, `RateLimited`, `PromptModerated`, …), each carrying
   `raw_http_body: Option<String>`.
3. `{X}GenericApiError` — unclassified: parse-error-with-body, uncategorized
   status-and-body, transport errors.
4. `{X}Error` — the top-level sum `{ Client, ApiSpecific, ApiGeneric }` with `From` impls
   per bucket, plus a `classify_{x}_http_error` free function that promotes generic
   responses into specific variants (body sniffing lives there and only there).

Shared plumbing (reqwest factory, retry/backoff, the taxonomy traits) lives in
`crates/clients/client-core` (`architecture.md` §5). The specific-error buckets are what the
job state machine maps onto `failure_code` — classification is a client concern, policy is a
service concern.

## 4. Rust: layering

**Sign-off:** _unsigned_

- **Thin handlers.** A handler: utoipa annotation, extractors, auth/entitlement guards as
  early returns, one call into `services/*`, response mapping. Nothing else.
- **No SQL outside `db/queries`** — one function per query, sqlx compile-checked
  (`architecture.md` §5). Handlers never see SQL; services call query functions.
  **Production-code rule only: tests in any crate may hit the database.** The dedicated
  `test_fixtures` crate (`create_test_db()`, fixture builders — itself SQL-free, calling
  `db/queries` to create rows) is imported as a dev-dependency wherever a crate wants
  end-to-end DB integration tests (`testing.md` §3–§5).
- Clients (`crates/clients/*`) are pure HTTP — no DB dependencies, no service imports.
- Never hold a DB transaction across provider I/O (`architecture.md` §9); clock is injected,
  not read ambiently (`testing.md` §1).
- Prefer combinator chains (`.as_ref().map(..).unwrap_or(..)`) over nested `if let`; prefer
  `match` with early `return` for extraction-or-bail.

## 5. Rust: tests

**Sign-off:** _unsigned_

Full strategy in `testing.md`; style rules here:

- Inline `#[cfg(test)] mod tests` in the same file as the code under test (integration-style
  suites may add a `tests/` dir).
- Inside a test module: constants first, then test cases — grouped into sub-modules when
  there are 2+ logical groups (ArtCraft: `explicit_checks` with literal
  `assert_serialization(Variant, "slug")` per variant, `mechanical_checks` with round-trips
  and exhaustiveness) — helper functions last.
- Golden/stability tests guard every wire format; strum `EnumIter`/`EnumCount` derived only
  under `#[cfg(test)]` to prove hand-written lists exhaustive.

## 6. TypeScript

**Sign-off:** _unsigned_

- Formatting: Prettier defaults with 2-space indent; **semicolons, double quotes**. ESLint
  flat config: `js.recommended` + `typescript-eslint.recommended` + `react-hooks`.
- `strict: true`; no `any` (use `unknown` and narrow).
- **Named function declarations for components** (`export function MediaCard(...)`), arrow
  consts for helpers/callbacks. No `React.FC`. Props via local `interface {X}Props`.
- Named exports only — no default exports, pages included.
- Early returns in components too: bail on loading/empty/error states before the main render.
- `interface` over `type` except for unions/small aliases.
- **API-boundary enums are string-literal unions** — that's simply what the OpenAPI codegen
  emits, and unions accept plain wire strings (`"queued"`) with zero runtime footprint.
  Hand-written TS `enum` is **permitted** where a namespaced constant set reads better;
  known trade-offs, which is why unions stay the default near wire data: `enum` compiles to
  a runtime object (disallowed under TS `erasableSyntaxOnly`, the Node type-stripping mode)
  and a string enum won't accept a bare matching literal without a cast. (ArtCraft de facto
  agrees: 2 enums total, unions everywhere else.)
- The API client is generated, never hand-written (DIVERGENCE from ArtCraft's hand-rolled
  class-based `ApiManager` — codegen was decided here precisely to retire that pattern).
- Directory-by-kind: `pages/`, `components/` (+ `components/ui/` for shadcn primitives),
  `hooks/`, `lib/`, `context/`, `layouts/`, `types/`; `@/` alias to `src/`.
- Tailwind utilities inline; `cn()` (`clsx` + `tailwind-merge`) for conditional classes.
- State: local `useState` first, React Context for cross-cutting concerns (auth, theme);
  reach for a store (Zustand) only when an app demonstrably needs one.

## 7. Schema declaration style

**Sign-off:** _unsigned_

- **`created_at` always sorts above the other timestamp columns** in a table declaration
  (then `updated_at`/event timestamps in whatever order reads best). Dimensional timestamps
  that are part of a table's key (e.g. a rollup's `period_start`) are data, not row
  metadata, and stay with the payload columns.
- Column order overall: `sequence_id`, `id`, FKs, payload, timestamps (`ids.md` §4,
  `naming_conventions.md` §4).
- Every design-doc table lists a `null?` column; nullability is decided at design time, not
  at migration time.

## 8. Review checklist (the recurring catches)

**Sign-off:** _unsigned_

- Wrapping/nesting creeping past ~2 levels → extract or invert with guards.
- A `_ =>` wildcard on one of our enums → almost always wrong; exhaustive matches are the
  point (`../reports/model-enums-expanded-considerations.md`).
- A hand-written thing that should be generated (TS enum, CHECK list, agent doc).
- SQL or reqwest in a layer that shouldn't see it.
- `unwrap` on a fallible path; a swallowed error without a `error!`/`warn!` log.
- An unpadded markdown table in design docs.
