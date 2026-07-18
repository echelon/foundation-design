# Database Conventions

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). NEW: collects every database-wide convention in one
place — the schema-declaration style extracted from `conventions_coding.md` §7 plus the
integrity-in-code rules (DECIDED 2026-07-17). Naming (table/column names, `is_`/`_at`, FK
column names): `conventions_naming.md` §4. Table definitions live with their domain docs;
the master index is `schema.md` §7.

---

## 1. Integrity lives in code, not the database (DECIDED 2026-07-17)

**Sign-off:** _unsigned_

- **NO database triggers.** Ever. Behavior attached invisibly to a table makes migrations
  extremely hard to reason about — a migration that touches a triggered table fires logic the
  migration author can't see. Anything a trigger would do (audit rows, cache updates,
  derived-column maintenance) is done explicitly in the transaction by `db/queries` /
  `services/*` code, where it is reviewable, testable, and greppable.
- **NO actual foreign-key constraints.** Columns documented as `text FK` in the design docs
  are **logical** references — the naming convention and the code enforce them; no
  `REFERENCES` clause exists in the DDL. Real FK constraints make large migrations,
  backfills, and table rewrites extremely hard to reason about (ordering restrictions,
  cascades, lock amplification) for integrity we already guarantee at the single write path
  (`conventions_coding.md` §4: no SQL outside `db/queries`).
- Same spirit as "no native Postgres enums" (§2): the database stores state; the rules about
  state live in Rust.

## 2. Column types & constraints

**Sign-off:** _unsigned_

- **No native Postgres enums** (DECIDED 2026-07-05 — `schema.md`): enum-ish columns are
  `text` + CHECK constraint; the closed sets are owned by `crates/core/enums` and the CHECK
  lists are generated from Rust. (CHECKs are value-shape validation on one column — cheap,
  local, migration-friendly — unlike triggers/FKs, which are cross-table behavior; that's
  why they stay.)
- All timestamps are `timestamptz` (`{foo}_at` — `conventions_naming.md` §4). `created_at`
  defaults to `now()`.
- Money is `bigint` micro-USD (`{name}_micros`); catalog unit rates are nano-USD
  (`{name}_nanos`). Newtypes in `crates/core/money`.
- Every table: `sid bigint generated always as identity` (never exposed, never joined on,
  `ORDER BY` only) + `id text primary key` = `{prefix}_{26-char Crockford ULID}` (`ids.md`).

## 3. Schema declaration style (moved from `conventions_coding.md` §7)

**Sign-off:** _unsigned_

- **`created_at` always sorts above the other timestamp columns** in a table declaration
  (then `updated_at`/event timestamps in whatever order reads best). Dimensional timestamps
  that are part of a table's key (e.g. a rollup's `period_start`) are data, not row
  metadata, and stay with the payload columns.
- Column order overall: `sid`, `id`, FKs, payload, timestamps (`ids.md` §4,
  `conventions_naming.md` §4).
- Every design-doc table lists a `null?` column; nullability is decided at design time, not
  at migration time.

## 4. Operational defaults

**Sign-off:** _unsigned_

- High-volume log tables: monthly `PARTITION BY RANGE (created_at)`, minimal indexes,
  partition machinery from day one (`logging.md` §3).
- Partial indexes are a first-line tool (the one-owner rule, `is_org_visible` read paths) —
  we are Postgres by decision (`architecture.md` §1) and don't design around MySQL's lack of
  them (`accounts.md` §2 account_members note).
- Postgres operational gotchas (autovacuum, XID wraparound, fillfactor): `architecture.md` §9.
