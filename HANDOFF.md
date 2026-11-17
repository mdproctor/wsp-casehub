# HANDOFF — casehub-examples (slot 196)

## Current State

#1017 landed on canonical neocortex main (3 commits). Slot marked ready for archive.

**#999 NOT landed** — `issue-999-cognitive-defaults-decomposition` branch exists in slot neocortex but was not carried to canonical. The close ceremony only handled #1017 (the queue's `covers` value). Next session must land #999 separately: rebase onto canonical main (no file overlap with #1017), fast-forward merge, stamp branch. The branch has a squash review marker.

**Blocker for #1018:** #1038 (pre-existing compile errors: `EmotionType.INTEREST`, `PipelineTrace` constructor) must be resolved before any cognition module work. Likely needs cross-slot sync.

## Queue (7 items remaining after #999 and #1017)

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
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
- Batch 6 (DONE): #999 — CognitiveDefaults structural decomposition (23→8 fields, 4 nested config groups)
- Batch 7 (DONE): #1017 — DriveModulation + AttentionDirective dispatch wiring
- Filed #1036: defensive validation for GoalOperation and AttentionDirective free-text fields
- Filed #1037: strategiesAttempted IndexOutOfBounds guard in dispatchCopingEffects
- Filed #1038: pre-existing cognition module compile errors on main

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
- Blog: `wsp-casehub/blog/2026-11-17-mdp01-the-record-that-knew-too-much.md` — decomposition + dispatch wiring

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
mvn install -pl cognition-api,cognition,cognitive-index -am -Dmaven.test.skip=true
```
