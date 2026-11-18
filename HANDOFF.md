# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-18a:** Closed #1098 — replaced keyword intent classifier with LLM-resolved IntentValence. Filed #1112 (eval test for valence classification quality). Blog: "The Second Time Keywords Lied". Updated era5, era7 research docs and mental-model design spec.

**All branches landed and published.** Slot 196 marked ready for archive.

## What Was Landed This Session

### #1098 — Replace keyword intent classifier with BDI-dimension semantic classification (neocortex)

**3 commits landed:**

1. **feat:** Added `IntentValence` enum (HOSTILE, COOPERATIVE, NEUTRAL) to `AttributedState`. `classifyIntentBias` now reads the valence field directly instead of substring matching. Removed `isHostile`, `isCooperative`, `containsAny`. LLM extraction prompt updated to include valence in JSON schema. `formatBdi` includes existing entries' valence in re-inference prompts. 6-arg backward-compatible constructor defaults to NEUTRAL. Serialization handles missing valence gracefully.

2. **fix:** `decayMap` was using the 6-arg constructor, silently resetting LLM-classified valence to NEUTRAL during confidence decay. Fixed to preserve `s.valence()`.

3. **docs:** Updated era5-relational-cognition-research, era7-personality-trait-integration-research, and mental-model-design spec to reflect the IntentValence approach.

### Blog promoted to examples
- `docs/blog/2026-11-18-mdp01-the-second-time-keywords-lied.md` — the semantic resolution boundary applied to BDI intent classification

## Issues Filed This Session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1112 | Eval test for LLM valence classification quality | S | Med | Verify the LLM produces correct hostile/cooperative/neutral labels across BDI scenarios |
| #1113 | Remove all keyword-based SEC checks — require LLM for cognitive appraisal | M | Med | All 4 SEC checks (Coping, Implication, Normative, Relevance) use keyword matching. Delete them, make SessionPipeline a hard requirement. |

## Next Up — #1113: Remove all keyword-based SEC checks

**Scaffolded as active issue.** `work continue` picks it up.

**What to do:**
1. Delete `CopingCheck.java`, `ImplicationCheck.java`, `NormativeCheck.java`, `RelevanceCheck.java`
2. Delete `LlmCopingCheck.java` (redundant — `LlmAppraisalStrategy` covers all dimensions)
3. Remove `SchererAppraisalStrategy` fallback from `CognitionDefaultBeans.appraisalStrategy()` — if no `SessionPipeline`, fail explicitly
4. Update tests that directly instantiate keyword checks — mock `SessionPipeline` instead
5. `LlmAppraisalStrategy` is already the active path and produces all 7 SEC dimensions in one call

**Files to touch:**
- `cognition/src/main/java/.../appraisal/CopingCheck.java` — delete
- `cognition/src/main/java/.../appraisal/ImplicationCheck.java` — delete
- `cognition/src/main/java/.../appraisal/NormativeCheck.java` — delete
- `cognition/src/main/java/.../appraisal/RelevanceCheck.java` — delete
- `cognition/src/main/java/.../appraisal/LlmCopingCheck.java` — delete
- `cognition/src/main/java/.../appraisal/SchererAppraisalStrategy.java` — delete or gut
- `cognition/src/main/java/.../CognitionDefaultBeans.java` — remove fallback, require SessionPipeline
- Tests: `SecCheckTest`, `SchererAppraisalStrategyTest`, `CopingCheckTest`, `SecDiscriminationTest`, `LlmBoundaryTest`, `CopingMicroLoopTest`, `CopingEmergenceEvalTest`

## Queue — 4 items remaining

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| **#1113** | Remove all keyword-based SEC checks | M | Med | **Next** — scaffolded as active |
| #1112 | Eval test for LLM valence classification quality | S | Med | After #1113 |
| #1097 | Personality-modulated CPTs for TheoryOfMindEngine | M | Med | Design-heavy — Bayesian priors |
| #1089 | Unbounded cache growth | XS | Low | Deferred |

## Keyword Classifier Audit — All Four SEC Checks Affected

Audit confirmed all four SEC checks in `SchererAppraisalStrategy` use substring matching on narrative text — `CopingCheck` (51 keywords), `ImplicationCheck` (20), `NormativeCheck` (14), `RelevanceCheck` (drive name matching). Every one has false positives ("can"→candy, "won"→wonder, "right"→copyright). `LlmAppraisalStrategy` already replaces the entire pipeline. Filed as #1113: delete all keyword checks, make `SessionPipeline` a hard requirement. You cannot run cognitive neocortex without an LLM.

## Slot Status

**Slot 196 marked ready for archive.** All canonical repos published. Cannot be archived from within the slot.

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s slot-settings.xml
```
