# Microservices (deployable topology)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-14). REVISES `architecture.md` §5's earlier
"one api-server" decision: there are now **two web backend gateways** plus the worker.
Siblings: `architecture.md`, `api.md`, `auth.md`.

---

## 1. The services

**Sign-off:** _unsigned_

| Service               | Crate                   | Serves                                                                                                                                                                           | Auth accepted                                    |
| --------------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **api-gateway**       | `bin/api-gateway`       | The product we sell: `/v1/any/*`, `/v1/models/*` generate/cost, `/v1/chat/*`, `/v1/jobs/*`, `/v1/files/*`, `/v1/models` catalog reads, `/mcp`, `/v1/public/*`, `/stripe/webhook` | API key, dashboard session cookie (sandbox), MCP |
| **dashboard-gateway** | `bin/dashboard-gateway` | The console's backend: `/v1/auth/*`, account/members/invitations, billing console, spend views, audit logs, log explorer, session management                                     | Session cookie only                              |
| **worker**            | `bin/worker`            | Queue consumer: dispatch/poll/ingest/webhooks/rollups/sweepers                                                                                                                   | none (internal)                                  |
| **admin-cli**         | `bin/admin-cli`         | Migrations, ops one-offs                                                                                                                                                         | operator                                         |

The dashboard's **sandbox generate calls go to api-gateway** (cookie-authed,
`source = dashboard`) — that's what keeps dashboard billing/logging identical to API billing
(`architecture.md` §4); the dashboard-gateway never proxies generation.

## 2. Why split (and why this split)

**Sign-off:** _unsigned_

- **Blast radius:** console deploys (frequent, UI-driven churn) can never take down the paid
  API; the api-gateway deploys rarely and carefully.
- **Security surface:** cookie/CSRF/browser concerns live almost entirely in one binary;
  the api-gateway's cookie acceptance is limited to the sandbox generate paths (CSRF-guarded,
  `SameSite=Lax` + custom-header check for mutations — `auth.md` §1).
- **Scaling shape:** api-gateway scales with customer traffic (held-open LLM streams,
  webhooks in); dashboard-gateway scales with humans-in-consoles — different curves,
  different instance counts.
- **Versioning:** the api-gateway speaks the date-versioned public contract (`api.md` §1);
  dashboard endpoints are internal, unversioned, free to churn with the frontend.

Both gateways are thin binaries over the SAME crates — routers from `core/api-defs`, logic
in `services/*`, one `AuthContext` (`auth.md` §5) — so the split is a deployment boundary,
not a code fork; endpoints could re-merge into one binary by mounting both routers
(the escape hatch if two deployables ever feel heavy pre-launch).

## 3. Shared infrastructure

**Sign-off:** _unsigned_

One Postgres, one Redis, one R2 bucket across all services (`architecture.md` §1). Queue
writes come from both gateways (admission inserts `queue_tasks`); only the worker consumes.
Request logging (`logging.md`) applies to both gateways with the same `request_logs` table
(`source` distinguishes traffic). Deployment: separate DO App Platform apps (or droplet
services) with independent instance counts; `/health` on each; the worker is the only
service with no ingress.
