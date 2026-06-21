# Prompt Generation

Generate prompts only after the artifacts are strong enough.

## Prompts artifact

When generating more than one prompt, prefer a single prompts document that contains:

1. `How to use these prompts`
2. `Execution Prompt`
3. `Reviewer Prompt`
4. `QA Prompt`

The `How to use these prompts` section should explain:

- execution prompt -> main implementation agent
- reviewer prompt -> reviewer subagent after each completed phase
- QA prompt -> QA subagent after each completed phase
- reviewer and QA approval are both required before advancing when the workflow is phase-gated

## Execution prompt

The execution prompt should include:

- artifact paths
- execution source of truth
- completion rule
- phase progression rule
- review / QA gate rule
- repo-specific guardrails
- blocker definition
- recovery rule (read progress file, resume from last completed task)
- progress tracking rule (update progress file after each task)
- expected final report

Good execution prompts:

- require phase-by-phase execution
- require verification before review
- require reviewer and QA approval before advancing
- say to continue until all phases/tasks are complete unless blocked
- explain how reviewer and QA prompts are meant to be used with the execution prompt
- include recovery instructions: on start, read the progress file and resume from last completed task
- include progress tracking: update the progress file after each task completion
- define blocker escalation: only escalate to human when no further autonomous progress is possible

## Gated execution default

If the tasks artifact has phases, gates, or subagent-oriented execution, the default execution prompt should require:

- complete current phase tasks
- run current phase verification
- send the completed phase to reviewer
- send the completed phase to QA
- fix findings and rerun verification if either rejects
- only then proceed to the next phase

## Reviewer prompt

The reviewer prompt should include:

- artifact paths
- scope of completed phase
- expectation to review actual code changes, not only the plan
- reachability check: confirm the changed files are actually live (reachable from a route / registered module / entry point), not dead code that merely compiles
- priority on bugs, regressions, missing tests, contract drift, and rule violations
- scope enforcement: flag any changes outside the phase scope
- artifact consistency check: if code diverges from the approved plan, report it as a finding
- partial approval rule: a phase either passes or fails as a whole; no partial approvals unless the plan explicitly allows it
- what to do with out-of-phase findings: log them but do not block the current phase for issues in future-phase scope
- requirement for clear approval or required fixes

## QA prompt

The QA prompt should include:

- artifact paths
- completed phase scope
- required verification commands or evidence
- runtime verification: exercise the actual behavior (drive the UI / hit the endpoint / run the migration against a real DB), covering reset/empty/failure paths — a green build and unit tests are necessary but not sufficient
- focus on regressions, validation gaps, rollout safety, and gate readiness
- scope enforcement: flag any verification gaps for the phase scope
- artifact consistency check: if verification results don't match plan expectations, report it
- partial approval rule: same as reviewer — phase passes or fails as a whole
- what to do with out-of-phase findings: log them but do not block the current phase
- requirement for clear approval or required fixes

## Grounding rules for all prompts

Every generated prompt (execution, reviewer, QA) must include these anti-hallucination rules:

### Execution prompt grounding rules

Include in every execution prompt:

- **Read before write**: Before modifying any file, read its current contents. Do not assume file structure from the plan alone.
- **Edit the live file, not a look-alike**: Before changing a target, confirm it is reachable from an entry point (route / registered module / `main`). A file existing is not proof it is used; two similarly-named files (`Foo` vs `redesign/Foo`) is a classic trap. Zero importers and not itself an entry point ⇒ stop and flag it instead of editing it.
- **Read before reference**: Do not reference files, functions, or contracts in the progress file or reports unless you have read them in this session.
- **Show your work**: When updating the progress file, include the actual command run and a summary of its output. “Tests pass” without output is not acceptable.
- **No invented paths**: If a file path from the plan does not exist, report it as a finding. Do not create files at assumed paths without verifying the plan intended it.
- **No assumed patterns**: If the plan says “follow the existing pattern in X,” read X first. Do not infer patterns from training data.

