# Frontend: Account Dashboard

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-17). The `dashboard` Nx app — account management,
billing, keys, jobs, logs (including webhook debug logs), team. Backend:
`dashboard-gateway` (`microservices.md`), reached via same-origin `/api/*` proxy rewrites;
sandbox generation goes to `api.example.com` (`architecture.md` §4). Auth: session cookie
(`auth.md`). Siblings: `frontend_overview.md`, `accounts.md` §6 (endpoint list),
`api_misc.md`, `logging.md` §6, `audit_logs.md` §4, `payments.md` §7.

---

## 1. Domain & URL model

**Sign-off:** _unsigned_

**Domain: `dashboard.example.com`** (placeholder apex — `frontend_overview.md` §2).

Two URL scopes, matching the auth model (`auth.md` §4):

- **User scope (`/user/...`, no account in the URL)** — things that belong to the human:
  profile, login methods, sessions, the user's own audit trail.
- **Account scope (`/{acct_id}/...`, Stripe-style)** — everything else. Every request
  authorizes as (session user, URL account); switching accounts is switching URLs. The
  account switcher rewrites the `{acct_id}` segment and keeps the rest of the path.

`/` resolves: not logged in → `/login`; logged in → last-used account's overview (or the
account picker when there is none). During impersonation (`auth.md` §6) a persistent banner
renders on every page.

## 2. Pages & routes — user scope

**Sign-off:** _unsigned_

| Route             | Page                                                                                                                                                                                                                                                            |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/login`          | Email/password + SSO buttons (Google, GitHub); rate-limit + "signup-as-login" behavior per `auth.md` §2                                                                                                                                                         |
| `/signup`         | Same form family; SSO-first layout                                                                                                                                                                                                                              |
| `/reset-password` | Token flow (`accounts.md` §2 password_resets); on consume: the "log out other devices?" ASK (`auth.md` §3)                                                                                                                                                      |
| `/invite/{code}`  | Invitation resolve/join page — code or magic-link token (`accounts.md` §4)                                                                                                                                                                                      |
| `/user/profile`   | Display name; avatar (upload/crop → `user_avatar_files`, gravatar dismiss/restore, letter/generated fallback preview — `accounts.md` §2)                                                                                                                        |
| `/user/security`  | Password set/change; login methods list (password entry + SSO connections, connect/disconnect — `accounts.md` §5); **sessions list** (device, IP, UA, "this device", per-row kill + "log out all" — `auth.md` §3); user audit trail (`GET /v1/user/audit_logs`) |
| `/user/accounts`  | Account picker: memberships + roles; "New account" (creator = owner)                                                                                                                                                                                            |

## 3. Pages & routes — account scope (`/{acct_id}/...`)

**Sign-off:** _unsigned_

| Route                        | Page                                                                                                                                                                                              | Min role |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| `/{acct_id}`                 | Overview: balance + top-up CTA, spend sparkline, recent jobs, failing-webhook and auto-refill-disabled alerts, getting-started checklist                                                          | member   |
| `/{acct_id}/jobs`            | Job list: status/model/media_type/date/end_user filters, cost per row (`GET /v1/jobs`)                                                                                                            | member   |
| `/{acct_id}/jobs/{job_id}`   | Job detail: params, outputs preview, charge/refund, and the **full timeline** — request log → provider calls → webhook deliveries (`logging.md` §4)                                               | member   |
| `/{acct_id}/files`           | Files browser: uploads + generated outputs, CDN links, storage class (`files.md`)                                                                                                                 | member   |
| `/{acct_id}/logs`            | Request-log explorer, Stripe-style: filter by path/status/api key/date; row → full redacted request/response (`logging.md` §6)                                                                    | member   |
| `/{acct_id}/api-keys`        | Keys list (prefix, title, last-used, expiry); create (secret shown once); edit description; revoke (`api_misc.md` §6)                                                                             | admin    |
| `/{acct_id}/webhooks`        | Endpoint config, secret view/rotate (two active during rotation), and the **webhook debug log**: every delivery attempt + customer response + retry state (`webhook_deliveries`, `logging.md` §3) | admin    |
| `/{acct_id}/billing`         | Balance + top-up (Stripe Checkout hand-off, success-poll — `payments.md` §3)                                                                                                                      | admin    |
| `/{acct_id}/billing/ledger`  | The money view: every ledger entry, running balance (`payments.md` §2)                                                                                                                            | admin    |
| `/{acct_id}/billing/usage`   | The chart view: spend by day / model / api key / end_user (`api_misc.md` §5)                                                                                                                      | member   |
| `/{acct_id}/billing/methods` | Saved cards: add (setup Checkout), remove, set default                                                                                                                                            | admin    |
| `/{acct_id}/billing/refill`  | Auto-refill rule editor with the safety rails visible: threshold, amount, daily max, auto-disable-after-3-failures state (`payments.md` §4)                                                       | admin    |
| `/{acct_id}/billing/limits`  | Spend limits: weekly/monthly caps, behavior=block (`payments.md` §2)                                                                                                                              | admin    |
| `/{acct_id}/team`            | Members list (role badges), invitations (code + magic link, revoke/re-send), invite form (`accounts.md` §4)                                                                                       | admin*   |
| `/{acct_id}/team/{user_id}`  | Member detail: role change, remove, and the member's **org-visible security events** (`GET /v1/account/users/{user_id}/audit_logs` — `audit_logs.md` §4)                                          | admin    |
| `/{acct_id}/audit`           | Account audit log, filterable by event/member/date (`audit_logs.md` §4)                                                                                                                           | admin    |
| `/{acct_id}/settings`        | Account name, billing address/contact fields, pinned API version display; **transfer ownership** (owner-only, fresh re-auth), close account (owner-only)                                          | admin    |

\* members can *view* the team list; all mutations are admin+ (`auth.md` §4 matrix — the UI
hides what the role can't do, and the gateway enforces it regardless).

**No sandbox pages here.** Interactive generation lives in the public playground
(`frontend_playground.md`); the dashboard deep-links into it ("open in playground" from a job
detail re-runs with the same params). Playground executions bill through the same session
cookie (`source = dashboard`).

## 4. UI conventions

**Sign-off:** _unsigned_

- Everything money renders from the ledger — no second bookkeeping surface (`payments.md` §7).
- Job/audit/log lists are cursor-paginated (`?limit=&after=`) mirroring the API; filters map
  1:1 to API query params — a dashboard URL is always reproducible as an API call.
- Empty states teach: no keys yet → create-key flow with curl snippet; no jobs yet → link to
  playground + quickstart.
- Danger zone patterns (revoke key, kill sessions, remove member, transfer ownership, close
  account): typed-confirmation modals; every one lands in the audit log.
