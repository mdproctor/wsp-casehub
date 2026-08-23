# PersonalityEvolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #118 — PersonalityEvolution pattern — interaction outcomes drive bounded trait drift
**Issue group:** #118, #126

**Goal:** Build a signal-driven orchestrator that translates interaction outcomes into bounded personality drift via eidos's existing JPAF pipeline.

**Architecture:** PersonalityEvolution composes eidos's JPAF pipeline (DispositionSignalStore → DispositionHealth.probe() → DispositionEvolution.evaluate()) by adding a signal translation SPI, periodic orchestration with decay, and safety rails (L2 ceiling, asymmetric dampening). Eidos-api gains valence-aware signal storage; blocks gains the orchestrator and default pressure source implementations.

**Tech Stack:** Java 21, JUnit 5, Mockito, CDI (jakarta.inject/enterprise), eidos-api, neocortex-memory-api

## Global Constraints

- Blocks has no Quarkus runtime — all tests are plain JUnit 5 + Mockito
- No CDI container in tests — construct beans manually
- Eidos-api changes are a prerequisite — install eidos-api before compiling blocks
- New package: `io.casehub.blocks.agentic.personality`
- Eidos-api new types go in: `io.casehub.eidos.api`
- Follow existing sealed interface patterns (see `EvolutionResult`, `DispositionStatus`, `ExecutionResult`)
- Follow existing `@FunctionalInterface` SPI patterns for single-method interfaces
- All records are immutable — `List.copyOf()` / `Map.copyOf()` in compact constructors

---

## Batch 1: Eidos-api prerequisite — valence-aware signal storage

### Task 1: Add SignalValence, ValenceCounts, DispositionProfileStore to eidos-api and update DispositionSignalStore

**Files:**
- Create: `eidos/api/src/main/java/io/casehub/eidos/api/SignalValence.java`
- Create: `eidos/api/src/main/java/io/casehub/eidos/api/ValenceCounts.java`
- Create: `eidos/api/src/main/java/io/casehub/eidos/api/DispositionProfileStore.java`
- Modify: `eidos/api/src/main/java/io/casehub/eidos/api/DispositionSignalStore.java`
- Test: `eidos/api/src/test/java/io/casehub/eidos/api/ValenceCountsTest.java`
- Test: `eidos/api/src/test/java/io/casehub/eidos/api/DispositionSignalStoreDefaultsTest.java`

**Interfaces:**
- Produces: `SignalValence { POSITIVE, NEGATIVE }` — used by blocks TraitActivation and signal recording
- Produces: `ValenceCounts(int positive, int negative)` with `effective(double dampeningFactor)` — used by probe
- Produces: `DispositionProfileStore.update(String agentId, String tenancyId, List<DispositionValue> newProfile)` — used by orchestrator
- Produces: `DispositionSignalStore.recordActivation(agentId, tenancyId, functionTerm, SignalValence)` — default method
- Produces: `DispositionSignalStore.valenceCounts(agentId, tenancyId)` → `Map<String, ValenceCounts>` — default method

