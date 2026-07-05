# RunPod, Featherless, and the adjacent service landscape

Report, 2026-07-05. Verified against official docs/pricing pages same day; press-sourced and
uncertain items flagged inline. Purpose: evaluate RunPod + Featherless, map the space, and
position our future "bring your own model, we spin it up and serve it through the gateway"
offering.

---

## 1. RunPod — evaluation

**What it is:** per-second-billed GPU cloud with four tiers that matter to us:

| Product           | Shape                                                                      |
| ----------------- | ---------------------------------------------------------------------------- |
| Pods              | Raw GPU containers. On-demand $/GPU/hr: H200 $4.39, H100 SXM $3.29, H100 PCIe $2.89, A100 $1.39–1.49, RTX 4090 $0.69. Secure Cloud (SLA) vs Community (cheaper, no SLA); spot up to ~60% off (5s SIGTERM) |
| Serverless        | Autoscaling endpoints, min=0 → scale-to-zero, FlashBoot cold starts (sub-200ms best case claimed), execution timeout up to 7 days, up to 3 GPU types in priority order. Flex $/hr: H200 $5.93, H100 $4.55, A100 $2.72 |
| Instant Clusters  | 2–8 nodes (16–64 GPUs) self-serve H100/H200/B200, up to 512 via sales; managed Slurm |
| Public Endpoints  | Their own pay-per-output model APIs (Flux, Kling, Sora 2…) — they're becoming a mini-Fal themselves |

