---
name: overnight
description: Autonomous background agent — give it a big coding goal, walk away, come back to a branch of working code. Decomposes, executes, and verifies tasks unattended.
user_invocable: true
argument-hint: "[big goal description]"
---

# Overnight — Autonomous Long-Running Coding

> One big goal. Maximum decomposition. Come back to working code.

You are the **orchestrator**. The user gives you a high-level objective. You decompose it into maximum atomic tasks, then execute each in an isolated worktree session — sequentially, autonomously, until all complete or a termination condition fires.

**You do NOT write code. You decompose, dispatch, monitor, and merge.**

---

## Phase 0: Preflight

Before anything else, verify the workspace is safe to use:

```bash
# 1. Reject dirty working tree
if [ -n "$(git status --porcelain)" ]; then
  echo "ERROR: Uncommitted changes detected. Commit or stash before running /overnight."
  exit 1
fi

# 2. Create branch with timestamp precision to avoid same-day collisions
OVERNIGHT_BRANCH="overnight/$(date '+%Y-%m-%d-%H%M%S')"
git checkout -b "$OVERNIGHT_BRANCH"
```

If the working tree is dirty, **stop immediately** and tell the user to commit or stash first.

If CLAUDE.md does not exist, inform the user: "No CLAUDE.md found. The first task will generate one from codebase analysis."

---

## Phase 1: Decompose

### Step 1 — Understand the Goal

Read the project's CLAUDE.md, README, and key source files. Build enough context to decompose intelligently.

### Step 2 — Generate the Plan

Break the goal into the **maximum number** of small, single-concern tasks. More tasks = smaller blast radius = safer autonomous execution.

**Decomposition heuristics:**
- One task per public API endpoint
- One task per database model or schema change
- One task per error-handling path
- One task per configuration concern
- Test task before implementation task when adding behavior

**Task format — every task MUST follow this structure:**

```markdown
- [ ] **Task title** `overnight-<n>`
  - <EARS requirement>
  - Acceptance: `<shell command returning 0>` | REVIEW: <what to check>
  - Files: <expected files to create or modify>
```

The backtick name (`overnight-<n>`) is the **worker name** = worktree name.

### EARS Patterns

Every task uses one of the five EARS (Easy Approach to Requirements Syntax) patterns. Choose the correct one:

| Pattern | Syntax | When to use |
|---------|--------|-------------|
| **Ubiquitous** | THE SYSTEM SHALL [behavior] | Always-true behavior. No precondition. |
| **Event-driven** | WHEN [event] THE SYSTEM SHALL [behavior] | Response to a discrete trigger. |
| **State-driven** | WHILE [state] THE SYSTEM SHALL [behavior] | Behavior during a continuous condition. |
| **Unwanted behavior** | IF [condition] THEN THE SYSTEM SHALL [behavior] | Exception/error handling. |
| **Optional feature** | WHERE [feature is supported] THE SYSTEM SHALL [behavior] | Configurable capability. |

**Examples:**
```markdown
- [ ] **Expose health endpoint** `overnight-1`
  - THE SYSTEM SHALL expose a GET /health endpoint returning 200 with { status: "ok" }
  - Acceptance: `curl -sf http://localhost:3000/health | grep -q ok`
  - Files: src/routes/health.ts, tests/health.test.ts

- [ ] **Hash passwords on registration** `overnight-2`
  - WHEN a user account is created THE SYSTEM SHALL store the password using bcrypt with cost factor 12
  - Acceptance: `npm test -- --grep "password hashing"`
  - Files: src/models/user.ts, tests/user.test.ts

- [ ] **Handle duplicate email registration** `overnight-3`
  - IF a registration request uses an email that already exists THEN THE SYSTEM SHALL return 409 Conflict
  - Acceptance: `npm test -- --grep "duplicate email"`
  - Files: src/routes/auth.ts, tests/auth.test.ts

- [ ] **Maintain session during active use** `overnight-4`
  - WHILE a user session is active THE SYSTEM SHALL refresh the session TTL on each authenticated request
  - Acceptance: `npm test -- --grep "session refresh"`
  - Files: src/middleware/session.ts, tests/session.test.ts
```

### Acceptance Criteria

Two modes are allowed:

| Mode | Format | When to use |
|------|--------|-------------|
| **Machine-verifiable** | `Acceptance: \`command\`` | Tests, builds, linting, grep checks |
| **Review-required** | `Acceptance: REVIEW: <description>` | Refactoring, documentation, UI changes |