- [ ] **Step 1: Write ValenceCounts test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class ValenceCountsTest {

    @Test
    void effectiveWithFullDampening() {
        var vc = new ValenceCounts(5, 3);
        assertThat(vc.effective(1.0)).isEqualTo(8);
    }

    @Test
    void effectiveWithHalfDampening() {
        var vc = new ValenceCounts(5, 3);
        assertThat(vc.effective(0.5)).isEqualTo(7); // 5 + round(3 * 0.5) = 5 + 2
    }

    @Test
    void effectiveWithZeroDampening() {
        var vc = new ValenceCounts(5, 3);
        assertThat(vc.effective(0.0)).isEqualTo(5);
    }

    @Test
    void effectiveWithZeroCounts() {
        var vc = new ValenceCounts(0, 0);
        assertThat(vc.effective(0.5)).isEqualTo(0);
    }

    @Test
    void effectiveWithOnlyNegative() {
        var vc = new ValenceCounts(0, 4);
        assertThat(vc.effective(0.5)).isEqualTo(2); // 0 + round(4 * 0.5)
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl api -Dtest=ValenceCountsTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Create SignalValence enum**

Create `eidos/api/src/main/java/io/casehub/eidos/api/SignalValence.java`:

```java
package io.casehub.eidos.api;

public enum SignalValence { POSITIVE, NEGATIVE }
```

- [ ] **Step 4: Create ValenceCounts record**

Create `eidos/api/src/main/java/io/casehub/eidos/api/ValenceCounts.java`:

```java
package io.casehub.eidos.api;

public record ValenceCounts(int positive, int negative) {
    public int effective(double dampeningFactor) {
        return positive + (int) Math.round(negative * dampeningFactor);
    }
}
```

- [ ] **Step 5: Run ValenceCounts test to verify it passes**

Run: `mvn --batch-mode test -pl api -Dtest=ValenceCountsTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 6: Write DispositionSignalStore defaults test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicReference;
import static org.assertj.core.api.Assertions.*;

class DispositionSignalStoreDefaultsTest {

    @Test
    void valenceAwareRecordDelegatesToOriginal() {
        var recorded = new AtomicReference<String>();
        DispositionSignalStore store = new DispositionSignalStore() {
            @Override public void recordActivation(String agentId, String tenancyId, String functionTerm) {
                recorded.set(functionTerm);
            }
            @Override public Map<String, Integer> activationCounts(String a, String t) { return Map.of(); }
            @Override public void decay(String a, String t, double f) {}
            @Override public void clear(String a, String t) {}
        };

        store.recordActivation("agent1", "tenant1", "ti", SignalValence.NEGATIVE);
        assertThat(recorded.get()).isEqualTo("ti");
    }

    @Test
    void valenceCountsWrapsExistingCountsAsPositive() {
        DispositionSignalStore store = new DispositionSignalStore() {
            @Override public void recordActivation(String a, String t, String f) {}
            @Override public Map<String, Integer> activationCounts(String a, String t) {
                var m = new LinkedHashMap<String, Integer>();
                m.put("ti", 5);
                m.put("ne", 3);
                return m;
            }
            @Override public void decay(String a, String t, double f) {}
            @Override public void clear(String a, String t) {}
        };

        var counts = store.valenceCounts("a", "t");
        assertThat(counts.get("ti")).isEqualTo(new ValenceCounts(5, 0));
        assertThat(counts.get("ne")).isEqualTo(new ValenceCounts(3, 0));
    }
}
```

- [ ] **Step 7: Update DispositionSignalStore with default methods**

Add to `DispositionSignalStore.java` (after existing methods):

```java
default void recordActivation(String agentId, String tenancyId,
                               String functionTerm, SignalValence valence) {
    recordActivation(agentId, tenancyId, functionTerm);
}

default Map<String, ValenceCounts> valenceCounts(String agentId, String tenancyId) {
    var counts = activationCounts(agentId, tenancyId);
    var result = new java.util.LinkedHashMap<String, ValenceCounts>();
    counts.forEach((k, v) -> result.put(k, new ValenceCounts(v, 0)));
    return result;
}
```

- [ ] **Step 8: Create DispositionProfileStore**

Create `eidos/api/src/main/java/io/casehub/eidos/api/DispositionProfileStore.java`:

```java
package io.casehub.eidos.api;

import java.util.List;

@FunctionalInterface
public interface DispositionProfileStore {
    void update(String agentId, String tenancyId, List<DispositionValue> newProfile);
}
```

- [ ] **Step 9: Run DispositionSignalStore defaults test**

Run: `mvn --batch-mode test -pl api -Dtest=DispositionSignalStoreDefaultsTest -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: PASS

- [ ] **Step 10: Install eidos-api to local Maven repo**

Run: `mvn --batch-mode install -pl api -DskipTests -f /Users/mdproctor/claude/casehub/eidos/pom.xml`

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/SignalValence.java api/src/main/java/io/casehub/eidos/api/ValenceCounts.java api/src/main/java/io/casehub/eidos/api/DispositionProfileStore.java api/src/main/java/io/casehub/eidos/api/DispositionSignalStore.java api/src/test/java/io/casehub/eidos/api/ValenceCountsTest.java api/src/test/java/io/casehub/eidos/api/DispositionSignalStoreDefaultsTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#118): add valence-aware signal storage — SignalValence, ValenceCounts, DispositionProfileStore, DispositionSignalStore defaults Refs casehubio/blocks#118"
```

---

## Batch 2: Blocks foundation types — SPI, records, sealed interface

### Task 2: Create TraitPressureSource SPI, TraitActivation, EvolutionTick, and PersonalityEvolutionConfig

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/TraitPressureSource.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/TraitActivation.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/EvolutionTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/PersonalityEvolutionConfig.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/EvolutionTickTest.java`

**Interfaces:**
- Consumes: `SignalValence` (eidos-api, Task 1)
- Consumes: `DispositionValue` (eidos-api, existing)
- Produces: `TraitPressureSource<E>` — used by orchestrator (Task 3)
- Produces: `TraitActivation(String functionTerm, SignalValence valence)` — returned by pressure sources
- Produces: `EvolutionTick` sealed interface (Stable, Drifting, Halted, Evolved, Dampened) — returned by orchestrator
- Produces: `PersonalityEvolutionConfig(double decayFactor, double l2Ceiling, double dampeningFactor)` — consumed by orchestrator

- [ ] **Step 1: Write EvolutionTick test**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.DispositionValue;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class EvolutionTickTest {

    @Test
    void stableIsSealed() {
        EvolutionTick tick = new EvolutionTick.Stable();
        assertThat(tick).isInstanceOf(EvolutionTick.class);
    }

    @Test
    void driftingCarriesMagnitude() {
        var tick = new EvolutionTick.Drifting(0.08);
        assertThat(tick.magnitude()).isEqualTo(0.08);
    }

    @Test
    void haltedCarriesMagnitude() {
        var tick = new EvolutionTick.Halted(0.20);
        assertThat(tick.magnitude()).isEqualTo(0.20);
    }

    @Test
    void evolvedCarriesTypeLabelsAndProfile() {
        var profile = List.of(new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20));
        var tick = new EvolutionTick.Evolved("TI-NE", "NE-TI", profile);
        assertThat(tick.previousTypeLabel()).isEqualTo("TI-NE");
        assertThat(tick.newTypeLabel()).isEqualTo("NE-TI");
        assertThat(tick.newProfile()).hasSize(2);
    }

    @Test
    void dampenedCarriesDecayFactor() {
        var tick = new EvolutionTick.Dampened(0.2);
        assertThat(tick.decayFactor()).isEqualTo(0.2);
    }

    @Test
    void exhaustiveSwitchCoversAllVariants() {
        List<EvolutionTick> ticks = List.of(
            new EvolutionTick.Stable(),
            new EvolutionTick.Drifting(0.05),
            new EvolutionTick.Halted(0.20),
            new EvolutionTick.Evolved("A", "B", List.of()),
            new EvolutionTick.Dampened(0.2)
        );
        for (EvolutionTick t : ticks) {
            String label = switch (t) {
                case EvolutionTick.Stable s -> "stable";
                case EvolutionTick.Drifting d -> "drifting";
                case EvolutionTick.Halted h -> "halted";
                case EvolutionTick.Evolved e -> "evolved";
                case EvolutionTick.Dampened d -> "dampened";
            };
            assertThat(label).isNotBlank();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=EvolutionTickTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Create TraitActivation record**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/TraitActivation.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.SignalValence;

public record TraitActivation(String functionTerm, SignalValence valence) {
    public TraitActivation {
        if (functionTerm == null || functionTerm.isBlank()) {
            throw new IllegalArgumentException("functionTerm must not be blank");
        }
        if (valence == null) {
            throw new IllegalArgumentException("valence must not be null");
        }
    }
}
```

- [ ] **Step 4: Create TraitPressureSource interface**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/TraitPressureSource.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import java.util.List;

public interface TraitPressureSource<E> {
    Class<E> eventType();
    List<TraitActivation> translate(E event, AgentDescriptor descriptor);
}
```

- [ ] **Step 5: Create EvolutionTick sealed interface**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/EvolutionTick.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.DispositionValue;
import java.util.List;

public sealed interface EvolutionTick {
    record Stable() implements EvolutionTick {}
    record Drifting(double magnitude) implements EvolutionTick {}
    record Halted(double magnitude) implements EvolutionTick {}
    record Evolved(String previousTypeLabel, String newTypeLabel,
                   List<DispositionValue> newProfile) implements EvolutionTick {
        public Evolved {
            newProfile = List.copyOf(newProfile);
        }
    }
    record Dampened(double decayFactor) implements EvolutionTick {}
}
```

- [ ] **Step 6: Create PersonalityEvolutionConfig**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/PersonalityEvolutionConfig.java`:

```java
package io.casehub.blocks.agentic.personality;

public record PersonalityEvolutionConfig(double decayFactor, double l2Ceiling, double dampeningFactor) {

    public static final double DEFAULT_DECAY_FACTOR = 0.8;
    public static final double DEFAULT_L2_CEILING = 0.15;
    public static final double DEFAULT_DAMPENING_FACTOR = 0.5;

    public PersonalityEvolutionConfig {
        if (decayFactor < 0.0 || decayFactor > 1.0) {
            throw new IllegalArgumentException("decayFactor must be in [0.0, 1.0]");
        }
        if (l2Ceiling <= 0.0) {
            throw new IllegalArgumentException("l2Ceiling must be positive");
        }
        if (dampeningFactor < 0.0 || dampeningFactor > 1.0) {
            throw new IllegalArgumentException("dampeningFactor must be in [0.0, 1.0]");
        }
    }

    public static PersonalityEvolutionConfig defaults() {
        return new PersonalityEvolutionConfig(DEFAULT_DECAY_FACTOR, DEFAULT_L2_CEILING, DEFAULT_DAMPENING_FACTOR);
    }
}
```

- [ ] **Step 7: Run EvolutionTick test to verify it passes**

Run: `mvn --batch-mode test -pl blocks -Dtest=EvolutionTickTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/ blocks/src/test/java/io/casehub/blocks/agentic/personality/
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#118): add PersonalityEvolution foundation types — TraitPressureSource, TraitActivation, EvolutionTick, config Refs #118"
```

---

## Batch 3: Orchestrator — record, tick, halt flag, CBR recording

### Task 3: Implement PersonalityEvolutionOrchestrator

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/PersonalityEvolutionOrchestrator.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/PersonalityEvolutionOrchestratorTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/HaltFlagTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/TickConcurrencyTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/EventTypeDispatchTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/CbrTransitionRecordingTest.java`

**Interfaces:**
- Consumes: `TraitPressureSource<E>`, `TraitActivation`, `EvolutionTick`, `PersonalityEvolutionConfig` (Task 2)
- Consumes: `DispositionSignalStore`, `DispositionHealth`, `DispositionEvolution`, `DispositionProfileStore`, `SignalValence`, `ValenceCounts` (eidos-api, Task 1)
- Consumes: `CbrCaseMemoryStore`, `PersonalityTransitionSchema` (neocortex-memory-api, existing)
- Consumes: `AgentDescriptor`, `DispositionValue`, `CapabilityHealth.ProbeContext` (eidos-api, existing)
- Produces: `PersonalityEvolutionOrchestrator` CDI bean with `record(E event, AgentDescriptor descriptor)` and `tick(AgentDescriptor descriptor, ProbeContext probeContext)` → `EvolutionTick`

- [ ] **Step 1: Write core tick cycle test — Stable path**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.DispositionHealth.DispositionStatus;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class PersonalityEvolutionOrchestratorTest {

    private DispositionSignalStore signalStore;
    private DispositionHealth health;
    private DispositionEvolution evolution;
    private DispositionProfileStore profileStore;
    private CbrCaseMemoryStore cbrStore;
    private PersonalityEvolutionOrchestrator orchestrator;
    private AgentDescriptor descriptor;
    private ProbeContext probeContext;

    @BeforeEach
    void setUp() {
        signalStore = mock(DispositionSignalStore.class);
        health = mock(DispositionHealth.class);
        evolution = mock(DispositionEvolution.class);
        profileStore = mock(DispositionProfileStore.class);
        cbrStore = mock(CbrCaseMemoryStore.class);
        @SuppressWarnings("unchecked")
        Instance<TraitPressureSource<?>> sources = mock(Instance.class);
        when(sources.stream()).thenReturn(java.util.stream.Stream.empty());

        orchestrator = new PersonalityEvolutionOrchestrator(
            signalStore, health, evolution, profileStore, cbrStore, sources,
            PersonalityEvolutionConfig.defaults());

        descriptor = mock(AgentDescriptor.class);
        when(descriptor.agentId()).thenReturn("agent-1");
        when(descriptor.tenancyId()).thenReturn("tenant-1");
        var disposition = mock(AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));
        when(descriptor.disposition()).thenReturn(disposition);

        probeContext = mock(ProbeContext.class);
    }

    @Test
    void tickReturnsStableWhenAligned() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.Aligned(Map.of("ti", 0.35, "ne", 0.20)));

        var result = orchestrator.tick(descriptor, probeContext);

        assertThat(result).isInstanceOf(EvolutionTick.Stable.class);
        verify(signalStore).decay("agent-1", "tenant-1", 0.8);
    }

    @Test
    void tickReturnsDriftingWhenBelowCeiling() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.Drifted(Map.of("ti", 0.38), "ti", 0.10));

        var result = orchestrator.tick(descriptor, probeContext);

        assertThat(result).isInstanceOf(EvolutionTick.Drifting.class);
        assertThat(((EvolutionTick.Drifting) result).magnitude()).isEqualTo(0.10);
    }

    @Test
    void tickReturnsHaltedWhenAboveCeiling() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.Drifted(Map.of("ti", 0.55), "ti", 0.20));

        var result = orchestrator.tick(descriptor, probeContext);

        assertThat(result).isInstanceOf(EvolutionTick.Halted.class);
        assertThat(((EvolutionTick.Halted) result).magnitude()).isEqualTo(0.20);
    }

    @Test
    void tickReturnsEvolvedAndPersistsProfile() {
        var newProfile = List.of(new DispositionValue("ne", 0.35), new DispositionValue("ti", 0.20));
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.EvolutionPending(
                () -> "DOMINANT_AUXILIARY_SWAP", "ne", Map.of("ne", 0.36, "ti", 0.30)));
        when(evolution.evaluate(eq(descriptor), any()))
            .thenReturn(new DispositionEvolution.EvolutionResult.Evolved(newProfile, "TI-NE", "NE-TI"));

        var result = orchestrator.tick(descriptor, probeContext);

        assertThat(result).isInstanceOf(EvolutionTick.Evolved.class);
        var evolved = (EvolutionTick.Evolved) result;
        assertThat(evolved.previousTypeLabel()).isEqualTo("TI-NE");
        assertThat(evolved.newTypeLabel()).isEqualTo("NE-TI");
        verify(profileStore).update("agent-1", "tenant-1", newProfile);
        verify(signalStore).clear("agent-1", "tenant-1");
    }

    @Test
    void tickReturnsDampenedAndDecays() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.EvolutionPending(
                () -> "DOMINANT_AUXILIARY_SWAP", "ne", Map.of()));
        when(evolution.evaluate(eq(descriptor), any()))
            .thenReturn(new DispositionEvolution.EvolutionResult.Dampened(0.2));

        var result = orchestrator.tick(descriptor, probeContext);

        assertThat(result).isInstanceOf(EvolutionTick.Dampened.class);
        verify(signalStore).decay("agent-1", "tenant-1", 0.2);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=PersonalityEvolutionOrchestratorTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Implement PersonalityEvolutionOrchestrator**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/PersonalityEvolutionOrchestrator.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.DispositionHealth.DispositionStatus;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.MemoryDomain;
