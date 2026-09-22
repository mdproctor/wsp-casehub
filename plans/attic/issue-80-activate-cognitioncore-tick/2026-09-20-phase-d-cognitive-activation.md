# Phase D — Cognitive Activation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #80 — Activate CognitionCore.tick()
**Issue group:** #80

**Goal:** Wire all 9 CognitionCore orchestrators and call `tick()` in the
game loop, with progressive eval evidence showing each subsystem's
contribution.

**Architecture:** Replace null orchestrators with real instances using
in-memory stores and default configs, following CognitionStack's proven
construction pattern. Add `tick()` at game cycle start with a
`SubjectResolver` that returns characters in the same room. Build eval
infrastructure as a permanent JUnit test suite under `-Pcognitive-eval`
with structural assertions and delta-based experiment output.

**Tech Stack:** Java 26, Quarkus, JUnit 5, AssertJ, blocks-core
CognitionCore, InMemoryCbrCaseMemoryStore, AgentProvider (LLM calls)

## Global Constraints

- Pre-release demo — single consumer (wacky-manor)
- All configs use `.defaults()` factory methods — no custom tuning until
  eval evidence suggests it
- In-memory stores only — no persistence, no schema work (D5)
- Use `agentProvider` for LLM-calling orchestrators (D6)
- Explicit DriveSource construction pattern — not CDI constructor (D11)
- Delta-only eval output — changes per tick per character, not full
  snapshots (D3)
- InnerLifeOrchestrator is stored in CognitionCore but NOT called by
  `tick()` — it has its own activation model. Construct it, but don't
  expect tick-driven output from it.
- Build with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`

---

## Batch 1: Cognitive Wiring — all orchestrators active, tick called

After this batch: CognitionCore has all 9 orchestrators wired (no nulls),
`tick()` is called every game cycle, and structural tests verify the wiring.

### Task 1: In-Memory Store Implementations

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryNarrativeStore.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryUserProfileStore.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryMentalModelStore.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemoryStrategyStore.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/InMemoryStoreTest.java`

**Interfaces:**
- Consumes: `NarrativeStore`, `UserProfileStore`, `MentalModelStore`,
  `StrategyStore` interfaces from blocks-core
- Produces: 4 in-memory implementations consumed by Task 2's orchestrator
  construction

- [ ] **Step 1: Write failing tests for all stores**

Read each store interface via `ide_read_file` (qualifiedNames:
`io.casehub.blocks.agentic.social.narrative.NarrativeStore`,
`io.casehub.blocks.agentic.social.UserProfileStore`,
`io.casehub.blocks.agentic.social.MentalModelStore`,
`io.casehub.blocks.agentic.social.StrategyStore`).

Write `InMemoryStoreTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.narrative.*;
import io.casehub.blocks.agentic.social.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class InMemoryStoreTest {

    @Test void narrativeStoreRoundTrip() {
        var store = new InMemoryNarrativeStore();
        var state = new NarrativeState("agent-1", "tenant-1",
            NarrativeScope.INDIVIDUAL, List.of(), Instant.now(), 0);
        store.store(state);
        assertThat(store.load("agent-1", "tenant-1")).isNotNull();
        assertThat(store.load("agent-1", "tenant-1").scopeId()).isEqualTo("agent-1");
        assertThat(store.load("other", "tenant-1")).isNull();
    }

    @Test void userProfileStoreRoundTrip() {
        var store = new InMemoryUserProfileStore();
        var profile = new UserProfile("agent-1", "subject-1", "tenant-1",
            0.5, "neutral", null, null, null, null);
        store.store(profile);
        assertThat(store.lookup("agent-1", "subject-1", "tenant-1")).isPresent();
        assertThat(store.findByAgent("agent-1", "tenant-1")).hasSize(1);
        assertThat(store.lookup("agent-1", "other", "tenant-1")).isEmpty();
    }

    @Test void mentalModelStoreRoundTrip() {
        var store = new InMemoryMentalModelStore();
        var snapshot = new MentalModelSnapshot("agent-1", "subject-1", "tenant-1",
            List.of(), List.of(), List.of(), Instant.now());
        store.store(snapshot);
        assertThat(store.lookup("agent-1", "subject-1", "tenant-1")).isPresent();
        assertThat(store.findByAgent("agent-1", "tenant-1")).hasSize(1);
        assertThat(store.lookup("agent-1", "other", "tenant-1")).isEmpty();
    }

    @Test void strategyStoreRoundTrip() {
        var store = new InMemoryStrategyStore();
        var profile = new StrategyProfile("agent-1", "tenant-1",
            java.util.Map.of(), List.of(), List.of());
        store.store(profile);
        assertThat(store.lookup("agent-1", "tenant-1")).isPresent();
        assertThat(store.lookup("other", "tenant-1")).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=InMemoryStoreTest -s slot-settings.xml`
