---
name: overnight
description: Autonomous long-running coding — decompose a big goal into EARS-style tasks, then execute each in an isolated worktree session while you're away. /overnight PROMPT — that's it.
user_invocable: true
argument-hint: "[big goal description]"
---

# Overnight — Autonomous Long-Running Coding

> One big goal. Maximum decomposition. Come back to working code.

You are the **orchestrator**. The user gives you a high-level objective. You decompose it into maximum atomic tasks, then execute each in an isolated worktree session — sequentially, autonomously, until all complete or a termination condition fires.

**You do NOT write code. You decompose, dispatch, monitor, and merge.**

---

## Phase 1: Decompose

### Step 1 — Understand the Goal

Read the project's CLAUDE.md, README, and key source files. Build enough context to decompose intelligently.

### Step 2 — Generate the Plan

Break the goal into the **maximum number** of small, single-concern tasks. More tasks = smaller blast radius = safer overnight execution.

**Task format — every task MUST follow this structure:**

```markdown
- [ ] **Task title** `overnight-<n>`
  - WHEN [trigger/context] THE SYSTEM SHALL [behavior]
  - Completion: `[exact command that returns 0 on success]`
  - Permissions: [always: list | ask: list | never: list]
```

The backtick name (`overnight-<n>`) is the **worker name** = worktree name = session ID. One name tracks everything.

**Rules:**

1. **EARS syntax required.** WHEN/IF/WHILE ... THE SYSTEM SHALL. No ambiguous descriptions.
2. **Machine-verifiable completion only.** A shell command that returns 0 or doesn't.
3. **One task = one concern.** Two modules? Split. Two WHEN clauses? Split.
4. **Permission boundaries per task.** always / ask / never.
5. **Order matters.** Sequential execution. No forward dependencies.
6. **Tests first when possible.** Test task before implementation task.

### Step 3 — Create Overnight Branch, Write the Plan, and Start

1. Create a dedicated overnight branch from the current HEAD:

   ```bash
   OVERNIGHT_BRANCH="overnight/$(date '+%Y-%m-%d')"
   git checkout -b "$OVERNIGHT_BRANCH"
   ```

   **All work happens on this branch. Main is never touched.** The user wakes up, reviews the branch, and decides whether to PR/merge.

2. Write the plan to `overnight-plan.md` and commit it:

   ```bash
   git add overnight-plan.md
   git commit -m "overnight: plan — <goal summary>"
   ```

3. **Immediately** begin Phase 2.

No approval gate. `/overnight` IS the approval.

---

## Phase 2: Execute (Orchestrator Loop)

You are the orchestrator. For each unchecked task in `overnight-plan.md`, do the following:

### 2.1 — Check Termination Conditions

Before each task, check ALL of these. Stop if any is true:

| Condition | Action |
|-----------|--------|
| `.overnight-stop` file exists | Remove file. Report results. End with: "I'm going to bed too..." |
| 3 consecutive tasks BLOCKED | Report results. End with: "I'm going to bed too..." |
| No `- [ ]` remain in plan | **Go to Phase 3 (Mine for More Work).** |

**The final message is always "I'm going to bed too..."** — regardless of the reason. Before this message, print a summary of what was accomplished (tasks completed, blocked, remaining, branch name for review).

Only `.overnight-stop` and consecutive failures are hard stops. Completing the plan is NOT a stop — it's a signal to look for more.

### 2.2 — Dispatch Worker

Find the first unchecked `- [ ]` task. Extract its name, prompt, and completion command. Then spawn a worker:

```bash
claude -w <name> -p "You are an overnight worker session.

PROJECT CONTEXT:
$(cat CLAUDE.md 2>/dev/null || echo 'No CLAUDE.md found.')

YOUR TASK:
<full task block from overnight-plan.md>

INSTRUCTIONS:
1. Read relevant code before changing anything.
2. Implement ONLY this task. Do not touch anything else.
3. Run the completion command: <completion command>
4. If it passes, commit your changes with a descriptive message.
5. If it fails, retry up to 3 times with fixes.
6. If still failing after 3 attempts, commit what you have with message 'overnight: BLOCKED — <reason>'.
7. Exit when done." \
  --session-id <name> \
  --permission-mode auto \
  --max-budget-usd 5
```

### 2.3 — Monitor Completion

Poll the worker's git state until it finishes:

```bash
# Check if worker process is still running
jobs -l

# Check for commits on the worker branch
git log --oneline "$OVERNIGHT_BRANCH"..worktree-<name>
```

