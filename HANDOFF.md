# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-17c:** Designed and started implementing #712 (Shadow archetypes — situational personality modulation). Design spec through 3 adversarial review rounds (rev 4). Batch 1 of implementation plan complete (3 tasks, 44 tests). Code review clean. Blog written. Research doc updated.

**Branch:** `issue-712-shadow-archetypes-personality-degradation` on both neocortex and examples repos.

## What Was Built This Session

### Design (complete)
- **Design spec** (rev 4, 850 lines): `wsp-casehub/specs/issue-712-shadow-archetypes-personality-degradation/2026-11-17-situational-personality-modulation-design.md`
- **5 architectural decisions** in `decisions.md`
- **Implementation plan** (11 tasks, 5 batches): `wsp-casehub/plans/2026-11-17-situational-personality-modulation.md`
- **Core model:** Elastic band — one personality state vector, situational attractors pull facets, displacement relaxes back toward home, prolonged exposure drifts home permanently. CDE is oblivious.
- **Detection model:** Gain-form evidence: `E = X·(1+β·N)·(1+β·R)`. Structural guarantee: X=0 ⇒ E=0. Five modes (stress, flow, 3 social) in 2 exclusive groups.

### Implementation — Batch 1 (complete, 44 tests green)
All in `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/`:

1. **Config loader + validator** (19 tests): `SituationalMode` enum, `SignalTransform` enum, `SituationalPersonalityConfig`, `SituationalPersonalityConfigLoader`, `situational-personality.yaml` with all 5 modes. Load-time validation: weight sums, topology node existence, transform constraints, sub-floor hold margin.

2. **Integrator** (13 tests): `PersonalityState`, `FacetState`, `PersonalityStateIntegrator` — advance() on wall-clock hourly grid, pull() for attractor displacement. Exact exponential relaxation, exposure weighting, fatigue/healing/drift at hour closes, cadence-independent.

3. **Detector** (12 tests): `SituationalDetector`, `SituationalEvidence`, `DetectionResult` — gain-form evidence model, engagement hysteresis, group-exclusive winner selection with switch margin, social gate.

### Issues Filed (by design review)
- **#1055** — EffectiveCognitiveDefaults (resting/effective split) — PREREQUISITE for Batch 4
- **eidos#234** — Situational profiles in Eidos (AmpdFacetName, SituationalProfileResolver) — PREREQUISITE for Batch 3
- **agency#484** — CognitivePipelineBuilder single assembly path — PREREQUISITE for wiring
- **#1056–#1059** — Deferred #712 scope items (biographical extraction, temporal drift, narrative frame, CARMA integration)
- **#1060–#1065** — Pre-existing defects (PEO never ticked, agentId-keyed caches, duplicated CAPS settlements, scheduler starvation, bounded search, no hierarchy roles)

### Blog
- "The Elastic Band: Personality That Bends Under Pressure" — `wsp-casehub/blog/2026-11-17-mdp01-the-elastic-band.md`

### Research Doc
- Updated `neocortex/docs/research/era7-personality-trait-integration-research.md` — added Era 7b section on situational personality modulation

## What's Next

**Batch 2 (no prerequisites):** Task 4 — Appraisal fixes (Part 0c). LlmAppraisalStrategy SEC dimensions, AppraisalTickParticipant accessors, RelationalBiasing clamp.

**Batch 3 (blocked on eidos#234):** Tasks 5-6 — SituationalStateService + MindMap persistence, profile resolution.

**Batch 4 (blocked on #1055):** Tasks 7-8 — SituationalPersonalityParticipant TERMINAL wiring, SituationalDeformationPhase.

**Batch 5:** Tasks 9-11 — Observability, E2E integration test, noise-floor topology test.

**Recommendation:** Next session should either (a) continue Batch 2 on this branch, or (b) start the prerequisites (#1055 or eidos#234) on a separate slot if parallel progress is desired.

## Queue (12 items remaining after #712)

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #712 | Shadow archetypes — personality degradation | L | High | **IN PROGRESS — Batch 1 complete** |
| #980 | ConsolidationScheduler journal observability | S | Low | Observability |
| #968 | Instrument remaining CognitionCore.tick() stages | M | Low | Observability |
| #1041 | CAPS coping → regulation topology bridge | S | Low | |
| #1042 | Verify inert coping nodes are predictor-only | XS | Low | |
| #1045 | Surface calibration in observability resolvers | XS | Low | |
| #974 | Fix eval test compilation | XS | Low | |
| #975 | Document CAPS modules in contributor guide | S | Low | |
| #1043 | Oscillation attractor strategic differentiation | S | Med | |
| #1044 | CognitiveEmergenceSampleTest distrust attractor | S | Med | |
| #976 | DriveProfileDerivationPhase personality classification | M | High | |
| #977 | PID-5 facet extension | M | High | |

## Key References

- Design spec: `wsp-casehub/specs/issue-712-shadow-archetypes-personality-degradation/2026-11-17-situational-personality-modulation-design.md`
- Implementation plan: `wsp-casehub/plans/2026-11-17-situational-personality-modulation.md`
- Decisions: `wsp-casehub/specs/issue-712-shadow-archetypes-personality-degradation/decisions.md`
- Research doc: `neocortex/docs/research/era7-personality-trait-integration-research.md` (§Era 7b)
- Blog: `wsp-casehub/blog/2026-11-17-mdp01-the-elastic-band.md`
- Design review: `/Users/mdproctor/reviews/casehub-slots/issue-712-situational-personality-20261117-063006/`

## Slot 211 Boundary

Slot 211 owns #1023 (Drive annotation provenance) and #827 (Cognitive curiosity epic). Issues #974, #975, #1043, #1044, #976, #977 are safe for either slot.

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="SituationalPersonalityConfigTest,PersonalityStateIntegratorTest,SituationalDetectorTest" -s examples/.mvn/slot-settings.xml
```