Expected: compilation failure — store classes don't exist yet.

- [ ] **Step 3: Implement all four stores**

Each store is a package-private `ConcurrentHashMap`-backed class following
the pattern in `CognitionStack.java` (lines 528-615).

`InMemoryNarrativeStore.java`:
```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.narrative.NarrativeState;
import io.casehub.blocks.agentic.social.narrative.NarrativeStore;
import org.jspecify.annotations.Nullable;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

final class InMemoryNarrativeStore implements NarrativeStore {
    private final Map<String, NarrativeState> states = new ConcurrentHashMap<>();

    @Override public void store(NarrativeState state) {
        states.put(state.scopeId() + ":" + state.tenantId(), state);
    }

    @Override public @Nullable NarrativeState load(String scopeId, String tenantId) {
        return states.get(scopeId + ":" + tenantId);
    }
}
```

`InMemoryUserProfileStore.java`:
```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.UserProfile;
import io.casehub.blocks.agentic.social.UserProfileStore;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

final class InMemoryUserProfileStore implements UserProfileStore {
    private final Map<String, UserProfile> profiles = new ConcurrentHashMap<>();

    @Override public void store(UserProfile profile) {
        profiles.put(key(profile.agentId(), profile.subjectId(), profile.tenantId()), profile);
    }
    @Override public Optional<UserProfile> lookup(String agentId, String subjectId, String tenantId) {
        return Optional.ofNullable(profiles.get(key(agentId, subjectId, tenantId)));
    }
    @Override public List<UserProfile> findByAgent(String agentId, String tenantId) {
        return profiles.values().stream()
            .filter(p -> p.agentId().equals(agentId) && p.tenantId().equals(tenantId)).toList();
    }
    @Override public void eraseSubject(String subjectId, String tenantId) {
        profiles.entrySet().removeIf(e ->
            e.getValue().subjectId().equals(subjectId) && e.getValue().tenantId().equals(tenantId));
    }
    private static String key(String a, String s, String t) { return a + ":" + s + ":" + t; }
}
```

`InMemoryMentalModelStore.java`:
```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.MentalModelSnapshot;
import io.casehub.blocks.agentic.social.MentalModelStore;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

final class InMemoryMentalModelStore implements MentalModelStore {
    private final Map<String, MentalModelSnapshot> snapshots = new ConcurrentHashMap<>();

    @Override public void store(MentalModelSnapshot snapshot) {
        snapshots.put(key(snapshot.agentId(), snapshot.subjectId(), snapshot.tenantId()), snapshot);
    }
    @Override public Optional<MentalModelSnapshot> lookup(String agentId, String subjectId, String tenantId) {
        return Optional.ofNullable(snapshots.get(key(agentId, subjectId, tenantId)));
    }
    @Override public List<MentalModelSnapshot> findByAgent(String agentId, String tenantId) {
        return snapshots.values().stream()
            .filter(s -> s.agentId().equals(agentId) && s.tenantId().equals(tenantId)).toList();
    }
    @Override public void eraseSubject(String subjectId, String tenantId) {
        snapshots.entrySet().removeIf(e ->
            e.getValue().subjectId().equals(subjectId) && e.getValue().tenantId().equals(tenantId));
    }
    private static String key(String a, String s, String t) { return a + ":" + s + ":" + t; }
}
```

