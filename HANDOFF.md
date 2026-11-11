# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `issue-932-smoke-scenario` (examples + workspace)
**Issue:** casehubio/neocortex#932 — smoke scenario, real pipeline integration test

### What's Done

- `SmokeScenarioEvalTest.java` created and compiling
  - `@QuarkusTest @Tag("llm-eval")` in `experiment/` package
  - 3 GENERIC agents: Vincent Marsh (hooded-claw), James Hartwell (peter-perfect), Clara Bellingham (penelope-pitstop)
  - Full cognitive pipeline via `ManorCognitivePipelineBuilder` — real MindMapStore, SessionPipeline, CapsEngine, appraisal + regulation
  - 25-tick loop following `PlaybookOrchestrator.runAutonomousTicks()` pattern
  - 6 scripted events at ticks 1, 5, 8, 12, 17, 22 spanning arrival → discovery → suspicion → confrontation → threat → resolution
  - Transcript saved to `docs/eval/smoke-scenario-<date>/transcript.json` in classify_emotions.py format
  - LLM judge scores sampled events (up to 15), asserts mean >= 3.5 and min >= 2.0

### What's Next

- Fast test run in progress — verifying no regressions
- Need to run the actual llm-eval test to verify end-to-end
- May need tuning: event descriptions, tick spacing, or threshold based on first run results

### Slot 211 Overlap

Moderate risk. Slot 211 works on eval calibration (#905) and pipeline coherence (#894). The smoke test asserts behavioral outcomes (drive quality scores), not pipeline internals, so it should be resilient to 211's changes. Don't couple to specific pipeline wiring or ordering.

## Previous Session

Implementation plan 10/10 complete for emotion regulation (#590/#820). A/B eval saved. All deferred items filed as issues.

## References

- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/SmokeScenarioEvalTest.java`
- Issue: casehubio/neocortex#932
- Key patterns followed: `PlaybookOrchestrator` (pipeline wiring), `DriveQualityEvalTest` (judge scoring), `TickExecutor` (parallel query)
