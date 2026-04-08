# Claude Code Prompt — Generate Training User Story

Copy everything below the line and paste it into your Claude Code cross-repo session:

---

I need your help creating a Jira user story for an AI-SDLC training demonstration.

This story will be used to walk students through the COMPLETE AI-DLC Basic Workflow —
all 8 stages from Jira Story to Merged PR. The story must exercise every stage
meaningfully so students see real value at each step, not just pass-through.

Here are the 8 stages the story must exercise:

## THE WORKFLOW THIS STORY MUST FIT

**STAGE 1 — Story Intake & Clarification**
The orchestrator reads the story, parses intent, and generates clarification questions
about ambiguities, edge cases, and scope boundaries. The developer answers inline,
producing a `requirements-verification.md`. The story must have enough complexity
to trigger meaningful clarification questions — not be so obvious that Stage 1 is trivial.

**STAGE 2 — Scoped Reverse Engineering**
The orchestrator identifies affected repos from the story scope and the CLAUDE.md repo map,
then analyzes each affected repo (relevant modules, entry points, models, dependencies,
events). Output is a Scoped Context Summary per repo. The story MUST touch at least
2 repos/services so this stage demonstrates cross-repo analysis.

**STAGE 3 — Workflow Planning**
The orchestrator generates a `workflow-plan.md` with changes per repo, execution order,
dependency chain across repos, task breakdown, and test strategy per change.
The developer reviews and approves (or requests revisions). The story should have a
non-trivial execution order — e.g., shared-libs first, then service A, then service B
consuming changes from A/shared-libs.

**STAGE 4 — Implementation (per affected repo)**
Feature branch creation, then implementation following the dependency order:
shared-libs (if affected, always first) → service A → service B.
Developer reviews diffs in VS Code. The story should produce real, reviewable code
changes — not just config or boilerplate.

**STAGE 5 — Test Generation & Execution**
Generate/update unit tests (Kotest/JUnit5), integration tests (DB, messaging, service
calls), and API contract tests. Run all tests locally. The story must have testable
behavior with clear scenarios that map to test cases.

**STAGE 6 — AI Reviewer Sub-Agent (Pre-PR)**
Dual-track review:
- **General Quality Track**: code style, Kotlin idioms, error handling, SOLID/clean
  architecture, duplication, naming/readability
- **Fintech Baseline Track**: PII exposure check, audit log coverage, idempotency on
  state changes, secrets/credential handling, data classification tags, regulatory
  compliance markers

The story MUST involve at least one fintech compliance concern (audit logging, PII,
idempotency, data classification) so Stage 6 has real findings to surface — not
just a clean pass.

**STAGE 7 — PR Creation**
Structured PR description with Jira link, repos affected, summary of changes,
test coverage added, AI reviewer sign-off, fintech checklist status.
One PR per affected repo.

**STAGE 8 — Human Review & Merge**
Human code review using AI review summary as context. Full audit trail:
`Story → Plan → Code → Tests → Review → Merge` in `aidlc-docs/audit.md`.

## WHAT I NEED YOU TO DO

**Step 1 — ANALYZE** the current codebase and architecture artifacts to identify
a feature or enhancement that naturally exercises all 8 stages above.

The ideal candidate should:
a) Touch at least 2 repos/services (ideally shared-libs + 1-2 services)
   to demonstrate the Stage 2 cross-repo analysis and Stage 4 ordered implementation
b) Involve a banking compliance concern (audit trail, PII handling, idempotency,
   data classification) so Stage 6's fintech review track has real substance
c) Be scoped to roughly 1-2 days of implementation — enough to be meaningful,
   not so large it overwhelms a training session
d) Have clear GIVEN/WHEN/THEN scenarios that translate directly to Kotest/JUnit5
   test cases in Stage 5
e) Require the orchestrator to reason about existing patterns, dependencies,
   and constraints in the codebase — demonstrating Stage 2 and Stage 3 value
f) Have at least one genuine architectural decision point where human review in
   Stage 3 (plan approval) or Stage 8 (PR review) adds real value
g) Generate meaningful clarification questions in Stage 1 — the story should
   have realistic ambiguities or scope boundary decisions

**Step 2 — PROPOSE** 2-3 candidate stories with a one-paragraph summary each:
- What the feature is
- Which repos/services it touches (use actual repo names)
- Why it's a good training demo (which stages it exercises particularly well)
- The key compliance/architectural constraint the orchestrator must respect
- What clarification questions Stage 1 would likely generate

**Step 3 — After I pick one**, GENERATE the full Jira story description with:

```
## Business Intent
(As a... I want... so that...)

## Scope
### In Scope
### Out of Scope

## Actors
(Users, systems, services — use real service names from the codebase)

## Behavior (Scenarios)
### Scenario N: {Title}
GIVEN ... WHEN ... THEN ...
(Include: happy path, error/edge cases, AND at least one compliance scenario
 e.g., "GIVEN a request with PII fields WHEN processed THEN audit log is emitted
 with classification 'Confidential'")

## Constraints
(Reference actual system constraints: regulatory, performance, security, integration.
 Be specific — name the regulations, the SLAs, the integration points.)

## Acceptance Criteria
(Checkboxes, testable, mapped to test types:
 - unit test criteria
 - integration test criteria
 - API contract test criteria
 - compliance/audit criteria)

## Technical Considerations
(Reference actual services, APIs, data models, patterns already in the codebase.
 Call out the implementation order: shared-libs → service A → service B.
 Note where implementation must align with existing architectural decisions.)

## Cross-Repo Impact
(Which repos are affected, what changes where, dependency chain,
 integration points between repos)

## Open Questions
(Genuine ambiguities that Stage 1 clarification should surface.
 These should be realistic architectural or scope decisions, not padding.)
```

Start with Step 1 and Step 2 — analyze and propose candidates.
