---
name: testing
description: "Run, write, and debug tests — unit, integration, and end-to-end — in whatever framework the project already uses (Jest, Vitest, PHPUnit, Pytest, Go test, cargo test, RSpec, JUnit). Invoke when the user asks to test something, run the suite, add or fix tests, check coverage, or investigate a failing test. Skip when they want a code review rather than tests, or are asking how a test framework works in the abstract."
---

# Testing

Find the project's test setup, use it, and write tests that would actually catch a
regression.

## 1. Find the setup before writing anything

Never guess the runner or invent a command:

```bash
ls package.json composer.json pyproject.toml pytest.ini Cargo.toml go.mod Gemfile pom.xml 2>/dev/null
sed -n '/"scripts"/,/}/p' package.json          # JS/TS, then jest/vitest/playwright config
ls phpunit.xml* tox.ini setup.cfg 2>/dev/null   # PHP / Python
```

**Read `CLAUDE.md`, `README.md`, and `CONTRIBUTING.md` for test rules first.** Projects
routinely have non-obvious requirements — a real database instead of an in-memory one, an
env file to source, serial execution, a fixture reset between cases. Those override every
default here.

Then read one existing test in full and match its structure, naming, helpers, and
assertion style. A test that doesn't look like its neighbours is a worse test even when it
passes.

## 2. Run the suite

Use the project's own command — `npm test`, `composer test`, `pytest`, `go test ./...`,
`cargo test` — not a hand-rolled invocation; the script usually carries required flags
(config path, serial workers, env loading).

Narrow down while iterating, then run the **full** suite before declaring done — a narrow
run hides cross-test breakage.

```bash
npx jest --config test/jest-e2e.json -t "sends a friend request"
pytest tests/test_auth.py::test_login -x
go test ./internal/auth -run TestLogin -v
```

## 3. What to write

Pick the level that matches what you're protecting:

- **Unit** — pure logic, edge cases, branching. Fast, no I/O. Validators, formatters,
  calculations, state machines.
- **Integration** — a service against a real database or dependency. Queries,
  transactions, locking — anything where the ORM's behaviour is the risk.
- **End-to-end** — a real request over the real stack. Status codes, response shape, auth,
  guard ordering, rate limits.

Where a project has one dominant style, follow it rather than introducing a parallel
testing culture nobody maintains.

**Test behaviour, not implementation.** Assert on what a caller observes — return value,
response body, status code, database state, emitted event. A test asserting a private
method was called breaks on every refactor and catches nothing.

**Cover what actually breaks:** the happy path once, then **every error path** (invalid
input, missing auth, wrong owner, not found, conflict), **boundaries** (empty/one/many,
zero/negative/max, first and last page), **idempotency** (doing it twice, out of order,
after it already happened), and **concurrency** where an invariant depends on it. Error
paths are where the bugs are; a suite that only tests success is decorative.

**Assertions:** one behaviour per test, named for what it guarantees (`rejects login for
an unverified email`, not `test login 2`). Assert **exact** wire strings, status codes,
and shapes when they are a contract — loose matchers let breaking changes through. No
conditional logic in a test; an `if` tests two things badly.

**Fixtures and isolation:** each test must pass **alone** and **in any order** — shared
mutable state is a defect in the suite. Follow the project's reset mechanism exactly
(truncate helper, transaction rollback, fresh container); note that a wrapping transaction
cannot work when the code under test runs over real HTTP on its own connection. Reset the
things that produce mystifying failures elsewhere: rate limiters, caches, recorded mail,
clocks, in-memory queues. Build fixtures with a factory, not copy-pasted literals. Mock at
the system boundary — third-party HTTP, payments, clocks, randomness — and do **not** mock
your own database into meaninglessness.

## 4. When a test fails

1. Read the actual failure output before changing anything; the diff usually names it.
2. Decide honestly which side is wrong. Fix the code when the test encodes the intended
   behaviour.
3. **Never** weaken an assertion, add a retry, or bump a timeout to turn red green. If a
   test is genuinely obsolete, say so and delete it deliberately.
4. Flaky? Find the shared state, ordering assumption, or real timing dependency. Do not
   paper over it with a sleep.
5. Reproduce with the narrowest run, fix, then re-run the full suite.

## 5. Report

The command and the pass/fail counts. If anything failed, the failing test names and the
real output — never round a failing run up to "done". What you added coverage for, what
you deliberately left uncovered and why, and anything you could not run (missing database,
absent credentials) with what it would take to run it.
