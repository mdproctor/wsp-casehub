# StrategyLearning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #124 — StrategyLearning pattern — multi-level reflection on interaction strategies
**Issue group:** #118, #119, #120, #121, #122, #123, #124, #126

**Goal:** Build the fifth social cognition orchestrator — evaluates engagement outcomes, learns interaction strategies per-subject and globally, produces actionable guidelines for agent prompts.

**Architecture:** Single `StrategyLearningOrchestrator` with `record()` + `tick()` + `reflect()` + `currentStrategy()`. Composes ReflectionOrchestrator, ContentSummariser, CbrCaseMemoryStore, TrendAnalyzer, and AgentProvider. Three internal tiers: heuristic counters (tier 1), conversation case storage (tier 2), and full LLM reflection (tier 3). StrategyStore SPI backed by CbrCaseMemoryStore for profile persistence.

**Tech Stack:** Java 21, JUnit 5, Mockito, neocortex-memory-api (EngagementEvent, ReflectionOrchestrator, TrendAnalyzer, CbrCaseMemoryStore), platform-agent-api (AgentProvider), blocks summarisation (ContentSummariser)

## Global Constraints

- Package: `io.casehub.blocks.agentic.social`
- No CDI container in tests — plain JUnit 5 with Mockito
- All records use defensive copies: `Map.copyOf()`, `List.copyOf()`
- `@Nullable` from `org.jspecify.annotations` for nullable fields
- Placeholder solution `"-"` for FeatureVectorCbrCase (GE-20260820-d4e011)
- Post-filter by producerAgentId after retrieveSimilar() (GE-20260820-c19b68) OR use CbrFilter where available
- `Path.root()` for CBR scope (GE-20260805-4336aa)

---

## Batch 1: Foundation Types

### Task 1: EngagementSignal + StrategyProfile + Config + Tick/Reflection outcomes

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/EngagementSignal.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyProfile.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningConfig.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyReflection.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/EngagementSignalTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/StrategyProfileTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/StrategyLearningConfigTest.java`

**Interfaces:**
- Consumes: `EngagementEvent` (neocortex-memory-api), `TrendProfile` (neocortex-memory-api), `MemoryDomain` (neocortex-memory-api)
- Produces: `EngagementSignal` (sealed: TurnOutcome, ConversationOutcome), `StrategyProfile` (record with toPromptSection()), `StrategyLearningConfig` (record with defaults()), `StrategyLearningTick` (sealed: NoChange, Observed, Learned), `StrategyReflection` (sealed: NoChange, Reflected)

- [ ] **Step 1: Write EngagementSignal tests**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.engagement.EngagementEvent;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class EngagementSignalTest {

    @Test void turnOutcome_requiresNonNullEvent() {
        assertThatThrownBy(() -> new EngagementSignal.TurnOutcome(null, Map.of(), "excerpt"))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void turnOutcome_requiresNonNullSnapshot() {
        assertThatThrownBy(() -> new EngagementSignal.TurnOutcome(dummyEvent(), null, "excerpt"))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void turnOutcome_defensiveCopiesSnapshot() {
        var mutable = new java.util.HashMap<>(Map.of("verbosity", 0.7));
        var signal = new EngagementSignal.TurnOutcome(dummyEvent(), mutable, "excerpt");
        mutable.put("hacked", 1.0);
        assertThat(signal.dimensionalSnapshot()).doesNotContainKey("hacked");
    }

    @Test void turnOutcome_allowsNullExcerpt() {
        var signal = new EngagementSignal.TurnOutcome(dummyEvent(), Map.of(), null);
        assertThat(signal.responseExcerpt()).isNull();
    }

    @Test void conversationOutcome_requiresNonBlankConversationId() {
        assertThatThrownBy(() -> new EngagementSignal.ConversationOutcome("", "summary", 5))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void conversationOutcome_requiresNonBlankConversationId_blank() {
        assertThatThrownBy(() -> new EngagementSignal.ConversationOutcome("  ", "summary", 5))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void conversationOutcome_rejectsNegativeTurnCount() {
        assertThatThrownBy(() -> new EngagementSignal.ConversationOutcome("conv-1", "summary", -1))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void conversationOutcome_acceptsZeroTurnCount() {
        var outcome = new EngagementSignal.ConversationOutcome("conv-1", "summary", 0);
        assertThat(outcome.turnCount()).isZero();
    }

    private EngagementEvent dummyEvent() {
        return new EngagementEvent("agent-1", "user-1", "tenant-1", "case-1",
            "turn-1", "test description", null, Map.of(),
            true, null, null, null, null, true);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl blocks -Dtest=EngagementSignalTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: Compilation failure — `EngagementSignal` class does not exist

- [ ] **Step 3: Implement EngagementSignal**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/EngagementSignal.java`:

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.engagement.EngagementEvent;
import org.jspecify.annotations.Nullable;

import java.util.Map;
import java.util.Objects;

public sealed interface EngagementSignal {

    record TurnOutcome(
            EngagementEvent event,
            Map<String, Double> dimensionalSnapshot,
            @Nullable String responseExcerpt
    ) implements EngagementSignal {
        public TurnOutcome {
            Objects.requireNonNull(event, "event required");
            Objects.requireNonNull(dimensionalSnapshot, "dimensionalSnapshot required");
            dimensionalSnapshot = Map.copyOf(dimensionalSnapshot);
        }
    }

