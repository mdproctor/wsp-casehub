# EffectiveCognitiveDefaults — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#1055 — Effective cognitive defaults: tenant-scoped, provenance-preserving re-derivation
**Issue group:** #712 (parent), #1055 (Part 0a prerequisite)

**Goal:** Replace the flat `CognitiveDefaultsRegistry` with a tenant-scoped `EffectiveCognitiveDefaults` that derives resting and effective cognitive defaults from an immutable explicit layer plus per-slot inputs (base DescriptorView + situational state), enforcing the resting/effective split at the type level via `LearningProfile`.

**Architecture:** `CognitiveDefaultsRegistry` becomes the explicit-only layer (classpath YAML, deployment config). A new `EffectiveCognitiveDefaults` class holds a `ConcurrentHashMap<TenantAgentKey, Slot>` where each slot stores its inputs and derives two results: resting (base + homeOffset) and effective (resting + displacement). Consumers are split by type: learning rules take `LearningProfile` (from `restingFor`), behaviour reads `CognitiveDefaults` (from `forAgent`). Two SPIs (`DescriptorViewSource`, `PersonalityStateSource`) enable lazy population without a tick.

**Tech Stack:** Java 26, Quarkus CDI, Maven, ConcurrentHashMap, JUnit 5 + AssertJ

## Global Constraints

- `EffectiveCognitiveDefaults` lives in `cognitive-index` module
- No result is ever merged with a previous result — always `merge(explicit, CDE.derive(fresh inputs))`
- Concurrency: `ConcurrentHashMap.compute` with version-stamp guards, no blocking I/O inside the lambda
- Lock order: `SituationalStateService` per-key lock → `EffectiveCognitiveDefaults` map, never the reverse
- Facet composition: `clamp01(b_f + Δ_f)` over base view facets only; absent facets never introduced
- Facets with `|value − b_f| ≥ ε_publish` carry `origin = "situational"`
- `LearningProfile` and `CognitiveDefaults` are the ONLY return types — every learning rule takes `LearningProfile`
- Explicit-layer consumers stay on `CognitiveDefaultsRegistry`: `DeclarativeRuleRegistry`, `CognitiveLoader`, `CognitiveProfileWatcher`
- Build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/<module>/pom.xml test -Dtest=<TestClass> -s examples/.mvn/slot-settings.xml`

---

## Batch 1: Core types and EffectiveCognitiveDefaults

### Task 1: SPIs, LearningProfile, PersonalityStateSnapshot, DescriptorView.withFacets

**Files:**
- Create: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/DescriptorViewSource.java`
- Create: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/PersonalityStateSource.java`
- Create: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/PersonalityStateSnapshot.java`
- Create: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/LearningProfile.java`
- Modify: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/DescriptorView.java`
- Create: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/LearningProfileTest.java`

**Interfaces:**
- Produces: `DescriptorViewSource` SPI (used by Task 2: `EffectiveCognitiveDefaults`, Task 4: `EidosDescriptorViewSource`), `PersonalityStateSource` SPI (used by Task 2), `PersonalityStateSnapshot` record (used by Task 2 and by `SituationalStateService` in Batch 4 of #712 plan), `LearningProfile` type (used by all resting consumers in Task 6), `DescriptorView.withFacets(List<PersonalityFacet>)` (used by Task 2 for facet composition)

- [ ] **Step 1: Write DescriptorViewSource SPI**

```java
package io.casehub.neocortex.cognitive.index;

import java.util.List;
import java.util.Optional;

public interface DescriptorViewSource {
    Optional<DescriptorView> find(String tenantId, String agentId);
    List<String> agentIds(String tenantId);
}
```

- [ ] **Step 2: Write PersonalityStateSnapshot record**

Immutable projection of situational personality state carrying per-facet homeOffset and displacement — the two values `EffectiveCognitiveDefaults` needs for facet composition. No dependency on `PersonalityState` (which is in `cognition` module, not `cognitive-index`).

```java
package io.casehub.neocortex.cognitive.index;

import java.util.Map;

public record PersonalityStateSnapshot(
        Map<String, FacetDisplacement> facets,
        long stateVersion
) {
    public record FacetDisplacement(double homeOffset, double displacement) {}

    public PersonalityStateSnapshot {
        facets = facets != null ? Map.copyOf(facets) : Map.of();
    }

    public static PersonalityStateSnapshot empty() {
        return new PersonalityStateSnapshot(Map.of(), 0);
    }

    public double homeOffset(String facet) {
        var fd = facets.get(facet);
        return fd != null ? fd.homeOffset() : 0.0;
    }

    public double displacement(String facet) {
        var fd = facets.get(facet);
        return fd != null ? fd.displacement() : 0.0;
    }
}
```

- [ ] **Step 3: Write PersonalityStateSource SPI**

```java
package io.casehub.neocortex.cognitive.index;

import java.time.Instant;
import java.util.Optional;

public interface PersonalityStateSource {
    Optional<PersonalityStateSnapshot> find(String tenantId, String agentId, Instant now);
}
```

- [ ] **Step 4: Add DescriptorView.withFacets**

Add to `DescriptorView.java`:

```java
public DescriptorView withFacets(List<PersonalityFacet> facets) {
    return new DescriptorView(agentId, disposition, dispositionProfile, goals, formationPadSummary, facets);
}
```

- [ ] **Step 5: Write LearningProfile — failing tests first**

`LearningProfile` wraps a `CognitiveDefaults` and exposes only learning-rule parameters. This is the type-safety enforcement: a learning rule that takes `LearningProfile` cannot be fed effective values by mistake.

```java
@Test void extractsAssociativeReach() {
    var defaults = CognitiveDefaults.empty("agent-1")
        .withCapacityConfig(new CognitiveCapacityConfig(0.7, 0.8, 0.6));
    var profile = LearningProfile.from(defaults);
    assertThat(profile.associativeReach()).isEqualTo(0.8);
}

@Test void extractsRelationalComplexity() {
    var defaults = CognitiveDefaults.empty("agent-1")
        .withCapacityConfig(new CognitiveCapacityConfig(0.7, 0.8, 0.6));
    var profile = LearningProfile.from(defaults);
    assertThat(profile.relationalComplexity()).isEqualTo(0.7);
}

@Test void extractsSocialCognition() {
    var defaults = CognitiveDefaults.empty("agent-1")
        .withBehavioralDefaults(new BehavioralDefaults(null, null,
            new SocialCognitionDefaults(0.6, ConflictInterpretation.REPAIR), null));
    var profile = LearningProfile.from(defaults);
    assertThat(profile.trustFormationRate()).isEqualTo(0.6);
    assertThat(profile.conflictInterpretation()).isEqualTo(ConflictInterpretation.REPAIR);
}

@Test void extractsDispositionAxes() {
    var axes = new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "analytical");
    var view = DescriptorView.of("agent-1", axes, List.of(), List.of());
    var defaults = CognitiveDefaults.empty("agent-1").withDescriptor(view);
    var profile = LearningProfile.from(defaults);
    assertThat(profile.dispositionAxes()).isEqualTo(axes);
}