import io.casehub.neocortex.memory.cbr.PersonalityTransitionSchema;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

import java.util.Comparator;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.locks.ReentrantLock;

@ApplicationScoped
public class PersonalityEvolutionOrchestrator {

    private final DispositionSignalStore signalStore;
    private final DispositionHealth health;
    private final DispositionEvolution evolution;
    private final DispositionProfileStore profileStore;
    private final CbrCaseMemoryStore cbrStore;
    private final List<TraitPressureSource<?>> pressureSources;
    private final PersonalityEvolutionConfig config;

    private final ConcurrentHashMap<String, AtomicBoolean> haltFlags = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ReentrantLock> tickLocks = new ConcurrentHashMap<>();

    @Inject
    public PersonalityEvolutionOrchestrator(
            final DispositionSignalStore signalStore,
            final DispositionHealth health,
            final DispositionEvolution evolution,
            final DispositionProfileStore profileStore,
            final CbrCaseMemoryStore cbrStore,
            final Instance<TraitPressureSource<?>> pressureSources,
            final PersonalityEvolutionConfig config) {
        this.signalStore = signalStore;
        this.health = health;
        this.evolution = evolution;
        this.profileStore = profileStore;
        this.cbrStore = cbrStore;
        this.pressureSources = pressureSources.stream().toList();
        this.config = config;
    }

