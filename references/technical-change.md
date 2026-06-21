# Technical Change

Use this path when the change is driven by platform, infrastructure, reliability, performance, architecture, or technical capability.

## Brief quality bar

The brief should define:

- technical problem
- desired technical outcome
- current architecture or operational pain
- preserved behavior / constraints
- rollout or migration constraints
- success criteria
- observability / verification expectations

## Questions to prioritize

- What technical problem are we solving?
- What evidence or pain makes this worth changing now?
- What behavior must remain unchanged?
- What operational risks matter most?
- How will we know the change succeeded?
- What migrations, cutovers, or feature flags are allowed?
- Which project-important concerns are implicated according to the repo scan, even if the user did not mention them explicitly?

## Planning emphasis

Bias the implementation plan toward:

- architecture boundaries
- migration path
- rollout safety
- observability
- performance / reliability / security impact
- reversibility

## Feature flag decision heuristic

When the change could benefit from feature flags, use this heuristic:

**Prefer feature flags when:**
- The change is user-facing and reversibility is critical
- The change can be meaningfully toggled without leaving orphaned state
- The team can commit to removing the flag within a bounded timeframe

**Skip feature flags when:**
- The change is purely internal (refactor, infra) with no user-facing toggle point
- The flag-off path would be untestable or meaningless
- The change is all-or-nothing (schema migration, data format change)
- The added code complexity of the flag outweighs the rollback benefit

When in doubt, ask: "If we need to turn this off in production, can a flag do it cleanly, or do we need a real rollback?" If the answer is "real rollback," a flag adds complexity without value.

## Common failure modes

- describing the solution without the problem
- under-specifying rollout
- forgetting operational ownership
- mixing refactor cleanup with technical capability work
- no measurable success criteria
- missing repo-obvious implications because the skill failed to inspect the project properly first
