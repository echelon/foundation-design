# Testing

**Sign-off:** _unsigned_
<!-- bt: initials + date when approved (e.g. "bt 2026-07-05"). Per-section sign-offs below;
     the top one covers the whole document. Assistants: see ../CLAUDE.md. -->

**Status:** living document (2026-07-05). How the workspace is tested: heavy unit coverage of
libraries/clients/endpoints, plus database-backed integration tests against a **dedicated
test Postgres** — annotated so they can run in isolation or be skipped entirely, and
triple-guarded against ever touching production.

---

## 1. Principles

**Sign-off:** bt 2026-07-06

- **Code is designed to be tested.** Handlers are thin; logic lives in `services/*` functions
  that take plain inputs (+ a DB handle where needed) and return values — callable from tests
  without HTTP. Provider adapters sit behind the client trait so tests substitute fakes.
- Two tiers, cleanly separated: **unit tests** (no I/O, no DB, run everywhere, always) and
  **database tests** (real Postgres, fixtures, annotated + gated).
- Determinism: no wall-clock/`now()` assumptions (clock injected), no real network in tests
  ever (provider HTTP is mocked; recorded fixtures for response parsing).
- Generated code ships its own generated tests (enum round-trips, slug lints, row-count
  matches — `enums.md`), so the mechanical layer can't silently drift.

## 2. Unit tests

**Sign-off:** _unsigned_

| Target                     | What's covered                                                                                                                                                                                                 |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `core/ids`                 | Prefix/parse round-trips, Crockford charset, length invariants, collision-shape sanity                                                                                                                         |
| `core/money`               | Micro/nano conversions, display formatting, overflow behavior, never-negative invariants                                                                                                                       |
| `core/enums`               | Generated batteries: serde round-trip, explicit wire strings, global slug uniqueness, models.md row-count match                                                                                                |
| `core/catalog`             | Every model has prices/modalities/route; exhaustiveness over `Model::ALL`; alias map acyclic + collision-free; cost function golden tests per price unit                                                       |
| `clients/*/{name}_client`  | Request construction + **response parsing against recorded fixtures** (real captured provider payloads checked into the crate, secrets scrubbed); error mapping to failure codes; retry/backoff classification |
| `router` / Any layer       | Param pruning/coercion tables, `param_adjustments` reporting, modality inference                                                                                                                               |
| `services/jobs`            | The transition function: exhaustive valid-transition matrix + invalid-transition rejection (pure function, no DB)                                                                                              |
| API request/response types | Serde round-trips, error-envelope shape, webhook signature generation/verification vectors                                                                                                                     |

**Endpoint tests without a database:** Axum routers are plain `tower::Service`s — tests drive
them in-process with `oneshot()` requests (no socket, no server). Handlers get fake service
impls; these tests assert routing, auth-header parsing (all four forms), validation errors,
and envelope shape. Anything asserting real persistence graduates to §3.

## 3. Database tests

**Sign-off:** _unsigned_

Real behavior gets tested against a real Postgres — same major version as production, throwaway
data. Comprehensive flow tests live here:

| Flow                      | Asserts                                                                                                                                        |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Signup (all three)        | User+account+membership+session created **in one transaction**; email collision → no silent merge; SSO connection row correctness              |
| Invites (all three doors) | Code + magic-link resolution, expiry, email-mismatch recording                                                                                 |
| SSO linking               | Link/unlink guards (≥1 method), cross-user connection conflict                                                                                 |
| Admission transaction     | Balance check → ledger entry → balance update → job insert → queue task, atomically; 402 paths (insufficient, spend limit); idempotency replay |
| Job lifecycle             | Full walk queued → terminal via the transition function; refund on failed/expired/canceled; `job_events` written in the same transaction       |
| Queue worker              | `FOR UPDATE SKIP LOCKED` claim semantics, visibility timeout, dead-lettering                                                                   |
| Webhooks                  | Delivery attempts logged, signature valid, Stripe-event dedupe                                                                                 |
| Ledger invariants         | `balance_after_micros` chain consistency; balance never negative                                                                               |

