# Run AI-DLC 8-Stage Story Workflow

Execute the complete 8-stage AI-DLC workflow for a Jira story through all gates from intake to merged PR.

**Story Key**: $ARGUMENTS

## Execution Instructions

1. Load `aidlc-docs/ai-dlc-8-stage-run-template.md` — defines **which** stages run and their
   orchestration constraints (model assignment, gates, artifact contracts). Does NOT define
   how stages execute; that is governed by CLAUDE.md and the rule-detail files.
2. **Authority order when sources conflict**: rule-detail file > CLAUDE.md > 8-stage template.
   The template's constraints narrow or extend CLAUDE.md defaults; they never override them.

## Task Classification

This command is for **standard story-level work** (cross-component, has ACs, goes to PR review).
For hotfixes or architectural changes, adapt stage depth as needed but maintain the gate structure.