    record ConversationOutcome(
            String conversationId,
            @Nullable String conversationSummary,
            int turnCount
    ) implements EngagementSignal {
        public ConversationOutcome {
            Objects.requireNonNull(conversationId, "conversationId required");
            if (conversationId.isBlank())
                throw new IllegalArgumentException("conversationId must not be blank");
            if (turnCount < 0)
                throw new IllegalArgumentException("turnCount must be >= 0");
        }
    }
}
```

- [ ] **Step 4: Run EngagementSignal tests**

Run: `mvn test -pl blocks -Dtest=EngagementSignalTest --batch-mode`
Expected: All 8 tests PASS

- [ ] **Step 5: Write StrategyProfile tests**

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class StrategyProfileTest {

    @Test void requiresNonNullFields() {
        assertThatThrownBy(() -> new StrategyProfile(
            null, "t", Map.of(), List.of(), Instant.now(), 0))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void defensiveCopiesDimensions() {
        var dims = new HashMap<>(Map.of("verbosity", 0.5));
        var profile = new StrategyProfile("a", "t", dims, List.of(), Instant.now(), 0);
        dims.put("hacked", 1.0);
        assertThat(profile.dimensions()).doesNotContainKey("hacked");
    }

    @Test void defensiveCopiesGuidelines() {
        var list = new java.util.ArrayList<>(List.of("be concise"));
        var profile = new StrategyProfile("a", "t", Map.of(), list, Instant.now(), 0);
        list.add("hacked");
        assertThat(profile.guidelines()).hasSize(1);
    }

    @Test void toPromptSection_emptyGuidelines() {
        var profile = new StrategyProfile("a", "t", Map.of(), List.of(), Instant.now(), 0);
        assertThat(profile.toPromptSection()).isEmpty();
    }

    @Test void toPromptSection_formatsGuidelines() {
        var profile = new StrategyProfile("a", "t", Map.of(),
            List.of("Be concise", "Ask questions"), Instant.now(), 0);
        String section = profile.toPromptSection();
        assertThat(section).contains("## Interaction Strategy");
        assertThat(section).contains("- Be concise");
        assertThat(section).contains("- Ask questions");
    }
}
```

- [ ] **Step 6: Implement StrategyProfile**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyProfile.java`:

```java
package io.casehub.blocks.agentic.social;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public record StrategyProfile(
        String agentId,
        String tenantId,
        Map<String, Double> dimensions,
        List<String> guidelines,
        Instant lastReflection,
        int evidenceCount) {

    public StrategyProfile {
        Objects.requireNonNull(agentId, "agentId required");
        Objects.requireNonNull(tenantId, "tenantId required");
        Objects.requireNonNull(dimensions, "dimensions required");
        Objects.requireNonNull(guidelines, "guidelines required");
        Objects.requireNonNull(lastReflection, "lastReflection required");
        dimensions = Map.copyOf(dimensions);
        guidelines = List.copyOf(guidelines);
    }

    public String toPromptSection() {
        if (guidelines.isEmpty()) return "";
        var sb = new StringBuilder("## Interaction Strategy\n\n");
        for (String guideline : guidelines) {
            sb.append("- ").append(guideline).append('\n');
        }
        return sb.toString();
    }
}
```

- [ ] **Step 7: Run StrategyProfile tests**

Run: `mvn test -pl blocks -Dtest=StrategyProfileTest --batch-mode`
Expected: All 5 tests PASS

- [ ] **Step 8: Write StrategyLearningConfig tests**

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.assertj.core.api.Assertions.*;

class StrategyLearningConfigTest {

    @Test void defaults_createsValidConfig() {
        var config = StrategyLearningConfig.defaults();
        assertThat(config.minSignalsForConversationCase()).isEqualTo(3);
        assertThat(config.minCasesForReflection()).isEqualTo(5);
        assertThat(config.maxReflectionSources()).isEqualTo(50);
        assertThat(config.defaultDimensionValue()).isEqualTo(0.5);
        assertThat(config.memoryDomain().name()).isEqualTo("strategy-learning");
    }

    @Test void rejectsZeroMinSignals() {
        assertThatThrownBy(() -> new StrategyLearningConfig(
            0, 5, 50, 10, 0.5, 100, Duration.ofHours(24),
            new io.casehub.neocortex.memory.MemoryDomain("test"),
            "engagement", "profile"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void rejectsZeroMaxBuffer() {
        assertThatThrownBy(() -> new StrategyLearningConfig(
            3, 5, 50, 10, 0.5, 0, Duration.ofHours(24),
            new io.casehub.neocortex.memory.MemoryDomain("test"),
            "engagement", "profile"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void rejectsNegativeTimeout() {
        assertThatThrownBy(() -> new StrategyLearningConfig(
            3, 5, 50, 10, 0.5, 100, Duration.ofHours(-1),
            new io.casehub.neocortex.memory.MemoryDomain("test"),
            "engagement", "profile"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void rejectsOutOfRangeDefault() {
        assertThatThrownBy(() -> new StrategyLearningConfig(
            3, 5, 50, 10, 1.5, 100, Duration.ofHours(24),
            new io.casehub.neocortex.memory.MemoryDomain("test"),
            "engagement", "profile"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void rejectsZeroMaxGuidelines() {
        assertThatThrownBy(() -> new StrategyLearningConfig(
            3, 5, 50, 0, 0.5, 100, Duration.ofHours(24),
            new io.casehub.neocortex.memory.MemoryDomain("test"),
            "engagement", "profile"))
            .isInstanceOf(IllegalArgumentException.class);
    }

    @Test void rejectsZeroMaxReflectionSources() {
        assertThatThrownBy(() -> new StrategyLearningConfig(
            3, 5, 0, 10, 0.5, 100, Duration.ofHours(24),
            new io.casehub.neocortex.memory.MemoryDomain("test"),
            "engagement", "profile"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 9: Implement StrategyLearningConfig**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningConfig.java`:

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.MemoryDomain;

