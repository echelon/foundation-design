# Product/API name ideas: the "router"-shaped word

Report, 2026-07-05. Premise: company = **Foundation**; we want a product/API name with the
instant-recognition quality of "router" — a word engineers already have a mental slot for.

## The word you might be remembering

Famous router-shaped names in engineering that you may have run across:

| Name          | What it is                                                        |
| ------------- | ------------------------------------------------------------------ |
| **Envoy**     | Lyft's L7 proxy — THE modern traffic-routing brand (taken, CNCF)    |
| **Zuul**      | Netflix's API gateway/router (their literal "router")               |
| **Relay**     | Facebook's GraphQL client (relay.dev) — but generic enough to reuse |
| **Conductor** | Netflix's orchestration engine                                      |
| **Traefik**   | The cloud-native reverse proxy                                      |
| **Istio/Linkerd** | Service meshes (routing layers)                                 |

## Candidates, ranked

| Name              | Why it works                                                       | Collisions / caveats                                  |
| ----------------- | ------------------------------------------------------------------- | ------------------------------------------------------ |
| **Relay**         | The strongest "router" synonym: a thing that receives and forwards. Instant mental model, short, verbs well ("relay your request to any model") | Facebook Relay (GraphQL) is big in frontend circles; different domain, but real |
| **Exchange**      | Telephone exchange — the original model-routing metaphor; also market exchange (fits an aggregator with prices) | Microsoft Exchange baggage; financial reading may confuse |
| **Switchboard**   | The human-scale router; warm, memorable, describes the omni/Any API perfectly | Long; slightly retro-twee                              |
| **Trunk**         | Trunk lines carry aggregated traffic between exchanges — deeply apt for an aggregator | trunk-based development; trunk.io (dev tools startup)  |
| **Spine**         | Leaf-spine network architecture; the backbone everything routes through; pairs weirdly well with "Foundation" (structural metaphors) | Less instantly obvious to non-network engineers        |
| **Crossbar**      | The crossbar switch — the literal hardware that connects any input to any output. Nerdy-perfect for "any model" | crossbar.io (WAMP router, small); obscure to younger engineers |
| **Broker**        | Message-broker mental model: accept, queue, deliver — which IS our job pipeline | Generic; financial-middleman reading                   |
| **Junction**      | Where all lines meet; clean, short                                   | Mild genericness                                       |
| **Interchange**   | Highway interchange — any road to any road                          | Long; EDI "interchange" baggage                        |
| **Patchbay**      | Studio hardware for routing any signal to any destination — extra apt for a *media* company | Audio-nerd specific; obscure outside music/AV          |
| **Grand Central** | Apple's Grand Central Dispatch + the station: everything routes through | Two words; Apple association; GCT trademark-ish        |
| **Dispatch**      | The verb of our dispatcher; ops-flavored                            | Very generic; many products                            |
| **Conduit**       | The pipe everything flows through                                    | Multiple OSS projects; plumbing-passive (routes don't *choose*) |
| **Manifold**      | Mechanical manifold: one intake, many outlets                        | Math reading; manifold.markets                         |
| **Loom**          | Weaves many threads into one fabric                                  | Loom (video) is a big brand — effectively taken        |

## Pairing with "Foundation"

The pattern that works: **Foundation** is the company; the routing product gets the short
word; the domain follows it (see `api-domain.md`):

- "**Relay** by Foundation" — `relay.foundation`? (the `.foundation` TLD exists) or `fdn.run`
- "Foundation **Exchange**" — reads like an institution (good for the Rankings product too)
- "Foundation **Switchboard**" — the friendliest docs voice

## Recommendation

1. **Relay** — if you want the word engineers map to "router" in under a second. The GraphQL
   collision is survivable (different category, and Meta has let it coast).
2. **Crossbar** — if you want precise-and-nerdy: the literal any-input-to-any-output switch;
   nobody serious owns the word in this space.
3. **Exchange** — if the Rankings/market-data identity becomes central to the brand; "the
   model exchange" is a story that writes itself.

Anti-recommendations: Envoy/Zuul/Conductor (taken in exactly our category), Gateway/Router
themselves (generic to invisibility), Loom (brand-squashed).
