# claude_pairing — working rules

This directory is the human/assistant pairing workspace for the Foundation design effort.

## Layout

| Path        | Purpose                                                                    |
| ----------- | -------------------------------------------------------------------------- |
| `design/`   | The living design decisions. These are the documents we execute against    |
| `reports/`  | One-off research answers and explainers. Immutable once delivered; superseded by new reports, not edited. **New reports land in `reports/` root**; bt files them into subfolders |
| `reports/read/` | Reports bt has processed. Immutable                                     |
| `reports/evergreen/` | Living reference reports (e.g. competitors) — the exception: these ARE updated in place on request |
| `reports/later/before-v1/`, `.../after-v1/` | Parked for revisit at that horizon             |
| `2026-07/`  | Dated historical drafts. Do not edit                                        |

## Sign-off protocol (design/ documents only)

Sign-offs are **per major heading** (every `##` section) PLUS one for the whole document at the
top. The top-of-doc block carries the instructions:

```
**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->
```

Each `##` section then has a bare one-liner directly under its heading (no instructions):

```
**Sign-off:** _unsigned_
```

Rules for assistants:

1. **Never** write bt's initials. Only bt signs.
2. On a **substantive** edit (changed decision, added/removed content, altered table), reset
   the sign-off of every touched section AND the top-of-doc sign-off to `_unsigned_`.
   Untouched sections keep their sign-offs — that's the point: bt re-reads only what changed.
3. Pure mechanical fixes (typos, whitespace, link repair, re-padding) reset nothing. When in
   doubt, reset.
4. When resetting, say so in the reply and name the un-signed sections.
5. New `design/` files and new sections start `_unsigned_`.
6. Reports in `reports/` carry no sign-off blocks.

## Formatting rules

- All markdown tables: pad cells so the `|` pipes align vertically. Never sparse tables.
- Keep table columns narrow (terse cells) so padding stays maintainable.

## Schema naming conventions

- **Foreign-key columns include the referenced table's name** (singular): `invited_by_user_id`,
  `accepted_by_user_id`, `created_by_user_id` — never bare role names like `invited_by` or
  `created_by`. Columns already of the form `{table}_id` (`account_id`, `user_id`, `job_id`,
  `payment_method_id`) are already correct. Applies to every design doc that declares tables.

## Naming

- "Foundation" is a placeholder product name; a real name is pending (see architecture.md open
  questions). Don't invent branding. Name-bearing strings so far: `Foundation-Version` header,
  `Foundation-Signature` header, `foundation_api_` key prefix, `foundation_whsec_` webhook
  secret prefix — all swap together when the name lands.