**Fixtures: a dedicated `test_fixtures` crate (REVISED 2026-07-14).** One crate entirely
designed around creating test data — `create_test_db()`, `fixture_account()`,
`fixture_user()`, `fixture_session()`, `fixture_job(...)` builders with sane defaults,
override-what-you-test; no SQL dump files. Rules:

- **Imported as a dev-dependency by any crate** — DB integration tests can live anywhere,
  next to the code they test. (The no-SQL-outside-`db/queries` rule is production-code only:
  `conventions_coding.md` §4.)
- **It contains no SQL of its own** — it creates rows by calling `db/queries` functions
  (and `services/*` where a flow, not a row, is the fixture), so fixtures exercise the same
  code paths production uses and schema drift breaks loudly.
- Account/user/session creation is the expected opener of the huge majority of tests — those
  builders get the most ergonomic-attention (one line, sane defaults).

**Isolation between tests:** each test runs in its own schema or its own transaction rolled
back at the end (per-test DB creation à la `#[sqlx::test]` is the fallback for tests that
must commit, e.g. SKIP LOCKED). Tests never depend on each other's rows and can run in
parallel.

## 4. Annotation & gating

**Sign-off:** _unsigned_

Database tests are **annotated** so they're selectable as a set:

- One attribute macro, `#[db_test]` (wrapping `#[sqlx::test]`-style per-test setup), marks
  every database test. It expands to a test that (a) runs the §5 safety checks, (b) acquires
  an isolated schema/transaction, (c) skips-with-message (not fails) when no test database is
  configured.
- Selection both directions, without a test DB nothing red:

| Invocation                                                          | Runs                                                                                                                                                            |
| ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `cargo test`                                                        | Unit tests; db tests auto-skip if no test DB configured                                                                                                         |
| `cargo test-db` (alias for a nextest filter on the `db_test` group) | Database tests only                                                                                                                                             |
| CI                                                                  | Both — Postgres service container is always present, so skips are treated as failures in CI (a guard env var `CI=1` turns "skip" into "error: test DB missing") |

## 5. Test-database configuration & production safety

**Sign-off:** _unsigned_

**Configuration** — one env var, easy to set, impossible to confuse with production:

- `FOUNDATION_TEST_DATABASE_URL` — read by tests **exclusively**. Tests never read
  `DATABASE_URL` and never fall back to it. Absent → db tests skip (§4).
- Committed `.env.test.example` with the local default
  (`postgres://localhost:5432/foundation_test`); `docker compose up test-db` (or a `just
  test-db` recipe) provides it in one command.

**Never against production — three independent checks**, all enforced inside the `#[db_test]`
harness before any test body runs, each sufficient alone to abort:

1. **Name check:** the database name in `FOUNDATION_TEST_DATABASE_URL` must end in `_test`.
   Production databases are named so they never can (`foundation`, `foundation_prod`).
2. **Marker check:** the harness requires a `_test_database_marker` table (created by the
   test-db setup script, never by production migrations) and refuses to run without it.
   Conversely, production migrations assert this table does NOT exist and refuse to apply if
   it does — the marker is a one-way valve.
3. **Host check:** non-loopback hosts are refused unless
   `FOUNDATION_TEST_DB_ALLOW_REMOTE=1` is explicitly set (for CI service containers with
   hostnames); even then checks 1 and 2 still apply.

Violation of any check panics before any query with a message naming the check. The checks
are themselves unit-tested.

## 6. What we deliberately don't test with mocks

**Sign-off:** _unsigned_

- **Provider APIs are never called in tests** — not even "sandbox" modes (they cost money and
  flake). Client crates are tested against recorded fixtures (§2); a separate, manually run
  `live-probe` binary (not a test) exercises real credentials when refreshing fixtures.
- **Stripe:** webhook handling is tested with signed synthetic payloads (we hold the test
  secret, so we can sign); flows are additionally exercised manually against Stripe test mode
  — but CI never talks to Stripe.
- No mocking of Postgres (no `sqlx` mock layers): the DB tier is the thing under test; a
  mocked database tests the mock.
