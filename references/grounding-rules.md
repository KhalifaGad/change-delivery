# Grounding Rules (anti-hallucination)

**This file is the single source of truth for the skill's grounding rules.** They apply to every step of the workflow — planning, execution, review, and QA — and they are what gets embedded into generated execution/reviewer/QA prompts. Do not restate these rules in full elsewhere; summarize and link here so there is only one copy to keep correct.

## Universal rules

### Read before reference

No agent may reference a file, function, contract, or pattern in any artifact unless it has read that file in the current session. If an agent cannot read the file (permissions, missing, etc.), it must say so instead of guessing the contents.

- Do not fabricate file paths. If you are unsure whether a file exists, check first.
- Do not invent API contracts, function signatures, or config shapes. Read the source.
- Do not describe code structure from memory or training data. Read the actual repo.

### Reachability before change (liveness check)

A file existing is NOT proof it is used. Before any artifact plans changes to a target file, verify it is actually reachable from a live entry point — a route, a registered controller/module, `main`, an exported public API, or something transitively imported by one. A file can compile, type-check, and look central while being dead code that nothing renders or calls.

- UI change: confirm the component is mounted on a route (or imported by something that is). Grep for importers and trace up to a router/entry. Two similarly-named components (`Foo` vs `redesign/Foo`, `OrdersPanel` vs `OrdersPage`) is a classic trap — confirm which one the running app actually uses.
- Backend change: confirm the handler/service is wired into a registered module/route, not an orphaned file.
- Zero importers and not itself an entry point ⇒ treat as dead: flag it, do not plan changes against it.
- "Read before reference" stops you inventing files; this rule stops you editing the wrong *real* file. Both are required — the second is the one that silently wastes a whole phase, because the dead file passes every other grounding check.

### Evidence over assertion

No agent may claim a task is complete, a test passes, or a verification succeeds without showing the actual output.

- "All tests pass" is not acceptable. Show the command and its output.
- "File updated" is not acceptable. Show the diff or the relevant section.
- "No regressions" is not acceptable. Show what was checked and the result.

Progress file updates must include evidence references (command run, output summary), not just status changes.

### Build green is not "works" (runtime verification)

A passing build, type-check, lint, and unit-test run prove the code COMPILES and that covered logic holds. They do not prove the change behaves correctly in the running system. Many real defects — layout/positioning, event races, state that only updates on a second interaction, a control wired to the wrong handler, a query that returns the wrong page, a filter that never fires — are invisible to static checks and surface only at runtime.

- For any user-facing or integration change, require a RUNTIME check: exercise the actual path (drive the UI, hit the endpoint, run the migration against a real DB) and observe the result — not just "it builds".
- Cover the destructive/reset/empty paths, not only the happy path: apply a filter AND clear it; open AND close; first page AND page N; the success case AND the failure case.
- If runtime verification is blocked (no browser, no environment), say so explicitly and treat the runtime check as an OPEN gap — never let "build passes" stand in for "works".

### Independent verification

Reviewer and QA agents must independently verify claims. They must not trust the execution agent's self-report.

- Reviewer must read the actual changed files, not just the execution agent's summary of what changed.
- QA must run or inspect verification commands independently, not accept "I ran the tests and they passed" from the execution agent.
- If a reviewer or QA agent cannot independently verify a claim (e.g., no access to run tests), it must flag that as a gap, not approve based on the execution agent's word.

### Plan grounding

Plans and task files must reference real, verified repo structures:

- File paths in plans must come from actual repo inspection, not assumed project layouts.
- **Every file in the plan's scope must be live** — reachable from an entry point (see "Reachability before change"). Validate the plan's file-scope against the live import graph BEFORE execution: a plan that targets dead code passes every other grounding check and still burns the entire phase. A wrong file-scope propagates silently from brief → plan → tasks → execution; catch it at the plan, not at review.
- Design decisions must reference actual code patterns found in the repo, not generic best practices.
- If the plan assumes a pattern exists (e.g., "the existing middleware chain"), the agent must verify it exists before writing it into the plan.

## Hallucination red flags

When reviewing any agent output, flag these as potential hallucination:

- File paths that were not verified to exist
- Function or class names that don't appear in the repo
- Claims about code behavior without file:line references
- Verification results without command output
- "I confirmed..." without showing what was confirmed
- References to patterns, libraries, or frameworks not present in the project
- A target file that exists but was never confirmed reachable from an entry point (dead-code risk — see "Reachability before change")
- "Build / tests / lint pass" offered as proof that a user-facing change *works*, with no runtime check of the actual behavior

## Embedding these rules into prompts

Every generated execution/reviewer/QA prompt must carry these rules. Embed the universal rules above verbatim, then add the per-role emphasis below. Do not paraphrase a second copy — point back to this file.

- **Execution prompt** — emphasize *Read before reference* (read every file before editing it) and *Reachability before change* ("edit the live file, not a look-alike": confirm the target is reachable from an entry point; zero importers and not an entry point ⇒ stop and flag). Plus: no invented paths, no assumed patterns, show your work in the progress file.
- **Reviewer prompt** — emphasize *Independent verification* (read every changed file yourself; do not rely on the execution agent's description) and *Reachability* (verify the changed files are live; flag any edited file you cannot trace to an entry point). Plus: verify every "tests pass" claim against progress-file evidence; flag unsupported claims.
- **QA prompt** — emphasize *Evidence over assertion* (run verification yourself; do not accept claimed output) and *Build green is not "works"* (exercise the behavior at runtime across reset/empty/failure paths; if you cannot run it, mark the runtime check OPEN — never approve on build-green alone). Plus: run the existing suite independently; reject any verification step with no evidence in the progress file.
