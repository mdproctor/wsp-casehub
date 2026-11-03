# Decisions — Emotion Regulation (#590)

## D1: Strategy scope

**Choice:** All five Gross strategies — design for the complete process model
**Alternatives:**
- Three post-appraisal only (distraction, reappraisal, suppression) — simpler but incomplete model, paints into a corner if goal-level strategies need different abstractions later
- Two core only (reappraisal + suppression) — misses distraction which Petter et al. show dominates at high intensity
**Rationale:** Designing for the full model ensures the abstraction handles both post-appraisal (pipeline-level) and antecedent (goal-level) strategies cleanly. Implementation batched: Batch 1 = post-appraisal three, Batch 2 = goal-level two.
**Trade-offs:** Larger design scope; goal-level strategies touch GoalFormationStrategy which is a different pipeline integration point
**Sources:** Gross 1998 five-strategy process model; casehubio/neocortex#590 issue body
**Exploration:** quick
**Status:** captured
