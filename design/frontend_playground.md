# Frontend: Playground & Model Pages

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). The `playground` Nx app: one public page per model —
interactive playground, model-specific documentation, example outputs (gallery), pricing,
changelog. **ABSORBED `model-pages.md` (2026-07-17)** — bt asked whether model pages should
merge into the frontend docs; yes: everything in that draft was frontend, and the URL question
(§2) had to be answered here anyway. Siblings: `frontend_overview.md`, `frontend_docs.md`,
`models.md` (catalog authority), `agents_and_mcp.md` (llms.txt), `rankings.md`.

---

## 1. What a model page contains

**Sign-off:** _unsigned_

One page per model, sections (anchored, not separate URLs):

| Section       | Content                                                                                                                                                |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Header        | Display name, canonical ID (one-click copy), owner, media type, status badge (preview/active/deprecated + shutdown dates), price summary               |
| **Gallery**   | Dozens of curated samples per image/video model — output + full prompt + params, filterable by modality/style. The breadth showcase (§4)               |
| **Released**  | First-release date of the model (upstream's release, not our integration date)                                                                         |
| **Changelog** | Every change since release: upstream snapshot/behavior changes, our price changes, param additions, deprecation notices — dated entries (§5)           |
| Playground    | Sandbox form rendered from the modality JSON Schemas; cookie-auth; runs bill the account normally                                                      |
| API           | Per-modality param tables (from the same schemas), curl/TS/Python examples (generated — correct by construction), polling + webhook lifecycle snippets |
| Pricing       | Unit prices per modality, worked examples ("5s of 1080p ≈ $0.35")                                                                                      |
| Modalities    | Which tasks this model supports, with links to each granular endpoint's docs                                                                           |
| Related       | Other versions/snapshots of the family, alternatives in the media type                                                                                 |

**100% public. No login wall, ever** (Fal gating docs behind auth is an own goal — docs are
top-of-funnel). Playground *execution* requires an account; *reading* everything requires
nothing.

## 2. URL shape: apex path, not a subdomain (RECOMMENDED 2026-07-17 — bt to confirm)

**Sign-off:** _unsigned_

bt's question: does putting model pages on `playground.example.com/models/{slug}` impinge on
Google traffic vs Fal's approach (`fal.ai/models/fal-ai/nano-banana/edit` — apex paths)?

**Recommendation: apex paths — `example.com/models/{model_id}` — with the `playground` app
still its own deploy target, mounted via a CDN proxy rewrite.** The deploy-target question and
the URL question are independent: Netlify proxy rewrites let the apex site route `/models/*`
to the separately-deployed playground site. We get every engineering pro bt listed AND the
SEO of apex paths. Fal's choice is the market-tested one.

**Subdomain (`playground.example.com/models/...`):**

| Pros                                                     | Cons                                                                                                                                                                                                                           |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Independently deployable target with zero routing config | **SEO: search engines treat subdomains as quasi-separate sites** — domain authority earned by the apex (backlinks, brand queries) does not flow reliably to a subdomain, and model pages are exactly our programmatic-SEO play |
| Trivially static/prerendered (codegen from catalog.json) | Every model-page backlink builds the *subdomain's* authority, not the apex's — the two halves of the site stop compounding                                                                                                     |
| Clean separation for engineers                           | Internal links homepage ↔ model pages count as cross-host, weakening crawl-equity flow                                                                                                                                         |
| No path reservations needed on the apex                  | Less canonical-looking URLs to share; extra host for users to remember; separate Search Console property, split analytics                                                                                                      |

**Apex path (`example.com/models/...`), proxy-mounted:**

| Pros                                                                                            | Cons                                                                                                                                  |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| One authority pool: homepage strength lifts model pages, model-page backlinks lift the homepage | Needs a proxy/rewrite layer (Netlify rewrite from the apex site → playground site) — one more config surface, care with cache headers |
| Still an independent Nx deploy target (the proxy decouples URL shape from deployability)        | Apex path namespace must be reserved (`/models/*` can never belong to `www`)                                                          |
| Still fully static/prerendered — the proxy serves prerendered pages verbatim                    | Apex cookies are visible to the playground app (benign: public apps carry no auth code; execution auths against `api.example.com`)    |
| Matches Fal (and OpenAI, Replicate) — the pattern proven in exactly our market                  |                                                                                                                                       |
| One sitemap/Search Console property; social cards and canonical tags all on the brand domain    |                                                                                                                                       |

The "accessibility" concern bt raised cuts the same way: one memorable domain, paths that
read like content (`example.com/models/kling_video_v3_pro`), no host-hopping while browsing
from marketing → model page → docs.

## 3. URLs

**Sign-off:** _unsigned_

(REVISED 2026-07-17 — model pages move from the old `docs.example.com` plan to the apex, per
§2. `docs.example.com` keeps the non-model docs — `frontend_docs.md`.)

| URL                                      | What                                                                                        |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| `example.com/models`                     | Catalog index: filter by media type, modality, owner, price; comparison view                |
| `example.com/models/{model_id}`          | THE model page (all sections above, anchored: `#gallery`, `#api`, `#pricing`, `#changelog`) |
| `example.com/models/{model_id}/llms.txt` | Agent-docs sibling (`agents_and_mcp.md`)                                                    |
| `example.com/generate/{media}`           | The Any-API per-media guides with the cross-model comparison matrix                         |

One page per model (not per modality — modalities are sections/tabs within it), matching the
decision that modalities are tabs, not page sprawl. Canonical URLs use the canonical slug;
alias slugs 301-redirect.

## 4. The gallery (the differentiator)

**Sign-off:** _unsigned_

- Target: **24–48 curated samples per image/video model**, deliberately spanning the model's
  breadth — styles, subjects, aspect ratios, each modality (t2v AND i2v with the source image
  shown), strengths and honest characteristics.
- Every sample shows: output media, the **exact prompt**, the full params JSON (copyable into
  the playground with one click — "remix this"), and generation date + snapshot.
- **Provenance: samples are generated through our own API** (real jobs, internal account), so
  every sample is reproducible and honest. Sample assets live in R2 under a dedicated prefix
  (`samples/{model_id}/{sample_id}.*`), served via `cdn.example.com`
  (`frontend_overview.md` §2), marked never-cold (they're the marketing).
- Authored as a **sample manifest per model in the catalog crate** (code, like everything
  else): prompt, params, asset key, tags, sort weight. The playground build renders manifests
  → gallery. Adding a sample = generate → curate → commit manifest entry (an admin-cli
  `gen-samples` helper batch-runs a prompt list against a model and emits draft manifest
  entries for curation).
- Video samples: poster frame + lazy-loaded muted looping preview; full-quality on click.

## 5. Release dates & changelogs

**Sign-off:** _unsigned_

Two authored fields in each catalog entry (code, versioned, reviewable):

- `released_at: 2026-03-26` — the model's first public release upstream.
- `changelog: [...]` — dated entries with a `kind`:
  `upstream_release | upstream_change | snapshot_added | price_change | params_change |
  deprecation | retirement`. Our own events (price changes, new snapshots) can be
  auto-appended by the codegen diffing catalog history (git does the bookkeeping);
  upstream behavior changes are authored by hand as we learn of them.

Rendered as a timeline on the page. This is genuinely uncovered ground — nobody in the space
answers "what changed about this model since I integrated it?" — and it compounds with the
Rankings product (changelog events annotated on usage trend charts: "price dropped here").

## 6. Rendering & SEO

**Sign-off:** _unsigned_

- The playground app **statically generates** all model pages at build time from
  `catalog.json` + sample manifests — no client-side-only rendering for content (Fal's
  JS-rendered pricing is invisible to crawlers; we watched our own research agents fail to
  read it). The playground *widget* hydrates client-side on top of the static page.
- Netlify CDN; sitemap.xml enumerating every model page; canonical tags; per-page OpenGraph
  images drawn from the gallery (the sample IS the social card); JSON-LD structured data
  (Product with price, plus VideoObject/ImageObject for samples).
- Page titles: "{Display Name} API — {media} generation" (the query people actually search).
- Rebuild trigger: catalog merge → CI regenerates catalog.json/manifests → playground deploy.
  Model pages are always exactly as current as the deployed catalog.

## 7. Open questions (draft — for your pass)

**Sign-off:** _unsigned_

1. Sample curation bar: who approves gallery entries? (Proposal: manifest PRs reviewed like
   code; taste is a merge gate.)
2. Do deprecated/retired models keep public pages? (Proposal: yes, marked, with migration
   pointers — SEO and changelog history keep value.)
3. Per-modality sample minimums for non-visual media (audio players inline? mesh/splat
   viewers — three.js glTF/splat viewer embed?).
4. "Remix this sample" deep-link into the playground with params pre-filled — v1 or later?
5. §2's apex-path recommendation — bt to confirm (the `frontend_overview.md` §2 subdomain
   table currently assumes it).
