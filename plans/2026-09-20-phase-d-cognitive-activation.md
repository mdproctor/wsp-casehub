# Phase D — Cognitive Activation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #80 — Activate CognitionCore.tick()
**Issue group:** #80, #76

**Goal:** Give wacky-manor characters a full inner life by activating all
CognitionCore orchestrators, adding tick() to the game loop, building a
progressive cognitive eval test suite, and extracting briefings into
directive-minimal form.

**Architecture:** Wire all 8 orchestrators with in-memory store
implementations into ScenarioOrchestrator. Add tick() call at game cycle
start. Build eval infrastructure as a permanent JUnit test suite under
-Pcognitive-eval with structural assertions and delta-based experiment
output. Extract 17 character briefings into directive + SocialConfig
separation.

**Tech Stack:** Java 21+, Quarkus, JUnit 5, blocks-core CognitionCore,
InMemoryMindMapStore, AgentProvider (LLM calls)

## Global Constraints

- Pre-release demo — single consumer (wacky-manor)
- In-memory stores only — no persistence, no schema work
- CognitionConfig flags control progressive activation
- Delta-only eval output — changes per tick per character, not snapshots
- No remnants in directive after seeding extraction

---

## Batch 1: Orchestrator Wiring + Tick Call Site

### Task 1: In-Memory Store Implementations

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryNarrativeStore.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryUserProfileStore.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryMentalModelStore.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryStrategyStore.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/InMemoryStoreTest.java`

**Interfaces:**
- Consumes: `NarrativeStore`, `UserProfileStore`, `MentalModelStore`, `StrategyStore` interfaces from blocks-core
- Produces: In-memory implementations used by Task 2

- [ ] **Step 1:** Read each store interface from blocks-core (use `ide_find_class` + `ide_read_file` for each). Identify required methods.
- [ ] **Step 2:** Write a test for each store — basic put/get/list operations.
- [ ] **Step 3:** Run tests — verify they fail (classes don't exist yet).
- [ ] **Step 4:** Implement each store as a `ConcurrentHashMap`-backed class. Minimal — just enough to satisfy the interface contracts.
- [ ] **Step 5:** Run tests — verify they pass.
- [ ] **Step 6:** Commit: `feat(#80): in-memory store implementations for cognitive orchestrators`

### Task 2: Wire All Orchestrators into ScenarioOrchestrator

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:130-145`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/CognitionCoreWiringTest.java`

**Interfaces:**
- Consumes: In-memory stores from Task 1, existing `agentProvider`, `MoodConfig`, `DriveConfig` from blocks-core
- Produces: Fully-wired `CognitionCore` instance with all orchestrators non-null

Construction order (respects dependency chain):

```java
// Phase 1 — no dependencies
var moodOrchestrator = new MoodOrchestrator(MoodConfig.defaults());
var narrativeOrchestrator = new NarrativeOrchestrator(new InMemoryNarrativeStore());

// Phase 2 — independent, need AgentProvider
var userModelOrchestrator = new UserModelOrchestrator(
    new InMemoryUserProfileStore(), agentProvider, UserModelConfig.defaults());
var mentalModelOrchestrator = new MentalModelOrchestrator(
    new InMemoryMentalModelStore(), agentProvider, MentalModelConfig.defaults());
var strategyOrchestrator = new StrategyLearningOrchestrator(
    new InMemoryStrategyStore(), null, null, agentProvider,
    StrategyLearningConfig.defaults());

// Phase 3 — depends on Phase 1 + 2
var driveOrchestrator = new DriveOrchestrator(
    moodOrchestrator, strategyOrchestrator, userModelOrchestrator,
    mentalModelOrchestrator, null, DriveConfig.defaults(), narrativeOrchestrator);

// Phase 4 — existing
// goalOrchestrator already wired
```

- [ ] **Step 1:** Write test verifying CognitionCore accepts all non-null orchestrators and config with all flags enabled.
- [ ] **Step 2:** Run test — verify it fails.
- [ ] **Step 3:** Read each orchestrator's constructor from blocks-core (verify exact param order and types). Update ScenarioOrchestrator to construct all orchestrators.
- [ ] **Step 4:** Update `CognitionConfig` to enable all flags: `CognitionConfig.all()`.
- [ ] **Step 5:** Run test — verify it passes. Run full test suite — verify no regressions.
- [ ] **Step 6:** Commit: `feat(#80): wire all cognitive orchestrators into ScenarioOrchestrator`

### Task 3: Add tick() Call Site

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — `runAutonomousTicks` method
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSubjectResolver.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSubjectResolverTest.java`

**Interfaces:**
- Consumes: `CognitionCore.tick(agentId, tenantId, descriptor, resolver)`, world state for `SubjectResolver`
- Produces: Per-tick cognitive updates for all active characters

`SubjectResolver` is a `@FunctionalInterface` returning `Set<String>` of nearby agent IDs — wacky-manor's world state already knows who's in each room.

- [ ] **Step 1:** Write `ManorSubjectResolver` — implements `SubjectResolver`, takes world state, returns agents in the same room as the given agent.
- [ ] **Step 2:** Write test for `ManorSubjectResolver` — two characters in same room → both appear. Character alone → empty set.
- [ ] **Step 3:** Run test — verify pass.
- [ ] **Step 4:** Add tick loop at start of `runAutonomousTicks` cycle, before character action selection:
```java
for (String agentId : activeAgents) {
    var descriptor = descriptors.get(agentId);
    if (descriptor != null) {
        cognitionCore.tick(agentId, ManorConstants.TENANCY_ID, descriptor, subjectResolver);
    }
}
```
- [ ] **Step 5:** Run full test suite — verify no regressions.
- [ ] **Step 6:** Commit: `feat(#80): add CognitionCore.tick() to game tick cycle`

---

## Batch 2: Cognitive Eval Test Suite

### Task 4: Eval Infrastructure — Maven Profile + Delta Capture

**Files:**
- Modify: `wacky-manor/pom.xml` — add `-Pcognitive-eval` profile
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveEvalScenario.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaCapture.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaRecord.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaCaptureTest.java`

**Interfaces:**
- Consumes: `CharacterCognition.renderCognitiveSections()`, `CognitionCore.promptSections()`, world state
- Produces: `CognitiveDeltaCapture` — tracks previous state per character, emits deltas only when cognitive sections change

`CognitiveDeltaRecord`:
```java
public record CognitiveDeltaRecord(
    int tick, String agentId, String type,
    Map<String, String> addedSections,
    Map<String, String> changedSections,
    List<String> removedSections,
    List<String> worldEvents) {}
```

- [ ] **Step 1:** Write test for `CognitiveDeltaCapture` — submit two identical states → no delta. Submit changed state → delta with only the changed section.
- [ ] **Step 2:** Run test — verify fail.
- [ ] **Step 3:** Implement `CognitiveDeltaCapture` — stores previous `Map<String, String>` (header → content) per agent. Compares on each tick. Emits only additions, changes, and removals.
- [ ] **Step 4:** Run test — verify pass.
- [ ] **Step 5:** Add Maven profile `-Pcognitive-eval` to pom.xml — includes test classes in `experiment/` package, excludes from standard runs.
- [ ] **Step 6:** Commit: `feat(#80): cognitive eval delta capture infrastructure`

### Task 5: Eval Scenario + Structural Assertions

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveEvalTest.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EvalConfigLevel.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CrossLevelDiffReport.java`

**Interfaces:**
- Consumes: `CognitiveEvalScenario`, `CognitiveDeltaCapture`, `CognitionConfig`
- Produces: Pass/fail structural assertions + `target/cognitive-eval/` output

`EvalConfigLevel` — enum of progressive config levels:
```java
public enum EvalConfigLevel {
    BASELINE(CognitionConfig.none().with("goals", true).with("characterDrives", true).with("needsPyramid", true)),
    MOOD(BASELINE.config().with("mood", true)),
    NARRATIVE(MOOD.config().with("narrative", true)),
    MODELS(NARRATIVE.config().with("userModel", true).with("mentalModel", true)),
    STRATEGY(MODELS.config().with("strategy", true)),
    DRIVES(STRATEGY.config().with("drives", true)),
    FULL(DRIVES.config().with("memoryHygiene", true).with("innerLife", true));
    // ...
}
```

- [ ] **Step 1:** Write `EvalConfigLevel` enum with progressive config levels.
- [ ] **Step 2:** Write structural assertion tests:
```java
@Test void moodSectionRenderedWhenEnabled()
@Test void narrativeSectionAbsentWhenDisabled()
@Test void deltaOutputNonEmptyWhenSubsystemActivated()
@Test void configLevelNProducesSupersetOfLevelNMinus1Sections()
```
- [ ] **Step 3:** Write `CognitiveEvalTest` — runs the dedicated scenario (2-3 characters: Hooded Claw + Penelope, fixed 10-tick sequence) at each config level. Captures deltas. Persists to `target/cognitive-eval/levels/<level>/deltas.json`.
- [ ] **Step 4:** Write `CrossLevelDiffReport` — compares consecutive level outputs. Produces `target/cognitive-eval/diffs/level-N-to-M-<label>.md` showing new sections, changed sections, and observed behavioural shifts.
- [ ] **Step 5:** Write `target/cognitive-eval/summary.md` — one-page overview: which subsystems produced observable changes.
- [ ] **Step 6:** Run with `mvn test -Pcognitive-eval`. Verify structural assertions pass, output directories populated.
- [ ] **Step 7:** Commit: `feat(#80): cognitive eval test suite with progressive config levels`

---

## Batch 3: Seeding Extraction

### Task 6: Extract and Split Character Briefings

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml` — all 17 characters
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` — add migrated behavioural data
- Test: existing `CharacterCognitionTest` + `DescriptorLoadTest` must still pass

**Interfaces:**
- Consumes: Existing briefing text per character
- Produces: Minimal directives (voice/role/constraints only) + enriched SocialConfig

For each of the 17 characters:

- [ ] **Step 1:** Read the character's briefing in `descriptors-composite.yaml`.
- [ ] **Step 2:** Classify each sentence: voice/identity → keep in descriptor; hard constraint → keep as AgentConstraint; behavioural instruction → check if covered by existing SocialConfig (drive, norm, belief, goal). If not covered, add to `social-config.yaml`.
- [ ] **Step 3:** Write the minimal directive — name + role + voice + hard constraints only.
- [ ] **Step 4:** Remove the old briefing text entirely. No commented-out remnants.
- [ ] **Step 5:** Run `DescriptorLoadTest` and `CharacterCognitionTest` — verify all characters still load and render cognitive sections.
- [ ] **Step 6:** Commit per character group (villains, heroes, sidekicks): `feat(#76): directive-minimal seeding — <group>`

### Task 7: Post-Seeding Verification

**Files:**
- No new files — re-run existing tests and eval suite

- [ ] **Step 1:** Run full test suite: `mvn test -pl wacky-manor -o`. Verify no regressions.
- [ ] **Step 2:** Run cognitive eval: `mvn test -Pcognitive-eval`. Compare output against pre-extraction baseline. Verify no section loss.
- [ ] **Step 3:** Spot-check 3 characters: read the rendered system prompt + observation sections. Verify no duplication between directive and observation content.
- [ ] **Step 4:** Commit: `test(#76): post-seeding verification — no behavioural regression`

---

## Batch 4: Full Scenario Validation

### Task 8: Full 17-Character Run

**Files:**
- No new files — run existing autonomous scenario with all subsystems active

- [ ] **Step 1:** Run the full autonomous scenario with `CognitionConfig.all()` — all 17 characters, all orchestrators, tick() active.
- [ ] **Step 2:** Observe for: LLM call latency per tick (acceptable?), memory growth (in-memory stores), character behaviour qualitative check.
- [ ] **Step 3:** If issues found, file follow-up issues. If clean, close #80 and #76.
- [ ] **Step 4:** Commit: `docs(#80): full scenario validation — all orchestrators active`

---

## References

- [2026-09-20-phase-d-cognitive-activation-design.md] — design spec
- [CognitionCore.java] — tick() method, constructor, promptSections()
- [ScenarioOrchestrator.java:130-145] — current constructor call
- [MoodOrchestrator.java] — constructor signature: `MoodConfig`
- [SubjectResolver.java] — functional interface for nearby agents
- [decisions.md] — D1-D8a
- [audit report §9g] — CognitionCore.tick() never called
- GitHub #80, #76
