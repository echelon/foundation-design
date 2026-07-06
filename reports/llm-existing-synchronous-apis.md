# How synchronous LLM APIs actually work (transports)

Report, 2026-07-05. Verified against official docs + live ALPN probes of the API hosts.
Answers: plain HTTP? long-lived? Comet? websockets?

## The one-paragraph answer

Every major LLM platform's text API is **plain HTTPS request/response**: a POST whose
response is either held open until the full JSON is ready (non-streaming) or written
incrementally as **Server-Sent Events** on that same POST (streaming). Nobody uses
websockets for ordinary text completion — with exactly two first-party exceptions (below),
and **zero** exceptions among gateways/aggregators. Grok's websockets are its voice product,
not chat.

## Terminology decoder (your "Comet?" question)

"Comet" (2006) = the umbrella term for pre-standard server-push hacks: long-polling and
forever-frame streaming. **SSE** is the forever-streaming pattern standardized: one long
HTTP response, `text/event-stream`, line-framed `data:` events, `:` comment lines usable as
keep-alives. **WebSockets** (RFC 6455) are a genuinely different thing: an Upgrade handshake
to a full-duplex message channel. Over **HTTP/2**, "SSE" isn't even chunked encoding anymore
— just DATA frames arriving over time on one multiplexed stream. So: **LLM streaming is
Comet-style forever-streaming with SSE framing over h2** — not websockets, not long-polling.
(Quirk: it's not browser-`EventSource`-compatible since EventSource can't POST — everyone
parses SSE by hand over `fetch()`.)

## Per-provider findings

| Provider        | Non-streaming        | Streaming | Keep-alive during long gens                | WebSockets?                                    |
| --------------- | -------------------- | --------- | ------------------------------------------- | ----------------------------------------------- |
| **OpenRouter**  | Held-open POST       | SSE       | Comment lines `: OPENROUTER PROCESSING`     | **None anywhere in the product**                |
| OpenAI          | Held-open POST       | SSE (typed events on Responses) | None for sync; answer = background mode (poll or resume stream by cursor) | Realtime API (voice: WebRTC/WS/SIP) **+ the counterexample: Responses API WebSocket mode** — `wss://api.openai.com/v1/responses`, pure text/agentic, incremental inputs via `previous_response_id`, 60-min connection cap, ~40% faster for 20+-tool-call workflows |
| Anthropic       | Held-open POST (SDKs force streaming for large max_tokens) | SSE (typed: message_start → content_block_delta → …) | `ping` SSE events | None in the platform API (third-party claims of a WS audio endpoint appear false) |
| **xAI / Grok**  | Held-open POST       | SSE (documented verbatim) | token deltas       | **Yes — but it's the Grok Voice Agent API**: `wss://api.x.ai/v1/realtime?model=grok-voice-latest`, OpenAI-Realtime-spec-compatible, LiveKit/WebRTC option. Text chat is NOT websocket. (Consumer grok.com appears to stream NDJSON over HTTP — reverse-engineered, unofficial) |
| DeepSeek        | Held-open POST — **emits empty lines** to keep the TCP conn alive while queued | SSE | empty lines | None. Documented limits: conn closed if inference hasn't started in ~10 min or finished in ~30 min |
| Z.ai / GLM      | OpenAI-compatible POST (also speaks Anthropic protocol for coding plan) | SSE | —          | None found                                      |
| Together / Fireworks / HF router / Novita / DeepInfra / AtlasCloud | OpenAI-compatible POST | SSE (Fireworks docs verbatim: "data-only server-sent events … `data: [DONE]`") | token deltas | **None — no gateway/aggregator offers WS text, period** |

**HTTP version** (live ALPN probes): everyone negotiates **HTTP/2** except
`api.deepinfra.com` (HTTP/1.1). Nobody documents this.

**The only websocket TEXT APIs in existence** are first-party: OpenAI Responses WS mode and
Google Gemini Live (`BidiGenerateContent` — multimodal sessions that accept text). Everything
else websocket-flavored is voice (OpenAI Realtime, xAI Voice Agent, Gemini Live audio).

## The keep-alive playbook (what we'll need to copy)

Long generations die at proxies/LBs unless bytes flow. The field-proven mechanisms:

| Vendor      | Mechanism                                                          |
| ----------- | ------------------------------------------------------------------- |
| OpenRouter  | SSE comment lines every so often (`: OPENROUTER PROCESSING`)         |
| Anthropic   | Real `ping` SSE events                                               |
| DeepSeek    | Empty lines on non-streaming responses while queued                  |
| OpenAI      | Doesn't keep alive — offers background mode + resumable stream cursors instead |

Documented duration caps worth knowing: DeepSeek ~10 min to first token / ~30 min total;
OpenAI SDK client default 600s; OpenAI WS mode 60 min per connection.

## Implications for our stream plane (`sync-and-async-apis.md`)

1. **Our LLM plane = held-open POST + SSE, exactly as designed.** OpenRouter-identical; no
   websockets needed; no gateway anywhere disagrees.
2. **Adopt the keep-alive comment pattern** (OpenRouter-style `: PROCESSING` comments while
   the upstream is quiet) — it's the cheapest defense against LB timeouts and clients
   already tolerate it.
3. **Mid-stream error semantics need explicit design**: once we've sent 200 + first tokens,
   upstream failure must arrive as an SSE error event with `finish_reason: "error"` (and no
   silent provider failover mid-stream) — copy OpenRouter's documented behavior.
4. **Cancellation**: client aborts the connection → we cancel upstream where the provider
   supports it (OpenAI/Anthropic/Fireworks do; Google/Groq don't) and bill accordingly —
   provider capability flag in the catalog.
5. **Watch-item, not v1**: OpenAI's Responses WS mode (persistent connection, incremental
   turns) is the first real crack in "WS is voice-only." If it becomes the norm for agentic
   loops, our stream plane grows a WS listener someday — the plane split in
   `sync-and-async-apis.md` is exactly the seam it would slot into.
6. Set our LB/idle timeouts from the observed ceilings: ≥10 min to first token, ≥30 min
   total, matching the most generous upstreams we proxy.
