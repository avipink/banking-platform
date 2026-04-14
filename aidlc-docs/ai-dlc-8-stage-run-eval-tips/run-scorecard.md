# AI-DLC Run Scorecard

> Purpose: Per-run quality signal. Fill in during the session at each stage gate.
> One file per story run — copy and rename to `run-scorecard-[STORY-KEY].md` before starting.

---

## Run Identity

| Field | Value |
|-------|-------|
| Story key | |
| Story title | |
| Date | |
| Operator | |
| Total session cost | |

---

## Stage Scorecard

| Stage | Gate artifact present? | Key finding (or "none") | Scope change? | Cost | Value |
|-------|----------------------|-------------------------|---------------|------|-------|
| 1 — Story Intake | Y / N | | Y / N | $    | H / M / L |
| 2 — Scoped RE | Y / N | | Y / N | $    | H / M / L |
| 3 — Workflow Planning | Y / N | | Y / N | $    | H / M / L |
| 4 — Implementation | Y / N | | Y / N | $    | H / M / L |
| 5 — Tests | Y / N | | Y / N | $    | H / M / L |
| 6 — AI Reviewer | Y / N | | Y / N | $    | H / M / L |
| 7 — PR Creation | Y / N | | Y / N | $    | H / M / L |
| 8 — Merge | Y / N | | Y / N | $    | H / M / L |

**Value key**: H = High (changed scope / caught assumption / blocked bad path) · M = Medium (confirmed understanding) · L = Low (predictable output, no surprises)

---

## Gate Checklist

### Stage 1 — Story Intake
- [ ] `requirements-verification.md` created
- [ ] `## Platform Alignment` section present
- [ ] No phantom infrastructure assumptions accepted silently
- [ ] PO notified on Jira (if alignment issues found)

### Stage 2 — Scoped RE
- [ ] One scoped context summary per affected repo
- [ ] All affected files/classes identified
- [ ] No missed cross-repo dependencies

### Stage 3 — Workflow Planning
- [ ] `workflow-plan.md` created
- [ ] Execution order respects dependency graph (accounts-core → payments-core → bff)
- [ ] Per-repo task breakdown specific enough to code from

### Stage 4 — Implementation
- [ ] Diff matches plan scope (no unplanned files changed)
- [ ] No `!!` operators in production code
- [ ] No hardcoded service URLs or credentials added
- [ ] Error types use sealed classes from `banking-contracts`

### Stage 5 — Tests
- [ ] All tests green
- [ ] Every Acceptance Criterion covered by at least one test
- [ ] Coverage acceptable (confirm threshold)

### Stage 6 — AI Reviewer
- [ ] General Quality track: approved
- [ ] Fintech Baseline track: approved
- [ ] All blocking findings resolved before gate passed

### Stage 7 — PR Creation
- [ ] One PR per affected repo
- [ ] Jira story linked in PR description
- [ ] AI reviewer sign-off block present
- [ ] Fintech checklist complete (PII, audit log, sealed errors, BigDecimal, no hardcoded URLs)
- [ ] PR readable cold (no prior context needed)

### Stage 8 — Merge
- [ ] PRs merged in dependency order
- [ ] PR URLs logged in `audit.md`
- [ ] Jira story transitioned to Done
- [ ] `audit.md` run summary entry complete

---

## Retrospective (fill in after merge)

**Did Stage 1 catch anything real?**


**Did Stage 3 plan match Stage 4 reality?**


**Did the AI Reviewer find anything the developer missed?**


**Was the PR description sufficient for a cold reviewer?**


**Tuning candidates (stages with Low value rating):**


**Overall run quality**: Excellent / Good / Needs improvement

---
