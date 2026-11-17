# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-17d:** Continued #712 implementation — Batch 2 (appraisal fixes), eidos#234 (situational profiles prerequisite), Batch 3 (MindMap persistence + delta conversion). Code review clean (0 CRITICAL, 5 WARNING all fixed). Blog written. Research doc updated. Two issues filed (#1068, #1069).

**Branch:** `issue-712-shadow-archetypes-personality-degradation` on neocortex and examples repos.
**Branch:** `issue-234-situational-profiles` on eidos repo (not yet merged — PR needed).

## What Was Built This Session

### Neocortex — Batch 2: Appraisal Fixes (9 tests)
- `LlmAppraisalStrategy.parseResponse`: SEC dimensions now passed through to AppraisalResult (were computed but dropped)
- `CognitionTickContext`: added `tickEventRef` field for tick stamping
- `AppraisalTickParticipant`: added `currentResultAt(agentId, tenantId, EventRef)` and `currentInputActivations(agentId, tenantId)` accessors
- `RelationalBiasing`: clamped output to [0, 1]

### Eidos — #234: Situational Profiles (26 tests, 616 total eidos tests green)
- **api**: `AmpdFacetShift` record (13 PID-5 facets, |delta| ≤ 50), `SituationalProfile` record (5 closed modes), `AgentDisposition.situationalProfiles` field + Builder, `AgentDescriptorComparator` coverage
- **eidos-core**: `DispositionDeserializer` parses `situationalProfiles` from YAML
- **vocab**: `archetype-situational-profiles.yaml` — 60 profiles (12 families × 5 modes), `SituationalProfileDefaults` loader
- **runtime**: `SituationalProfileDeriver` — runs after `ArchetypeDeriver`, resolves archetype→family, injects family defaults

### Neocortex — Batch 3: State Infrastructure (16 tests)
- `SubgraphTypes.PERSONALITY_STATE` constant in mindmap-api
- `SituationalStateService`: load/create/persist personality state per tenant+agent in MindMap, deterministic SHA-256 node naming, tenant-scoped subgraph, property serialization round-trip
- `SituationalDeltaConverter`: eidos SituationalProfile (0-100 AMPD) → neocortex SituationalMode → facet delta maps (0-1 normalized)

### Code Review Fixes
- `SituationalStateService.persist`: creates node if missing (was silently dropping state)
- `SituationalProfileDefaults`: InputStream wrapped in try-with-resources
- `SituationalProfileDeriver`: uses builder instead of fragile 12-arg constructor
- `DispositionDeserializer`: null-safety guard on shift node fields

### Issues Filed
- **#1068** — LlmAppraisalStrategy prompt clause for situational facets (XS/Low, blocked by #1055)
- **#1069** — AppraisalTickParticipant map cleanup for agent churn (XS/Low)

### Blog
- "Where Personality Lives" — `wsp-casehub/blog/2026-11-17-mdp05-where-personality-lives.md`

### Research Doc
- Updated `neocortex/docs/research/era7-personality-trait-integration-research.md` — Batches 1-3 status, eidos#234 delivered

## What's Next

**Batch 4 (blocked on #1055):** Tasks 7-8 — SituationalPersonalityParticipant TERMINAL wiring, SituationalDeformationPhase.

**Batch 5 (blocked on Batch 4):** Tasks 9-11 — Observability, E2E integration test, noise-floor topology test.

**#1055 is the critical path.** L/High — EffectiveCognitiveDefaults with tenant-scoped, provenance-preserving re-derivation. 108 production callsite migration across cognition, cognitive-index, mindmap-intelligence, cognitive-eidos. Lives in neocortex.

**eidos#234 needs a PR** — branch `issue-234-situational-profiles` on eidos. 6 commits, 12 files, 954 insertions. All installed to local Maven but not pushed/merged.

**Next session: implement #1055** — EffectiveCognitiveDefaults. The branch covers both #712 and #1055. Start by reading the four key files:
1. `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDefaultsRegistry.java` — current registry (replace)
2. `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` — `deriveAndMerge` → pure `merge`
3. `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridge.java` — delete `bridgeAll`/`registerFromDescriptor`, keep translation
4. `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java` — wire baseline via bridge `toDescriptorView`

The ~108 callsite migration (CognitiveDefaultsRegistry → EffectiveCognitiveDefaults) is mechanical — `forAgent(agentId)` → `forAgent(tenantId, agentId)`. Use IntelliJ `ide_find_references` to enumerate all call sites.

## Queue (11 items remaining after #712)

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #712 | Shadow archetypes — personality degradation | L | High | **IN PROGRESS — Batches 1-3 complete, Batch 4 blocked on #1055** |
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
- Blogs: `wsp-casehub/blog/2026-11-17-mdp01-the-elastic-band.md`, `wsp-casehub/blog/2026-11-17-mdp05-where-personality-lives.md`

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="SituationalPersonalityConfigTest,PersonalityStateIntegratorTest,SituationalDetectorTest,SituationalStateServiceTest,SituationalDeltaConverterTest,AppraisalFixesTest" -s examples/.mvn/slot-settings.xml
```

## Slot 211 Boundary

Slot 211 owns #1023 (Drive annotation provenance) and #827 (Cognitive curiosity epic). Slot 211 landed #980 (ConsolidationScheduler journal observability). Issues #974, #975, #1043, #1044, #976, #977 are safe for either slot.
