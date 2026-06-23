# Artifact Standards

Use these standards when shaping and reviewing artifacts.

## Brief

A strong brief usually includes:

- title / topic
- purpose
- scope
- goals
- non-goals
- invariants
- current-state assumptions
- success criteria
- rollout constraints
- testing constraints (if any)
- planning/output expectations if relevant

## Implementation plan

A strong implementation plan usually includes:

- context summary
- design decisions
- phases
- file scope by phase
- verification by phase
- risks / migration / rollback
- open decisions only when truly unresolved

Good plans are:

- brownfield-aware
- file-scoped
- verification-heavy
- explicit about what not to change
- precise about exact file paths where the likely impact is knowable

Weak plans usually:

- paraphrase the brief
- stop at schema or architecture theory
- use oversized phases
- avoid migration details
- omit verification
- mention rollback without defining trigger conditions, steps, or verification

## Rollback plan

When the brief confirms rollback is in scope, the implementation plan must include a rollback section with:

- rollback trigger conditions (what signals that rollback is needed)
- rollback steps per affected layer (database, API, deployment, config, etc.)
- data reversibility assessment (can data written by the new version be safely handled after rollback?)
- rollback verification (how to confirm rollback succeeded)
- partial rollback risks (what happens if only one layer is rolled back while others stay forward)

Use `references/rollback-approaches.md` to identify the right approach per layer. Do not include rollback boilerplate — only include approaches that match the actual change.

### File references

- Prefer exact file paths in plans and tasks where likely impact is knowable.
- Do not force line-by-line references throughout a plan.
- Use line references only when calling out:
  - a risky integration point
  - a specific contract or function that must be preserved
  - a known drift location

## Tasks

A strong tasks file usually includes:

- phases
- small tasks
- prerequisites
- files in scope
- expected output/change
- verification
- stop/review gate
- parallelization rules only when safe

Weak task files usually:

- repeat plan phases without decomposition
- assign overlapping write scopes
- use fake verification (e.g., "verify it works" without a concrete command)
- skip operational preconditions
- let phases advance without real gates

### Verification evidence standard

Every verification step in a tasks file must be concrete enough that an agent can run it and produce evidence:

- Good: `run npm test -- --filter=auth` / `verify 200 response from GET /api/users`
- Bad: `verify it works` / `check everything is fine` / `confirm no regressions`

Prefer the **cheapest command that yields the signal** — a targeted typecheck (e.g. `tsc --noEmit`) or a single test pattern over a full production build or the whole suite. Reserve the full build + full test suite for the **QA regression pass**, where breadth is the point. This trims redundant cost without weakening the runtime verification the grounding rules require.

When the execution agent completes a verification step, the progress file must include:

- the exact command run
- a summary of the output (pass/fail, error messages if any)
- timestamp

Reviewer and QA agents must check this evidence independently. A verification step with no evidence in the progress file is an automatic rejection.

For any user-facing or integration change, at least one verification step must be a **runtime** check, not only build/type/lint/unit-test. Build-green proves the code compiles; it does not prove the behavior is correct (positioning, races, second-interaction state, a control wired to the wrong handler, a query returning the wrong page all pass a build). The runtime step must exercise the real path and cover the reset/empty/failure cases, not just the happy path. If runtime verification is environmentally blocked, the tasks file must say so and record it as an open gap rather than treating build-green as sufficient.

## Prompts

Strong prompts:

- point to concrete artifacts
- define the execution rule clearly
- define what must not happen
- define completion conditions
- define reviewer / QA expectations when relevant
- explain how the prompts should be used when more than one prompt is generated
- define scope enforcement: reviewer/QA must flag out-of-scope changes, not just bugs
- define partial approval rules: whether a phase can be partially approved or must pass/fail as a whole
- define what to do when artifacts are inconsistent with the code (escalate, not silently accept)

Weak prompts:

- rely on persona fluff
- say “do a good job” instead of defining behavior
- omit stop/go rules
- omit source artifacts
- dump execution/reviewer/QA prompts without explaining orchestration
- allow partial approval without defining what that means

## Project concern check

Before accepting a brief or plan, derive the project-important concerns from repo docs and code, then check whether the change should explicitly address them.

Do not use a fixed global checklist as if every repository values the same things equally.

Instead:

- inspect policy docs and context files
- inspect routing, auth, API, rendering, build, and deployment structure
- identify what this repository treats as cross-cutting or fragile
- compare the proposed change against those concerns

If a project-important concern is clearly relevant from repo context and omitted from the artifact, that is a quality gap.

Common concern categories may include:

- routing / URL contracts
- auth / session / cookies
- localization / translation runtime
- rendering / SEO / metadata
- PWA behavior
- API origin / base-url / proxy behavior
- build/dev/deployment topology
- source-of-truth doc updates

But these are examples, not a fixed required list for every project.

## Topology-change doc sync

If the change alters app boundaries, host ownership, deployment topology, or project structure, final doc sync should usually include every source-of-truth doc the repo maintains, for example:

- root agent/context docs (e.g. `AGENTS.md` or a project-context doc)
- affected app-level agent/context docs
- affected README or local setup docs

Do not treat any single context file as the only required doc update for topology changes — sync all the ones this repo actually keeps.

## Review checklist

Use this checklist when reviewing plans or tasks:

- Are goals, non-goals, and invariants clear?
- Is the current state clear enough?
- Is the output grounded in the repo?
- Are phases small and ordered?
- Are file scopes believable?
- Are migration and compatibility covered?
- Are gates and verification real?
- Are there hidden shared-file conflicts for subagents?

### Grounding checks (anti-hallucination)

These are the review-time application of the canonical grounding rules in `references/grounding-rules.md` (see there for the full definitions). Additionally, check for hallucination signals:

- Do all file paths in the artifact actually exist in the repo?
- Is every file in scope actually **live** — reachable from a route / registered module / entry point — and not dead code that merely exists? (A whole phase can target a dead look-alike file and pass every other check.)
- Do referenced functions, classes, or contracts exist where claimed?
- Are design decisions based on actual repo patterns, or assumed best practices?
- Does every verification step have a concrete, runnable command — and does a user-facing change include a **runtime** check, not just build/test?
- Are claims about current-state behavior backed by file:line references?
- If an agent says "I verified X," is the actual output included?

Flag any item that fails these checks as a grounding gap, ordered by severity alongside other findings.
