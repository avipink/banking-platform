# AI-DLC 8-Stage Run Template — Standard Story Orchestration

> **Purpose**: Defines which AI-DLC workflow stages to execute for a standard Jira story, in what
> order, and with what story-specific constraints. This file is an **orchestration selector** —
> it does NOT redefine how stages work. Stage execution is fully governed by `CLAUDE.md` and the
> rule-detail files under `.aidlc-rule-details/`. Any conflict between this file and CLAUDE.md
> must be resolved in favour of CLAUDE.md.
>
> **Scope**: One story, end-to-end — from Jira intake to merged PR  
> **Fill in**: `[STORY-KEY]` and `[story title]` when starting a run

---

## Pre-Run Checklist

- [ ] Story selected and confirmed in Jira (key: `[STORY-KEY]`)
- [ ] Codebase reverse engineering artifacts exist (`aidlc-docs/product-architecture.md`)
- [ ] All affected repos are checked out locally and build cleanly
- [ ] Feature branch naming convention agreed (e.g., `feature/[STORY-KEY]-[repo-name]`)

---

## Stage Selection Rationale

This template selects 8 stages for a standard cross-repo story with ACs and a PR review gate.

**Stages 1–5** are standard AI-DLC workflow stages — each delegates fully to CLAUDE.md and its
rule-detail file for execution. This template only adds story-specific constraints on top.

**Stages 6–8** are 8-stage template extensions. The standard AI-DLC Operations phase
(`operations/operations.md`) is a placeholder scoped to future deployment and monitoring work —
it does not define code review, PR creation, or human merge gates. These three stages implement
that Operations gate for fintech story delivery and are fully defined inline in this template.

| Stage | Name | AI-DLC Phase | Rule Detail | Model | Why Always Included |
|-------|------|--------------|-------------|-------|---------------------|
| 1 | Story Intake & Clarification | Inception → Requirements Analysis | `inception/requirements-analysis.md` ✅ | opus | Every story needs verified requirements and platform alignment |
| 2 | Scoped Reverse Engineering | Inception → Reverse Engineering | `inception/reverse-engineering.md` ✅ | sonnet | Brownfield platform — must scope impact before planning |
| 3 | Workflow Planning | Inception → Workflow Planning | `inception/workflow-planning.md` ✅ | opus | Cross-repo dependency ordering requires explicit planning |
| 4 | Implementation (per repo) | Construction → Code Generation | `construction/code-generation.md` ✅ | sonnet | Every story requires implementation |
| 5 | Test Generation & Execution | Construction → Build and Test | `construction/build-and-test.md` ✅ | sonnet | All ACs must be test-verified before review |
| 6 | AI Reviewer Sub-Agent | (8-stage extension — defined inline) | — | opus | Fintech baseline enforcement before human review |
| 7 | PR Creation | (8-stage extension — defined inline) | — | sonnet | Structured PRs required for human review |
| 8 | Human Review & Merge | (8-stage extension — defined inline) | — | sonnet | Human gate before merge; audit trail closure |

**How to apply model assignment:**
- **opus stages**: ALWAYS spawn as a sub-agent with `model: opus`. Do NOT stop to prompt the
  user to switch model — spawn automatically and present the result at the stage gate.
- **sonnet stages**: proceed with the current session model.

**Execution order**: stages must run in strict sequence — 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8.
Do not begin a stage until the previous stage's approval gate is passed.

---

## Stage 1 — Story Intake & Clarification

**Delegates to**: CLAUDE.md → Requirements Analysis  
**Rule detail**: `.aidlc-rule-details/inception/requirements-analysis.md`  
**Model**: opus

**Inputs**
- Jira story: `[STORY-KEY]` — `[story title]`
- `CLAUDE.md` ecosystem map + Verified Platform State section
- Existing RE artifacts under `aidlc-docs/inception/reverse-engineering/`

**8-stage constraints** *(applied on top of CLAUDE.md defaults)*
- Platform Alignment check is **mandatory** — cross-reference every story assumption against
  the Verified Platform State section in `CLAUDE.md`. If misalignments are found, follow the
  CLAUDE.md Requirements Analysis escalation sequence — which requires **both** a blocking
  in-session prompt to the developer AND a Jira comment to the Product Owner (per `## Team
  Registry`) presenting Option A (proceed descoped) / Option B (pause for platform discussion).
- Clarification depth: standard (functional + non-functional requirements).

