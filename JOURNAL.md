# Design Journal — issue-851-dynamic-regulatory-capacity

## 2026-11-08 — Predictor validation + proactive regulation design

### What happened

Closed #852 (already implemented — dominantEmotionType persistence). Wrote
`PredictorScenarioValidationTest` for #829 — exercised predictor against all 5
BASELINE characters across 4 emotion categories. Finding: 95% prediction rate,
all characters resolve as strong reappraisers (capacity 0.80-1.00), no
RESPONSE_MODULATION predictions. Only penelope-pitstop falls through for ANGER
(CC/RM gap 0.25 < 0.30 threshold). Closed #829.

Designed #819 (proactive regulation — pre-appraisal situation avoidance). Full
brainstorming with 5 decisions, light decision review (17 issues, 7 HIGH). Three
decisions revised: D2 (add RegulationContext to CognitionTickContext instead of
parsing observation text), D3 (appendGoals() instead of registerGoals() — replace
semantics would clobber proactive goals), D4 (use existing redirectTendency instead
of deriving a new avoidance-sensitivity score). D5 added for temporal decay
(Bonanno & Burton 2013 regulatory flexibility). Spec and implementation plan
written — 6 tasks across 3 batches. Ready for execution.

### Key finding: registerGoals() replaces, not appends

`GoalProposalOrchestrator.registerGoals()` does `state.registeredGoals = List.copyOf(goals)`.
Multiple callers (proactive from FOUNDATION, reactive from TERMINAL) would clobber
each other. Fix: add `appendGoals()` with additive semantics.

### Key finding: no suppressor diversity in BASELINE cast

All 5 characters have reappraisal capacity 0.80-1.00. The predictor never pins
RESPONSE_MODULATION. A character with avoidant+conservative+low-autonomy axes
would provide suppressor coverage.

### Artifacts

- Spec: `specs/issue-851-dynamic-regulatory-capacity/2026-11-08-proactive-regulation-design.md`
- Plan: `plans/2026-11-08-proactive-regulation.md`
- Blog: `blog/2026-11-08-mdp07-your-agent-should-be-afraid-of-the-library.md`
- Test: `neocortex/cognition/.../regulation/PredictorScenarioValidationTest.java`
