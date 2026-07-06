# Competitive Landscape — AI model gateways, aggregators & inference clouds

Report, 2026-07-05 (deep-dives appended same day). Source-quality tags: **[filing]** SEC,
**[co]** company-claimed, **[press]** reputable press, **[est]** analyst estimate (Sacra
decent, GetLatka weak), **[rumor]** unconfirmed.

Relevance markings (bt, 2026-07-05): `TRUE` = direct competitor; `FALSE` = not relevant
(moved to the bottom, data retained); blank = unmarked pending bt. Featherless, Modal, and
RunPod moved to TRUE per bt (2026-07-05).

## 1. Master table

### Direct competitors (Relevant = TRUE)

| Rel? | Company        | Category                        | Sub-offerings                              | Funding (latest)                            | ARR / run-rate                       | OSS?                    |
| ---- | -------------- | -------------------------------- | ------------------------------------------- | -------------------------------------------- | ------------------------------------- | ------------------------ |
| TRUE | **Fal**        | Media-model aggregator           | 600+ models, LoRA training, hosts Veo       | ~$340M; $140M D @ $4.5B ($8B talks [rumor])  | **~$400M annualized [press]**          | Closed                   |
| TRUE | **Replicate**  | Serverless open-weights (50k models) | Media catalog, Cog, fine-tunes, Deployments | ~$58M; **acquired by Cloudflare Nov 2025** | not broken out                        | Cog OSS + closed cloud   |
| TRUE | **OpenRouter** | LLM gateway/router (archetype)   | 400+ models, routing, video API, Agent SDK, server web-search | ~$153M; $113M B @ $1.3B May 2026 | ~$1B GMV; **~$50M take-rate [est]**   | Closed                   |
| TRUE | **Modal**      | Serverless GPU compute           | Agent sandboxes (~$100M ARR alone)          | ~$465M; $355M C @ $4.65B                     | ~$300M annualized [est]                | Closed + OSS SDK         |
| TRUE | **RunPod**     | GPU cloud (dev-focused)          | Serverless endpoints, Hub, Public Endpoints (mini-Fal) | ~$120M; $100M A @ $1B Jun 2026    | $240M ARR [co]                         | Closed + OSS workers     |
| TRUE | **Featherless**| Flat-rate long-tail LLM inference| 30k+ HF models, RWKV research               | ~$25M; $20M A (AMD Ventures)                 | ~$3–6M ARR [press]                     | Closed + OSS research    |

### Unmarked (funding descending)

