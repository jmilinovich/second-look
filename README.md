# second-look

A Claude Code plugin for writing, auditing, and maintaining tests. Give your tests a second look before they earn their place.

> Every test is a bet that its future signal will exceed its future maintenance cost. Write tests that win the bet. Delete the ones that don't.

## Why "second-look"

Named for Andrew Look, whose advice on testing shaped the philosophy: signal over coverage, maintenance cost as the real constraint, integration-first, mock at boundaries, table-driven tests for combinatorial cases. A test audit is literally a second look at your suite. The practice the plugin encodes is the habit of giving each test a second look — against the five audit questions — before it earns its place.

## What's in the plugin

| Component | What it is | How it activates |
|---|---|---|
| `test-quality` skill | Full philosophy, five audit questions, patterns, smells, decisions | Auto-triggers when you talk about tests, coverage, mocks, flaky tests, etc. |
| `test-auditor` agent | Structured five-phase audit of an existing test suite | Invoked automatically for audit tasks, or manually via `/agents` |
| `/second-look` command | Explicit trigger to audit tests in a specific path | Type `/second-look src/auth` (or any path) |

The CLAUDE.md snippet is included separately (`CLAUDE-snippet.md`) for users who want always-on testing principles in their project CLAUDE.md files. Most users won't need it — the skill covers the common cases.

## Install

### From GitHub (recommended)

Once this is pushed to GitHub:

```bash
# In Claude Code
/plugin install github:<your-username>/second-look
```

Scope defaults to `user` (available across all your projects). Add `--scope project` to commit it to a specific project.

### From a local directory

For testing before publishing, or if you'd rather keep it local:

```bash
# Clone/copy the plugin directory somewhere stable
cp -r second-look ~/plugins/second-look

# Start Claude Code with the plugin loaded for this session
claude --plugin-dir ~/plugins/second-look
```

### Verify

```bash
# In Claude Code
/plugin list
```

You should see `second-look` listed. Try:

- Ask Claude "what makes a good test?" — the skill should inform the answer.
- Run `/second-look tests/` in a project with tests.
- Run `/agents` — you should see `test-auditor`.

## Usage

### Writing new tests

Just work normally. The skill auto-triggers when you ask Claude to write tests, and it applies the five audit questions before committing any test. No special invocation needed.

### Auditing an existing suite

Three ways, most explicit to least:

1. **`/second-look <path>`** — runs the full five-phase audit, returns a prioritized plan. Best for "give me a concrete action list."
2. **Ask conversationally** — "these tests are brittle, help me figure out what to do." The skill triggers and applies the framework.
3. **Invoke the agent directly** — "use the test-auditor to review `src/auth/tests/`" — useful when you want the structured subagent output without the command wrapper.

### Adding always-on guidance (optional)

If you want the core principles in your project's `CLAUDE.md` so agents see them on every task without invoking the skill:

```bash
# From inside the plugin directory
cat CLAUDE-snippet.md  # copy the ## Testing section
# Paste into your project's CLAUDE.md
```

This is redundant with the skill for most workflows. Use it only if you've found the skill under-triggers in your project.

## The philosophy in one page

**Five audit questions, applied to every test:**

1. **Signal** — does a CI failure immediately tell you what broke?
2. **Maintenance** — does the test need incidental updates when unrelated code changes?
3. **Uniqueness** — does this cover something no other test does?
4. **Behavior, not implementation** — does this survive refactors that don't change external behavior?
5. **Readability** — can someone grok it in 30 seconds?

A test that fails two or more is a candidate for rewrite or deletion.

**Defaults:**

- Integration tests over unit tests, most of the time.
- Mock at external boundaries only (DB, HTTP, time, randomness) — never internal collaborators.
- Table-driven tests (`pytest.mark.parametrize`) for combinatorial cases.
- Name tests after behavior, not functions.
- AAA structure (Arrange / Act / Assert).
- Fix flaky tests at the root cause — never retry.

Full details in `skills/test-quality/SKILL.md` and its references.

## Updating

```bash
/plugin update second-look
```

## Uninstall

```bash
/plugin uninstall second-look
```

## Source material

Philosophy synthesized from Andrew Look's direct advice, Kent C. Dodds's testing trophy model, Vladimir Khorikov's *Unit Testing: Principles, Practices, and Patterns*, Martin Fowler's TestPyramid writing, and the test-smells literature.

## License

MIT