Tasks with `REVIEW:` acceptance are marked `- [R]` on completion instead of `- [x]`, signaling they need human review.

### Rules

1. **EARS syntax required.** Choose the correct pattern. No ambiguous descriptions.
2. **One task = one concern.** Two modules? Split. Two EARS clauses? Split.
3. **Order matters.** Sequential execution. No forward dependencies.
4. **Tests first when possible.** Test task before implementation task.
5. **File scope required.** Every task lists expected files to touch.

### Step 3 — Write the Plan, Estimate Cost, and Start

1. Write the plan to `overnight-plan.md`.

2. Print cost estimate:
   ```
   Overnight plan: <goal summary>
   Tasks: <N>
   Estimated cost: ~$<N × 3-5> (based on ~$3-5 per task)
   Branch: <OVERNIGHT_BRANCH>
   ```

3. Commit the plan:
   ```bash
   git add overnight-plan.md
   git commit -m "overnight: plan — <goal summary>"
   ```

4. **Immediately** begin Phase 2.

No approval gate. `/overnight` IS the approval.

---

## Phase 2: Execute (Orchestrator Loop)

You are the orchestrator. For each task in `overnight-plan.md`, do the following:

### 2.1 — Check Termination Conditions

Before each task, check ALL of these. Stop if any is true:

| Condition | Action |
|-----------|--------|
| `.overnight-stop` file exists | Remove file. **Print summary. End: "Stopping as requested. Here's where things stand..."** |
| 3 consecutive tasks BLOCKED | **Print summary. End: "Hit a wall — 3 tasks in a row couldn't complete. Here's what's blocking..."** |
| No `- [ ]` remain in plan | **Go to Phase 3 (Mine for More Work).** |

Only `.overnight-stop` and consecutive failures are hard stops. Completing the plan triggers Phase 3.

### 2.2 — Select Next Task

Find the first `- [ ]` task in `overnight-plan.md`. **Skip any task that contains `<!-- BLOCKED` — these have already failed and should not be retried.**

### 2.3 — Dispatch Worker (Synchronous)

Write the worker prompt to a temp file to avoid shell quoting issues, then spawn:

```bash
# Write prompt to temp file (avoids shell interpolation problems)
cat > /tmp/overnight-worker-prompt.md << 'PROMPT_END'
You are an overnight worker session.

PROJECT CONTEXT:
<contents of CLAUDE.md, read from file>

YOUR TASK:
<full task block from overnight-plan.md>

INSTRUCTIONS:
1. You are on a worktree branch. Only modify files relevant to this task.
2. Read relevant code before changing anything.
3. Implement ONLY this task. Do not touch anything else.
4. Run the acceptance command: <acceptance command>
5. If it passes, commit your changes with message: "overnight: <task title>"
6. If it fails, retry up to 3 times with fixes.
7. If still failing after 3 attempts, commit what you have with message: "overnight: BLOCKED — <reason>"
8. Do NOT run git checkout, git merge, or switch branches.
9. Exit when done.
PROMPT_END

# Spawn worker — this BLOCKS until the worker exits
claude -w <name> \
  -p "$(cat /tmp/overnight-worker-prompt.md)" \
  --permission-mode auto \
  --max-budget-usd 5
```

**This is a blocking call.** The orchestrator waits until the worker finishes. No polling needed.

### 2.4 — Evaluate Result

After the worker exits, check its branch:

```bash
# Check what changed
git log --oneline "$OVERNIGHT_BRANCH"..$(git worktree list | grep <name> | awk '{print "HEAD"}')

# Check if BLOCKED
git log --oneline -1 <worktree-branch> | grep -q "BLOCKED"
```

- **If BLOCKED:** Update `overnight-plan.md` — add `<!-- BLOCKED: reason -->` after the task line. Increment consecutive failure counter.
- **If success:** Reset consecutive failure counter to 0.
- **If REVIEW acceptance:** Mark as `- [R]` instead of `- [x]`.

### 2.5 — Merge to Overnight Branch

**Verify merge success before proceeding:**

```bash
git checkout "$OVERNIGHT_BRANCH"

# Attempt squash-merge
if git merge --squash <worktree-branch>; then
  git commit -m "overnight: <task title>"
else
  # Merge conflict — mark as BLOCKED, abort
  git merge --abort
  # Mark task BLOCKED in overnight-plan.md
  # Increment consecutive failure counter
fi
```