| Rel? | Company        | Category                          | Sub-offerings                               | Funding (latest)                       | ARR / run-rate                    | OSS?                                  |
| ---- | -------------- | ---------------------------------- | -------------------------------------------- | --------------------------------------- | ---------------------------------- | -------------------------------------- |
|      | Baseten        | Serverless open-weights inference  | Dedicated deploys, training, speech          | ~$2.2B; $1.5B @ $13B Jun 2026           | ~$600M annualized [press/est]      | **Proprietary core; MIT Truss client only** (§Deep-dives) |
|      | Together AI    | Neocloud (~⅔ GPU clusters now)     | 200+ model APIs, fine-tune, Code Interpreter | ~$1.63B; $800M C @ $8.3B Jul 2026       | ~$1B annualized [press/est]        | Closed + OSS research                  |
|      | SambaNova      | Custom-silicon inference           | SambaCloud, on-prem                          | ~$1.5B; $350M @ $2.2B (57% down round)  | ~$100M [est, weak]                 | Closed                                 |
|      | Vercel         | Frontend cloud + AI Gateway        | AI SDK (OSS), v0, zero-markup gateway        | ~$860M; $300M F @ $9.3B                 | ~$340M run-rate [est] (parent)     | Gateway closed; SDK OSS                |
|      | ElevenLabs     | Model lab w/ API (audio)           | TTS/STT/music/agents/dubbing                 | $781M; $500M D @ $11B ($22B tender [rumor]) | ~$500M ARR [co]                | Closed                                 |
|      | Black Forest Labs | Model lab (FLUX)                | Open weights + partner APIs                  | $450M; $300M B @ $3.25B                 | undisclosed                        | Open weights + closed API              |
|      | Hugging Face   | Model hub + inference aggregator   | Inference Providers (no markup), Endpoints   | ~$395M; last round 2023 @ $4.5B         | ~$130M 2024 [est]; profitable      | **OSS tooling; Hub platform + infra proprietary** (§Deep-dives) |
|      | Kong           | API gateway middleware             | AI Gateway plugins (semantic tier enterprise-gated) | ~$345M; $175M E @ $2B            | $100M+ ARR [press]                 | Apache-2.0 core; AI semantics enterprise (§Deep-dives) |
|      | Fireworks AI   | Serverless open-weights inference  | Fine-tune/RFT, dedicated, media APIs         | ~$327M; $250M C @ $4B ($15B talks [rumor]) | $280M [co] → ~$800M [est]      | Closed                                 |
|      | SiliconFlow (CN)| Serverless inference / MaaS       | DeepSeek/Qwen APIs                           | ~$294M B Jun 2026                       | undisclosed                        | Closed                                 |
|      | DeepInfra      | Serverless open-weights (price floor) | GPU instances, media/speech               | $133M; $107M B May 2026                 | undisclosed; "tripled 2026" [co]   | Closed                                 |
|      | GMI Cloud      | GPU cloud                          | Inference engine                             | ~$93M (mostly debt)                     | ~$7.4M [est, weak]                 | Closed                                 |
|      | Poe (Quora)    | Consumer multi-model aggregator    | Bot economy, creator monetization, dev API   | $75M (parent, 2024)                     | ~$65M subs [rumor]                 | Closed                                 |
|      | Runware        | Serverless media inference         | Custom HW, day-zero onboarding               | $66M; $50M A Dec 2025                   | undisclosed                        | Closed                                 |
|      | Bria           | Licensed-data visual models        | Attribution/royalty engine                   | ~$65M                                   | undisclosed [co: +400%]            | Weights on HF (licensed)               |
|      | Recraft        | Model lab (design imagery)         | Brand-controlled gen, API                    | ~$44M; $30M B                           | $5M+ at Series B [press]           | Closed                                 |
|      | Parasail       | Serverless inference supercloud    | Cross-provider GPU arbitrage                 | $42M; $32M A Apr 2026                   | 500B tokens/day [co]               | Closed                                 |
|      | Hyperbolic     | GPU marketplace + inference        | Verifiable inference, crypto layer           | $20M                                    | undisclosed                        | Closed                                 |
|      | Portkey        | AI gateway middleware              | OSS gateway, guardrails, governance          | ~$18M; **acquired by Palo Alto May 2026** | ~$5M [est, weak]                 | MIT repo — **silent since deal closed** (§Deep-dives) |
|      | Martian        | LLM router → interpretability      | Enterprise routing                           | $9M+ seed                               | none public                        | Closed + OSS benchmarks                |
|      | Unify AI       | Router → AI agents (pivoted)       | legacy benchmarking                          | $8M seed                                | none                               | Legacy OSS, dormant                    |
|      | TensorZero     | OSS LLM gateway — **SHUT DOWN Jun 2026** | Rust gateway + evals                   | $7.3M seed; ~half returned to investors | none                               | Fully OSS (Apache-2.0, archived) (§Deep-dives) |
|      | OpenPipe       | Fine-tune platform                 | ART RL toolkit                               | $6.7M; acquired by CoreWeave 2025       | never disclosed; legacy platform ends 2026-07-30 | ART OSS               |
|      | Eden AI        | EU AI gateway / multi-API aggregator | OCR/STT/vision/LLM, workflows, GDPR        | ~$4.8M (€3M seed Nov 2024)              | ~$2.3M [est, weak]                 | OSS connectors + closed cloud (§Deep-dives) |
|      | Vast.ai        | GPU spot marketplace               | Serverless layer (2025)                      | ~$4M, bootstrapped                      | unknown                            | Closed + OSS CLI                       |
|      | Requesty       | LLM gateway                        | Routing, DLP/PII                             | $3M seed                                | >$1.5M ARR [co]                    | Closed + OSS SDKs                      |
|      | LiteLLM (BerriAI) | OSS gateway middleware standard | SDK + proxy, virtual keys, budgets           | $1.6M seed (YC W23), no Series A        | **~$6M ARR [co, self-reported]**; 52.7k★ (§Deep-dives) | MIT + commercial `enterprise/` carve-out |
|      | Segmind        | Media aggregator (150+ models)     | PixelFlow workflows                          | ~$1.2M seed                             | unknown                            | Some OSS models                        |
|      | Helicone       | Observability gateway              | Proxy logging, model gateway                 | ~$0.5M; **acquired by Mintlify Mar 2026**, maintenance mode | none public    | Apache-2.0, still actively committed (§Deep-dives) |
|      | Novita AI      | Serverless inference + media       | Media APIs, GPU cloud                        | undisclosed/bootstrapped                | unknown (top OpenRouter provider)  | Closed                                 |
|      | AtlasCloud     | Full-modal serverless inference    | GPU rental, SGLang co-dev                    | $0 — deliberately VC-free               | $50M in ~4 months [co, single source] | Closed                              |
|      | Kie.ai         | Grey-market media reseller         | Sora/Veo/Suno resale                         | none on record; anonymous LLC           | unknown                            | Closed                                 |
|      | PiAPI          | Unofficial/account-based media APIs| Host-your-account                            | bootstrapped                            | unknown                            | Closed                                 |
|      | AI/ML API      | LLM + media API gateway (600+)     | OpenAI-compatible                            | undisclosed                             | undisclosed                        | Closed                                 |
|      | WaveSpeedAI    | Media aggregator (1,000+ models)   | Novita resells its video                     | undisclosed                             | undisclosed                        | Closed                                 |
|      | Chutes         | Decentralized inference (Bittensor)| Token-subsidized open-model hosting          | no VC; TAO emissions                    | contested                          | Runs open models                       |
|      | NanoGPT        | Consumer pay-per-prompt aggregator | Crypto payments, no-account                  | bootstrapped                            | undisclosed                        | Closed + OSS client                    |
|      | one-api / new-api | OSS self-host LLM gateways      | Billing, MJ/Suno relay                       | community projects                      | none public                        | MIT (stale) / AGPL+commercial (41k★, active) (§Deep-dives) |
|      | Cloudflare (NET)| Edge cloud + AI Gateway           | Workers AI, **owns Replicate**, 5% unified billing | public; mkt cap ~$59–75B          | $2.17B FY25 [filing] (parent)      | Closed service                         |
|      | Freepik/Magnific| Prosumer creative aggregator      | Model-agnostic video/image                   | bootstrapped/PE (EQT)                   | $230M ARR [co]                     | Closed                                 |
|      | CoreWeave (CRWV)| Hyperscale GPU cloud (context)    | owns OpenPipe, W&B, Marimo                   | public                                  | >$5B FY25 [filing]                 | Closed                                 |

