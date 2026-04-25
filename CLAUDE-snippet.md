# CLAUDE.md Snippet: Testing Principles

Paste this into the `CLAUDE.md` at the root of any project where you want agents to apply the test-quality philosophy automatically. Keep it short — CLAUDE.md is always-in-context, so verbosity is expensive. Deeper guidance lives in the `test-quality` skill, which agents can invoke when they need the full playbook.

---

## Testing

Every test must earn its place. A test earns its place when a CI failure tells you exactly what broke, in a way no other test already does, and the test doesn't need incidental updates every time the code is refactored. Optimize for signal, not coverage.

**Before writing or modifying a test, apply these five questions:**

1. **Signal** — if this fails in CI, will I immediately know what's broken?
2. **Maintenance** — will this need changes every time something incidental in the code shifts?
3. **Uniqueness** — does any other test already cover this?
4. **Behavior, not implementation** — will this survive a refactor that doesn't change what callers observe?
5. **Readability** — can someone unfamiliar with the code read this in 30 seconds and know what it's asserting?

If any answer is "no," fix the test before committing.

**Defaults for this project:**

- Prefer integration tests on API endpoints for the happy path and main error cases. Add unit tests where logic is complex enough that an integration failure wouldn't pinpoint the cause.
- Mock only at external boundaries — databases, HTTP APIs, the filesystem, time, randomness. Never mock your own modules' internal interactions.
- Assert on observable outcomes (return values, state changes, side effects a caller would see), not on internal call order or method invocation counts.
- Use `pytest.mark.parametrize` for combinatorial cases. Three or more similar test functions is a signal to parametrize.
- Name tests after the behavior being verified, not after the function being called. `test_returns_zero_for_tax_exempt_items`, not `test_calculate_tax`.
- Structure tests as Arrange / Act / Assert, separated by blank lines.
- Fix flaky tests at the root cause. Never retry them. If you can't fix immediately, skip with a tracking issue.

**Do not:**

- Add tests to raise coverage numbers.
- Delete tests just because they "always pass" — quiet is the goal, churn is the problem.
- Write tests that break on internal refactors.
- Create multiple tests that differ only in input values — parametrize.

For auditing an existing suite, diagnosing flaky or brittle tests, or deciding whether a test should be kept, rewritten, or deleted, invoke the `test-quality` skill.
