# Do we need integer PKs at all? (Postgres edition)

Report, 2026-07-05. Question: the MySQL habit was `int unsigned auto_increment` PK + a
`varchar(32)` unique public ID, using the int for (1) replication, (2) ordering, (3) sorting.
Does Postgres need that? Answer: **no — the prefixed string PK alone is sufficient**, with one
optional nuance for the ledger. Reasoning below, including "what if we need ordered results!?"

---

## Why MySQL wanted the int (and why Postgres doesn't)

| MySQL reason              | What it really was                                                    | Postgres situation                                     |
| ------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------ |
| (1) "Replication"         | Mostly folklore for InnoDB **clustered index** behavior: rows are physically stored in PK order, so random PKs (UUIDv4) cause page splits, buffer-pool churn, and bloated secondary indexes (every secondary index stores the PK). Sequential ints kept inserts appending. Row-based replication itself never cared | Postgres tables are **heaps** — rows have no PK-imposed physical order, secondary indexes point to heap TIDs, not the PK. The clustered-index penalty *does not exist*. The only real cost of string PKs is B-tree size, and our time-ordered IDs keep the PK index append-mostly anyway |
| (2) Ordering              | `ORDER BY id` = insertion order for free                                | Our IDs are ULID-style: 48-bit ms timestamp first, so **`ORDER BY id` IS chronological order**, lexicographically, using the PK index. Same capability, no extra column |
| (3) Sorting/joins compact | 4–8 byte keys = small indexes, fast comparisons                         | Real but bounded: text PKs cost ~3–4× index size and memcmp vs int compare. At our row counts this is noise; measured impact shows up only at billions of rows per table, and the fix then is targeted, not architectural |

## "What if we need to return ordered results!?"

Covered, three ways:

1. **`ORDER BY id`** — chronological to millisecond precision, index-backed, works for every
   list endpoint (and cursor pagination: `WHERE id > $after ORDER BY id LIMIT n` is a single
   index range scan — this is exactly how our `?after=` cursors work).
2. **`ORDER BY created_at`** — exists on every table anyway; use where human-facing sort
   semantics matter.
3. **Caveat, same-millisecond ties:** within one ms, ULID random bits make order arbitrary
   (not insertion order). For 99% of tables nobody can observe or care. Where *strict arrival
   order* is auditably meaningful — realistically only `ledger_entries` — add a **non-PK**
   `seq bigint generated always as identity` column. It's an ordering column, not an
   identifier: never exposed, never joined on, invisible to the API. This is the one
   defensible remnant of the old pattern, and it's optional (balance_after_micros under the
   row lock already gives the ledger a total order per account).

## Other things the int PK is sometimes credited with

- **"Smaller FKs everywhere"** — true, accepted cost (architecture.md §3). The devops win of
  self-describing IDs in every log line was your explicit priority, and it's the right call.
- **Gapless numbering** — neither sequences nor identity columns are gapless (rollbacks burn
  values) — the int never actually gave you this in MySQL either. If a gapless business
  number is ever needed (invoice numbers), that's its own counter table with a row lock.
- **Cheap "how many rows" intuition** — max(id) as a proxy dies with strings; use
  `reltuples` estimates. Minor.
- **Partitioning keys** — our partitioned tables key on `created_at`, not id. No loss.

## Recommendation (matches what's in the design)

- **No integer PKs anywhere.** One `text` PK per table, `{prefix}_{crockford128}`,
  time-ordered payload.
- Cursor pagination and default sorts use the PK.
- Optionally add non-PK `seq bigint identity` to `ledger_entries` only, if you want
  belt-and-suspenders arrival ordering for audits. (Not currently in schema.md; say the word.)
- If a specific table someday measurably suffers (billions of rows, join-heavy), the targeted
  fix is an internal surrogate int on THAT table — a migration, not a redesign. We are not
  pre-paying that complexity across the whole schema.
