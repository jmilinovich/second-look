# Test Smells

A catalog of anti-patterns to look for when auditing a test suite. For each smell: how to recognize it, why it's a problem, and what to do about it.

## Implementation coupling

**Symptoms:**
- Test asserts `mock.assert_called_with(...)` instead of asserting on outputs or state.
- Test uses reflection / private-method access to test internals.
- Test structure mirrors the implementation structure — one test per private method.
- Test breaks when you refactor without changing external behavior.

**Why it's bad:** the test is testing how the code is written, not what it does. Any refactor forces a test change, which trains developers to either avoid refactoring or to update tests mechanically without thinking about whether the change was a real behavior change.

**Fix:** rewrite to assert on observable outcomes — return values, emitted events, state changes, logged errors. If you can't observe the behavior from outside, ask whether that behavior actually matters to any caller. If it doesn't, delete the assertion. If it does, expose it through the interface.

## Over-mocking

**Symptoms:**
- Test setup is majority mocks.
- Mocks are applied to the module's own dependencies, not to external I/O.
- Tests pass even when the real code would fail, because the mocks don't reflect reality.

**Why it's bad:** mocks that replace your own code turn tests into verifications of "the code does what the mocks say it does" — a tautology. The real behavior (the interaction between your functions) is never exercised. Plus, mocks drift from the real API over time, silently breaking realism.

**Fix:** mock only at ports — external services, I/O, time, randomness. Let your own modules run together in the test. This is often the move from "unit" to "integration" and usually increases signal while reducing the mock count.

## Flaky tests

**Symptoms:**
- Sometimes passes, sometimes fails, same code.
- Teams have learned to retry the CI run when it fails.
- Flaky tests have been quarantined or marked with `@pytest.mark.flaky`.

**Why it's bad:** flakiness is the single biggest erosion of trust in a test suite. Once a test is flaky, failures get assumed-flaky, and real bugs slip through. Retries mask the problem.

**Fix:** fix the root cause, don't retry. Common root causes:
- **Async timing** — sleeps used instead of waiting on a condition. Replace with explicit condition waits with a timeout.
- **Shared state** — tests depend on ordering because they share state (module-level variables, test DB rows, temp files). Isolate state per test.
- **Time dependence** — `datetime.now()` compared to a fixed expectation. Inject a clock.
- **External services** — test hits real external API. Mock at the boundary.
- **Randomness** — seed the RNG deterministically for tests.

If you can't fix it now, mark it skipped with a tracking issue. Never leave it "retry three times."

## Redundant tests

**Symptoms:**
- Multiple tests with minor variations that all pass or fail together.
- Tests that exercise the same code path through slightly different entry points.
- A unit test and an integration test that both verify the same invariant.

**Why it's bad:** each redundant test is maintenance cost with no added signal. When behavior changes, you update N tests instead of 1.

**Fix:**
- If the tests vary only in input/output values, collapse into a parametrized test.
- If two tests verify the same invariant at different layers, keep the higher-level one (usually the integration test).
- Delete the rest. Commit with a clear message explaining what was removed and why.

## God fixtures / shared mega-setup

**Symptoms:**
- A single fixture or `setUp` method that creates 10+ objects.
- Tests that depend on a fixture have no visible context for what's set up.
- Changes to the fixture break unrelated tests.

**Why it's bad:** you can't read a test and know what it's exercising. Failure debugging requires tracing through the fixture to understand what state produced the failure.

**Fix:** split fixtures by purpose. Each test should use fixtures that are relevant to it. Inline setup that's specific to one test.

## Assertion roulette

**Symptoms:**
- A single test with many unrelated assertions.
- When the test fails, the failure message doesn't tell you which of the N things went wrong.
- Setup for the test includes multiple acts ("do A, then do B, then assert about both").

**Why it's bad:** you lose the ability to know what broke without re-reading and re-running the test.

**Fix:** split into one test per behavior. Use parametrize when the assertions are "the same check against multiple inputs" — each case becomes its own test with its own failure. For genuinely compound assertions describing one behavior (e.g., "response is 200 and body is valid JSON and has a user_id field"), keep them together but use clear assertion messages:

```python
assert response.status_code == 200, f"expected 200, got {response.status_code}: {response.text}"
```

## Obscure tests

**Symptoms:**
- Test name doesn't describe what it verifies.
- Test body is hard to read because of clever abstractions or helpers.
- Reader can't tell in 30 seconds what the expected behavior is.

**Why it's bad:** tests are documentation. An obscure test fails to document.

**Fix:**
- Rename the test to describe the behavior being verified.
- Inline helpers that add indirection without clarity.
- Use explicit expected values (`assert total == 27.00`) rather than recomputed ones (`assert total == subtotal + tax + shipping`).

## Slow tests in the fast tier

**Symptoms:**
- A "unit test" hits a real database, filesystem, or network.
- Local test run takes minutes, not seconds.
- Developers have stopped running tests locally because it's too slow.

**Why it's bad:** slow tests don't get run. Fast feedback is the point.

**Fix:** identify what makes it slow. If it's I/O, mock it or move the test to a slower integration tier that runs on demand. If it's CPU, figure out if the computation is really being tested or if it's incidental.

Partition your suite into tiers:
- **Fast unit tests** — milliseconds each, run on every save.
- **Integration tests** — seconds each, run on every commit.
- **E2E / system tests** — slow, run in CI only, or nightly.

## Test interdependence

**Symptoms:**
- Tests fail when run in a different order or in isolation.
- Tests share state via global variables, files, or a persistent database.
- A test "sets up" state that a later test depends on.

**Why it's bad:** you can't run a single test to debug. Parallel test execution is impossible. Adding a new test can silently break existing ones.

**Fix:** each test must set up and tear down its own state. Use fresh fixtures, transactional database rollbacks, temp directories. Never rely on another test having run first.

## Magic values / incidental coupling to data

**Symptoms:**
- Test expects a specific ID or value that comes from an external source.
- Changing seed data in a separate file breaks tests in unrelated places.

**Fix:** set up the exact data the test needs, inline or in a test-specific fixture. Don't rely on environment, seed files, or production-shaped data unless that's explicitly what's being tested.

## Dead tests

**Symptoms:**
- Tests that are skipped, commented out, or marked `xfail` without a tracking issue.
- Tests for code that no longer exists.
- Tests that are technically running but don't actually assert anything meaningful.

**Fix:** delete them. If there's a reason to keep them skipped, that reason is a tracked issue, not a comment.

---

## Quick scan checklist

When scanning a test file during an audit, look for these in order:

1. `time.sleep(` — likely flaky
2. `.assert_called_with(` / `.assert_called_once(` — likely implementation coupling
3. `mocker.patch(...)` counts > 3 in one test — likely over-mocked
4. `@pytest.mark.skip` / `@pytest.mark.xfail` without comment — likely dead
5. `@pytest.mark.flaky` — flakiness hidden by retries
6. Fixture used by 1 test and 50 lines long — god fixture
7. Test name is just the function name being tested — poor naming
8. Three+ tests in a row with near-identical bodies — parametrize candidate