**The part that matters for us:** everything is programmable via REST
(`https://rest.runpod.io/v1/` — OpenAPI-spec'd). `POST /v1/endpoints` with a **vLLM or SGLang
worker template** + `gpuTypeIds` + `workersMin/Max` yields an **OpenAI-compatible URL**
(`api.runpod.ai/v2/{endpoint}/openai/v1`) in one call. Weights pre-staged on network volumes
($0.07/GB/mo) via their **S3-compatible API**, so cold start ≈ disk-load time.

**How it slots into our roadmap** ("customer brings a fine-tune, we serve it"):
one RunPod endpoint per customer model (or shared multi-LoRA servers — see §4), created by our
control plane, proxied through our gateway with our auth/billing/logs. Watch-outs: no
reseller/sub-account billing (we meter and rebill from one prepaid account), per-DC GPU
availability (mitigate with multi-GPU-priority + multi-DC), serverless quotas negotiated via
sales, spot/community prices partially unverified.

## 2. Featherless — what it actually is

**Genuinely different from both OpenRouter and Fal.** It's not an aggregator: Featherless
**runs its own GPU fleet** (AMD-backed — $20M Series A co-led by AMD Ventures, ROCm
partnership) with proprietary orchestration that hot-loads models from the Hugging Face long
tail on demand. The product bet is *breadth-per-GPU*:

| Dimension     | Featherless                                            | OpenRouter                      | Fal                               |
| ------------- | ------------------------------------------------------- | -------------------------------- | ---------------------------------- |
| Runs GPUs?    | Yes — own fleet, dynamic model placement                 | No — pure routing/billing layer  | Yes — curated media endpoints      |
| Catalog       | ~40,000 HF text models (auto-onboards any public model with 100+ downloads) | ~340 curated LLMs across 60+ providers | ~1,000 curated media models |
| Pricing       | **Flat-rate subscription** ($10–$200/mo), limits by **concurrency slots**, "unlimited tokens"; credit per-request option exists | Per-token passthrough + 5.5% deposit fee | Per-output |
| Media?        | No — text only                                           | Inputs yes; gen via new video API | Core business                     |

Corrections to common beliefs (verified in their docs): **no LoRA serving** — adapters must be
merged to full weights and published to HF (safetensors, fp16, supported architectures only);
so "any HF model" has real constraints. No evidence of any acquisition.

**Lesson for us:** the flat-rate/concurrency pricing model is how you make the long tail
economical — you can't per-token-price a model with three users. If we ever serve long-tail
open weights, we either copy that mechanic or don't compete there.

## 3. The adjacent landscape (one line each; details verified 2026-07-05)

| Player            | Category & differentiation                                                  |
| ----------------- | ----------------------------------------------------------------------------- |
| Together AI       | Serverless per-token + dedicated endpoints ($6.49/hr H100) + fine-tuning; one API across tiers |
| Fireworks AI      | Speed-focused serverless + dedicated ($7/hr H100); killer feature: **LoRA fine-tunes serve at base-model token rates** |
| Baseten           | Deepest BYO-model story (Truss packaging), per-minute dedicated autoscale w/ scale-to-zero |
| Modal             | Serverless Python GPU functions (H100 ≈ $3.95/hr eq.); bring code, no catalog |
| Replicate         | 50k-model catalog + Cog BYOM + private Deployments; **being acquired by Cloudflare** (announced 2025-11) — platforms are converging on our shape |
| DeepInfra         | Cheapest per-token tier + one-call `POST /deploy/llm` for any HF repo on dedicated GPUs (H100 $1.79/hr) |
| Novita AI         | The breadth cousin: LLM + media APIs + GPU instances + dedicated endpoints **with LoRA adapter support** |
| Nebius Token Factory | Enterprise-grade per-token + dedicated + managed fine-tuning on their own hyperscale cloud |
| Groq / Cerebras / SambaNova | Custom-silicon speed plays; BYO weights gated behind enterprise sales at all three (Groq–Nvidia and SambaNova funding drama press-sourced) |
| Predibase         | WAS the self-serve LoRA-serving leader (LoRAX) — **acquired by Rubrik, self-serve withdrawn; LoRAX remains Apache-2.0** |
| OpenPipe          | Traffic-capture → distill → fine-tune loop — **acquired by CoreWeave** |
| Hugging Face      | Two products: Inference Providers (`router.huggingface.co` — an OpenRouter analogue, pass-through pricing, 17 partners) and Inference Endpoints (managed dedicated deploy of any Hub model, real API) |
| Vast.ai           | Decentralized GPU marketplace (H100 from ~$2/hr, interruptible cheaper) + new serverless layer; cheapest raw compute, weakest guarantees |
| Lambda            | Training-scale GPU cloud; **killed its per-token inference API** (Sept 2025) |
| Beam / Cerebrium  | Small serverless-GPU-functions players (H100 ~$1.74–3.40/hr, fast cold starts); Beam's stack is open-source (beta9) |
| fal (compute side)| Beyond media endpoints: fal Serverless (BYO container) + fal Compute (dedicated H100 $3.99 list / $1.89 floor) + managed Flux LoRA training |

## 4. Market map and where we fit

Categories: **(a)** gateways/aggregators (OpenRouter, HF router) · **(b)** serverless
open-weights inference (Together, Fireworks, DeepInfra, Featherless, …) · **(c)** serverless
GPU functions (Modal, Beam, RunPod Serverless) · **(d)** managed dedicated deployments
(Baseten, HF Endpoints, DeepInfra Custom, Replicate Deployments) · **(e)** raw GPU
marketplaces (Vast, RunPod Pods, Lambda) · **(f)** fine-tune/LoRA hosting (Fireworks,
Together; Predibase/OpenPipe just vacated).

**The empty quadrant is (a)+(d):** no *gateway* today offers "bring your own fine-tune or
open-weights model and we host it behind the same gateway key, billing, and logs." OpenRouter
is BYO-*keys* only; HF's router serves partner catalogs only. Every substitute forces the
customer into a second vendor relationship they must stitch together themselves. And **media +
LLM under one neutral gateway is already rare** (only Novita and Replicate credibly span both,
neither as a neutral aggregator) — which is us. Adding BYOM hosting to that has no direct
competitor today.

**Recommended future posture** (roadmap, not v1): RunPod Serverless as primary spin-up backend
(REST-created vLLM/SGLang endpoints, S3-staged weights); DeepInfra Custom LLMs / HF Inference
Endpoints as second source; Fireworks/Together dedicated as the premium SLA tier; LoRAX-style
multi-adapter serving so small customer fine-tunes share GPUs instead of each paying for an
idle H100. Our differentiation is the thing we already build for the catalog: one key, one
ledger, one debug timeline — extended to the customer's private models.

**Timing caveat:** Cloudflare buying Replicate signals big platforms converging on exactly
"catalog + BYOM behind one edge API." The gap is real but probably not open for years.

## 5. Fit into our current design (what to reserve now)

Nothing structural — the seams already exist: a customer model = a catalog entry whose
`Provider` is a compute backend (`runpod`) and whose route carries the endpoint id; jobs,
billing (per-second price units exist), logging, and Rankings (private models excluded)
all apply unchanged. The only new nouns later are `custom_models` (the resource owning
weights-as-files — anticipated in `files-naming.md`) and provider adapters for the compute
backends. No schema reservations needed today.
