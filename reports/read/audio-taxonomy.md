# The "audio" space: taxonomy for the gateway

Report, 2026-07-05. Verified against official docs (ElevenLabs, Suno, ByteDance/Volcano,
Stability, Google, MiniMax, Treblo, Mureka, Beatoven, Loudly, Kits, Deepgram, Hume,
AssemblyAI, FakeYou, Uberduck); uncertainty flagged. Bottom line first:

**"Audio" is not one media type operationally. It splits along three orthogonal axes:**

1. **Task** — what's generated (speech, music, sfx, transcript, converted voice…)
2. **Resource dependency** — does it need a persistent **voice** (a platform resource, not a
   parameter)?
3. **Lifecycle/transport** — job-queueable batch vs. websocket *session* (which cannot be a
   job at all)

Recommendation at the end: keep `audio` as the media type; split by task + lifecycle; make
VOICE a first-class resource.

---

## 1. ElevenLabs — a microcosm of the whole space

Their API surface alone spans every category we're debating:

| Product                  | Invocation                                             | Lesson                                   |
| ------------------------ | ------------------------------------------------------ | ----------------------------------------- |
| TTS                      | Sync HTTP + chunked streaming                          | Batch TTS fits our jobs fine              |
| Realtime TTS             | **WebSocket** (`stream-input`, multi-context)          | Confirmed: realtime = websockets          |
| Text to Dialogue (v3)    | Sync HTTP, ≤10 voices per request                      | Multi-speaker is a TTS sub-task           |
| **Voices**               | Full CRUD resource: premade / designed-from-prompt / instant clone (sync) / professional clone (**async training job** with poll state) / shared library | The voice is a RESOURCE with a lifecycle, referenced by `voice_id` everywhere |
| Agents                   | WebSocket **and** WebRTC (LiveKit), phone/SIP          | A stateful product, not a generation call |
| STT (Scribe)             | Sync, or async via `webhook=true`; realtime = WS       |                                           |
| Music                    | Sync HTTP; **composition plans** (`/v1/music/plan` — sectioned styles/lyrics); **inpaint/section-edit exists** (`store_for_inpainting`, Enterprise-gated) |                                           |
| SFX                      | Sync, 0.5–30s, `loop` flag                             | Thin schema, nothing like music's         |
| Dubbing                  | **Always async job** (create → poll → per-language audio) | Composite task (STT+translate+TTS)     |
| Voice Changer (STS)      | Sync + streaming — the only hosted *streaming* VC found |                                          |
| Audio isolation / forced alignment | Sync utilities                               |                                           |

## 2. Suno (what a music API wants to be)

v5.5 verified features: Personas (reusable vocal/style identity), **Voices** (clone your own
singing voice, ownership-verified), Custom Models (fine-tune on your catalog), Covers,
Extend, Remaster, **Replace Section (inpainting, 10–30s windows)**, stems up to 12 (Advanced
Split from ~100 instruments), ReMi lyrics model, instrumental, upload+mashup/sample, 8-min
generations. API status: confirmed "exploring," curated-partner intake (2026-07-01), nothing
launched.

A hypothetical Suno-shaped API is task-verbed and resource-backed — exactly our jobs model:
tasks `compose | cover | extend | replace_section | stems | remaster | lyrics`; resources
`clips, personas, voices, custom_models`; async jobs throughout (30s–3min gen times).

## 3. ByteDance Seed Audio

Real and big, but mostly a **speech** stack commercially: Volcano/BytePlus expose TTS 2.0
(websocket bidirectional + HTTP/SSE), voice cloning (train → persistent `S_xxx` speaker_id —
their voice-as-resource), streaming + file ASR, realtime S2S dialogue (WS), simultaneous
interpretation (WS). **Seed Audio 1.0** (`doubao-seed-audio-1-0`, ~2026-06) is the
interesting one: unified speech+music+SFX+dialogue from one prompt, voice cloning from
reference, ≤~2min, async. Seed-Music's note-level editing/SVC remain research-only; music
commercially ships via the SAMI product (async submit/poll/callback, CN-focused). Vendor
unification (one model doing speech+music+sfx) does NOT mean interface unification — the
param schemas stay disjoint.

## 4. Music APIs — the feature grammar

| Vendor                  | Extend | Inpaint/section | Ref-audio/style | Lyrics | Stems | Structure control      | Delivery                   |
| ----------------------- | ------ | ---------------- | ---------------- | ------ | ----- | ----------------------- | --------------------------- |
| Treblo (ex-**Sonauto**) | ✅     | ✅ (v2 only)     | ❌               | ✅     | ❌    | tags/length             | **Async + webhook**         |
| Stable Audio 2.5/3.0    | via A2A| ✅               | ✅ audio-to-audio| ❌     | ❌    | ❌                      | Sync (<2s)                  |
| Lyria 3 / RealTime      | ❌     | ❌               | ❌               | ✅ timed | ❌  | BPM/intensity/timestamps| Sync; RealTime = **WebSocket** |
| MiniMax music-2.6       | ❌     | ❌               | ✅ cover         | ✅     | ❌    | section tags            | Sync + stream               |
| ElevenLabs Music        | plan   | ✅ (Enterprise)  | ❌               | ✅     | ❌    | ✅ composition plan     | Sync + stream               |
| Mureka                  | ✅ both ends | ✅ region   | ✅ ref/vocal/melody ids | ✅ | ✅ | ❌                    | Async poll + stream_url     |
| Beatoven                | ❌     | ❌               | ✅ audio/video-to-music | ❌ | ✅ | prompt                 | Async poll                  |
| Loudly                  | ❌     | ❌               | ✅ remix         | ✅     | ✅    | ✅ structure_id/BPM/key | Sync                        |
| Udio                    | —      | —                | —                | —      | —     | —                       | **No API** (UMG walled garden) |

