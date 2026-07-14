# Identifiers

**Sign-off:** bt 2026-07-14

<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). Peeled from `architecture.md` §3, which now points
here. Siblings: `architecture.md`, `schema.md`, `naming_conventions.md`. Background analysis:
`../reports/read/identifiers.md`.

---

## 1. Format

**Sign-off:** bt 2026-07-14

Every row's public identifier is a **prefixed ULID string**: `{prefix}_{26-char Crockford
base32}`, e.g. `job_01j0qs3fw9x7k2m5n8p1r4t7v0`. This string lives in the `id` column and is
the **primary key** (see §4 for the companion `sid`). Used everywhere: FKs, joins,
logs, API responses. A bare ID in a log line tells you what it is — that's the point.

The payload is a standard **ULID — exactly 128 bits**: a 48-bit unix-millisecond timestamp
followed by **80 bits of randomness**, canonically encoded as **exactly 26 Crockford base32
characters** (26 × 5 = 130 bits of capacity for 128 bits, so the first character can only be
`0–7`). The leading section (`01j0qs3fw9`) is the timestamp: sortable, index-friendly.

| Property           | Answer                                                                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Payload length     | Always exactly 26 chars (canonical ULID text)                                                                                                         |
| Total length       | `len(prefix) + 1 + 26` — constant per type: `job_` → 31, `acct_` → 32, `enduser_` → 35                                                                |
| Globally constant? | Only if we padded prefixes to equal width — don't; per-type constancy is what validation and column sizing care about, and `text` doesn't care at all |
| Charset            | Crockford base32: `0-9 a-hjkmnp-tv-z` — no `i l o u`, case-insensitive by spec (we emit lowercase only), no vowel-slang accidents beyond that         |

Validation regex per type is therefore trivial and total:
`^job_[0-7][0-9a-hjkmnp-tv-z]{25}$`.

Cost accepted: TEXT keys make PK/FK indexes roughly 3–4× larger than bigint and joins
slightly slower. At our scale this is noise; debuggability wins. (Full integer-PK analysis:
`../reports/read/identifiers.md`.)

## 2. Entropy & collisions

**Sign-off:** bt 2026-07-14

**Enough entropy?** 80 random bits *per millisecond*. Collision requires two IDs in the SAME
millisecond among 2⁸⁰ ≈ 1.2×10²⁴ values. Even a sustained 100,000 IDs per millisecond
(≈ 100M/sec — absurdly beyond us) gives a per-ms birthday probability of ~4×10⁻¹⁵; across a
year that's still ~10⁻⁴ cumulative at that impossible rate, and reality is 5+ orders of
magnitude below it. Plus every PK is UNIQUE — the theoretical collision is one retried
insert, not corruption. Verdict: yes, comfortably, through trillions of rows.

## 3. The embedded millisecond clock (DECIDED 2026-07-14: keep it)

**Sign-off:** bt 2026-07-14

Question considered: should the ID embed a millisecond timestamp at all, or spend all 128
bits on randomness (UUIDv4-style)? **Decision: keep the 48-bit clock — it's the single
highest-value design choice in the format.**

What it costs:

- **Entropy: 48 of the 128 bits.** That leaves 80 random bits per millisecond, and §2 shows
  that's comfortable through trillions of rows. The clock costs us nothing we'd ever miss.
- **IDs leak creation time.** Anyone holding an ID can read when the row was minted.
  Acceptable for every entity we have; if some future resource must hide its age, that one
  type can use a random payload — a per-prefix choice, not an architecture change.
- Timestamp is **writer clock time**, subject to skew across hosts. It's for ordering,
  locality, and debugging — `created_at` and `sid` (§4) remain the DB-authoritative
  truth.

What it buys, especially at distributed scale:

- **The ID is self-ordering and self-dating with zero coordination.** In a microservices
  world where downstream systems see *only* the ID — a log line, a queue message, a webhook
  payload, a cache key, an analytics event — they can sort chronologically and recover
  creation time to the millisecond **without** `created_at`, without `sid`, without a
  DB round trip, and without any shared sequence service. Independent processes on separate
  hosts mint IDs that interleave correctly to clock precision.
- **B-tree locality.** Time-ordered high bits keep PK inserts append-mostly (right-leaning),
  not random page splits — and `ORDER BY id` is chronological, index-backed, which is exactly
  how `?after=` cursor pagination works.
- **Debugging leverage.** "This job ID is from 14:32:07.114" read straight off the ID is a
  real, daily devops win — same spirit as the self-describing prefix.

Skip the ULID spec's optional **monotonic mode** (incrementing the random part within one
millisecond): it adds shared-state contention inside one process and guarantees nothing
across processes. Our ordering story — millisecond precision from the clock, `sid`
where strict arrival order matters — already covers it.

## 4. `sid` — every table (REVISED 2026-07-14)

**Sign-off:** bt 2026-07-14

Every table gets an auto-increment column: **`sid bigint generated always as identity`**
(named `sid` — bt 2026-07-14, less to type), declared first, with the prefixed string `id`
(the primary key) declared immediately after it.

- **Never exposed** in any API response, log line, or webhook. **Never joined on.** FKs
  reference `id` only.
- Legitimate use: **`ORDER BY` clauses** — strict, gap-tolerant arrival order as observed by
  the database, immune to same-millisecond ULID ties and writer clock skew.
- Rationale: insurance. If it winds up wasteful it can be dropped later (cheap migration);
  retrofitting it later is also possible but backfill order would be lost. This supersedes
  `../reports/read/identifiers.md`'s "ledger-only, optional" recommendation — bt decided
  belt-and-suspenders everywhere.
- Not gapless (rollbacks burn values) — anything needing gapless business numbering (invoice
  numbers) is its own counter table.
- **Design docs list it explicitly (bt 2026-07-14):** every table listing in `schema.md` and
  its siblings opens with the `sid` row, `id` directly beneath — no implicit columns.

## 5. Prefix registry

**Sign-off:** bt 2026-07-14

- The prefix registry lives in `crates/core/ids` as one enum with a compile-time uniqueness
  test. Each table's prefix is listed with its schema (see `schema.md` §7 pointer index).
- Prefixes are short, lowercase, underscore-free (`job_`, `acct_`, `enduser_`, `txn_`,
  `emailver_`, …).
- Secrets are the exception to the short-prefix rule — service-declaring by design:
  `foundation_api_` (API keys) and `foundation_whsec_` (webhook signing secrets); the
  `foundation` half swaps when the product is named.
