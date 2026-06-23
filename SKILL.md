---
name: "change-delivery"
description: "Use when a user needs help turning a business change, technical change, or refactor into execution-ready delivery artifacts (shaped brief, implementation plan, task breakdown) and/or running the implementation through gated, independently reviewed and QA'd multi-agent execution with anti-hallucination grounding. Not for trivial one-file fixes unless the user explicitly wants the full workflow."
---

# Change Delivery

Use this skill to standardize the workflow we used for large brownfield changes:

1. shape the brief
2. create the implementation plan
3. split the work into execution tasks
4. generate execution prompts

This skill is collaborative and standards-enforcing. It should challenge weak inputs, not just template-fill.

## When to use

Use this skill when the user wants any of the following:

- help shaping a change before implementation
- a strong planning prompt for a coding agent
- a brownfield implementation plan
- a task-split artifact for subagent execution
- execution, reviewer, or QA prompts
- a standardized workflow for business changes, technical changes, or refactors

Do not use this skill for tiny fixes, narrow local refactors, or obvious one-file work unless the user explicitly wants the full workflow.

## Cost, scale, and mode

Gated multi-agent execution is powerful but expensive: a single feature can spawn dozens of subagents (planner, per-phase execution, reviewer, QA, rework loops). Set the expectation before starting, and scale the ceremony to the change. Pick a mode up front and tell the user which one you're running:

- **Lite** — small, low-risk, or well-understood changes. Brief + a short task list, a single execution pass, one self-review against the grounding rules (including the reachability and runtime checks). No separate reviewer/QA subagents; skip artifacts that add no signal.
- **Standard** — a normal feature/refactor. Full brief → plan → tasks → prompts, gated execution. You MAY collapse reviewer+QA into one agent for low-risk phases (as long as it does both the independent code read AND the runtime check), and reserve the full reviewer+QA pair for the risky phases (migrations, shared-component edits, anything touching money, auth, or multi-tenant scoping).
- **Full** — high-risk, broad, or compliance-sensitive work. Everything: full artifacts, reviewer AND QA on every phase, adversarial verification.

Cost knobs that don't sacrifice the method:
- Collapse reviewer+QA into one agent for a phase when risk is low.
- Gate only the phases that carry real risk; let mechanical phases self-verify against the grounding rules.
- Prefer fewer, larger-context agents over many tiny ones for sequential work — it also shrinks the mid-task-stall surface (see stall recovery in `references/execution-orchestration.md`).
- **Cut cold re-discovery, not coverage.** The dominant hidden cost is every fresh agent re-reading large files and all artifacts from zero. Carry *facts* forward (a code map, pinned `file:line` anchors, the frozen contract) so agents verify instead of re-discover; scope rework and re-gates to the delta; reuse a warm environment; use targeted typecheck/test commands and reserve the full build+suite for the QA regression pass. None of this weakens the independent runtime verification — see "Context economy" and "Rework and re-gate" in `references/execution-orchestration.md`.

Do not silently run Full mode on a change that warranted Lite — the user pays for it.

## Core behavior

- Inspect repo context before shaping artifacts.
- Classify the change as `business`, `technical`, or `refactor`.
- After repo inspection, derive a project-specific concern map from repo docs and code.
- Ask one question at a time by default.
- Ask up to 3 questions only when the input is too weak to proceed efficiently.
- Push back on vague goals, weak constraints, missing invariants, or missing rollout assumptions.
- Do not assume omitted project-specific concerns are irrelevant just because the user did not name them.
- Do not advance to the next artifact until the current artifact is strong enough.
- Write files automatically when certainty is effectively complete.
- Keep the workflow brownfield-aware and grounded in actual repo constraints.
- Prefer a reusable core workflow plus project-specific constraints from repo docs.

## Artifact sequence

Always move in this order unless the user explicitly asks to stop earlier:

1. brief
2. implementation plan
3. tasks
4. prompts

If the user already has one artifact, review it critically and continue from there instead of starting over.

## Step 0: Understand the project

Before classifying the change, establish project context:

- Ask the user what type of project this is (single app, multi-app, library, etc.) if not obvious from the repo structure.
- Ask whether there are testing constraints or infrastructure limitations that would affect planning (e.g., no integration tests, slow CI, limited staging environment).
- These answers feed into the brief and plan. Do not assume defaults.

