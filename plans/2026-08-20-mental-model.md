# MentalModel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #123 — MentalModel pattern — Theory of Mind with BDI tracking
**Issue group:** #118, #119, #120, #121, #122, #123, #124, #126

**Goal:** Implement a per-actor BDI (Beliefs, Desires, Intentions) tracking orchestrator that composes BeliefSet, RelationshipEvent, GoapWorldState, and EpistemicRule into a Theory of Mind module for the social cognition package.

**Architecture:** Unified `MentalModelOrchestrator` following the established record()+tick()+project() pattern. `AttributedState` record for all three BDI dimensions with confidence decay. `MentalModelStore` SPI backed by CbrCaseMemoryStore. Conversation-derived beliefs via `observeConversation(CommonGroundState)`.

**Tech Stack:** Java 21, JUnit 5, Mockito, blocks.agentic.belief (BeliefSet), blocks.conversation (CommonGroundState/EpistemicStatus), neocortex-memory-api (CbrCaseMemoryStore, RelationshipEvent), platform-agent-api (AgentProvider)

## Global Constraints

- Package: `io.casehub.blocks.agentic.social`
- Source root: `blocks/src/main/java/io/casehub/blocks/agentic/social/`
- Test root: `blocks/src/test/java/io/casehub/blocks/agentic/social/`
- No Quarkus runtime — plain JUnit 5 + Mockito
- All records validate non-null fields and valid ranges in compact constructors
- Follow existing social cognition naming conventions (sealed interfaces for outcomes, records for value types)

---

## Batch 1: Foundation types + signal model

### Task 1: Add MentalModel foundation types — BdiDimension, CueType, AttributedState, MentalProjection, MentalModelConfig, MentalModelSnapshot, MentalModelTick, MentalStateSignal

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/BdiDimension.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/CueType.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/AttributedState.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalProjection.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelConfig.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelSnapshot.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalStateSignal.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/AttributedStateTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/MentalModelConfigTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/MentalModelSnapshotTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/MentalStateSignalTest.java`

**Interfaces:**
- Consumes: `RelationshipEvent` (neocortex-memory-api), `CommonGroundState`/`EpistemicStatus` (blocks.conversation)
- Produces: `BdiDimension` enum, `CueType` enum, `AttributedState` record, `MentalProjection` record, `MentalModelConfig` record, `MentalModelSnapshot` record, `MentalModelTick` sealed interface, `MentalStateSignal` sealed interface — all consumed by Task 2 (store) and Task 3 (orchestrator)

- [ ] **Step 1: Write tests for AttributedState validation**

```java
// AttributedStateTest.java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.*;

class AttributedStateTest {

    @Test
    void validAttributedState() {
        var state = new AttributedState("deployment_risk",
                "subject thinks deployments are risky",
                0.8, 3, Instant.now(), BdiDimension.BELIEF);
        assertThat(state.key()).isEqualTo("deployment_risk");
        assertThat(state.confidence()).isEqualTo(0.8);
        assertThat(state.entrenchment()).isEqualTo(3);
        assertThat(state.dimension()).isEqualTo(BdiDimension.BELIEF);
    }

