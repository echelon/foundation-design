# claude_pairing — working rules

This directory is the human/assistant pairing workspace for the Foundation design effort.

## Layout

| Path        | Purpose                                                                    |
| ----------- | -------------------------------------------------------------------------- |
| `design/`   | The living design decisions (architecture, api, schema, models, enums, agents). These are the documents we execute against |
| `reports/`  | One-off research answers and explainers. Immutable once delivered; superseded by new reports, not edited |
| `2026-07/`  | Dated historical drafts. Do not edit                                        |

## Sign-off protocol (design/ documents only)

Every file in `design/` begins with a sign-off block directly under the H1:

```
**Sign-off:** _unsigned_
<!-- bt: replace "_unsigned_" with initials + date (e.g. "bt 2026-07-05") when approved.
     Assistants: reset to _unsigned_ on any substantive edit. See ../CLAUDE.md. -->
```

Rules for assistants:

1. **Never** write bt's initials. Only bt signs.
2. On any **substantive** edit to a signed file — anything that changes a decision, adds or
   removes content, or alters a table — reset the line to `**Sign-off:** _unsigned_` in the
   same edit, so bt knows to re-read.
3. Pure mechanical fixes (typos, whitespace, link repair, table re-padding) do **not** reset a
   sign-off. When in doubt, reset.
4. When resetting, mention it in your reply ("this un-signs schema.md") so re-review is
   explicit, and if practical say *which sections* changed so bt can re-read only those.
5. New `design/` files always start `_unsigned_`.

## Formatting rules

- All markdown tables: pad cells so the `|` pipes align vertically. Never sparse tables.
- Keep table columns narrow (terse cells) so padding stays maintainable.

## Naming

- "Foundation" is a placeholder product name; a real name is pending (see architecture.md open
  questions). Don't invent branding.