### Reviewer prompt grounding rules

Include in every reviewer prompt:

- **Independent read**: Read every changed file yourself. Do not rely on the execution agent's description of what changed.
- **Verify claims**: If the execution agent claims tests pass, check the progress file for evidence (command + output). If no evidence exists, flag it.
- **Check file existence**: Verify that all file paths referenced in the execution agent's report actually exist in the repo.
- **Check reachability**: Verify the changed files are live — reachable from a route / registered module / entry point. A change to dead code compiles, passes review on its own terms, and ships nothing. Flag any edited file you cannot trace to an entry point.
- **Flag unsupported claims**: If the execution report references behavior, contracts, or patterns without file:line evidence, flag it as unverified.

### QA prompt grounding rules

Include in every QA prompt:

- **Run independently**: Run verification commands yourself. Do not accept the execution agent's claimed output.
- **Runtime, not just build**: A green build / tests / lint is necessary but not sufficient for a user-facing change. Exercise the actual behavior at runtime and observe the result, covering reset/empty/failure paths — not only the happy path. If you cannot run it, mark the runtime check as an OPEN gap; do not approve on build-green alone.
- **Verify evidence**: Check that the progress file contains actual command output, not just status labels.
- **Regression check**: Run the project's existing test suite (or the relevant subset) independently, not just the tests the execution agent says it ran.
- **Flag missing evidence**: If any verification step in the tasks file has no corresponding evidence in the progress file, reject the phase.

## Prompt writing principles

- Prefer behavior rules over persona rules.
- Avoid brand-roleplay instructions like “engineer from X company.”
- Use operational rules such as:
  - preserve invariants
  - do not widen scope
  - do not stop at partial progress
  - fix review findings before advancing

## Standard completion rule

Use language equivalent to:

"Do not stop after a single phase, partial implementation, or intermediate progress report. Continue until every phase and task is complete unless a real blocker requires human input."

## Orchestration instructions

When generating prompts for orchestrated execution, include a `How to use these prompts` section that explains:

- The main agent acts as orchestrator and does not do execution work.
- The orchestrator should load any available orchestration skills the user has for coordinating sub-agents, and use those alongside the built-in orchestration behavior in this skill.
- The execution prompt is passed to the execution sub-agent.
- After each phase completion, the reviewer prompt is passed to the reviewer sub-agent with the completed phase scope.
- After reviewer approval, the QA prompt is passed to the QA sub-agent with the completed phase scope.
- If either rejects, findings go back to the execution sub-agent for the same phase.
- Only after both approve does the orchestrator advance to the next phase.
- The orchestrator maintains the progress file and enforces gate sequencing.
- If sub-agents are unavailable (or the change does not justify their cost), the same prompts still apply — run the execution / reviewer / QA roles sequentially in one agent that changes hats, per "Single-agent fallback" in `SKILL.md`. The reviewer/QA prompts then become self-applied checklists; honor their independence rules by re-deriving every claim from freshly re-read code and real runtime output, not recollection.

## Recovery instructions for execution prompts

Every execution prompt must include:

- **Progress file path**: `docs/plans/YYYY-MM-DD-<topic>-progress.md`
- **On start**: read the progress file. If it exists, resume from the next incomplete task. Do not re-execute completed tasks unless the progress file indicates a rejection requiring rework.
- **On task completion**: update the progress file with the completed task, verification status, and any findings.
- **On crash/restart**: the next agent session reads the progress file and continues. No work is lost.
- **On blocker**: record the blocker in the progress file. Continue with non-blocked tasks in the same phase if any are independent. Escalate to the human only when no further autonomous progress is possible.

## Standard blocker rule

Use language equivalent to:

"A blocker means a missing decision, broken environment, or contradiction in the approved artifacts that cannot be safely resolved from repo context. Anything else is part of the work and should be resolved by the agent."
