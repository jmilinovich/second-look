# Audit Playbook: Improving an Existing Test Suite

Use this when the task is to improve a test suite that already exists, rather than to write new tests. The goal is to increase signal and reduce maintenance drag without rewriting everything at once.

## Step 1: Inventory

Before touching anything, build a picture. Run these and record the answers:

- **How many tests?** Broken down by directory/module if possible.
- **Test type distribution** — unit, integration, E2E. Rough count is fine.
- **Total runtime** — local and in CI.
- **Flakiness** — which tests fail intermittently? (Check CI history if available. GitHub Actions, CircleCI, and most CI systems expose this.)
- **Current coverage** — by module, not aggregate. Aggregate coverage is a vanity metric.
- **Recent churn on tests** — use `git log --follow` on test files and count commits touching them. Tests that appear in many unrelated PRs are high-maintenance.

Output: a one-page snapshot. This is the baseline you'll improve against.

## Step 2: Classify smells

For each test module (don't try to do the whole suite at once), scan for the smells catalogued in `smells.md`. The high-impact ones to look for first:

1. **Flaky tests** — fix first. They poison the well. Every flaky test trains the team to ignore failures, which masks real bugs.
2. **High-churn tests** — tests modified in many unrelated commits. These are usually implementation-coupled.
3. **Implementation-detail tests** — tests that assert on private methods, internal call order, or structural properties of the code.
4. **Over-mocked tests** — tests where more than half the setup is mocking. Usually means the test is asserting on internal choreography instead of outcomes.
5. **Duplicate/near-duplicate tests** — multiple tests exercising the same behavior with minor variations, usually candidates for parametrization.
6. **Slow tests in the fast tier** — a "unit" test that hits a real database doesn't belong there.

## Step 3: Prioritize

Rank fixes by **impact per hour of work**. A rough ordering that usually works:

1. **Flaky tests** — highest priority regardless of effort. Quarantine any that can't be fixed immediately (mark skipped with an issue link, don't just retry them).
2. **Tests that break CI frequently due to implementation coupling** — biggest tax on day-to-day work.
3. **Slow tests in critical paths** — target the p95 of CI time.
4. **Redundant tests** — low risk to delete, small wins compound.
5. **Unclear tests with poor failure messages** — improves future debugging time.

Don't touch low-churn, quiet, clear tests even if they could theoretically be "better." The maintenance cost is zero. Leave them alone.

## Step 4: Fix in slices

Pick one module. Apply the five audit questions from SKILL.md to every test in it. For each test, one of four actions:

- **Keep as-is.** It passes the audit. Move on.
- **Improve.** Rename for clarity, add a failure message, tighten assertions, remove dead setup. Cheap wins.
- **Rewrite.** Test is coupled to implementation details or over-mocked. Rewrite to assert on observable behavior. Usually involves changing what's mocked (mock less, or mock at a different boundary).
- **Delete.** Redundant with another test, or validates something no longer meaningful. Commit with a clear message explaining what was removed and why.
- **Parametrize.** Multiple near-duplicate tests collapsed into one table-driven test. See `patterns.md`.

Ship the slice. Then the next.

## Step 5: Prevent regression

An audit without prevention is a treadmill. Add prevention at three levels:

1. **CLAUDE.md** (project root) — the top-of-mind principles agents see on every task. Use the snippet that accompanies this skill.
2. **PR template / review checklist** — for every test added or modified, does it pass the five audit questions? Especially: is it asserting on behavior or implementation?
3. **Lint / CI** — there are tools for some test smells. Consider:
   - A maximum test runtime threshold that fails CI.
   - A flakiness tracker (most CI platforms have one, or use a plugin).
   - Import linting to flag tests that import private modules.

## Anti-patterns during an audit

Things not to do while auditing:

- **Don't rewrite the whole suite at once.** You'll introduce bugs, the PR will be unreviewable, and you'll lose momentum.
- **Don't optimize for higher coverage.** Coverage may go *down* during a good audit (you're deleting redundant tests), and that's fine.
- **Don't delete tests without understanding what they prove.** Even a badly-written test may be the only thing guarding some behavior. If in doubt, rewrite rather than delete.
- **Don't fix tests in isolation from the code they cover.** Sometimes the test is bad because the code is untestable. A function with too many responsibilities or hidden dependencies often needs to be refactored for the test to be good.

## Deliverable shape

If you're running this audit as an agent, produce:

1. **The inventory snapshot** (step 1).
2. **A prioritized fix list** — grouped by module, with an action (keep/improve/rewrite/delete/parametrize) per test, and an estimate of effort.
3. **Shipped PRs** — one per slice, each with a clear title describing the change and a one-paragraph rationale referencing which audit questions each change addresses.
4. **Prevention artifacts** — updated CLAUDE.md, review checklist, any CI hooks added.

Keep the audit summary in a project doc (or a Grove note) so future work can track what was done and what's left.
