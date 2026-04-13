# Run AI-DLC 8-Stage Story Workflow

Execute the complete 8-stage AI-DLC workflow for a Jira story through all gates from intake to merged PR.

**Story Key**: $ARGUMENTS

## Execution Instructions

1. Load the full 8-stage workflow definition from `aidlc-docs/ai-dlc-8-stage-run-template.md`
2. Load rule detail files from `.aidlc-rule-details/` for each stage as it executes
3. Execute stages in strict sequence: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8
4. Do NOT skip any stage without explicit user instruction
5. Do NOT proceed past any approval gate without explicit user confirmation
6. Log every interaction and approval to `aidlc-docs/audit.md` (append only — never overwrite)
7. Track stage completion in `aidlc-docs/aidlc-state.md` immediately after each stage

## Stage Sequence & Model Assignment

| Stage | Name | Model | Rationale |
|-------|------|-------|-----------|
| 1 | Story Intake & Clarification | **opus** | Complex reasoning: platform alignment analysis, ambiguity resolution, PO escalation judgment |
| 2 | Scoped Reverse Engineering | sonnet | Structured task: file reading, pattern extraction, summarization |
| 3 | Workflow Planning | **opus** | Architectural judgment: dependency ordering, decision points, risk assessment |
| 4 | Implementation (per repo) | sonnet | Follows approved plan: code generation within defined boundaries |
| 5 | Test Generation & Execution | sonnet | Structured mapping: ACs → test cases, well-defined inputs |
| 6 | AI Reviewer Sub-Agent | **opus** | Dual-track review: quality judgment + fintech baseline enforcement |
| 7 | PR Creation | sonnet | Templated output: structured PR descriptions, git operations |
| 8 | Human Review & Merge | sonnet | Coordination: merge ordering, audit trail closure |

## Model Switching Instructions

For stages marked **opus**: spawn as a sub-agent with `model: opus` or prompt the user to switch model
before beginning that stage with:
> "Stage [N] — [Name] uses Opus for best results. Please switch model if not already on Opus (/model opus), then confirm to continue."

For stages marked sonnet: proceed with current session model (Sonnet is the default).

## Stage Gates

| Stage | Gate |
|-------|------|
| 1 | Developer confirms requirements-verification.md — including Platform Alignment section |
| 2 | Developer confirms no affected areas missed |
| 3 | Developer approves plan — primary human judgment checkpoint |
| 4 | Developer reviews diff per repo before next repo |
| 5 | All tests green, coverage confirmed |
| 6 | All blocking findings resolved, sign-off obtained |
| 7 | PRs open with structured descriptions |
| 8 | PRs merged, story closed, audit trail complete |

## Task Classification

This command is for **standard story-level work** (cross-component, has ACs, goes to PR review).
For hotfixes or architectural changes, adapt stage depth as needed but maintain the gate structure.
