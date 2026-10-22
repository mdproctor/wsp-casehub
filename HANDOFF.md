# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `issue-116-manor-cognitive-pipeline-builder` (examples)
**Issue:** casehubio/examples#116 — Extract CognitivePipelineBuilder for composable, testable pipeline assembly
**Status:** All tasks complete. Ready for work end.

## START HERE — What's Next

Close #116 via `work end`. All implementation is done:

1. **ManorCognitivePipelineBuilder** — composable builder with per-seed-step toggles, now extended with `experienceService`, `cognitiveProfile`, and `situationClassifier` parameters
2. **ManorCognitivePipeline** — result type with structure queries
3. **PlaybookOrchestrator refactored** — 105 lines of inline pipeline assembly replaced by builder call (net -59 lines)
4. **9 tests pass** — 6 structure verification tests + 3 dead-end regression tests

### Commits on this branch

1. `bddb5ff` — ManorCognitivePipeline + ManorCognitivePipelineBuilder + 6 structure tests
2. `f36b443` — fix compilation for neocortex#653 constructor change
3. `da3e1fd` — refactor PlaybookOrchestrator to use builder (-105/+46 lines)
4. `a3eae92` — 3 dead-end regression tests (drives→DriveAdaptationPhase, formation memories→ExperienceRecorder, seedCapsState→CapsEngine)

### Dead-end regression tests

| Boundary | Test | Assertion |
|----------|------|-----------|
| Drives → DriveAdaptationPhase | `seeded_drives_have_properties_drive_adaptation_phase_expects` | Drive nodes have `cognitiveKind=drive-intensity`, `drive-type`, `intensity`, `agent-id` in "cognitive" subgraph |
| Formation memories → BehavioralPromptSection | `seeded_formation_memories_reach_experience_recorder` | ExperienceRecorder receives FormativeExperience events for the character |
| seedCapsState in production | `seeded_caps_state_exists_after_build` | CapsEngine.loadState returns non-null AgentCapsState with correct agent/tenant |

## What was completed this session

### PlaybookOrchestrator refactoring

Extended ManorCognitivePipelineBuilder with 3 new fields (`cognitiveProfile`, `experienceService`, `situationClassifier`) so it produces fully-configured CharacterCognition instances matching the orchestrator's original construction.

Replaced ~105 lines of inline cognitive pipeline assembly in PlaybookOrchestrator.runScenario() with a 14-line builder call. Capability tag setting (world-state concern) and consolidation cache management stay in PlaybookOrchestrator. seedPeople uses full socialConfigs (including inactive characters) to preserve the complete social knowledge graph.

### Dead-end regression tests

Added 3 tests verifying the boundaries fixed in #115 are connected through the builder path. All 9 ManorCognitivePipelineTest tests pass.

## Still Open — Verified Incomplete

**neocortex#656** — CDI path: orchestrators get null SessionPipeline. CDI-created beans have null pipeline as final field. Needs either @Produces SessionPipeline or factory re-creates orchestrators.

## Branch State

- `issue-116-manor-cognitive-pipeline-builder` (examples) — active, 4 commits
- `issue-653-cognitive-pipeline-builder` (neocortex) — stamped closed, landed as e0d8a3ea
- `issue-641-degree-accumulator-config` (neocortex) — stamped closed, landed as 307dc2ca

## Key Files

- `examples/wacky-manor/.../ManorCognitivePipeline.java` — result type with structure queries
- `examples/wacky-manor/.../ManorCognitivePipelineBuilder.java` — composable builder (3 new fields added)
- `examples/wacky-manor/.../ManorCognitivePipelineTest.java` — 9 tests (6 structure + 3 regression)
- `examples/wacky-manor/.../PlaybookOrchestrator.java` — refactored to use builder

## Important: neocortex JARs

The neocortex cognition and caps-engine modules were installed locally. If compilation fails in a new session, re-run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml install -Dmaven.test.skip=true -pl cognition,caps-engine -am -q
```