### Not relevant (Relevant = FALSE — parked, data retained)

| Rel?  | Company    | Category                          | Sub-offerings                       | Funding (latest)                              | ARR / run-rate                     | OSS?    |
| ----- | ---------- | ---------------------------------- | ------------------------------------ | ----------------------------------------------- | ----------------------------------- | -------- |
| FALSE | Nebius     | GPU neocloud (public)              | Token Factory inference, fine-tuning | ~$8.8B raised; mkt cap ~$54–61B                 | $1.92B AI-cloud ARR [filing]        | Closed   |
| FALSE | Cerebras   | Custom-silicon inference (public)  | HW sales, OpenAI compute deal        | >$4B private + $5.5B IPO May 2026 (~$56B)       | $510M FY25, profitable [filing]     | Closed   |
| FALSE | Groq       | Custom-silicon inference           | GroqCloud, LPU IP→Nvidia             | ~$3.4B; $650M Jun 2026 (+~$17–20B Nvidia license [press]) | ">$500M" 2025 proj. [press]  | Closed   |
| FALSE | Lambda     | GPU neocloud                       | Clusters, HW; killed inference API   | ~$2.4B; $1.5B E @ ~$6B; IPO [rumor]             | ~$760M annualized [est]             | Closed   |
| FALSE | Higgsfield | Media aggregator / consumer studio | Sora/Veo/Kling + own models          | ~$138–188M; $80M @ $1.3B ($5B round [rumor])    | ~$500M run-rate [co, credibility flagged] | Closed |
| FALSE | Krea       | Media aggregator / creative canvas | 60+ models + own, realtime           | $83M; $47M B @ ~$500M                           | ~$8M ARR Apr 2025 [press]           | Closed + open weights |
| FALSE | Leonardo.ai| Model lab / creative app           | Creative Engine API                  | ~$39M; acquired by Canva 2024 (~$320M [press])  | consolidated                        | Closed   |

## 2. Funding-only (total raised, descending)

