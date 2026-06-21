# Execution Orchestration & Recovery

Load this reference when the user asks to **start execution** (not during brief/plan/tasks shaping). It covers how the orchestrator drives gated execution and how to recover when an agent crashes, stalls, or hits a blocker.

For the no-sub-agent path, see "Single-agent fallback" in `SKILL.md`. The grounding rules referenced throughout are in `references/grounding-rules.md`.

## Orchestrator role

When the user asks to start execution, the main agent acts as orchestrator:

1. Load any user-available orchestration skills for coordinating sub-agents.
2. Use the generated prompts artifact as the source of truth for sub-agent instructions.
3. Pass the execution prompt to the execution sub-agent.
4. After each phase completion, pass the reviewer prompt to the reviewer sub-agent with the completed phase scope.
5. After reviewer approval, pass the QA prompt to the QA sub-agent with the completed phase scope.
6. If either reviewer or QA rejects, pass findings back to the execution sub-agent for the same phase.
7. Only advance the execution sub-agent to the next phase after both reviewer and QA approve.

The orchestrator should not do execution work itself. Its job is coordination, sequencing, and gate enforcement.

## Recovery

Execution agents may crash, time out, or lose context. The skill must support recovery at any point.

### Progress tracking

Maintain a progress file (default `docs/plans/YYYY-MM-DD-<topic>-progress.md`, or your repo's plan-doc convention). Use `references/progress-template.md` as the structure — fill its evidence slots rather than writing free prose, so a missing command/output is visibly unfilled.

The execution agent must update this file after completing each task.

### Recovery rule

When an execution agent starts (or restarts), it must:

1. Read the progress file.
2. Identify the last completed task.
3. Resume from the next incomplete task.
4. Do not re-execute completed tasks unless the progress file indicates a rejection that requires rework.

The execution prompt must include this recovery rule.

### Stall recovery (agent came to rest mid-task)

A subagent can stop mid-task without finishing and without hitting a real blocker — it "comes to rest" partway, often leaving the work half-applied (e.g. a type was changed but its call-sites were not yet updated, sometimes leaving the build still green). This is distinct from both a completion and a blocker, and the orchestrator must actively detect it: a final report that trails off mid-action ("Now I'll update X…"), an unexpectedly low step count, or a progress file that still shows the task incomplete.

When it happens:

1. Do NOT assume the task is done because the agent stopped. Inspect the actual repo state (read the touched files, run the build) to learn exactly how far it got.
2. Dispatch a finisher with the precise remaining delta — what is already applied vs. what is left — so it neither redoes completed work nor re-investigates from scratch.
3. Keep the code in a compiling state across the boundary; if a partial edit left it broken, the finisher's first job is to restore a buildable state.

Mitigate stalls up front: size tasks so each is completable in one focused pass; make the execution prompt insist on meeting the phase's exit criteria before stopping; prefer fewer larger-context agents over many tiny hand-offs for sequential work.

### Blocker escalation

A blocker means a missing decision, broken environment, or contradiction in the approved artifacts that cannot be safely resolved from repo context.

When a blocker is hit:

1. Record it in the progress file.
2. Stop execution on the blocked task.
3. Continue with non-blocked tasks in the same phase if any are independent.
4. Escalate to the human only when no further progress is possible.

The goal is to minimize human interaction during execution. Sub-agents should resolve ambiguity from the artifacts and repo context. Only true blockers — missing decisions, broken environments, or artifact contradictions — should reach the human.
