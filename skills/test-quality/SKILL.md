---
name: test-quality
description: Evaluate, improve, and write tests using a signal-over-coverage philosophy. Use this skill whenever the user asks to write new tests, audit or improve an existing test suite, diagnose flaky or brittle tests, decide whether a test is worth keeping, review test PRs, or work out how many tests a feature needs. Also trigger when the user mentions "test coverage," "unit vs integration," "mocks," "test smells," "parametrize," "table-driven tests," or expresses frustration with tests being brittle, slow, or uninformative. Applies to any language, with Python/pytest examples.
---

# Test Quality

## Core principle

**Every test is a bet that its future signal will exceed its future maintenance cost.**

A test earns its place when a CI failure tells you exactly what broke, in a way no other test already does, and the test doesn't need incidental updates every time you refactor. Tests that fail this bet aren't just dead weight — they actively erode trust in the suite.

The goal is not coverage. The goal is **provable behavior** with the lowest maintenance drag possible.

## When this skill applies

- Writing new tests for a feature, endpoint, or module
- Auditing an existing test suite (see `references/audit-playbook.md`)
- Deciding whether to keep, rewrite, delete, or parametrize a test
- Diagnosing why a test is flaky, slow, or brittle
- Reviewing a PR where tests were added or changed
- Answering "how many tests should I add?"

## The five audit questions

Apply these to every test, whether writing new or reviewing old. A test that fails two or more is a candidate for rewrite or deletion.

1. **Signal** — If this fails in CI, do I immediately know what's broken and where to look?
2. **Maintenance** — How often does this test need incidental changes when the code it tests hasn't meaningfully changed? (Churn is the real cost. Always-passing is fine. Always-needing-updates is the problem.)
3. **Uniqueness** — Does this validate something no other test already covers?
4. **Behavior, not implementation** — Does this survive a refactor that doesn't change external behavior? Or does it break the moment I rename a private method or reorder internal calls?
5. **Readability** — Can someone unfamiliar with the code read the test in 30 seconds and know what it's asserting and why?

If a test fails #2 (high churn) or #4 (implementation-coupled), rewrite it. If it fails #3 (redundant), delete it. If it fails #1 or #5, improve it.

## Core principles

**Signal over coverage.** Line coverage is a proxy metric and a bad one. Optimize for "failures are immediately actionable."

**Integration before unit, most of the time.** Especially early in a project: test main API endpoints, cover the happy path and the main error cases, mock the database or external services, exercise control flow. You get the most signal per maintenance dollar. Add unit tests where logic is complex enough that an integration failure wouldn't pinpoint the cause, or for pure functions worth locking down. This is roughly the "testing trophy" shape (Kent C. Dodds): more integration, some unit, a little E2E, static analysis as the foundation.

**Test behavior, not implementation.** Assert on outputs, state changes, side effects a caller would observe. Never on the fact that an internal method was called, what order internal calls happened in, or on private methods via reflection. A test that breaks when you refactor without changing behavior is a tax on refactoring.

**Mock at boundaries, not internals.** Mock external I/O (databases, HTTP APIs, message queues, the filesystem when relevant). Don't mock your own modules' interactions with each other — those are the interactions you most want to test. Rule of thumb: mock only ports, assert only outcomes.

**Tests are executable documentation.** A passing test is a proof: "for these inputs, the function returns this output." When someone reads a test, they should understand the contract of the code. Name tests after the behavior being verified, not after the function being called. `test_returns_zero_for_tax_exempt_items` beats `test_calculateTax`.

**Each test asserts one thing.** Multiple unrelated assertions turn one failure into an ambiguous mess. If you need multiple assertions to describe one behavior (e.g., "the response is 200 AND the body matches"), that's fine — they describe one logical behavior. Split tests when they describe separate behaviors.

**Prefer table-driven tests for combinatorial cases.** When you have a function that needs to be tested across many input/output pairs, use `pytest.mark.parametrize` (or the equivalent). Each case runs as its own test, fails individually, is one line to add, and makes the expected behavior readable at a glance. See `references/patterns.md`.

**Failure legibility is a feature.** Write assertion messages and test names that make the failure self-describing. A good test failure in CI does not require you to open the test file to understand what went wrong.

## Workflows

### Auditing an existing project

Use when the user says something like "help me improve the tests in this project," "these tests are a mess," or "our test suite is slow and brittle." Full playbook in `references/audit-playbook.md`. Short version:

1. **Inventory** — how many tests, what types (unit/integration/E2E), runtime, current failure rate, flakiness.
2. **Identify smells** — scan for the patterns in `references/smells.md`. Rank candidates by impact (how often they fire, how much they slow CI, how often they churn).
3. **Prioritize** — flaky tests first (they erode trust in the whole suite), then brittle/high-churn, then redundant, then unclear.
4. **Fix in slices** — don't rewrite everything. Pick a module, apply the five audit questions to every test in it, rewrite/delete/parametrize, ship.
5. **Prevent regression** — add the relevant principles to CLAUDE.md and the review checklist.

### Writing new tests

Use when the user says "add tests for this" or when you've just written a feature. Default approach:

1. **Start with the integration test.** What's the main user-facing behavior? What's the happy path? Write that first, end-to-end through the control flow, mocking only at external boundaries.
2. **Add tests for the main error cases.** Not every error — the meaningful ones a caller cares about.
3. **Consider: is there pure logic here that's complex enough to need unit tests?** If yes, write them as parametrized table tests where there are multiple cases. If there's only one case, a single test is fine.
4. **For each test, run the five audit questions before committing.** If you can't confidently answer "yes" to all five, fix the test.
5. **When a bug is found later, add a regression test that would have caught it.** This is often the highest-signal test in the suite.

## Reference files

Load these as needed. Each is independently useful.

- **`references/audit-playbook.md`** — Step-by-step process for auditing and improving an existing test suite. Read when the task is "fix my tests."
- **`references/patterns.md`** — Core patterns: AAA structure, table-driven testing with pytest.mark.parametrize, mocking at boundaries, test naming. Read when writing or rewriting tests.
- **`references/smells.md`** — Catalog of test smells and anti-patterns with what to do about each. Read when scanning an existing suite or reviewing a PR.
- **`references/decisions.md`** — Decision framework for "add this test / keep this test / delete this test / parametrize this test." Read when evaluating individual tests.

## What not to do

- Don't add tests just to raise coverage numbers. Coverage without signal is noise.
- Don't retry flaky tests instead of fixing them. Retries hide real bugs and train the team to ignore failures.
- Don't write tests that couple to implementation ("check that method X was called"). Test observable outcomes.
- Don't mock your own code's internal calls. Mock external boundaries only.
- Don't write a new test function for every parameter combination. Use table-driven tests.
- Don't delete a test that "always passes" just because it always passes. Always-passing is fine — that's what "no regressions" looks like. Delete tests that churn, not tests that are quiet.
