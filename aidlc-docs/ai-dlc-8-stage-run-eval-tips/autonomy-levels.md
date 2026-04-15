# AI-DLC Run Autonomy Levels

> Purpose: Guide for configuring how much human confirmation is required during an 8-stage story run.
> Implement by editing the Stage Gates section in `.claude/commands/run-story.md`.

---

## Background

Every stage gate in `run-story.md` today carries the instruction:
> "Do NOT proceed past any approval gate without explicit user confirmation"

This is a hard instruction to the model. It can be selectively relaxed per stage as trust in the
workflow is established — typically after several manual runs that calibrate which gates add real value.

---

## Level 1 — Selective Gates (Recommended starting point)

Keep gates only at high-judgment stages. Remove them from mechanical, objective-output stages.

| Stage | Gate | Rationale |
|-------|------|-----------|
| 1 — Story Intake | **Keep** | Platform alignment decision requires human judgment |
| 2 — Scoped RE | Remove | Mechanical — file identification, no decisions made |
| 3 — Workflow Planning | **Keep** | Primary human checkpoint before any code is written |
| 4 — Implementation | **Keep** (per repo) | Diff review before proceeding to next repo |
| 5 — Tests | Remove | Pass/fail is objective — no judgment needed |
| 6 — AI Reviewer | **Keep** | Blocking findings require human resolution |
| 7 — PR Creation | Remove | Templated mechanical output |
| 8 — Merge | **Keep** | Irreversible action — always human-gated |

**When to adopt**: after 3-4 fully manual runs where Stages 2, 5, and 7 gates consistently added no value.

---

## Level 2 — Conditional Gates

Gates fire only when the stage produced something non-trivial. Otherwise the workflow auto-proceeds.

| Stage | Gate condition |
|-------|---------------|
| 1 — Story Intake | Fire only if platform alignment issues were found |
| 3 — Workflow Planning | Fire only if plan spans more than one repo or has unresolved ambiguities |
| 4 — Implementation | Fire only if diff touches files outside the approved plan (scope creep detected) |
| 6 — AI Reviewer | Fire only if blocking findings exist |
| 8 — Merge | Always fire — irreversible |

**When to adopt**: after Level 1 has been stable across multiple story types and you trust the model's
judgment on what constitutes a non-trivial finding.

---

## Level 3 — Fully Autonomous

All gates removed. The model runs all 8 stages end-to-end and opens the PR without interruption.
Human review happens on GitHub after the PR is created.

**When to adopt**: only for low-risk story types (tactical fixes, documentation updates) where the
cost of a wrong decision is recoverable and the story scope is narrow and well-defined.

**Never use for**: stories touching banking-contracts, authentication, payment logic, or any
cross-repo changes — the blast radius of a wrong autonomous decision is too high.

---

## How to Implement

Edit the Stage Gates table in `.claude/commands/run-story.md`.

For each gate to remove, replace:
```
| [N] | [condition] |
```
With:
```
| [N] | Auto-proceed (no human gate) |
```

For conditional gates (Level 2), replace with:
```
| [N] | Gate fires only if: [condition] — otherwise auto-proceed |
```

---

## Recommendation

Start with **Level 1** after your first few practice runs. Stages 1, 3, and 8 should remain
human-gated indefinitely — they involve irreversible decisions or architectural judgment that
automation should not own regardless of trust level.
