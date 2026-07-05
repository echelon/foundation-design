# Enum practice review: is the ArtCraft style good? What would I change?

Report, 2026-07-05. Honest commentary on the pattern we're adapting
(`crates/schema/public/enums` in ArtCraft) and what I'd do differently.

---

## What ArtCraft does well (keep all of this)

1. **Explicit `#[serde(rename)]` on every variant.** The wire string is a contract; making it
   visible at the definition site (instead of trusting `rename_all` transforms) means renames
   are deliberate diffs, greppable, and reviewable. Keep.
2. **Mechanical test batteries** — round-trip, explicit expected-string assertions, charset
   lint, variant counts. These catch the exact bug class that matters (accidental wire
   breakage) at near-zero cost. Keep, and generate them with a macro so they can't be
   forgotten.
3. **Slug-only charset discipline** enforced by tests. Keep.
4. **Copy, Eq, Hash derives** — enums as cheap value types. Keep.

## Where ArtCraft's pattern creaks (avoid these)

1. **Parallel mirror enums with hand-written O(n) conversions.** `CommonImageModel` →
   `CommonModelType` (and `RouterImageModel` mirroring both) means every new model touches
   3+ files with mechanical match arms — the doc comment even says "Mirrors
   artcraft_router::api::router_image_model". That's the definition of a maintenance tax, and
   the compiler only saves you if every match is exhaustive-without-wildcard. **Our design
   already avoids the worst of it** (one enum per media type + one `Model` umbrella, no
   Common/Router split), but the temptation will recur at the API-defs/router boundary —
   resist it: reuse the core enums across layers instead of mirroring them per layer.
2. **Deprecated variants linger in the wire enum** (`GrokVideo` with a `#[deprecated]`
   attribute). Serde still accepts them, so they're not really deprecated — they're aliases.
   Our alias map is the right home for retired slugs; the enum should hold only current
   canon.
3. **Metadata via scattered impl methods** (`get_model_class()`, per-enum `to_*` conversions)
   rather than one registry. Fine at 20 variants, painful at 200. Our catalog-as-code design
   fixes this: the enum is identity only; ALL metadata (display name, media type, provider,
   prices, modalities) lives in the catalog registry keyed by the enum.

## The alternative designs, weighed

| Design                              | Verdict                                                                        |
| ----------------------------------- | ------------------------------------------------------------------------------ |
| Plain enums + hand impls (ArtCraft) | Good at small scale; conversion/mirror tax grows linearly                       |
| **Declarative macro table** (recommended) | One `models! { ... }` invocation per media type declaring slug + variant; the macro emits the enum, serde renames, `FromStr` (alias-aware), slug tests, and the catalog-registration stub. New model = one line + one catalog module. Still plain-enum ergonomics after expansion, greppable output via `cargo expand` |
| Proc-macro / derive framework       | More power than we need; slower builds, harder debugging — declarative macro covers it |
| `strum` crate                       | Use it for the boring derives (`EnumIter`, `Display`) instead of hand-rolling; it does NOT solve the mirror-enum or metadata problem |
| String newtype + runtime registry (`ModelId(String)`) | Maximum flexibility, zero compile-time safety — a typo'd model in a match arm compiles fine. Wrong for us: exhaustive matches over models/providers are exactly where the compiler earns its keep (adapter dispatch must cover every provider) |
| Codegen from a data file (TOML/JSON → build.rs) | Tempting "single source of truth", but it moves truth OUT of Rust into a data file — recreating the config-not-code smell you rejected for the DB. The macro table keeps truth in Rust |

## Recommendation

Keep the ArtCraft *discipline* (explicit renames, test batteries, slug charset), drop the
ArtCraft *topology* (mirror enums, in-enum deprecated variants, scattered metadata impls).
Concretely:

1. One `models!`-style declarative macro per media type generating enum + serde + tests.
2. `strum` for iteration/display derives.
3. Metadata exclusively in the catalog registry; enums stay identity-only.
4. Retired slugs move to the alias map; enums hold only live canon.
5. A compile-time global-uniqueness test across all media-type enums (slugs share one
   namespace because the `Model` umbrella serializes bare).

This is a refinement of what `design/enums.md` already specifies, not a change of direction —
the doc's "Style" section should eventually reference the macro once it exists.