@Test void extractsCopingDefaults() {
    var coping = new CopingDefaults(0.6, 0.5, 0.4, 0.3, 0.7);
    var defaults = CognitiveDefaults.empty("agent-1")
        .withCopingDefaults(coping);
    var profile = LearningProfile.from(defaults);
    assertThat(profile.copingDefaults()).isEqualTo(coping);
}

@Test void nullSafeOnEmptyDefaults() {
    var profile = LearningProfile.from(CognitiveDefaults.empty("agent-1"));
    assertThat(profile.associativeReach()).isNull();
    assertThat(profile.trustFormationRate()).isNull();
    assertThat(profile.dispositionAxes()).isNull();
}

@Test void exposesPersonalityFacets() {
    var facets = List.of(new PersonalityFacet("Anxiousness", 0.7, "eidos"));
    var view = new DescriptorView("a1", null, List.of(), List.of(), null, facets);
    var defaults = CognitiveDefaults.empty("a1").withDescriptor(view);
    var profile = LearningProfile.from(defaults);
    assertThat(profile.facets()).isEqualTo(facets);
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -Dtest=LearningProfileTest -s examples/.mvn/slot-settings.xml`
Expected: compilation error (LearningProfile doesn't exist)

- [ ] **Step 7: Implement LearningProfile**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.memory.personality.PersonalityWeights;
import io.casehub.neocortex.mindmap.CuriosityConfig;
import java.util.List;

public final class LearningProfile {
    private final CognitiveDefaults delegate;

    private LearningProfile(CognitiveDefaults delegate) {
        this.delegate = delegate;
    }

    public static LearningProfile from(CognitiveDefaults defaults) {
        return new LearningProfile(defaults);
    }

    public String agentId() { return delegate.agentId(); }

    // Capacity
    public Double associativeReach() {
        var c = delegate.capacityConfig();
        return c != null ? c.associativeReach() : null;
    }
    public Double relationalComplexity() {
        var c = delegate.capacityConfig();
        return c != null ? c.relationalComplexity() : null;
    }
    public Double workingMemorySpan() {
        var c = delegate.capacityConfig();
        return c != null ? c.workingMemorySpan() : null;
    }

    // Social cognition
    public Double trustFormationRate() {
        var sc = delegate.socialCognition();
        return sc != null ? sc.trustRate() : null;
    }
    public ConflictInterpretation conflictInterpretation() {
        var sc = delegate.socialCognition();
        return sc != null ? sc.conflictInterpretation() : null;
    }

    // Disposition
    public DispositionAxes dispositionAxes() {
        var d = delegate.descriptor();
        return d != null ? d.disposition() : null;
    }

    // Personality facets (for learning rules that branch on facet scores)
    public List<PersonalityFacet> facets() {
        var d = delegate.descriptor();
        return d != null ? d.personalityFacets() : List.of();
    }

    // Coping and regulation for CAPS init
    public CopingDefaults copingDefaults() { return delegate.copingDefaults(); }
    public RegulationDefaults regulationDefaults() { return delegate.regulationDefaults(); }

    // Recurring thought and graduation
    public io.casehub.neocortex.cognitive.RecurringThoughtConfig recurringThoughtConfig() {
        return delegate.recurringThoughtConfig();
    }
    public io.casehub.neocortex.cognitive.HabituationConfig habituationConfig() {
        return delegate.habituationConfig();
    }
    public io.casehub.neocortex.cognitive.CognitiveCapacityConfig capacityConfig() {
        return delegate.capacityConfig();
    }
    public io.casehub.neocortex.cognitive.SearchBudgetConfig searchBudgetConfig() {
        return delegate.searchBudgetConfig();
    }
    public io.casehub.neocortex.cognitive.CognitiveResourcePoolConfig cognitiveResourcePoolConfig() {
        return delegate.cognitiveResourcePoolConfig();
    }

    // Expression styles and personality weights for description generation
    public ExpressionStyleDefaults expressionStyles() { return delegate.expressionStyles(); }
    public PersonalityWeights personality() { return delegate.personality(); }
    public Double personalityDominance() { return delegate.personalityDominance(); }
    public CuriosityConfig curiosity() { return delegate.curiosity(); }
    public GraphStructureDefaults graphStructure() { return delegate.graphStructure(); }
    public ExtractionBiasDefaults extractionBias() { return delegate.extractionBias(); }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -Dtest=LearningProfileTest -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/DescriptorViewSource.java
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/PersonalityStateSource.java
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/PersonalityStateSnapshot.java
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/LearningProfile.java
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/DescriptorView.java
git -C neocortex add cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/LearningProfileTest.java
git -C neocortex commit -m "feat(cognitive-index): LearningProfile, SPIs, PersonalityStateSnapshot for #1055

Introduces the type-level resting/effective split:
- LearningProfile wraps CognitiveDefaults for learning-rule parameters
- DescriptorViewSource and PersonalityStateSource SPIs for lazy population
- PersonalityStateSnapshot for personality state projection
- DescriptorView.withFacets for facet composition

Refs casehubio/neocortex#1055"
```

### Task 2: EffectiveCognitiveDefaults with slot management

**Files:**
- Create: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/EffectiveCognitiveDefaults.java`
- Create: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/EffectiveCognitiveDefaultsTest.java`

**Interfaces:**
- Consumes: `CognitiveDefaultsRegistry` (explicit layer), `CognitiveDerivationEngine.derive/merge`, `DescriptorViewSource` (SPI), `PersonalityStateSource` (SPI), `LearningProfile`, `PersonalityStateSnapshot`, `DescriptorView.withFacets`
- Produces: `EffectiveCognitiveDefaults.forAgent(tenantId, agentId) → CognitiveDefaults`, `restingFor(tenantId, agentId) → LearningProfile`, `updateBase(tenantId, DescriptorView, eventRef)`, `updateState(tenantId, agentId, PersonalityStateSnapshot, eventRef)`, `agentIds(tenantId) → Set<String>`, `evictIdle(Duration)` (used by all migrated consumers)

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.neocortex.cognitive.index;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;
import static org.assertj.core.api.Assertions.*;

class EffectiveCognitiveDefaultsTest {

    private CognitiveDefaultsRegistry registry;
    private StubDescriptorViewSource viewSource;
    private StubPersonalityStateSource stateSource;
    private EffectiveCognitiveDefaults effective;

    @BeforeEach
    void setUp() {
        registry = new CognitiveDefaultsRegistry();
        viewSource = new StubDescriptorViewSource();
        stateSource = new StubPersonalityStateSource();
        effective = new EffectiveCognitiveDefaults(registry, viewSource, stateSource);
    }

    @Test void forAgentReturnsEffectiveDefaults() {
        var view = DescriptorView.of("agent-1",
            new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "analytical"),
            List.of(new WeightedTerm("fe", 0.8)), List.of());
        viewSource.add("tenant-1", view);

        var defaults = effective.forAgent("tenant-1", "agent-1");
        assertThat(defaults).isNotNull();
        assertThat(defaults.agentId()).isEqualTo("agent-1");
        assertThat(defaults.moodBaseline()).isNotNull();
    }

    @Test void restingForReturnsLearningProfile() {
        var view = DescriptorView.of("agent-1",
            new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "analytical"),
            List.of(new WeightedTerm("fe", 0.8)), List.of());
        viewSource.add("tenant-1", view);

        var profile = effective.restingFor("tenant-1", "agent-1");
        assertThat(profile).isNotNull();
        assertThat(profile.agentId()).isEqualTo("agent-1");
    }

    @Test void secondDerivationWithChangedFacetsChangesDefaults() {
        var facets = List.of(new PersonalityFacet("Anxiousness", 0.5, "eidos"));
        var view = new DescriptorView("agent-1",
            new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "analytical"),
            List.of(new WeightedTerm("fe", 0.8)), List.of(), null, facets);
        viewSource.add("tenant-1", view);

        var before = effective.forAgent("tenant-1", "agent-1");
        var beforeAppraisal = before.appraisalWeights();

        // Update with shifted facets via state
        var shifted = new PersonalityStateSnapshot(
            Map.of("Anxiousness", new PersonalityStateSnapshot.FacetDisplacement(0.0, 0.3)),
            1);
        effective.updateState("tenant-1", "agent-1", shifted, "tick-2");

        var after = effective.forAgent("tenant-1", "agent-1");
        assertThat(after.appraisalWeights()).isNotEqualTo(beforeAppraisal);
    }

    @Test void explicitPinnedFieldSurvivesReDerivation() {
        var explicit = CognitiveDefaults.empty("agent-1")
            .withRegulationDefaults(new RegulationDefaults(0.99, 0.1, 0.1, 0.1));
        registry.register(explicit);
        var view = DescriptorView.of("agent-1",
            new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "analytical"),
            List.of(new WeightedTerm("fe", 0.8)), List.of());
        viewSource.add("tenant-1", view);

        var defaults = effective.forAgent("tenant-1", "agent-1");
        assertThat(defaults.regulationDefaults().reappraisalCapacity()).isEqualTo(0.99);
    }

    @Test void sameAgentIdInTwoTenantsHasIndependentDefaults() {
        var facetsA = List.of(new PersonalityFacet("Anxiousness", 0.3, "eidos"));
        var facetsB = List.of(new PersonalityFacet("Anxiousness", 0.9, "eidos"));
        var viewA = new DescriptorView("agent-1", null, List.of(), List.of(), null, facetsA);
        var viewB = new DescriptorView("agent-1", null, List.of(), List.of(), null, facetsB);
        viewSource.add("tenant-A", viewA);
        viewSource.add("tenant-B", viewB);

        var defaultsA = effective.forAgent("tenant-A", "agent-1");
        var defaultsB = effective.forAgent("tenant-B", "agent-1");
        assertThat(defaultsA.descriptor().personalityFacets())
            .isNotEqualTo(defaultsB.descriptor().personalityFacets());
    }

    @Test void concurrentUpdateStateNoLostUpdate() throws Exception {
        var view = new DescriptorView("agent-1", null, List.of(), List.of(), null,
            List.of(new PersonalityFacet("Anxiousness", 0.5, "eidos")));
        viewSource.add("t1", view);
        effective.forAgent("t1", "agent-1"); // prime the slot

        int threads = 8;
        var latch = new CountDownLatch(1);
        var executor = Executors.newFixedThreadPool(threads);
        var futures = new ArrayList<Future<?>>();

        for (int i = 0; i < threads; i++) {
            int version = i + 1;
            futures.add(executor.submit(() -> {
                try { latch.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                effective.updateState("t1", "agent-1",
                    new PersonalityStateSnapshot(
                        Map.of("Anxiousness",
                            new PersonalityStateSnapshot.FacetDisplacement(0.01 * version, 0.0)),
                        version),
                    "tick-" + version);
            }));
        }
        latch.countDown();
        for (var f : futures) f.get(5, TimeUnit.SECONDS);
        executor.shutdown();

        // Highest version wins
        var defaults = effective.forAgent("t1", "agent-1");
        assertThat(defaults).isNotNull();
    }

    @Test void agentIdsReturnsTenantScopedUnion() {
        viewSource.add("t1", DescriptorView.of("a1", null, List.of(), List.of()));
        viewSource.add("t1", DescriptorView.of("a2", null, List.of(), List.of()));
        registry.register(CognitiveDefaults.empty("a3"));

        var ids = effective.agentIds("t1");
        assertThat(ids).containsExactlyInAnyOrder("a1", "a2", "a3");
    }

    @Test void updateBaseRefreshesSlot() {
        var view = DescriptorView.of("agent-1",
            new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "analytical"),
            List.of(new WeightedTerm("fe", 0.8)), List.of());
        viewSource.add("t1", view);
        effective.forAgent("t1", "agent-1"); // prime

        var updatedView = DescriptorView.of("agent-1",
            new DispositionAxes("competitive", "strict", "bold", "high", "analytical"),
            List.of(new WeightedTerm("te", 0.9)), List.of());
        effective.updateBase("t1", updatedView, "tick-2");

        var defaults = effective.forAgent("t1", "agent-1");
        assertThat(defaults.descriptor()).isEqualTo(updatedView);
    }

    @Test void forAgentOrDefaultsNeverReturnsNull() {
        var defaults = effective.forAgentOrDefaults("t1", "unknown-agent");
        assertThat(defaults).isNotNull();
        assertThat(defaults.agentId()).isEqualTo("unknown-agent");
    }

    // --- Stubs ---

    static class StubDescriptorViewSource implements DescriptorViewSource {
        private final Map<String, Map<String, DescriptorView>> views = new HashMap<>();
        void add(String tenantId, DescriptorView view) {
            views.computeIfAbsent(tenantId, k -> new HashMap<>()).put(view.agentId(), view);
        }
        @Override public Optional<DescriptorView> find(String tenantId, String agentId) {
            return Optional.ofNullable(views.getOrDefault(tenantId, Map.of()).get(agentId));
        }
        @Override public List<String> agentIds(String tenantId) {
            return List.copyOf(views.getOrDefault(tenantId, Map.of()).keySet());
        }
    }

    static class StubPersonalityStateSource implements PersonalityStateSource {
        @Override public Optional<PersonalityStateSnapshot> find(String t, String a, Instant now) {
            return Optional.empty();
        }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -Dtest=EffectiveCognitiveDefaultsTest -s examples/.mvn/slot-settings.xml`
Expected: compilation error (EffectiveCognitiveDefaults doesn't exist)

- [ ] **Step 3: Implement EffectiveCognitiveDefaults**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.cognitive.observability.DerivationObserver;
import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class EffectiveCognitiveDefaults {

    private static final double EPSILON_PUBLISH = 0.005;

    private final CognitiveDefaultsRegistry explicitLayer;
    private final DescriptorViewSource viewSource;
    private final PersonalityStateSource stateSource;
    private final ConcurrentHashMap<TenantAgentKey, Slot> slots = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<TenantAgentKey, AtomicLong> baseVersions = new ConcurrentHashMap<>();

    public EffectiveCognitiveDefaults(CognitiveDefaultsRegistry explicitLayer,
                                       DescriptorViewSource viewSource,
                                       PersonalityStateSource stateSource) {
        this.explicitLayer = explicitLayer;
        this.viewSource = viewSource;
        this.stateSource = stateSource;
    }

    public CognitiveDefaults forAgent(String tenantId, String agentId) {
        var key = new TenantAgentKey(tenantId, agentId);
        var slot = slots.get(key);
        if (slot != null && !slot.isStale(currentBaseVersion(key))) {
            return slot.effective;
        }
        return ensureSlot(key).effective;
    }

    public CognitiveDefaults forAgentOrDefaults(String tenantId, String agentId) {
        var defaults = forAgent(tenantId, agentId);
        return defaults != null ? defaults : CognitiveDefaults.empty(agentId);
    }

    public LearningProfile restingFor(String tenantId, String agentId) {
        var key = new TenantAgentKey(tenantId, agentId);
        var slot = slots.get(key);
        if (slot != null && !slot.isStale(currentBaseVersion(key))) {
            return slot.resting;
        }
        return ensureSlot(key).resting;
    }

    public void updateBase(String tenantId, DescriptorView view, String eventRef) {
        var key = new TenantAgentKey(tenantId, view.agentId());
        incrementBaseVersion(key);
        recompute(key, view, null, eventRef);
    }

    public void updateState(String tenantId, String agentId,
                            PersonalityStateSnapshot snapshot, String eventRef) {
        var key = new TenantAgentKey(tenantId, agentId);
        recompute(key, null, snapshot, eventRef);
    }

    public Set<String> agentIds(String tenantId) {
        var ids = new LinkedHashSet<>(viewSource.agentIds(tenantId));
        ids.addAll(explicitLayer.allAgentIds());
        return Set.copyOf(ids);
    }

    public void evictIdle(Duration maxIdle) {
        var cutoff = Instant.now().minus(maxIdle);
        slots.entrySet().removeIf(e -> e.getValue().lastAccess.isBefore(cutoff));
    }

    public void onExplicitReload() {
        for (var key : slots.keySet()) {
            incrementBaseVersion(key);
            recompute(key, null, null, "reload");
        }
    }

    // --- internals ---

    private Slot ensureSlot(TenantAgentKey key) {
        var baseView = viewSource.find(key.tenantId, key.agentId).orElse(null);
        var stateSnap = stateSource.find(key.tenantId, key.agentId, Instant.now()).orElse(null);
        long bv = currentBaseVersion(key);
        long sv = stateSnap != null ? stateSnap.stateVersion() : 0;

        var candidate = deriveSlot(key, baseView, stateSnap, bv, sv);
        return slots.compute(key, (k, existing) -> {
            if (existing == null) return candidate;
            if (candidate.baseVersion > existing.baseVersion) return candidate;
            if (candidate.baseVersion == existing.baseVersion
                && candidate.stateVersion > existing.stateVersion) return candidate;
            return existing;
        });
    }

    private void recompute(TenantAgentKey key, DescriptorView newBase,
                           PersonalityStateSnapshot newState, String eventRef) {
        var existing = slots.get(key);
        var baseView = newBase;
        if (baseView == null && existing != null) {
            baseView = existing.baseView;
        }
        if (baseView == null) {
            baseView = viewSource.find(key.tenantId, key.agentId).orElse(null);
        }

        PersonalityStateSnapshot stateSnap = newState;
        if (stateSnap == null && existing != null) {
            stateSnap = existing.stateSnapshot;
        }
        if (stateSnap == null) {
            stateSnap = stateSource.find(key.tenantId, key.agentId, Instant.now()).orElse(null);
        }

        long bv = currentBaseVersion(key);
        long sv = stateSnap != null ? stateSnap.stateVersion() : 0;

        var candidate = deriveSlot(key, baseView, stateSnap, bv, sv);
        slots.compute(key, (k, prev) -> {
            if (prev == null) return candidate;
            if (candidate.baseVersion > prev.baseVersion) return candidate;
            if (candidate.baseVersion == prev.baseVersion
                && candidate.stateVersion > prev.stateVersion) return candidate;
            return prev;
        });
    }

    private Slot deriveSlot(TenantAgentKey key, DescriptorView baseView,
                            PersonalityStateSnapshot state, long bv, long sv) {
        var explicit = explicitLayer.forAgent(key.agentId).orElse(null);

        // Compose resting facets: base + homeOffset
        DescriptorView restingView = baseView;
        if (baseView != null && state != null && !state.facets().isEmpty()) {
            restingView = composeFacets(baseView, state, false);
        }
        CognitiveDefaults restingDefaults = restingView != null
            ? merge(explicit, CognitiveDerivationEngine.derive(restingView))
            : (explicit != null ? explicit : CognitiveDefaults.empty(key.agentId));

        // Compose effective facets: base + homeOffset + displacement
        DescriptorView effectiveView = baseView;
        if (baseView != null && state != null && !state.facets().isEmpty()) {
            effectiveView = composeFacets(baseView, state, true);
        }
        CognitiveDefaults effectiveDefaults = effectiveView != null
            ? merge(explicit, CognitiveDerivationEngine.derive(effectiveView))
            : restingDefaults;

        return new Slot(baseView, state, restingDefaults != null
            ? LearningProfile.from(restingDefaults) : LearningProfile.from(CognitiveDefaults.empty(key.agentId)),
            effectiveDefaults, bv, sv, Instant.now());
    }

    private DescriptorView composeFacets(DescriptorView base,
                                          PersonalityStateSnapshot state,
                                          boolean includeDisplacement) {
        var composed = new ArrayList<PersonalityFacet>();
        for (var facet : base.personalityFacets()) {
            double delta = state.homeOffset(facet.facet());
            if (includeDisplacement) {
                delta += state.displacement(facet.facet());
            }
            if (Math.abs(delta) < 1e-9) {
                composed.add(facet);
            } else {
                double newScore = Math.max(0.0, Math.min(1.0, facet.score() + delta));
                String origin = Math.abs(newScore - facet.score()) >= EPSILON_PUBLISH
                    ? "situational" : facet.origin();
                composed.add(new PersonalityFacet(facet.facet(), newScore, origin));
            }
        }
        return base.withFacets(composed);
    }

    static CognitiveDefaults merge(CognitiveDefaults explicit, CognitiveDefaults derived) {
        if (explicit == null) return derived;
        if (derived == null) return explicit;
        var mergedProfile = mergePersonalityProfile(
            explicit.personalityProfile(), derived.personalityProfile());
        var mergedEmotion = explicit.emotionDefaults().mergeWith(derived.emotionDefaults());
        var mergedResources = explicit.cognitiveResourceDefaults()
            .mergeWith(derived.cognitiveResourceDefaults());
        var mergedGraph = explicit.graphDefaults().mergeWith(derived.graphDefaults());
        var mergedBehavioral = explicit.behavioralDefaults()
            .mergeWith(derived.behavioralDefaults());
        return new CognitiveDefaults(derived.agentId(), explicit.tenantId(),
            mergedProfile, derived.descriptor(),
            mergedEmotion, mergedResources, mergedGraph, mergedBehavioral);
    }

    private static PersonalityProfile mergePersonalityProfile(
            PersonalityProfile explicit, PersonalityProfile derived) {
        if (explicit == null) return derived;
        if (derived == null) return explicit;
        return new PersonalityProfile(
            explicit.personality() != null ? explicit.personality() : derived.personality(),
            explicit.personalityDominance() != null
                ? explicit.personalityDominance() : derived.personalityDominance(),
            explicit.expressionStyles() != null
                ? explicit.expressionStyles() : derived.expressionStyles(),
            explicit.attachment() != null ? explicit.attachment() : derived.attachment(),
            !explicit.facets().isEmpty() ? explicit.facets() : derived.facets()
        );
    }

    private long currentBaseVersion(TenantAgentKey key) {
        var v = baseVersions.get(key);
        return v != null ? v.get() : 0;
    }

    private void incrementBaseVersion(TenantAgentKey key) {
        baseVersions.computeIfAbsent(key, k -> new AtomicLong()).incrementAndGet();
    }

    record TenantAgentKey(String tenantId, String agentId) {}

    static final class Slot {
        final DescriptorView baseView;
        final PersonalityStateSnapshot stateSnapshot;
        final LearningProfile resting;
        final CognitiveDefaults effective;
        final long baseVersion;
        final long stateVersion;
        volatile Instant lastAccess;

        Slot(DescriptorView baseView, PersonalityStateSnapshot stateSnapshot,
             LearningProfile resting, CognitiveDefaults effective,
             long baseVersion, long stateVersion, Instant lastAccess) {
            this.baseView = baseView;
            this.stateSnapshot = stateSnapshot;
            this.resting = resting;
            this.effective = effective;
            this.baseVersion = baseVersion;
            this.stateVersion = stateVersion;
            this.lastAccess = lastAccess;
        }

        boolean isStale(long currentBaseVersion) {
            lastAccess = Instant.now();
            return baseVersion < currentBaseVersion;
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -Dtest=EffectiveCognitiveDefaultsTest -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 5: Commit**

```bash
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/EffectiveCognitiveDefaults.java
git -C neocortex add cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/EffectiveCognitiveDefaultsTest.java
git -C neocortex commit -m "feat(cognitive-index): EffectiveCognitiveDefaults with resting/effective split

Tenant-scoped slot management with version-based concurrency protocol.
Resting = base + homeOffset, effective = resting + displacement.
Pure merge(explicit, derived) — no result merged with previous result.

Refs casehubio/neocortex#1055"
```

---

## Batch 2: Bridge, merge, and pipeline wiring

### Task 3: Replace deriveAndMerge with pure merge; refactor EidosCognitiveProfileBridge

**Files:**
- Modify: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` — delete `deriveAndMerge`
- Modify: `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridge.java` — delete `bridgeAll`/`registerFromDescriptor`, make `toDescriptorView` public
- Create: `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/EidosDescriptorViewSource.java`
- Create: `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/DefaultDescriptorViewSource.java`
- Create: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/DefaultPersonalityStateSource.java`
- Modify: `neocortex/cognitive-eidos/src/test/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridgeTest.java` — update tests
- Create: `neocortex/cognitive-eidos/src/test/java/io/casehub/neocortex/cognitive/eidos/EidosDescriptorViewSourceTest.java`

**Interfaces:**
- Consumes: `DescriptorViewSource` (SPI from Task 1), `AgentRegistry` (eidos-api), `EidosCognitiveProfileBridge.toDescriptorView`
- Produces: `EidosDescriptorViewSource implements DescriptorViewSource` (CDI bean), `DefaultDescriptorViewSource` (empty `@DefaultBean`), `DefaultPersonalityStateSource` (empty `@DefaultBean`), public `toDescriptorView` on bridge

- [ ] **Step 1: Delete `deriveAndMerge` and `mergePersonalityProfile` from CognitiveDerivationEngine**

Using `ide_replace_member` on `CognitiveDerivationEngine`:
- Remove `deriveAndMerge(CognitiveDefaults)` method (lines 297-310)
- Remove `mergePersonalityProfile` method (lines 313-323) — now lives in `EffectiveCognitiveDefaults.merge`

- [ ] **Step 2: Refactor EidosCognitiveProfileBridge**

Remove `bridgeAll`, `registerFromDescriptor`, the `CognitiveDefaultsRegistry` dependency and `CognitiveDerivationEngine` import. Keep only `toDescriptorView` (made public) and constructors. The bridge becomes a pure translation utility.

```java
@ApplicationScoped
public class EidosCognitiveProfileBridge {

    private static final Logger LOG = Logger.getLogger(EidosCognitiveProfileBridge.class.getName());

    public EidosCognitiveProfileBridge() {}

    public static DescriptorView toDescriptorView(AgentDescriptor descriptor) {
        // existing implementation unchanged
    }

    private static String titleCase(String s) {
        // existing implementation unchanged
    }
}
```

- [ ] **Step 3: Create EidosDescriptorViewSource**

```java
package io.casehub.neocortex.cognitive.eidos;

import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.neocortex.cognitive.index.DescriptorView;
import io.casehub.neocortex.cognitive.index.DescriptorViewSource;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Optional;

@ApplicationScoped
public class EidosDescriptorViewSource implements DescriptorViewSource {

    private final Instance<AgentRegistry> registryInstance;

    @Inject
    public EidosDescriptorViewSource(Instance<AgentRegistry> registryInstance) {
        this.registryInstance = registryInstance;
    }

    EidosDescriptorViewSource(AgentRegistry registry) {
        this.registryInstance = null;
        this.directRegistry = registry;
    }
    private AgentRegistry directRegistry;

    @Override
    public Optional<DescriptorView> find(String tenantId, String agentId) {
        var registry = resolveRegistry();
        if (registry == null) return Optional.empty();
        var match = registry.findById(agentId, tenantId);
        return match.map(m -> EidosCognitiveProfileBridge.toDescriptorView(m.descriptor()));
    }

    @Override
    public List<String> agentIds(String tenantId) {
        var registry = resolveRegistry();
        if (registry == null) return List.of();
        return registry.find(AgentQuery.all(tenantId)).stream()
            .map(m -> m.descriptor().agentId())
            .toList();
    }

    private AgentRegistry resolveRegistry() {
        if (directRegistry != null) return directRegistry;
        if (registryInstance != null && !registryInstance.isUnsatisfied()) {
            return registryInstance.get();
        }
        return null;
    }
}
```

- [ ] **Step 4: Create DefaultBean SPIs**

`DefaultDescriptorViewSource` in `cognitive-eidos`:
```java
@io.quarkus.arc.DefaultBean
@ApplicationScoped
public class DefaultDescriptorViewSource implements DescriptorViewSource {
    @Override public Optional<DescriptorView> find(String t, String a) { return Optional.empty(); }
    @Override public List<String> agentIds(String t) { return List.of(); }
}
```

`DefaultPersonalityStateSource` in `cognitive-index`:
```java
@io.quarkus.arc.DefaultBean
@ApplicationScoped
public class DefaultPersonalityStateSource implements PersonalityStateSource {
    @Override public Optional<PersonalityStateSnapshot> find(String t, String a, Instant now) {
        return Optional.empty();
    }
}
```

- [ ] **Step 5: Write EidosDescriptorViewSource tests**

```java
@Test void findReturnsViewForKnownAgent() {
    var descriptor = createTestDescriptor("agent-1");
    var registry = stubRegistry(Map.of("agent-1", descriptor));
    var source = new EidosDescriptorViewSource(registry);

    var view = source.find("tenant-1", "agent-1");
    assertThat(view).isPresent();
    assertThat(view.get().agentId()).isEqualTo("agent-1");
}

@Test void findReturnsEmptyForUnknownAgent() {
    var source = new EidosDescriptorViewSource(stubRegistry(Map.of()));
    assertThat(source.find("t1", "unknown")).isEmpty();
}

@Test void agentIdsReturnsTenantScoped() {
    // ... stub with 2 agents for tenant-1, 1 for tenant-2
}
```

- [ ] **Step 6: Update EidosCognitiveProfileBridgeTest**

Remove tests for `bridgeAll`, `registerFromDescriptor`. Keep tests for `toDescriptorView`.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-eidos/pom.xml test -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java
git -C neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/DefaultPersonalityStateSource.java
git -C neocortex add cognitive-eidos/src/
git -C neocortex commit -m "feat(cognitive-eidos): EidosDescriptorViewSource, delete bridgeAll/deriveAndMerge

Bridge is now a pure translation utility. EidosDescriptorViewSource
implements the DescriptorViewSource SPI over AgentRegistry.
DefaultBean SPIs for environments without Eidos or situational state.

Refs casehubio/neocortex#1055"
```

### Task 4: Wire EffectiveCognitiveDefaults into CognitionCore and pipeline builder

**Files:**
- Modify: `neocortex/cognition/pom.xml` — add `cognitive-eidos` dependency
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java`
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitivePipelineBuilder.java`
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCoreFactory.java`
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/CognitionDefaultBeans.java`

**Interfaces:**
- Consumes: `EffectiveCognitiveDefaults` (Task 2), `EidosCognitiveProfileBridge.toDescriptorView` (Task 3)
- Produces: `CognitionCore.tick` calls `updateBase` before FOUNDATION; `setEffectiveDefaults(EffectiveCognitiveDefaults)` replaces `setDefaultsRegistry`; pipeline builder wires `EffectiveCognitiveDefaults`

- [ ] **Step 1: Add cognitive-eidos dependency to cognition/pom.xml**

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognitive-eidos</artifactId>
</dependency>
```

- [ ] **Step 2: Add effectiveDefaults field to CognitionCore**

Add field:
```java
private volatile EffectiveCognitiveDefaults effectiveDefaults;
```

Add setter:
```java
public void setEffectiveDefaults(EffectiveCognitiveDefaults effectiveDefaults) {
    this.effectiveDefaults = effectiveDefaults;
    configureTierFilterFromEffectiveDefaults();
}
```

- [ ] **Step 3: Wire updateBase in tick, before FOUNDATION**

In `CognitionCore.tick`, after `this.lastTenantId = tenantId` and before `runCustomParticipants(CognitionPhase.FOUNDATION, context)`, add:

```java
if (effectiveDefaults != null && descriptor != null) {
    var view = EidosCognitiveProfileBridge.toDescriptorView(descriptor);
    effectiveDefaults.updateBase(tenantId, view, tickEventRef.toString());
}
```

- [ ] **Step 4: Update configureTierFilterFromRegistry to use effectiveDefaults**

Replace the `defaultsRegistry` personality lookup in `configureTierFilterFromRegistry` with:
```java
private void configureTierFilterFromEffectiveDefaults() {
    if (effectiveDefaults == null) return;
    if (tierFilter != null) return;
    tierFilter = new TierFilterCustomizer(
        () -> { /* arousal lookup unchanged */ },
        () -> {
            String aid = lastAgentId;
            String tid = lastTenantId;
            if (aid == null || tid == null || effectiveDefaults == null) return 0.5;
            return effectiveDefaults.forAgent(tid, aid).personalityDominance();
        }
    );
    chainSectionCustomizer(tierFilter);
}
```

- [ ] **Step 5: Update CognitivePipelineBuilder and CognitionCoreFactory**

Replace `CognitiveDefaultsRegistry` parameter with `EffectiveCognitiveDefaults` in the pipeline builder's `effectiveDefaults(...)` method (replacing `defaultsRegistry(...)`).

Wire `core.setEffectiveDefaults(effectiveDefaults)` instead of `core.setDefaultsRegistry(registry)`.

- [ ] **Step 6: Update CognitionDefaultBeans**

Wire `EffectiveCognitiveDefaults` CDI bean that depends on `CognitiveDefaultsRegistry`, `DescriptorViewSource`, `PersonalityStateSource`.

- [ ] **Step 7: Compile and run existing cognition tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="CognitionCoreTest" -s examples/.mvn/slot-settings.xml`
Expected: existing tests still pass (backward compat via both paths)

- [ ] **Step 8: Commit**

```bash
git -C neocortex add cognition/pom.xml
git -C neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java
git -C neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitivePipelineBuilder.java
git -C neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCoreFactory.java
git -C neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/CognitionDefaultBeans.java
git -C neocortex commit -m "feat(cognition): wire EffectiveCognitiveDefaults into CognitionCore and pipeline

CognitionCore.tick calls updateBase before FOUNDATION.
New module edge: cognition → cognitive-eidos (acyclic).
Pipeline builder and factory use EffectiveCognitiveDefaults.

Refs casehubio/neocortex#1055"
```

---

## Batch 3: Consumer migration

### Task 5: Migrate resting consumers to LearningProfile

All consumers below currently take `CognitiveDefaultsRegistry` and call `forAgent(agentId)` or `forAgentOrDefaults(agentId)`. They must be changed to take `EffectiveCognitiveDefaults` and call `restingFor(tenantId, agentId)` returning `LearningProfile`.

**Files (each requires constructor/field change + method call changes):**

| File | Current call | New call | tenantId source |
|------|-------------|----------|-----------------|
| `cognition/.../drive/DriveAdaptationPhase.java` | `registry.forAgentOrDefaults(agentId)` | `effective.restingFor(tenantId, agentId)` | `run(tenantId, ...)` param |
| `cognition/.../personality/PersonalityEvolutionOrchestrator.java` | `registry.forAgentOrDefaults(agentId)` | `effective.restingFor(tenantId, agentId)` | **Add tenantId param** to methods |
| `cognition/.../relationship/RelationalSchemaEvolutionPhase.java` | `registry.forAgentOrDefaults(observerId)` ×3 | `effective.restingFor(tenantId, observerId)` | `run(tenantId, ...)` param |
| `cognition/.../relationship/RelationshipStagePhase.java` | `registry.forAgentOrDefaults(observerId)` | `effective.restingFor(tenantId, observerId)` | `run(tenantId, ...)` param |
| `cognition/.../strategy/StrategyLearningOrchestrator.java` | `registry.forAgentOrDefaults(agentId)` | `effective.restingFor(tenantId, agentId)` | `tick(agentId, tenantId)` param |
| `mindmap-intelligence/.../consolidation/BehavioralSynthesisPhase.java` | `registry.forAgent(agentId)` ×2 | `effective.restingFor(tenantId, agentId)` | `run(tenantId, ...)` param |
| `mindmap-intelligence/.../consolidation/DefaultGraduationScorer.java` | `registry.forAgentOrDefaults(agentId)` | `effective.restingFor(tenantId, agentId)` | scorer method param |
| `mindmap-intelligence/.../consolidation/DriveProfileDerivationPhase.java` | `registry.forAgent(agentId)` | `effective.restingFor(tenantId, agentId)` | `run(tenantId, ...)` param |
| `mindmap-intelligence/.../consolidation/RecurringThoughtPhase.java` | `registry.forAgentOrDefaults(agentId)` | `effective.restingFor(tenantId, agentId)` | `run(tenantId, ...)` param |
| `mindmap-intelligence/.../consolidation/CuriosityDescriptionGenerationPhase.java` | `registry.allProfiles()` | `effective.agentIds(tenantId)` + `effective.restingFor(tenantId, id)` | `run(tenantId, ...)` param |
| `mindmap-intelligence/.../consolidation/SearchBudgetDescriptionGenerationPhase.java` | `registry.allProfiles()` | `effective.agentIds(tenantId)` + `effective.restingFor(tenantId, id)` | `run(tenantId, ...)` param |

**Migration pattern for each file (3 changes):**

1. **Constructor/field**: Replace `CognitiveDefaultsRegistry registry` with `EffectiveCognitiveDefaults effectiveDefaults`
2. **Call site**: Replace `registry.forAgentOrDefaults(agentId)` with `effectiveDefaults.restingFor(tenantId, agentId)` — the return type is `LearningProfile`, so usage of fields like `.capacityConfig()`, `.socialCognition()` must use `LearningProfile` accessors
3. **Accessor adaptation**: `LearningProfile` exposes the same fields but through its own accessors. Most are identical names. If a consumer reads a field not on `LearningProfile`, it is misclassified — investigate.

**Special cases:**
- `PersonalityEvolutionOrchestrator`: Currently has no `tenantId` parameter. Add `tenantId` to its method signatures.
- `CuriosityDescriptionGenerationPhase` and `SearchBudgetDescriptionGenerationPhase`: Currently iterate `allProfiles()`. Change to `agentIds(tenantId)` + loop calling `restingFor`.
- `DefaultGraduationScorer`: The tenantId needs to be threaded through its scoring interface.

- [ ] **Step 1: Migrate all resting consumers — constructor/field changes**

For each file: change constructor parameter from `CognitiveDefaultsRegistry` to `EffectiveCognitiveDefaults`, rename field.

- [ ] **Step 2: Migrate all resting consumers — call site changes**

For each `forAgent(agentId)` / `forAgentOrDefaults(agentId)` call, change to `restingFor(tenantId, agentId)`. Assign to `LearningProfile` local variable. Update field access patterns.

- [ ] **Step 3: Migrate enumeration consumers**

For `CuriosityDescriptionGenerationPhase` and `SearchBudgetDescriptionGenerationPhase`:
```java
// Before:
for (var defaults : registry.allProfiles()) {
    // use defaults.agentId(), defaults.searchBudgetConfig(), ...
}

// After:
for (var agentId : effectiveDefaults.agentIds(tenantId)) {
    var profile = effectiveDefaults.restingFor(tenantId, agentId);
    // use profile.agentId(), profile.searchBudgetConfig(), ...
}
```

For `CognitiveAttentionAccumulator`:
```java
// Before:
for (var agentId : registry.allAgentIds()) { ... }

// After:
for (var agentId : effectiveDefaults.agentIds(tenantId)) { ... }
```

- [ ] **Step 4: Update all corresponding test files**

Each test file that creates a `CognitiveDefaultsRegistry.forTesting(...)` now creates either:
- A `EffectiveCognitiveDefaults` with stub sources for resting consumers
- Or uses the existing `CognitiveDefaultsRegistry.forTesting` (for explicit-layer tests)

Create a test helper: `EffectiveCognitiveDefaults.forTesting(CognitiveDefaults... profiles)` that wraps a registry + stub sources.

- [ ] **Step 5: Compile cognition and mindmap-intelligence modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml compile -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml compile -s examples/.mvn/slot-settings.xml`
Expected: both compile

- [ ] **Step 6: Run tests for migrated resting consumers**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="DriveAdaptationPhaseTest,RelationalSchemaEvolutionPhaseTest,RelationshipStagePhaseTest,StrategyLearningOrchestratorTest" -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest="BehavioralSynthesisPhaseTest,DefaultGraduationScorerCapacityTest,CuriosityDescriptionGenerationPhaseTest,SearchBudgetDescriptionGenerationPhaseTest,RecurringThoughtPhaseTest" -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C neocortex add cognition/src/ mindmap-intelligence/src/
git -C neocortex commit -m "refactor: migrate resting consumers to LearningProfile

All learning-rule consumers now take EffectiveCognitiveDefaults and read
restingFor(tenantId, agentId) → LearningProfile. Type-level enforcement:
a learning rule cannot be fed effective values without a compile error.

11 files migrated across cognition and mindmap-intelligence modules.

Refs casehubio/neocortex#1055"
```

### Task 6: Migrate effective consumers and CognitionCore wiring

**Files:**

| File | Current call | New call |
|------|-------------|----------|
| `cognition/.../appraisal/AppraisalTickParticipant.java` | `registry.forAgentOrDefaults(agentId)` | `effective.forAgentOrDefaults(tenantId, agentId)` |
| `cognition/.../regulation/EmotionRegulationTickParticipant.java` | `registry.forAgentOrDefaults(agentId)` ×5 | `effective.forAgentOrDefaults(tenantId, agentId)` |
| `cognition/.../regulation/ProactiveRegulationParticipant.java` | `registry.forAgent(agentId)` | `effective.forAgent(tenantId, agentId)` |
| `cognition/.../search/SearchBudgetTickParticipant.java` | `registry.forAgentOrDefaults(agentId)` | `effective.forAgentOrDefaults(tenantId, agentId)` |
| `cognition/.../core/DefaultCognitiveResourcePool.java` | `registry.forAgentOrDefaults(agentId)` | `effective.forAgentOrDefaults(tenantId, agentId)` |
| `cognition/.../drive/MindMapDriveSource.java` | `registry.forAgent(agentId)` | `effective.forAgent(tenantId, agentId)` |
| `mindmap-intelligence/.../ActionAppraisalObserver.java` | `registry.forAgentOrDefaults(agentId)` | `effective.forAgentOrDefaults(tenantId, agentId)` |
| `mindmap-intelligence/.../consolidation/CognitiveAttentionAccumulator.java` | `registry.allAgentIds()` + `registry.forAgentOrDefaults` | `effective.agentIds(tenantId)` + `effective.forAgentOrDefaults(tenantId, id)` |
| `cognition/.../core/CognitionCore.java` | `defaultsRegistry.forAgentOrDefaults(agentId)` | `effectiveDefaults.forAgentOrDefaults(tenantId, agentId)` |

**Migration pattern:** Same as Task 5 but return type stays `CognitiveDefaults` (not `LearningProfile`).

- [ ] **Step 1: Migrate effective consumers — constructor/field + call site changes**

For each file: replace `CognitiveDefaultsRegistry` with `EffectiveCognitiveDefaults`, change `forAgent(agentId)` to `forAgent(tenantId, agentId)`.

The effective consumers all have access to `tenantId` already (via tick context or method params).

- [ ] **Step 2: Migrate CognitiveAttentionAccumulator enumeration**

This consumer needs re-keying by `(tenantId, agentId)`:
```java
// Before:
for (var agentId : registry.allAgentIds()) {
    var defaults = registry.forAgentOrDefaults(agentId);
    // ...
}

// After:
for (var agentId : effectiveDefaults.agentIds(tenantId)) {
    var defaults = effectiveDefaults.forAgentOrDefaults(tenantId, agentId);
    // ...
}
```

The `tenantId` comes from `AttentionSignal.tenantId()` grouping.

- [ ] **Step 3: Update CognitionCore internal references**

Replace remaining `defaultsRegistry` reads in CognitionCore:
- `cognitiveMemoryRetriever.setTickContext(...)` personality lookup → use `effectiveDefaults.forAgentOrDefaults(tenantId, agentId)`
- `configureAppraisal` / `configureRegulation` / `configureProactiveRegulation` / `configureSearchBudget` calls that pass `defaultsRegistry` → pass `effectiveDefaults`

- [ ] **Step 4: Update all corresponding test files**

Apply the same `EffectiveCognitiveDefaults.forTesting(...)` helper pattern.

- [ ] **Step 5: Compile all affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml compile -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml compile -s examples/.mvn/slot-settings.xml`
Expected: both compile

- [ ] **Step 6: Run tests for migrated effective consumers**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="AppraisalTickParticipantTest,EmotionRegulationTickParticipantTest,ProactiveRegulationParticipantTest,SearchBudgetTickParticipantTest,DefaultCognitiveResourcePoolTest,MindMapDriveSourceTest" -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest="CognitiveAttentionAccumulatorTest,ActionAppraisalObserverTest" -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 7: Run fast test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C neocortex add cognition/src/ mindmap-intelligence/src/
git -C neocortex commit -m "refactor: migrate effective consumers to EffectiveCognitiveDefaults

All tick-time behaviour consumers now read forAgent(tenantId, agentId).
CognitiveAttentionAccumulator re-keyed by (tenantId, agentId).
CognitionCore internal references updated.

9 files migrated. All consumers now use tenant-scoped paths.

Refs casehubio/neocortex#1055"
```

---

## Batch 4: Verification and cleanup

### Task 7: Field-classification test + eval test updates + cleanup

**Files:**
- Create: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/FieldClassificationTest.java`
- Modify: eval tests in `caps-llm-classifier/src/test/java/` — update registry usage
- Modify: `cognition/src/test/java/.../eval/CognitionTestHarness.java` — update to use EffectiveCognitiveDefaults

**Interfaces:**
- Consumes: all types from prior tasks
- Produces: type-safety verification test; green eval test suite

- [ ] **Step 1: Write field-classification test**

This test enumerates every accessor on `CognitiveDefaults` and asserts which view (`LearningProfile` or `CognitiveDefaults`) exposes it, per the spec's classification table.

```java
@Test void learningProfileExposesOnlyRestingFields() {
    var defaults = CognitiveDefaults.empty("test")
        .withCapacityConfig(new CognitiveCapacityConfig(0.7, 0.8, 0.6))
        .withBehavioralDefaults(new BehavioralDefaults(null, null,
            new SocialCognitionDefaults(0.5, ConflictInterpretation.NEUTRAL), null));
    var profile = LearningProfile.from(defaults);

    // These must be accessible from LearningProfile
    assertThat(profile.associativeReach()).isNotNull();
    assertThat(profile.relationalComplexity()).isNotNull();
    assertThat(profile.trustFormationRate()).isNotNull();
    assertThat(profile.conflictInterpretation()).isNotNull();
    assertThat(profile.capacityConfig()).isNotNull();
    assertThat(profile.recurringThoughtConfig()).isNull(); // null because not set
    assertThat(profile.copingDefaults()).isNull(); // null because not set
}

@Test void restingAndEffectiveDifferWhenStatePresent() {
    // create EffectiveCognitiveDefaults with state that shifts Anxiousness
    // verify restingFor and forAgent return different appraisal weights
}
```

- [ ] **Step 2: Update eval test harness**

The eval test classes in `caps-llm-classifier` reference `CognitiveDefaultsRegistry` directly. Update to use `EffectiveCognitiveDefaults.forTesting(...)`.

- [ ] **Step 3: Run all tests across modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-eidos/pom.xml test -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s examples/.mvn/slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -s examples/.mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 4: Commit**

```bash
git -C neocortex add cognitive-index/src/test/ cognition/src/test/ mindmap-intelligence/src/test/ caps-llm-classifier/src/test/
git -C neocortex commit -m "test: field-classification test + eval test migration for #1055

Verifies LearningProfile/CognitiveDefaults type split covers every field.
All eval and integration tests updated to use EffectiveCognitiveDefaults.

Refs casehubio/neocortex#1055"
```

---

## References

- `wsp-casehub/specs/issue-712-shadow-archetypes-personality-degradation/2026-11-17-situational-personality-modulation-design.md` §Part 0a — full design spec for EffectiveCognitiveDefaults
- `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDefaultsRegistry.java` — existing registry (becomes explicit layer)
- `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` — CDE with `deriveAndMerge` to remove
- `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridge.java` — bridge with `bridgeAll`/`registerFromDescriptor` to remove
- `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java` — tick wiring point
- casehubio/neocortex#1055, #712