import java.time.Duration;
import java.util.Objects;

public record StrategyLearningConfig(
        int minSignalsForConversationCase,
        int minCasesForReflection,
        int maxReflectionSources,
        int maxGuidelines,
        double defaultDimensionValue,
        int maxBufferSize,
        Duration staleStateTimeout,
        MemoryDomain memoryDomain,
        String engagementCaseType,
        String profileCaseType) {

    public StrategyLearningConfig {
        if (minSignalsForConversationCase < 1)
            throw new IllegalArgumentException("minSignalsForConversationCase must be >= 1");
        if (minCasesForReflection < 1)
            throw new IllegalArgumentException("minCasesForReflection must be >= 1");
        if (maxReflectionSources < 1)
            throw new IllegalArgumentException("maxReflectionSources must be >= 1");
        if (maxGuidelines < 1)
            throw new IllegalArgumentException("maxGuidelines must be >= 1");
        if (maxBufferSize < 1)
            throw new IllegalArgumentException("maxBufferSize must be >= 1");
        if (defaultDimensionValue < 0.0 || defaultDimensionValue > 1.0)
            throw new IllegalArgumentException("defaultDimensionValue must be in [0,1]");
        if (staleStateTimeout.isNegative() || staleStateTimeout.isZero())
            throw new IllegalArgumentException("staleStateTimeout must be positive");
        Objects.requireNonNull(memoryDomain, "memoryDomain required");
        Objects.requireNonNull(engagementCaseType, "engagementCaseType required");
        Objects.requireNonNull(profileCaseType, "profileCaseType required");
    }

    public static StrategyLearningConfig defaults() {
        return new StrategyLearningConfig(
                3, 5, 50, 10, 0.5, 100, Duration.ofHours(24),
                new MemoryDomain("strategy-learning"),
                "engagement-evidence", "strategy-profile");
    }
}
```

- [ ] **Step 10: Implement StrategyLearningTick + StrategyReflection**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningTick.java`:

```java
package io.casehub.blocks.agentic.social;

import org.jspecify.annotations.Nullable;
import java.util.List;

public sealed interface StrategyLearningTick {

    record NoChange(@Nullable String reason) implements StrategyLearningTick {}

    record Observed(int signalsProcessed, double engagementRate,
                    double meanSentiment) implements StrategyLearningTick {}

    record Learned(int signalsProcessed, double engagementRate,
                   double meanSentiment, List<String> conversationsStored,
                   int casesStored) implements StrategyLearningTick {
        public Learned { conversationsStored = List.copyOf(conversationsStored); }
    }
}
```

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyReflection.java`:

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.cbr.TrendProfile;
import org.jspecify.annotations.Nullable;
import java.util.List;

public sealed interface StrategyReflection {

    record NoChange(@Nullable String reason) implements StrategyReflection {}

    record Reflected(StrategyProfile profile, List<String> newGuidelines,
                     TrendProfile trends, int evidenceCases)
            implements StrategyReflection {
        public Reflected { newGuidelines = List.copyOf(newGuidelines); }
    }
}
```

- [ ] **Step 11: Run all foundation tests**

Run: `mvn test -pl blocks -Dtest="EngagementSignalTest,StrategyProfileTest,StrategyLearningConfigTest" --batch-mode`
Expected: All tests PASS

- [ ] **Step 12: Commit**

```bash
git add blocks/src/main/java/io/casehub/blocks/agentic/social/EngagementSignal.java blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyProfile.java blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningConfig.java blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningTick.java blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyReflection.java blocks/src/test/java/io/casehub/blocks/agentic/social/EngagementSignalTest.java blocks/src/test/java/io/casehub/blocks/agentic/social/StrategyProfileTest.java blocks/src/test/java/io/casehub/blocks/agentic/social/StrategyLearningConfigTest.java
git commit -m "feat(#124): add StrategyLearning foundation types — EngagementSignal, StrategyProfile, config, tick/reflection outcomes Refs #124"
```

---

## Batch 2: Storage SPI