**Output artifacts**
- `aidlc-docs/inception/requirements/requirements-verification.md`
  - Original story summary
  - `## Platform Alignment` — invalidated assumptions + resolution, or "No misalignments found"
  - Clarification questions + developer answers
  - Resolved scope decisions and constraints

**Approval gate**: Developer confirms `requirements-verification.md` is complete and accurate —
including the Platform Alignment section — before Stage 2.

---

## Stage 2 — Scoped Reverse Engineering

**Delegates to**: CLAUDE.md → Reverse Engineering (scoped, brownfield)  
**Rule detail**: `.aidlc-rule-details/inception/reverse-engineering.md`  
**Model**: sonnet

**Inputs**
- Approved `requirements-verification.md`
- Repo map from `CLAUDE.md` ecosystem map
- Existing RE artifacts (`aidlc-docs/inception/reverse-engineering/`)

**8-stage constraints**
- Scope to affected repos only — do not re-run full RE if current artifacts are recent.
- Produce one Scoped Context Summary per affected repo; do not combine multiple repos into one file.

**Output artifacts**
- `aidlc-docs/inception/reverse-engineering/scoped-context-[repo-name].md` (one per affected repo)
  - **File header must include**: `**Story**: [STORY-KEY] — [story title]` on line 2
  - Affected files and classes
  - Existing patterns to follow
  - Gaps or inconsistencies relevant to this story
  - Integration points with other affected repos

**Approval gate**: Developer confirms no affected areas were missed before Stage 3.

---

## Stage 3 — Workflow Planning

**Delegates to**: CLAUDE.md → Workflow Planning  
**Rule detail**: `.aidlc-rule-details/inception/workflow-planning.md`  
**Model**: opus

**Inputs**
- Approved `requirements-verification.md`
- Approved scoped context summaries from Stage 2

**8-stage constraints**
- Plan must cover all affected repos in dependency order (shared libs first rule from `CLAUDE.md`
  dependency flow).
- Every open architectural decision point (DP-N) must be flagged for developer resolution before
  Stage 4 begins. Blocking DPs must be explicitly resolved at the Stage 3 gate.
- Test strategy must be defined per repo at planning time (unit / integration / contract /
  compliance); not deferred to Stage 5.

**Output artifacts**
- `aidlc-docs/inception/plans/workflow-plan.md`
  - Execution order with dependency rationale
  - Per-repo task breakdown with checkboxes (files, changes, approach)
  - Test strategy per change
  - Architectural decision points (DP-N) flagged for human resolution
  - Definition of Done per repo

**Approval gate**: Developer approves plan — including resolution of all blocking DPs.
This is the primary human judgment checkpoint before any code is written.

---

## Stage 4 — Implementation (per affected repo)

**Delegates to**: CLAUDE.md → Construction Phase → Code Generation  
**Rule detail**: `.aidlc-rule-details/construction/code-generation.md`  
**Model**: sonnet

**Inputs**
- Approved `workflow-plan.md`
- Scoped context summaries from Stage 2

**8-stage constraints**
- Execute one repo at a time, strictly in the dependency order from Stage 3.
- Create feature branch `feature/[STORY-KEY]-[repo-name]` before making any changes.
- Mark each `workflow-plan.md` checkbox immediately after completing that task — not in batch.
- Developer must review and approve the diff for the current repo before the next repo begins.
  Do NOT begin the next repo until the current diff is explicitly approved.

**Output artifacts** (per affected repo)
- Feature branch with committed changes
- Updated checkboxes in `aidlc-docs/inception/plans/workflow-plan.md`
- `aidlc-docs/construction/[repo-name]/code/implementation-summary.md`
  - Files created or modified
  - Key decisions made during implementation
  - Deviations from plan (if any) with rationale

**Approval gate**: Developer reviews and approves diff per repo before moving to the next repo.

---

## Stage 5 — Test Generation & Execution

**Delegates to**: CLAUDE.md → Construction Phase → Build and Test  
**Rule detail**: `.aidlc-rule-details/construction/build-and-test.md`  
**Model**: sonnet

**Inputs**
- Implemented code from Stage 4
- Acceptance Criteria from `requirements-verification.md`
- Test strategy defined in `workflow-plan.md`

**8-stage constraints**
- One test file per logical concern per repo (unit, controller, repository, exception handler,
  compliance) — do not collapse all tests into one file.
- All tests must be green before presenting the Stage 5 gate. Red tests are not a gate item;
  fix failures before surfacing the gate.
- Compliance tests (PII absence, audit log coverage) are mandatory for any story touching
  payment processing, account data, or audit logging.

