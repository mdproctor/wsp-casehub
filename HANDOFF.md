# HANDOFF — casehub-examples (slot 196)

## Current State

Branch `issue-1006-clamp-drive-modulation-delta` in neocortex — 3 commits landing #1006, #1007, #1002. Ready for `work end`. #1011 was already implemented (closed, no code change needed).

## Queue (9 items remaining)

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #999 | CognitiveDefaults structural decomposition | M | Med | Refactor — nested config groups |
| #1017 | DriveModulation + AttentionDirective dispatch | S | Low | Dispatch wiring |
| #1018 | Seed CAPS coping attractor weights from CopingDefaults | S | Med | CAPS init |
| #1019 | CAPS topology connections to coping outputs | M | Med | CAPS topology |
| #921 | Prediction calibration analysis | S | Med | Eval — predictor vs LLM |
| #1031 | Extract shared eval infrastructure | XS | Low | Refactor — EvalTestSupport |
| #712 | Shadow archetypes — personality degradation | L | High | New capability |
| #980 | ConsolidationScheduler journal observability | S | Low | Observability |
| #968 | Instrument remaining CognitionCore.tick() stages | M | Low | Observability |

## What's Done

**Coping epic (#595):**
- Batch 1 (DONE): #991, #992, #993 — types, predictor, LLM generation
- Batch 2 (DONE): #994, #996 — micro-loop orchestrator, goal dispatch, keyword expansion
- Batch 3 (DONE): #995 — regulation refactoring
- Batch 4 (DONE): #997 — CAPS coping attractors + pipeline wiring
- Batch 5 (DONE): #998 — coping emergence eval (5 tests, floor 4/5)
- Code review fixes (DONE): #1011 (already implemented), #1006 (intensityDelta clamping), #1007 (regulation-routed prompt guidance), #1002 (strategy family priority ordering)
- Filed #1036: defensive validation for GoalOperation and AttentionDirective free-text fields

**Regulation epic (#821):**
- Batches 1–4 (DONE): all individual strategies, fatigue, automatic, proactive, monitoring, per-strategy + per-modality evals, CAPS integration, observability
- Batch 5 remaining: examples#203 (Traitors format), mosaic#315 (dashboard) — NOT in this slot's queue
- Independent items in queue above

## Slot 211 Boundary

Slot 211 owns #1023 (Drive annotation provenance) and #827 (Cognitive curiosity epic + children). Do not duplicate.

## Key References

- Design spec: `wsp-casehub-neocortex/specs/issue-595-coping-strategies/2026-11-15-coping-strategies-design.md`
- Era7 research: `neocortex/docs/research/era7-personality-trait-integration-research.md`
- Blog: `examples/docs/blog/2026-11-16-mdp01-the-loop-that-learns-to-chain.md`
- Blog: `examples/docs/blog/2026-11-16-mdp02-every-field-is-a-boundary.md`

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
mvn install -pl cognition-api,cognition,cognitive-index -am -Dmaven.test.skip=true
```