### Task 2: StrategyStore SPI + StrategyProfileSchema + CbrStrategyStore

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyStore.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyProfileSchema.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/CbrStrategyStore.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/CbrStrategyStoreTest.java`

**Interfaces:**
- Consumes: `StrategyProfile` (Task 1), `CbrCaseMemoryStore`, `FeatureVectorCbrCase`, `CbrQuery`, `FeatureValue`, `Path`, `EraseRequest`, `MemoryDomain`
- Produces: `StrategyStore` (SPI: store, lookup, subjectInsights, eraseAgent, eraseSubject), `CbrStrategyStore` (@DefaultBean impl)

- [ ] **Step 1: Write CbrStrategyStore tests**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.EraseRequest;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class CbrStrategyStoreTest {

    private CbrCaseMemoryStore cbrStore;
    private CbrStrategyStore store;

    @BeforeEach
    void setUp() {
        cbrStore = mock(CbrCaseMemoryStore.class);
        store = new CbrStrategyStore(cbrStore, StrategyLearningConfig.defaults());
    }

    @Test void store_writesProfileAsCbrCase() {
        var profile = new StrategyProfile("agent-1", "tenant-1",
            Map.of("verbosity", 0.7), List.of("Be concise"), Instant.now(), 5);

        store.store(profile);

        var captor = ArgumentCaptor.forClass(FeatureVectorCbrCase.class);
        verify(cbrStore).store(captor.capture(), eq("strategy-profile"),
            eq("agent-1"), any(MemoryDomain.class), eq("tenant-1"),
            isNull(), eq(Path.root()));

        var cbrCase = captor.getValue();
        assertThat(cbrCase.problem()).contains("agent-1");
        assertThat(cbrCase.solution()).isEqualTo("-");
        assertThat(cbrCase.producerAgentId()).isEqualTo("agent-1");
    }

    @Test void lookup_returnsEmpty_whenNoCases() {
        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of());
        assertThat(store.lookup("agent-1", "tenant-1")).isEmpty();
    }

    @Test void lookup_returnsProfile_whenCaseExists() {
        var features = Map.<String, FeatureValue>of(
            "agent_id", FeatureValue.string("agent-1"),
            "verbosity", FeatureValue.number(0.7),
            "formality", FeatureValue.number(0.5),
            "initiative", FeatureValue.number(0.5),
            "directness", FeatureValue.number(0.5),
            "questionRate", FeatureValue.number(0.5),
            "evidence_count", FeatureValue.number(5));
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.features()).thenReturn(features);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.problem()).thenReturn("guidelines: Be concise");
        var scored = new ScoredCbrCase<>(cbrCase, "case-1", 1.0, false,
            Map.of(), Instant.now(), Path.root(), null);

        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of(scored));

        Optional<StrategyProfile> result = store.lookup("agent-1", "tenant-1");
        assertThat(result).isPresent();
        assertThat(result.get().agentId()).isEqualTo("agent-1");
        assertThat(result.get().dimensions().get("verbosity")).isEqualTo(0.7);
    }

    @Test void eraseAgent_deletesAllCasesForAgent() {
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.features()).thenReturn(Map.of());
        var scored = new ScoredCbrCase<>(cbrCase, "case-1", 1.0);

        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of(scored));

        store.eraseAgent("agent-1", "tenant-1");

        verify(cbrStore).erase(any(EraseRequest.class));
    }

    @Test void eraseSubject_deletesEngagementCasesForSubject() {
        var features = Map.<String, FeatureValue>of(
            "subjectId", FeatureValue.string("user-X"));
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.features()).thenReturn(features);
        var scored = new ScoredCbrCase<>(cbrCase, "case-1", 1.0);

        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of(scored));

        store.eraseSubject("user-X", "tenant-1");

        verify(cbrStore).erase(any(EraseRequest.class));
    }

    @Test void subjectInsights_returnsFormattedInsights() {
        var features = Map.<String, FeatureValue>of(
            "subjectId", FeatureValue.string("user-X"),
            "continuationRate", FeatureValue.number(0.8),
            "avgResponseLength", FeatureValue.number(245.0),
            "meanSentimentShift", FeatureValue.number(0.15));
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.features()).thenReturn(features);
        var scored = new ScoredCbrCase<>(cbrCase, "case-1", 1.0);

        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of(scored));

        List<String> insights = store.subjectInsights("agent-1", "user-X", "tenant-1");
        assertThat(insights).isNotEmpty();
        assertThat(insights.get(0)).contains("user-X");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl blocks -Dtest=CbrStrategyStoreTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: Compilation failure — classes don't exist

- [ ] **Step 3: Implement StrategyStore SPI**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyStore.java`:

```java
package io.casehub.blocks.agentic.social;

import java.util.List;
import java.util.Optional;

public interface StrategyStore {

    void store(StrategyProfile profile);

    Optional<StrategyProfile> lookup(String agentId, String tenantId);

    List<String> subjectInsights(String agentId, String subjectId, String tenantId);

    void eraseAgent(String agentId, String tenantId);

    void eraseSubject(String subjectId, String tenantId);
}
```

