# Competitive Landscape — AI model gateways, aggregators & inference clouds

Report, 2026-07-05. Verified against filings/press/company posts same day. Source-quality
tags: **[filing]** SEC/public, **[co]** company-claimed, **[press]** reputable press,
**[est]** analyst/tracker estimate (Sacra decent, GetLatka weak), **[rumor]** unconfirmed.

## 1. Master table (sorted by funding, descending)

| Company           | Predominant type                  | Sub-offerings                                   | Funding (latest)                          | ARR / run-rate                     | OSS?                          |
| ----------------- | ---------------------------------- | ------------------------------------------------ | ------------------------------------------ | ----------------------------------- | ------------------------------ |
| Nebius (NBIS)     | GPU neocloud (public)              | Token Factory serverless inference, fine-tuning   | ~$8.8B raised; mkt cap ~$54–61B            | $1.92B AI-cloud ARR [filing]        | Closed                         |
| Cerebras (CBRS)   | Custom-silicon inference (public)  | HW sales, $10B OpenAI compute deal                | >$4B private + $5.5B IPO (May 2026)        | $510M FY25, profitable [filing]     | Closed                         |
| Groq              | Custom-silicon inference           | GroqCloud, sovereign deals, LPU IP→Nvidia         | ~$3.4B; $650M Jun 2026                     | ">$500M" 2025 proj. [press]         | Closed                         |
| Lambda            | GPU neocloud                       | Inference API (killed 2025), clusters, HW         | ~$2.4B; $1.5B E @ ~$6B; IPO [rumor]        | ~$760M annualized [est]             | Closed                         |
| Baseten           | Serverless open-weights inference  | Dedicated deploys (Truss), training, speech       | ~$2.2B; **$1.5B @ $13B Jun 2026**          | ~$600M annualized [press/est]       | Truss OSS + closed cloud       |
| Together AI       | Neocloud (was serverless inference)| 200+ model APIs, fine-tune, FLUX, CodeSandbox     | ~$1.63B; **$800M @ $8.3B Jul 2026**        | ~$1B annualized [press/est]         | Closed + OSS research          |
| SambaNova         | Custom-silicon inference           | SambaCloud, on-prem, turnkey DCs                  | ~$1.5B; $350M @ $2.2B (57% down round)     | ~$100M [est, weak]                  | Closed                         |
| Vercel            | Frontend cloud + **AI Gateway**    | AI SDK (OSS), v0, zero-markup gateway             | ~$860M; $300M F @ $9.3B                    | ~$340M run-rate [est] (parent)      | Gateway closed; SDK OSS        |
| ElevenLabs        | Model lab w/ API (audio)           | TTS/STT/music/SFX, agents, dubbing, voice mkt     | $781M; $500M D @ $11B ($22B tender [rumor])| **~$500M ARR [co]**                 | Closed                         |
| Modal             | Serverless GPU compute             | Agent sandboxes (~$100M ARR alone), inference     | ~$465M; $355M C @ $4.65B                   | ~$300M annualized [est]             | Closed + OSS SDK               |
| Black Forest Labs | Model lab (FLUX)                   | Open weights + partner APIs                       | $450M; $300M B @ $3.25B                    | undisclosed                         | Open weights + closed API      |
| Hugging Face      | Model hub + inference aggregator   | Inference Providers (no markup), Endpoints        | ~$395M; last round 2023 @ $4.5B            | ~$130M 2024 [est]; profitable       | Core OSS                       |
| Kong              | API gateway middleware             | AI gateway plugins, semantic cache, MCP gov       | ~$345M; $175M E @ $2B                      | $100M+ ARR [press] (no AI breakout) | OSS core + enterprise          |
| **Fal**           | **Media-model aggregator**         | 600+ models, LoRA training, hosts Veo             | ~$340M; $140M D @ $4.5B ($8B talks [rumor])| **~$400M annualized [press]**       | Closed                         |
| Fireworks AI      | Serverless open-weights inference  | Fine-tune/RFT, dedicated, media APIs              | ~$327M; $250M C @ $4B ($15B talks [rumor]) | $280M [co] → ~$800M [est]           | Closed                         |
| SiliconFlow (CN)  | Serverless inference / MaaS        | DeepSeek/Qwen APIs, 10k corp clients              | ~$294M B Jun 2026                          | undisclosed                         | Closed                         |
| **OpenRouter**    | **LLM gateway/router**             | 400+ models, routing, video API, speech           | ~$153M; **$113M B @ $1.3B May 2026**       | ~$1B GMV; **~$50M take-rate [est]** | Closed                         |
| Higgsfield        | Media aggregator / consumer studio | Sora/Veo/Kling + own models                       | ~$138–188M; $80M @ $1.3B                   | ~$500M run-rate [co, credibility flagged] | Closed                   |
| DeepInfra         | Serverless open-weights (price floor)| GPU instances, media/speech                     | $133M; $107M B May 2026                    | undisclosed; "tripled 2026" [co]    | Closed                         |
| RunPod            | GPU cloud (dev-focused)            | Serverless endpoints, Hub, community cloud        | ~$120M; $100M A @ $1B Jun 2026             | **$240M ARR [co]** ($120M Jan 2026) | Closed + OSS workers           |
| Krea              | Media aggregator / creative canvas | 60+ models + own Krea 1/2, realtime               | $83M; $47M B @ ~$500M                      | ~$8M ARR Apr 2025 [press]           | Closed + open weights          |
| Poe (Quora)       | Consumer multi-model aggregator    | Developer API (2025), creator bots                | $75M (parent, 2024)                        | ~$65M subs [rumor]                  | Closed                         |
| Runware           | Serverless media inference         | Custom HW, day-zero model onboarding              | $66M; $50M A Dec 2025                      | undisclosed                         | Closed                         |
| Bria              | Licensed-data visual models        | Attribution/royalty engine                        | ~$65M                                      | undisclosed [co: +400%]             | Weights on HF (licensed)       |
| Replicate         | Serverless open-weights (50k models)| Media catalog, Cog, fine-tunes                   | ~$58M; **acquired by Cloudflare Nov 2025** | not broken out                      | Cog OSS + closed cloud         |
| Recraft           | Model lab (design imagery)         | Brand-controlled gen, API                         | ~$44M; $30M B                              | $5M+ at Series B [press]            | Closed                         |
| Parasail          | Serverless inference supercloud    | Cross-provider GPU arbitrage                      | $42M; $32M A Apr 2026                      | 500B tokens/day [co], no $          | Closed                         |
| Leonardo.ai       | Model lab / creative app           | Creative Engine API                               | ~$39M; **acquired by Canva 2024** (~$320M [press]) | consolidated                 | Closed                         |
| Featherless       | Flat-rate long-tail LLM inference  | 30k+ HF models, RWKV research                     | ~$25M; $20M A (AMD Ventures)               | ~$3–6M ARR [press]                  | Closed + OSS research          |
| Hyperbolic        | GPU marketplace + inference        | Decentralized/verifiable inference                | $20M                                       | undisclosed                         | Closed                         |
| Portkey           | AI gateway middleware              | OSS gateway, guardrails, governance               | ~$18M; **acquired by Palo Alto Jun 2026**  | ~$5M [est, weak]                    | OSS core (MIT) + enterprise    |
| Martian           | LLM router → interpretability      | Enterprise routing (Accenture)                    | $9M+ seed                                  | none public                         | Closed + OSS benchmarks        |
| Unify AI          | Router → AI agents (pivoted)       | legacy benchmarking                               | $8M seed                                   | none                                | Legacy OSS, dormant            |
| TensorZero        | OSS LLM gateway — **SHUT DOWN**    | Rust gateway + evals                              | $7.3M seed; wound down Jun 2026            | none                                | **Fully OSS** (archived)       |
| OpenPipe          | Fine-tune platform                 | ART RL toolkit                                    | $6.7M; **acquired by CoreWeave 2025**      | never disclosed                     | ART OSS                        |
| Eden AI           | Multi-API aggregator → gateway     | OCR/STT/vision aggregation, EU/GDPR               | ~$4.8M                                     | ~$2.3M [est, weak]                  | OSS connectors                 |
| Vast.ai           | GPU spot marketplace               | Serverless layer (2025)                           | ~$4M, bootstrapped                         | unknown                             | Closed + OSS CLI               |
| Requesty          | LLM gateway ("Cloudflare for AI")  | Routing, DLP/PII                                  | $3M seed                                   | >$1.5M ARR [co]                     | Closed + OSS SDKs              |
| LiteLLM (BerriAI) | **OSS** gateway middleware standard| 100+ providers, proxy, MCP gateway                | seed only (YC)                             | ~$0.5–1M [est]; 45k stars, ~95M dl/mo | **Fully OSS core (MIT)** + ent |
| Segmind           | Media aggregator ("gateway", 150+) | PixelFlow workflows, own distilled models         | ~$1.2M seed                                | unknown                             | Some OSS models                |
| Helicone          | Observability gateway              | Proxy logging, 100+ model gateway                 | ~$0.5M; **acquired by Mintlify Mar 2026**, maintenance mode | none public        | OSS core (Apache-2.0)          |
| Novita AI         | Serverless inference + media       | Media APIs, GPU cloud, sandboxes                  | undisclosed/bootstrapped                   | unknown (top OpenRouter provider)   | Closed                         |
| AtlasCloud        | Full-modal serverless inference    | GPU rental, SGLang co-dev                         | $0 — deliberately VC-free                  | $50M in ~4 months [co, single source] | Closed                       |
| Kie.ai            | Grey-market media reseller         | Sora/Veo/Suno resale 30–80% off                   | none on record; anonymous LLC              | unknown                             | Closed                         |
| PiAPI             | Unofficial/account-based media APIs| Host-your-account model                           | bootstrapped                               | unknown                             | Closed                         |
| WaveSpeedAI       | Media aggregator (1,000+ models)   | Even Novita resells its video                     | undisclosed                                | undisclosed                         | Closed                         |
| one-api / new-api | **OSS** self-host LLM gateways     | key redistribution, billing, MJ/Suno relay        | community (one-api stalled; new-api active, 41k stars) | none                   | **Fully OSS** (MIT / AGPL+com) |
| Cloudflare (NET)  | Edge cloud + AI Gateway            | Workers AI, **owns Replicate**, 5% unified billing| public; mkt cap ~$59–75B                   | $2.17B FY25 [filing] (gw not broken out) | Closed service            |
| Freepik/Magnific  | Prosumer creative aggregator       | Model-agnostic video/image + own tools            | bootstrapped/PE (EQT)                      | **$230M ARR [co]**, ~half video     | Closed                         |
| CoreWeave (CRWV)  | Hyperscale GPU cloud (context)     | owns OpenPipe, W&B, Marimo                        | public                                     | >$5B FY25; $99.4B backlog [filing]  | Closed                         |

