# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-17b:** Completed #921 and #1031 on neocortex. #921 commits on main (CalibrationReport + PredictionCalibrationAnalyzer + tests). #1031 on branch `issue-1031-extract-shared-eval-infrastructure` (eval infrastructure extraction). Both need work-end close ceremony.

**Prior session carry-over:** #1018 and #1019 implemented on branch `issue-1019-caps-topology-coping-connections` (4 commits). #999 on branch `issue-999-cognitive-defaults-decomposition`. Neither landed — branches need close ceremony.

**What was built this session:**
- #921: `PredictionCalibrationAnalyzer` — static utility comparing predictor vs LLM regulation effectiveness per agent. `CalibrationReport` record with per-source summaries, delta, miscalibration flag, inconclusive status. 23 tests (10 unit + 13 integration).
- #1031: Extracted `extractJson`, `writeResult`, `judge` from `CopingEmergenceEvalTest` and `SleepDerivationEvalTest` into `EvalTestSupport`. Eliminated three copies of `extractJson`, two copies of `writeResult`, two retry/backoff judge loops. Net -50 lines.
- Filed #1045: Surface calibration report in `@McpDomain("cognition")` GraphQL resolvers (XS/Low)
- Updated research doc: `docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md` — added §6 on calibration analysis
- Blog: "Knowing When the Model Is Wrong" — prediction calibration, population-level vs individual variation

## Queue (13 items remaining)

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1031 | Extract shared eval infrastructure | XS | Low | DONE on branch |
| #712 | Shadow archetypes — personality degradation | L | High | New capability |
| #980 | ConsolidationScheduler journal observability | S | Low | Observability |
| #968 | Instrument remaining CognitionCore.tick() stages | M | Low | Observability |
| #1041 | CAPS coping → regulation topology bridge | S | Low | Closes coping-regulation gap |
| #1042 | Verify inert coping nodes are predictor-only | XS | Low | Documentation |
| #1045 | Surface calibration in observability resolvers | XS | Low | Depends on #921 |
| #974 | Fix eval test compilation | XS | Low | Build fix |
| #975 | Document CAPS modules in contributor guide | S | Low | Documentation |
| #1043 | Oscillation attractor strategic differentiation | S | Med | CAPS enhancement |
| #1044 | CognitiveEmergenceSampleTest distrust attractor | S | Med | Test fix |
| #976 | DriveProfileDerivationPhase personality classification | M | High | New capability |
| #977 | PID-5 facet extension | M | High | New capability |

## What's Done

**Calibration (#921):**
- CalibrationReport + PredictionCalibrationAnalyzer on neocortex main
- Integration tests proving the full flow: write episodes → load summaries → analyze

**Coping epic (#595):**
- Batches 1–8: all complete (see prior handoff for details)
- Filed #1036, #1037, #1038, #1041, #1042

**Regulation epic (#821):**
- Batches 1–4 complete, Batch 5 remaining items not in this slot

## Slot 211 Boundary

Slot 211 owns #1023 (Drive annotation provenance) and #827 (Cognitive curiosity epic + children). Do not duplicate. Issues #974, #975, #1043, #1044, #976, #977 are safe for either slot — not in 211's queue.

## Key References

- Research doc: `neocortex/docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md` — updated with §6 on calibration
- Blog: `wsp-casehub/blog/2026-11-17-mdp03-knowing-when-the-model-is-wrong.md` — calibration analysis
- Blog: `wsp-casehub/blog/2026-11-17-mdp01-the-record-that-knew-too-much.md` — decomposition + dispatch
- Blog: `wsp-casehub/blog/2026-11-17-mdp02-wiring-the-coping-layer.md` — CAPS coping wiring
- Design spec: `wsp-casehub-neocortex/specs/issue-595-coping-strategies/2026-11-15-coping-strategies-design.md`

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
mvn install -pl cognition-api,cognition,cognitive-index -am -Dmaven.test.skip=true
```
