# Code models, tool calls, and agent harnesses

Report, 2026-07-05. Verified against OpenRouter docs/collections + vendor announcements
(figures from JS-rendered ranking pages are point-in-time approximations).

## 1. Does OpenRouter do mostly LLMs or code models?

**Coding IS OpenRouter's dominant workload now.** Programming went from ~11% of token volume
in early 2025 to **>50% of all tokens** (their own "State of AI" 100T-token study + press).
By the July 2026 collection snapshots: coding ≈ 17.9T tokens/period vs roleplay ≈ 4T — about
**4–4.5x the largest non-coding category**. The gateway business is, in practice, a coding-
traffic business.

Who dominates gateway coding (share of usage): Xiaomi MiMo V2.5 26%, MiniMax M3 13%, GLM 5.2
7%, Tencent Hy3 6%, DeepSeek V4 Flash/Pro ~10% combined, Claude Opus 4.8/4.7 ~8% combined —
i.e. cheap Chinese open-weight models own the top, Anthropic is the premium Western entrant.

## 2. Does OpenRouter support harnesses, agents, MCP?

| Capability          | OpenRouter status                                                          |
| ------------------- | ---------------------------------------------------------------------------- |
| Tool/function calls | Yes — OpenAI-compatible `tools`; can filter routing to tool-capable models    |
| Structured outputs  | Yes — JSON Schema response format                                             |
| Prompt caching      | Yes — passthrough of provider caching (auto + Anthropic `cache_control`)      |
| MCP                 | Partial — docs show converting MCP tool defs to OpenAI tools **client-side**; they also ship their own MCP server (docs/rankings/pricing/test-inference) for coding agents |
| Server-side tools   | Web search + web fetch executed server-side (Exa-powered) for any model; **no server-side code execution** |
| Agent hosting       | **No** — the new Agent SDK is client-side scaffolding (loops, Zod tools, state). No sandbox, no compute. It remains a completion API |

## 3. Who actually hosts agent execution

The execution layer lives elsewhere: OpenAI **Codex cloud** (repo-in-sandbox, tests, PRs),
**Devin** (fully managed engineer with a real REST API, billed in ACUs), Anthropic's Claude
Code / Agent SDK (local-first harness), and the sandbox-infra tier (**E2B** microVMs,
**Daytona**, **Modal** — the only one with in-sandbox GPUs, Vercel Sandbox). The single
inference provider bundling execution is **Together** (CodeSandbox acquisition → Code
Interpreter). **No gateway bundles execution.**

## 4. Code models are a distinct product category in 2026 — but not a distinct *media type*

Verified dedicated code SKUs: GPT-5.3-Codex ($1.75/$14 per M — sources conflict with $2/$10),
Kimi K2.7-Code (open weights, $0.95/$4, $19/mo CLI plans), Devstral 2 (open, free-for-now
API), Qwen3-Coder-Next (Apache-2.0, 1M ctx), Grok Build 0.1 ($1/$2, 8 parallel subagents).
The 2026 pattern: **coding agent CLI + captive code model + subscription seat**, priced as
separate SKUs with code-specific features (forced reasoning, ~10x cached-input discounts).

## 5. Recommendations for us

1. **Do NOT split code models from LLMs in the catalog.** They're the same media type
   (`text`), same `chat` modality, same wire shape. The split that matters is metadata: a
   `capabilities` set on each language model (`tool_calling`, `structured_outputs`,
   `prompt_caching`, `coding_tuned`, `reasoning`) driving docs filters and a `/models?capability=`
   query — mirroring OpenRouter's "collections" without new schema.
2. **Tool calls are table stakes, not optional.** Our chat pass-through must carry `tools`,
   `tool_choice`, structured outputs, and caching directives faithfully from day one —
   coding harnesses (the majority workload!) break without them. This belongs in the LLM
   adapter acceptance tests.
3. **A "coding" collection page** in docs (models tagged coding_tuned + comparison table)
   is cheap and matches where >50% of gateway demand actually is.
4. **Don't build execution.** Sandboxed agent hosting is a different business (E2B/Daytona/
   Modal exist; Together bought their way in). Our MCP server (design/mcp.md) is the right-
   sized agent play: let agents *call* us; don't run them.
5. Watch-item: OpenRouter's server-side web search tool is the one gateway-native tool
   pattern with traction — a candidate for our LLM plane later (server tool that works
   across all models), noted for the roadmap, not v1.
