# AI-DLC Run Validation — Three Layers

> Purpose: Guide for tracking and validating results of an 8-stage AI-DLC story run (Jira story → merged PR)

---

## Layer 1 — Per-Stage Gate Artifacts

Each stage produces a specific artifact. Inspect it before approving the gate — do not approve on trust alone.

| Stage | Artifact to inspect | What to validate |
|-------|---------------------|------------------|
| 1 — Story Intake | `requirements-verification.md` | Platform Alignment section present; assumptions cross-referenced; no phantom infra (Kafka, Redis, OAuth, etc.) |
| 2 — Scoped RE | Scoped context summaries (one per repo) | Affected files/classes correctly identified; no missed dependencies |
| 3 — Workflow Planning | `workflow-plan.md` | Execution order respects dependency graph; per-repo task breakdown is specific enough to code from |
| 4 — Implementation | Git diff per repo | Changes match plan scope exactly; no scope creep (see **Explanation Note** below); no `!!` operators; no hardcoded URLs |
| 5 — Tests | Test output + coverage | All tests green; every Acceptance Criterion has a corresponding test case |
| 6 — AI Reviewer | Review report | Both tracks (General Quality + Fintech Baseline) signed off; no blocking findings left open |
| 7 — PR Creation | PR description on GitHub | All 5 fintech checklist items checked; AI sign-off block present (see **Explanation Note** below); readable cold |
| 8 — Merge | `audit.md` final entry + Jira story | Merge order respected (same as Stage 4 execution order); story transitioned to Done |

> **Explanation Note — What "no scope creep" means**
> The Stage 4 diff must stay within the boundaries approved in Stage 3. If `workflow-plan.md` names
> specific files to modify, the diff should touch only those files. Any additional change — a "related
> improvement" or "while we're here" addition — is scope creep, even if well-intentioned.
>
> This matters especially in an AI context: the model may identify opportunities during implementation
> and act on them without flagging it. These unplanned changes bypass the human judgment checkpoint
> at Stage 3, may introduce unintended side effects, and make the diff harder to review at Stages 4 and 6.
>
> **How to detect it**: at the Stage 4 gate, cross-check every changed file against `workflow-plan.md`.
> Any file not in the plan requires an explicit justification — not silence.

> **Explanation Note — What "AI sign-off block present" means**
> The PR description must include the AI Reviewer sign-off section produced at Stage 6:
>
> ```
> ## AI Reviewer Sign-Off
> - General Quality Track: ✅ / ❌
> - Fintech Baseline Track: ✅ / ❌
> - Blocking findings resolved: ✅ / ❌ [list if any]
> ```
>
> This section is the traceability link between Stage 6 (AI review completed) and Stage 7 (PR created).
> A human reviewer must be able to see at a glance that automated review was done and what the verdict was —
> without having to go back and read the audit log.
> If the block is missing or left blank, the PR fails the Stage 7 gate regardless of code quality.

---

## Layer 2 — Cross-Cutting Signals

Watch these across the entire run, not just at individual gates.

**`audit.md`**
Should show a continuous, timestamped log of every stage, every approval, and every raw user input.
Gaps in the log = the workflow was shortcut somewhere.

**`aidlc-state.md`**
Shows which stages were executed vs. skipped. After the run, verify every skip was justified.
A skipped Functional Design on a story introducing a new data model is a red flag.

**Scope creep detector**
Compare Stage 3 `workflow-plan.md` against the Stage 4 diffs.
If code was changed in files not listed in the plan, it needs explicit justification — not silence.

**Platform alignment signal**
If Stage 1 produced zero alignment findings on a complex story, be skeptical.
The gate only has value if it fires when it should. A clean pass on a trivial story is expected;
a clean pass on a story referencing external integrations warrants a second look.

**Cost & value log**
Each stage appends a cost delta and value rating to `audit.md` (see `run-story.md` for format).
Low-value ratings that repeat across multiple runs are tuning candidates in the rule files.

---

## Layer 3 — Session-Level Retrospective

Run after each story completes. The most effective learning mechanism for a training context.

**Five questions to answer:**

1. **Did Stage 1 catch anything real?**
   If not — was the story genuinely clean, or did the platform alignment gate miss something?

2. **Did Stage 3 plan match Stage 4 reality?**
   Divergence (unplanned files changed, planned files untouched) reveals planning quality and scope discipline.

3. **Did the AI Reviewer find anything the developer missed?**
   Measures the effectiveness of the review track. If it never finds anything, the track needs sharpening.

4. **Was the PR description sufficient for a human reviewer with no prior context?**
   Read it as if you know nothing about the story. Would you approve it?

5. **Which stages consumed the most tokens for the least delta?**
   Candidates for depth reduction (`minimal` depth level) in future runs of similar story types.

---

## Practical Guidance

- Run `/cost` at every stage gate to track per-stage token spend (session-cumulative delta)
- Use the `run-scorecard.md` template alongside each session for a quick pass/fail signal
- After 3-4 runs a pattern emerges — which stages need tuning for your specific story types