Notes: Sonauto **rebranded to Treblo** (2026-06, `api.treblo.com`); Stable Audio 3.0 does up
to 6-minute pieces. Your instinct was right: **extend / inpaint / style-reference are
modalities (task verbs), not separate models** — and they already exist across half the
market.

## 5. Voice conversion & the speaker-identity pattern

- **RVC** ecosystem: a "voice" = `.pth` weights + `.index` file, shared as zips; AI Hub
  Discord and voice-models.com (~28k models) alive, weights.gg dead; active successors
  Applio (training UX) and Plachtaa/seed-vc (zero-shot, NOT ByteDance). Hosted RVC exists on
  Replicate (takes a model-zip URL); no hosted realtime-WS VC found anywhere.
- **Kits.AI**: async job REST for conversion + voice blending, voices by trained model ID,
  licensed-artist voices with revenue share.
- **FakeYou** (yours — noted for pattern completeness): keyless catalog → `POST
  /tts/inference` with model token + idempotency UUID → poll job (`pending → started →
  complete_success | complete_failure | dead`); voices are typed Crockford tokens (`TM:`,
  `U:`, `WT:`). I.e., the gateway design we've been building — typed tokens, async jobs,
  idempotency — is the FakeYou pattern generalized.
- **Uberduck**: alive but gutted (rights-holder purge from ~5,000 community voices to a few
  hundred; now largely resells `polly_*`/`google_*` voices). The cautionary tale: unlicensed
  voice catalogs are legal glass. Suno's ownership-verified Voices is the licensed evolution.

## 6. Realtime transports (your websocket hunch: confirmed)

WebSocket is the universal server-side transport for anything realtime: ElevenLabs (TTS
stream-input, realtime STT, agents — plus WebRTC at the client edge), Cartesia (TTS + Ink
STT), OpenAI Realtime (WebRTC+WS+SIP), Gemini Live (WS), **Lyria RealTime (WS music
steering)**, Deepgram (STT/TTS/Voice Agent), Hume EVI, AssemblyAI streaming, ByteDance
(everything realtime). SSE appears only as a minor TTS-output option. **Sessions cannot be
job-queued** — they need token-minting (ephemeral keys) + a proxy/broker plane, which is a
different product from our job gateway.

## 7. Taxonomy recommendation

**Keep `audio` as the single media type.** Split by task (modalities), add a lifecycle
dimension, and introduce one new platform resource:

| Subcategory        | Verdict                                                                        |
| ------------------ | ------------------------------------------------------------------------------- |
| speech (tts)       | Modalities under audio: `text_to_speech`, `dialogue_to_speech` (multi-speaker). Depends on VOICE resource |
| **voice**          | **Not a task — a platform RESOURCE** (`voice_` prefix): created by design-from-prompt / instant-clone (sync) / professional-clone (async training job); listed, shared, deleted; referenced by voice_id from tts, dialogue, voice_conversion, and music (singing voice). Every serious vendor converges here |
| voice_conversion   | Modality under audio (`voice_conversion`, audio-in): async everywhere; references a VOICE |
| music              | Modality family under audio with its own verb grammar: `text_to_music`, `extend_music`, `inpaint_music` (section replace), `cover_music` (style transfer), `remaster`, `stem_split`, plus structured params (composition plan, lyrics, instrumental, reference audio, persona/voice) |
| sfx                | Stays its own modality (`text_to_sfx`) — thin schema (prompt, duration, loop); don't merge with music even though vendors' models do |
| stt                | Modality (`speech_to_text`); consumes rather than produces, but identical gateway shape |
| dubbing            | Composite modality (`dub_audio` / video) — always async; a natural fit for jobs; later |
| audio utilities    | Modalities: `audio_isolation`, `forced_alignment`, `stem_split` |
| **realtime**       | **Not a subcategory — a lifecycle value.** Catalog gains `lifecycle: sync \| async_job \| session`; `session` models (realtime TTS/STT, Lyria RealTime, S2S dialogue) are **out of the job gateway** — they'd need a session-broker plane (ephemeral tokens + WS proxy). Punt the plane; the catalog field costs nothing now |
| agents             | Out of scope entirely — a stateful product (agent CRUD + duplex sessions + telephony), not generation |

**Changes this implies if you sign off** (not yet applied anywhere):

1. `models.md` §11 gains the music verb modalities (`extend_music`, `inpaint_music`,
   `cover_music`, `remaster`, `stem_split`, `dialogue_to_speech`, `voice_conversion`,
   `audio_isolation`, `forced_alignment`) and audio model rows get corrected modality lists.
2. A future `voices` resource: `voice_` prefix, per-account, kinds `designed | instant_clone
   | professional_clone | licensed_library`, with async training states — schema sketch on
   request. v1 can launch without it (TTS with provider-preset voices only), but the moment
   we ship cloning, this is the shape.
3. Catalog entries gain a `lifecycle` field; `session` models are listed in the catalog but
   not invocable via `/v1/generate` (docs point to the future realtime plane).
4. Rankings/jobs need no changes — batch audio flows through the existing pipeline untouched.