`InMemoryStrategyStore.java`:
```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.StrategyProfile;
import io.casehub.blocks.agentic.social.StrategyStore;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

final class InMemoryStrategyStore implements StrategyStore {
    private final Map<String, StrategyProfile> profiles = new ConcurrentHashMap<>();

    @Override public void store(StrategyProfile profile) {
        profiles.put(profile.agentId() + ":" + profile.tenantId(), profile);
    }
    @Override public Optional<StrategyProfile> lookup(String agentId, String tenantId) {
        return Optional.ofNullable(profiles.get(agentId + ":" + tenantId));
    }
    @Override public List<String> subjectInsights(String agentId, String subjectId, String tenantId) {
        return List.of();
    }
    @Override public void eraseAgent(String agentId, String tenantId) {
        profiles.remove(agentId + ":" + tenantId);
    }
    @Override public void eraseSubject(String subjectId, String tenantId) {}
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=InMemoryStoreTest -s slot-settings.xml`
Expected: 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/InMemory*.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/InMemoryStoreTest.java
git commit -m "feat(#80): in-memory store implementations for cognitive orchestrators"
```

---

### Task 2: Wire All Orchestrators + Tick Call + Structural Test

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
  - Lines 131-140: replace orchestrator construction
  - Lines 184-185: update `runAutonomousTicks` call to pass `cognitionCore`
  - Lines 211-236: add `cognitionCore` param, SubjectResolver, tick loop
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/CognitiveActivationTest.java`

**Interfaces:**
- Consumes: 4 in-memory stores from Task 1; `MoodOrchestrator(MoodConfig)`,
  `NarrativeOrchestrator(NarrativeStore)`, `UserModelOrchestrator(UserProfileStore, AgentProvider, UserModelConfig)`,
  `MentalModelOrchestrator(MentalModelStore, AgentProvider, MentalModelConfig)`,
  `StrategyLearningOrchestrator(StrategyStore, CbrCaseMemoryStore, ReflectionOrchestrator, AgentProvider, StrategyLearningConfig)`,
  `MemoryHygieneOrchestrator(...)`, `CuriosityDrive`, `CompetenceDrive`, `AffiliationDrive`, `AutonomyDrive`,
  `DriveOrchestrator(DriveSource×4, MoodOrchestrator, DriveComposer, DriveConfig)`,
  `InnerLifeOrchestrator(ReflectionOrchestrator, AgentProvider, List, InnerLifeConfig, DriveOrchestrator)`,
  `CognitionCore.tick(agentId, tenantId, descriptor, SubjectResolver)`
- Produces: Fully-wired `CognitionCore` with `tick()` called per game cycle;
  `CognitiveActivationTest` verifying section presence per config level

- [ ] **Step 1: Write structural test**

`CognitiveActivationTest.java` — plain JUnit, no Quarkus CDI:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.*;
import io.casehub.blocks.agentic.social.drive.*;
import io.casehub.blocks.agentic.social.goal.*;
import io.casehub.blocks.agentic.social.narrative.*;
import io.casehub.blocks.agentic.social.prompt.*;
import io.casehub.blocks.memory.*;
import io.casehub.neocortex.memory.cbr.ScopeDecay;
import io.casehub.neocortex.memory.cbr.TemporalDecay;
import io.casehub.neocortex.memory.cbr.inmem.InMemoryCbrCaseMemoryStore;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.MethodSource;
import java.time.Duration;
import java.util.*;
import java.util.stream.Stream;
import static org.assertj.core.api.Assertions.assertThat;

class CognitiveActivationTest {

    static CognitionCore buildCore(CognitionConfig config) {
        // Phase 1 — Foundation
        var mood = new MoodOrchestrator(MoodConfig.defaults());
        var narrativeOrch = new NarrativeOrchestrator(new InMemoryNarrativeStore());
        var cbrStore = new InMemoryCbrCaseMemoryStore();
        var memoryHygiene = new MemoryHygieneOrchestrator(
            cbrStore,
            new CompositeConfidenceScorer(List.of(
                new WeightedScorer(new ArousalScorer(), 0.5),
                new WeightedScorer(new SurpriseScorer(), 0.5))),
            new TemporalDecay.HalfLife(Duration.ofDays(365)),
            new ScopeDecay.Step(1.0), null,
            StrategyLearningConfig.defaults().memoryDomain(),
            List.of(StrategyLearningConfig.defaults().engagementCaseType()),
            RetentionConfig.DEFAULT, 10, 0.7, event -> {});

        // Phase 2 — Independent
        ReflectionOrchestrator noOpReflection = (a, t, s, m) -> List.of();
        var userModel = new UserModelOrchestrator(
            new InMemoryUserProfileStore(), null, UserModelConfig.defaults());
        var mentalModel = new MentalModelOrchestrator(
            new InMemoryMentalModelStore(), null, MentalModelConfig.defaults());
        var strategy = new StrategyLearningOrchestrator(
            new InMemoryStrategyStore(), cbrStore, noOpReflection,
            null, StrategyLearningConfig.defaults());

        // Phase 3 — Explicit DriveSource pattern (D11)
        var drives = new DriveOrchestrator(
            new CuriosityDrive(memoryHygiene), new CompetenceDrive(strategy),
            new AffiliationDrive(userModel, 0.3, Duration.ofHours(1)),
            new AutonomyDrive(mentalModel, 0.5),
            mood, new DriveComposer(), DriveConfig.defaults());

        // Phase 4 — InnerLife
        var innerLife = new InnerLifeOrchestrator(
            noOpReflection, null, List.of(), InnerLifeConfig.defaults(), drives);

        // Phase 5 — Goals
        var goals = new GoalProposalOrchestrator(
            drives, List.of(), null, Optional.empty(),
            null, null, null,
            GoalProposalConfig.defaults(), GoalEscalationConfig.defaults(),
            java.time.Clock.systemUTC());

        return new CognitionCore(mood, drives, userModel, mentalModel, strategy,
            narrativeOrch, goals, memoryHygiene, innerLife, null, config, null, null);
    }