    @SuppressWarnings("unchecked")
    public <E> void record(final E event, final AgentDescriptor descriptor) {
        var agentKey = agentKey(descriptor);
        if (haltFlags.computeIfAbsent(agentKey, k -> new AtomicBoolean()).get()) {
            return;
        }
        var profile = descriptor.disposition().dispositionProfile();
        if (profile == null || profile.isEmpty()) {
            return;
        }
        var profileTerms = profile.stream().map(DispositionValue::term).toList();

        for (var source : pressureSources) {
            if (source.eventType().isInstance(event)) {
                var typedSource = (TraitPressureSource<E>) source;
                var activations = typedSource.translate(event, descriptor);
                if (activations != null) {
                    for (var activation : activations) {
                        if (profileTerms.contains(activation.functionTerm())) {
                            signalStore.recordActivation(
                                descriptor.agentId(), descriptor.tenancyId(),
                                activation.functionTerm(), activation.valence());
                        }
                    }
                }
                return;
            }
        }
    }

    public EvolutionTick tick(final AgentDescriptor descriptor, final ProbeContext probeContext) {
        var agentKey = agentKey(descriptor);
        var lock = tickLocks.computeIfAbsent(agentKey, k -> new ReentrantLock());
        lock.lock();
        try {
            return doTick(descriptor, probeContext, agentKey);
        } finally {
            lock.unlock();
        }
    }

