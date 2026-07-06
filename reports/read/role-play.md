# Role-play & personality models

Report, 2026-07-05. Verified against OpenRouter collections, SillyTavern docs, provider
sites; JS-rendered share figures approximate.

## 1. The market shape

Roleplay is the **largest non-coding LLM category** (~4T tokens/period on OpenRouter — it was
the historical #1 before coding exploded past 50%). Yes, OpenRouter supports it — passively:
the traffic arrives because RP frontends teach users to BYOK through OpenRouter.

**Who dominates gateway RP:** DeepSeek ≈46% combined (V4 Flash 26%, V3.2 10%, V4 Pro 10%),
then Gemma 4 27B, Gemini 3 Flash. Cheap + permissive + long-context general models win by
volume; famous uncensored fine-tunes are long-tail by share. Grok is not in the top 5.

## 2. The ecosystem map

| Layer                | Players                                                                   |
| -------------------- | --------------------------------------------------------------------------- |
| Closed platforms     | Character.ai (**no public API** — app-only), Poe (real bot economy: prompt bots + server bots, per-message creator monetization via Stripe) |
| BYOK frontends       | **JanitorAI, Chub.ai** — their own help docs teach "plug in a proxy": DeepSeek direct, **OpenRouter** (most popular), Chutes. This is literally why DeepSeek tops OpenRouter's RP chart |
| Power-user frontend  | **SillyTavern** (self-hosted): connects to anything OpenAI-compatible + local backends (KoboldCpp, Ollama, TabbyAPI); character cards, lorebooks, vector memory |
| RP-courting providers| **Featherless** (flat-rate unlimited tokens over the 30k-model HF long tail — the RP-finetune home), Infermatic (flat-rate curated large models), **ArliAI** ("unrestricted, zero-log" — moderation posture IS the product), NovelAI (own storytelling models, vertically integrated), AI Horde (free crowdsourced, kudos queue) |
| Fine-tune ecosystem  | TheDrummer (Cydonia/Behemoth — has a branded OpenRouter page), Magnum/Anthracite, NousResearch — open weights on HF, run via Featherless/local GGUF/Horde |

## 3. Personality tech (your fine-tune / vector question)

Three approaches, layered in practice:

1. **Character cards** (prompt-time) — dominant by far. Portable PNG-embedded JSON, V2 spec
   (`chara_card_v2`) and V3 (`ccv3`: native lorebooks, assets). Zero training, model-agnostic.
2. **LoRA persona fine-tunes** — bake style into weights (the TheDrummer economy). Hosted
   tuning: Together (tune + serve custom checkpoints per-token), Fireworks SFT/LoRA
   (serverless-LoRA status currently contested), OpenPipe **sunsetting** (legacy platform
   ends 2026-07-30, migrating into W&B). No major "persona fine-tune API" product exists —
   it's generic LoRA infra used for personas.
3. **RAG/vector memory of a corpus** — SillyTavern's Data Bank + Vector Storage,
   vector-activated lorebooks (semantic triggering). Client-side today; no dominant hosted
   "persona memory" API.

So: "memorizing a vector of a personality" exists — as client-side RAG in the frontends, not
as a provider offering. The provider-side opportunity is **long-tail fine-tune hosting**
(Featherless's moat) and someday multi-LoRA persona serving.

## 4. What a gateway needs to win RP traffic (and whether we should)

Requirements synthesized from the evidence:

| Requirement                          | Our status                                                   |
| ------------------------------------ | ------------------------------------------------------------- |
| Long-tail fine-tune hosting          | Not at launch — this is exactly the future BYOM/compute lane (`other-service-offerings.md`); Featherless proved flat-rate + long tail is the mechanic |
| Permissive/zero-log tiers            | **Business/policy decision for you** — ArliAI/Infermatic sell moderation posture; our logs-as-product design is the *opposite* posture by default (we'd need a no-log mode per key). Flagging, not deciding |
| Full sampler surface                 | Pass-through `min_p`, `top_a`, `repetition_penalty`, `logit_bias` like OpenRouter — cheap, do it. **DRY/XTC samplers remain local-only industry-wide** — a real gap, only closable on self-hosted lanes where we control the inference stack |
| Text-completion (raw prompt) mode    | RP power users need instruct-template control; reserve `/v1/completions` alongside chat (OpenRouter keeps both) |
| Big cheap context + caching          | Comes free with our provider pass-through                     |
| Flat-rate pricing                    | Tension with our prepaid-USD metering; flat-rate is what wins *prosumer* RP. Defer — only relevant if we enter long-tail hosting |

**Recommendation:** don't chase RP as a launch vertical — the volume follows cheap permissive
models (which we'll carry anyway: DeepSeek, Gemma, GLM via LLM providers) and BYOK frontends
will find us if our chat API + sampler pass-through is solid. Concretely for v1: (1) sampler
pass-through parity with OpenRouter, (2) reserve `/v1/completions`, (3) carry the DeepSeek/
GLM/Gemma tier at good prices. The deeper RP plays — long-tail fine-tune hosting, persona
LoRA serving, zero-log tiers — all live in the BYOM/compute lane and the moderation-policy
decision, both already on the roadmap as their own decisions.
