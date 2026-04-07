# AI-DLC Basic Workflow — 8-Stage Run Template

> Purpose: Execution template for running a Jira story through the complete AI-DLC Basic Workflow  
> Scope: One story, end-to-end — from Jira intake to merged PR  
> Fill in: `[STORY-KEY]` and `[story title]` when starting a run

---

## Pre-Run Checklist

- [ ] Story selected and confirmed in Jira (key: `[STORY-KEY]`)
- [ ] Codebase reverse engineering artifacts exist (`aidlc-docs/product-architecture.md`)
- [ ] All affected repos are checked out locally and build cleanly
- [ ] Feature branch naming convention agreed (e.g., `feature/[STORY-KEY]-short-description`)

---

## Stage 1 — Story Intake & Clarification

**Goal**: Parse the story, surface ambiguities, produce a verified requirements document.

**Inputs**
- Jira story: `[STORY-KEY]` — `[story title]`

**Process**
1. Orchestrator reads the full story (Business Intent, Scope, Scenarios, Constraints, AC)
2. Identifies ambiguities, missing edge cases, and scope boundary decisions
3. Generates clarification questions (multiple-choice or open-ended as appropriate)
4. Developer answers questions inline
5. Orchestrator resolves any remaining ambiguities

**Output**
- `aidlc-docs/inception/requirements/requirements-verification.md`
  - Original story summary
  - Clarification questions + developer answers
  - Resolved scope decisions
  - Any constraints added or modified as a result

**Approval gate**: Developer confirms requirements-verification.md is complete and accurate before Stage 2.

---

## Stage 2 — Scoped Reverse Engineering

**Goal**: Identify affected repos and produce a targeted context summary for each — no full RE, only what is relevant to this story.

**Inputs**
- Approved `requirements-verification.md`
- Repo map from `CLAUDE.md`
- Existing RE artifacts (`aidlc-docs/product-architecture.md`)

**Process**
1. Orchestrator identifies affected repos from story scope + CLAUDE.md ecosystem map
2. For each affected repo, analyzes:
   - Relevant entry points (controllers, endpoints)
   - Service layer methods that will change
   - Domain models and contracts involved
   - Existing error handling and exception patterns
   - Inter-service dependencies and call chains
   - Test coverage in affected areas
3. Produces one Scoped Context Summary per repo

**Output** (one file per affected repo)
- `aidlc-docs/inception/reverse-engineering/scoped-context-[repo-name].md`
  - Affected files and classes
  - Existing patterns to follow
  - Gaps or inconsistencies relevant to this story
  - Integration points with other affected repos

**Approval gate**: Developer reviews scoped context summaries, confirms no affected areas were missed.

---

## Stage 3 — Workflow Planning

**Goal**: Produce an ordered, dependency-aware implementation plan across all affected repos.

**Inputs**
- Approved `requirements-verification.md`
- Approved scoped context summaries from Stage 2

**Process**
1. Orchestrator determines execution order based on repo dependencies (shared-libs first rule)
2. For each repo, defines:
   - Specific files to create or modify
   - Change description and rationale
   - Dependencies on other repos in this story
3. Defines test strategy per repo (unit, integration, contract, compliance)
4. Identifies the key architectural decision point(s) requiring human judgment
5. Generates workflow visualization (dependency chain)

**Output**
- `aidlc-docs/inception/plans/workflow-plan.md`
  - Execution order with dependency rationale
  - Per-repo task breakdown (files, changes, approach)
  - Test strategy per change
  - Architectural decision points flagged for human review
  - Definition of Done per repo

**Approval gate**: Developer reviews and approves plan — or requests revisions. This is the primary human judgment checkpoint before code is written.

---

## Stage 4 — Implementation (per affected repo)

**Goal**: Implement approved changes in dependency order, one repo at a time.

**Inputs**
- Approved `workflow-plan.md`
- Scoped context summaries from Stage 2

**Process** (repeat for each repo in approved execution order)
1. Create feature branch: `feature/[STORY-KEY]-[repo-name]`
2. Implement changes per the workflow plan
3. Follow existing patterns identified in Stage 2 (naming, error handling, structure)
4. Mark each workflow-plan checkbox as complete immediately after each task
5. Developer reviews diff in VS Code before proceeding to next repo
6. Do not begin next repo until current repo diff is approved

**Output** (per repo)
- Feature branch with committed changes
- Updated checkboxes in `workflow-plan.md`
- `aidlc-docs/construction/[repo-name]/code/implementation-summary.md`
  - Files created/modified
  - Key decisions made during implementation
  - Deviations from plan (if any) with rationale

**Approval gate**: Developer reviews and approves diff per repo before moving to the next.

---

## Stage 5 — Test Generation & Execution

**Goal**: Generate and run a complete test suite covering all scenarios in the story.

**Inputs**
- Implemented code from Stage 4
- Acceptance Criteria from the Jira story
- Scenarios from `requirements-verification.md`

**Process**
1. For each affected repo, generate:
   - **Unit tests** (Kotest/JUnit5): one test per AC checkbox, one test per scenario
   - **Integration tests**: service call chains, error paths, boundary conditions
   - **API contract tests**: request/response shape, error codes, headers
   - **Compliance tests**: audit log field assertions, PII absence checks, classification tags
