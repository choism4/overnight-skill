# Overnight

> "One big goal. Maximum decomposition. Come back to working code."
> "If you can't write a completion command, you don't understand the task yet."
> "The orchestrator never writes code. It decomposes, dispatches, monitors, and merges."

## What is Overnight?

Overnight turns a big goal into a fully executed codebase — autonomously, for hours, while you're away.

1. **Maximum decomposition** — Breaks your goal into the highest possible number of atomic, testable tasks
2. **EARS-style requirements** — Every task uses WHEN/IF/WHILE...SHALL syntax. No ambiguity survives
3. **Machine-verifiable completion** — Every task has a shell command that proves it's done
4. **Isolated worktree per task** — Each task runs in its own `claude -w` session. Clean git isolation
5. **Proactive work discovery** — When planned tasks are done, it actively mines for more meaningful work
6. **Zero user intervention** — One command. Walk away. Come back to a branch full of working code

## Install

```bash
npx skills add jadenkim/overnight-skill
```

## Usage

```
/overnight Add user authentication with email/password, OAuth, and session management
```

That's it. Go do something else.

Come back and check:

```bash
git log --oneline overnight/2026-03-18   # one squash-merge per task
cat overnight-plan.md                     # [x] vs [ ] vs BLOCKED

# Happy? Create a PR
gh pr create --head overnight/2026-03-18

# Not happy? Discard
git branch -D overnight/2026-03-18
```

## How It Works

```
┌──────────────────────────────────────────────────────────────────────┐
│                         OVERNIGHT                                    │
│                                                                      │
│  /overnight "Add auth with OAuth and sessions"                      │
│           │                                                          │
│           ▼                                                          │
│  PHASE 1: DECOMPOSE                                                 │
│  ├── Create branch: overnight/YYYY-MM-DD                            │
│  ├── Read CLAUDE.md + codebase                                      │
│  ├── Break goal into maximum atomic tasks                           │
│  ├── Apply EARS syntax (WHEN/IF/WHILE...SHALL)                      │
│  ├── Assign worker name: overnight-1, overnight-2, ...              │
│  └── Write overnight-plan.md + commit                               │
│           │                                                          │
│           ▼  (immediately)                                           │
│                                                                      │
│  PHASE 2: EXECUTE (orchestrator loop)                               │
│  ┌─────────────────────────────────────────────────┐                │
│  │  For each unchecked task:                       │                │
│  │  ├── Check termination conditions               │                │
│  │  ├── Spawn worker:                              │                │
│  │  │     claude -w overnight-N                    │                │
│  │  │       --session-id overnight-N               │                │
│  │  │       --permission-mode auto                 │                │
│  │  ├── Poll: jobs -l + git log                    │                │
│  │  ├── Evaluate: success or BLOCKED               │                │
│  │  ├── Squash-merge → overnight branch            │                │
│  │  ├── Cleanup worktree + update plan [x]         │                │
│  │  └── Next task                                  │                │
│  └─────────────────────────────────────────────────┘                │
│           │  all tasks done                                          │
│           ▼                                                          │
│  PHASE 3: MINE FOR MORE WORK                                       │
│  ├── Review code, run tests, run linter                             │
│  ├── Look for: edge cases, error handling, security,                │
│  │   integration tests, BLOCKED retries, TODOs                      │
│  ├── Found work → append tasks → back to Phase 2                   │
│  └── Nothing left → "I'm going to bed too..."                       │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

## Architecture

```
┌─────────────────────┐
│    ORCHESTRATOR      │    Current Claude session.
│    (this session)    │    Never writes code.
│                      │    Decomposes, dispatches,
│                      │    monitors, merges.
└──────────┬──────────┘
           │ claude -w overnight-1
           ▼
┌─────────────────────┐     ┌─────────────────────┐
│  WORKER overnight-1 │     │  WORKER overnight-2  │
│  (isolated worktree) │     │  (isolated worktree) │
│                      │     │                      │
│  Implements ONE task │     │  Implements ONE task │
│  Runs completion cmd │     │  Runs completion cmd │
│  Commits + exits     │     │  Commits + exits     │
└──────────────────────┘     └──────────────────────┘
        (sequential — 2 starts after 1 is merged)