| Rank | Company        | Total raised          | Rank | Company      | Total raised     |
| ---- | -------------- | --------------------- | ---- | ------------ | ----------------- |
| 1    | Nebius         | ~$8.8B (+public)      | 13   | Fal          | ~$340M            |
| 2    | Cerebras       | >$4B + $5.5B IPO      | 14   | Fireworks    | ~$327M            |
| 3    | Groq           | ~$3.4B                | 15   | SiliconFlow  | ~$294M            |
| 4    | Lambda         | ~$2.4B                | 16   | Higgsfield   | ~$138–188M        |
| 5    | Baseten        | ~$2.2B                | 17   | OpenRouter   | ~$153M            |
| 6    | Together AI    | ~$1.63B               | 18   | DeepInfra    | $133M             |
| 7    | SambaNova      | ~$1.5B                | 19   | RunPod       | ~$120M            |
| 8    | Vercel         | ~$860M                | 20   | GMI Cloud    | ~$93M (mostly debt) |
| 9    | ElevenLabs     | $781M                 | 21   | Krea         | $83M              |
| 10   | Modal          | ~$465M                | 22   | Poe (parent) | $75M              |
| 11   | BFL            | $450M                 | 23   | Runware      | $66M              |
| 12   | Hugging Face   | ~$395M; Kong ~$345M   | 24   | Bria         | ~$65M             |

Sub-$60M and acquired/bootstrapped: Replicate ~$58M (pre-acq), Recraft $44M, Parasail $42M,
Leonardo ~$39M (pre-acq), Featherless $25M, Hyperbolic $20M, Portkey ~$18M (pre-acq),
Martian $9M, Unify $8M, TensorZero $7.3M (returned ~half), OpenPipe $6.7M (pre-acq),
Eden AI ~$4.8M, Vast ~$4M, Requesty $3M, LiteLLM $1.6M, Segmind ~$1.2M, Helicone ~$0.5M
(pre-acq); Novita / AtlasCloud / Kie / PiAPI / WaveSpeed / NanoGPT / Freepik = $0 or
undisclosed.

## 3. Revenue-only (known figures, descending; quality noted)

| Company        | ARR / run-rate                 | As of      | Quality                        |
| -------------- | ------------------------------- | ---------- | ------------------------------- |
| CoreWeave      | >$5B FY25 revenue               | Q1 2026    | Filing                          |
| Cloudflare     | $2.17B FY25 (parent)            | FY2025     | Filing                          |
| Nebius         | $1.92B AI-cloud ARR             | Q1 2026    | Filing                          |
| Together AI    | ~$1B annualized                 | Feb 2026   | Press/est                       |
| Fireworks      | ~$800M annualized ($280M [co])  | May 2026   | Est (co-claim Oct 2025)         |
| Lambda         | ~$760M annualized               | 2025       | Est (Sacra)                     |
| Baseten        | ~$600M annualized               | Mar 2026   | Press/est                       |
| Cerebras       | $510M FY25 (profitable)         | FY2025     | Filing                          |
| Higgsfield     | ~$500M run-rate                 | Jun 2026   | Co-claimed, credibility flagged |
| ElevenLabs     | ~$500M ARR                      | May 2026   | Co-claimed                      |
| Groq           | ">$500M" (cut from $2B proj.)   | 2025       | Press (projection)              |
| **Fal**        | **~$400M annualized**           | Mar 2026   | Press (The Information)         |
| Vercel         | ~$340M run-rate (parent)        | Mar 2026   | Est                             |
| Modal          | ~$300M annualized               | May 2026   | Est/press                       |
| RunPod         | $240M ARR                       | Jun 2026   | Co-claimed                      |
| Freepik        | $230M ARR                       | Apr 2026   | Co-claimed                      |
| Hugging Face   | ~$130M                          | 2024       | Est                             |
| Kong           | $100M+ (parent)                 | 2024       | Press                           |
| SambaNova      | ~$100M                          | mid-2025   | Est, weak                       |
| Poe (Quora)    | ~$65M subs                      | 2026 proj. | Rumor                           |
| **OpenRouter** | ~$50M take-rate (~$1B GMV)      | Mar 2026   | Est — GMV ≠ revenue             |
| AtlasCloud     | $50M in first ~4 months         | 2025       | Co-claimed, single source       |
| Krea           | ~$8M                            | Apr 2025   | Press                           |
| GMI Cloud      | ~$7.4M                          | 2025       | Est, weak                       |
| LiteLLM        | ~$6M ARR                        | ~late 2025 | Co-claimed (own job posting)    |
| Featherless    | ~$3–6M                          | Apr 2026   | Press                           |
| Eden AI        | ~$2.3M                          | Jul 2025   | Est, weak                       |
| Requesty       | >$1.5M ARR                      | Sep 2025   | Co-claimed                      |

