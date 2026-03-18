# Overnight

One prompt. Zero babysitting. Come back to working code.

## Install

```bash
npx skills add choism4/overnight-skill
```

## Usage

```
/overnight Add user authentication with email/password, OAuth, and session management
```

That's it. Walk away.

Come back and check:

```bash
git log --oneline overnight/2026-03-18-231500   # one commit per task
cat overnight-plan.md                            # [x] done, [R] needs review, BLOCKED failed

# Ship it
gh pr create --head overnight/2026-03-18-231500

# Or discard
git branch -D overnight/2026-03-18-231500
```

## What It Does

Overnight is an autonomous background agent for Claude Code. It takes a big goal, decomposes it into the maximum number of small testable tasks, then executes them one by one in isolated worktree sessions — for hours, unattended.

```
/overnight "PROMPT"
     │
     ▼
Phase 0: Preflight
├── Check clean working tree (refuse if dirty)
├── Create branch: overnight/YYYY-MM-DD-HHMMSS
│
Phase 1: Decompose
├── Read CLAUDE.md + codebase
├── Break goal into maximum atomic tasks (EARS syntax)
├── Write overnight-plan.md + cost estimate
│
Phase 2: Execute (orchestrator loop)
│  ┌─────────────────────────────────────────┐
│  │  For each task:                         │
│  │  ├── Skip BLOCKED tasks                 │
│  │  ├── claude -w overnight-N              │
│  │  │     --permission-mode auto           │
│  │  │     --max-budget-usd 5               │
│  │  │   (blocks until worker exits)        │
│  │  ├── Verify merge → squash-merge        │
│  │  ├── Safe cleanup (git branch -d)       │
│  │  └── Update plan [x] or [R] or BLOCKED  │
│  └─────────────────────────────────────────┘
│
Phase 3: Mine for More Work (max 2 cycles)
├── Run tests + linter
├── Find concrete gaps (failures, TODOs, BLOCKED retries)
├── Generate new tasks → back to Phase 2
└── No gaps found → print summary → "I'm going to bed too..."
```

## Architecture

```
┌─────────────────────────┐
│     ORCHESTRATOR         │    Current Claude session.
│     (this session)       │    Never writes code.
│                          │    Decomposes, dispatches,
│                          │    monitors, merges.
└────────────┬────────────┘
             │ claude -w overnight-1 (blocking)
             ▼
┌─────────────────────────┐
│   WORKER overnight-1     │    Isolated worktree.
│                          │    Implements ONE task.
│   1. Read relevant code  │    Runs acceptance check.
│   2. Implement task      │    Commits + exits.
│   3. Run acceptance cmd  │
│   4. Commit + exit       │    Then orchestrator merges
└─────────────────────────┘    and spawns overnight-2.
```

- Each worker gets a **fresh worktree** branched from the latest overnight branch
- Workers run **synchronously** — orchestrator waits, no polling
- **Main is never touched** — all work on `overnight/YYYY-MM-DD-HHMMSS`

## Task Format

Every task uses [EARS syntax](https://en.wikipedia.org/wiki/EARS_%28requirements_engineering%29) with two acceptance modes:

```markdown
- [ ] **Hash passwords on registration** `overnight-2`
  - WHEN a user account is created THE SYSTEM SHALL store the password using bcrypt
  - Acceptance: `npm test -- --grep "password hashing"`
  - Files: src/models/user.ts, tests/user.test.ts

- [ ] **Refactor auth middleware** `overnight-5`
  - THE SYSTEM SHALL separate authentication and authorization into distinct middleware
  - Acceptance: REVIEW: auth.ts split into authenticate.ts and authorize.ts
  - Files: src/middleware/auth.ts → src/middleware/authenticate.ts, authorize.ts
```

| Mode | Format | Result |
|------|--------|--------|
| Machine-verifiable | `Acceptance: \`command\`` | Marked `[x]` on pass |
| Human review needed | `Acceptance: REVIEW: description` | Marked `[R]` — needs your eyes |

## What to Expect

| Metric | Typical range |
|--------|--------------|
| Tasks generated | 8–30 |
| Cost per task | ~$3–5 |
| Total cost | $25–100 |
| Runtime | 1–4 hours |
| Output | One branch, one squash-merge per task |

## Termination

| Trigger | Message |
|---------|---------|
| `.overnight-stop` file | "Stopping as requested. Here's where things stand..." |
| 3 consecutive BLOCKED tasks | "Hit a wall — 3 tasks in a row couldn't complete..." |
| All tasks done + Phase 3 exhausted | "I'm going to bed too..." |

Every termination prints a structured summary with completed/blocked/remaining counts and next-step commands.

## State

`overnight-plan.md` is the entire state machine:

```markdown
- [x] **Create User model** `overnight-1`                     ← done
- [R] **Refactor auth middleware** `overnight-2`               ← done, needs review
- [ ] **Add OAuth flow** `overnight-3`                         ← next up
- [ ] **Add rate limiting** <!-- BLOCKED: reason -->            ← failed, skipped
```

## Safety

| Concern | How it's handled |
|---------|-----------------|
| Main branch | Never touched. All work on `overnight/` branch. |
| Dirty working tree | Refuses to start. Must commit or stash first. |
| Branch collision | Timestamp precision (`HHMMSS`) prevents same-day collisions. |
| Worker crash | Squash-merge verified before branch deletion. |
| Merge conflict | Detected and aborted. Task marked BLOCKED. |
| Branch deletion | Uses `git branch -d` (safe). Refuses if not merged. |
| Runaway mining | Phase 3 capped at 2 cycles. Only objective gaps. |
| Cost | `--max-budget-usd 5` per worker. Total estimate printed upfront. |

## CLAUDE.md Matters

Each worker reads `CLAUDE.md` as its briefing. No CLAUDE.md = the first task creates one.

**Good CLAUDE.md for overnight:**

```markdown
# Build & Test
npm run build          # TypeScript compilation
npm test               # Jest suite
npm run lint           # ESLint

# Architecture
src/models/     — Sequelize models, one per table
src/routes/     — Express route handlers
src/middleware/  — Auth, validation, error handling

# Conventions
- All routes return { data, error, meta } envelope
- Tests use factory functions, not raw fixtures

# Do NOT
- Do not use raw SQL — use Sequelize query builder
- Do not add dependencies without checking package.json
```

## Debugging

```bash
# Quick status
cat overnight-plan.md | grep -E '^\- \[' | head -20

# Find blocked tasks
grep "BLOCKED" overnight-plan.md

# See all changes vs main
git diff --stat main...overnight/2026-03-18-231500

# Fix a blocked task manually, then resume
vim overnight-plan.md    # mark [x] or edit the task
bash overnight.sh        # or re-run /overnight on the branch
```

## Philosophy

- **Maximum decomposition** — More tasks > fewer tasks. Smaller blast radius per worktree.
- **EARS for clarity** — Structured requirements eliminate ambiguity for autonomous agents.
- **Orchestrator ≠ worker** — Separation of concerns. Plan and track vs. execute.
- **Never stop early** — When planned work is done, mine for more. Stop only when there's genuinely nothing left.
- **Main stays clean** — All work on a branch. You decide to PR or discard.
- **Zero friction** — `/overnight PROMPT`. That's the entire interface.

## Inspired By

- [Ralph Loop](https://github.com/tradesdontlie/ralph-loop-skills) — "one task per context window" and "checkboxes as state machines"
- [p-worktree](https://github.com/anthropics/claude-code) pattern — orchestrator + isolated worktree workers via `claude -w`

## License

MIT