- [ ] **Step 4: Implement StrategyProfileSchema**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyProfileSchema.java`:

Follow `UserProfileSchema` pattern exactly — package-private, static methods, feature key constants.

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;

import java.time.Instant;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

final class StrategyProfileSchema {

    static final String AGENT_ID = "agent_id";
    static final String VERBOSITY = "verbosity";
    static final String FORMALITY = "formality";
    static final String INITIATIVE = "initiative";
    static final String DIRECTNESS = "directness";
    static final String QUESTION_RATE = "questionRate";
    static final String EVIDENCE_COUNT = "evidence_count";
    static final String GUIDELINES_SEPARATOR = "\n";

    private StrategyProfileSchema() {}

    static Map<String, FeatureValue> toFeatures(StrategyProfile profile) {
        var features = new LinkedHashMap<String, FeatureValue>();
        features.put(AGENT_ID, FeatureValue.string(profile.agentId()));
        for (var entry : profile.dimensions().entrySet()) {
            features.put(entry.getKey(), FeatureValue.number(entry.getValue()));
        }
        features.put(EVIDENCE_COUNT, FeatureValue.number(profile.evidenceCount()));
        return Map.copyOf(features);
    }

    static String toSummary(StrategyProfile profile) {
        if (profile.guidelines().isEmpty()) {
            return "Strategy profile for " + profile.agentId() + " [no guidelines]";
        }
        return "guidelines: " + String.join(GUIDELINES_SEPARATOR, profile.guidelines());
    }

    static StrategyProfile fromCase(ScoredCbrCase<CbrCase> scored,
                                     String agentId, String tenantId) {
        var features = scored.cbrCase().features();
        var storedAt = scored.storedAt() != null ? scored.storedAt() : Instant.now();

        var dimensions = new LinkedHashMap<String, Double>();
        for (String dim : List.of(VERBOSITY, FORMALITY, INITIATIVE, DIRECTNESS, QUESTION_RATE)) {
            dimensions.put(dim, numberVal(features, dim, 0.5));
        }
        for (var entry : features.entrySet()) {
            if (!dimensions.containsKey(entry.getKey())
                    && entry.getValue() instanceof FeatureValue.NumberVal nv
                    && !AGENT_ID.equals(entry.getKey())
                    && !EVIDENCE_COUNT.equals(entry.getKey())) {
                dimensions.put(entry.getKey(), nv.value());
            }
        }

        var guidelines = new ArrayList<String>();
        String problem = scored.cbrCase().problem();
        if (problem != null && problem.startsWith("guidelines: ")) {
            String raw = problem.substring("guidelines: ".length());
            for (String line : raw.split(GUIDELINES_SEPARATOR)) {
                if (!line.isBlank()) guidelines.add(line.trim());
            }
        }

        return new StrategyProfile(agentId, tenantId, Map.copyOf(dimensions),
                List.copyOf(guidelines), storedAt,
                (int) numberVal(features, EVIDENCE_COUNT, 0));
    }

    private static double numberVal(Map<String, FeatureValue> features,
                                     String key, double defaultVal) {
        var val = features.get(key);
        if (val instanceof FeatureValue.NumberVal nv) return nv.value();
        return defaultVal;
    }
}
```

- [ ] **Step 5: Implement CbrStrategyStore**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/CbrStrategyStore.java`:

Follow `CbrUserProfileStore` pattern — @DefaultBean @ApplicationScoped, constructor takes CbrCaseMemoryStore + config.

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.EraseRequest;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.platform.api.path.Path;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class CbrStrategyStore implements StrategyStore {

    private final CbrCaseMemoryStore cbrStore;
    private final MemoryDomain domain;
    private final String profileCaseType;
    private final String engagementCaseType;

    CbrStrategyStore(CbrCaseMemoryStore cbrStore, StrategyLearningConfig config) {
        this.cbrStore = cbrStore;
        this.domain = config.memoryDomain();
        this.profileCaseType = config.profileCaseType();
        this.engagementCaseType = config.engagementCaseType();
    }

    @Override
    public void store(StrategyProfile profile) {
        var features = StrategyProfileSchema.toFeatures(profile);
        var summary = StrategyProfileSchema.toSummary(profile);
        var cbrCase = new FeatureVectorCbrCase(
                summary, "-", null, null, features, null, profile.agentId());
        cbrStore.store(cbrCase, profileCaseType, profile.agentId(), domain,
                profile.tenantId(), null, Path.root());
    }

    @Override
    public Optional<StrategyProfile> lookup(String agentId, String tenantId) {
        var query = CbrQuery.of(tenantId, domain, Path.root(), profileCaseType,
                        Map.of(StrategyProfileSchema.AGENT_ID,
                                FeatureValue.string(agentId)), 10)
                .withMinSimilarity(0.0);
        var results = cbrStore.retrieveSimilar(query, CbrCase.class);
        return results.stream()
                .filter(s -> agentId.equals(s.cbrCase().producerAgentId()))
                .findFirst()
                .map(s -> StrategyProfileSchema.fromCase(s, agentId, tenantId));
    }

    @Override
    public List<String> subjectInsights(String agentId, String subjectId,
                                         String tenantId) {
        var query = CbrQuery.of(tenantId, domain, Path.root(), engagementCaseType,
                        Map.of("subjectId", FeatureValue.string(subjectId)), 50)
                .withMinSimilarity(0.0);
        var results = cbrStore.retrieveSimilar(query, CbrCase.class);
        var insights = new ArrayList<String>();
        for (var scored : results) {
            if (!agentId.equals(scored.cbrCase().producerAgentId())) continue;
            var features = scored.cbrCase().features();
            var sv = features.get("subjectId");
            if (!(sv instanceof FeatureValue.StringVal s) || !subjectId.equals(s.value()))
                continue;

            double contRate = numberVal(features, "continuationRate", -1);
            double avgLen = numberVal(features, "avgResponseLength", -1);
            double sentiment = numberVal(features, "meanSentimentShift", 0);

            if (contRate >= 0 || avgLen >= 0) {
                insights.add(String.format(
                    "With %s: engagement rate %.0f%%, avg response length %.0f, sentiment %+.2f",
                    subjectId, contRate * 100, avgLen, sentiment));
            }
        }
        return List.copyOf(insights);
    }

    @Override
    public void eraseAgent(String agentId, String tenantId) {
        eraseCases(agentId, tenantId, profileCaseType);
        eraseCases(agentId, tenantId, engagementCaseType);
    }

    @Override
    public void eraseSubject(String subjectId, String tenantId) {
        var query = CbrQuery.of(tenantId, domain, Path.root(), engagementCaseType,
                        Map.of("subjectId", FeatureValue.string(subjectId)), 100)
                .withMinSimilarity(0.0);
        var results = cbrStore.retrieveSimilar(query, CbrCase.class);
        for (var scored : results) {
            var sv = scored.cbrCase().features().get("subjectId");
            if (sv instanceof FeatureValue.StringVal s && subjectId.equals(s.value())) {
                cbrStore.erase(new EraseRequest(
                        scored.cbrCase().producerAgentId() != null
                                ? scored.cbrCase().producerAgentId() : "unknown",
                        domain, tenantId, scored.caseId()));
            }
        }
    }

    private void eraseCases(String agentId, String tenantId, String caseType) {
        var query = CbrQuery.of(tenantId, domain, Path.root(), caseType,
                        Map.of(), 100)
                .withMinSimilarity(0.0);
        var results = cbrStore.retrieveSimilar(query, CbrCase.class);
        for (var scored : results) {
            if (agentId.equals(scored.cbrCase().producerAgentId())) {
                cbrStore.erase(new EraseRequest(agentId, domain, tenantId,
                        scored.caseId()));
            }
        }
    }

    private static double numberVal(Map<String, FeatureValue> features,
                                     String key, double defaultVal) {
        var val = features.get(key);
        if (val instanceof FeatureValue.NumberVal nv) return nv.value();
        return defaultVal;
    }
}
```

