# Business Change

Use this path when the change is driven by product or domain behavior.

## Brief quality bar

The brief should define:

- business objective
- target user-facing behavior
- explicit non-goals
- invariants / what must remain unchanged
- current-state assumptions
- success criteria
- scope boundaries
- open product decisions still needing answers

## Questions to prioritize

- What is the business change in one sentence?
- What should users be able to do after this change?
- What must remain unchanged?
- What is explicitly out of scope for this phase?
- What does the current system do today?
- What would make this change successful?
- Which project-important constraints from the repo does this change obviously touch, even if the user did not mention them?

## Planning emphasis

Bias the implementation plan toward:

- product/domain rules
- data model implications
- UX implications
- API and contract changes
- migration and compatibility
- rollout safety

## Common failure modes

- vague target state
- missing current-state description
- forgetting non-product invariants
- hiding product decisions inside technical wording
- planning the schema but not the user flows
- ignoring project-obvious concerns because the skill failed to infer them from repo docs and code
