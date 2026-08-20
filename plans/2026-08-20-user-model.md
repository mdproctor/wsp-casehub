# UserModel Pattern Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #122 — UserModel pattern — per-user behavioral profile synthesis
**Issue group:** #118, #119, #120, #121, #122, #123, #124, #126

**Goal:** Build a per-subject profile synthesis orchestrator that composes
RelationshipEvent, ExperienceRecorder, TrendAnalyzer, and CbrCaseMemoryStore
into structured behavioral profiles with tiered synthesis (heuristic + LLM).

**Architecture:** `UserModelOrchestrator` follows the established `record()` +
`tick()` pattern from PersonalityEvolution/InnerLife. Signals accumulate via
`record()` (O(1)). `tick()` runs heuristic fold for core dimensions (familiarity
score, stage transitions) and optionally triggers LLM synthesis for open-ended
dimensions (communication style, topics, preferences). Profiles persist via
`UserProfileStore` SPI backed by `CbrCaseMemoryStore`.

**Tech Stack:** Java 21, JUnit 5, Mockito, CDI (`@ApplicationScoped`,
`@DefaultBean`), neocortex-memory-api, platform-agent-api (AgentProvider)

## Global Constraints

- Package: `io.casehub.blocks.agentic.social` (D22 — rename from `personality`)
- No Quarkus runtime in tests — plain JUnit 5 + Mockito
- Sealed types for outcome enums — no open inheritance
- All records validate in compact constructors
- Per-subject `ReentrantLock` for tick serialisation
- `@Nullable` annotations from org.jspecify on nullable fields

---

## Batch 1: Package rename + Foundation types

### Task 1: Rename `agentic.personality` → `agentic.social`

D22 revised: the triad (PersonalityEvolution, InnerLife, UserModel) is social
cognition, not personality. No code merged to main — rename is free.

**Files:**
- Move: all 20 files in `blocks/src/main/java/io/casehub/blocks/agentic/personality/` → `blocks/src/main/java/io/casehub/blocks/agentic/social/` (use `ide_move_file`)
- Move: all 11 files in `blocks/src/test/java/io/casehub/blocks/agentic/personality/` → `blocks/src/test/java/io/casehub/blocks/agentic/social/` (use `ide_move_file`)
- Modify: `CLAUDE.md` — update package references from `agentic/personality` to `agentic/social`

**Interfaces:**
- Consumes: nothing
- Produces: `io.casehub.blocks.agentic.social` package with all existing types relocated

- [ ] **Step 1: Create target directories**

```bash
mkdir -p blocks/src/main/java/io/casehub/blocks/agentic/social
mkdir -p blocks/src/test/java/io/casehub/blocks/agentic/social
```

- [ ] **Step 2: Move all main source files**

Use `ide_move_file` for each of the 20 main source files. IntelliJ updates
package declarations and internal imports automatically. Move each file:

```
ide_move_file(file="blocks/src/main/java/io/casehub/blocks/agentic/personality/<File>.java",
              targetDir="blocks/src/main/java/io/casehub/blocks/agentic/social/")
```

Files to move (all 20): BehavioralSignalPressureSource, CivilityCheck,
CivilityConstraint, ConsecutiveInitiationCooldownConstraint, ContentQualityGate,
EvolutionTick, GoalOutcomePressureSource, InitiationContext, InnerLifeConfig,
InnerLifeOrchestrator, InnerLifeTick, MaxPerWindowConstraint, MinimumGapConstraint,
MotivationAssessment, PersonalityEvolutionConfig, PersonalityEvolutionOrchestrator,
RelationshipPressureSource, TokenJaccardDistance, TraitActivation, TraitPressureSource.

- [ ] **Step 3: Move all test source files**

Use `ide_move_file` for each of the 11 test files:

Files: BehavioralSignalPressureSourceTest, CivilityConstraintTest,
EventTypeDispatchTest, EvolutionTickTest, GoalOutcomePressureSourceTest,
HaltFlagTest, InnerLifeOrchestratorTest, InnerLifeTickTest,
PersonalityEvolutionOrchestratorTest, RelationshipPressureSourceTest,
TokenJaccardDistanceTest.

