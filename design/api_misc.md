# API Misc (endpoint groups without a home doc)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). SPLIT from `api.md` (2026-07-17): this doc collects
the specific endpoint groups that have no dedicated sibling doc — catalog, files, billing,
account/team/keys, MCP mount, public. Whole-API-shape conventions (auth, versioning, errors,
webhook signing): `api_overview.md`. Endpoints are served across three ingress binaries per
`microservices.md`. Base URL placeholder: `https://api.example.dev`.

---

## 1. Jobs

**Sign-off:** _unsigned_

**Moved to `jobs.md` (2026-07-05)** — endpoints, the status machine (internal + external),
cancel/refund semantics, and the retry policy all live there. Jobs are the universal
primitive; every generate call (`api_overview.md` §3) and every LLM call (§2) produces one.

## 2. LLM

**Sign-off:** _unsigned_

**Moved to `apis_llm.md` (2026-07-14):** the OpenAI-compatible `/v1/chat/completions` plane —
param pass-through rules, response deltas (job ID as completion `id`), streaming, metered
billing, and the reserved LLM paths.

## 3. Catalog (read-only; served from the code catalog)

**Sign-off:** _unsigned_

| Endpoint                               | Method | Purpose                                        |
| -------------------------------------- | ------ | ---------------------------------------------- |
| `/v1/models`                           | GET    | Filter by media_type, owner, modality, status  |
| `/v1/models/{model}`                   | GET    | Metadata, modalities, prices, deprecation info |
| `/v1/models/{model}/schema/{modality}` | GET    | JSON Schema for that endpoint                  |

## 4. Files

**Sign-off:** _unsigned_

| Endpoint         | Method | Purpose                                                |
| ---------------- | ------ | ------------------------------------------------------ |
| `/v1/files`      | POST   | Multipart upload → `file_...` usable in any input slot |
| `/v1/files/{id}` | GET    | Metadata + CDN URL                                     |

Inputs may also be plain HTTPS URLs (we mirror them to R2 before dispatch).
Tables (`files` + `outputs`) and the storage/provenance split: **`files.md`**.
(Naming analysis — files vs media vs artifacts: `../reports/read/files-naming.md`.)

## 5. Billing & spend

**Sign-off:** _unsigned_

| Endpoint                      | Method          | Purpose                                                   |
| ----------------------------- | --------------- | --------------------------------------------------------- |
| `/v1/billing/balance`         | GET             | Current balance (micros + display)                        |
| `/v1/billing/ledger`          | GET             | Ledger entries: topups, charges, refunds — the spend list |
| `/v1/billing/usage`           | GET             | Rollups: spend by day / model / api key / end_user        |
| `/v1/billing/topup`           | POST            | Create Stripe payment for a chosen amount                 |
| `/v1/billing/payment-methods` | GET/POST/DELETE | Card on file management                                   |
| `/v1/billing/auto-refill`     | GET/PUT         | Threshold, refill amount, daily max, enable/disable       |
| `/v1/billing/spend-limits`    | GET/PUT         | Weekly/monthly caps (behavior: block)                     |

"Show users what they've spent": `GET /v1/jobs` (each row: job, status, model, cost) is the
per-job view; `/v1/billing/ledger` is the money view; `/v1/billing/usage` is the chart view.

## 6. Account, team, keys

**Sign-off:** _unsigned_

| Endpoint                         | Method          | Purpose                                                                                                                         |
| -------------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `/v1/account`                    | GET/PATCH       | Account settings                                                                                                                |
| `/v1/account/members`            | GET             | Team list                                                                                                                       |
| `/v1/account/members/{id}`       | PATCH/DELETE    | Role change, removal                                                                                                            |
| `/v1/account/invitations`        | GET/POST/DELETE | Invite teammates                                                                                                                |
| `/v1/api-keys`                   | GET/POST        | Create: `title` (immutable), `description`, `expires_at?` — `foundation_api_...` secret shown once                              |
| `/v1/api-keys/{id}`              | PATCH/DELETE    | PATCH edits description only; DELETE = revoke                                                                                   |
| `/v1/webhook-secret`             | GET/POST        | View/rotate `foundation_whsec_...` (two active during rotation)                                                                 |
| `/v1/auth/{signup,login,logout}` | POST            | Console session auth (`auth.md`; full auth surface: `accounts.md` §6 — login_methods, sessions, audit logs, ownership transfer) |
| `/v1/auth/password-reset`        | POST            | Token flow                                                                                                                      |

## 7. MCP

**Sign-off:** _unsigned_

`/mcp` — streamable-HTTP MCP endpoint mounted in api-gateway; tools mirror this API
(generate/cost/jobs/models). Full design: `mcp.md`.

## 8. Public (unauthenticated)

**Sign-off:** _unsigned_

| Endpoint              | Method | Purpose                                              |
| --------------------- | ------ | ---------------------------------------------------- |
| `/v1/public/rankings` | GET    | Aggregated model usage stats (see `rankings.md`)     |
| `/llms.txt`           | GET    | Redirect to docs-domain agent docs (see `agents.md`) |
