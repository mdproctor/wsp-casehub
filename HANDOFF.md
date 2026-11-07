# HANDOFF — casehub-examples (slot 196)

## Last Session

**Issue:** #830 — regulation predictor quality
**Phase 2 fully implemented + calibrated. Three boundary cases remain at floor=3.**

### What was built and calibrated (two sessions)

**Implementation (6 tasks, 5 batches):**
- `RegulationDefaults` record with Barańczuk 2019 weight tables in CDE
- Continuous capacity thresholds replacing boolean `isSuppressor`/`isReappraiser`
- `ShameContext` with memory compounding, relational closeness, personality-gated depth
- Three-layer architecture: CDE derivation → tick participant assembly → generation phase rendering
- Extended eval: 11 cases (8 original + 3 shame-specific), strategy + temporal + shame depth judges

**Calibration (6 eval runs, 3 structural fixes):**
- Weight recalibration: avoidant→suppression 0.25→0.10, flexible→reappraisal 0.10→0.15, flexible→redirect +0.05→-0.10
- Viable alternative threshold (0.40): don't pin RM when reappraisal is viable for controllable emotions
- Temporal staging: avoidant impulse first, then reappraisal progression (fixed incoherent-anger 3→4)
- Surface-level shame pins RM: deflection is modulation not reappraisal (fixed surface-shame 2→5)
- Old boolean classification removed entirely

### Calibration state

**Stable 5s (never vary across 6 runs):** suppressor-anger, reappraiser-anger, anyone-fear, reappraiser-shame

**Fixed (were failing, now stable):** incoherent-anger (3→4), surface-shame (2→5)

**Boundary cases (swing 3-4 across runs):**
1. **mixed-anger** — judge says "betrayal of trust resists reappraisal." Situation-specific objection, not personality.
2. **compounding-shame** — judge says "4th episode, serial suppression failing." Episode count not connected to fatigue model.
3. **suppressor-shame** — judge says "AD serves avoidant style better than RM for shame." Falls through, LLM picks inconsistently.

**Key insight:** Cases that score 5 NEVER have judge variance. Cases at 3-4 ALWAYS have variance. Variance is not noise — it is the model telling you where it lacks sufficient context or has architectural gaps.

## Next Action

**`work continue` — resolve the three boundary cases to stable 4+.**

### How to approach this

This is not a prompt engineering problem. The stable 5s prove the architecture works when the model has enough information. The boundary cases fail because the model is MISSING something the judge sees. The next session needs to find what's missing.

**Use first principles.** Each boundary case has a judge objection that points to a specific gap. Don't patch the symptom — understand what psychological mechanism the judge is invoking, find the literature that describes it, and build the mechanism into the model.

**Do a targeted literature search.** The three boundary cases map to three research questions:
1. **Situation-strategy interaction** (mixed-anger): Does the literature distinguish emotion triggers that resist specific strategies? Betrayal, moral violations, relational wounds — are some triggers "reappraisal-resistant" regardless of personality? Look for: Sheppes 2011 (strategy-situation fit), Troy et al. 2013 (controllability × reappraisal), Ford & Troy 2019 (regulatory flexibility).
2. **Serial suppression dynamics** (compounding-shame): What happens to suppression effectiveness across repeated episodes of the same emotion? How does episode count interact with strategy choice? Look for: Gross & John 2003 (habitual suppression costs), Butler et al. 2003 (physiological rebound), English et al. 2017 (flexibility vs rigidity).
3. **Strategy disambiguation for avoidant shame** (suppressor-shame): What does the avoidant personality ACTUALLY do with shame it can't reappraise and can't suppress? Is there a distinct regulation mode the Gross taxonomy doesn't capture? Look for: Nathanson 1992 (compass of shame — four poles: withdrawal, attack self, attack other, avoidance), Elison et al. 2006 (compass validation), De Hooge et al. 2018 (shame action tendencies).

**Try experiments.** Run the eval with specific modifications and observe what changes:
- Pin different strategies for each boundary case and see what the judge scores — this isolates whether the problem is strategy selection or narrative quality
- Modify the judge rubric to distinguish impulse vs reflective choice — does this change the scores?
- Add situation-awareness to the prompt ("betrayal" as a trigger category) and see if mixed-anger stabilises
- Connect episode count to fatigue rendering and see if compounding-shame stabilises

**Think deeply about what "3" means.** A score of 3 is "acceptable but the personality would more naturally choose something else." For boundary cases, the judge is saying there IS a better answer — the model just can't see it. What information would let the model see it? That's the question.

**Use max effort.** This is the hardest part of the regulation work — the cases where personality-only prediction meets its limits. The answers are in the literature, not in parameter tuning.

### Files to read first
- `docs/eval/regulation-e2e-phase2-calibrated.json` — latest eval results with per-case reasoning
- `docs/research/2026-11-07-regulation-continuous-weights-literature-review.md` — literature review + calibration results section
- `docs/research/2026-11-05-post-era6-regulation-and-identity.md` — full regulation research thread
- `cognition/.../regulation/RegulationStrategyPredictor.java` — viable alternative + continuous thresholds
- `cognition/.../regulation/EmotionRegulationGenerationPhase.java` — temporal staging + surface-level RM
- `caps-llm-classifier/.../eval/RegulationEndToEndEvalTest.java` — eval test with all 11 cases

### Filed this session
- casehubio/neocortex#844 — Regulation prediction feedback loop (track outcomes to validate predictions)

## Branches

**Neocortex:** `issue-830-regulation-predictor-quality` — 16 commits (Phase 1 + Phase 2 + calibration)
**Examples:** `issue-830-regulation-predictor-quality` — no code changes
**Workspace:** `issue-830-regulation-predictor-quality` — specs, plans, decisions, 3 blog entries

## Queue Status

Issue #830 active. 11 remaining issues after #830 (including #844 feedback loop).