## 2. Funding-only (descending, condensed)

| Rank | Company        | Total raised          | Rank | Company      | Total raised     |
| ---- | -------------- | --------------------- | ---- | ------------ | ----------------- |
| 1    | Nebius         | ~$8.8B (+public)      | 12   | Kong         | ~$345M            |
| 2    | Cerebras       | >$4B + $5.5B IPO      | 13   | Fal          | ~$340M            |
| 3    | Groq           | ~$3.4B                | 14   | Fireworks    | ~$327M            |
| 4    | Lambda         | ~$2.4B                | 15   | SiliconFlow  | ~$294M            |
| 5    | Baseten        | ~$2.2B                | 16   | Higgsfield   | ~$138–188M        |
| 6    | Together AI    | ~$1.63B               | 17   | OpenRouter   | ~$153M            |
| 7    | SambaNova      | ~$1.5B                | 18   | DeepInfra    | $133M             |
| 8    | Vercel         | ~$860M                | 19   | RunPod       | ~$120M            |
| 9    | ElevenLabs     | $781M                 | 20   | Krea         | $83M              |
| 10   | Modal          | ~$465M                | 21   | Runware      | $66M              |
| 11   | BFL            | $450M                 | 22   | Featherless  | ~$25M             |

(Sub-$25M and acquired/bootstrapped players in §1.)

