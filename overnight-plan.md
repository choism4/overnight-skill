# Overnight Plan: Resilience & Self-Improvement for overnight-skill

Goal: Add exponential backoff retry strategy, session monitoring, and continuous self-improvement capabilities to the overnight skill.

---

## Foundation

- [x] **Create CLAUDE.md for overnight-skill repo** `overnight-1`
  - THE SYSTEM SHALL have a CLAUDE.md describing the project structure, conventions, and build/test commands
  - Acceptance: `test -f CLAUDE.md && grep -q "SKILL.md" CLAUDE.md`
  - Files: CLAUDE.md

## Exponential Backoff Retry

- [x] **Add exponential backoff retry strategy to SKILL.md worker dispatch** `overnight-2`
  - WHEN a worker task fails its acceptance command THE SYSTEM SHALL retry with exponential backoff (2s, 4s, 8s) before marking BLOCKED
  - Acceptance: `grep -q "exponential backoff" SKILL.md && grep -q "backoff" SKILL.md`
  - Files: SKILL.md

- [x] **Document backoff strategy in README** `overnight-3`
  - WHEN the README describes worker retry behavior THE SYSTEM SHALL explain the exponential backoff pattern with timing details
  - Acceptance: `grep -q "backoff" README.md && grep -q "2s.*4s.*8s" README.md`
  - Files: README.md

## Session Monitoring

- [x] **Add progress reporting between tasks in SKILL.md** `overnight-4`
  - WHILE the orchestrator loop is executing THE SYSTEM SHALL print a progress line after each task showing elapsed time, tasks completed/remaining, and current success rate
  - Acceptance: `grep -q "elapsed" SKILL.md && grep -q "progress" SKILL.md`
  - Files: SKILL.md

- [x] **Add progress reporting to README documentation** `overnight-5`
  - WHEN the README describes the execution phase THE SYSTEM SHALL include an example of the progress output format
  - Acceptance: `grep -q "Progress:" README.md`
  - Files: README.md

## Stale Reference Cleanup

- [x] **Remove stale overnight.sh reference from README** `overnight-6`
  - THE SYSTEM SHALL not reference overnight.sh in any documentation since it was removed
  - Acceptance: `! grep -q "overnight.sh" README.md`
  - Files: README.md

- [x] **Remove stale p-worktree reference from README** `overnight-7`
  - THE SYSTEM SHALL update the Inspired By section to reference Agent tool worktree isolation pattern instead of p-worktree
  - Acceptance: `! grep -q "p-worktree" README.md && grep -q "Agent tool" README.md`
  - Files: README.md

## Orchestrator Resilience

- [x] **Add orchestrator self-recovery guidance to SKILL.md** `overnight-8`
  - IF the orchestrator session is interrupted or context becomes degraded THEN THE SYSTEM SHALL document how to resume from overnight-plan.md state
  - Acceptance: `grep -q "Resume" SKILL.md && grep -q "recovery" SKILL.md`
  - Files: SKILL.md

- [x] **Add resume documentation to README** `overnight-9`
  - WHEN the README describes debugging THE SYSTEM SHALL include instructions for resuming an interrupted overnight session from plan state
  - Acceptance: `grep -q "resume" README.md && grep -q "interrupted" README.md`
  - Files: README.md