    private EvolutionTick doTick(final AgentDescriptor descriptor,
                                  final ProbeContext probeContext,
                                  final String agentKey) {
        signalStore.decay(descriptor.agentId(), descriptor.tenancyId(), config.decayFactor());

        var status = health.probe(descriptor, probeContext);

        return switch (status) {
            case DispositionStatus.Aligned a -> {
                haltFlags.computeIfAbsent(agentKey, k -> new AtomicBoolean()).set(false);
                yield new EvolutionTick.Stable();
            }
            case DispositionStatus.Drifted d -> {
                if (d.driftMagnitude() >= config.l2Ceiling()) {
                    haltFlags.computeIfAbsent(agentKey, k -> new AtomicBoolean()).set(true);
                    yield new EvolutionTick.Halted(d.driftMagnitude());
                } else {
                    haltFlags.computeIfAbsent(agentKey, k -> new AtomicBoolean()).set(false);
                    yield new EvolutionTick.Drifting(d.driftMagnitude());
                }
            }
            case DispositionStatus.EvolutionPending p -> {
                var result = evolution.evaluate(descriptor, p);
                yield switch (result) {
                    case DispositionEvolution.EvolutionResult.Evolved e -> {
                        profileStore.update(descriptor.agentId(), descriptor.tenancyId(), e.newProfile());
                        recordTransitionCase(descriptor, e, p.type().name());
                        signalStore.clear(descriptor.agentId(), descriptor.tenancyId());
                        haltFlags.computeIfAbsent(agentKey, k -> new AtomicBoolean()).set(false);
                        yield new EvolutionTick.Evolved(e.previousTypeLabel(), e.newTypeLabel(), e.newProfile());
                    }
                    case DispositionEvolution.EvolutionResult.Dampened d -> {
                        signalStore.decay(descriptor.agentId(), descriptor.tenancyId(), d.decayFactor());
                        haltFlags.computeIfAbsent(agentKey, k -> new AtomicBoolean()).set(true);
                        yield new EvolutionTick.Dampened(d.decayFactor());
                    }
                };
            }
        };
    }

    private void recordTransitionCase(final AgentDescriptor descriptor,
                                       final DispositionEvolution.EvolutionResult.Evolved evolved,
                                       final String triggerType) {
        var oldProfile = descriptor.disposition().dispositionProfile();
        var oldSorted = oldProfile != null
            ? oldProfile.stream().sorted(Comparator.comparingDouble(DispositionValue::weight).reversed()).toList()
            : List.<DispositionValue>of();
        var newSorted = evolved.newProfile().stream()
            .sorted(Comparator.comparingDouble(DispositionValue::weight).reversed()).toList();

        var features = Map.<String, FeatureValue>of(
            "agent_id", FeatureValue.categorical(descriptor.agentId()),
            "old_dominant", FeatureValue.categorical(oldSorted.isEmpty() ? "unknown" : oldSorted.get(0).term()),
            "new_dominant", FeatureValue.categorical(newSorted.isEmpty() ? "unknown" : newSorted.get(0).term()),
            "old_auxiliary", FeatureValue.categorical(oldSorted.size() < 2 ? "unknown" : oldSorted.get(1).term()),
            "new_auxiliary", FeatureValue.categorical(newSorted.size() < 2 ? "unknown" : newSorted.get(1).term()),
            "trigger_type", FeatureValue.categorical(triggerType)
        );
        var transitionCase = CbrCase.of(PersonalityTransitionSchema.schema(), features);
        cbrStore.store(transitionCase, descriptor.agentId(), descriptor.tenancyId(),
                       MemoryDomain.AGENT, PersonalityTransitionSchema.CASE_TYPE, null, null);
    }

