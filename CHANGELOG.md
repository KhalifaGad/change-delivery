# Changelog

All notable changes to this skill are documented here. The format loosely follows
[Keep a Changelog](https://keepachangelog.com/). Releases are git tags used as human-readable
anchors; `npx skills add KhalifaGad/change-delivery` installs the current default branch, and the
`skills` CLI tracks updates by tree-SHA.

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

[0.1.0]: https://github.com/KhalifaGad/change-delivery/releases/tag/v0.1.0
