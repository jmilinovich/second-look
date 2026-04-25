---
description: Take a second look at your tests. Audits a test suite and produces a prioritized improvement plan.
argument-hint: "[path to audit, e.g. src/auth or tests/]"
---

Take a second look at the tests at `$ARGUMENTS` (or the entire test suite if no path is provided).

Use the `test-auditor` subagent. Apply the five-phase process from the test-quality skill:

1. **Inventory** — count, types, runtime, flakiness, churn
2. **Smell analysis** — flag tests by smell category
3. **Prioritized fix list** — keep / improve / rewrite / delete / parametrize per test, with S/M/L effort estimates
4. **PR slices** — groupings that each fit in one reviewable PR
5. **Prevention** — CLAUDE.md additions, review checklist items, CI hooks

Return a structured report in markdown. Do not modify any test files — the output is a plan, not code changes.
