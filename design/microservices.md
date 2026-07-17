# Microservices (deployable topology)

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). REVISES `architecture.md` §5's earlier
"one api-server" decision: there are now **three web ingress services** (api-gateway,
dashboard-gateway, webhooks-inbound) plus the worker.
Siblings: `architecture.md`, `api_overview.md`, `auth.md`, `payments.md`.

---

## 1. The services

**Sign-off:** _unsigned_

| Service               | Crate                   | Serves                                                                                                                                                        | Auth accepted                                                                 |
| --------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| **api-gateway**       | `bin/api-gateway`       | The product we sell: `/v1/any/*`, `/v1/models/*` generate/cost, `/v1/chat/*`, `/v1/jobs/*`, `/v1/files/*`, `/v1/models` catalog reads, `/mcp`, `/v1/public/*` | API key, dashboard session cookie (sandbox), MCP                              |
| **dashboard-gateway** | `bin/dashboard-gateway` | The console's backend: `/v1/auth/*`, account/members/invitations, billing console, spend views, audit logs, log explorer, session management                  | Session cookie only                                                           |
| **webhooks-inbound**  | `bin/webhooks-inbound`  | Third-party inbound webhooks from our integration partners: `/stripe/webhook`, `/webhooks/{partner}` — verify signature, dedupe, enqueue, return `200` fast   | Per-source signature (Stripe-Signature; partner HMAC) — no API key, no cookie |
| **worker**            | `bin/worker`            | Queue consumer: dispatch/poll/ingest/webhooks/rollups/sweepers                                                                                                | none (internal)                                                               |
| **admin-cli**         | `bin/admin-cli`         | Migrations, ops one-offs                                                                                                                                      | operator                                                                      |

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
- **Scaling shape:** api-gateway scales with customer traffic (held-open LLM streams);
  dashboard-gateway scales with humans-in-consoles; webhooks-inbound scales with partner
  delivery bursts — three different curves, three different instance counts.
- **Versioning:** the api-gateway speaks the date-versioned public contract (`api_overview.md` §1);
  dashboard endpoints are internal, unversioned, free to churn with the frontend;
  webhooks-inbound endpoints are contracts owned by each *partner*, versioned on their terms.
- **Partner ingress isolation (webhooks-inbound):** inbound webhooks authenticate by
  **per-source signature** (Stripe-Signature; a partner's HMAC), never our API keys or session
  cookies — a wholly separate auth surface, kept off both the paid API and the console. Partner
  traffic is also bursty and retry-heavy (providers hammer on any 5xx), so isolating it means a
  webhook storm can't degrade customer generation. **Direction matters:** this service is
  *inbound* (partners → us); our *outbound* webhooks (us → customers) are delivered by the
  worker's `deliver_webhook` task and signed per-account (`api_overview.md` §4) — different direction,
  different service, different signing.

All three ingress binaries are thin over the SAME crates — routers from `core/api-defs`, logic
in `services/*`, one `AuthContext` (`auth.md` §5) — so each split is a deployment boundary,
not a code fork; endpoints could re-merge by mounting the routers together (the escape hatch if
the separate deployables ever feel heavy pre-launch).

## 3. Shared infrastructure

**Sign-off:** _unsigned_

One Postgres, one Redis, one R2 bucket across all services (`architecture.md` §1). Queue
writes come from all three ingress services (admission inserts `queue_tasks`; webhooks-inbound
inserts `process_stripe_event` and partner-callback tasks — `payments.md` §5); only the worker
consumes. Request logging (`logging.md`) applies to every gateway with the same `request_logs`
table (`source` distinguishes traffic). Deployment: separate DO App Platform apps (or droplet
services) with independent instance counts; `/health` on each; the worker is the only
service with no ingress.
