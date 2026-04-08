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

## Stage Sequence

| Stage | Name | Gate |
|-------|------|------|
| 1 | Story Intake & Clarification | Developer confirms requirements-verification.md |
| 2 | Scoped Reverse Engineering | Developer confirms no affected areas missed |
| 3 | Workflow Planning | Developer approves plan — primary human judgment checkpoint |
| 4 | Implementation (per repo) | Developer reviews diff per repo before next repo |
| 5 | Test Generation & Execution | All tests green, coverage confirmed |
| 6 | AI Reviewer Sub-Agent | All blocking findings resolved, sign-off obtained |
| 7 | PR Creation | PRs open with structured descriptions |
| 8 | Human Review & Merge | PRs merged, story closed, audit trail complete |

## Task Classification

This command is for **standard story-level work** (cross-component, has ACs, goes to PR review).
For hotfixes or architectural changes, adapt stage depth as needed but maintain the gate structure.