## Step 1: Classify the change

Start by classifying the work:

- `business`: domain behavior, user-facing workflow, product shape, API/domain changes driven by business intent
- `technical`: technical capability, infrastructure, platform behavior, architecture/platform improvements, operational or reliability work
- `refactor`: internal restructuring where preserved behavior is the main constraint

If unclear, ask one question to classify it.

### Hybrid changes

Many changes straddle two types (e.g., a business feature requiring a technical migration, or a refactor motivated by a business need). When the change is clearly hybrid:

- Classify with the primary type (the one driving the change).
- Load the primary type's reference file.
- Also load the secondary type's reference file.
- Pull the planning emphasis and failure modes from both into the plan.

If the secondary type's concerns would materially change the plan shape, treat it as a first-class input, not an afterthought.

For clearly-hybrid changes, don't over-invest in the type-reference ceremony: in practice the **project concern map** (derived from the repo below) usually shapes the plan more than the per-type reference files do. Load the refs for their failure-mode checklists, but let the concern map and repo grounding do the heavy lifting.

Then load the relevant reference(s):

- `references/business-change.md`
- `references/technical-change.md`
- `references/refactor-change.md`

Always also load:

- `references/artifact-standards.md`
- `references/prompt-generation.md`
- `references/grounding-rules.md`

## Project concern derivation

Before shaping the brief, inspect the project and build a concern map for this change.

The concern map is not fixed across repositories. Infer it from:

- policy docs such as `AGENTS.md`, project context files, or other repo-level instructions
- package manifests and project layout
- routing/layout structure
- authentication/session code
- API client setup
- translation/localization setup
- rendering model
- deployment/proxy/dev-topology files
- any other subsystem that this repo clearly treats as important

The purpose is to discover what this specific project considers cross-cutting and high-risk.

Then compare the user’s change against that concern map.

If the change likely affects a project-important concern that the user did not mention, ask about it before planning.

Examples:

- if a repo has strong localization infrastructure and the change moves UI or app boundaries, ask how localization assets and runtime behavior should be handled
- if a repo uses host-based routing or proxy logic and the change moves app boundaries, ask how host/routing behavior should be preserved
- if a repo has strong PWA, SSR, SEO, or auth/session constraints, ask about those when the change likely touches them

## Step 2: Shape the brief

The brief is the input artifact for planning. Do not let it stay vague.

Before asking the user anything, compare the change against the project-specific concern map you derived from the repo.

If a concern is clearly important in this project and the change likely affects it, ask about it instead of silently planning around it.

Check for:

- goal
- non-goals
- invariants / what must not change
- current-state assumptions
- intended scope
- success criteria
- rollout constraints
- testing constraints (if any were identified in Step 0)
- known conflicts or drift

If these are weak or missing:

- ask one focused question at a time
- if the input is very weak, ask up to 3 short questions in one turn

Challenge weak input directly. Examples:

- if the user describes the target state but not the current state, ask for the current system shape
- if the user describes desired behavior but not constraints, ask what must remain unchanged
- if the user wants a refactor but cannot define preserved behavior, stop and tighten that first
- if the change touches app or module boundaries, ask about the project-important concerns revealed by the repo scan instead of assuming defaults
- if the repo makes a concern look important, do not wait for the user to mention it

When the brief is strong enough:

- write it to `docs/plans/YYYY-MM-DD-<topic>.md` if no file exists yet
- if a file already exists, update that file in place only when certainty is high

## Step 2.5: Rollback assessment

After the brief is shaped, ask the user: **should the plan include a rollback section?**

If yes, load `references/rollback-approaches.md` and walk through the rollback questioning flow:

1. Does this change touch persistent state (database schema, stored data, file storage)?
2. Does this change affect APIs consumed by other services or clients?
3. Does this change alter deployment artifacts (containers, static assets, infrastructure)?
4. Does this change modify configuration or environment variables?
5. Does this change involve long-running workflows, pipelines, or state machines?
6. Does this change affect frontend assets served to end users?

For each "yes," identify the matching rollback category from the reference and ask follow-up questions to select the right approach. Use the trade-offs and failure modes from the reference to challenge weak rollback assumptions.

