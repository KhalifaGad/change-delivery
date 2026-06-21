# Refactor

Use this path when the main goal is better internal structure while preserving external behavior.

## Brief quality bar

The brief should define:

- what is structurally wrong today
- what behavior must remain unchanged
- what quality attributes should improve
- scope boundaries
- migration/cutover expectations
- verification strategy

## Questions to prioritize

- What is the current pain in the code or architecture?
- What behavior must not change?
- What parts of the system are in scope?
- What quality improvement do we want: clarity, modularity, boundaries, testability, maintainability?
- What regressions are most likely?
- How will we prove behavior stayed intact?

If the refactor changes app boundaries, routing layers, or host ownership, also ask about:

- the project-important concerns revealed by repo inspection
- preserved visible contracts
- shared code extraction vs duplication
- source-of-truth doc ownership after cutover

## Planning emphasis

Bias the implementation plan toward:

- preserved behavior
- sequencing
- rollback safety
- test coverage
- migration of internals without user-facing drift

## Common failure modes

- calling something a refactor when it changes product behavior
- no explicit preserved-behavior list
- vague phases like “clean up X”
- no regression plan
- opportunistic scope creep
- silently inheriting project-important decisions from repo context without confirming them
