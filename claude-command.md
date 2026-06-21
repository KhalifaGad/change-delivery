---
description: Shape a business, technical, or refactor change into a strong brief, implementation plan, task breakdown, and execution prompts.
---

Use the `change-delivery` workflow. Read these files (paths are relative to this skill's own directory):

- `SKILL.md`
- `references/artifact-standards.md`
- `references/prompt-generation.md`

First, classify the change as `business`, `technical`, or `refactor`, then read the matching reference file:

- `references/business-change.md`
- `references/technical-change.md`
- `references/refactor-change.md`

Follow the workflow exactly:

1. Shape the brief through adaptive questions.
2. Create or tighten the implementation plan.
3. Create or tighten the execution tasks.
4. Generate execution, reviewer, and QA prompts when the artifacts are ready.

Behavior rules:

- Ask one question at a time by default.
- Ask up to 3 short questions only when the input is too weak to proceed efficiently.
- Challenge vague goals, weak constraints, missing invariants, and weak rollout assumptions.
- Do not advance to the next artifact until the current artifact is strong enough.
- Write files automatically when certainty is effectively complete.
- Stay brownfield-aware and grounded in repo constraints and code.
- Prefer operational rules over persona fluff.

If the user already has a brief, plan, or tasks file, review that artifact critically and continue from there instead of restarting.

If the user asks for execution prompts, generate well-crafted prompts grounded in the actual artifact files and phase/gate rules, not generic “do a good job” prompts.