**Output artifacts** (per affected repo)
- Test files in `src/test/kotlin/...` (standard location per repo)
- `aidlc-docs/construction/build-and-test/test-results-[repo-name].md`
  - Test suite summary (count, pass/fail, test class breakdown)
  - AC-to-test traceability (which test covers which AC)
  - Any known gaps or deferred test cases with rationale

**Approval gate**: All tests green. Developer confirms coverage is acceptable before Stage 6.

---

## Stage 6 — AI Reviewer Sub-Agent

> **8-stage extension** — no CLAUDE.md equivalent. Fully defined here.

**Model**: opus (spawned as sub-agent automatically)

**Inputs**
- All diffs from Stage 4 (read from feature branches)
- Test results from Stage 5
- Story constraints and compliance requirements from `requirements-verification.md`

**Review tracks**

**Track 1 — General Quality**
- Kotlin idioms and null safety (`!!` operator forbidden in production code)
- Exhaustive sealed class branch coverage in `when` expressions
- SOLID principles and clean architecture alignment
- Naming, readability, and duplication
- Consistency with existing patterns identified in Stage 2

**Track 2 — Fintech Baseline**
- PII exposure: `holderName`, `balance`, account numbers must not appear in unclassified logs
- Audit log coverage: every state-changing operation must emit a structured log entry
- Idempotency on state changes: duplicate requests must be handled
- Error contracts: sealed types from `banking-contracts` used throughout — no raw strings
- BigDecimal for all monetary comparisons — no Double or Float
- No hardcoded credentials or service URLs added

**Output artifacts** (per affected repo)
- `aidlc-docs/construction/[repo-name]/review/ai-review-[repo-name].md`
  - Finding per track (compliant / non-compliant / N/A with rationale)
  - Blocking findings (must fix before PR)
  - Advisory findings (recommended, non-blocking)
  - Sign-off line: `AI REVIEWER: APPROVED` or `AI REVIEWER: BLOCKED — [reason]`

**Approval gate**: All blocking findings resolved. AI sign-off line reads `APPROVED` before Stage 7.

---

## Stage 7 — PR Creation

> **8-stage extension** — no CLAUDE.md equivalent. Fully defined here.

**Model**: sonnet

**Inputs**
- Approved diffs from Stage 4 (feature branches)
- Test results from Stage 5
- AI reviewer sign-off from Stage 6

**Process**
1. Push each feature branch to remote (one per affected repo)
2. Create one PR per repo with the structured description below
3. Link each PR to Jira story `[STORY-KEY]`
4. Log all PR URLs in `aidlc-docs/audit.md`
5. Present PRs in dependency order (same as Stage 4 implementation order)

**PR description template**
```
## Jira Story
[STORY-KEY] — [story title]

## Summary
- [what changed and why]
- [key design decision made]
- [compliance concern addressed]

## Repos Affected
| Repo | Branch | Changes |
|------|--------|---------|
| [repo] | feature/[STORY-KEY]-[repo] | [summary] |

## Test Coverage Added
- Unit tests: [count] new
- Integration / slice tests: [count] new
- Compliance tests: [count] new

## AI Reviewer Sign-Off
- General Quality: ✅ / ❌
- Fintech Baseline: ✅ / ❌
- Blocking findings resolved: ✅ / ❌

## Fintech Checklist
- [ ] No PII in logs or unclassified event payloads
- [ ] Audit log entry on every state change
- [ ] Error contracts use sealed types from banking-contracts
- [ ] BigDecimal used for all monetary comparisons
- [ ] No hardcoded credentials or service URLs added
```

**Output artifacts**
- One open PR per affected repo (URLs logged in `audit.md`)

**Approval gate**: All PRs open with structured descriptions and linked to `[STORY-KEY]`.

---

## Stage 8 — Human Review & Merge

> **8-stage extension** — no CLAUDE.md equivalent. Fully defined here.

**Model**: sonnet

**Inputs**
- Open PRs from Stage 7
- AI review summaries from Stage 6

**Process**
1. Human reviewer reads AI review summary as pre-read context
2. Reviews diff focusing on architectural decision points flagged in Stage 3
3. Approves or requests changes
4. Merge PRs in dependency order (same order as Stage 4)
5. Verify all services build and tests pass post-merge on the base branch
6. Close Jira story `[STORY-KEY]`
7. Archive story artifacts:
   a. Copy `aidlc-docs/inception/` → `aidlc-docs/stories/[STORY-KEY]/inception/`
   b. Copy `aidlc-docs/construction/` → `aidlc-docs/stories/[STORY-KEY]/construction/`
   c. Copy `aidlc-docs/aidlc-state.md` → `aidlc-docs/stories/[STORY-KEY]/aidlc-state.md`
      (snapshot of stage completion status for this story)