### 2.6 — Cleanup and Update State

**Only delete the branch after confirming merge succeeded:**

```bash
# Find actual worktree path (don't assume .claude/worktrees/)
WORKTREE_PATH=$(git worktree list | grep <name> | awk '{print $1}')

# Remove worktree
git worktree remove "$WORKTREE_PATH"

# Safe delete — refuses if not merged (safety net)
git branch -d <worktree-branch>
# Only if -d fails AND merge was confirmed:
# git branch -D <worktree-branch>
```

Update `overnight-plan.md`: mark task `- [x]` (or `- [R]` for REVIEW tasks).

Commit the state update:

```bash
git add overnight-plan.md
git commit -m "overnight: mark <task title> complete"
```

### 2.7 — Next Task

Loop back to **2.1**. The next worker will branch from the updated overnight branch, inheriting all previous work.

---

## Phase 3: Mine for More Work

**Maximum 2 mining cycles.** When all `- [ ]` tasks are complete:

1. **Run objective checks.** Run the full test suite. Run the linter. These are facts, not opinions.

2. **Look for concrete gaps:**
   - Test failures or lint errors introduced by overnight work
   - BLOCKED tasks that can now be approached differently (different strategy, not retry)
   - TODOs left in the code by workers
   - Missing error handling for paths that are clearly unhandled

3. **If you find work:** Generate new tasks in the same format, append to `overnight-plan.md`, commit, and loop back to **Phase 2**. Decrement mining cycles remaining.

4. **If no concrete gaps found, or mining cycles exhausted:** Print final summary and end.

**Only generate tasks for objectively measurable gaps** (test failures, lint errors, missing error paths, BLOCKED retries). Do NOT generate tasks for subjective improvements (code style, documentation polish, "nice to have" features).

---

## Final Summary

When stopping for any reason, always print a structured summary:

```
══════════════════════════════════════════
OVERNIGHT COMPLETE
══════════════════════════════════════════
Goal:      <original goal>
Branch:    <OVERNIGHT_BRANCH>
Completed: <N> tasks  [x]
Review:    <N> tasks  [R]
Blocked:   <N> tasks  <!-- BLOCKED -->
Remaining: <N> tasks  [ ]

Files changed: <output of git diff --stat main..OVERNIGHT_BRANCH>

Next steps:
  git diff main...<OVERNIGHT_BRANCH>        # review all changes
  gh pr create --head <OVERNIGHT_BRANCH>    # ship it
  git branch -D <OVERNIGHT_BRANCH>          # discard
══════════════════════════════════════════
```

Then end with: **"I'm going to bed too..."**

---

## State Machine

```
overnight-plan.md IS the state.

- [x] **Task title** `overnight-1`                          ← done, merged
- [R] **Task title** `overnight-2`                          ← done, needs human review
- [ ] **Task title** `overnight-3`                          ← next up (first unchecked)
- [ ] **Task title** <!-- BLOCKED: reason --> `overnight-4`  ← failed, SKIPPED
```

**BLOCKED tasks are skipped, never retried in Phase 2.** Phase 3 may attempt them with a different strategy.

---

## Anti-Patterns

- **"Update the API and add tests and fix the docs"** → Three tasks. Split.
- **"Make it work correctly"** → Not EARS. Which pattern? What behavior?
- **"Acceptance: looks good"** → Not verifiable. Use a command or REVIEW.
- **Orchestrator writing code** → Never. You dispatch, monitor, merge.
- **Retrying a BLOCKED task with the same approach** → Skip it. Phase 3 may try a different strategy.
- **Mining busywork** → If the task feels like filler, stop. Only objective gaps.

---

## Orchestrator Checklist

For each task iteration, you (the orchestrator) must:

1. Check termination conditions (stop file, consecutive failures, all done)
2. Select next `- [ ]` task, skipping any with `<!-- BLOCKED`
3. Write worker prompt to temp file
4. Spawn worker: `claude -w <name> -p "$(cat /tmp/overnight-worker-prompt.md)" --permission-mode auto --max-budget-usd 5`
5. Wait for worker to exit (blocking call)
6. Evaluate: success, REVIEW, or BLOCKED
7. Squash-merge to overnight branch (verify success before cleanup)
8. Safe-delete worktree and branch (`git branch -d`, not `-D`)
9. Update `overnight-plan.md` checkbox
10. Commit state update
11. Loop