The selected rollback approaches become a required section in the implementation plan.

If the user says no to rollback, note it in the brief as an explicit non-goal and move on.

## Step 3: Create the implementation plan

Use the brief as the source artifact. Convert it into a brownfield implementation plan.

The plan must include:

- context summary
- key design decisions
- phased implementation plan
- per-phase file scope
- verification per phase
- risks / migration / rollback notes
- open decisions if any remain

The plan must be:

- repo-grounded
- execution-oriented
- explicit about dependencies
- suitable for subagent-driven execution
- precise about exact file paths where they are reasonably knowable
- sparing with line references, using them only for risky integration points or existing contracts that must be preserved

If the plan is generated by another agent, review it critically before accepting it. Tighten gaps before moving on.

Write the implementation plan when it is strong enough. Prefer:

- `docs/plans/YYYY-MM-DD-<topic>-implementation-plan.md`

## Step 4: Split into tasks

Convert the implementation plan into execution-ready tasks.

The tasks artifact must:

- organize by phase
- break work into small executable tasks
- define prerequisites
- define file scope
- define verification
- define stop/review gates
- mark safe parallelization only where ownership is disjoint
- express cross-phase dependencies at the task level when needed

### Cross-phase task dependencies

Phase-level gates are sometimes too coarse. When a task in a later phase depends on a specific earlier task's output (not the whole phase), express that dependency explicitly:

- Use `depends_on: [1.2, 1.4]` or equivalent notation to link tasks across phases.
- Do not rely on phase gates as a substitute for task-level ordering.
- Cross-phase dependencies should be the exception, not the rule. If there are many, the phase boundaries may be wrong.

The tasks file is not another plan. It is an execution artifact.

If generated by another agent, review it critically for:

- oversized tasks
- soft gates
- shared-file conflicts
- fake verification
- missing operational preconditions

Write the tasks file when it is strong enough. Prefer:

- `docs/plans/YYYY-MM-DD-<topic>-tasks.md`

## Step 5: Generate prompts

Generate prompts only after the brief, plan, and tasks are strong enough.

By default generate:

- execution prompt
- reviewer prompt
- QA prompt

These prompts must be grounded in the real artifacts and workflow, not generic “do a good job” text.

When generating multiple prompts, write them into a single prompts artifact that starts with a short `How to use these prompts` section.

If the tasks artifact is phase-gated or intended for subagent execution, default to gated execution:

- the execution prompt must tell the execution agent to complete one phase at a time
- run phase verification
- get reviewer approval
- get QA approval
- fix findings and resubmit the same phase if needed
- only then move to the next phase

Use `references/prompt-generation.md` for output shape.

If the user only wants one prompt, generate only that one.

## File-writing rule

Write artifacts automatically when certainty is effectively complete.

Use this test:

- the artifact has the required sections
- the key decisions are no longer materially ambiguous
- the user’s latest answers resolve the important uncertainties

If not, keep shaping through questions and review instead of writing too early.

The `docs/plans/YYYY-MM-DD-<topic>...` paths used throughout this skill are defaults. If the repo already has an established location or naming convention for plan/design docs, detect it and follow that instead — consistent with the rest of this skill's repo-awareness.

## Grounding rules (anti-hallucination)

The grounding rules are the spine of this skill — they apply to every step (planning, execution, review, QA) and get embedded into every generated prompt. They live in one canonical file so there is a single copy to keep correct: **`references/grounding-rules.md`**. Load it whenever shaping, reviewing, or generating prompts.

In short:

- **Read before reference** — never name a file/function/contract you haven't read this session.
- **Reachability before change** — a file existing isn't proof it's used; confirm it's reachable from a live entry point before planning edits to it (the dead look-alike passes every other check).
- **Evidence over assertion** — show the command and its output; "tests pass" is not evidence.
- **Build green is not "works"** — user-facing changes need a runtime check across reset/empty/failure paths, not just a passing build.
- **Independent verification** — reviewer/QA re-derive claims from the code and real output; they never trust the execution agent's self-report.
- **Plan grounding** — every file in the plan's scope must be real and live; validate file-scope against the import graph before execution.

See `references/grounding-rules.md` for the full rules, the hallucination red-flag list, and the per-role emphasis to embed into execution/reviewer/QA prompts.