- [ ] **Step 4: Verify — run diagnostics and tests**

```bash
mvn --batch-mode test
```

Expected: all existing tests pass with the new package.

- [ ] **Step 5: Update CLAUDE.md**

Update all references from `agentic/personality` and `agentic.personality`
to `agentic/social` and `agentic.social` using Edit tool.

- [ ] **Step 6: Commit**

```bash
git add blocks/src CLAUDE.md
git commit -m "refactor(#122): rename agentic.personality → agentic.social — social cognition triad (D22)

Refs #118, #119, #122"
```

---

### Task 2: Foundation types — InteractionSignal, UserProfile, UserModelTick, configs

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/InteractionSignal.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StageTier.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/RelationshipStageConfig.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/UserProfile.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/UserModelTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/UserModelConfig.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/InteractionSignalTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/RelationshipStageConfigTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/UserProfileTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/UserModelTickTest.java`

**Interfaces:**
- Consumes: `QualitySignal` (neocortex-memory-api), `RelationshipEvent` (neocortex-memory-api), `ExperienceEvent` (neocortex-memory-api)
- Produces:
  - `InteractionSignal` sealed interface with `description()`, `quality()` — permits `RelationshipSignal`, `ExperienceSignal`, `CustomSignal`
  - `StageTier` record: `(String name, double threshold)`
  - `RelationshipStageConfig` record: `(List<StageTier> tiers, double decayRate, double positiveWeight, double negativeWeight)` with `defaults()` and `resolveStage(double score)`
  - `UserProfile` record: core fields + nullable LLM fields + `Map<String, String> metadata`
  - `UserModelTick` sealed interface: `Unchanged(@Nullable String reason)`, `Updated(UserProfile profile)`, `Synthesised(UserProfile profile, @Nullable UserProfile previousProfile)`
  - `UserModelConfig` record with defaults

- [ ] **Step 1: Write InteractionSignal test**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.relationship.QualitySignal;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class InteractionSignalTest {

    @Test
    void customSignalCarriesDescriptionAndQuality() {
        var signal = new InteractionSignal.CustomSignal("hello", QualitySignal.POSITIVE);
        assertThat(signal.description()).isEqualTo("hello");
        assertThat(signal.quality()).isEqualTo(QualitySignal.POSITIVE);
    }

    @Test
    void relationshipSignalDelegatesToEvent() {
        var event = new io.casehub.neocortex.memory.relationship.RelationshipEvent(
            "agent-1", "user-1", "t1", null, null, "chat",
            QualitySignal.POSITIVE, "friendly exchange", null, java.util.Map.of());
        var signal = new InteractionSignal.RelationshipSignal(event);
        assertThat(signal.description()).isEqualTo("friendly exchange");
        assertThat(signal.quality()).isEqualTo(QualitySignal.POSITIVE);
    }

    @Test
    void experienceSignalUsesProvidedQuality() {
        var event = new io.casehub.neocortex.memory.experience.Observation(
            "agent-1", "t1", "case-1", "turn-1", "observed something", null, java.util.Map.of());
        var signal = new InteractionSignal.ExperienceSignal(event, QualitySignal.NEUTRAL);
        assertThat(signal.description()).isEqualTo("observed something");
        assertThat(signal.quality()).isEqualTo(QualitySignal.NEUTRAL);
    }

    @Test
    void sealedTypeExhaustiveness() {
        InteractionSignal signal = new InteractionSignal.CustomSignal("x", QualitySignal.NEUTRAL);
        var result = switch (signal) {
            case InteractionSignal.RelationshipSignal r -> "relationship";
            case InteractionSignal.ExperienceSignal e -> "experience";
            case InteractionSignal.CustomSignal c -> "custom";
        };
        assertThat(result).isEqualTo("custom");
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=InteractionSignalTest`
Expected: FAIL — InteractionSignal not found

