# change-delivery

**An agent skill that turns a change request into execution-ready delivery artifacts — and verifies the work adversarially as it ships.**

`change-delivery` is a methodology, not a wrapper. It takes a business change, a technical change, or a refactor and drives it through a disciplined pipeline: a shaped **brief**, a brownfield **implementation plan**, an execution **task breakdown**, and **execution / reviewer / QA prompts** — then runs the work in **gated phases** where an independent reviewer and QA must approve before advancing.

It exists because LLMs are very good at producing changes that *look* right — compile, pass type-checks, read plausibly — and confidently ship bugs a second pair of eyes would have caught. This skill builds that second (and third) pair of eyes into the process, plus the grounding rules that stop an agent from inventing or editing the wrong thing.

---

## Why it's different

Most "AI dev workflow" prompts help you *plan*. This one is built around **adversarial verification** and **anti-hallucination grounding**:

- **Separation of roles.** The agent that writes the code never reviews or QAs it. A fresh, independent agent re-reads the change cold and tries to break it. Independence is the bug-catching mechanism — self-review rubber-stamps.
- **Grounding rules.** Read before reference, evidence over assertion, and two rules learned the hard way:
  - **Reachability before change** — a file existing isn't proof it's *used*. Confirm the target is reachable from a route/entry point before editing it. (Editing a dead look-alike file compiles, reviews fine on its own terms, and ships nothing.)
  - **Build green is not "works"** — a passing build/test/lint proves it compiles, not that it behaves. User-facing changes require a *runtime* check, covering the reset/empty/failure paths, not just the happy path.
- **Gated execution with recovery.** Phases advance only on reviewer + QA approval; a progress file externalizes state so work survives crashes, stalls, and hand-offs.

## It catches real bugs

In a single real feature, the gates flagged — in code another agent wrote and was confident about:

- a composite **database index the query planner would never use** (a column-to-column comparison a B-tree can't serve) — pure write-cost, no read benefit;
- an entire **phase implemented against dead code** — a component that compiled and type-checked but was never mounted on a route;
- a **filter that only ever returned the first page** of results, because filtering ran client-side over a capped fetch;
- a **"Reset" button and a date selector that silently did nothing**, due to collapsed state-setter calls racing each other;
- a **count badge that under-counted**, because two hand-maintained status lists had drifted apart.

Every one of those passes a green build. None survived an independent review + runtime check.

---

## The artifacts

The skill produces and maintains, in order:

1. **Brief** — goal, non-goals, invariants, current-state assumptions, success criteria, rollout/test constraints.
2. **Implementation plan** — phased, file-scoped, verification-per-phase, repo-grounded.
3. **Tasks** — small, gated, executable units with concrete verification and stop/review gates.
4. **Prompts** — execution / reviewer / QA prompts grounded in the real artifacts.
5. **Progress file** — the running source of truth for recovery and evidence.

## Modes (scale the ceremony to the change)

Gated multi-agent execution is powerful but expensive. Pick a mode up front:

- **Lite** — small, low-risk, well-understood changes. Brief + short task list, one execution pass, one honest self-review (using the single-agent fallback). No separate sub-agents.
- **Standard** — a normal feature/refactor. Full artifacts, gated execution; collapse reviewer+QA on low-risk phases, reserve the full pair for the risky ones (migrations, shared code, money/auth/multi-tenant).
- **Full** — high-risk or compliance-sensitive. Everything, reviewer **and** QA on every phase.

> No sub-agents in your harness? The same gated method runs **sequentially in one agent that changes hats** (implement → review cold → runtime-QA), with the honest caveat that it's weaker than true independence. See *Single-agent fallback* in [`SKILL.md`](SKILL.md).

---

## Install

`change-delivery` is a standard **Agent Skill** — a folder with [`SKILL.md`](SKILL.md) (frontmatter `name` + `description`) and a [`references/`](references/) directory. It's plain markdown; drop it where your agent discovers skills.

**Claude Code**
```bash
# user-level (all projects)
git clone https://github.com/KhalifaGad/change-delivery ~/.claude/skills/change-delivery
# or project-level
git clone https://github.com/KhalifaGad/change-delivery <your-project>/.claude/skills/change-delivery
```

**Codex**
```bash
git clone https://github.com/KhalifaGad/change-delivery ~/.codex/skills/change-delivery
```

**Any other harness** — place `SKILL.md` + `references/` wherever your agent loads skills/playbooks, then point it at `SKILL.md`. The paths inside the skill are all relative, so it works from any install location.

[`claude-command.md`](claude-command.md) is an optional slash-command launcher for harnesses that invoke via slash commands — adapt or ignore per your setup.

> Exact skill directories vary by tool and version; check your harness's docs if the paths above differ.

## Usage

Invoke the skill (e.g. `/change-delivery`, or however your harness triggers skills) and describe the change. It will:

1. Inspect the repo and classify the change as `business`, `technical`, or `refactor`.
2. Shape the brief through focused questions — pushing back on vague goals, missing invariants, and weak rollout assumptions.
3. Write the plan → tasks → prompts, each grounded in the actual repo.
4. On request, orchestrate gated execution (or hand you the prompts to run yourself).

It will not advance an artifact until it's strong enough, and it writes files automatically once the decisions are no longer ambiguous.

---

## Repository layout

```
change-delivery/
├── SKILL.md                         # the skill (entry point)
├── claude-command.md                # optional slash-command launcher
├── references/
│   ├── artifact-standards.md        # what a strong brief/plan/tasks/prompt looks like
│   ├── business-change.md           # business-change emphasis + questions
│   ├── technical-change.md          # technical-change emphasis + feature-flag heuristic
│   ├── refactor-change.md           # refactor emphasis (preserved behavior)
│   ├── prompt-generation.md         # execution/reviewer/QA prompt shapes + grounding rules
│   └── rollback-approaches.md       # per-layer rollback options
├── README.md
└── LICENSE
```

## Contributing

Issues and PRs welcome — especially real-world failure modes the grounding rules should catch, and harness-specific install notes.

## License

[MIT](LICENSE) © Khalifa Gad
