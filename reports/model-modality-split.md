# Is "modality never in the model ID" actually universal?

Report, 2026-07-05. You suspected the rule isn't hard-and-fast. Verdict up front: **your
suspicion is correct at the weights level, but the rule survives at the catalog level with
two amendments we've already been applying implicitly.** Evidence below is from the verified
Fal/Replicate/provider catalogs gathered 2026-07-04/05.

## Where the unified-model view holds cleanly

Most modern flagship models really are one model with multiple tasks, and Fal models them as
endpoints under one ID:

| Model family        | Fal structure                                                        |
| ------------------- | ---------------------------------------------------------------------- |
| Kling (any tier)    | `fal-ai/kling-video/v3/pro/text-to-video` + `/image-to-video` — same model, sibling endpoints |
| Veo 3.1             | one family with `/image-to-video`, `/reference-to-video`, `/first-last-frame-to-video`, `/extend-video` |
| Seedance 2.0 (native API) | ONE task endpoint; modality expressed by the `content[]` array (text/image/video/audio roles) — the fully unified extreme |
| Sora 2              | `/text-to-video`, `/image-to-video`, `/video-to-video/remix`           |
| Nano Banana / GPT Image | generate + edit as endpoints of one model                          |

For these, our rule is exactly right: one ID, modalities as endpoints/inference.

## Hard case 1: one marketing name, genuinely different weights

Some families ship **separate checkpoints per task** under one brand:

| Family          | Reality                                                              | Fal's treatment                         |
| --------------- | ---------------------------------------------------------------------- | ---------------------------------------- |
| Wan 2.1         | `Wan2.1-T2V-14B` and `Wan2.1-I2V-14B-720P` are distinct open-weights checkpoints | Separate IDs: `fal-ai/wan-t2v`, `fal-ai/wan-i2v` |
| Hunyuan Video   | i2v is a separate checkpoint from t2v                                   | Separate endpoints (`hunyuan-video-image-to-video`) |
| CogVideoX       | t2v vs i2v checkpoints                                                  | Endpoints under one ID                    |
| SD-era inpainting | dedicated inpainting checkpoints                                      | Separate models historically             |

**Resolution — and why our design already absorbs this:** our catalog routes per
**(model, modality)**, not per model. `alibaba_wan_v2_2` with modalities `text_to_video,
image_to_video` simply maps each modality row to a different upstream artifact. The customer
sees one coherent product model; the routing table tells the truth about weights. No ID
change needed. This is a *feature* of the granular-API design: the modality is already in the
endpoint, so it doesn't need to be in the ID even when weights differ.

## Hard case 2: the task IS the product identity

Some "models" are productized tasks with their own names, weights, and pricing — there is no
unified parent to attach them to:

| Example                          | Why it's a separate model, not a modality                      |
| -------------------------------- | ---------------------------------------------------------------- |
| FLUX Fill / Kontext / Depth / Canny | BFL themselves name and version them as distinct models        |
| Grok Imagine Image vs Video      | xAI ships them as different models with different lifecycles     |
| Kling LipSync                    | Different model from Kling video gen, same brand umbrella        |
| ElevenLabs Scribe vs Eleven v3   | STT and TTS are obviously not one model                          |
| Ideogram Character               | A character-consistency model, not an Ideogram-v3 endpoint       |

Here the task word (fill, lipsync, scribe, character, image/video) legitimately appears in
the model NAME — because it's the **upstream's brand**, not a modality suffix we appended.
`bfl_flux_fill_pro` doesn't violate the rule; "Fill" is what BFL calls the model.

## The amended rule (recommendation)

> **We never append a modality suffix to an ID.** Beyond that, a decision procedure:
>
> 1. Same brand, same weights, parallel tasks → one model, tasks as modalities
>    (Kling v3, Veo 3.1, GPT Image).
> 2. Same brand, different weights per task, coherent product → STILL one model; the
>    (model, modality) routing table maps each task to its upstream artifact (Wan 2.1).
> 3. Upstream productizes the task as a distinctly named/priced/versioned model → separate
>    model, keeping the upstream's task-bearing name (FLUX Fill, Kling LipSync,
>    Grok Imagine Video).

Gray-zone tiebreaker when upstream branding is ambiguous: does it share the family's version
train and pricing? If yes → modality (Ideogram v3 remix/edit). If it versions independently →
model (Ideogram Character).

## What (if anything) to change in models.md

The catalog as written already follows the amended rule — the current §1 bullet just
overstates it ("NEVER ... those are different models and the media word stays" compresses
cases 2 and 3 into one sentence and skips case 2 entirely). Suggested edit once you agree:
replace that bullet with the three-case decision procedure above. One concrete row to
double-check under case 2: `alibaba_wan_vace_14b` (we made VACE its own model — correct,
Alibaba brands it separately) and `xai_grok_imagine_video_v1_5` listed with only
`image_to_video` (correct per xAI docs — v1.5 genuinely shipped i2v-only; a case-2 example
where modality sets differ across versions of one family, which the per-version modality
lists already handle).
