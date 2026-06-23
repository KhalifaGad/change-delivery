# Changelog

All notable changes to this skill are documented here. The format loosely follows
[Keep a Changelog](https://keepachangelog.com/). Releases are git tags used as human-readable
anchors; `npx skills add KhalifaGad/change-delivery` installs the current default branch, and the
`skills` CLI tracks updates by tree-SHA.

## [0.2.0] - 2026-06-23

Refinements from running the skill end-to-end on a large brownfield feature — all aimed at cutting
wall-clock/token cost **without weakening** the independent runtime verification that catches real bugs.

### Added

- **Context economy** (`references/execution-orchestration.md`) — the "carry facts, not conclusions"
  principle that lets fresh agents keep their isolation while skipping cold re-discovery, backed by an
  accreting **Code map** in the progress file, per-role minimal reading lists, pinned `file:line` facts in
  spawn messages, warm-environment reuse, and targeted verification commands.
- **Delta-scoped rework & re-gate** — rework agents and re-gates verify the fix + a regression smoke, not the
  whole phase again; plus a **trivial-fix** class (un-fattened fix agent by default; inline only for
  zero-judgment edits).
- **Contract-amendment protocol** — explicit amend + delta re-gate when a frozen contract proves inadequate,
  instead of silent divergence.
- **Gate-verdict sanity check** — the orchestrator treats an unevidenced APPROVE as "not verified" and does
  not advance on it.
- **Open-verification-gaps ledger** in the progress template — every `OPEN:` runtime check is tracked and the
  final report must drain it, so deferred verification can't silently evaporate between phases.

### Changed

- Reconciled the stateless-recovery vs. stateful-orchestration contradiction: sub-agents are treated as
  **one-shot**; a rejection routes into a rework pass that resumes from the progress file, not a message back
  to a live agent.
- Hybrid changes: noted that the project concern map usually shapes the plan more than per-type reference
  files do — load the refs for their checklists, but lean on repo grounding.

## [0.1.0] - 2026-06-21

Initial public release.

### Added

- **Core workflow** — classify a change (`business` / `technical` / `refactor`), then drive it through
  brief → implementation plan → tasks → execution/reviewer/QA prompts, challenging weak inputs at each step.
- **Gated multi-agent execution** — independent reviewer and QA must both approve a phase before it advances,
  plus a single-agent "changes hats" fallback for harnesses without sub-agents.
- **Canonical grounding rules** (`references/grounding-rules.md`) — read-before-reference,
  reachability-before-change, evidence-over-assertion, build-green-is-not-"works" (runtime verification),
  independent verification, and plan grounding, with a hallucination red-flag checklist.
- **Cost/scale modes** — Lite / Standard / Full, so the ceremony matches the change.
- **Execution orchestration & recovery** (`references/execution-orchestration.md`) — orchestrator steps,
  progress tracking, crash/stall detection, and blocker escalation, with a fillable
  `references/progress-template.md`.
- **Per-change-type references** — business / technical / refactor emphasis, rollback approaches,
  artifact standards, and prompt-generation guidance.
- **Docs** — README with "How it works" diagrams (gated loop + single-agent fallback); MIT license.

[0.2.0]: https://github.com/KhalifaGad/change-delivery/releases/tag/v0.2.0
[0.1.0]: https://github.com/KhalifaGad/change-delivery/releases/tag/v0.1.0
