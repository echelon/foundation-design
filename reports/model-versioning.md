# Provider sub-version pinning ("snapshots")

Report, 2026-07-05. Some providers let you pin a model to an exact frozen build; we need that
expressible **inside our model identifier**.

---

## What the industry calls it

| Provider   | Their mechanism                                        | Their term                 |
| ---------- | ------------------------------------------------------- | -------------------------- |
| OpenAI     | `gpt-4o` (rolling alias) vs `gpt-4o-2024-08-06` (frozen) | **"model snapshot"**       |
| Anthropic  | `claude-sonnet-4-5` alias vs `claude-sonnet-4-5-20250929` | "model version" / snapshot |
| xAI        | `grok-2-image` vs `grok-2-image-1212` (MMDD builds); aliases `-latest` | version suffix |
| Google     | `gemini-2.5-pro` vs preview-dated builds (`-preview-05-06`) | "stable" vs dated preview |
| Replicate  | `owner/name` (latest) vs `owner/name:{64-hex}`           | "version" (immutable)      |

**Recommended term: "snapshot."** It says exactly the right thing — a frozen point-in-time
build of the same model — and matches the biggest vocabulary (OpenAI). "Version" stays
reserved for the *marketing* version already inside our IDs (`v3_pro`); overloading it would
be confusing ("version 3 pinned to version 2024-08-06"?).

## How it works in our identifiers

Rule: an optional trailing **date token** `_{yyyymmdd}` pins a snapshot. Charset stays
`[a-z0-9_]` — still double-click copyable. Unpinned = the provider's rolling/latest build
(what 95% of users want).

| Ours (unpinned, rolling)        | Ours (pinned snapshot)                    | Upstream equivalent            |
| ------------------------------- | ----------------------------------------- | ------------------------------ |
| `openai_gpt_v5_1`               | `openai_gpt_v5_1_20251113`                | `gpt-5.1-2025-11-13`           |
| `anthropic_claude_sonnet_v5`    | `anthropic_claude_sonnet_v5_20260210`     | `claude-sonnet-5-20260210`     |
| `xai_grok_imagine_video_v1_5`   | `xai_grok_imagine_video_v1_5_20260614`    | dated Grok build               |
| `google_veo_v3_1`               | `google_veo_v3_1_20260501`                | dated preview build            |

Mechanics:

- The catalog declares which snapshots exist per model and what upstream string each maps to.
  A pinned ID is a **first-class catalog entry** (same pricing machinery, own deprecation
  status) — not string parsing at request time.
- Docs list snapshots on the model page ("pin for reproducibility; unpinned tracks latest").
- An 8-digit trailing token is visually unambiguous as a date; the slug lint enforces it can
  only appear as the final token.
- Non-date upstream pins (Replicate's 64-hex) don't fit in the slug and don't need to: if we
  ever route through such a provider, the catalog maps our dated snapshot ID → their hash
  internally.
- When upstream retires a snapshot: catalog marks it `retired`, requests fail with
  `provider_deprecated`, docs show the migration target. The unpinned alias just moves.

## Not doing

- `@` / `:` separators (breaks double-click selection — the whole point of our charset).
- Auto-pinning every job to a snapshot (providers mostly don't expose one per request;
  we record `provider_model_id` used on the job row, which is the honest audit trail).