- [ ] **Step 6: Run CbrStrategyStore tests**

Run: `mvn test -pl blocks -Dtest=CbrStrategyStoreTest --batch-mode`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyStore.java blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyProfileSchema.java blocks/src/main/java/io/casehub/blocks/agentic/social/CbrStrategyStore.java blocks/src/test/java/io/casehub/blocks/agentic/social/CbrStrategyStoreTest.java
git commit -m "feat(#124): add StrategyStore SPI + CbrStrategyStore — strategy profile persistence via CbrCaseMemoryStore Refs #124"
```

---

## Batch 3: Orchestrator

### Task 3: StrategyLearningOrchestrator — record() + tick() + reflect() + currentStrategy()

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningOrchestrator.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/StrategyLearningOrchestratorTest.java`

**Interfaces:**
- Consumes: `EngagementSignal` (Task 1), `StrategyProfile` (Task 1), `StrategyLearningConfig` (Task 1), `StrategyLearningTick` (Task 1), `StrategyReflection` (Task 1), `StrategyStore` (Task 2), `CbrCaseMemoryStore`, `ReflectionOrchestrator`, `AgentProvider`, `ContentSummariser<EngagementSignal>`, `TrendAnalyzer`, `TrendProfile`, `FeatureVectorCbrCase`, `FeatureValue`, `CbrQuery`, `Path`, `MemoryDomain`, `CbrFilter`, `AgentSessionConfig`, `AgentEvent`, `Clock`
- Produces: `StrategyLearningOrchestrator` (composition root with record/tick/reflect/currentStrategy)

- [ ] **Step 1: Write orchestrator tests — record and basic tick**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.TrendProfile;
import io.casehub.neocortex.memory.engagement.EngagementEvent;
import io.casehub.neocortex.memory.reflection.ReflectionOrchestrator;
import io.casehub.platform.agent.api.AgentProvider;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Clock;
import java.time.Instant;
import java.time.ZoneId;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

class StrategyLearningOrchestratorTest {

    private StrategyStore strategyStore;
    private CbrCaseMemoryStore cbrStore;
    private ReflectionOrchestrator reflectionOrchestrator;
    private AgentProvider agentProvider;
    private StrategyLearningOrchestrator orchestrator;
    private Clock clock;

    @BeforeEach
    void setUp() {
        strategyStore = mock(StrategyStore.class);
        cbrStore = mock(CbrCaseMemoryStore.class);
        reflectionOrchestrator = mock(ReflectionOrchestrator.class);
        agentProvider = mock(AgentProvider.class);
        clock = Clock.fixed(Instant.parse("2026-08-21T00:00:00Z"), ZoneId.of("UTC"));
        orchestrator = new StrategyLearningOrchestrator(
            strategyStore, cbrStore, reflectionOrchestrator, agentProvider,
            null, StrategyLearningConfig.defaults(), clock);
    }

    @Test void tick_noSignals_returnsNoChange() {
        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(StrategyLearningTick.NoChange.class);
    }

    @Test void tick_withTurnOutcomes_returnsObserved() {
        orchestrator.record(turnOutcome(true, 0.5), "agent-1", "user-1", "tenant-1");
        orchestrator.record(turnOutcome(true, -0.2), "agent-1", "user-1", "tenant-1");
        orchestrator.record(turnOutcome(false, 0.0), "agent-1", "user-1", "tenant-1");

        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(StrategyLearningTick.Observed.class);
        var observed = (StrategyLearningTick.Observed) result;
        assertThat(observed.signalsProcessed()).isEqualTo(3);
        assertThat(observed.engagementRate()).isCloseTo(0.667, within(0.01));
        assertThat(observed.meanSentiment()).isCloseTo(0.1, within(0.01));
    }

