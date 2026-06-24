---
name: testing-standards
description: Expert testing for the team's services — pytest for FastAPI/Python, contract tests that verify the API matches what clients expect, and resilience (chaos) tests proving the system degrades gracefully when external systems fail. Use whenever work involves writing, fixing, or reviewing tests, setting up fixtures or a test database, verifying an endpoint or schema or integration, checking coverage, or asserting failure-mode behavior — even if the user just says "add tests." Framework-flexible (matches the repo's existing test setup rather than imposing one) with pytest plus in-memory SQLite using transactional rollback as the worked default. Do NOT use to implement features, change the API contract, or build infrastructure — it tests what other agents build.
---

# Testing Standards

Tests exist to let the team change code without fear. Good tests pin *behavior* and the *contract*; they don't pin implementation detail — tests that mirror internals break on every refactor and get deleted. Two things are judged: does the suite catch real regressions fast, and does it prove the system behaves under failure, not just on the happy path.

## How this skill works

1. **Match the repo.** Detect the runner (pytest), layout (`tests/`), fixtures, and async style, and follow them. Don't introduce a new framework or restructure unprompted.
2. **Test at the right level** (pyramid below).
3. **Test behavior, not internals.** Assert observable outcomes — response shape, status, side effect — not which private methods were called.
4. **Cover the unhappy paths.** Errors, validation, empty results, and external-failure cases are where bugs live.
5. **Keep tests isolated and deterministic.** Each test owns its setup/teardown; no order dependence; no real network or wall-clock.

Pointers: test DB + fixtures → `references/test-database.md`; resilience/chaos → `references/resilience-tests.md`.

## The pyramid (sizes match the repo)

- **Unit** — services and pure logic with dependencies mocked. Fast, most numerous.
- **Integration** — router through to the DB via the app, using the test database. Verifies wiring, status codes, response shapes.
- **Contract** — the team-critical layer. Verify backend responses match the schemas clients generate from `/openapi.json`, and that request validation rejects bad input. A failing contract test means a client *will* break — treat it as a release blocker.
- **e2e** — a couple of full-flow smoke tests; keep minimal and slow-path only.

## FastAPI testing (worked default)

- Drive the app with `httpx.AsyncClient` / `TestClient` and **dependency overrides** — `get_session` → test session, `get_current_user` → a fake principal. This is how you test routes without real auth or a real DB.
- Each test runs against **in-memory SQLite inside a transaction that rolls back** on teardown — fast and fully isolated. Full fixture in `references/test-database.md`.
- Never hit real external systems; mock at the **adapter boundary** (the integration clients), injected via override — not deep inside `httpx`.

## Contract testing (don't skip this)

- Assert response bodies validate against the published `response_model` / schema.
- Assert invalid requests return `422` with the expected shape.
- Strongest guard against silent drift: a CI step that regenerates types from the live `/openapi.json` and fails on a diff. Recommend it even if you don't own CI.

## External systems in tests

Mock the integration adapter, then assert *your code's reaction to failure*: a timeout is retried then surfaced cleanly; a duplicate webhook is deduped; a mail failure leaves the domain action succeeded. Asserting "it works when the dependency works" is half a test — see `references/resilience-tests.md`.

## Quality floor (non-negotiable, never announced)

Deterministic, isolated, hermetic (no real network/clock/random unless injected and controlled); test names state the behavior verified; one behavioral focus per test; no `sleep()` for timing (use fakes/freezegun); a red test is fixed or deleted, never committed.

## Self-critique before delivering

- **Coverage pass:** are unhappy paths and the contract tested, or only the happy path?
- **Resilience pass:** is there at least one test proving graceful degradation when an external system fails?
- **Refactor pass:** would these survive an internal refactor that preserves behavior? If not, they test internals — rewrite against observable behavior.