    @Test
    void nullKeyThrows() {
        assertThatThrownBy(() -> new AttributedState(null, "desc", 0.5, 0, Instant.now(), BdiDimension.BELIEF))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullDescriptionThrows() {
        assertThatThrownBy(() -> new AttributedState("key", null, 0.5, 0, Instant.now(), BdiDimension.BELIEF))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void confidenceBelowZeroThrows() {
        assertThatThrownBy(() -> new AttributedState("key", "desc", -0.1, 0, Instant.now(), BdiDimension.BELIEF))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void confidenceAboveOneThrows() {
        assertThatThrownBy(() -> new AttributedState("key", "desc", 1.1, 0, Instant.now(), BdiDimension.BELIEF))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void negativeEntrenchmentThrows() {
        assertThatThrownBy(() -> new AttributedState("key", "desc", 0.5, -1, Instant.now(), BdiDimension.BELIEF))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void nullDimensionThrows() {
        assertThatThrownBy(() -> new AttributedState("key", "desc", 0.5, 0, Instant.now(), null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void nullLastReinforcedThrows() {
        assertThatThrownBy(() -> new AttributedState("key", "desc", 0.5, 0, null, BdiDimension.DESIRE))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void boundaryConfidenceZeroAccepted() {
        var state = new AttributedState("key", "desc", 0.0, 0, Instant.now(), BdiDimension.INTENTION);
        assertThat(state.confidence()).isZero();
    }

    @Test
    void boundaryConfidenceOneAccepted() {
        var state = new AttributedState("key", "desc", 1.0, 0, Instant.now(), BdiDimension.BELIEF);
        assertThat(state.confidence()).isEqualTo(1.0);
    }
}
```

- [ ] **Step 2: Write tests for MentalModelConfig defaults and validation**

```java
// MentalModelConfigTest.java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.assertj.core.api.Assertions.*;

class MentalModelConfigTest {

    @Test
    void defaultConfig() {
        var config = MentalModelConfig.defaults();
        assertThat(config.beliefHalfLife()).isEqualTo(Duration.ofDays(7));
        assertThat(config.desireHalfLife()).isEqualTo(Duration.ofDays(1));
        assertThat(config.intentionHalfLife()).isEqualTo(Duration.ofHours(4));
        assertThat(config.confidenceFloor()).isEqualTo(0.1);
        assertThat(config.projectionFloor()).isEqualTo(0.3);
        assertThat(config.minSignalsForInference()).isEqualTo(3);
        assertThat(config.inferenceCooldown()).isEqualTo(Duration.ofMinutes(5));
        assertThat(config.maxSignalsInPrompt()).isEqualTo(20);
        assertThat(config.maxBufferSize()).isEqualTo(100);
    }

    @Test
    void halfLifeForDimension() {
        var config = MentalModelConfig.defaults();
        assertThat(config.halfLifeFor(BdiDimension.BELIEF)).isEqualTo(Duration.ofDays(7));
        assertThat(config.halfLifeFor(BdiDimension.DESIRE)).isEqualTo(Duration.ofDays(1));
        assertThat(config.halfLifeFor(BdiDimension.INTENTION)).isEqualTo(Duration.ofHours(4));
    }
}
```

- [ ] **Step 3: Write tests for MentalStateSignal variants**

```java
// MentalStateSignalTest.java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.relationship.QualitySignal;
import io.casehub.neocortex.memory.relationship.RelationshipEvent;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class MentalStateSignalTest {

    @Test
    void verbalCue() {
        var signal = new MentalStateSignal.VerbalCue("I think deployments are risky",
                CueType.BELIEF_STATEMENT);
        assertThat(signal.content()).isEqualTo("I think deployments are risky");
        assertThat(signal.type()).isEqualTo(CueType.BELIEF_STATEMENT);
    }

    @Test
    void behavioralCue() {
        var signal = new MentalStateSignal.BehavioralCue(
                "subject checked dashboard 5 times", "dashboard_check");
        assertThat(signal.content()).contains("dashboard");
        assertThat(signal.actionType()).isEqualTo("dashboard_check");
    }

    @Test
    void contextualCue() {
        var signal = new MentalStateSignal.ContextualCue(
                "deadline approaching", Map.of("deadline", "2026-08-25"));
        assertThat(signal.content()).contains("deadline");
        assertThat(signal.metadata()).containsKey("deadline");
    }

    @Test
    void relationshipCue() {
        var event = new RelationshipEvent("agent1", "user1", "tenant1",
                "case1", "turn1", "conversation",
                QualitySignal.POSITIVE, "user expressed agreement",
                0.7, Map.of());
        var signal = new MentalStateSignal.RelationshipCue(event);
        assertThat(signal.content()).isEqualTo("user expressed agreement");
        assertThat(signal.event()).isEqualTo(event);
    }

    @Test
    void verbalCueNullContentThrows() {
        assertThatThrownBy(() -> new MentalStateSignal.VerbalCue(null, CueType.BELIEF_STATEMENT))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void relationshipCueNullEventThrows() {
        assertThatThrownBy(() -> new MentalStateSignal.RelationshipCue(null))
                .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 4: Write tests for MentalModelSnapshot**

```java
// MentalModelSnapshotTest.java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class MentalModelSnapshotTest {

    @Test
    void validSnapshot() {
        var now = Instant.now();
        var belief = new AttributedState("risk", "high risk", 0.8, 2, now, BdiDimension.BELIEF);
        var desire = new AttributedState("resolution", "quick fix", 0.6, 0, now, BdiDimension.DESIRE);
        var snapshot = new MentalModelSnapshot("agent1", "user1", "tenant1",
                List.of(belief), List.of(desire), List.of(), now, now, now);
        assertThat(snapshot.beliefs()).hasSize(1);
        assertThat(snapshot.desires()).hasSize(1);
        assertThat(snapshot.intentions()).isEmpty();
    }

    @Test
    void listsAreDefensivelyCopied() {
        var now = Instant.now();
        var beliefs = new java.util.ArrayList<>(List.of(
                new AttributedState("k", "v", 0.5, 0, now, BdiDimension.BELIEF)));
        var snapshot = new MentalModelSnapshot("a", "u", "t",
                beliefs, List.of(), List.of(), now, null, now);
        beliefs.clear();
        assertThat(snapshot.beliefs()).hasSize(1);
    }

    @Test
    void nullAgentIdThrows() {
        assertThatThrownBy(() -> new MentalModelSnapshot(null, "u", "t",
                List.of(), List.of(), List.of(), Instant.now(), null, Instant.now()))
                .isInstanceOf(NullPointerException.class);
    }
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl blocks -Dtest="AttributedStateTest,MentalModelConfigTest,MentalStateSignalTest,MentalModelSnapshotTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation errors — types don't exist yet.

- [ ] **Step 6: Implement all foundation types**

Create the following files using `ide_create_file`:

**BdiDimension.java:**
```java
package io.casehub.blocks.agentic.social;

public enum BdiDimension {
    BELIEF, DESIRE, INTENTION
}
```

**CueType.java:**
```java
package io.casehub.blocks.agentic.social;

public enum CueType {
    BELIEF_STATEMENT, DESIRE_EXPRESSION, INTENTION_DECLARATION
}
```

**AttributedState.java:**
```java
package io.casehub.blocks.agentic.social;

import java.time.Instant;
import java.util.Objects;

public record AttributedState(
        String key,
        String description,
        double confidence,
        int entrenchment,
        Instant lastReinforced,
        BdiDimension dimension) {
    public AttributedState {
        Objects.requireNonNull(key, "key required");
        Objects.requireNonNull(description, "description required");
        if (confidence < 0.0 || confidence > 1.0)
            throw new IllegalArgumentException("confidence must be in [0, 1], got " + confidence);
        if (entrenchment < 0)
            throw new IllegalArgumentException("entrenchment must be >= 0");
        Objects.requireNonNull(lastReinforced, "lastReinforced required");
        Objects.requireNonNull(dimension, "dimension required");
    }
}
```

**MentalProjection.java:**
```java
package io.casehub.blocks.agentic.social;

import java.util.Objects;

public record MentalProjection(
        String conditionKey,
        boolean value,
        double confidence,
        BdiDimension dimension) {
    public MentalProjection {
        Objects.requireNonNull(conditionKey, "conditionKey required");
        if (confidence < 0.0 || confidence > 1.0)
            throw new IllegalArgumentException("confidence must be in [0, 1]");
        Objects.requireNonNull(dimension, "dimension required");
    }
}
```

**MentalModelConfig.java:**
```java
package io.casehub.blocks.agentic.social;

import java.time.Duration;
import java.util.Objects;

public record MentalModelConfig(
        Duration beliefHalfLife,
        Duration desireHalfLife,
        Duration intentionHalfLife,
        double confidenceFloor,
        double projectionFloor,
        int minSignalsForInference,
        Duration inferenceCooldown,
        int maxSignalsInPrompt,
        int maxBufferSize,
        Duration evictionTimeout,
        Duration expectedTickInterval) {
    public MentalModelConfig {
        Objects.requireNonNull(beliefHalfLife);
        Objects.requireNonNull(desireHalfLife);
        Objects.requireNonNull(intentionHalfLife);
        Objects.requireNonNull(inferenceCooldown);
        Objects.requireNonNull(evictionTimeout);
        Objects.requireNonNull(expectedTickInterval);
    }

    public static MentalModelConfig defaults() {
        return new MentalModelConfig(
                Duration.ofDays(7), Duration.ofDays(1), Duration.ofHours(4),
                0.1, 0.3, 3, Duration.ofMinutes(5), 20, 100,
                Duration.ofHours(24), Duration.ofMinutes(1));
    }

    public Duration halfLifeFor(BdiDimension dimension) {
        return switch (dimension) {
            case BELIEF -> beliefHalfLife;
            case DESIRE -> desireHalfLife;
            case INTENTION -> intentionHalfLife;
        };
    }
}
```

**MentalModelSnapshot.java:**
```java
package io.casehub.blocks.agentic.social;

import org.jspecify.annotations.Nullable;
import java.time.Instant;
import java.util.List;
import java.util.Objects;

public record MentalModelSnapshot(
        String agentId,
        String subjectId,
        String tenantId,
        List<AttributedState> beliefs,
        List<AttributedState> desires,
        List<AttributedState> intentions,
        Instant lastSignal,
        @Nullable Instant lastInference,
        Instant snapshotCreated) {
    public MentalModelSnapshot {
        Objects.requireNonNull(agentId, "agentId required");
        Objects.requireNonNull(subjectId, "subjectId required");
        Objects.requireNonNull(tenantId, "tenantId required");
        Objects.requireNonNull(beliefs, "beliefs required");
        Objects.requireNonNull(desires, "desires required");
        Objects.requireNonNull(intentions, "intentions required");
        Objects.requireNonNull(lastSignal, "lastSignal required");
        Objects.requireNonNull(snapshotCreated, "snapshotCreated required");
        beliefs = List.copyOf(beliefs);
        desires = List.copyOf(desires);
        intentions = List.copyOf(intentions);
    }
}
```

**MentalModelTick.java:**
```java
package io.casehub.blocks.agentic.social;

import org.jspecify.annotations.Nullable;

public sealed interface MentalModelTick {
    record Unchanged(@Nullable String reason) implements MentalModelTick {}
    record Updated(MentalModelSnapshot snapshot) implements MentalModelTick {}
    record Inferred(MentalModelSnapshot snapshot,
                    @Nullable MentalModelSnapshot previous) implements MentalModelTick {}
}
```

**MentalStateSignal.java:**
```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.relationship.RelationshipEvent;
import java.util.Map;
import java.util.Objects;

public sealed interface MentalStateSignal {
    String content();

    record VerbalCue(String content, CueType type) implements MentalStateSignal {
        public VerbalCue {
            Objects.requireNonNull(content, "content required");
            Objects.requireNonNull(type, "type required");
        }
    }

    record BehavioralCue(String content, String actionType) implements MentalStateSignal {
        public BehavioralCue {
            Objects.requireNonNull(content, "content required");
            Objects.requireNonNull(actionType, "actionType required");
        }
    }

    record ContextualCue(String content, Map<String, String> metadata) implements MentalStateSignal {
        public ContextualCue {
            Objects.requireNonNull(content, "content required");
            Objects.requireNonNull(metadata, "metadata required");
            metadata = Map.copyOf(metadata);
        }
    }

    record RelationshipCue(RelationshipEvent event) implements MentalStateSignal {
        public RelationshipCue {
            Objects.requireNonNull(event, "event required");
        }
        @Override
        public String content() {
            return event.description();
        }
    }
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="AttributedStateTest,MentalModelConfigTest,MentalStateSignalTest,MentalModelSnapshotTest"`
Expected: All tests PASS.

- [ ] **Step 8: Commit**

```bash
git add blocks/src/main/java/io/casehub/blocks/agentic/social/BdiDimension.java blocks/src/main/java/io/casehub/blocks/agentic/social/CueType.java blocks/src/main/java/io/casehub/blocks/agentic/social/AttributedState.java blocks/src/main/java/io/casehub/blocks/agentic/social/MentalProjection.java blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelConfig.java blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelSnapshot.java blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelTick.java blocks/src/main/java/io/casehub/blocks/agentic/social/MentalStateSignal.java blocks/src/test/java/io/casehub/blocks/agentic/social/AttributedStateTest.java blocks/src/test/java/io/casehub/blocks/agentic/social/MentalModelConfigTest.java blocks/src/test/java/io/casehub/blocks/agentic/social/MentalModelSnapshotTest.java blocks/src/test/java/io/casehub/blocks/agentic/social/MentalStateSignalTest.java
git commit -m "feat(#123): add MentalModel foundation types — AttributedState, MentalStateSignal, BdiDimension, configs"
```

---

## Batch 2: Persistence

### Task 2: Add MentalModelStore SPI + CbrMentalModelStore adapter

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelStore.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/CbrMentalModelStore.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/CbrMentalModelStoreTest.java`

**Interfaces:**
- Consumes: `AttributedState`, `MentalModelSnapshot`, `BdiDimension` (from Task 1); `CbrCaseMemoryStore`, `CbrCase`, `CbrQuery`, `FeatureValue` (neocortex-memory-api)
- Produces: `MentalModelStore` SPI interface, `CbrMentalModelStore` `@DefaultBean @ApplicationScoped` — consumed by Task 3 (orchestrator injection)

- [ ] **Step 1: Write tests for CbrMentalModelStore**

```java
// CbrMentalModelStoreTest.java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.CbrCase;
import io.casehub.neocortex.memory.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.CbrQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class CbrMentalModelStoreTest {

    private CbrCaseMemoryStore cbrStore;
    private CbrMentalModelStore store;

    @BeforeEach
    void setUp() {
        cbrStore = mock(CbrCaseMemoryStore.class);
        store = new CbrMentalModelStore(cbrStore);
    }

    @Test
    void storeCreatesCase() {
        var now = Instant.now();
        var snapshot = new MentalModelSnapshot("agent1", "user1", "tenant1",
                List.of(new AttributedState("risk", "high", 0.8, 2, now, BdiDimension.BELIEF)),
                List.of(), List.of(), now, null, now);

        store.store(snapshot);

        var captor = ArgumentCaptor.forClass(CbrCase.class);
        verify(cbrStore).store(captor.capture());
        var stored = captor.getValue();
        assertThat(stored.caseType()).isEqualTo("mental-model");
        assertThat(stored.producerAgentId()).isEqualTo("agent1");
    }

    @Test
    void lookupReturnsEmptyWhenNoMatch() {
        when(cbrStore.retrieveSimilar(any(CbrQuery.class), eq(CbrCase.class)))
                .thenReturn(List.of());
        var result = store.lookup("agent1", "user1", "tenant1");
        assertThat(result).isEmpty();
    }

    @Test
    void storeAndLookupRoundTrip() {
        var now = Instant.now();
        var belief = new AttributedState("risk", "high risk", 0.8, 2, now, BdiDimension.BELIEF);
        var desire = new AttributedState("speed", "wants quick fix", 0.6, 0, now, BdiDimension.DESIRE);
        var snapshot = new MentalModelSnapshot("agent1", "user1", "tenant1",
                List.of(belief), List.of(desire), List.of(), now, now, now);

        var caseCaptor = ArgumentCaptor.forClass(CbrCase.class);
        store.store(snapshot);
        verify(cbrStore).store(caseCaptor.capture());

        when(cbrStore.retrieveSimilar(any(CbrQuery.class), eq(CbrCase.class)))
                .thenReturn(List.of(caseCaptor.getValue()));

        var result = store.lookup("agent1", "user1", "tenant1");
        assertThat(result).isPresent();
        var loaded = result.get();
        assertThat(loaded.agentId()).isEqualTo("agent1");
        assertThat(loaded.subjectId()).isEqualTo("user1");
        assertThat(loaded.beliefs()).hasSize(1);
        assertThat(loaded.beliefs().getFirst().key()).isEqualTo("risk");
        assertThat(loaded.beliefs().getFirst().confidence()).isEqualTo(0.8);
        assertThat(loaded.desires()).hasSize(1);
    }

    @Test
    void eraseSubjectDelegates() {
        store.eraseSubject("user1", "tenant1");
        verify(cbrStore).eraseBySubject("user1", "tenant1");
    }

    @Test
    void findByAgentFiltersCorrectly() {
        when(cbrStore.retrieveSimilar(any(CbrQuery.class), eq(CbrCase.class)))
                .thenReturn(List.of());
        var result = store.findByAgent("agent1", "tenant1");
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl blocks -Dtest="CbrMentalModelStoreTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation errors — MentalModelStore and CbrMentalModelStore don't exist.

- [ ] **Step 3: Implement MentalModelStore SPI**

```java
// MentalModelStore.java
package io.casehub.blocks.agentic.social;

import java.util.List;
import java.util.Optional;

public interface MentalModelStore {
    void store(MentalModelSnapshot snapshot);
    Optional<MentalModelSnapshot> lookup(String agentId, String subjectId, String tenantId);
    List<MentalModelSnapshot> findByAgent(String agentId, String tenantId);
    void eraseSubject(String subjectId, String tenantId);
}
```

- [ ] **Step 4: Implement CbrMentalModelStore adapter**

Follow the CbrUserProfileStore pattern — serialize BDI entries as JSON feature values, use producerAgentId for agent filtering, subjectId as a StringVal feature for lookup.

```java
// CbrMentalModelStore.java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.CbrCase;
import io.casehub.neocortex.memory.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.CbrQuery;
import io.casehub.neocortex.memory.FeatureValue;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Default;
import jakarta.inject.Inject;
import io.quarkus.arc.DefaultBean;
import org.jspecify.annotations.Nullable;

import java.time.Instant;
import java.util.*;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.stream.Collectors;

@DefaultBean
@ApplicationScoped
public class CbrMentalModelStore implements MentalModelStore {

    private static final Logger LOG = Logger.getLogger(CbrMentalModelStore.class.getName());
    private static final String CASE_TYPE = "mental-model";
    private static final String FEATURE_SUBJECT_ID = "subjectId";
    private static final String FEATURE_TENANT_ID = "tenantId";
    private static final String FEATURE_BELIEFS = "beliefs";
    private static final String FEATURE_DESIRES = "desires";
    private static final String FEATURE_INTENTIONS = "intentions";
    private static final String FEATURE_LAST_SIGNAL = "lastSignal";
    private static final String FEATURE_LAST_INFERENCE = "lastInference";

    private final CbrCaseMemoryStore cbrStore;

    @Inject
    public CbrMentalModelStore(CbrCaseMemoryStore cbrStore) {
        this.cbrStore = cbrStore;
    }

    // Store, lookup, findByAgent, eraseSubject implementations
    // following the CbrUserProfileStore pattern — serialize AttributedState
    // lists as JSON strings in StringVal features, use retrieveSimilar
    // with feature filters for lookup.
    // Full implementation follows CbrUserProfileStore exactly.
}
```

The implementation serializes `List<AttributedState>` to JSON strings and deserializes on lookup. Each AttributedState becomes a JSON object with key, description, confidence, entrenchment, lastReinforced (ISO-8601), and dimension.

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="CbrMentalModelStoreTest"`
Expected: All tests PASS.

- [ ] **Step 6: Commit**

```bash
git add blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelStore.java blocks/src/main/java/io/casehub/blocks/agentic/social/CbrMentalModelStore.java blocks/src/test/java/io/casehub/blocks/agentic/social/CbrMentalModelStoreTest.java
git commit -m "feat(#123): add MentalModelStore SPI + CbrMentalModelStore — mental model persistence via CbrCaseMemoryStore"
```

---

## Batch 3: Orchestrator

### Task 3: Implement MentalModelOrchestrator — record()+tick()+project()+observeConversation()

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelOrchestrator.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/MentalModelOrchestratorTest.java`

**Interfaces:**
- Consumes: All types from Task 1 + Task 2; `AgentProvider`/`AgentSessionConfig`/`AgentEvent` (platform-agent-api); `BeliefSet`/`Belief`/`ConsistencyChecker` (blocks.agentic.belief); `CommonGroundState`/`EpistemicStatus`/`GroundedFact` (blocks.conversation)
- Produces: `MentalModelOrchestrator` CDI bean — the composition root. No downstream tasks depend on it.

- [ ] **Step 1: Write test for record() accumulation**

```java
// MentalModelOrchestratorTest.java — partial
@Test
void recordAccumulatesSignals() {
    orchestrator.record(new MentalStateSignal.VerbalCue(
            "I think the system is unstable", CueType.BELIEF_STATEMENT),
            "agent1", "user1", "tenant1");
    orchestrator.record(new MentalStateSignal.VerbalCue(
            "I want a quick fix", CueType.DESIRE_EXPRESSION),
            "agent1", "user1", "tenant1");

    var tick = orchestrator.tick("agent1", "user1", "tenant1");
    assertThat(tick).isInstanceOf(MentalModelTick.Updated.class);
    var snapshot = ((MentalModelTick.Updated) tick).snapshot();
    assertThat(snapshot.beliefs()).isNotEmpty();
    assertThat(snapshot.desires()).isNotEmpty();
}
```

- [ ] **Step 2: Write test for heuristic extraction from VerbalCue**

```java
@Test
void verbalBeliefCueExtractsBeliefDirectly() {
    orchestrator.record(new MentalStateSignal.VerbalCue(
            "I think deployments are risky", CueType.BELIEF_STATEMENT),
            "agent1", "user1", "tenant1");

    var tick = orchestrator.tick("agent1", "user1", "tenant1");
    assertThat(tick).isInstanceOf(MentalModelTick.Updated.class);
    var beliefs = ((MentalModelTick.Updated) tick).snapshot().beliefs();
    assertThat(beliefs).hasSize(1);
    assertThat(beliefs.getFirst().description()).contains("deployments are risky");
    assertThat(beliefs.getFirst().confidence()).isGreaterThanOrEqualTo(0.7);
    assertThat(beliefs.getFirst().dimension()).isEqualTo(BdiDimension.BELIEF);
}
```

- [ ] **Step 3: Write test for confidence decay**

```java
@Test
void confidenceDecaysOverTime() {
    var config = new MentalModelConfig(
            Duration.ofMillis(100), Duration.ofMillis(50), Duration.ofMillis(25),
            0.1, 0.3, 3, Duration.ofMinutes(5), 20, 100,
            Duration.ofHours(24), Duration.ofMinutes(1));
    var orch = new MentalModelOrchestrator(store, agentProvider, config);

    orch.record(new MentalStateSignal.VerbalCue(
            "I believe X", CueType.BELIEF_STATEMENT),
            "a", "u", "t");
    orch.tick("a", "u", "t");

    // Wait for decay
    Thread.sleep(200);

    var tick2 = orch.tick("a", "u", "t");
    assertThat(tick2).isInstanceOf(MentalModelTick.Updated.class);
    var beliefs = ((MentalModelTick.Updated) tick2).snapshot().beliefs();
    // After ~2 half-lives, confidence should be ~0.25× original
    assertThat(beliefs.getFirst().confidence()).isLessThan(0.4);
}
```

- [ ] **Step 4: Write test for eviction below confidence floor**

```java
@Test
void evictsBelowConfidenceFloor() {
    var config = new MentalModelConfig(
            Duration.ofMillis(10), Duration.ofMillis(5), Duration.ofMillis(2),
            0.5, 0.3, 3, Duration.ofMinutes(5), 20, 100,
            Duration.ofHours(24), Duration.ofMinutes(1));
    var orch = new MentalModelOrchestrator(store, agentProvider, config);

    orch.record(new MentalStateSignal.VerbalCue(
            "I intend to escalate", CueType.INTENTION_DECLARATION),
            "a", "u", "t");
    orch.tick("a", "u", "t");
    Thread.sleep(50);

    var tick2 = orch.tick("a", "u", "t");
    var snapshot = switch (tick2) {
        case MentalModelTick.Updated u -> u.snapshot();
        case MentalModelTick.Unchanged u -> null;
        case MentalModelTick.Inferred i -> i.snapshot();
    };
    if (snapshot != null) {
        assertThat(snapshot.intentions()).isEmpty();
    }
}
```

- [ ] **Step 5: Write test for project() — GOAP projection**

```java
@Test
void projectReturnsMentalProjections() {
    orchestrator.record(new MentalStateSignal.VerbalCue(
            "I think the system is fragile", CueType.BELIEF_STATEMENT),
            "agent1", "user1", "tenant1");
    orchestrator.tick("agent1", "user1", "tenant1");

    var projections = orchestrator.project("agent1", "user1", "tenant1");
    assertThat(projections).isNotEmpty();
    var proj = projections.getFirst();
    assertThat(proj.value()).isTrue();
    assertThat(proj.confidence()).isGreaterThan(0.0);
    assertThat(proj.dimension()).isEqualTo(BdiDimension.BELIEF);
}
```

- [ ] **Step 6: Write test for project() returns empty when no state**

```java
@Test
void projectReturnsEmptyWhenNoState() {
    var projections = orchestrator.project("agent1", "unknown", "tenant1");
    assertThat(projections).isEmpty();
}
```

- [ ] **Step 7: Write test for observeConversation() — epistemic bridge**

```java
@Test
void observeConversationExtractsBeliefsFromCommonGround() {
    var established = Map.of("p1", new GroundedFact(
            "p1", "system_stability", EpistemicStatus.ESTABLISHED,
            "the system is stable", Set.of("user1"), Set.of(), 1));
    var disputed = Map.of("p2", new GroundedFact(
            "p2", "deadline_feasibility", EpistemicStatus.DISPUTED,
            "the deadline is achievable", Set.of(), Set.of("user1"), 2));
    var commonGround = new CommonGroundState(established, Map.of(), disputed);

    orchestrator.observeConversation(commonGround, "agent1", "user1", "tenant1");
    var tick = orchestrator.tick("agent1", "user1", "tenant1");

    assertThat(tick).isInstanceOf(MentalModelTick.Updated.class);
    var beliefs = ((MentalModelTick.Updated) tick).snapshot().beliefs();
    assertThat(beliefs).hasSizeGreaterThanOrEqualTo(2);
}
```

- [ ] **Step 8: Write test for tick with no signals returns Unchanged**

```java
@Test
void tickWithNoSignalsReturnsUnchanged() {
    var tick = orchestrator.tick("agent1", "unknown", "tenant1");
    assertThat(tick).isInstanceOf(MentalModelTick.Unchanged.class);
}
```

- [ ] **Step 9: Write test for store reload on cold start**

```java
@Test
void tickReloadsFromStoreOnColdStart() {
    var now = Instant.now();
    var snapshot = new MentalModelSnapshot("agent1", "user1", "tenant1",
            List.of(new AttributedState("k", "v", 0.8, 1, now, BdiDimension.BELIEF)),
            List.of(), List.of(), now, null, now);
    when(store.lookup("agent1", "user1", "tenant1")).thenReturn(Optional.of(snapshot));

    var tick = orchestrator.tick("agent1", "user1", "tenant1");
    assertThat(tick).isInstanceOf(MentalModelTick.Updated.class);
    var loaded = ((MentalModelTick.Updated) tick).snapshot();
    assertThat(loaded.beliefs()).hasSize(1);
}
```

- [ ] **Step 10: Write test for RelationshipCue signal processing**

```java
@Test
void relationshipCueAccumulatesSignal() {
    var event = new RelationshipEvent("agent1", "user1", "tenant1",
            "case1", "turn1", "conversation",
            QualitySignal.POSITIVE, "user expressed trust", 0.8, Map.of());
    orchestrator.record(new MentalStateSignal.RelationshipCue(event),
            "agent1", "user1", "tenant1");
    var tick = orchestrator.tick("agent1", "user1", "tenant1");
    assertThat(tick).isNotInstanceOf(MentalModelTick.Unchanged.class);
}
```

- [ ] **Step 11: Run all tests to verify they fail**

Run: `mvn --batch-mode test -pl blocks -Dtest="MentalModelOrchestratorTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation errors — MentalModelOrchestrator doesn't exist.

- [ ] **Step 12: Implement MentalModelOrchestrator**

Follow UserModelOrchestrator structure:
- `@ApplicationScoped` with `@Inject` constructor taking `MentalModelStore`, `AgentProvider`, `MentalModelConfig`
- `ConcurrentHashMap<String, SubjectMentalState>` for per-subject state
- `ConcurrentHashMap<String, ReentrantLock>` for per-subject tick locks
- `SubjectMentalState` inner class with beliefs/desires/intentions as `Map<String, AttributedState>`, signal buffer as bounded `ArrayDeque<String>`, counters
- `record()` — heuristic extraction from VerbalCue (CueType-based), buffer append
- `tick()` — decay confidence, evict below floor, optionally invoke LLM, persist, evict stale in-memory state
- `project()` — iterate all BDI entries above projection floor, create MentalProjection per entry
- `observeConversation()` — extract beliefs from CommonGroundState: ESTABLISHED→0.9, DISPUTED→0.3 confidence
- LLM inference uses AgentProvider with merge semantics: present entries update, absent entries preserved
- BeliefSet used for AGM revision when ConsistencyChecker is non-trivial (construct temporary BeliefSet, revise, read back)

- [ ] **Step 13: Run all orchestrator tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="MentalModelOrchestratorTest"`
Expected: All tests PASS.

- [ ] **Step 14: Run full test suite to verify no regressions**

Run: `mvn --batch-mode test -pl blocks`
Expected: All tests PASS (including all prior social cognition tests).

- [ ] **Step 15: Commit**

```bash
git add blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelOrchestrator.java blocks/src/test/java/io/casehub/blocks/agentic/social/MentalModelOrchestratorTest.java
git commit -m "feat(#123): implement MentalModelOrchestrator — BDI tracking with confidence decay, GOAP projection, epistemic bridge"
```

---

## References

- [2026-08-20-mental-model-design.md] — design spec this plan implements
- [blocks/src/main/java/io/casehub/blocks/agentic/social/UserModelOrchestrator.java] — precedent orchestrator
- [blocks/src/main/java/io/casehub/blocks/agentic/social/CbrUserProfileStore.java] — precedent store adapter
- [blocks/src/main/java/io/casehub/blocks/agentic/belief/BeliefSet.java] — AGM belief revision
- [blocks/src/main/java/io/casehub/blocks/conversation/CommonGroundState.java] — epistemic common ground
- [blocks/src/main/java/io/casehub/blocks/conversation/EpistemicStatus.java] — belief classification
- [GitHub #123] — MentalModel pattern — Theory of Mind with BDI tracking
- [D23-D31] — validated design decisions