    @Test void tick_withConversationOutcome_returnsLearned() {
        orchestrator.record(turnOutcome(true, 0.3), "agent-1", "user-1", "tenant-1");
        orchestrator.record(turnOutcome(true, 0.5), "agent-1", "user-1", "tenant-1");
        orchestrator.record(turnOutcome(true, 0.1), "agent-1", "user-1", "tenant-1");
        orchestrator.record(
            new EngagementSignal.ConversationOutcome("case-1", "summary", 3),
            "agent-1", "user-1", "tenant-1");

        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(StrategyLearningTick.Learned.class);
        var learned = (StrategyLearningTick.Learned) result;
        assertThat(learned.signalsProcessed()).isEqualTo(3);
        assertThat(learned.casesStored()).isEqualTo(1);
        verify(cbrStore).store(any(), eq("engagement-evidence"),
            eq("agent-1"), any(), eq("tenant-1"), isNull(), any());
    }

    @Test void tick_conversationCorrelation_matchesByCaseId() {
        var event1 = engagementEvent("case-1", true, 0.3);
        var event2 = engagementEvent("case-2", true, 0.5);
        orchestrator.record(new EngagementSignal.TurnOutcome(event1, Map.of(), null),
            "agent-1", "user-1", "tenant-1");
        orchestrator.record(new EngagementSignal.TurnOutcome(event2, Map.of(), null),
            "agent-1", "user-1", "tenant-1");

        orchestrator.record(
            new EngagementSignal.ConversationOutcome("case-1", "summary", 1),
            "agent-1", "user-1", "tenant-1");

        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(StrategyLearningTick.Learned.class);
        assertThat(((StrategyLearningTick.Learned) result).casesStored()).isEqualTo(1);
    }

    @Test void currentStrategy_returnsEmpty_whenNoProfile() {
        when(strategyStore.lookup("agent-1", "tenant-1")).thenReturn(java.util.Optional.empty());
        assertThat(orchestrator.currentStrategy("agent-1", "tenant-1")).isEmpty();
    }