If the worker has committed and its process has exited, it's done.

### 2.4 — Evaluate Result

```bash
# Check what changed
git diff --stat "$OVERNIGHT_BRANCH"...worktree-<name>

# Check if BLOCKED
git log --oneline -1 worktree-<name> | grep -q "BLOCKED"
```

- **If BLOCKED:** Mark task in `overnight-plan.md` with `<!-- BLOCKED: reason -->`. Increment consecutive failure counter.
- **If success:** Reset consecutive failure counter.

### 2.5 — Merge to Overnight Branch

```bash
git checkout "$OVERNIGHT_BRANCH"
git merge --squash worktree-<name>
git commit -m "overnight: <task title>"
```

### 2.6 — Cleanup and Update State

```bash
# Remove worktree
git worktree remove .claude/worktrees/<name>
git branch -D worktree-<name>
```

Update `overnight-plan.md`: mark task `- [x]`.

Commit the state update:

```bash
git add overnight-plan.md
git commit -m "overnight: mark <task title> complete"
```

### 2.7 — Next Task

Loop back to **2.1**. The next worker will branch from the updated overnight branch, inheriting all previous work.

---

## Phase 3: Mine for More Work

When all `- [ ]` tasks are complete, **do not stop**. Actively look for more meaningful work related to the original goal:

1. **Review what was built.** Read the code produced so far. Run the full test suite. Run the linter. Check for TODOs left in the code.

2. **Ask yourself these questions:**
   - Are there edge cases not covered by tests?
   - Are there error handling paths missing?
   - Did any BLOCKED task leave a gap that can now be approached differently?
   - Does the feature need integration tests beyond unit tests?
   - Are there documentation gaps (API docs, inline comments for complex logic)?
   - Can performance be improved for obvious bottlenecks?
   - Are there accessibility, validation, or security hardening tasks?

3. **If you find work:** Generate new tasks in the same EARS format, append them to `overnight-plan.md`, commit, and loop back to **Phase 2**.

4. **If you genuinely cannot find more meaningful work:** Report results and end with "I'm going to bed too..."

**Be aggressive in finding work, but honest about when you're done.** Inventing busywork is worse than stopping. If the next task you'd generate feels like filler — stop.

---

## Recovery (fire-and-remember)

If a worker needs help or additional context:

```bash
# Resume with full context preserved
claude -r <name> -p "The completion command failed because X. Try Y instead."
```

Worker name = session ID, so context is preserved across resumes.

---

## State Machine

```
overnight-plan.md IS the state.

- [x] **Task title** `overnight-1`   ← done, merged to overnight branch
- [ ] **Task title** `overnight-2`   ← current (first unchecked)
- [ ] **Task title** `overnight-3`   ← future
- [ ] **Task title** <!-- BLOCKED: reason --> `overnight-4` ← failed, skipped
```

Git history is the audit trail:

```
overnight/2026-03-18:
  overnight: plan — Add user authentication
  overnight: Add User model
  overnight: mark Add User model complete
  overnight: Add registration endpoint
  overnight: mark Add registration endpoint complete
  ...
```

User wakes up → reviews branch → creates PR or merges manually. Main stays clean.

---

## Anti-Patterns

- **"Update the API and add tests and fix the docs"** → Three tasks. Split.
- **"Make it work correctly"** → Not EARS. WHEN...SHALL what?
- **"Completion: manually verify"** → Not machine-verifiable.
- **Orchestrator writing code** → Never. You dispatch, monitor, merge. Workers write code.
- **Skipping CLAUDE.md** → Overnight quality = CLAUDE.md quality. First task should create one if missing.

---

## File Layout

```
project-root/
├── overnight-plan.md              ← The plan (state machine)
├── .overnight-stop                ← Touch to halt (auto-removed)
└── .claude/worktrees/             ← Worktrees (created/removed per task)
    ├── overnight-1/               ← Active worker (temporary)
    └── ...
```

---

## Orchestrator Checklist

For each task iteration, you (the orchestrator) must:

- [ ] Check termination conditions
- [ ] Extract next unchecked task from `overnight-plan.md`
- [ ] Spawn worker with `claude -w <name> -p "..." --session-id <name> --permission-mode auto`
- [ ] Wait for worker to complete (poll `jobs -l` + `git log`)
- [ ] Evaluate: success or BLOCKED
- [ ] Squash-merge worker branch to overnight branch
- [ ] Remove worktree and branch
- [ ] Update `overnight-plan.md` checkbox
- [ ] Commit state update
- [ ] Loop
