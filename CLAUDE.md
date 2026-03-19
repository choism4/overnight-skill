# Overnight Skill

Autonomous background agent for Claude Code. Give it a big coding goal, walk away, come back to a branch of working code. Decomposes, executes, and verifies tasks unattended.

## Project Structure

```
SKILL.md          — Skill definition (the actual skill that Claude Code loads and executes)
README.md         — User-facing documentation, install instructions, usage examples
CLAUDE.md         — This file. Project context for contributors and overnight workers.
LICENSE           — MIT
.gitignore        — Ignores runtime artifacts (overnight-plan.md, .overnight-stop, .overnight-logs/)
```

This is a **markdown-only skill**. There are no source code files, no build steps, no tests, and no dependencies.

## How It Works

- SKILL.md defines the full orchestrator protocol: preflight, decomposition, execution loop, mining, and summary.
- The orchestrator dispatches workers via the **Agent tool with `isolation: "worktree"`**, giving each worker an isolated repo copy.
- **EARS syntax** (Easy Approach to Requirements Syntax) is used for all task requirements: Ubiquitous, Event-driven, State-driven, Unwanted behavior, and Optional feature patterns.
- **Markdown checkboxes are the state machine**: `- [ ]` pending, `- [x]` done, `- [R]` needs review, `<!-- BLOCKED -->` failed.
- `overnight-plan.md` is the runtime state file (generated in user projects, not in this repo).

## Build / Test

None. This project contains only markdown files. There is nothing to build or test.

## Conventions

- Task requirements use EARS syntax exclusively. No freeform descriptions.
- One task = one concern. If a task touches two modules or has two EARS clauses, split it.
- The orchestrator never writes code. It decomposes, dispatches, monitors, and merges.
- Workers run sequentially (orchestrator blocks on each Agent call).
- All work happens on an `overnight/YYYY-MM-DD-HHMMSS` branch. Main is never touched.
- Squash-merge per task. `git branch -D` after merge (not `-d`, because squash creates new commits).

## Do NOT

- Do not add code files (Python, TypeScript, shell scripts, etc.). This is a markdown-only skill.
- Do not rename the skill. It must remain `overnight` in SKILL.md frontmatter.
- Do not change the SKILL.md structure without updating README.md to match (and vice versa). Keep them in sync.
- Do not remove EARS syntax requirements. They are core to the skill's design.
- Do not add an approval gate to Phase 2. `/overnight` IS the approval.
- Do not commit runtime artifacts (`overnight-plan.md`, `.overnight-stop`, `.overnight-logs/`). They are gitignored.