2. Run all tests locally
3. Fix any failures — do not proceed with red tests

**Output** (per repo)
- Test files in standard `src/test/kotlin` location
- `aidlc-docs/construction/build-and-test/test-results-[repo-name].md`
  - Test suite summary (count, pass/fail)
  - Coverage on changed paths
  - Any known gaps or deferred test cases

**Approval gate**: All tests green. Developer confirms coverage is acceptable before Stage 6.

---

## Stage 6 — AI Reviewer Sub-Agent (Pre-PR)

**Goal**: Dual-track automated review of all changes before PR creation.

**Inputs**
- All diffs from Stage 4
- Test results from Stage 5
- Story constraints and compliance requirements

**Tracks**

**Track 1 — General Quality**
- Kotlin idioms and null safety (`!!` operator forbidden in production code)
- Error handling completeness (all sealed class branches covered)
- SOLID principles and clean architecture alignment
- Naming, readability, and duplication
- Consistency with existing patterns identified in Stage 2

**Track 2 — Fintech Baseline**
- PII exposure check (no `holderName`, `balance`, or account numbers in unclassified logs)
- Audit log coverage (every state-changing operation has an audit entry)
- Idempotency on state changes (duplicate requests handled)
- Data classification tags present and correct (`CONFIDENTIAL` where required)
- Secrets and credentials — no hardcoded values
- Regulatory compliance markers (error codes, audit fields) present

**Output**
- `aidlc-docs/construction/[repo-name]/review/ai-review-[repo-name].md`
  - Finding per track (compliant / non-compliant / N/A)
  - Blocking findings (must fix before PR)
  - Advisory findings (recommended but non-blocking)
  - Sign-off status: `AI REVIEWER: APPROVED` or `AI REVIEWER: BLOCKED — [reason]`

**Approval gate**: All blocking findings resolved. AI reviewer sign-off obtained before Stage 7.

---

## Stage 7 — PR Creation

**Goal**: Create one structured PR per affected repo with full context for human reviewers.

**Inputs**
- Approved diffs from Stage 4
- Test results from Stage 5
- AI reviewer sign-off from Stage 6

**Process** (one PR per affected repo, in implementation order)
1. Push feature branch to remote
2. Create PR with structured description (see template below)
3. Link PR to Jira story `[STORY-KEY]`

**PR Description Template**
```
## Jira Story
[STORY-KEY] — [story title]

## Summary
- [bullet: what changed and why]
- [bullet: key design decision made]
- [bullet: compliance concern addressed]

## Repos Affected
| Repo | Branch | Changes |
|------|--------|---------|
| [repo] | feature/[STORY-KEY]-[repo] | [summary] |

## Test Coverage Added
- Unit tests: [count] new tests
- Integration tests: [count] new tests
- Compliance tests: [count] new tests

## AI Reviewer Sign-Off
- General Quality Track: ✅ / ❌
- Fintech Baseline Track: ✅ / ❌
- Blocking findings resolved: ✅ / ❌ [list if any]

## Fintech Checklist
- [ ] No PII in logs or unclassified event payloads
- [ ] Audit log entry on every state change
- [ ] Error contracts use sealed types from banking-contracts
- [ ] BigDecimal used for all monetary comparisons
- [ ] No hardcoded credentials or service URLs added
```

**Output**
- One open PR per affected repo
- PR URLs logged in `aidlc-docs/audit.md`

---

## Stage 8 — Human Review & Merge

**Goal**: Human code review informed by AI summary. Merge and close the audit trail.

**Inputs**
- Open PRs from Stage 7
- AI review summaries from Stage 6

**Process**
1. Human reviewer reads AI review summary as pre-read context
2. Reviews diff with focus on the architectural decision points flagged in Stage 3
3. Approves or requests changes
4. Merge PRs in dependency order (same order as Stage 4 implementation)
5. Verify all services build and tests pass post-merge
6. Close Jira story `[STORY-KEY]`

**Audit trail closure**
Update `aidlc-docs/audit.md` with final entry:
```
## Story Complete — [STORY-KEY]
Story → requirements-verification.md → scoped-context → workflow-plan.md
→ implementation (per repo) → tests green → AI review approved
→ PRs merged → story closed
```

**Output**
- Merged PRs (links in audit.md)
- Closed Jira story
- Complete audit trail: `Story → Plan → Code → Tests → Review → Merge`

---

## Artifact Map

```
aidlc-docs/
├── inception/
│   ├── requirements/
│   │   └── requirements-verification.md          ← Stage 1
│   ├── reverse-engineering/
│   │   └── scoped-context-[repo-name].md         ← Stage 2 (one per repo)
│   └── plans/
│       └── workflow-plan.md                      ← Stage 3
├── construction/
│   ├── [repo-name]/
│   │   ├── code/
│   │   │   └── implementation-summary.md         ← Stage 4
│   │   └── review/
│   │       └── ai-review-[repo-name].md          ← Stage 6
│   └── build-and-test/
│       └── test-results-[repo-name].md           ← Stage 5
└── audit.md                                      ← Updated throughout
```
