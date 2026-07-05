# Naming the "omni" API — a product, not an endpoint

Report, 2026-07-05. The thing being named: one API per media type where the model is a
parameter, the schema is the superset of every model's params, and irrelevant params are
pruned or coerced. Integrate once, swap model IDs forever. It deserves a name that survives a
decade of marketing, docs, SDK method names, and conference talks.

## Why "omni" is compromised

"Omni" now means **omni-modal models** in the industry: GPT-4o's "o" is literally "omni";
Google/DeepMind describe Gemini as omni-modal; "omni model" = one model that natively does
text+image+audio. If we call our API "Omni," every future sentence like "use the Omni API to
call an omni model" is mud. ArtCraft using `omni_gen` internally doesn't obligate the product
name. Retire it.

## The full exploration

### Descriptive names (say what it is)

| Name              | For                                                            | Against                                                       |
| ----------------- | --------------------------------------------------------------- | ------------------------------------------------------------- |
| **Unified API**   | The category's own generic term; instantly understood            | Zero distinctiveness — every gateway claims "unified API"; also we use "unified" for API shape #2, collision inside our own docs |
| **Universal API** | Bold, accurate promise                                           | Bland; "universal" is marketing wallpaper                     |
| **Superset API**  | Precisely true (superset schema); fresh in this space            | **Apache Superset** (huge BI tool) owns the word in dev minds |
| **Combo API**     | Your suggestion; honest                                          | Sounds like a value meal; undersells a flagship               |
| **Common API**    | Understated, accurate                                            | Forgettable; reads as "lowest common denominator" — the opposite of superset-of-params |
| **One API**       | "One integration, every model" falls out of the name              | "OneAPI" is Intel's oneAPI; near-invisible in search           |
| **Any API**       | Reads beautifully in code and prose: `/v1/any`, "generate with any model" | Slightly cute; "AnyAPI" genericism                     |
| **Meta API**      | Accurate (an API over APIs)                                      | Meta the company killed the word                              |

### Evocative names (brandable)

| Name             | For                                                             | Against                                                        |
| ---------------- | ---------------------------------------------------------------- | -------------------------------------------------------------- |
| **Rosetta**      | Perfect metaphor: one stone, every language — we translate one request shape to every model's dialect. Great writeup material | Apple's Rosetta (binary translation) is prominent; still, different domain and the metaphor is stronger than the collision |
| **Switchboard**  | Warm, mechanical-era charm: plug any call into any line          | Retro can read gimmicky; long word in code                     |
| **Polyglot**     | Speaks every model's language                                    | Programmer association is "polyglot persistence/repos"; adjective-y |
| **Babel**        | Tower of all languages                                           | babel.js owns it for JS devs; also connotes *confusion*        |
| **Prism**        | One input, full spectrum of models                                | Prisma ORM adjacency; several dev tools named Prism            |
| **Loom**         | Weaves many threads (models) into one fabric                     | Loom (video messaging) is a big brand                          |
| **Nexus**        | The connection point                                             | Sonatype Nexus; sci-fi generic                                 |
| **Keystone**     | The piece that holds the arch                                    | Keystone.js, OpenStack Keystone                                |
| **Omnibus**      | Winks at "omni" while being its own word ("for everything")      | Legal/legislative connotation; still omni-adjacent confusion   |
| **Chorus**       | Many voices, one performance                                     | Soft; doesn't say "API"                                        |
| **Grid**         | The power grid for models — plug in anywhere                     | Overused in infra                                              |
| **Conduit**      | The pipe everything flows through                                 | Plumbing connotation, several OSS projects                     |

### Structural observation that lowers the stakes

The **endpoint paths never carry the brand**: `/v1/generate/video` + `"model": "..."` is the
API regardless of what we call the product. The name lives in docs headings, marketing, and
SDK sugar (`client.rosetta.video(...)` vs `client.generate.video(...)`). So we can pick for
brand quality, and even rename later without breaking a single integration. That's deliberate
— keep it that way.

## Recommendation

1. **Rosetta** — my top pick as the *product* brand. The metaphor is exactly our mechanism
   (translation of one canonical request into every model's dialect — the `router` crate is
   literally a translation layer), it's memorable, writeup-friendly ("the Rosetta API for
   generative models"), and the Apple collision is in a different universe (and Apple's is
   fading from relevance).
2. **Any** — runner-up if you want descriptive-but-fresh: "the Any API", `generate with any
   model`, reads perfectly in every doc sentence and needs zero explanation.
3. **Switchboard** — runner-up if you want personality and don't mind retro.

Whichever wins: paths stay `/v1/generate/{media}`; "omni" gets purged from docs (and
eventually from internal names — `router` is already the honest internal term); the granular
and per-model shapes don't need product names (they're just "the model APIs").
