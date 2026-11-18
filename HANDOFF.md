# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-18b:** Completed #1113 and #1112 on branch `issue-1113-remove-keyword-sec-checks`.

**Session 2026-11-18a:** Closed #1098 — replaced keyword intent classifier with LLM-resolved IntentValence. Filed #1112 (eval test for valence classification quality). Blog: "The Second Time Keywords Lied". Updated era5, era7 research docs and mental-model design spec.

**Branch `issue-1113-remove-keyword-sec-checks` has 3 commits ready for work-end.**

## What Was Landed This Session

### #1098 — Replace keyword intent classifier with BDI-dimension semantic classification (neocortex)

**3 commits landed:**

1. **feat:** Added `IntentValence` enum (HOSTILE, COOPERATIVE, NEUTRAL) to `AttributedState`. `classifyIntentBias` now reads the valence field directly instead of substring matching. Removed `isHostile`, `isCooperative`, `containsAny`. LLM extraction prompt updated to include valence in JSON schema. `formatBdi` includes existing entries' valence in re-inference prompts. 6-arg backward-compatible constructor defaults to NEUTRAL. Serialization handles missing valence gracefully.

2. **fix:** `decayMap` was using the 6-arg constructor, silently resetting LLM-classified valence to NEUTRAL during confidence decay. Fixed to preserve `s.valence()`.

3. **docs:** Updated era5-relational-cognition-research, era7-personality-trait-integration-research, and mental-model-design spec to reflect the IntentValence approach.

### Blog promoted to examples
- `docs/blog/2026-11-18-mdp01-the-second-time-keywords-lied.md` — the semantic resolution boundary applied to BDI intent classification

## Issues Filed This Session (18a)

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1112 | Eval test for LLM valence classification quality | S | Med | **Done (18b)** — `ValenceClassificationEvalTest` with 14 scenarios + edge case |
| #1113 | Remove all keyword-based SEC checks — require LLM for cognitive appraisal | M | Med | **Done (18b)** — deleted 8 source, 1 API, 11 test files; -1828 lines |

## What Was Done This Session (18b)

### #1113 — Remove all keyword-based SEC checks
- Deleted 8 keyword/individual-LLM check classes and SchererAppraisalConfig
- `CognitionDefaultBeans.appraisalStrategy()` now throws if no SessionPipeline
- `CopingMicroLoop` takes `AppraisalStrategy` instead of `List<SecCheck>` for re-evaluation
- Deleted 11 test files, updated 3 test files
- Updated CLAUDE.md and cognitive-model-reference.md

### #1112 — Eval test for LLM valence classification quality
- `ValenceClassificationEvalTest` with 14 BDI scenarios across hostile/cooperative/neutral
- Corpus accuracy test (>= 75% threshold)
- LLM-judged edge case: benevolent manipulation (cooperative goals via hostile means)

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
