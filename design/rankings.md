# Rankings (public model usage stats)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). OpenRouter's version of this is **"LLM Rankings"**
(https://openrouter.ai/rankings) — token market share by model/lab/app, famously
headline-generating (press cites it as a de-facto industry index; their "State of AI" posts
are built on it). Ours covers what they can't: **image, video, audio, mesh, splat AND LLMs**
in one place.

**Product name:** proposed **"Rankings"** (rides the mindshare OpenRouter built; instantly
understood). Alternates if you want distinct branding: **Pulse** (living heartbeat of the
media-gen market), **Index**, **Observatory**, **Leaderboard**. bt to pick; docs say
Rankings until then.

---

## 1. What the world sees

**Sign-off:** _unsigned_

- Public dashboard (on `www`): per media type — model market share by jobs and by output
  volume (video seconds, images, tokens), trend lines, mover-of-the-week, lab/owner
  aggregates. Time ranges: day / week / month / all-time.
- Periodic writeups ("State of AI Video, Q3 2026") built from the same data — the
  headline-generating part.
- Public API: `GET /v1/public/rankings?media_type=video&period=week` — unauthenticated,
  CDN-cached; being embeddable/citable is the growth loop.

**Privacy rules (hard):** aggregates only; a model appears only when ≥ K distinct accounts
used it in the period (K=5 to start — prevents deanonymizing a single big customer); no
account, end-user, or dollar dimensions ever (volume shares, not revenue); our own dashboard
traffic (`source=dashboard`) included but flagged internally.

## 2. Tables

**Sign-off:** _unsigned_

One rollup table, hourly grain; coarser grains are derived, not stored separately.

### usage_rollups  (`roll_`)

| column             | type        | notes                                                                       |
| ------------------ | ----------- | --------------------------------------------------------------------------- |
| sequence_id        | bigint      | identity; never exposed (ids.md §4)                                         |
| id                 | text PK     | `roll_...`                                                                  |
| grain              | text        | `hour \| day \| week \| month`                                              |
| period_start       | timestamptz | UTC bucket start                                                            |
| media_type         | text        | `image \| video \| audio \| mesh \| splat \| text`                          |
| model              | text        | catalog slug                                                                |
| modality           | text        |                                                                             |
| provider           | text        | upstream actually used                                                      |
| source             | text        | `api \| dashboard \| mcp` (also rolled up as `all`)                         |
| jobs_total         | bigint      |                                                                             |
| jobs_succeeded     | bigint      |                                                                             |
| jobs_failed        | bigint      |                                                                             |
| output_units       | numeric     | unit depends on media: seconds, images, tokens…                             |
| tokens_in          | bigint      | LLM only                                                                    |
| tokens_out         | bigint      | LLM only                                                                    |
| distinct_accounts  | int         | PRIVATE — powers the K-anonymity gate, never exposed                        |
| spend_micros       | bigint      | PRIVATE — internal analytics only                                           |
| p50_latency_ms     | int         | queue→terminal; interesting public stat                                     |
| created_at         | timestamptz |                                                                             |
|                    |             | UNIQUE (grain, period_start, media_type, model, modality, provider, source) |

Tiny forever: even 500 models × ~50 dimension combos × 8,760 hours/year is a few million
skinny rows/year — no partitioning pressure for a long time.

## 3. Jobs (queue tasks)

**Sign-off:** _unsigned_

| Task                  | Schedule                | What it does                                                |
| --------------------- | ----------------------- | ------------------------------------------------------------ |
| `rollup_usage:hour`   | ~5 min past each hour   | Aggregates `jobs` for the closed hour → upsert (`ON CONFLICT` replace) — idempotent, re-runnable, backfillable |
| `rollup_usage:day`    | daily                   | Sums the 24 hourly rows → day rows (likewise week/month from days) |
| `rollup_repair`       | daily                   | Re-rolls the trailing 48h (late-finishing jobs move buckets) |
| `rankings_snapshot`   | daily                   | Exports public JSON snapshots to R2 (`rankings/{period}.json`) for writeups + archival |

Recorded on the `jobs` table clock: a job counts in the bucket of its `completed_at` (terminal
time), not enqueue time — "what ran" not "what was requested".

## 4. Serving: live rollups + static snapshots (both)

**Sign-off:** _unsigned_

Your either/or question — the answer is both, split by purpose:

- **Live:** `GET /v1/public/rankings` reads `usage_rollups` directly (rows are pre-aggregated,
  queries are trivial), behind CDN caching (5–15 min TTL). The dashboard uses this — always
  current, no publish step.
- **Static:** the daily `rankings_snapshot` JSON in R2 is the citable artifact — stable URLs
  per period for writeups, press, and historical archive (snapshots never change once written;
  live data can shift as late jobs land).

No third system: no OLAP store, no separate analytics DB until Postgres rollups actually hurt
(they won't for a long time — the raw `jobs` table is the only big scan, and it's one
hour-slice at a time).