    private static String agentKey(final AgentDescriptor descriptor) {
        return descriptor.tenancyId() + ":" + descriptor.agentId();
    }
}
```

- [ ] **Step 4: Run core tick cycle test**

Run: `mvn --batch-mode test -pl blocks -Dtest=PersonalityEvolutionOrchestratorTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 5: Write halt flag test**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.CapabilityHealth.ProbeContext;
import io.casehub.eidos.api.DispositionHealth.DispositionStatus;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import java.util.concurrent.atomic.AtomicInteger;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class HaltFlagTest {

    private DispositionSignalStore signalStore;
    private DispositionHealth health;
    private PersonalityEvolutionOrchestrator orchestrator;
    private AgentDescriptor descriptor;
    private ProbeContext probeContext;
    private AtomicInteger recordCount;

    @BeforeEach
    void setUp() {
        recordCount = new AtomicInteger();
        signalStore = mock(DispositionSignalStore.class);
        doAnswer(inv -> { recordCount.incrementAndGet(); return null; })
            .when(signalStore).recordActivation(anyString(), anyString(), anyString(), any());
        health = mock(DispositionHealth.class);
        var evolution = mock(DispositionEvolution.class);
        var profileStore = mock(DispositionProfileStore.class);
        var cbrStore = mock(CbrCaseMemoryStore.class);

        TraitPressureSource<String> stringSource = new TraitPressureSource<>() {
            @Override public Class<String> eventType() { return String.class; }
            @Override public List<TraitActivation> translate(String event, AgentDescriptor d) {
                return List.of(new TraitActivation("ti", SignalValence.POSITIVE));
            }
        };
        @SuppressWarnings("unchecked")
        Instance<TraitPressureSource<?>> sources = mock(Instance.class);
        when(sources.stream()).thenReturn(java.util.stream.Stream.of(stringSource));

        orchestrator = new PersonalityEvolutionOrchestrator(
            signalStore, health, evolution, profileStore, cbrStore, sources,
            PersonalityEvolutionConfig.defaults());

        descriptor = mock(AgentDescriptor.class);
        when(descriptor.agentId()).thenReturn("agent-1");
        when(descriptor.tenancyId()).thenReturn("tenant-1");
        var disposition = mock(AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(
            List.of(new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));
        when(descriptor.disposition()).thenReturn(disposition);

        probeContext = mock(ProbeContext.class);
    }

    @Test
    void recordingStopsWhenHalted() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.Drifted(Map.of("ti", 0.55), "ti", 0.20));

        orchestrator.tick(descriptor, probeContext);

        recordCount.set(0);
        orchestrator.record("event", descriptor);
        assertThat(recordCount.get()).isZero();
    }

    @Test
    void recordingResumesAfterSubCeilingDrift() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.Drifted(Map.of("ti", 0.55), "ti", 0.20))
            .thenReturn(new DispositionStatus.Drifted(Map.of("ti", 0.38), "ti", 0.08));

        orchestrator.tick(descriptor, probeContext);
        orchestrator.tick(descriptor, probeContext);

        recordCount.set(0);
        orchestrator.record("event", descriptor);
        assertThat(recordCount.get()).isEqualTo(1);
    }

    @Test
    void dampenedSetsHaltFlag() {
        when(health.probe(descriptor, probeContext))
            .thenReturn(new DispositionStatus.EvolutionPending(
                () -> "DOMINANT_AUXILIARY_SWAP", "ne", Map.of()));
        var evolution = mock(DispositionEvolution.class);
        when(evolution.evaluate(eq(descriptor), any()))
            .thenReturn(new DispositionEvolution.EvolutionResult.Dampened(0.2));

        // Need a fresh orchestrator with this evolution mock
        @SuppressWarnings("unchecked")
        Instance<TraitPressureSource<?>> sources = mock(Instance.class);
        TraitPressureSource<String> stringSource = new TraitPressureSource<>() {
            @Override public Class<String> eventType() { return String.class; }
            @Override public List<TraitActivation> translate(String e, AgentDescriptor d) {
                return List.of(new TraitActivation("ti", SignalValence.POSITIVE));
            }
        };
        when(sources.stream()).thenReturn(java.util.stream.Stream.of(stringSource));
        var orch = new PersonalityEvolutionOrchestrator(
            signalStore, health, evolution, mock(DispositionProfileStore.class),
            mock(CbrCaseMemoryStore.class), sources, PersonalityEvolutionConfig.defaults());

        orch.tick(descriptor, probeContext);

        recordCount.set(0);
        orch.record("event", descriptor);
        assertThat(recordCount.get()).isZero();
    }
}
```

- [ ] **Step 6: Run halt flag test**

Run: `mvn --batch-mode test -pl blocks -Dtest=HaltFlagTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 7: Write event type dispatch test**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import io.casehub.eidos.api.DispositionHealth.DispositionStatus;
import io.casehub.neocortex.memory.cbr.CbrCaseMemoryStore;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.concurrent.atomic.AtomicBoolean;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class EventTypeDispatchTest {

    @Test
    void recordMatchesEventToCorrectSource() {
        var stringMatched = new AtomicBoolean();
        var intMatched = new AtomicBoolean();

        TraitPressureSource<String> stringSrc = new TraitPressureSource<>() {
            @Override public Class<String> eventType() { return String.class; }
            @Override public List<TraitActivation> translate(String e, AgentDescriptor d) {
                stringMatched.set(true);
                return List.of(new TraitActivation("ti", SignalValence.POSITIVE));
            }
        };
        TraitPressureSource<Integer> intSrc = new TraitPressureSource<>() {
            @Override public Class<Integer> eventType() { return Integer.class; }
            @Override public List<TraitActivation> translate(Integer e, AgentDescriptor d) {
                intMatched.set(true);
                return List.of(new TraitActivation("ne", SignalValence.NEGATIVE));
            }
        };

        @SuppressWarnings("unchecked")
        Instance<TraitPressureSource<?>> sources = mock(Instance.class);
        when(sources.stream()).thenReturn(java.util.stream.Stream.of(stringSrc, intSrc));

        var signalStore = mock(DispositionSignalStore.class);
        var orch = new PersonalityEvolutionOrchestrator(
            signalStore, mock(DispositionHealth.class), mock(DispositionEvolution.class),
            mock(DispositionProfileStore.class), mock(CbrCaseMemoryStore.class),
            sources, PersonalityEvolutionConfig.defaults());

        var descriptor = mock(AgentDescriptor.class);
        when(descriptor.agentId()).thenReturn("a");
        when(descriptor.tenancyId()).thenReturn("t");
        var disposition = mock(AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(
            List.of(new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));
        when(descriptor.disposition()).thenReturn(disposition);

        orch.record(42, descriptor);

        assertThat(intMatched.get()).isTrue();
        assertThat(stringMatched.get()).isFalse();
    }
}
```

- [ ] **Step 8: Run event type dispatch test**

Run: `mvn --batch-mode test -pl blocks -Dtest=EventTypeDispatchTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 9: Run all orchestrator tests together**

Run: `mvn --batch-mode test -pl blocks -Dtest="PersonalityEvolutionOrchestratorTest,HaltFlagTest,EventTypeDispatchTest" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/PersonalityEvolutionOrchestrator.java blocks/src/test/java/io/casehub/blocks/agentic/personality/
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#118): implement PersonalityEvolutionOrchestrator — tick cycle, halt flag, CBR recording, event dispatch Refs #118"
```

---

## Batch 4: Default pressure sources

### Task 4: Implement BehavioralSignalPressureSource

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/BehavioralSignalPressureSource.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/BehavioralSignalPressureSourceTest.java`

**Interfaces:**
- Consumes: `TraitPressureSource<BehavioralSignal>`, `TraitActivation`, `SignalValence` (Tasks 1, 2)
- Consumes: `AgentDescriptor`, `AgentDisposition`, `DispositionValue`, `BehavioralSignal` (eidos-api)
- Consumes: `VocabularyRegistry`, `VocabularyTerm` (eidos-api) — for Layer 2 structural enrichment
- Produces: `BehavioralSignalPressureSource` — CDI discoverable default pressure source

- [ ] **Step 1: Write test for SUCCESS mapping to dominant**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class BehavioralSignalPressureSourceTest {

    private AgentDescriptor descriptorWithProfile(List<DispositionValue> profile) {
        var descriptor = mock(AgentDescriptor.class);
        var disposition = mock(AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(profile);
        when(descriptor.disposition()).thenReturn(disposition);
        return descriptor;
    }

    @Test
    void eventTypeIsBehavioralSignal() {
        var source = new BehavioralSignalPressureSource();
        assertThat(source.eventType()).isEqualTo(BehavioralSignal.class);
    }

    @Test
    void successActivatesDominantPositive() {
        var source = new BehavioralSignalPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35),
            new DispositionValue("ne", 0.20),
            new DispositionValue("fi", 0.10)));

        var result = source.translate(BehavioralSignal.SUCCESS, descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ti");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.POSITIVE);
    }

    @Test
    void declineActivatesAuxiliaryNegative() {
        var source = new BehavioralSignalPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35),
            new DispositionValue("ne", 0.20)));

        var result = source.translate(BehavioralSignal.DECLINE, descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ne");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.NEGATIVE);
    }

    @Test
    void compliantActivatesDominantPositive() {
        var source = new BehavioralSignalPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("openness", 0.30),
            new DispositionValue("conscientiousness", 0.25)));

        var result = source.translate(BehavioralSignal.COMPLIANT, descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("openness");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.POSITIVE);
    }

    @Test
    void emptyProfileReturnsEmptyList() {
        var source = new BehavioralSignalPressureSource();
        var descriptor = descriptorWithProfile(List.of());

        var result = source.translate(BehavioralSignal.SUCCESS, descriptor);

        assertThat(result).isEmpty();
    }

    @Test
    void singleFunctionProfileDeclineStillActivates() {
        var source = new BehavioralSignalPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.80)));

        var result = source.translate(BehavioralSignal.DECLINE, descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ti");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.NEGATIVE);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=BehavioralSignalPressureSourceTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Implement BehavioralSignalPressureSource**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/BehavioralSignalPressureSource.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.BehavioralSignal;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.eidos.api.SignalValence;

import java.util.Comparator;
import java.util.List;

public class BehavioralSignalPressureSource implements TraitPressureSource<BehavioralSignal> {

    @Override
    public Class<BehavioralSignal> eventType() {
        return BehavioralSignal.class;
    }

    @Override
    public List<TraitActivation> translate(final BehavioralSignal event,
                                            final AgentDescriptor descriptor) {
        var profile = descriptor.disposition().dispositionProfile();
        if (profile == null || profile.isEmpty()) {
            return List.of();
        }
        var sorted = profile.stream()
            .sorted(Comparator.comparingDouble(DispositionValue::weight).reversed())
            .toList();

        return switch (event) {
            case SUCCESS, COMPLIANT -> List.of(
                new TraitActivation(sorted.get(0).term(), SignalValence.POSITIVE));
            case DECLINE, VIOLATED -> {
                var target = sorted.size() > 1 ? sorted.get(1) : sorted.get(0);
                yield List.of(new TraitActivation(target.term(), SignalValence.NEGATIVE));
            }
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl blocks -Dtest=BehavioralSignalPressureSourceTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/BehavioralSignalPressureSource.java blocks/src/test/java/io/casehub/blocks/agentic/personality/BehavioralSignalPressureSourceTest.java
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#118): add BehavioralSignalPressureSource — SUCCESS/COMPLIANT/DECLINE/VIOLATED to trait activations Refs #118"
```

### Task 5: Implement RelationshipPressureSource and GoalOutcomePressureSource

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/RelationshipPressureSource.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/GoalOutcomePressureSource.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/RelationshipPressureSourceTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/GoalOutcomePressureSourceTest.java`

**Interfaces:**
- Consumes: `TraitPressureSource`, `TraitActivation`, `SignalValence` (Tasks 1, 2)
- Consumes: `RelationshipEvent`, `QualitySignal` (neocortex-memory-api)
- Consumes: `GoalOutcomeCounts` (eidos-api)
- Produces: `RelationshipPressureSource` — maps POSITIVE/NEGATIVE/NEUTRAL quality to dominant/auxiliary activations
- Produces: `GoalOutcomePressureSource` — maps success rate to dominant/auxiliary activations with configurable thresholds

- [ ] **Step 1: Write RelationshipPressureSource test**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import io.casehub.neocortex.memory.relationship.QualitySignal;
import io.casehub.neocortex.memory.relationship.RelationshipEvent;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class RelationshipPressureSourceTest {

    private AgentDescriptor descriptorWithProfile(List<DispositionValue> profile) {
        var descriptor = mock(AgentDescriptor.class);
        var disposition = mock(AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(profile);
        when(descriptor.disposition()).thenReturn(disposition);
        return descriptor;
    }

    private RelationshipEvent eventWithQuality(QualitySignal quality) {
        return new RelationshipEvent("a1", "a2", "t1", "c1", "turn1",
            "interaction", quality, "desc", 0.5, Map.of());
    }

    @Test
    void positiveQualityActivatesDominantPositive() {
        var source = new RelationshipPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));

        var result = source.translate(eventWithQuality(QualitySignal.POSITIVE), descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ti");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.POSITIVE);
    }

    @Test
    void negativeQualityActivatesAuxiliaryNegative() {
        var source = new RelationshipPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));

        var result = source.translate(eventWithQuality(QualitySignal.NEGATIVE), descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ne");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.NEGATIVE);
    }

    @Test
    void neutralQualityReturnsEmpty() {
        var source = new RelationshipPressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));

        var result = source.translate(eventWithQuality(QualitySignal.NEUTRAL), descriptor);

        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Write GoalOutcomePressureSource test**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

class GoalOutcomePressureSourceTest {

    private AgentDescriptor descriptorWithProfile(List<DispositionValue> profile) {
        var descriptor = mock(AgentDescriptor.class);
        var disposition = mock(AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(profile);
        when(descriptor.disposition()).thenReturn(disposition);
        return descriptor;
    }

    @Test
    void highSuccessRateActivatesDominantPositive() {
        var source = new GoalOutcomePressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));

        var result = source.translate(new GoalOutcomeCounts(8, 2), descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ti");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.POSITIVE);
    }

    @Test
    void lowSuccessRateActivatesAuxiliaryNegative() {
        var source = new GoalOutcomePressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));

        var result = source.translate(new GoalOutcomeCounts(2, 8), descriptor);

        assertThat(result).hasSize(1);
        assertThat(result.get(0).functionTerm()).isEqualTo("ne");
        assertThat(result.get(0).valence()).isEqualTo(SignalValence.NEGATIVE);
    }

    @Test
    void midRangeSuccessRateReturnsEmpty() {
        var source = new GoalOutcomePressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35), new DispositionValue("ne", 0.20)));

        var result = source.translate(new GoalOutcomeCounts(5, 5), descriptor);

        assertThat(result).isEmpty();
    }

    @Test
    void zeroCasesReturnsEmpty() {
        var source = new GoalOutcomePressureSource();
        var descriptor = descriptorWithProfile(List.of(
            new DispositionValue("ti", 0.35)));

        var result = source.translate(new GoalOutcomeCounts(0, 0), descriptor);

        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 3: Run both tests to verify they fail**

Run: `mvn --batch-mode test -pl blocks -Dtest="RelationshipPressureSourceTest,GoalOutcomePressureSourceTest" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: FAIL

- [ ] **Step 4: Implement RelationshipPressureSource**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/RelationshipPressureSource.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.eidos.api.SignalValence;
import io.casehub.neocortex.memory.relationship.QualitySignal;
import io.casehub.neocortex.memory.relationship.RelationshipEvent;

import java.util.Comparator;
import java.util.List;

public class RelationshipPressureSource implements TraitPressureSource<RelationshipEvent> {

    @Override
    public Class<RelationshipEvent> eventType() {
        return RelationshipEvent.class;
    }

    @Override
    public List<TraitActivation> translate(final RelationshipEvent event,
                                            final AgentDescriptor descriptor) {
        var profile = descriptor.disposition().dispositionProfile();
        if (profile == null || profile.isEmpty()) {
            return List.of();
        }
        if (event.qualitySignal() == QualitySignal.NEUTRAL) {
            return List.of();
        }
        var sorted = profile.stream()
            .sorted(Comparator.comparingDouble(DispositionValue::weight).reversed())
            .toList();

        if (event.qualitySignal() == QualitySignal.POSITIVE) {
            return List.of(new TraitActivation(sorted.get(0).term(), SignalValence.POSITIVE));
        }
        var target = sorted.size() > 1 ? sorted.get(1) : sorted.get(0);
        return List.of(new TraitActivation(target.term(), SignalValence.NEGATIVE));
    }
}
```

- [ ] **Step 5: Implement GoalOutcomePressureSource**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/GoalOutcomePressureSource.java`:

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.eidos.api.GoalOutcomeCounts;
import io.casehub.eidos.api.SignalValence;

import java.util.Comparator;
import java.util.List;

public class GoalOutcomePressureSource implements TraitPressureSource<GoalOutcomeCounts> {

    private static final double SUCCESS_THRESHOLD = 0.7;
    private static final double FAILURE_FLOOR = 0.3;

    @Override
    public Class<GoalOutcomeCounts> eventType() {
        return GoalOutcomeCounts.class;
    }

    @Override
    public List<TraitActivation> translate(final GoalOutcomeCounts event,
                                            final AgentDescriptor descriptor) {
        var profile = descriptor.disposition().dispositionProfile();
        if (profile == null || profile.isEmpty()) {
            return List.of();
        }
        double rate = event.successRate();
        if (Double.isNaN(rate)) {
            return List.of();
        }
        var sorted = profile.stream()
            .sorted(Comparator.comparingDouble(DispositionValue::weight).reversed())
            .toList();

        if (rate >= SUCCESS_THRESHOLD) {
            return List.of(new TraitActivation(sorted.get(0).term(), SignalValence.POSITIVE));
        }
        if (rate <= FAILURE_FLOOR) {
            var target = sorted.size() > 1 ? sorted.get(1) : sorted.get(0);
            return List.of(new TraitActivation(target.term(), SignalValence.NEGATIVE));
        }
        return List.of();
    }
}
```

- [ ] **Step 6: Run both tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="RelationshipPressureSourceTest,GoalOutcomePressureSourceTest" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 7: Run all personality tests together**

Run: `mvn --batch-mode test -pl blocks -Dtest="io.casehub.blocks.agentic.personality.*" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/RelationshipPressureSource.java blocks/src/main/java/io/casehub/blocks/agentic/personality/GoalOutcomePressureSource.java blocks/src/test/java/io/casehub/blocks/agentic/personality/
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#118): add RelationshipPressureSource and GoalOutcomePressureSource — complete default pressure source set Refs #118"
```

- [ ] **Step 9: Run full project build**

Run: `mvn --batch-mode install -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 10: Update CLAUDE.md with new package documentation**

Add the `personality` sub-package entry to the agentic section in CLAUDE.md, following the established table format for sub-packages.

- [ ] **Step 11: Commit CLAUDE.md update**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "docs(#118): add agentic.personality package to CLAUDE.md Refs #118"
```

---

## References

- `specs/issue-126-autonomous-agent-patterns/2026-08-18-personality-evolution-design.md` — design spec this plan implements
- `specs/issue-126-autonomous-agent-patterns/decisions.md` — 6 design decisions (D1–D6)
- `eidos/api/src/main/java/io/casehub/eidos/api/DispositionSignalStore.java` — existing signal accumulation SPI
- `eidos/runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultDispositionHealth.java` — probe logic
- `eidos/runtime/src/main/java/io/casehub/eidos/runtime/health/DefaultDispositionEvolution.java` — JPAF rules
- `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/cbr/PersonalityTransitionSchema.java` — CBR case schema
- `GE-20260811-e941cc` — AgentDisposition vs DispositionProfile type distinction
- `GE-20260728-a53632` — vocabulary-generic structural navigation technique
- GitHub #118 — PersonalityEvolution pattern
- GitHub #126 — epic: autonomous agent patterns