## 3. Revenue-only (descending; quality noted)

| Company        | ARR / run-rate                 | As of      | Quality                        |
| -------------- | ------------------------------- | ---------- | ------------------------------- |
| CoreWeave      | >$5B FY25 revenue               | Q1 2026    | Filing                          |
| Nebius         | $1.92B AI-cloud ARR             | Q1 2026    | Filing                          |
| Together AI    | ~$1B annualized                 | Feb 2026   | Press/est                       |
| Fireworks      | ~$800M annualized ($280M [co])  | May 2026   | Est (co-claim was Oct 2025)     |
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
| **OpenRouter** | ~$50M take-rate (~$1B GMV)      | Mar 2026   | Est — GMV ≠ revenue             |
| AtlasCloud     | $50M in first ~4 months         | 2025       | Co-claimed, single source       |
| Krea           | ~$8M                            | Apr 2025   | Press                           |
| Featherless    | ~$3–6M                          | Apr 2026   | Press                           |

## 4. Open source status

**Fully open source products:** LiteLLM (MIT core, the de facto OSS gateway standard — 45k
stars, ~95M monthly downloads), new-api (AGPL fork that displaced the stalled one-api),
TensorZero (Apache-2.0 — **and it shut down 2026-06**, returning half its seed). **OSS core +
closed cloud:** Portkey (MIT gateway — acquired by Palo Alto), Helicone (Apache —acquired,
maintenance mode), Baseten (Truss), Replicate (Cog), Kong, Hugging Face, Eden AI.
**Everything at commercial scale is closed** — OpenRouter is not open source in the
slightest, nor are Fal, Fireworks, Together, ElevenLabs. Note the pattern: *pure-OSS gateway
companies died or exited* (TensorZero dead, Portkey/Helicone absorbed) while OSS-as-standard
LiteLLM thrives without meaningful revenue. Open source is a distribution strategy here, not
a business model — relevant to how we position our own OSS story.

