# Decisions: Add, Keep, Delete, Parametrize

How to decide what happens to a test — whether you're writing new ones or evaluating existing ones.

## "How many tests should I add?"

There is no number. The question is the wrong shape. Replace it with:

**For each behavior worth locking down, is there at least one test that fails loudly when that behavior breaks?**

Behaviors worth locking down:
- Contracts with external callers (API endpoints, public functions)
- Invariants that have been violated before (regression tests)
- Logic complex enough that you can't hold it in your head
- Error-handling paths that must not silently swallow failures
- Anything where a wrong result would cause damage (money, data loss, security)

Behaviors *not* worth separate tests:
- Trivial getters and setters
- Pass-through wrappers around well-tested libraries
- Implementation details that happen to be public
- Behavior already covered by an integration test at a higher layer

## Decide: add a new test

Before writing a new test, ask:

1. **What behavior am I trying to prove?** State it as a sentence — "the function returns X when Y." If you can't name the behavior, the test will be incoherent.
2. **Does an existing test already cover this?** If yes, don't write a new one — either the existing test is sufficient, or extend it (parametrize to add a case).
3. **Is this behavior testable at a higher level?** If an integration test on the API endpoint would catch a regression here, the unit test may be redundant. Prefer the higher-level test unless you have a specific reason to isolate.
4. **Will this test fail loudly and informatively if the behavior breaks?** If it would only fail in a confusing way, rewrite the setup or the assertion before committing.

If all four pass: write the test. Structure with AAA, name after the behavior, use parametrize if there are multiple cases of the same kind.

## Decide: keep an existing test

For an existing test, the default is **keep** unless it fails the audit. Apply the five audit questions:

1. Signal — will a failure tell me what broke?
2. Maintenance — does this test get touched for incidental reasons often?
3. Uniqueness — does it cover something no other test does?
4. Behavior not implementation — does it survive refactors?
5. Readability — can someone read it in 30 seconds?

| Fails question | Action |
|---|---|
| 1 (signal) | Improve: better assertion message, clearer name |
| 2 (maintenance) | Rewrite: likely coupled to implementation; move to behavior assertions |
| 3 (uniqueness) | Delete or merge into another test |
| 4 (behavior) | Rewrite: assert on outputs, not internal calls |
| 5 (readability) | Improve: rename, simplify setup, use AAA structure |

Tests that pass all five: leave them alone, even if they're "old-looking." Age isn't a reason to rewrite.

## Decide: delete a test

Delete when:

- **It's a duplicate.** Another test already exercises the same path with the same assertion.
- **It tests behavior that no longer exists.** The code it covered has been removed; the test is now testing nothing meaningful, or tests a wrapper that should also be removed.
- **It's been skipped or xfailed for a long time with no tracking issue.** Dead code.
- **It asserts on implementation details that are now inaccessible or irrelevant.** The refactor that would have broken it has happened, but the test was mechanically updated to match, and it's no longer validating anything a caller would care about.
- **The behavior it covers is fully covered by a higher-level integration test** and the lower-level test is high-maintenance.

Don't delete when:

- **It always passes.** Always-passing is what "no regressions" looks like. A quiet test doing its job isn't a problem.
- **You don't understand what it's testing.** First understand, then decide. If it's protecting something you didn't know about, that's exactly the kind of test you want.
- **It's slow but valuable.** Move it to a different tier; don't delete.

When deleting, commit with a message explaining what was removed and why. Future readers will thank you.

## Decide: parametrize

Parametrize when you have (or are about to write) three or more test functions that:

- Exercise the same function or behavior
- Differ only in input values and expected outputs
- Have the same setup and assertion structure

Signs to parametrize an existing group of tests:

- You find yourself copy-pasting a test and changing one literal.
- You see `test_foo_case_1`, `test_foo_case_2`, `test_foo_empty`, etc., all near each other with similar bodies.
- Adding a new case requires writing a whole new function.

Don't parametrize when:

- The tests have meaningfully different setup or assertions (false uniformity — let them stay separate).
- There are only two cases and they're going to stay that way (overhead isn't worth it).
- The cases share names but actually test different behaviors (parametrizing would hide the fact that they're unrelated).

## When you're unsure: default to behavior

The strongest heuristic across all these decisions: **test what the caller cares about.**

A test that could be written by someone who only knew the public interface of the module — and no internal details — is almost always the right test. If you find yourself reaching into private state or mocking internal collaborators to make a test pass, step back and ask what the caller would observe. Test that.

## Applying this to an existing project

When pointed at a project with an existing suite, don't apply these decisions wholesale. Pick one module. Evaluate every test in it using this framework. Ship the changes as one PR. Then the next module. Incremental always beats a big rewrite — both for review quality and for avoiding accidental regressions.
