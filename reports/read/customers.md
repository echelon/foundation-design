# Naming (and modeling) our customers' customers

Report, 2026-07-05. The question: we sell an API to developers ("our customers" = accounts);
they build products for *their* users, whose activity flows through us. What do we call that
third party, and what do we build around it?

---

## Recommendation: **end users** (`end_users`, field `end_user`, prefix `endu_`)

"End user" is the term the industry already uses for exactly this position in the chain: the
person at the end of the value chain, distinct from the paying integrator. It's unambiguous in
every sentence we'll ever write — "the account's end users" can't be confused with "our
customers," and it survives the docs, the dashboard, and the schema without a glossary.

Precedents in our exact shape:

| Who                | What they call it / do                                                       |
| ------------------ | ---------------------------------------------------------------------------- |
| OpenAI             | `user` field on API requests — "a unique identifier representing your end-user," used for abuse attribution (newer APIs: `safety_identifier`) |
| Anthropic          | `metadata.user_id` — same purpose, explicitly described as your end-user     |
| Auth0/Clerk        | "end users" as the standing term for your-customers'-users                   |
| Stripe Connect     | calls them "customers of connected accounts" — workable only because Stripe's object graph disambiguates; the prose is famously confusing |
| Metering platforms | (OpenMeter etc.) call them "subjects" — precise but sterile                  |

## Rejected candidates

| Term               | Why not                                                                       |
| ------------------ | ----------------------------------------------------------------------------- |
| `customers`        | Your own objection is right: ambiguous. Every doc sentence would need "their customers, not ours." Stripe Connect proves the confusion is permanent |
| `clients`          | Collides fatally with API clients (`crates/clients/`), client libraries, OAuth clients |
| `consumers`        | In API-land, "API consumer" already means *our* customer                       |
| `tenants`          | Implies org-shaped sub-accounts; end users are usually individuals. Also collides with our own multi-tenancy vocabulary |
| `subjects`         | Precise (metering jargon) but cold and opaque to normal developers             |
| `principals`       | IAM jargon; wrong register                                                     |
| `members`          | Taken (`account_members` = teammates)                                          |
| `downstream_users` | Accurate but verbose; "end user" says the same thing shorter                   |

## What it's for (why model it at all)

1. **Abuse attribution.** When upstream providers flag content, we (and our customer) can
   isolate *which of their users* did it instead of suspending the whole account — exactly why
   OpenAI/Anthropic ask for the field. We can pass a hashed form upstream where providers
   support it.
2. **Per-end-user usage & spend rollups.** Our customers can meter and bill *their* users off
   our data: `GET /v1/billing/usage?end_user=...`. This is a genuine product differentiator
   for an aggregator (nobody hands integrators clean per-end-user cost data today).
3. **Per-end-user rate/spend limits.** Account-configured rules: "no end user may spend more
   than $5/day through my key." Kills the classic leaked-key / hostile-user blowup.
4. **Log filtering.** The debug timeline filtered to one end user is a support superpower for
   our customers.

## Proposed mechanics (v1-light)

- Request field on any generate/chat call: `"end_user": "<their opaque id, ≤256 chars>"`.
  Optional. We never see PII by design — docs tell integrators to send *their* stable ID or a
  hash, not an email.
- Auto-upsert on first sight (no pre-registration API needed in v1).
- `jobs.end_user_id` nullable FK; flows into request_logs and usage rollups.

### Schema sketch: `end_users`  (`endu_`)

| column       | type        | notes                                       |
| ------------ | ----------- | ------------------------------------------- |
| id           | text PK     | `endu_...`                                  |
| account_id   | text FK     | → accounts                                  |
| external_id  | text        | the customer's identifier for this user     |
| first_seen_at| timestamptz |                                             |
| last_seen_at | timestamptz | throttled updates                           |
| metadata     | jsonb       | customer-supplied, opaque to us             |
|              |             | UNIQUE (account_id, external_id)            |

Not yet designed (future, if demand): end-user-scoped limits table, end-user blocklist,
per-end-user webhook filtering.

**If approved, this gets folded into `design/schema.md` and `design/api.md`.**