    @Test void reflect_insufficientCases_returnsNoChange() {
        when(cbrStore.retrieveSimilar(any(), any())).thenReturn(List.of());
        var result = orchestrator.reflect("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(StrategyReflection.NoChange.class);
    }

    private EngagementSignal.TurnOutcome turnOutcome(boolean responded, double sentiment) {
        return new EngagementSignal.TurnOutcome(
            engagementEvent("case-1", responded, sentiment),
            Map.of("verbosity", 0.5, "formality", 0.5),
            "excerpt");
    }

    private EngagementEvent engagementEvent(String caseId, boolean responded, double sentiment) {
        return new EngagementEvent("agent-1", "user-1", "tenant-1", caseId,
            "turn-1", "test description", null, Map.of(),
            responded, null, 100, sentiment, null, responded);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl blocks -Dtest=StrategyLearningOrchestratorTest -Dsurefire.failIfNoSpecifiedTests=false --batch-mode`
Expected: Compilation failure — `StrategyLearningOrchestrator` does not exist

- [ ] **Step 3: Implement StrategyLearningOrchestrator**

Create `blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningOrchestrator.java`.

This is the largest file (~350 lines). Key implementation points:
- `ConcurrentHashMap<String, AgentLearningState>` for per-agent state
- `ReentrantLock` per agent for tick/reflect concurrency
- `record()` is O(1) — appends to bounded deque
- `tick()` runs tier 1 (drain → counters) then tier 2 (conversation cases)
- `reflect()` runs tier 3 (CBR query → TrendAnalyzer → ReflectionOrchestrator → LLM synthesis → profile update)
- `currentStrategy()` returns in-memory profile or queries store
- Package-private constructor takes `Clock` for testability
- LLM parse failure returns `StrategyReflection.NoChange` (UserModel precedent)

Implementation follows the exact patterns from `UserModelOrchestrator` and `MentalModelOrchestrator` — record/tick/reflect lifecycle, ConcurrentHashMap state, ReentrantLock concurrency, JSON parse with graceful degradation.

- [ ] **Step 4: Run orchestrator tests**

Run: `mvn test -pl blocks -Dtest=StrategyLearningOrchestratorTest --batch-mode`
Expected: All tests PASS

- [ ] **Step 5: Add reflect() LLM synthesis test**

Add to `StrategyLearningOrchestratorTest`:

```java
@Test void reflect_withSufficientCases_producesReflected() {
    // Set up 5 CBR cases (minCasesForReflection = 5)
    var cases = new java.util.ArrayList<ScoredCbrCase<CbrCase>>();
    for (int i = 0; i < 5; i++) {
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.features()).thenReturn(Map.of(
            "agentId", FeatureValue.string("agent-1"),
            "conversationTimestamp", FeatureValue.number((double) (1000 + i * 100)),
            "continuationRate", FeatureValue.number(0.7 + i * 0.02),
            "avgResponseLength", FeatureValue.number(200.0 + i * 10),
            "meanSentimentShift", FeatureValue.number(0.1 + i * 0.05),
            "avgSnapshot_verbosity", FeatureValue.number(0.5),
            "avgSnapshot_formality", FeatureValue.number(0.5)));
        cases.add(new ScoredCbrCase<>(cbrCase, "case-" + i, 1.0));
    }
    when(cbrStore.retrieveSimilar(any(), any())).thenReturn(cases);
    when(reflectionOrchestrator.reflect(any(), any(), any(), anyInt()))
        .thenReturn(List.of("Agent tends to be verbose"));

    // Mock AgentProvider to return valid JSON
    var textDelta = mock(AgentEvent.TextDelta.class);
    when(textDelta.text()).thenReturn(
        "{\"guidelines\":[\"Be more concise\"],\"dimensionDeltas\":{\"verbosity\":-0.1}}");
    when(agentProvider.invoke(any()))
        .thenReturn(io.smallrye.mutiny.Multi.createFrom().item(textDelta));

    var result = orchestrator.reflect("agent-1", "tenant-1");
    assertThat(result).isInstanceOf(StrategyReflection.Reflected.class);
    var reflected = (StrategyReflection.Reflected) result;
    assertThat(reflected.newGuidelines()).contains("Be more concise");
    assertThat(reflected.profile().dimensions().get("verbosity")).isLessThan(0.5);
    verify(strategyStore).store(any(StrategyProfile.class));
}

@Test void reflect_malformedLlmOutput_returnsNoChange() {
    // 5 cases to pass threshold
    var cases = new java.util.ArrayList<ScoredCbrCase<CbrCase>>();
    for (int i = 0; i < 5; i++) {
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.features()).thenReturn(Map.of(
            "agentId", FeatureValue.string("agent-1"),
            "conversationTimestamp", FeatureValue.number((double) (1000 + i))));
        cases.add(new ScoredCbrCase<>(cbrCase, "case-" + i, 1.0));
    }
    when(cbrStore.retrieveSimilar(any(), any())).thenReturn(cases);
    when(reflectionOrchestrator.reflect(any(), any(), any(), anyInt()))
        .thenReturn(List.of());

    var textDelta = mock(AgentEvent.TextDelta.class);
    when(textDelta.text()).thenReturn("not valid json at all");
    when(agentProvider.invoke(any()))
        .thenReturn(io.smallrye.mutiny.Multi.createFrom().item(textDelta));

    var result = orchestrator.reflect("agent-1", "tenant-1");
    assertThat(result).isInstanceOf(StrategyReflection.NoChange.class);
    verify(strategyStore, never()).store(any());
}

@Test void reflect_clampsDeltas() {
    // Set up profile with verbosity at 0.95
    var profile = new StrategyProfile("agent-1", "tenant-1",
        Map.of("verbosity", 0.95), List.of(), Instant.now(), 0);
    when(strategyStore.lookup("agent-1", "tenant-1"))
        .thenReturn(java.util.Optional.of(profile));

    var cases = new java.util.ArrayList<ScoredCbrCase<CbrCase>>();
    for (int i = 0; i < 5; i++) {
        var cbrCase = mock(CbrCase.class);
        when(cbrCase.producerAgentId()).thenReturn("agent-1");
        when(cbrCase.features()).thenReturn(Map.of(
            "agentId", FeatureValue.string("agent-1"),
            "conversationTimestamp", FeatureValue.number((double) (1000 + i))));
        cases.add(new ScoredCbrCase<>(cbrCase, "case-" + i, 1.0));
    }
    when(cbrStore.retrieveSimilar(any(), any())).thenReturn(cases);
    when(reflectionOrchestrator.reflect(any(), any(), any(), anyInt()))
        .thenReturn(List.of());

    // LLM suggests +0.2 which would push verbosity to 1.15
    var textDelta = mock(AgentEvent.TextDelta.class);
    when(textDelta.text()).thenReturn(
        "{\"guidelines\":[\"Test\"],\"dimensionDeltas\":{\"verbosity\":0.2}}");
    when(agentProvider.invoke(any()))
        .thenReturn(io.smallrye.mutiny.Multi.createFrom().item(textDelta));

    var result = orchestrator.reflect("agent-1", "tenant-1");
    assertThat(result).isInstanceOf(StrategyReflection.Reflected.class);
    var reflected = (StrategyReflection.Reflected) result;
    assertThat(reflected.profile().dimensions().get("verbosity")).isEqualTo(1.0);
}
```

- [ ] **Step 6: Run all orchestrator tests**

Run: `mvn test -pl blocks -Dtest=StrategyLearningOrchestratorTest --batch-mode`
Expected: All tests PASS

- [ ] **Step 7: Run full test suite**

Run: `mvn test -pl blocks --batch-mode`
Expected: All tests PASS (existing + new)

- [ ] **Step 8: Commit**

```bash
git add blocks/src/main/java/io/casehub/blocks/agentic/social/StrategyLearningOrchestrator.java blocks/src/test/java/io/casehub/blocks/agentic/social/StrategyLearningOrchestratorTest.java
git commit -m "feat(#124): implement StrategyLearningOrchestrator — multi-level reflection on interaction strategies Refs #124"
```

---

## References

- `work/specs/issue-126-autonomous-agent-patterns/2026-08-21-strategy-learning-design.md` — design spec
- `blocks/src/main/java/io/casehub/blocks/agentic/social/UserModelOrchestrator.java` — tiered pattern reference
- `blocks/src/main/java/io/casehub/blocks/agentic/social/CbrUserProfileStore.java` — CBR adapter pattern reference
- `blocks/src/main/java/io/casehub/blocks/agentic/social/UserProfileSchema.java` — schema helper reference
- `blocks/src/main/java/io/casehub/blocks/agentic/social/MentalModelOrchestrator.java` — Clock injection reference
- GE-20260820-c19b68 (producerAgentId post-filtering)
- GE-20260820-d4e011 (non-blank solution placeholder)
- GE-20260805-4336aa (Path scope matching)
- Decisions D32–D41
- GitHub #124