    @Test void tickRunsWithoutErrorOnFullConfig() {
        var core = buildCore(CognitionConfig.all());
        // null descriptor — mood, narrative, strategy, userModel, mentalModel tick;
        // drives and goals skip (need descriptor)
        core.tick("test-agent", "test-tenant", null, (a, t) -> Set.of("other"));
    }

    @Test void allSectionsPresentWithFullConfig() {
        var core = buildCore(CognitionConfig.all());
        core.tick("test-agent", "test-tenant", null, (a, t) -> Set.of("other"));
        var sections = core.promptSections();
        assertThat(sections).anyMatch(s -> s instanceof MoodPromptSection);
        assertThat(sections).anyMatch(s -> s instanceof DrivePromptSection);
        assertThat(sections).anyMatch(s -> s instanceof NarrativePromptSection);
        assertThat(sections).anyMatch(s -> s instanceof UserModelPromptSection);
        assertThat(sections).anyMatch(s -> s instanceof MentalModelPromptSection);
        assertThat(sections).anyMatch(s -> s instanceof StrategyPromptSection);
        assertThat(sections).anyMatch(s -> s instanceof GoalPromptSection);
    }

    @Test void sectionAbsentWhenSubsystemDisabled() {
        var config = CognitionConfig.none().with("goals", true);
        var core = buildCore(config);
        core.tick("test-agent", "test-tenant", null, (a, t) -> Set.of());
        var sections = core.promptSections();
        assertThat(sections).anyMatch(s -> s instanceof GoalPromptSection);
        assertThat(sections).noneMatch(s -> s instanceof MoodPromptSection);
        assertThat(sections).noneMatch(s -> s instanceof NarrativePromptSection);
    }

    record ConfigLevel(String label, CognitionConfig config, int minSections) {}

    static Stream<ConfigLevel> configProgression() {
        var base = CognitionConfig.none()
            .with("goals", true).with("characterDrives", true).with("needsPyramid", true);
        return Stream.of(
            new ConfigLevel("baseline", base, 1),
            new ConfigLevel("+mood", base.with("mood", true), 2),
            new ConfigLevel("+narrative", base.with("mood", true).with("narrative", true), 3),
            new ConfigLevel("+models", base.with("mood", true).with("narrative", true)
                .with("userModel", true).with("mentalModel", true), 5),
            new ConfigLevel("+strategy", base.with("mood", true).with("narrative", true)
                .with("userModel", true).with("mentalModel", true).with("strategy", true), 6),
            new ConfigLevel("+drives", CognitionConfig.all().without("memoryHygiene", "innerLife"), 7),
            new ConfigLevel("full", CognitionConfig.all(), 7)
        );
    }