## Challenge standard

This skill should challenge weak inputs. It should not be passive.

Push back when:

- goals are mixed with non-goals
- the user wants a plan without stating what must remain unchanged
- the user wants execution without task splitting
- the user is mixing business decisions with unresolved technical assumptions
- the user wants a refactor plan without clear preserved behavior
- the repo clearly signals a project-important concern and the brief ignores it

## Review standard

When reviewing plan or tasks artifacts, focus on:

- real execution risk
- missing compatibility contracts
- hidden migration assumptions
- verification gaps
- file ownership conflicts
- false-precision or vague phases

Findings should come first, ordered by severity, with file references.

## Repo-awareness

This skill is reusable across projects, but it must always absorb project constraints from repo docs and code before shaping artifacts.

Treat project rules as an overlay on top of the reusable workflow.

For major changes, especially topology or app-boundary changes, treat source-of-truth doc updates as first-class work. Update every affected context/source-of-truth doc the repo maintains (e.g. `AGENTS.md`, a project-context doc, app-level docs, READMEs) — do not assume a single context file is the only one that needs updating.

## Execution

When the user asks to start execution, the main agent acts as orchestrator: it drives gated execution one phase at a time (execution → reviewer → QA → advance only when both approve) and handles crash/stall/blocker recovery. The orchestrator coordinates and enforces gates; it does not do execution work itself. Treat sub-agents as **one-shot** — each dispatch (initial, rework, gate) is a fresh agent that reconstructs context from the progress file + code map, so a rejection routes into a rework pass, not a message back to a live agent.

Load **`references/execution-orchestration.md`** for the full orchestrator steps, the progress/recovery rules, stall detection, blocker escalation, and — to keep runs fast without weakening verification — **Context economy** (carry facts not conclusions; the code map; per-role reading lists; warm env), **delta-scoped rework/re-gate**, **contract amendments**, and **sanity-checking gate verdicts**. Track state with `references/progress-template.md`.

### Single-agent fallback (no sub-agents)

Sub-agents are an optimization, not a requirement. If your harness has no sub-agent support — or you are in Lite mode and choose not to spend on them — run the same gated method **sequentially in one agent that changes hats**, one phase at a time:

1. **Implement** the phase.
2. **Review hat.** Stop. Write what you changed to the progress file, then re-read every changed file *as if you had not written it* and review adversarially against the grounding rules — reachability, no regression, contract drift, scope. Record findings.
3. **QA hat.** Independently run the phase's verification, including the runtime check (build green is not "works"). Record evidence.
4. Fix any findings and repeat the review/QA hats until clean, then advance to the next phase.

Why this is weaker, and how to compensate: a single agent reviewing its own work carries the author's bias — it "knows what it meant" and tends to confirm rather than refute. The separate-agent flow's power is exactly the *independence* the single agent lacks. So compensate deliberately:

- Treat each hat as a real pass, not a rubber stamp. Switch intent explicitly ("now I am the skeptic trying to break this").
- Re-derive every claim from **freshly re-read code and actual command/runtime output**, never from your own recollection of having written it.
- Default to "not verified" — if you cannot show evidence, it is a finding, not a pass.
- Write findings to the progress file *between* hats so the next hat reasons from the artifact, not memory.

This keeps the method's discipline (grounded, gated, runtime-verified) intact; it only gives up parallelism and true independence. It is strictly better than no review — and strictly weaker than separate agents. Use it when sub-agents are unavailable or the change does not justify their cost.

Recovery, stall detection, and blocker escalation for the orchestrated path live in `references/execution-orchestration.md`.

## Skill goals

This skill serves two goals:

1. **Planning phase**: Write a well-crafted plan based on human input, existing docs and code, and a focused series of questions to remove all ambiguity before execution starts.
2. **Execution phase**: Reduce human interaction as much as possible. Sub-agents (execution, reviewer, QA) work as a team, report to each other through the orchestrator, and resolve issues autonomously. Only true blockers escalate to the human.

All design decisions in this skill should be evaluated against these two goals.

## Output discipline

- Prefer concise, direct guidance.
- Do not flood the user with giant taxonomies.
- Keep questions focused.
- Move artifact by artifact.
- Do not skip critical shaping just because the user already has a rough doc.