8. Clear working areas: delete all files under `aidlc-docs/inception/` and
   `aidlc-docs/construction/` — prevents orphaned artifacts contaminating the next story
9. Reset `aidlc-docs/aidlc-state.md` for the next story

**Output artifacts**
- Merged PRs (URLs in `audit.md`)
- `aidlc-docs/stories/[STORY-KEY]/` — complete, browsable snapshot:
  - `inception/` — requirements, scoped RE, workflow plan
  - `construction/` — implementation summaries, test results, AI reviews
  - `aidlc-state.md` — stage completion record for this story
- Closed Jira story
- Final audit entry in `audit.md`:

```
## Story Complete — [STORY-KEY]
[STORY-KEY] → requirements-verification.md → scoped-context → workflow-plan.md
→ implementation (per repo) → tests green → AI review approved
→ PRs merged → story closed → artifacts archived to aidlc-docs/stories/[STORY-KEY]/
```

**Approval gate**: PRs merged, story closed, artifact archive confirmed.

---

## Cost & Value Logging

At every stage approval gate, before presenting the gate to the developer:

1. Run `/cost` to get the current session cumulative cost
2. Compute delta from the previous stage's cumulative cost (track running total in-session)
3. Append a structured entry to `aidlc-docs/audit.md`:

```markdown
## Stage [N] — [Stage Name] — Cost & Value Log
**Timestamp**: [ISO 8601]
**Token cost (this stage)**: ~$[delta]
**Key output**: [artifact name or action completed]
**Findings surfaced**: [count + brief description, or "none"]
**Scope changes**: [yes + description, or "no"]
**Value rating**: [High / Medium / Low] — [one-line rationale]

Rating rules:
- High = produced a finding that changed scope, caught an assumption, or blocked a bad path
- Medium = confirmed existing understanding, no surprises
- Low = stage executed but output was predictable/trivial given inputs

---
```

**Note**: `/cost` is session-cumulative. If the session was restarted mid-run, log
"session restarted — delta unavailable" and record the new baseline.

After Stage 8, append a run summary to `aidlc-docs/audit.md`:

```markdown
## Run Summary — Cost & Value
**Story**: [STORY-KEY] — [title]
**Total session cost**: ~$[total]

| Stage | Cost | Value | Key Finding |
|-------|------|-------|-------------|
| 1 | $[n] | High/Med/Low | [one line] |
| 2 | $[n] | High/Med/Low | [one line] |
| 3 | $[n] | High/Med/Low | [one line] |
| 4 | $[n] | High/Med/Low | [one line] |
| 5 | $[n] | High/Med/Low | [one line] |
| 6 | $[n] | High/Med/Low | [one line] |
| 7 | $[n] | High/Med/Low | [one line] |
| 8 | $[n] | High/Med/Low | [one line] |

**Highest cost stage**: Stage [N] ($[amount])
**Highest value stage**: Stage [N] ([rationale])
**Tuning candidates**: [stages where Low value rating was assigned]

---
```

---

## Artifact Map

```
aidlc-docs/
├── inception/                                        ← working area (reset after Stage 8 archive)
│   ├── requirements/
│   │   └── requirements-verification.md             ← Stage 1
│   ├── reverse-engineering/
│   │   └── scoped-context-[repo-name].md            ← Stage 2 (one per affected repo)
│   └── plans/
│       └── workflow-plan.md                         ← Stage 3
├── construction/                                     ← working area (reset after Stage 8 archive)
│   ├── [repo-name]/
│   │   ├── code/
│   │   │   └── implementation-summary.md            ← Stage 4 (one per affected repo)
│   │   └── review/
│   │       └── ai-review-[repo-name].md             ← Stage 6 (one per affected repo)
│   └── build-and-test/
│       └── test-results-[repo-name].md              ← Stage 5 (one per affected repo)
├── stories/
│   └── [STORY-KEY]/                                 ← Stage 8 archive (permanent)
│       ├── inception/                               ← snapshot of inception/ at story close
│       └── construction/                            ← snapshot of construction/ at story close
├── aidlc-state.md                                   ← active story + stage status
└── audit.md                                         ← append-only, never reset
```