    @ParameterizedTest(name = "{0}")
    @MethodSource("configProgression")
    void configLevelProducesSections(ConfigLevel level) {
        var core = buildCore(level.config());
        core.tick("test-agent", "test-tenant", null, (a, t) -> Set.of("other"));
        var sections = core.promptSections();
        assertThat(sections).hasSizeGreaterThanOrEqualTo(level.minSections());
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CognitiveActivationTest -s slot-settings.xml`
Expected: compilation failure or test failure depending on constructor
signatures. Adjust imports and constructor calls as needed.

- [ ] **Step 3: Fix compilation issues and verify tests pass**

The test constructs CognitionCore identically to how ScenarioOrchestrator
will. If tests fail due to constructor mismatches, read the actual
constructor via `ide_read_file` and fix.

Run: same command. Expected: all tests PASS.

- [ ] **Step 4: Update ScenarioOrchestrator — orchestrator construction**

Replace the existing construction block (lines ~131-140) with the full
wiring. The code mirrors `buildCore()` from the test but uses
`agentProvider` instead of null for LLM-calling orchestrators:

```java
// Phase 1 — Foundation (no dependencies)
var moodOrch = new MoodOrchestrator(MoodConfig.defaults());
var narrativeOrch = new NarrativeOrchestrator(new InMemoryNarrativeStore());
var cbrStore = new InMemoryCbrCaseMemoryStore();
var memoryHygiene = new MemoryHygieneOrchestrator(
    cbrStore,
    new CompositeConfidenceScorer(java.util.List.of(
        new WeightedScorer(new ArousalScorer(), 0.5),
        new WeightedScorer(new SurpriseScorer(), 0.5))),
    new TemporalDecay.HalfLife(Duration.ofDays(365)),
    new ScopeDecay.Step(1.0), null,
    StrategyLearningConfig.defaults().memoryDomain(),
    java.util.List.of(StrategyLearningConfig.defaults().engagementCaseType()),
    RetentionConfig.DEFAULT, 10, 0.7, event -> {});

// Phase 2 — Independent, need AgentProvider for LLM calls (D6)
io.casehub.neocortex.memory.reflection.ReflectionOrchestrator noOpReflection =
    (agentId, tenantId, since, maxEntries) -> java.util.List.of();
var userModelOrch = new UserModelOrchestrator(
    new InMemoryUserProfileStore(), agentProvider, UserModelConfig.defaults());
var mentalModelOrch = new MentalModelOrchestrator(
    new InMemoryMentalModelStore(), agentProvider, MentalModelConfig.defaults());
var strategyOrch = new StrategyLearningOrchestrator(
    new InMemoryStrategyStore(), cbrStore, noOpReflection,
    agentProvider, StrategyLearningConfig.defaults());

// Phase 3 — Explicit DriveSource pattern (D11)
var curiosityDrive = new CuriosityDrive(memoryHygiene);
var competenceDrive = new CompetenceDrive(strategyOrch);
var affiliationDrive = new AffiliationDrive(userModelOrch, 0.3, Duration.ofHours(1));
var autonomyDrive = new AutonomyDrive(mentalModelOrch, 0.5);
var driveOrch = new DriveOrchestrator(curiosityDrive, competenceDrive,
    affiliationDrive, autonomyDrive, moodOrch, new DriveComposer(), DriveConfig.defaults());

// Phase 4 — InnerLife (depends on Phase 3)
var innerLifeOrch = new InnerLifeOrchestrator(
    noOpReflection, agentProvider, java.util.List.of(),
    InnerLifeConfig.defaults(), driveOrch);

// Phase 5 — Goals (update existing: pass driveOrch instead of null)
var goalOrchestrator = new GoalProposalOrchestrator(
    driveOrch, java.util.List.of(), null, java.util.Optional.empty(),
    null, null, null,
    GoalProposalConfig.defaults(), GoalEscalationConfig.defaults(),
    java.time.Clock.systemUTC());

// Assemble CognitionCore — all orchestrators wired, all subsystems enabled
var cognitionCore = new CognitionCore(
    moodOrch, driveOrch, userModelOrch, mentalModelOrch, strategyOrch,
    narrativeOrch, goalOrchestrator, memoryHygiene, innerLifeOrch,
    agentProvider, CognitionConfig.all(), mmStore, new ManorNeedTierMappingProvider());
```

- [ ] **Step 5: Update ScenarioOrchestrator — tick call site**

Add `CognitionCore cognitionCore` parameter to `runAutonomousTicks`:

```java
private void runAutonomousTicks(WorldState world, java.util.Set<String> activeSet,
                                 ActionResolver actionResolver, ManorEventDispatcher dispatcher,
                                 AgentInvocationService invocationService, NarratorAgent narratorAgent,
                                 java.util.Map<String, CharacterCognition> cognitions,
                                 ManorPlanEvaluator planEvaluator,
                                 CognitionCore cognitionCore) {
```

Update call site (~line 185) to pass `cognitionCore`.

Add SubjectResolver and tick loop at the start of `runAutonomousTicks`,
before the while loop define the resolver, then inside the loop after
pause handling and before `actingThisTick` filtering:

```java
// SubjectResolver (D9) — characters in the same room
SubjectResolver subjectResolver = (agentId, tenantId) -> {
    var character = world.character(agentId);
    if (character == null) return java.util.Set.of();
    return world.charactersInRoom(character.currentRoom()).stream()
        .map(io.casehub.examples.manor.model.CharacterState::agentId)
        .filter(id -> !id.equals(agentId))
        .collect(java.util.stream.Collectors.toSet());
};

// ... inside while loop, after pause handling, before actingThisTick:

// Cognitive tick — all active agents at cycle start (D4: batch consistency)
for (var c : activeAgents) {
    if (!c.isActive()) continue;
    var desc = agentRegistry.findById(c.agentId(), ManorConstants.TENANCY_ID).orElse(null);
    cognitionCore.tick(c.agentId(), ManorConstants.TENANCY_ID, desc, subjectResolver);
}
```

- [ ] **Step 6: Run structural test — verify still passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CognitiveActivationTest -s slot-settings.xml`

- [ ] **Step 7: Run full test suite — verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`

- [ ] **Step 8: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/CognitiveActivationTest.java
git commit -m "feat(#80): wire all cognitive orchestrators and add tick() to game loop"
```

---

## Batch 2: Progressive Evaluation — empirical evidence per subsystem

After this batch: `mvn test -pl wacky-manor -Pcognitive-eval` produces
delta streams and cross-level diff reports showing each subsystem's
contribution.

### Task 3: Eval Infrastructure + Progressive Eval Test

**Files:**
- Modify: `wacky-manor/pom.xml` — add `cognitive-eval` profile
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaCapture.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveEvalTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaCaptureTest.java`

**Interfaces:**
- Consumes: `CognitionCore.promptSections()`, `AgentRegistry`,
  `AgentProvider`, `WorldState`
- Produces: `target/cognitive-eval/levels/<level>/deltas.json`,
  `target/cognitive-eval/diffs/level-N-to-M-<label>.md`,
  `target/cognitive-eval/summary.md`

- [ ] **Step 1: Write failing test for DeltaCapture**

`CognitiveDeltaCaptureTest.java`:
```java
package io.casehub.examples.manor.experiment;

import org.junit.jupiter.api.Test;
import java.util.*;
import static org.assertj.core.api.Assertions.assertThat;

class CognitiveDeltaCaptureTest {

    @Test void noDeltaWhenStateUnchanged() {
        var capture = new CognitiveDeltaCapture();
        var state = Map.of("Mood", "neutral", "Goals", "explore");
        capture.record(1, "agent-1", state);
        capture.record(2, "agent-1", state);
        var deltas = capture.deltas();
        assertThat(deltas.stream().filter(d -> d.tick() == 2)).isEmpty();
    }

    @Test void deltaEmittedWhenSectionChanges() {
        var capture = new CognitiveDeltaCapture();
        capture.record(1, "agent-1", Map.of("Mood", "neutral"));
        capture.record(2, "agent-1", Map.of("Mood", "anxious"));
        var deltas = capture.deltas();
        assertThat(deltas.stream().filter(d -> d.tick() == 2)).hasSize(1);
        var delta = deltas.stream().filter(d -> d.tick() == 2).findFirst().orElseThrow();
        assertThat(delta.changedSections()).containsKey("Mood");
    }

    @Test void deltaEmittedWhenSectionAdded() {
        var capture = new CognitiveDeltaCapture();
        capture.record(1, "agent-1", Map.of("Mood", "neutral"));
        capture.record(2, "agent-1", Map.of("Mood", "neutral", "Narrative", "emerging arc"));
        var deltas = capture.deltas();
        var delta = deltas.stream().filter(d -> d.tick() == 2).findFirst().orElseThrow();
        assertThat(delta.addedSections()).containsKey("Narrative");
    }
}
```

- [ ] **Step 2: Implement DeltaCapture**

`CognitiveDeltaCapture.java`:
```java
package io.casehub.examples.manor.experiment;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class CognitiveDeltaCapture {
    private final Map<String, Map<String, String>> previousState = new ConcurrentHashMap<>();
    private final List<DeltaRecord> deltaList = new ArrayList<>();

    public record DeltaRecord(int tick, String agentId,
        Map<String, String> addedSections, Map<String, String> changedSections,
        List<String> removedSections) {}

    public void record(int tick, String agentId, Map<String, String> currentSections) {
        var prev = previousState.get(agentId);
        if (prev != null) {
            var added = new LinkedHashMap<String, String>();
            var changed = new LinkedHashMap<String, String>();
            var removed = new ArrayList<String>();
            for (var e : currentSections.entrySet()) {
                if (!prev.containsKey(e.getKey())) added.put(e.getKey(), e.getValue());
                else if (!prev.get(e.getKey()).equals(e.getValue())) changed.put(e.getKey(), e.getValue());
            }
            for (var key : prev.keySet()) {
                if (!currentSections.containsKey(key)) removed.add(key);
            }
            if (!added.isEmpty() || !changed.isEmpty() || !removed.isEmpty()) {
                deltaList.add(new DeltaRecord(tick, agentId, added, changed, removed));
            }
        } else if (!currentSections.isEmpty()) {
            deltaList.add(new DeltaRecord(tick, agentId, new LinkedHashMap<>(currentSections),
                Map.of(), List.of()));
        }
        previousState.put(agentId, new LinkedHashMap<>(currentSections));
    }

    public List<DeltaRecord> deltas() { return List.copyOf(deltaList); }
    public void clear() { previousState.clear(); deltaList.clear(); }
}
```

- [ ] **Step 3: Run DeltaCapture test — verify passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CognitiveDeltaCaptureTest -s slot-settings.xml`

- [ ] **Step 4: Add `cognitive-eval` Maven profile to pom.xml**

Add after the existing `llm-eval` profile:
```xml
<profile>
    <id>cognitive-eval</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <groups>cognitive-eval</groups>
                    <excludedGroups combine.self="override"/>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile>
```

- [ ] **Step 5: Write CognitiveEvalTest**

`CognitiveEvalTest.java` — `@QuarkusTest` + `@Tag("cognitive-eval")`,
runs a 2-character scenario at each config level:

```java
package io.casehub.examples.manor.experiment;

import io.casehub.blocks.agentic.social.CognitionConfig;
import io.casehub.examples.manor.agent.CognitiveActivationTest;
import io.casehub.blocks.speech.PromptSection;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;
import java.io.IOException;
import java.nio.file.*;
import java.util.*;
import java.util.stream.Stream;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("cognitive-eval")
class CognitiveEvalTest {

    record ConfigLevel(String label, CognitionConfig config) {}

    static List<ConfigLevel> levels() {
        var base = CognitionConfig.none()
            .with("goals", true).with("characterDrives", true).with("needsPyramid", true);
        return List.of(
            new ConfigLevel("0-baseline", base),
            new ConfigLevel("1-mood", base.with("mood", true)),
            new ConfigLevel("2-narrative", base.with("mood", true).with("narrative", true)),
            new ConfigLevel("3-models", base.with("mood", true).with("narrative", true)
                .with("userModel", true).with("mentalModel", true)),
            new ConfigLevel("4-strategy", base.with("mood", true).with("narrative", true)
                .with("userModel", true).with("mentalModel", true).with("strategy", true)),
            new ConfigLevel("5-drives", CognitionConfig.all().without("memoryHygiene", "innerLife")),
            new ConfigLevel("6-full", CognitionConfig.all())
        );
    }

    @Test void progressiveEvalWithDeltaCapture() throws IOException {
        var outputBase = Path.of("target/cognitive-eval");
        Files.createDirectories(outputBase.resolve("levels"));
        Files.createDirectories(outputBase.resolve("diffs"));
        var mapper = new ObjectMapper();
        Map<String, List<CognitiveDeltaCapture.DeltaRecord>> allDeltas = new LinkedHashMap<>();

        for (var level : levels()) {
            var core = CognitiveActivationTest.buildCore(level.config());
            var capture = new CognitiveDeltaCapture();

            for (int tick = 1; tick <= 10; tick++) {
                core.tick("hooded-claw", "wacky-manor", null,
                    (a, t) -> Set.of("penelope-pitstop"));
                core.tick("penelope-pitstop", "wacky-manor", null,
                    (a, t) -> Set.of("hooded-claw"));

                capture.record(tick, "hooded-claw", renderSections(core, "hooded-claw"));
                capture.record(tick, "penelope-pitstop", renderSections(core, "penelope-pitstop"));
            }

            var deltas = capture.deltas();
            allDeltas.put(level.label(), deltas);
            var levelDir = outputBase.resolve("levels").resolve(level.label());
            Files.createDirectories(levelDir);
            mapper.writerWithDefaultPrettyPrinter()
                .writeValue(levelDir.resolve("deltas.json").toFile(), deltas);
        }

        // Structural assertion: each level produces >= sections of previous
        var levelList = levels();
        for (int i = 1; i < levelList.size(); i++) {
            var prev = allDeltas.get(levelList.get(i - 1).label());
            var curr = allDeltas.get(levelList.get(i).label());
            assertThat(curr.size()).isGreaterThanOrEqualTo(prev.size());
        }

        // Cross-level diff reports
        for (int i = 1; i < levelList.size(); i++) {
            var prev = levelList.get(i - 1);
            var curr = levelList.get(i);
            var report = generateDiffReport(prev, curr, allDeltas);
            Files.writeString(outputBase.resolve("diffs")
                .resolve("level-%s-to-%s.md".formatted(prev.label(), curr.label())), report);
        }

        // Summary
        var summary = generateSummary(allDeltas);
        Files.writeString(outputBase.resolve("summary.md"), summary);
    }

    private Map<String, String> renderSections(
            io.casehub.blocks.agentic.social.CognitionCore core, String agentId) {
        var result = new LinkedHashMap<String, String>();
        for (var section : core.promptSections()) {
            result.put(section.getClass().getSimpleName(),
                section.render(agentId, "wacky-manor"));
        }
        return result;
    }

    private String generateDiffReport(ConfigLevel prev, ConfigLevel curr,
            Map<String, List<CognitiveDeltaCapture.DeltaRecord>> allDeltas) {
        var sb = new StringBuilder();
        sb.append("## %s → %s\n\n".formatted(prev.label(), curr.label()));
        var prevDeltas = allDeltas.get(prev.label());
        var currDeltas = allDeltas.get(curr.label());
        var newSectionTypes = new TreeSet<String>();
        for (var d : currDeltas) {
            newSectionTypes.addAll(d.addedSections().keySet());
        }
        for (var d : prevDeltas) {
            newSectionTypes.removeAll(d.addedSections().keySet());
        }
        if (!newSectionTypes.isEmpty()) {
            sb.append("**New section types:** ").append(String.join(", ", newSectionTypes)).append("\n\n");
        }
        sb.append("**Delta count:** %d (prev: %d)\n".formatted(currDeltas.size(), prevDeltas.size()));
        return sb.toString();
    }

    private String generateSummary(
            Map<String, List<CognitiveDeltaCapture.DeltaRecord>> allDeltas) {
        var sb = new StringBuilder("# Cognitive Eval Summary\n\n");
        sb.append("| Level | Delta Count | Agents With Changes |\n");
        sb.append("|-------|------------|--------------------|\n");
        for (var entry : allDeltas.entrySet()) {
            var agents = entry.getValue().stream()
                .map(CognitiveDeltaCapture.DeltaRecord::agentId).distinct().count();
            sb.append("| %s | %d | %d |\n".formatted(entry.getKey(), entry.getValue().size(), agents));
        }
        return sb.toString();
    }
}
```

- [ ] **Step 6: Run eval**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pcognitive-eval -s slot-settings.xml`

Verify:
- Structural assertions pass
- `target/cognitive-eval/levels/` has 7 subdirectories with `deltas.json`
- `target/cognitive-eval/diffs/` has 6 cross-level reports
- `target/cognitive-eval/summary.md` exists

- [ ] **Step 7: Commit**

```bash
git add wacky-manor/pom.xml
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaCapture.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveDeltaCaptureTest.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveEvalTest.java
git commit -m "feat(#80): cognitive eval test suite with progressive config levels"
```

---

## References

- [2026-09-20-phase-d-cognitive-activation-design.md] — design spec (D1-D11)
- [CognitionCore.java] — tick() phases, constructor (9 orchestrators), promptSections()
- [ScenarioOrchestrator.java:137-140] — current null-based constructor call
- [CognitionStack.java:100-200] — reference wiring pattern (in-memory stores, construction order)
- [SubjectResolver.java] — `@FunctionalInterface` for per-subject tick resolution (D9)
- [CuriosityDrive.java, CompetenceDrive.java, AffiliationDrive.java, AutonomyDrive.java] — explicit DriveSource implementations (D11)
- [MoodConfig, DriveConfig, UserModelConfig, MentalModelConfig, StrategyLearningConfig, InnerLifeConfig] — all have `.defaults()` factory methods
- [InMemoryCbrCaseMemoryStore] — no-arg constructor from neocortex dependency
- [CognitiveInfluenceEvalTest.java] — existing eval pattern (`@QuarkusTest` + `@Tag`)
- GitHub #80 — Activate CognitionCore.tick()
