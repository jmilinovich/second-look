# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`second-look` is a **Claude Code plugin** that encodes a signal-over-coverage testing philosophy. It is not application code — it is a bundle of Markdown-defined components that get loaded into other Claude Code sessions when a user installs the plugin. Edits here change how Claude behaves about tests in any project that has the plugin installed.

## Plugin architecture

Three components, three different activation paths. Keep them in sync — the README documents what users see; this section is what to remember when editing.

| File | Component | Activation |
|---|---|---|
| `skills/test-quality/SKILL.md` | Skill (autonomous) | Triggers automatically when a user mentions tests, coverage, mocks, flakes, parametrize, etc. The `description` field in the frontmatter is the trigger surface — it must enumerate the situations where Claude should invoke the skill. |
| `agents/test-auditor.md` | Subagent | Invoked by Claude (or by `/agents`) when a structured five-phase audit is the right shape of work. The `tools:` line in the frontmatter restricts what the agent can do — it is read-only by design (`Read, Grep, Glob, Bash`) so it never rewrites tests itself. |
| `commands/second-look.md` | Slash command | Explicit user trigger: `/second-look <path>`. Delegates to the `test-auditor` subagent. |
| `CLAUDE-snippet.md` | Optional paste-in | Distributed snippet for users who want always-on testing principles in their own project's `CLAUDE.md`. Not loaded by the plugin — it is documentation. |
| `.claude-plugin/plugin.json` | Plugin manifest | Name, version, description, keywords. Bump `version` on user-visible changes. |

The skill's `references/` directory (`audit-playbook.md`, `patterns.md`, `smells.md`, `decisions.md`) is loaded on demand — the SKILL.md body is the always-loaded surface, references are pulled in when the relevant workflow needs them. Keep the SKILL.md body lean and push detail into references.

## Editing rules

- **Frontmatter is load-bearing.** The `description` on a skill or agent is what Claude reads to decide whether to invoke it. Vague descriptions cause under-triggering. When changing what a component does, update its description first.
- **Skill body cap.** Skill bodies should stay under ~500 lines. Push catalogs, playbooks, and tables into `references/` files and link to them from the body.
- **Don't drift from the philosophy.** Every change should still answer "yes" to the five audit questions (signal, maintenance, uniqueness, behavior-not-implementation, readability). If you're tempted to add a recommendation that contradicts "mock at boundaries only" or "integration before unit," that is a signal to reread `SKILL.md` first, not to merge.
- **The agent does not write code.** `test-auditor` outputs a plan, never modifies tests. If you find yourself adding write tools to its frontmatter, you are turning it into a different thing — split it into a separate agent rather than expanding this one.

## Manual test loop

There is no automated test suite. Verify changes by installing the plugin into a real Claude Code session and exercising each surface.

```bash
# Load the plugin from this directory into a Claude Code session
claude --plugin-dir /Users/jm/Desktop/second-look

# Then in that session:
/plugin list                  # confirm "second-look" appears
/second-look tests/           # exercise the command + agent path
# Ask conversationally: "what makes a good test?"  → skill should fire
/agents                       # confirm test-auditor is listed
```

If the skill is failing to auto-trigger on relevant prompts, the fix is almost always to broaden the `description` frontmatter, not to rewrite the body.

## Publishing

`.claude-plugin/plugin.json` carries the version. Bump it when a change is user-visible (new behavior, changed defaults, added trigger phrases). README install instructions assume `github:jmilinovich/second-look` — update them if the repo location ever changes.