## 4. Deep-dives (bt's follow-up questions, researched 2026-07-05)

### Baseten — is their offering open source? **No.**
Only **Truss** (MIT, ~1.2k★, actively maintained) is open — a model-packaging *client* whose
main deploy target is Baseten's cloud (`truss push` = onboarding ramp). The platform
(orchestration, autoscaling) and the inference runtime (custom kernels, speculative decoding)
are proprietary — Baseten *markets* the stack's proprietary-ness as its differentiator.
Master table corrected: "open-source-friendly," not an OSS company.

### Kong — what is it?
Founded as **Mashape** (2009, Milan → SF; Marietti + Palladino), an API marketplace whose
internal gateway was open-sourced in 2015 and ate the company (renamed Kong Inc. 2017;
marketplace sold to RapidAPI). **Kong Gateway is genuinely Apache-2.0** (~43.7k★, now
tagline'd "The API and AI Gateway"). The **AI Gateway** is plugins on that core, GA Feb 2024:
free tier = basic multi-LLM proxying + regex/template prompt controls (`ai-proxy` etc.);
**everything semantic is enterprise-gated** — semantic caching/routing/guarding, token-cost
rate limiting, PII sanitization, RAG injection, third-party guardrails — plus Konnect SaaS
charging **$100/month per LLM model**. $100M+ ARR, profitable, ~$345M raised @ $2B. Lesson:
a real OSS core with the AI value locked behind enterprise — the standard middleware
monetization shape.

### Hugging Face — entirely open source? **No, and the claim needs precision.**
Open: the tooling ecosystem (transformers, diffusers, TGI, TEI, candle, gradio — all
Apache-2.0/MIT) and the world's largest open-model repository. NOT open: the **Hub platform
itself** (private `moon-landing` repo), Inference Providers/Endpoints infrastructure, Spaces
orchestration, and all paid tiers — i.e., everywhere monetization lives. Also: hosted models
carry uploader licenses (Llama community, OpenRAIL, non-commercial) — "on HF" ≠ open source.
And precedent: **TGI was relicensed away from Apache-2.0 (HFOIL) for ~8.5 months in
2023–24** before reverting — the OSS commitment is strategic, not absolute. Master table
corrected.

### TensorZero — what happened?
Repo **archived June 12, 2026**; no formal postmortem — the founder (Gabriel Bianconi)
explained in the HN thread: *"An open-source company has to find product-market fit twice:
first for the OSS project and again for a commercial product… The market shifted under us."*
Orderly wind-down, not insolvency: ~2.5 years old, less than half the $7.3M seed spent, no
debt, **remaining capital returned to investors**. Code stays Apache-2.0/read-only; no
maintained fork has emerged. Context: hyperscalers shipping free gateways + ClickHouse
buying Langfuse squeezed seed-stage LLMOps middleware from both ends.

### Portkey & Helicone — what happened?
**Portkey → Palo Alto Networks** (announced Apr 30, closed May 29, 2026; price undisclosed,
"~$700M-class" per The New Stack [rumor]). Rationale: AI agents as an unmanaged attack
surface — Portkey becomes the AI Gateway inside Prisma AIRS (security product, not a neutral
gateway anymore). OSS risk flag: the MIT gateway repo has had **zero public commits since
the deal closed** and no OSS commitment from PANW exists.
**Helicone → Mintlify** (Mar 3, 2026; terms undisclosed). YC W23 founders joined Mintlify to
build "AI knowledge infrastructure"; Helicone cloud is in **maintenance mode** (security
fixes + new models only, no sunset date). Unlike Portkey, the Apache-2.0 repo is still
actively committed to (last push this week).

### LiteLLM (BerriAI) — successful?
**Yes — the one independent survivor of the gateway-middleware collapse.** YC W23 (Dholakia +
Jaffer); pivoted from a "ChatGPT on your data" product to LiteLLM in **July 2023**. It's two
things: the ubiquitous Python SDK (unified OpenAI-format across 100+ providers) and the
self-hosted proxy/gateway (virtual keys, budgets, spend tracking). **License: MIT with a
commercial `enterprise/` carve-out** (SSO, audit logs, RBAC — contact-sales pricing).
Traction: ~52.7k stars, ~2B cumulative PyPI downloads, Netflix/NASA/Adobe-class adopters
[co]. **Funding: $1.6M seed only, no Series A, ~10 people. Revenue: ~$6M ARR
[co, self-reported in their own YC job posting]** — deliberately lean and revenue-funded.
As a near-competitor: it's the self-host default for LLM routing (no media, no hosted
service, no billing rails) — more likely an integration target (people point LiteLLM at us)
than a head-to-head. **Disambiguation:** LightLLM (ModelTC) is an unrelated vLLM-class
inference engine.

