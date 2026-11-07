# HANDOFF — casehub-examples (slot 196)

## Last Session

**Issue:** #830 — regulation predictor quality
**Phase 2 implementation complete + calibration in progress. Three boundary cases remain.**

### What was built
- RegulationDefaults (continuous weights from Barańczuk 2019)
- ShameContext (memory compounding, relational closeness, personality-gated depth)
- Three-layer architecture: CDE derivation → tick participant assembly → generation phase rendering
- Viable alternative threshold (don't pin RM when reappraisal is viable for controllable emotions)
- Temporal staging guidance (avoidant impulse first, then reappraisal progression)
- Surface-level shame → pin RM (deflection, not genuine reappraisal)

### Calibration results (6 eval runs)

**Solved (stable):**
- incoherent-anger: 3/3/3 → **4/4** (temporal staging fix)
- surface-shame: 4/3/2 → **5/5** (pin RM for surface-level)

**Boundary cases (swing 3-4 across runs):**
- mixed-anger: 4/3/3/4 — judge says "betrayal resists reappraisal" (situation-specific)
- compounding-shame: 4/4/4/3/3 — judge says "4th episode, serial suppression failing" (connects to fatigue #817)
- suppressor-shame: 4/4/3/4 — judge says "AD over RM for shame"

**Stable 5s (never vary):** suppressor-anger, reappraiser-anger, anyone-fear, reappraiser-shame

### Key insight for next session

The user observed: the 5s NEVER have variance. The 3-4 boundary cases always have variance. **This is not noise — it's a signal that these cases are genuinely on the boundary.** Either the application model doesn't give the LLM enough context, or the judge rubric doesn't handle these cases well. Both warrant investigation.

## Next Action

**`work continue` — resolve the three boundary cases to stable 4+.**

Approach for next session:
1. **Literature search** — targeted: how do avoidant personalities regulate shame across repeated episodes? Does the literature support the judge's claim that serial suppression degrades? What strategy does the literature recommend for the 4th+ episode? This grounds the fix in evidence rather than prompt engineering.
2. **Examine the judge** — the boundary cases may reveal a rubric limitation. "What this personality WOULD NATURALLY CHOOSE" doesn't distinguish impulse (fast, automatic) from reflective (slow, deliberate). The mixed-anger judge says "betrayal resists reappraisal" — that's a situation judgment, not a personality one.
3. **Consider multi-run eval** — add `@RepeatedTest(3)` or similar to get median scores per case, eliminating single-run variance from the floor measurement.

### Files to read
- `docs/eval/regulation-e2e-phase2-calibrated.json` — latest eval results
- `cognition/.../regulation/RegulationStrategyPredictor.java` — viable alternative + continuous thresholds
- `cognition/.../regulation/EmotionRegulationGenerationPhase.java` — temporal staging + surface-level RM pin
- `cognitive-index/.../CognitiveDerivationEngine.java` — weight tables (lines 138-169)

## Branches

**Neocortex:** `issue-830-regulation-predictor-quality` — 14 commits (Phase 1 + Phase 2 + calibration)
**Examples:** `issue-830-regulation-predictor-quality` — no code changes
**Workspace:** `issue-830-regulation-predictor-quality` — specs, plans, decisions, blog

## Queue Status

Issue #830 active. 11 remaining issues after #830 (including #844 feedback loop).
