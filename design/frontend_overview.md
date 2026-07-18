# Frontend Overview

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). NEW: the index doc for every user-facing web
property — apps, deploy targets, subdomains, shared conventions. Breakouts:
`frontend_marketing_homepage.md`, `frontend_account_dashboard.md`, `frontend_playground.md`
(absorbed `model-pages.md`, 2026-07-17), `frontend_docs.md`. Code conventions:
`conventions_coding.md` §6; names: `conventions_naming.md` §7. Domain placeholder:
**`example.com`** — swaps with the product name when it lands (`architecture.md` §11).

---

## 1. The apps (one Nx workspace, four deploy targets)

**Sign-off:** _unsigned_

One Nx workspace (`frontend/`), TypeScript + React + Tailwind + Vite, hosted on Netlify
(`architecture.md` §1). Each app is an **independent Nx deploy target** — its own Netlify
site, its own deploy cadence, no shared runtime coupling.

| App          | Deploy target | Serves                                                         | Doc                              |
| ------------ | ------------- | -------------------------------------------------------------- | -------------------------------- |
| `www`        | Netlify site  | Marketing homepage, pricing, rankings, company pages           | `frontend_marketing_homepage.md` |
| `dashboard`  | Netlify site  | Account management console: keys, billing, jobs, logs, team    | `frontend_account_dashboard.md`  |
| `playground` | Netlify site  | Model pages: interactive playground + per-model docs + gallery | `frontend_playground.md`         |
| `docs`       | Netlify site  | Everything the playground doesn't cover: guides, API reference | `frontend_docs.md`               |

(REVISED 2026-07-17 — replaces the earlier three-app split `www / docs / console`
(`architecture.md` §5): `console` renamed `dashboard`, and the old `docs` app split into
`playground` (model pages) + `docs` (guides/reference).)

Shared code lives in Nx libs: `frontend/libs/api-client/` (generated — never hand-written),
`frontend/libs/ui/` (shared components), `frontend/libs/catalog/` (typed `catalog.json`
consumers). Apps depend on libs; apps never depend on apps.

## 2. USER-FACING SUBDOMAINS — the complete list

**Sign-off:** _unsigned_

Every hostname a user, customer, or integration partner can touch. This table is THE
registry; a subdomain missing here doesn't exist. All on the `example.com` placeholder.

| Subdomain               | What                                                          | Served by                               |
| ----------------------- | ------------------------------------------------------------- | --------------------------------------- |
| `example.com`           | Marketing homepage (apex; `www.` 301s here)                   | `www` app (Netlify)                     |
| `www.example.com`       | 301 → apex                                                    | Netlify redirect                        |
| `dashboard.example.com` | Account dashboard console                                     | `dashboard` app (Netlify)               |
| `example.com/models/*`  | Model pages + playground (apex path — RECOMMENDED, see below) | `playground` app (Netlify via proxy)    |
| `docs.example.com`      | Guides, API reference, non-model docs                         | `docs` app (Netlify)                    |
| `api.example.com`       | The product API (`/v1/*`, `/mcp`)                             | `api-gateway` (`microservices.md`)      |
| `hooks.example.com`     | Partner-facing inbound webhooks (`/stripe/webhook`, …)        | `webhooks-inbound` (`microservices.md`) |
| `cdn.example.com`       | Public media CDN: outputs, files, gallery samples (R2-backed) | Cloudflare R2 / CDN (`files.md`)        |
| `status.example.com`    | Status page (hosted status product; not our code)             | third-party                             |

- **Model pages on the APEX as a path, not a subdomain** — RECOMMENDED 2026-07-17, bt to
  confirm; pros/cons in `frontend_playground.md` §2. The `playground` app remains an
  independent deploy target; a CDN-layer proxy rewrite maps `example.com/models/*` onto it
  (deployability and URL shape are independent choices).
- `dashboard.example.com` is where the session cookie lives; cookie `Domain` is scoped to
  exactly this host (plus `api.example.com` for sandbox calls — `auth.md` §1 CSRF rules).
- The dashboard-gateway is NOT separately user-visible: the `dashboard` app talks to it via
  `dashboard.example.com/api/*` proxy rewrites (same-origin cookies, no CORS).
- `hooks.example.com` is partner-facing only (Stripe et al. configure delivery URLs there);
  no human traffic.
- No other subdomain may be minted without landing in this table first.

## 3. Shared frontend conventions

**Sign-off:** _unsigned_

The authoritative lists live in `conventions_coding.md` §6 (style/structure) and
`conventions_naming.md` §7 (names). Frontend-wide recap + what's decided at the app level:

- **Build:** Vite; Nx task graph for affected-only CI builds; Prettier + ESLint flat config.
- **API access:** every app uses the generated `api-client` lib (OpenAPI codegen from
  utoipa). No hand-written fetch wrappers, no hand-written DTOs; enums arrive as
  string-literal unions (`enums.md`).
- **Catalog data:** public apps (`www`, `playground`, `docs`) consume the build-time
  `catalog.json` artifact — never the live API — so public pages are fully static
  (`architecture.md` §2).
- **State:** local `useState` first, Context for cross-cutting (auth, theme); Zustand only on
  demonstrated need. Named exports only; `@/` → `src/`.
- **Auth:** only `dashboard` (and the playground's execute-path) ever sees a session; the
  public apps have zero auth code. Playground execution calls `api.example.com` with the
  dashboard session cookie (sandbox flow, `source = dashboard` — `architecture.md` §4).
- **Deploys are independent:** a marketing copy change never redeploys the dashboard; a
  catalog merge rebuilds `playground` + `docs` (and `www` pricing) via the catalog.json CI
  trigger, not the dashboard.