```

- **Worker 2 inherits Worker 1's code** — each worktree branches from the latest overnight branch
- **If a worker fails**, `claude -r overnight-3` resumes it with full context preserved
- **Main is never touched** — all work on `overnight/YYYY-MM-DD` branch

## Task Format

Every generated task follows this structure:

```markdown
- [ ] **Create User model** `overnight-1`
  - WHEN a user record is created THE SYSTEM SHALL store email and bcrypt-hashed password
  - Completion: `npm test -- --grep "User model"`
  - Permissions: always: read/write src/models/ | ask: install deps | never: drop tables
```

| Element | Purpose |
|---------|---------|
| **EARS syntax** (WHEN/IF/WHILE...SHALL) | Eliminates ambiguity. Forces behavioral spec. |
| **Completion command** | Machine-verifiable. Returns 0 or doesn't. |
| **Permission boundaries** | Prevents unattended sessions from doing damage. |

## Termination

| Trigger | What happens |
|---------|-------------|
| `.overnight-stop` file | Hard stop. Report + "I'm going to bed too..." |
| 3 consecutive BLOCKED tasks | Hard stop. Something is fundamentally broken. |
| All planned tasks done | **Not a stop.** Go to Phase 3 — mine for more work. |
| No more meaningful work found | Final stop. "I'm going to bed too..." |

```bash
# Emergency stop (finishes current task first)
touch .overnight-stop
```

## State

`overnight-plan.md` is the entire state machine:

```markdown
- [x] **Create User model** `overnight-1`              ← done, merged
- [x] **Add registration endpoint** `overnight-2`      ← done, merged
- [ ] **Add OAuth flow** `overnight-3`                  ← current task
- [ ] **Add session management** `overnight-4`          ← queued
- [ ] **Add rate limiting** <!-- BLOCKED: reason -->     ← failed, skipped
```

Git history on the overnight branch:

```
overnight/2026-03-18:
  overnight: plan — Add user authentication
  overnight: Create User model
  overnight: mark Create User model complete
  overnight: Add registration endpoint
  overnight: mark Add registration endpoint complete
  ...
```

## Recovery

```bash
# Worker stuck? Resume with context preserved
claude -r overnight-3 -p "The test failed because X. Try Y instead."

# Check what a worker did
git diff --stat overnight/2026-03-18...worktree-overnight-3

# See worker's full history
git log --oneline overnight/2026-03-18..worktree-overnight-3
```

## CLAUDE.md Is Everything

Each worker reads the project's `CLAUDE.md` as its briefing. Overnight quality = CLAUDE.md quality.

**Good CLAUDE.md for overnight:**

```markdown
# Build & Test
npm run build          # TypeScript compilation
npm test               # Jest test suite
npm run lint           # ESLint

# Architecture
src/models/     — Sequelize models, one file per table
src/routes/     — Express route handlers
src/middleware/  — Auth, validation, error handling

# Conventions
- All routes return { data, error, meta } envelope
- Tests use factory functions, not raw fixtures

# Do NOT
- Do not use raw SQL — use Sequelize query builder
- Do not add dependencies without checking package.json first
```

No CLAUDE.md? Overnight's first task will be creating one.

## Philosophy

- **Maximum decomposition** — More tasks > fewer tasks. Smaller blast radius per worktree.
- **Tests are the spec** — If you can't write a completion command, you don't understand the task yet.
- **Orchestrator ≠ worker** — Separation of concerns. Plan and track vs. execute.
- **Never stop early** — When planned work is done, mine for more. Stop only when there's genuinely nothing left.
- **Main stays clean** — All work on a branch. You decide to PR or discard.
- **Zero friction** — `/overnight PROMPT`. That's the entire interface.

## Inspired By

- [Ralph Loop](https://github.com/tradesdontlie/ralph-loop-skills) — "one task per context window" and "checkboxes as state machines"
- [p-worktree](https://github.com/anthropics/claude-code) pattern — orchestrator + isolated worktree workers via `claude -w`

## License

MIT
