# End Users (our customers' customers)

**Sign-off:** bt 2026-07-06

<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). Terminology **"end users" signed off by bt
2026-07-05** (analysis: `../reports/customers.md`). Decided same day: always optional,
auto-upsert, customer-supplied `external_id` ≤256 chars, our internal `enduser_` IDs.

---

## 1. The contract

**Sign-off:** bt 2026-07-06

- Any generate/cost/chat request MAY include `"end_user": "<their opaque id, ≤256 chars>"`.
  **Never required.** Absent = null everywhere downstream; no behavior changes.
- First sight of a new `(account, external_id)` auto-upserts an `end_users` row and mints
  `enduser_...`. No pre-registration API.
- Docs instruct customers to send *their own stable ID or a hash* — never emails/PII. The
  value is opaque to us.

## 2. Schema

**Sign-off:** bt 2026-07-06

`end_users` table and `jobs.end_user_id` are canonical in `schema.md` (§1, §2). Recap:

| column        | type        | notes                             |
| ------------- | ----------- | --------------------------------- |
| id            | text PK     | `enduser_...`                     |
| account_id    | text FK     | → accounts                        |
| external_id   | text        | customer's identifier, ≤256 chars |
| first_seen_at | timestamptz |                                   |
| last_seen_at  | timestamptz | throttled updates                 |
| metadata      | jsonb       | customer-supplied, opaque         |
|               |             | UNIQUE (account_id, external_id)  |

## 3. The superpowers (all approved 2026-07-05)

**Sign-off:** bt 2026-07-06

### Abuse attribution

When a provider flags content or a key looks abused, we (and the customer) isolate *which end
user* instead of nuking the account. Where providers accept a user identifier
(OpenAI `user`/`safety_identifier`, Anthropic `metadata.user_id`), we pass through a **hash**
of `enduser_id` — attribution upstream without leaking customer identifiers.

### Per-end-user usage & spend rollups

`GET /v1/billing/usage?end_user=...` and `GET /v1/jobs?end_user=...` — customers can meter
and bill their own users from our data. Served from the jobs table at first (indexed
`(account_id, end_user_id, created_at)`), rollup table if it gets hot.

### Per-end-user spend / rate limits (optional, account-configured)

We won't be building this in v1, but could in the future.

Future table (not in v1 schema; listed in `schema.md` §7):

| column       | type    | notes                                      |
| ------------ | ------- | ------------------------------------------ |
| id           | text PK | `eulimit_...`                              |
| account_id   | text FK | → accounts                                 |
| scope        | text    | `all_end_users, single` (+ end_user_id FK) |
| window       | text    | `hourly, daily, weekly, monthly`           |
| limit_micros | bigint  | null = no spend cap                        |
| max_requests | int     | null = no rate cap                         |
| behavior     | text    | `block`                                    |

Enforced in the admission transaction, same pattern as account spend limits. A request with
no `end_user` can't hit per-end-user limits (documented; accounts wanting enforcement make
the field mandatory *in their own client code*).

### Log filtering

`end_user_id` flows onto `request_logs`; console log explorer and job list filter by it.
"Show me everything user X did" is the support superpower.

## 4. Boundaries

**Sign-off:** bt 2026-07-06

- End users are **not** identities: no auth, no logins, no cross-account linkage, no global
  identity graph. Strictly per-account labels.
- We never join end users across accounts, even for abuse (account-level signals only).
- Rankings/public stats never expose end-user dimensions.
