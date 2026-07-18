# Frontend: Marketing Homepage

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). The `www` Nx app on the apex (`example.com`) —
marketing materials, pricing, rankings, company pages. Fully static, no auth code.
Siblings: `frontend_overview.md`, `frontend_playground.md` (which owns `example.com/models/*`
via proxy — that path prefix is reserved and never served by this app), `rankings.md`.

---

## 1. Pages

**Sign-off:** _unsigned_

| Route                            | Page                                                                                                                               |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `/`                              | Homepage: value proposition, live-feel demo reel (gallery samples via `cdn.example.com`), model breadth, CTA → playground + signup |
| `/pricing`                       | Pricing: prepaid dollar model (NO "credits" vocabulary — `payments.md`), per-model price table rendered from `catalog.json`        |
| `/rankings`                      | Public model-usage rankings (`rankings.md`; data: `/v1/public/rankings`)                                                           |
| `/company`                       | About / contact                                                                                                                    |
| `/legal/terms`, `/legal/privacy` | Legal pages                                                                                                                        |
| `/blog/...`                      | Announcements / model launches (each new catalog model is a launch post opportunity)                                               |

Reserved on the apex, served by OTHER targets via proxy (`frontend_overview.md` §2):
`/models/*` and `/generate/*` (playground), `/llms.txt` (agents.md redirect).

## 2. Conventions

**Sign-off:** _unsigned_

- Static build from `catalog.json` + rankings snapshot; zero client-side data fetching above
  the fold (the rankings page may hydrate live data below a static shell).
- Sample media reused from the playground gallery manifests — the marketing IS the product
  output (`frontend_playground.md` §4); served from `cdn.example.com`.
- SEO plumbing shared with the playground: sitemap, canonical tags, OpenGraph, JSON-LD.
- Copy rule: dollars, never credits (`payments.md` vocabulary rule).