## 5. Notable events (12 months)

- **Acquisitions:** Replicate→Cloudflare (Nov 2025), OpenPipe→CoreWeave, Portkey→Palo Alto,
  Helicone→Mintlify, Leonardo→Canva (~$320M [press]), NVIDIA's inference-layer roll-up
  (OctoAI ~$250M, Lepton AI) + the Groq "not-acqui-hire" (~$17–20B LPU license [press],
  hired away the CEO).
- **Deaths/pivots:** TensorZero shut down; one-api dormant; Banana.dev long gone; SambaNova
  57% down round after failed Intel sale; Together is now ~⅔ a GPU-cluster business;
  Lambda killed its per-token inference API.
- **Gateway margin compression:** Cloudflare (5% fee), Vercel (zero markup), and HF (no
  markup) all run gateways as loss-leader features — the context for every middleware exit.
- **Grey market:** Kie.ai/PiAPI resell upstream APIs with anonymous ownership; PiAPI already
  forced to sunset its Midjourney product.

## 6. What this means for us

1. **The revenue benchmark for our exact shape is Fal: ~$400M run-rate, $4.5B valuation,
   possibly re-pricing to $8B.** Media aggregation monetizes ~8x better than LLM gatewaying
   (OpenRouter: $1B GMV → ~$50M take). Our media-first + LLM-second weighting is pointed the
   right way.
2. **Pure LLM gateway economics are structurally bad** — hyperscalers give gateways away.
   Anything we build LLM-side must ride the media margin, not carry the business.
3. **Nobody in the top tier is open source.** An OSS gateway with a commercial cloud is
   *differentiation* against Fal/OpenRouter, but §4's graveyard says OSS is our distribution
   wedge, not the revenue plan.
4. **Consolidation is fast** (Cloudflare, NVIDIA, CoreWeave, Palo Alto all bought in within
   12 months). Being independent, neutral, and multi-media is scarcer every quarter — both
   an opportunity and an acquirer magnet.
