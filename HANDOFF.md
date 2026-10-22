# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `issue-116-manor-cognitive-pipeline-builder` (examples)
**Issue:** casehubio/examples#116 — Extract CognitivePipelineBuilder for composable, testable pipeline assembly
**Status:** Core infrastructure delivered, refactoring and dead-end regression tests remain

## START HERE — What's Next

Continue examples#116. The builder and result type are working (6 tests pass, 628 fast suite green). Two tasks remain:

### 1. Refactor PlaybookOrchestrator to use ManorCognitivePipelineBuilder

PlaybookOrchestrator.runScenario() lines 169-317 duplicate the orchestrator creation and seeding that the builder now encapsulates. Replace the inline assembly with:

```java
var pipeline = ManorCognitivePipelineBuilder
    .create(mmStore, ManorConstants.TENANCY_ID, cbrRecordStore)
    .sessionPipeline(sessionPipeline)
    .config(config)
    .capsEngine(capsEngine)
    .experienceRecorder(experienceRecorder)
    .agentRegistry(agentRegistry)
    .socialConfigs(socialConfigs)
    .build();
var cognitionCore = pipeline.cognitionCore();
var cognitions = pipeline.cognitions();
```

The builder already handles all seeding steps. Key difference: PlaybookOrchestrator also creates services (observation, experience, reflection, goals, plans) that the builder doesn't handle — those stay in PlaybookOrchestrator.

### 2. Dead-end regression tests from #115

The 4 fixed dead ends from #115 need regression tests in ManorCognitivePipelineTest:
- Boundary 1: Drives → DriveAdaptationPhase (verify drive nodes found)
- Boundary 2: Formation memories → BehavioralPromptSection (verify behavioralEnabled reaches prompt)
- Boundary 3: seedCapsState called in production (verify CAPS state exists after seeding)
- Boundary 4: seedPeople called in production (verify social subgraph exists) — DONE (test already exists)

Use the same InMemoryMindMapStore + ManorSocialConfigLoader pattern as the existing tests.

## What was completed this session

### Issue triage and closure — 7 issues closed

**neocortex#661** — monotonic consolidation trigger. Verified complete: SignificanceAccumulator is fully monotonic, DegreeAccumulator only in caps-engine for CAPS classification.

**neocortex#653** — CognitivePipelineBuilder. Cherry-picked 6 commits from `issue-650-elastic-session-scheduler` branch, squashed into `e0d8a3ea`, merged to main. Constructor-inject SessionPipeline in 11 orchestrators + CognitionCore. 424 cognition tests pass.

**neocortex#641** — DegreeAccumulator externalization. Cherry-picked 1 commit, merged as `307dc2ca`. DegreeAccumulatorConfig record, dual-rate accumulator (fast CUSUM + slow integration), externalized to application.properties. 62 caps-engine tests pass.

**neocortex#444** — CognitionCore self-wiring. Verified functionally complete — CognitionCoreFactory and CognitivePipelineBuilder handle all configure*() calls internally.

**examples#133, #136** — agentProvider.invoke migration. Verified all 7 production callers migrated to SessionPipeline.

**neocortex#662, #663, #664** — previously closed, confirmed.

### examples#116 — ManorCognitivePipeline (in progress)

Created 3 new files on branch `issue-116-manor-cognitive-pipeline-builder`:
- `ManorCognitivePipeline.java` — result type with structure queries (hasDriveNodes, hasBehavioralNodes, hasSocialSubgraph, hasCognitiveSubgraph, driveNodeCount, seededDriveNames)
- `ManorCognitivePipelineBuilder.java` — composable builder with per-seed-step toggles
- `ManorCognitivePipelineTest.java` — 6 structure verification tests

Also fixed 3 existing test files for neocortex#653 constructor change (added SessionPipeline param).

### Neocortex commits pushed to all remotes

Both #653 and #641 pushed to origin (fork), upstream (casehubio), and canonical (local).

## Still Open — Verified Incomplete

**examples#116** — broader pipeline builder vision. Core done, refactoring + dead-end tests remain.
**neocortex#656** — CDI path: orchestrators get null SessionPipeline. CDI-created beans have null pipeline as final field. Needs either @Produces SessionPipeline or factory re-creates orchestrators.

## Branch State

- `issue-116-manor-cognitive-pipeline-builder` (examples) — active, 2 commits
- `issue-653-cognitive-pipeline-builder` (neocortex) — stamped closed, landed as e0d8a3ea
- `issue-641-degree-accumulator-config` (neocortex) — stamped closed, landed as 307dc2ca

## Key Files

- `examples/wacky-manor/.../ManorCognitivePipeline.java` — result type with structure queries
- `examples/wacky-manor/.../ManorCognitivePipelineBuilder.java` — composable builder
- `examples/wacky-manor/.../ManorCognitivePipelineTest.java` — 6 structure tests
- `examples/wacky-manor/.../PlaybookOrchestrator.java` — lines 169-317 to refactor
- `neocortex/cognition/.../CognitivePipelineBuilder.java` — neocortex-level CognitionCore builder
- `neocortex/caps-engine/.../DegreeAccumulator.java` — dual-rate with config record

## Important: neocortex JARs

The neocortex cognition and caps-engine modules were installed locally (`mvn install -pl cognition,caps-engine -am`) to pick up #653 and #641 changes. If compilation fails in a new session, re-run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml install -Dmaven.test.skip=true -pl cognition,caps-engine -am -q
```