### Eden AI — more
Lyon, France (launched Sep 2021; Zemmouri + Melaine). A unified API across **500+ models
spanning classic ML APIs** (OCR/document parsing, STT/TTS, translation, vision, moderation)
plus LLMs, with a node-based workflow builder. Differentiators: **EU sovereignty/GDPR**
(EU endpoints, European routing) and pricing = provider passthrough + **5.5% platform fee**
(same mechanic as OpenRouter). €4.5M total; ~$2.3M ARR [est, weak]; Datadog/Algolia/Docker
founders as angels. Closest thing to "us, but for classic ML APIs, in Europe."

### new-api / QuantumNous — more
The **AGPL-3.0 + commercial-license** fork of one-api (MIT, stale since Feb 2025) that now
owns the self-host gateway lineage: ~41.2k stars, 502 releases, very active. Features beyond
one-api: cross-format conversion (OpenAI ⇄ Claude ⇄ Gemini endpoints), multi-tenant
token/quota billing, **Stripe/EPay payments built in**, relays for Midjourney-Proxy, Suno,
Rerank, and OpenAI Realtime. Self-host only, no cloud edition; behind it is the QuantumNous
GitHub org (lead dev "Calcium-Ion"; whether it's a real company is unverified). Notable as:
(a) proof of demand for self-hosted gateway-with-billing — our OSS story's most direct
audience; (b) the resale-economy infrastructure (its MJ/Suno relays power the grey market).

### "Evals" — what that offering is
A product category (now converging with LLM observability): platforms that **score model
outputs** (LLM-as-judge, heuristics, human queues), maintain golden datasets from production
traces, run **regression tests when you change a prompt or migrate models**, and gate
releases in CI. Players: Braintrust ($36M a16z, Notion/Stripe/Vercel), LangSmith
(LangChain), Langfuse (OSS — **acquired by ClickHouse Jan 2026**), TensorZero's evals (dead
with the company). Buyers: AI platform teams preventing regressions across model swaps —
which is why gateways keep bundling eval-ish features. Adjacent to us (model-migration pain
is a gateway-adjacent problem), not core.

## 5. Open source status (corrected)

Fully OSS products: **LiteLLM** (MIT core + commercial carve-out — the standard),
**new-api** (AGPL+commercial, active), one-api (MIT, stale), TensorZero (Apache-2.0, dead).
OSS-core-with-gated-value: **Kong** (real Apache core; semantic AI features enterprise).
OSS-client-only: **Baseten** (Truss). OSS-tooling-not-platform: **Hugging Face**.
Everything at commercial scale — OpenRouter, Fal, Fireworks, Together, ElevenLabs — is
closed. Pattern, sharpened by the deep-dives: *pure-OSS gateway companies died or exited*
(TensorZero dead; Portkey and Helicone absorbed; Langfuse absorbed) while LiteLLM survives
on a $1.6M seed by staying tiny. Open source here is distribution, not a business model.

## 6. Notable events & takeaways (unchanged conclusions, strengthened evidence)

- **H1 2026 = the collapse of the independent LLM-middleware layer**: Langfuse→ClickHouse
  (Jan), Helicone→Mintlify (Mar), Portkey→PANW (May), TensorZero dead (Jun). LiteLLM is the
  lone independent, alive by frugality.
- The revenue benchmark for our shape remains **Fal (~$400M, $4.5B)** vs OpenRouter's ~$50M
  take-rate on $1B GMV — media aggregation monetizes ~8x better than LLM routing.
- Gateway margin compression from hyperscalers (Cloudflare 5%, Vercel/HF zero-markup) is
  structural; LLM-side revenue must ride the media margin.
- Consolidation buyers so far: Cloudflare, NVIDIA, CoreWeave, Palo Alto, Canva, Mintlify,
  ClickHouse — independence + neutrality + multi-media is getting scarcer each quarter.
