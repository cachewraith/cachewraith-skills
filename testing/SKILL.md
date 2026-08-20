---
name: testing
description: "Run, write, and debug tests — unit, integration, and end-to-end — for whatever framework the project uses (Jest, Vitest, PHPUnit, Pytest, Go test, cargo test, RSpec, JUnit). Invoke when the user asks to test something, run the suite, add or fix tests, check coverage, or investigate a failing test. Skip when the user wants a code review rather than tests, or is only asking how a test framework works in the abstract."
---

# Testing

Find the project's test setup, use it, and write tests that would actually catch a
regression.

## 1. Find the setup before writing anything

Never guess the runner or invent a command:

```bash
ls package.json composer.json pyproject.toml pytest.ini Cargo.toml go.mod Gemfile pom.xml 2>/dev/null
```

Then read the scripts/targets and look for a config:

```bash
# JS/TS
sed -n '/"scripts"/,/}/p' package.json
ls jest.config* vitest.config* playwright.config* test/*.json 2>/dev/null
# PHP
ls phpunit.xml* && grep -n '"scripts"' -A10 composer.json
# Python
ls tox.ini setup.cfg && sed -n '/\[tool.pytest/,/^\[/p' pyproject.toml
```

**Read `CLAUDE.md`, `README.md`, and `CONTRIBUTING.md` for test rules first.** Projects
routinely have non-obvious requirements — a real database instead of an in-memory one,
an env file to source, serial execution, a fixture that must be reset between cases.
Those override every default in this skill.

Then read one existing test in full before writing a new one. Match its structure,
naming, helpers, and assertion style. A test that doesn't look like its neighbours is a
worse test even when it passes.

## 2. Run the suite

Use the project's own command — `npm test`, `composer test`, `pytest`, `go test ./...`,
`cargo test` — not a hand-rolled invocation, since the script usually carries required
flags (config path, serial workers, env loading).

Narrow down when iterating:

```bash
npx jest --config test/jest-e2e.json test/auth.e2e-spec.ts   # one file
npx jest --config test/jest-e2e.json -t "sends a friend request"  # one case
pytest tests/test_auth.py::test_login -x
go test ./internal/auth -run TestLogin -v
```

Run the **full** suite before declaring done — a narrow run hides cross-test breakage.

## 3. What to write

Pick the level that matches what you're protecting:

- **Unit** — pure logic, edge cases, branching. Fast, no I/O, no database. Best for
  validators, formatters, calculations, state machines.
- **Integration** — a service against a real database or a real dependency. Best for
  queries, transactions, locking, anything where the ORM's behaviour is the risk.
- **End-to-end** — a real request over the real stack. Best for status codes, response
  shape, auth, middleware/guard ordering, rate limits.

Where a project has one dominant style (e.g. e2e over HTTP), follow it rather than
introducing a parallel unit-test culture nobody maintains.

### Test the behaviour, not the implementation

Assert on what a caller observes — return value, response body, status code, database
state, emitted event. A test that asserts a private method was called breaks on every
refactor and catches nothing.

### Cover the cases that actually break

For each thing under test:

- The happy path, once.
- **Every error path** — invalid input, missing auth, wrong owner, not found, conflict.
- **Boundaries** — empty, one, many; zero, negative, max; first page, last page, past
  the end.
- **Idempotency and repeats** — doing it twice, out of order, after it already happened.
- **Concurrency**, where an invariant depends on it — two requests racing for one row.

Error paths are where the bugs are. A suite that only tests success is decorative.

### Assertions

- One behaviour per test; a clear name saying what it guarantees
  (`rejects login for an unverified email`, not `test login 2`).
- Assert on **exact** wire strings, status codes, and shapes when they are a contract.
  Loose matchers let a breaking change through.
- No conditional logic in a test. A test with an `if` tests two things badly.
- Failure messages should say what broke without reading the test body.

### Fixtures and isolation

- Each test must pass **alone** and **in any order**. Shared mutable state between tests
  is a defect in the suite.
- Follow the project's reset mechanism exactly — a truncate helper, a transaction
  rollback, a fresh container. Note that a wrapping transaction cannot work when the
  code under test runs over real HTTP on its own connection; use the project's helper
  rather than reasoning from scratch.
- Reset the things that are easy to forget and produce mystifying failures elsewhere:
  rate limiters, caches, recorded mail, clocks, in-memory queues.
- Build fixtures with a factory/helper, not copy-pasted literals.
- Mock at the system boundary — third-party HTTP, payment providers, clocks, randomness.
  Do **not** mock your own database into meaninglessness.

## 4. When a test fails

1. Read the actual failure output before changing anything. The diff usually names it.
2. Decide honestly which side is wrong: the test or the code. Fix the code when the
   test encodes the intended behaviour.
3. **Never** weaken an assertion, add a retry, or bump a timeout to make a red test go
   green. If a test is genuinely obsolete, say so and delete it deliberately.
4. Flaky test? Find the shared state, the ordering assumption, or the real timing
   dependency. Do not paper over it with a sleep.
5. Reproduce with the narrowest run, fix, then re-run the full suite.

## 5. Report

Say what you ran, the actual result, and what remains:

- The command and the pass/fail counts.
- If anything failed, the failing test names and the real output — never round a
  failing run up to "done".
- What you added coverage for, and what you deliberately left uncovered and why.
- Anything you could not run (missing database, absent credentials) and what it would
  take to run it.
