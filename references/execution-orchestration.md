# Execution Orchestration & Recovery

Load this reference when the user asks to **start execution** (not during brief/plan/tasks shaping). It covers how the orchestrator drives gated execution and how to recover when an agent crashes, stalls, or hits a blocker.

For the no-sub-agent path, see "Single-agent fallback" in `SKILL.md`. The grounding rules referenced throughout are in `references/grounding-rules.md`.

## Orchestrator role

When the user asks to start execution, the main agent acts as orchestrator:

1. Load any user-available orchestration skills for coordinating sub-agents.
2. Use the generated prompts artifact as the source of truth for sub-agent instructions.
3. Dispatch the execution prompt for the current phase, with the phase's facts pinned into the spawn message (see "Context economy" below).
4. After each phase completion, dispatch the reviewer prompt scoped to the completed phase.
5. After reviewer approval, dispatch the QA prompt scoped to the completed phase.
6. If either reviewer or QA rejects, route the findings into a **rework pass** for the same phase (see "Rework and re-gate").
7. **Sanity-check each gate verdict before trusting it** (see "Sanity-checking gate verdicts"). Only advance to the next phase after both reviewer and QA approve *and* their verdicts carry real evidence.

The orchestrator should not do execution work itself. Its job is coordination, sequencing, gate enforcement, and keeping the progress file + code map current.

### Sub-agents may be one-shot — design for it

Do not assume you can "send a message back to the same sub-agent." Many harnesses spawn a fresh agent each time; there is no persistent conversation to resume. The orchestration here is therefore **stateless by design**: every dispatch (initial, rework, gate, re-gate) is a *new* agent that reconstructs what it needs from the progress file + code map + the spawn message. This is not a downside — it is exactly the isolation that keeps one agent's mistakes from propagating into the next (see "Context economy"). When this reference says "pass findings back to the execution sub-agent," read it as "dispatch a rework agent that resumes from the progress file," not "reuse a live agent."

## Context economy: carry facts, not conclusions

The single biggest cost in stateless multi-agent execution is **cold re-discovery** — every fresh agent grepping and re-reading large files and all four artifacts from zero just to learn where things are. Left unchecked this dominates both wall-clock and tokens. But you cannot fix it by giving agents stale context, because the *reason* fresh agents are valuable is that they don't inherit earlier agents' mistakes.

Resolve this by separating two things "fresh context" conflates:

- **Fresh reasoning** (keep): a new agent forms its own conclusions about correctness; it never inherits a prior agent's "it works", verdicts, or half-formed theories.
- **Cold re-discovery** (eliminate): a new agent should not have to re-find where the code lives.

The rule: **carry facts, never conclusions.**

- **Facts** are safe to hand forward because they're cheap to verify and don't encode judgment: file paths, symbol names, line anchors, the frozen contract, "the cursor codec lives in `types/order-cursor.ts`". A fresh agent that's handed facts still reasons about correctness from scratch — and still reads the specific lines it edits (read-before-write is untouched).
- **Conclusions / state** are NOT carried: "I verified this", "this is correct", "the bug is in X". Those are exactly what fresh isolation exists to discard.

### The code map

Maintain a **Code map** section in the progress file (see `references/progress-template.md`): a compact, accreting list of facts the work depends on — paths, key symbols, line anchors, the frozen contract(s). Phase 1 has to discover these anyway; capture them once. Every later agent reads the map first instead of re-grepping the repo cold. The map holds *facts only* — if it ever contains a judgment ("this is the right approach"), that's a defect; move it out. Because the map is verifiable facts, it can't become a vector for an inherited mistake: a wrong path is caught the moment the agent opens it (read-before-write), unlike an inherited wrong *conclusion*, which is believed.

### Pin facts into the spawn message

Counterintuitively, a *longer, more precise* spawn message is *faster*: when the orchestrator pins the exact `file:line` anchors, the relevant contract excerpt, and the code-map entries an agent needs, the agent verifies specific spots instead of spending minutes re-discovering them. Hand **locations and contracts** (facts), never **interpretations** (conclusions).

### Per-role minimal reading lists

Do not tell every agent to "read all four artifacts." Scope reading to role:

- **Execution agent**: its phase's tasks + the frozen contract + the code map. Not the brief's prose, not the prompts file.
- **Reviewer**: the phase scope + the changed files + the contract + the code map.
- **QA**: the phase's verification steps + the progress evidence + the runtime surfaces. 

And prefer **targeted reads** — read the named functions/regions an agent will touch, not entire files, unless the file is small. A 1000-line page read in full by every agent is mostly waste.

### Warm shared environment + targeted commands

- **Reuse one environment.** If a dev server / authenticated session / browser context is already up, agents reuse it rather than each cold-starting and re-authenticating. Re-doing a slow login per agent is pure overhead.
- **Targeted verification commands.** Use the cheapest command that gives the signal: a typecheck (e.g. `tsc --noEmit`) instead of a full production build when only types matter; a single test pattern instead of the whole suite. Reserve the full build + full suite for the **QA regression pass**, where breadth is the point. (This trims redundant cost, not coverage — the runtime checks in the grounding rules still run in full.)

## Rework and re-gate (delta-scoped)

A rejection does not restart the phase. Scope the rework — and its re-gate — to the delta:

- The **rework agent** gets: the specific findings + the changed files + the code map. It does NOT re-read all artifacts or re-discover the codebase.
- The **re-gate** verifies the *fix*, not the whole phase again: re-run the previously-failing check, run the build/typecheck (cheap cross-effect catch), and re-run the runtime check **for the affected surface** plus a quick regression smoke. Re-verifying untouched surfaces that already passed produces no new signal — skip it. (This is a redundancy cut, not a coverage cut: the surfaces being skipped were already independently verified earlier in the same phase.)

### Trivial fixes

For a finding that is a single mechanical edit (a rename, a type narrow, a one-line correction the reviewer already specified):

- **Default**: dispatch an un-fattened trivial-fix agent that receives only the finding + the one file + a targeted verify command — no artifact re-read, no full rebuild. Independence is preserved; the fat is removed.
- The orchestrator MAY apply such a fix inline **only** when it is a zero-judgment edit (the reviewer named the exact change and there is nothing to reason about), and the build/gate still verifies it. Anything with judgment goes to an agent.

## Contract amendments

Freezing a contract before downstream phases is correct — but a later phase can reveal the freeze was insufficient (e.g. a param mode that collides with an existing pagination model). Do not let an agent silently diverge from the frozen contract to cope.

When a frozen contract proves inadequate:

1. **Amend explicitly.** Update the contract in the plan (and the code-map's frozen-contract entry), noting what changed and why.
2. **Re-gate only the delta** — the surfaces the amendment touches, plus anything that consumed the old shape.
3. Record the amendment in the progress file so later agents build against the new shape, not the stale one.

A silent divergence is a contract drift finding; an explicit amendment is normal engineering. Make it the latter.

## Sanity-checking gate verdicts

The gates are themselves sub-agents and can rubber-stamp. Before advancing on an APPROVE, the orchestrator spot-checks it:

- An APPROVE with no command output / no runtime evidence cited is itself a finding — treat it as "not verified" and send it back, do not advance on it.
- A suspiciously fast or generic verdict on a high-risk phase (migrations, money, auth, multi-tenant scoping, shared components) warrants a second look at whether the gate actually exercised the runtime path it claims.
- The orchestrator never has to re-do the gate's work — but it must not advance the phase on an *unevidenced* approval.

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
