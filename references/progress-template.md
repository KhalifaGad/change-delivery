# Progress Template

Copy this into `docs/plans/YYYY-MM-DD-<topic>-progress.md` (or your repo's plan-doc location) at the start of execution and fill it in as you go. The point of a template over prose: an empty **evidence** slot is visibly unfilled, so "tests pass ✓" with no command/output can't quietly substitute for the real thing. Per `references/grounding-rules.md`, a verification step with no evidence here is an automatic rejection.

```markdown
# <topic> — execution progress

- Mode: Lite | Standard | Full
- Plan: docs/plans/YYYY-MM-DD-<topic>-implementation-plan.md
- Tasks: docs/plans/YYYY-MM-DD-<topic>-tasks.md
- Current phase: <n>
- Last completed task: <phase.task>
- Status: in-progress | blocked | phase-complete | done

## Phase <n> — <name>

### Task <n.m> — <title>
- State: not-started | in-progress | done | rejected-rework
- Files touched:
- What changed: <diff or relevant section — not "file updated">
- Verification command(s):
  ```
  <exact command run>
  ```
- Verification output: <pass/fail + key lines; not "all green">
- Runtime check: <what real path was exercised + observed result, incl. reset/empty/failure — or OPEN: reason it could not run>
- Reviewer disposition: pending | approved | rejected — <findings>
- QA disposition: pending | approved | rejected — <findings>

## Blockers
- <none, or: blocker description + what decision/environment is missing>
```

Rules for filling it in:

- Update after **each task**, not each phase.
- Every "Verification output" line must be backed by the command above it and its real output — never a bare status label.
- A user-facing change MUST have a non-empty "Runtime check" line (or an explicit `OPEN:` with the reason). Build/test/lint green alone does not fill this slot.
- Reviewer/QA dispositions stay `pending` until that agent (or hat) has independently verified — not when the execution agent claims done.