- [ ] **Step 3: Implement InteractionSignal**

Create `InteractionSignal.java` with sealed interface and three record permits:
`RelationshipSignal(RelationshipEvent event)`, `ExperienceSignal(ExperienceEvent event, QualitySignal quality)`,
`CustomSignal(String description, QualitySignal quality)`. Each implements `description()` and `quality()`.

- [ ] **Step 4: Run test — verify it passes**

Run: `mvn --batch-mode test -pl blocks -Dtest=InteractionSignalTest`
Expected: PASS

- [ ] **Step 5: Write RelationshipStageConfig test**

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class RelationshipStageConfigTest {

    @Test
    void defaultsHaveFiveTiers() {
        var config = RelationshipStageConfig.defaults();
        assertThat(config.tiers()).hasSize(5);
        assertThat(config.tiers().get(0).name()).isEqualTo("stranger");
        assertThat(config.tiers().get(4).name()).isEqualTo("confidant");
    }

    @Test
    void resolveStageReturnsHighestMatchingTier() {
        var config = RelationshipStageConfig.defaults();
        assertThat(config.resolveStage(0.0)).isEqualTo("stranger");
        assertThat(config.resolveStage(0.19)).isEqualTo("stranger");
        assertThat(config.resolveStage(0.2)).isEqualTo("acquaintance");
        assertThat(config.resolveStage(0.5)).isEqualTo("familiar");
        assertThat(config.resolveStage(0.8)).isEqualTo("confidant");
        assertThat(config.resolveStage(1.0)).isEqualTo("confidant");
    }

    @Test
    void stageTierRejectsInvalidThreshold() {
        assertThatThrownBy(() -> new StageTier("bad", -0.1))
            .isInstanceOf(IllegalArgumentException.class);
        assertThatThrownBy(() -> new StageTier("bad", 1.1))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void emptyTiersRejected() {
        assertThatThrownBy(() -> new RelationshipStageConfig(
            java.util.List.of(), 0.01, 1.0, 0.5))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 6: Implement StageTier and RelationshipStageConfig**

Create `StageTier.java` (record with validation) and `RelationshipStageConfig.java`
(record with `defaults()` factory and `resolveStage(double score)` method that
iterates tiers and returns the name of the highest tier whose threshold ≤ score).

- [ ] **Step 7: Run test — verify it passes**

Run: `mvn --batch-mode test -pl blocks -Dtest=RelationshipStageConfigTest`
Expected: PASS

- [ ] **Step 8: Write UserProfile and UserModelTick tests**

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class UserProfileTest {

    @Test
    void constructionWithRequiredFields() {
        var now = Instant.now();
        var profile = new UserProfile("agent-1", "user-1", "t1",
            "stranger", 0.0, 0, 0, 0, 0, now, now, null,
            null, null, null, null, Map.of());
        assertThat(profile.agentId()).isEqualTo("agent-1");
        assertThat(profile.subjectId()).isEqualTo("user-1");
        assertThat(profile.communicationStyle()).isNull();
    }

    @Test
    void metadataIsImmutable() {
        var now = Instant.now();
        var mutable = new java.util.HashMap<String, String>();
        mutable.put("key", "value");
        var profile = new UserProfile("a", "s", "t", "stranger", 0.0,
            0, 0, 0, 0, now, now, null, null, null, null, null, mutable);
        mutable.put("new", "entry");
        assertThat(profile.metadata()).doesNotContainKey("new");
    }

    @Test
    void nullAgentIdRejected() {
        assertThatThrownBy(() -> new UserProfile(null, "s", "t", "stranger",
            0.0, 0, 0, 0, 0, Instant.now(), Instant.now(), null,
            null, null, null, null, Map.of()))
            .isInstanceOf(NullPointerException.class);
    }
}
```

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class UserModelTickTest {

    @Test
    void unchangedCarriesReason() {
        var tick = new UserModelTick.Unchanged("no signals");
        assertThat(tick.reason()).isEqualTo("no signals");
    }

    @Test
    void synthesisedCarriesBothProfiles() {
        var now = Instant.now();
        var prev = new UserProfile("a", "s", "t", "stranger", 0.1,
            5, 3, 1, 1, now, now, null, null, null, null, null, Map.of());
        var curr = new UserProfile("a", "s", "t", "acquaintance", 0.3,
            10, 7, 2, 1, now, now, now, "formal", "tech", null, null, Map.of());
        var tick = new UserModelTick.Synthesised(curr, prev);
        assertThat(tick.profile().relationshipStage()).isEqualTo("acquaintance");
        assertThat(tick.previousProfile().relationshipStage()).isEqualTo("stranger");
    }

    @Test
    void sealedExhaustiveness() {
        UserModelTick tick = new UserModelTick.Unchanged(null);
        var result = switch (tick) {
            case UserModelTick.Unchanged u -> "unchanged";
            case UserModelTick.Updated u -> "updated";
            case UserModelTick.Synthesised s -> "synthesised";
        };
        assertThat(result).isEqualTo("unchanged");
    }
}
```

- [ ] **Step 9: Implement UserProfile, UserModelTick, UserModelConfig**

Create `UserProfile.java` (record with compact constructor validation),
`UserModelTick.java` (sealed interface with 3 record permits),
`UserModelConfig.java` (record with defaults: `minSignalsForSynthesis=5`,
`synthesisCooldown=Duration.ofHours(1)`, `expectedTickInterval=Duration.ofHours(1)`,
`evictionTimeout=Duration.ofDays(7)`, `maxObservationsInPrompt=50`,
`memoryDomain="user-model"`, `caseType="user-profile"`,
`stageConfig=RelationshipStageConfig.defaults()`).

- [ ] **Step 10: Run all foundation tests**

Run: `mvn --batch-mode test -pl blocks -Dtest="InteractionSignalTest,RelationshipStageConfigTest,UserProfileTest,UserModelTickTest"`
Expected: all PASS

- [ ] **Step 11: Commit**

```bash
git add blocks/src
git commit -m "feat(#122): add UserModel foundation types — InteractionSignal, UserProfile, UserModelTick, configs

Refs #122"
```

---

## Batch 2: Storage + Orchestrator

### Task 3: UserProfileStore SPI + CbrUserProfileStore adapter

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/UserProfileStore.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/CbrUserProfileStore.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/UserProfileSchema.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/CbrUserProfileStoreTest.java`

**Interfaces:**
- Consumes: `UserProfile` (Task 2), `CbrCaseMemoryStore` (neocortex-memory-api), `FeatureValue`, `FeatureVectorCbrCase`, `CbrQuery`, `EraseRequest`
- Produces:
  - `UserProfileStore` interface: `void store(UserProfile)`, `Optional<UserProfile> lookup(String agentId, String subjectId, String tenantId)`, `List<UserProfile> findByAgent(String agentId, String tenantId)`, `void eraseSubject(String subjectId, String tenantId)`
  - `CbrUserProfileStore` `@DefaultBean @ApplicationScoped` implementation

- [ ] **Step 1: Write CbrUserProfileStore test — store and lookup**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.*;

import java.time.Instant;
import java.util.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class CbrUserProfileStoreTest {

    @Mock CbrCaseMemoryStore cbrStore;
    CbrUserProfileStore store;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        store = new CbrUserProfileStore(cbrStore);
    }

    @Test
    void storeConvertsProfileToCbrCase() {
        var now = Instant.now();
        var profile = new UserProfile("agent-1", "user-1", "t1",
            "acquaintance", 0.3, 10, 7, 2, 1, now, now, null,
            "formal", "tech", null, null, Map.of());

        store.store(profile);

        var captor = ArgumentCaptor.forClass(CbrCase.class);
        verify(cbrStore).store(captor.capture(), eq("agent-1"), eq("t1"),
            any(), eq("user-profile"), isNull(), isNull());
        var stored = captor.getValue();
        assertThat(stored.features().get("subject_id"))
            .isEqualTo(FeatureValue.string("user-1"));
        assertThat(stored.features().get("familiarity_score"))
            .isEqualTo(FeatureValue.number(0.3));
    }

    @Test
    void lookupReturnsEmptyWhenNoProfile() {
        when(cbrStore.retrieveSimilar(any(), eq(CbrCase.class)))
            .thenReturn(List.of());

        var result = store.lookup("agent-1", "user-1", "t1");
        assertThat(result).isEmpty();
    }

    @Test
    void eraseSubjectRemovesAllAgentProfiles() {
        var scored = mock(ScoredCbrCase.class);
        var cbrCase = mock(CbrCase.class);
        when(scored.cbrCase()).thenReturn(cbrCase);
        when(scored.caseId()).thenReturn("case-1");
        when(scored.entityId()).thenReturn("entity-1");
        when(cbrCase.features()).thenReturn(Map.of(
            "subject_id", FeatureValue.string("user-1")));
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrStore.retrieveSimilar(any(), eq(CbrCase.class)))
            .thenReturn(List.of(scored));

        store.eraseSubject("user-1", "t1");

        verify(cbrStore).erase(any(EraseRequest.class));
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=CbrUserProfileStoreTest`
Expected: FAIL — UserProfileStore, CbrUserProfileStore not found

- [ ] **Step 3: Implement UserProfileStore interface**

```java
package io.casehub.blocks.agentic.social;

import java.util.List;
import java.util.Optional;

public interface UserProfileStore {
    void store(UserProfile profile);
    Optional<UserProfile> lookup(String agentId, String subjectId, String tenantId);
    List<UserProfile> findByAgent(String agentId, String tenantId);
    void eraseSubject(String subjectId, String tenantId);
}
```

- [ ] **Step 4: Implement UserProfileSchema (package-private)**

Package-private class with static constants for feature key names and methods
to convert `UserProfile → Map<String, FeatureValue>` and
`Map<String, FeatureValue> → UserProfile`. Feature keys: `subject_id`,
`relationship_stage`, `familiarity_score`, `total_interactions`,
`positive_signals`, `negative_signals`, `neutral_signals`,
`communication_style`, `topics_of_interest`, `preferences`.

- [ ] **Step 5: Implement CbrUserProfileStore**

`@DefaultBean @ApplicationScoped` class implementing `UserProfileStore`.
Constructor injects `CbrCaseMemoryStore`. Uses `UserProfileSchema` for
conversions. `store()` builds `FeatureVectorCbrCase` and stores with
`producerAgentId=agentId`. `lookup()` queries by subject_id feature and
post-filters by agentId (GE-20260820-c19b68). `eraseSubject()` queries
all profiles for a subject across agents and erases each.

- [ ] **Step 6: Run test — verify it passes**

Run: `mvn --batch-mode test -pl blocks -Dtest=CbrUserProfileStoreTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add blocks/src
git commit -m "feat(#122): add UserProfileStore SPI + CbrUserProfileStore — profile persistence via CbrCaseMemoryStore

Refs #122"
```

---

### Task 4: UserModelOrchestrator — record() + tick() with tiered synthesis

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/UserModelOrchestrator.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/SynthesisResult.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/UserModelOrchestratorTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/FamiliarityScoreTest.java`

**Interfaces:**
- Consumes: `InteractionSignal` (Task 2), `UserProfile` (Task 2), `UserModelTick` (Task 2), `UserModelConfig` (Task 2), `RelationshipStageConfig` (Task 2), `UserProfileStore` (Task 3), `AgentProvider` (platform-agent-api)
- Produces:
  - `UserModelOrchestrator` `@ApplicationScoped`: `void record(InteractionSignal, String agentId, String subjectId, String tenantId)`, `UserModelTick tick(String agentId, String subjectId, String tenantId)`, `@Nullable UserProfile currentProfile(String agentId, String subjectId, String tenantId)`

- [ ] **Step 1: Write FamiliarityScoreTest**

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class FamiliarityScoreTest {

    private final RelationshipStageConfig config = RelationshipStageConfig.defaults();

    @Test
    void zeroInteractionsProducesZeroScore() {
        double score = UserModelOrchestrator.computeFamiliarity(0, 0, 0, config, 0);
        assertThat(score).isEqualTo(0.5);
    }

    @Test
    void allPositiveProducesHighScore() {
        double score = UserModelOrchestrator.computeFamiliarity(20, 0, 0, config, 0);
        assertThat(score).isGreaterThan(0.8);
    }

    @Test
    void allNegativeProducesLowScore() {
        double score = UserModelOrchestrator.computeFamiliarity(0, 20, 0, config, 0);
        assertThat(score).isLessThan(0.3);
    }

    @Test
    void decayReducesScoreOverTime() {
        double fresh = UserModelOrchestrator.computeFamiliarity(10, 0, 0, config, 0);
        double decayed = UserModelOrchestrator.computeFamiliarity(10, 0, 0, config, 100);
        assertThat(decayed).isLessThan(fresh);
    }

    @Test
    void negativeWeightDampensNegativeSignals() {
        double equal = UserModelOrchestrator.computeFamiliarity(5, 5, 0, config, 0);
        assertThat(equal).isGreaterThan(0.5);
    }
}
```

- [ ] **Step 2: Write UserModelOrchestratorTest — core cycle**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.relationship.QualitySignal;
import io.casehub.platform.agent.api.AgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.*;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class UserModelOrchestratorTest {

    @Mock UserProfileStore profileStore;
    @Mock AgentProvider agentProvider;
    UserModelOrchestrator orchestrator;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        orchestrator = new UserModelOrchestrator(profileStore, agentProvider,
            UserModelConfig.defaults());
    }

    @Test
    void tickWithNoSignalsReturnsUnchanged() {
        var tick = orchestrator.tick("agent-1", "user-1", "t1");
        assertThat(tick).isInstanceOf(UserModelTick.Unchanged.class);
    }

    @Test
    void recordThenTickProducesUpdated() {
        var signal = new InteractionSignal.CustomSignal("hello", QualitySignal.POSITIVE);
        orchestrator.record(signal, "agent-1", "user-1", "t1");

        var tick = orchestrator.tick("agent-1", "user-1", "t1");
        assertThat(tick).isInstanceOf(UserModelTick.Updated.class);

        var updated = (UserModelTick.Updated) tick;
        assertThat(updated.profile().totalInteractions()).isEqualTo(1);
        assertThat(updated.profile().positiveSignals()).isEqualTo(1);

        verify(profileStore).store(any(UserProfile.class));
    }

    @Test
    void multiplePositiveSignalsIncreaseStage() {
        for (int i = 0; i < 20; i++) {
            orchestrator.record(
                new InteractionSignal.CustomSignal("chat " + i, QualitySignal.POSITIVE),
                "agent-1", "user-1", "t1");
        }

        var tick = orchestrator.tick("agent-1", "user-1", "t1");
        assertThat(tick).isInstanceOf(UserModelTick.Updated.class);
        var profile = ((UserModelTick.Updated) tick).profile();
        assertThat(profile.relationshipStage()).isNotEqualTo("stranger");
    }

    @Test
    void currentProfileReturnsNullBeforeFirstTick() {
        assertThat(orchestrator.currentProfile("agent-1", "user-1", "t1")).isNull();
    }

    @Test
    void currentProfileReturnsCachedAfterTick() {
        orchestrator.record(
            new InteractionSignal.CustomSignal("hi", QualitySignal.POSITIVE),
            "agent-1", "user-1", "t1");
        orchestrator.tick("agent-1", "user-1", "t1");

        assertThat(orchestrator.currentProfile("agent-1", "user-1", "t1"))
            .isNotNull();
    }

    @Test
    void tickDoesNotTriggerLlmBelowMinSignals() {
        orchestrator.record(
            new InteractionSignal.CustomSignal("hi", QualitySignal.POSITIVE),
            "agent-1", "user-1", "t1");

        orchestrator.tick("agent-1", "user-1", "t1");

        verifyNoInteractions(agentProvider);
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `mvn --batch-mode test -pl blocks -Dtest="FamiliarityScoreTest,UserModelOrchestratorTest"`
Expected: FAIL — UserModelOrchestrator not found

- [ ] **Step 4: Implement SynthesisResult (package-private)**

```java
package io.casehub.blocks.agentic.social;

import org.jspecify.annotations.Nullable;

record SynthesisResult(
    @Nullable String communicationStyle,
    @Nullable String topicsOfInterest,
    @Nullable String preferences,
    @Nullable String synthesisNotes) {}
```

- [ ] **Step 5: Implement UserModelOrchestrator**

`@ApplicationScoped` CDI bean. Key implementation details:

1. **Per-subject state** in `ConcurrentHashMap<String, SubjectState>` keyed by
   `agentId:subjectId:tenantId`. `SubjectState` is a package-private mutable
   holder with signal counters, text buffer, cumulative totals, timestamps,
   cached profile, and a `ReentrantLock`.

2. **`record()`**: under state-object synchronization, increment the counter
   matching `signal.quality()`, append `signal.description()` to text buffer,
   update `lastInteractionTimestamp` and `totalInteractions`.

3. **`tick()`**: acquire per-subject lock. Snapshot and clear signal counters
   and text buffer. Merge into cumulatives. Compute familiarity score via
   `computeFamiliarity()` (static, package-visible for testing). Resolve stage.
   If stage or score unchanged → `Unchanged`. Otherwise build `UserProfile`,
   check LLM synthesis gate (`textBuffer.length() >= minSignalsForSynthesis`
   AND `timeSinceLastSynthesis >= synthesisCooldown`), invoke `AgentProvider`
   if gate passes (same text-collect-and-parse pattern as InnerLife), persist
   via `profileStore.store()`, return `Updated` or `Synthesised`.

4. **`computeFamiliarity()`**: static package-visible method:
   `rawScore = (positive * positiveWeight - negative * negativeWeight) / (positive + negative + neutral + 1)`,
   normalise to [0,1], apply decay: `score * pow(1 - decayRate, ticksSinceLastInteraction)`.

5. **`currentProfile()`**: return cached profile from state, null if absent.

6. **LLM prompt**: system prompt instructs JSON output with `communicationStyle`,
   `topicsOfInterest`, `preferences`, `synthesisNotes`. User prompt includes
   current profile summary + recent signal descriptions. Parse via JSON.
   On failure: log warning, retain previous LLM fields.

7. **Eviction**: during tick, scan state map for entries where
   `now - lastActivityTimestamp > evictionTimeout`, remove them.

- [ ] **Step 6: Run tests — verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="FamiliarityScoreTest,UserModelOrchestratorTest"`
Expected: PASS

- [ ] **Step 7: Write LLM synthesis test**

```java
@Test
void tickTriggersLlmWhenMinSignalsReached() {
    var config = new UserModelConfig(
        3, Duration.ofMillis(0), 0.01, 1.0, 0.5,
        RelationshipStageConfig.defaults(),
        Duration.ofHours(1), Duration.ofDays(7),
        "user-model", "user-profile", 50);
    orchestrator = new UserModelOrchestrator(profileStore, agentProvider, config);

    mockAgentProviderResponse("{\"communicationStyle\":\"casual\",\"topicsOfInterest\":\"gaming\",\"preferences\":null,\"synthesisNotes\":null}");

    for (int i = 0; i < 5; i++) {
        orchestrator.record(
            new InteractionSignal.CustomSignal("event " + i, QualitySignal.POSITIVE),
            "a", "s", "t");
    }

    var tick = orchestrator.tick("a", "s", "t");
    assertThat(tick).isInstanceOf(UserModelTick.Synthesised.class);
    var synth = (UserModelTick.Synthesised) tick;
    assertThat(synth.profile().communicationStyle()).isEqualTo("casual");
    assertThat(synth.profile().topicsOfInterest()).isEqualTo("gaming");
}

@Test
void llmParseFailureRetainsPreviousFields() {
    var config = new UserModelConfig(
        1, Duration.ofMillis(0), 0.01, 1.0, 0.5,
        RelationshipStageConfig.defaults(),
        Duration.ofHours(1), Duration.ofDays(7),
        "user-model", "user-profile", 50);
    orchestrator = new UserModelOrchestrator(profileStore, agentProvider, config);

    mockAgentProviderResponse("not json at all");

    orchestrator.record(
        new InteractionSignal.CustomSignal("test", QualitySignal.POSITIVE),
        "a", "s", "t");

    var tick = orchestrator.tick("a", "s", "t");
    assertThat(tick).isInstanceOf(UserModelTick.Updated.class);
}
```

(Helper `mockAgentProviderResponse(String)` sets up AgentProvider mock to return
a Multi emitting a TextDelta event with the given text, following the established
pattern from InnerLifeOrchestratorTest.)

- [ ] **Step 8: Run all orchestrator tests**

Run: `mvn --batch-mode test -pl blocks -Dtest="FamiliarityScoreTest,UserModelOrchestratorTest"`
Expected: PASS

- [ ] **Step 9: Run full test suite**

Run: `mvn --batch-mode test -pl blocks`
Expected: all tests pass

- [ ] **Step 10: Commit**

```bash
git add blocks/src
git commit -m "feat(#122): implement UserModelOrchestrator — tiered synthesis with record()+tick(), heuristic fold + LLM gate

Refs #122"
```

---

## Batch 3: Documentation

### Task 5: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: all types from Tasks 2-4
- Produces: updated CLAUDE.md with `agentic.social` package listing including UserModel types

- [ ] **Step 1: Add UserModel types to the personality/social package table**

Update the existing `agentic/personality` section in CLAUDE.md to `agentic/social`
and add UserModel entries:

| Class | What it does |
|---|---|
| `InteractionSignal` | Sealed: `RelationshipSignal`, `ExperienceSignal`, `CustomSignal` — uniform signal recording API |
| `UserProfile` | Record: core heuristic fields + LLM-synthesised fields + metadata |
| `UserModelTick` | Sealed: `Unchanged`, `Updated`, `Synthesised` |
| `UserModelOrchestrator` | CDI bean: `record()`, `tick()`, `currentProfile()` — per-subject profile synthesis |
| `UserModelConfig` | Record with defaults: synthesis gate, decay, stage config |
| `RelationshipStageConfig` | Record: stage tiers, decay rate, signal weights |
| `StageTier` | Record: `(String name, double threshold)` |
| `UserProfileStore` | SPI: `store()`, `lookup()`, `findByAgent()`, `eraseSubject()` |
| `CbrUserProfileStore` | `@DefaultBean`: CbrCaseMemoryStore-backed UserProfileStore |

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(#122): add UserModel types to CLAUDE.md — social package

Refs #122"
```

---

## References

- `2026-08-20-user-model-design.md` — design spec this plan implements
- `2026-08-18-personality-evolution-design.md` — established orchestrator pattern
- `2026-08-18-inner-life-design.md` — established tick + LLM pattern
- `RelationshipEvent.java` (neocortex-memory-api) — composed relationship event type
- `ExperienceEvent.java` (neocortex-memory-api) — composed experience event type
- `CbrCaseMemoryStore` (neocortex-memory-api) — backing store for profile persistence
- `AgentProvider` (platform-agent-api) — LLM invocation for open-ended synthesis
- GE-20260820-c19b68 — CbrQuery producerAgentId post-filtering
- GE-20260811-e941cc — type distinction gotcha (informed UserProfileStore SPI design)
- GE-20260820-aa31ab — composite score masking gotcha
- GitHub #122 — focal issue
- GitHub #126 — epic
