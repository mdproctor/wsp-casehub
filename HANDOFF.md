# HANDOFF — casehub-examples (slot 196)

## Last Session

Implemented and validated neocortex#932 — smoke scenario integration test. Landed as `2fa64e87` on examples main.

### What Was Built

`SmokeScenarioEvalTest` — a 25-tick, 3-agent integration test running the full cognitive pipeline end-to-end: appraisal, regulation, drives, MindMap evolution, CAPS situation classification. First real pipeline test that sits between unit tests (mocked) and the 300-tick full run (hours).

- **Result:** mean eval score 4.93/5.0, min 4.0/5.0, 137 events, 44 min runtime
- **Agents:** Vincent Marsh (scheming), James Hartwell (gallantry), Clara Bellingham (social-harmony)
- **Transcript:** `wacky-manor/docs/eval/smoke-scenario-2026-11-11/transcript.json`

### Discovered and Fixed

- **CDI: InformationGapDetector** — added `@Dependent` annotation in neocortex (committed to neocortex main `3b1c4b2d`). Filed examples#205.
- **CDI: entity-token-cdi** — added explicit dependency to wacky-manor pom.xml. Filed neocortex#936 (should be non-optional in cognition pom).
- **Surefire fork timeout** — increased to 3600s for llm-eval profile.
- **ManorEvent types** — only `action`, `dialogue`, `aside`, `narrator` are valid.

### Filed Issues

- neocortex#932 — smoke scenario (CLOSED, landed)
- neocortex#935 — audit all LLM interactions for session reuse (OPEN)
- neocortex#936 — entity-token-cdi should not be optional (OPEN)
- examples#205 — InformationGapDetector CDI fix (OPEN, fix committed)
- examples#206 — BackendFactoryDiscoveryTest pre-existing failure (OPEN)

### Key Finding: LLM Session Waste

`LlmSituationClassifier` creates a new session (`UUID.randomUUID()`) per classify call — ~150 cold sessions in the smoke test. Sticky sessions per agent would cut 60-70% of the 44-minute runtime. Broader audit filed as neocortex#935 covering all cognitive pipeline LLM interaction points.

## Priorities for Next Session

1. **neocortex#935** — LLM session reuse audit (biggest perf win, affects all eval tests)
2. **neocortex#936** — make entity-token-cdi non-optional (XS fix, unblocks clean downstream deps)
3. **.queue items** — predictor quality (#830), scenario validation (#829), and remaining cognitive initiative items

## Slot 211 Overlap

Slot 211 works on eval calibration (#905) and pipeline coherence (#894). The smoke test asserts behavioral outcomes, not pipeline internals — no conflict. But coordinate on neocortex main changes to `CognitionCoreFactory` and session pipeline.

## Neocortex Build Note

Full neocortex reactor build fails at `schema-generator` module. Individual module builds work (used `mindmap-intelligence` and `entity-token-cdi` successfully). May need investigation if full reactor build is needed.

## References

- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/SmokeScenarioEvalTest.java`
- Eval transcript: `wacky-manor/docs/eval/smoke-scenario-2026-11-11/transcript.json`
- Neocortex CDI fix: commit `3b1c4b2d` on neocortex main
