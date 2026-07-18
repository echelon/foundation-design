# Environments

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). NEW: the environment model — development, staging,
production, test, plus white-label deployments. Siblings: `architecture.md` (stack/hosting),
`microservices.md` (deployables), `frontend_overview.md` (apps/subdomains), `testing.md`
(test-database safety), `conventions_database.md`.

---

## 1. The environments, first-class

**Sign-off:** bt 2026-07-18

Environments are **first-class and known to every frontend and backend**: each deployable
knows *where it is running* AND *where it targets* — injected at build/startup, never
inferred from hostnames at runtime. An `Environment` enum lives in `crates/core` (mirrored to
the frontend via the generated client), with the usual explicit serde renames (`enums.md`).

| Environment | Slug    | Domain placeholder   | Exists today?               |
| ----------- | ------- | -------------------- | --------------------------- |
| Development | `dev`   | `localhost:{port}`   | yes — every dev machine     |
| Staging     | `stage` | `examplestaging.com` | NO — anticipated, not built |
| Production  | `prod`  | `example.com`        | yes — the product           |
| Test        | `test`  | none (CI)            | yes — CI only               |

- Staging gets a **separate registrable domain** (`examplestaging.com`), not a subdomain of
  the production domain — clean cookie/CORS isolation, no crawler leakage, and the whole
  subdomain map of `frontend_overview.md` §2 replays on it (`dashboard.examplestaging.com`,
  `api.examplestaging.com`, …).
- **White-label** (customer domains/branding) is supported first-class but is NOT a fifth
  environment — it's a *deployment dimension* of production. See §6.

## 2. Development (`dev`)

**Sign-off:** _unsigned_

- **Frontend targeting is a startup configuration.** A frontend app started in local dev mode
  targets, per config: backend services on `localhost:{port}` (the default), OR staging, OR
  production. This is how you develop UI against real data without running the whole backend.
- **Backends are all-local:** every backend service talks to localhost Postgres, localhost
  Redis, and a development R2 bucket (or a local fake — e.g. MinIO/R2-emulator; which one is
  an implementation choice, both must work behind the same config seam).
- **Hard-coded ports.** Every service has a fixed, well-known localhost port — no dynamic
  allocation, no lookup: bookmarks, curl history, and REST-client collections stay stable.

| Service           | Port | Service    | Port |
| ----------------- | ---- | ---------- | ---- |
| api-gateway       | 4000 | www        | 3000 |
| dashboard-gateway | 4100 | dashboard  | 3100 |
| webhooks-inbound  | 4200 | playground | 3200 |
| worker (health)   | 4300 | docs       | 3300 |

  (Backends 4xxx, frontends 3xxx; gaps of 100 leave room for sidecars. Proposal — renumber
  freely.)

- **CORS:** frontends on `localhost:3xxx` calling backends on `localhost:4xxx` are
  cross-origin, and cookie-authed dashboard flows may require running a browser in
  CORS-permissive mode (or a browser profile with relaxed security) when targeting staging or
  production backends. Accepted dev-only friction; production never relaxes CORS.

## 3. Staging (`stage`) — anticipated, not yet built

**Sign-off:** _unsigned_

We will NOT start with staging; the design reserves its slot so adding it is configuration,
not redesign.

- **Auto-deploy:** CI/CD deploys every backend service to staging on merge — unless
  deployments are **locked** (a lock flag ops can set; deploys queue behind it).
- **Not publicly reachable:** staging domains resolve only on our private network (VPN/
  tailnet); no public web traffic, no crawlers, no customer accidents.
- Domain: `examplestaging.com` (placeholder, same swap-later rule as `example.com`).

## 4. Production (`prod`)

**Sign-off:** _unsigned_

Humans deploy production; machines only prepare.

- **Backends:** GitHub CI builds the artifacts; a human runs the deploy. No auto-deploy to
  prod, ever.
- **Frontends:** Netlify builds run automatically but the sites are **locked** — a human
  promotes a build to live. (Netlify's locked-deploys feature: builds accumulate, publish is
  manual.)

## 5. Test (`test`)

**Sign-off:** _unsigned_

- **`test` is the CI environment.** Network connections may be stubbed; a real Postgres may
  be spun up for fixture-backed integration tests (`testing.md` §3–§5).
- **Local test runs are NOT the `test` environment.** `cargo test` on a dev machine runs in
  the `dev` environment and hits a *test-named database inside the local instances* — a
  different Postgres database name (the `_test` suffix + marker-table safety scheme,
  `testing.md` §5) and a different Redis database index. Environment ≠ which database a test
  uses; the safety harness cares about the database name, not the environment slug.

## 6. White-label deployments

**Sign-off:** _unsigned_

We support **white-labeled** offerings: a customer's own domain, branding, and (as needed)
isolated environment, served by our stack. Modeled as a *deployment dimension*, not an
environment: a white-label deployment IS `prod` (or its own staging), parameterized by a
per-tenant config — domains, branding assets, feature surface.

**OPEN — bt to shape (2026-07-17):** the load-bearing decisions are unmade and recorded here
deliberately: shared multi-tenant infrastructure vs isolated-stack-per-customer (DB/R2/Redis
separation), where tenant config lives (code like the catalog, or ops config), custom-domain
TLS automation, and whether white-label frontends are the same Netlify targets with runtime
theming or per-tenant builds. Design when the first customer makes it real.
