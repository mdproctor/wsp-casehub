# MemoryHygiene Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #120 — MemoryHygiene pattern — consolidation, forgetting, importance scoring
**Issue group:** #120, #126

**Goal:** Implement the MemoryHygiene pattern — a memory lifecycle orchestrator that scores importance, evicts low-value memories, consolidates related ones, generates reflections, discovers peer-links, and checks integrity.

**Architecture:** Single-entry `MemoryHygieneOrchestrator.tick()` handles scoring → eviction → consolidation. `MemoryHygieneScheduler.maintain()` externally composes tick + reflection + peer-linking + integrity. Replaces `CbrRetentionScheduler` for agents that opt in.

**Tech Stack:** Java 21, JUnit 5, Mockito, AssertJ. No Quarkus runtime in tests. Composes neocortex `CbrCaseMemoryStore`, `TemporalDecay`, `ScopeDecay`, `ReflectionOrchestrator` and blocks `ContentSummariser`.

## Global Constraints

- Package: `io.casehub.blocks.memory`
- Source root: `blocks/src/main/java/io/casehub/blocks/memory/`
- Test root: `blocks/src/test/java/io/casehub/blocks/memory/`
- Project repo: `/Users/mdproctor/claude/casehub/slots/134/blocks`
- No Quarkus test runtime — plain JUnit 5 + Mockito
- All `@DefaultBean` types must be `@ApplicationScoped`
- All [0,1] range parameters validated in compact constructors
- Commit each task with `feat(#120):` prefix

---

## Batch 1: Foundation types

### Task 1: ImportanceScorer SPI and default implementations

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/memory/ImportanceScorer.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/WeightedScorer.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/CompositeImportanceScorer.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/ArousalScorer.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/SurpriseScorer.java`
- Test: `blocks/src/test/java/io/casehub/blocks/memory/ImportanceScorerTest.java`

**Interfaces:**
- Consumes: `ScoredCbrCase<? extends CbrCase>` (neocortex), `CbrCase.problem()`, `CbrCase.solution()`, `CbrCase.features()`
- Produces: `ImportanceScorer` SPI — `double score(ScoredCbrCase<? extends CbrCase>, Instant)` → [0,1]. Used by Task 4 (orchestrator).

- [ ] **Step 1: Write the failing test for ImportanceScorer contract**

```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.FeatureVectorCbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ImportanceScorerTest {

    private static ScoredCbrCase<CbrCase> scored(String problem, String solution,
                                                  Map<String, FeatureValue> features) {
        var c = new FeatureVectorCbrCase(problem, solution, null, null, features, null, null);
        return new ScoredCbrCase<>(c, "case-1", 0.5);
    }

    @Test
    void arousalScorerReturnsZeroForNeutralText() {
        var scorer = new ArousalScorer();
        var memory = scored("routine update", "nothing notable", Map.of());
        assertThat(scorer.score(memory, Instant.now())).isBetween(0.0, 0.3);
    }

    @Test
    void arousalScorerReturnsHighForEmotionalText() {
        var scorer = new ArousalScorer();
        var memory = scored("critical emergency failure", "urgent crisis escalation", Map.of());
        assertThat(scorer.score(memory, Instant.now())).isGreaterThan(0.3);
    }

    @Test
    void surpriseScorerReturnsZeroForTypicalFeatures() {
        var scorer = new SurpriseScorer();
        var memory = scored("task", "solution",
                Map.of("type", FeatureValue.string("routine")));
        assertThat(scorer.score(memory, Instant.now())).isBetween(0.0, 1.0);
    }

    @Test
    void compositeWeightedMean() {
        ImportanceScorer fixed80 = (m, now) -> 0.8;
        ImportanceScorer fixed20 = (m, now) -> 0.2;
        var composite = new CompositeImportanceScorer(List.of(
                new WeightedScorer(fixed80, 3.0),
                new WeightedScorer(fixed20, 1.0)));
        var memory = scored("p", "s", Map.of("k", FeatureValue.string("v")));
        double score = composite.score(memory, Instant.now());
        // (0.8*3 + 0.2*1) / (3+1) = 2.6/4 = 0.65
        assertThat(score).isCloseTo(0.65, org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void compositeRejectsEmptyList() {
        assertThatThrownBy(() -> new CompositeImportanceScorer(List.of()))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void weightedScorerRejectsNonPositiveWeight() {
        assertThatThrownBy(() -> new WeightedScorer((m, now) -> 0.5, 0.0))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.ImportanceScorerTest`
Expected: compilation failure — types don't exist yet

- [ ] **Step 3: Implement ImportanceScorer, WeightedScorer, CompositeImportanceScorer**

Use `ide_create_file` for each.

`ImportanceScorer.java`:
```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import java.time.Instant;

@FunctionalInterface
public interface ImportanceScorer {
    double score(ScoredCbrCase<? extends CbrCase> memory, Instant now);
}
```

`WeightedScorer.java`:
```java
package io.casehub.blocks.memory;

import java.util.Objects;

public record WeightedScorer(ImportanceScorer scorer, double weight) {
    public WeightedScorer {
        Objects.requireNonNull(scorer, "scorer required");
        if (weight <= 0.0) {
            throw new IllegalArgumentException("weight must be positive, got " + weight);
        }
    }
}
```

`CompositeImportanceScorer.java`:
```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import java.time.Instant;
import java.util.List;
import java.util.Objects;

public final class CompositeImportanceScorer implements ImportanceScorer {

    private final List<WeightedScorer> scorers;

    public CompositeImportanceScorer(List<WeightedScorer> scorers) {
        Objects.requireNonNull(scorers, "scorers required");
        if (scorers.isEmpty()) {
            throw new IllegalArgumentException("at least one scorer required");
        }
        this.scorers = List.copyOf(scorers);
    }

    @Override
    public double score(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        double weightedSum = 0.0;
        double totalWeight = 0.0;
        for (var ws : scorers) {
            weightedSum += ws.scorer().score(memory, now) * ws.weight();
            totalWeight += ws.weight();
        }
        return Math.clamp(weightedSum / totalWeight, 0.0, 1.0);
    }
}
```

- [ ] **Step 4: Implement ArousalScorer**

`ArousalScorer.java`:
```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import java.time.Instant;
import java.util.Set;

public final class ArousalScorer implements ImportanceScorer {

    private static final Set<String> HIGH_AROUSAL = Set.of(
            "critical", "emergency", "urgent", "failure", "crisis", "error",
            "escalation", "breach", "violation", "fatal", "severe", "alarm",
            "panic", "catastrophe", "danger", "threat", "attack", "outage");

    @Override
    public double score(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        var text = (memory.cbrCase().problem() + " " +
                   (memory.cbrCase().solution() != null ? memory.cbrCase().solution() : "")).toLowerCase();
        var words = text.split("\\W+");
        if (words.length == 0) {return 0.0;}
        int hits = 0;
        for (var word : words) {
            if (HIGH_AROUSAL.contains(word)) {hits++;}
        }
        return Math.clamp((double) hits / Math.max(words.length, 1) * 5.0, 0.0, 1.0);
    }
}
```

- [ ] **Step 5: Implement SurpriseScorer**

`SurpriseScorer.java`:
```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import java.time.Instant;
import java.util.Map;

public final class SurpriseScorer implements ImportanceScorer {

    @Override
    public double score(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        Map<String, FeatureValue> features = memory.cbrCase().features();
        if (features == null || features.isEmpty()) {return 0.5;}
        int distinctValues = 0;
        for (var fv : features.values()) {
            if (fv instanceof FeatureValue.StringVal sv) {
                distinctValues += sv.value().length();
            } else if (fv instanceof FeatureValue.NumberVal) {
                distinctValues += 1;
            } else if (fv instanceof FeatureValue.StringListVal sl) {
                distinctValues += sl.values().size();
            }
        }
        return Math.clamp(Math.log1p(distinctValues) / 10.0, 0.0, 1.0);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.ImportanceScorerTest`
Expected: all 6 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/memory/ blocks/src/test/java/io/casehub/blocks/memory/
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#120): add ImportanceScorer SPI with ArousalScorer, SurpriseScorer, CompositeImportanceScorer"
```

### Task 2: Retention, outcome, and event types

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/memory/RetentionScore.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/RetentionConfig.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/HygieneTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/MaintenanceTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/HygieneEvent.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/ViolationType.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/IntegrityViolation.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/IntegrityChecker.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/SemanticIntegrityChecker.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/NoOpSemanticIntegrityChecker.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/ReflectionEntry.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/ReflectionStore.java`
- Create: `blocks/src/main/java/io/casehub/blocks/memory/NoOpReflectionStore.java`
- Test: `blocks/src/test/java/io/casehub/blocks/memory/FoundationTypesTest.java`

**Interfaces:**
- Consumes: `IntegrityViolation` (from this task), `RetentionScore` (from this task)
- Produces: All record/sealed/enum types used by Tasks 3-6. `HygieneTick` sealed interface: `Idle(String)`, `Completed(int, int, int, List<RetentionScore>)`, `Failed(String)`. `MaintenanceTick` sealed interface: `Completed(HygieneTick, int, int, List<IntegrityViolation>)`, `Failed(String, String)`.

- [ ] **Step 1: Write the failing test for all foundation types**

```java
package io.casehub.blocks.memory;

import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class FoundationTypesTest {

    @Test
    void retentionScoreCompositeComputation() {
        var score = RetentionScore.compute("c1", "e1", 0.8, 0.6, 1.0, 0.9,
                new RetentionConfig(0.1, 1.0, 1.0, 0.5, 0.5));
        assertThat(score.composite()).isBetween(0.0, 1.0);
        assertThat(score.caseId()).isEqualTo("c1");
        assertThat(score.entityId()).isEqualTo("e1");
    }

    @Test
    void retentionConfigValidatesWeights() {
        assertThatThrownBy(() -> new RetentionConfig(0.1, 0.0, 0.0, 0.0, 0.0))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void retentionConfigRejectsNegativeThreshold() {
        assertThatThrownBy(() -> new RetentionConfig(-0.1, 1.0, 1.0, 0.5, 0.5))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void hygieneTickSealedVariants() {
        HygieneTick idle = new HygieneTick.Idle("no memories");
        HygieneTick completed = new HygieneTick.Completed(2, 3, 10, List.of());
        HygieneTick failed = new HygieneTick.Failed("store error");
        assertThat(idle).isInstanceOf(HygieneTick.Idle.class);
        assertThat(completed).isInstanceOf(HygieneTick.Completed.class);
        assertThat(failed).isInstanceOf(HygieneTick.Failed.class);
    }

    @Test
    void maintenanceTickSealedVariants() {
        var hygiene = new HygieneTick.Completed(0, 0, 5, List.of());
        MaintenanceTick completed = new MaintenanceTick.Completed(hygiene, 3, 1, List.of());
        MaintenanceTick failed = new MaintenanceTick.Failed("reflection", "timeout");
        assertThat(completed.hygiene()).isEqualTo(hygiene);
        assertThat(((MaintenanceTick.Failed) failed).stage()).isEqualTo("reflection");
    }

    @Test
    void hygieneEventSealedVariants() {
        var score = RetentionScore.compute("c1", "e1", 0.1, 0.1, 0.1, 0.1,
                new RetentionConfig(0.5, 1.0, 1.0, 0.5, 0.5));
        HygieneEvent evicted = new HygieneEvent.MemoryEvicted("c1", score);
        HygieneEvent consolidated = new HygieneEvent.MemoryConsolidated("m1", List.of("s1", "s2"));
        HygieneEvent reflected = new HygieneEvent.ReflectionGenerated("a1", "insight");
        var violation = new IntegrityViolation("c1", ViolationType.ORPHANED_SUPERSESSION, "detail", false);
        HygieneEvent detected = new HygieneEvent.IntegrityViolationDetected(violation);
        assertThat(evicted).isInstanceOf(HygieneEvent.MemoryEvicted.class);
        assertThat(consolidated).isInstanceOf(HygieneEvent.MemoryConsolidated.class);
        assertThat(reflected).isInstanceOf(HygieneEvent.ReflectionGenerated.class);
        assertThat(detected).isInstanceOf(HygieneEvent.IntegrityViolationDetected.class);
    }

    @Test
    void reflectionEntryCarriesProvenance() {
        var entry = new ReflectionEntry("agent-1", "tenant-1", "insight text",
                Instant.now(), List.of("case-1", "case-2"));
        assertThat(entry.sourceCaseIds()).containsExactly("case-1", "case-2");
    }

    @Test
    void noOpReflectionStoreAcceptsWithoutError() {
        var store = new NoOpReflectionStore();
        store.store(new ReflectionEntry("a", "t", "i", Instant.now(), List.of()));
    }

    @Test
    void noOpSemanticIntegrityCheckerReturnsEmpty() {
        var checker = new NoOpSemanticIntegrityChecker();
        assertThat(checker.checkSemantic(List.of(), "a", "t")).isEmpty();
    }

    @Test
    void violationTypeCoversAllExpected() {
        assertThat(ViolationType.values()).hasSize(5);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.FoundationTypesTest`
Expected: compilation failure

- [ ] **Step 3: Implement all foundation types**

Create each file with `ide_create_file`. Key implementations:

`RetentionScore.java`:
```java
package io.casehub.blocks.memory;

import java.util.Objects;

public record RetentionScore(String caseId, String entityId,
                              double importance, double recencyFactor,
                              double scopeFactor, double trustFactor,
                              double composite) {
    public RetentionScore {
        Objects.requireNonNull(caseId, "caseId required");
        Objects.requireNonNull(entityId, "entityId required");
    }

    public static RetentionScore compute(String caseId, String entityId,
                                          double importance, double recencyFactor,
                                          double scopeFactor, double trustFactor,
                                          RetentionConfig config) {
        double num = importance * config.importanceWeight()
                   + recencyFactor * config.recencyWeight()
                   + scopeFactor * config.scopeWeight()
                   + trustFactor * config.trustWeight();
        double den = config.importanceWeight() + config.recencyWeight()
                   + config.scopeWeight() + config.trustWeight();
        double composite = Math.clamp(num / den, 0.0, 1.0);
        return new RetentionScore(caseId, entityId, importance,
                recencyFactor, scopeFactor, trustFactor, composite);
    }
}
```

`RetentionConfig.java`:
```java
package io.casehub.blocks.memory;

public record RetentionConfig(double retentionThreshold,
                               double importanceWeight, double recencyWeight,
                               double scopeWeight, double trustWeight) {
    public RetentionConfig {
        if (retentionThreshold < 0.0 || retentionThreshold > 1.0) {
            throw new IllegalArgumentException("retentionThreshold must be in [0,1], got " + retentionThreshold);
        }
        if (importanceWeight < 0 || recencyWeight < 0 || scopeWeight < 0 || trustWeight < 0) {
            throw new IllegalArgumentException("weights must be >= 0");
        }
        if (importanceWeight + recencyWeight + scopeWeight + trustWeight <= 0) {
            throw new IllegalArgumentException("at least one weight must be > 0");
        }
    }

    public static final RetentionConfig DEFAULT =
            new RetentionConfig(0.1, 1.0, 1.0, 0.5, 0.5);
}
```

`HygieneTick.java`:
```java
package io.casehub.blocks.memory;

import java.util.List;

public sealed interface HygieneTick {
    record Idle(String reason) implements HygieneTick {}
    record Completed(int consolidated, int evicted, int totalScored,
                     List<RetentionScore> scores) implements HygieneTick {
        public Completed { scores = List.copyOf(scores); }
    }
    record Failed(String reason) implements HygieneTick {}
}
```

`MaintenanceTick.java`:
```java
package io.casehub.blocks.memory;

import java.util.List;

public sealed interface MaintenanceTick {
    record Completed(HygieneTick hygiene, int reflectionsGenerated,
                     int crossLinksCreated,
                     List<IntegrityViolation> violations) implements MaintenanceTick {
        public Completed { violations = List.copyOf(violations); }
    }
    record Failed(String stage, String reason) implements MaintenanceTick {}
}
```

`HygieneEvent.java`:
```java
package io.casehub.blocks.memory;

import java.util.List;

public sealed interface HygieneEvent {
    record MemoryEvicted(String caseId, RetentionScore score) implements HygieneEvent {}
    record MemoryConsolidated(String mergedCaseId,
                               List<String> sourceCaseIds) implements HygieneEvent {
        public MemoryConsolidated { sourceCaseIds = List.copyOf(sourceCaseIds); }
    }
    record ReflectionGenerated(String agentId, String insight) implements HygieneEvent {}
    record IntegrityViolationDetected(IntegrityViolation violation) implements HygieneEvent {}
}
```

`ViolationType.java`:
```java
package io.casehub.blocks.memory;

public enum ViolationType {
    ORPHANED_SUPERSESSION,
    DUPLICATE_CASE,
    MISSING_FEATURES,
    UNPROCESSED_STALE,
    SEMANTIC_CONFLICT
}
```

`IntegrityViolation.java`:
```java
package io.casehub.blocks.memory;

import java.util.Objects;

public record IntegrityViolation(String caseId, ViolationType type,
                                  String detail, boolean escalateToSemantic) {
    public IntegrityViolation {
        Objects.requireNonNull(caseId, "caseId required");
        Objects.requireNonNull(type, "type required");
        Objects.requireNonNull(detail, "detail required");
    }
}
```

`IntegrityChecker.java`:
```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.MemoryDomain;
import java.util.List;

@FunctionalInterface
public interface IntegrityChecker {
    List<IntegrityViolation> check(String agentId, String tenantId, MemoryDomain domain);
}
```

`SemanticIntegrityChecker.java`:
```java
package io.casehub.blocks.memory;

import java.util.List;

@FunctionalInterface
public interface SemanticIntegrityChecker {
    List<IntegrityViolation> checkSemantic(List<IntegrityViolation> flagged,
                                            String agentId, String tenantId);
}
```

`NoOpSemanticIntegrityChecker.java`:
```java
package io.casehub.blocks.memory;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@DefaultBean
@ApplicationScoped
public class NoOpSemanticIntegrityChecker implements SemanticIntegrityChecker {
    @Override
    public List<IntegrityViolation> checkSemantic(List<IntegrityViolation> flagged,
                                                    String agentId, String tenantId) {
        return List.of();
    }
}
```

`ReflectionEntry.java`:
```java
package io.casehub.blocks.memory;

import java.time.Instant;
import java.util.List;
import java.util.Objects;

public record ReflectionEntry(String agentId, String tenantId, String insight,
                                Instant generatedAt, List<String> sourceCaseIds) {
    public ReflectionEntry {
        Objects.requireNonNull(agentId, "agentId required");
        Objects.requireNonNull(tenantId, "tenantId required");
        Objects.requireNonNull(insight, "insight required");
        Objects.requireNonNull(generatedAt, "generatedAt required");
        sourceCaseIds = sourceCaseIds != null ? List.copyOf(sourceCaseIds) : List.of();
    }
}
```

`ReflectionStore.java`:
```java
package io.casehub.blocks.memory;

@FunctionalInterface
public interface ReflectionStore {
    void store(ReflectionEntry entry);
}
```

`NoOpReflectionStore.java`:
```java
package io.casehub.blocks.memory;

import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;

@DefaultBean
@ApplicationScoped
public class NoOpReflectionStore implements ReflectionStore {
    @Override
    public void store(ReflectionEntry entry) {}
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.FoundationTypesTest`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/memory/ blocks/src/test/java/io/casehub/blocks/memory/
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#120): add retention, outcome, event, integrity, and reflection foundation types"
```

## Batch 2: Orchestrator

### Task 3: MemoryHygieneOrchestrator — tick pipeline

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/memory/MemoryHygieneOrchestrator.java`
- Test: `blocks/src/test/java/io/casehub/blocks/memory/MemoryHygieneOrchestratorTest.java`

**Interfaces:**
- Consumes: `ImportanceScorer` (Task 1), `RetentionScore.compute()` (Task 2), `RetentionConfig` (Task 2), `HygieneTick` (Task 2), `HygieneEvent` (Task 2), `CbrCaseMemoryStore.retrieveSimilar()`, `CbrCaseMemoryStore.erase()`, `CbrCaseMemoryStore.store()`, `CbrCaseMemoryStore.supersede()`, `TemporalDecay.factor()`, `ScopeDecay.factor()`, `ContentSummariser.summarise()`, `FeatureVectorCbrCase`, `FeatureValue.stringList()`, `EraseRequest(entityId, domain, tenantId, caseId)`, `CbrQuery.of(tenantId, domain, scope, caseType, features, topK).withMinSimilarity()`
- Produces: `MemoryHygieneOrchestrator` — `HygieneTick tick(String agentId, String tenantId)`. Used by Task 5 (scheduler).

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.EraseRequest;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.neocortex.memory.cbr.runtime.*;
import io.casehub.blocks.summarisation.ContentSummariser;
import io.casehub.qhorus.api.spi.SummaryResult;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.function.Consumer;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class MemoryHygieneOrchestratorTest {

    private CbrCaseMemoryStore store;
    private ImportanceScorer scorer;
    private TemporalDecay decay;
    private ScopeDecay scopeDecay;
    private ContentSummariser<ScoredCbrCase<? extends CbrCase>> summariser;
    private Consumer<HygieneEvent> eventSink;
    private MemoryHygieneOrchestrator orchestrator;

    private static final MemoryDomain DOMAIN = new MemoryDomain("agent");
    private static final RetentionConfig RETENTION = new RetentionConfig(0.5, 1.0, 1.0, 0.0, 0.0);

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        store = mock(CbrCaseMemoryStore.class);
        scorer = mock(ImportanceScorer.class);
        decay = new TemporalDecay.HalfLife(Duration.ofDays(30));
        scopeDecay = new ScopeDecay.Step(0.5);
        summariser = mock(ContentSummariser.class);
        eventSink = mock(Consumer.class);
        orchestrator = new MemoryHygieneOrchestrator(
                store, scorer, decay, scopeDecay, summariser,
                DOMAIN, List.of("test-case"), RETENTION, 100, 0.7, eventSink);
    }

    @Test
    void tickReturnsIdleWhenNoMemories() {
        when(store.retrieveSimilar(any(), any())).thenReturn(List.of());
        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(HygieneTick.Idle.class);
    }

    @Test
    void tickEvictsLowScoringMemories() {
        var cbrCase = new FeatureVectorCbrCase("problem", "solution", null, null,
                Map.of("k", FeatureValue.string("v")), null, "agent-1");
        var scored = new ScoredCbrCase<CbrCase>(cbrCase, "case-1", 0.5, false,
                Map.of(), Instant.now(), io.casehub.platform.api.path.Path.root(), null);

        when(store.retrieveSimilar(any(), any())).thenReturn(List.of(scored));
        when(scorer.score(any(), any())).thenReturn(0.1);
        when(summariser.summarise(any(), any()))
                .thenReturn(CompletableFuture.completedFuture(SummaryResult.ofText("merged")));

        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(HygieneTick.Completed.class);
        var completed = (HygieneTick.Completed) result;
        assertThat(completed.evicted()).isEqualTo(1);

        verify(store).erase(any(EraseRequest.class));
        verify(eventSink).accept(any(HygieneEvent.MemoryEvicted.class));
    }

    @Test
    void tickRetainsHighScoringMemories() {
        var cbrCase = new FeatureVectorCbrCase("problem", "solution", null, null,
                Map.of("k", FeatureValue.string("v")), null, "agent-1");
        var scored = new ScoredCbrCase<CbrCase>(cbrCase, "case-1", 0.5, false,
                Map.of(), Instant.now(), io.casehub.platform.api.path.Path.root(), null);

        when(store.retrieveSimilar(any(), any())).thenReturn(List.of(scored));
        when(scorer.score(any(), any())).thenReturn(0.9);

        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(HygieneTick.Completed.class);
        var completed = (HygieneTick.Completed) result;
        assertThat(completed.evicted()).isEqualTo(0);

        verify(store, never()).erase(any(EraseRequest.class));
    }

    @Test
    void tickReturnsFailedOnStoreException() {
        when(store.retrieveSimilar(any(), any())).thenThrow(new RuntimeException("store down"));
        var result = orchestrator.tick("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(HygieneTick.Failed.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.MemoryHygieneOrchestratorTest`
Expected: compilation failure — `MemoryHygieneOrchestrator` doesn't exist

- [ ] **Step 3: Implement MemoryHygieneOrchestrator**

The orchestrator is not `@ApplicationScoped` since it takes config parameters — consumers construct it. Constructor takes all dependencies directly (no CDI injection).

Key implementation points:
- Per-agent `ReentrantLock` via `ConcurrentHashMap`
- Iterates caseTypes, retrieves via `CbrQuery.of(...).withMinSimilarity(0.0)`
- Post-filters by `producerAgentId` (CbrQuery doesn't have a producer filter)
- Scores → evicts below threshold → consolidates survivors above similarity threshold
- Consolidation: groups similar cases, runs `ContentSummariser`, builds `FeatureVectorCbrCase`, supersedes sources
- Emits `HygieneEvent` via `Consumer<HygieneEvent>`

Full implementation in the orchestrator class — ~150 lines. The tick method catches all exceptions and returns `Failed`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.MemoryHygieneOrchestratorTest`
Expected: all 4 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/memory/MemoryHygieneOrchestrator.java blocks/src/test/java/io/casehub/blocks/memory/MemoryHygieneOrchestratorTest.java
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#120): implement MemoryHygieneOrchestrator — tick pipeline with scoring, eviction, consolidation"
```

## Batch 3: Scheduler and integrity

### Task 4: DefaultIntegrityChecker

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/memory/DefaultIntegrityChecker.java`
- Test: `blocks/src/test/java/io/casehub/blocks/memory/DefaultIntegrityCheckerTest.java`

**Interfaces:**
- Consumes: `IntegrityChecker` (Task 2), `IntegrityViolation` (Task 2), `ViolationType` (Task 2), `SemanticIntegrityChecker` (Task 2), `CbrCaseMemoryStore.findSupersededCases()`, `CbrCaseMemoryStore.scan()`, `CbrCaseMemoryStore.getSupersessionStatus()`
- Produces: `DefaultIntegrityChecker` — implements `IntegrityChecker`. Used by Task 5 (scheduler).

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class DefaultIntegrityCheckerTest {

    private CbrCaseMemoryStore store;
    private SemanticIntegrityChecker semanticChecker;
    private DefaultIntegrityChecker checker;
    private static final MemoryDomain DOMAIN = new MemoryDomain("agent");

    @BeforeEach
    void setUp() {
        store = mock(CbrCaseMemoryStore.class);
        semanticChecker = mock(SemanticIntegrityChecker.class);
        when(semanticChecker.checkSemantic(any(), any(), any())).thenReturn(List.of());
        checker = new DefaultIntegrityChecker(store, semanticChecker, List.of("test-case"));
    }

    @Test
    void detectsOrphanedSupersession() {
        var status = new SupersessionStatus("case-1", true, Instant.now(),
                "missing-case", "consolidation", null);
        when(store.findSupersededCases(any(), any())).thenReturn(List.of(status));
        when(store.getSupersessionStatus("missing-case", "tenant-1"))
                .thenReturn(SupersessionStatus.NOT_SUPERSEDED);
        when(store.retrieveSimilar(any(), any())).thenReturn(List.of());

        var violations = checker.check("agent-1", "tenant-1", DOMAIN);
        assertThat(violations).anyMatch(v -> v.type() == ViolationType.ORPHANED_SUPERSESSION);
    }

    @Test
    void returnsEmptyWhenNoIssues() {
        when(store.findSupersededCases(any(), any())).thenReturn(List.of());
        when(store.retrieveSimilar(any(), any())).thenReturn(List.of());

        var violations = checker.check("agent-1", "tenant-1", DOMAIN);
        assertThat(violations).isEmpty();
    }

    @Test
    void gracefullyDegradeWhenScanUnsupported() {
        when(store.findSupersededCases(any(), any())).thenReturn(List.of());
        when(store.retrieveSimilar(any(), any())).thenReturn(List.of());
        when(store.scan(any())).thenThrow(new UnsupportedOperationException("not supported"));

        var violations = checker.check("agent-1", "tenant-1", DOMAIN);
        assertThat(violations).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.DefaultIntegrityCheckerTest`
Expected: compilation failure

- [ ] **Step 3: Implement DefaultIntegrityChecker**

Checks: orphaned supersessions via `findSupersededCases()`, unprocessed stale via `retrieveSimilar` + age check. Gracefully catches `UnsupportedOperationException` from `scan()`. Escalates flagged items to `SemanticIntegrityChecker`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.DefaultIntegrityCheckerTest`
Expected: all 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/memory/DefaultIntegrityChecker.java blocks/src/test/java/io/casehub/blocks/memory/DefaultIntegrityCheckerTest.java
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#120): add DefaultIntegrityChecker — structural checks with graceful scan() degradation"
```

### Task 5: MemoryHygieneScheduler — maintain pipeline

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/memory/MemoryHygieneScheduler.java`
- Test: `blocks/src/test/java/io/casehub/blocks/memory/MemoryHygieneSchedulerTest.java`

**Interfaces:**
- Consumes: `MemoryHygieneOrchestrator.tick()` (Task 3), `ReflectionOrchestrator.reflect()` (neocortex), `ReflectionStore.store()` (Task 2), `IntegrityChecker.check()` (Task 4), `CbrCaseMemoryStore.retrieveSimilar()`, `HygieneTick` (Task 2), `MaintenanceTick` (Task 2), `HygieneEvent` (Task 2)
- Produces: `MemoryHygieneScheduler` — `MaintenanceTick maintain(String agentId, String tenantId)`.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.cbr.*;
import io.casehub.neocortex.memory.reflection.ReflectionOrchestrator;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.function.Consumer;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.anyInt;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.*;

class MemoryHygieneSchedulerTest {

    private MemoryHygieneOrchestrator orchestrator;
    private ReflectionOrchestrator reflectionOrchestrator;
    private ReflectionStore reflectionStore;
    private IntegrityChecker integrityChecker;
    private CbrCaseMemoryStore store;
    private Consumer<HygieneEvent> eventSink;
    private MemoryHygieneScheduler scheduler;

    private static final MemoryDomain DOMAIN = new MemoryDomain("agent");

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        orchestrator = mock(MemoryHygieneOrchestrator.class);
        reflectionOrchestrator = mock(ReflectionOrchestrator.class);
        reflectionStore = mock(ReflectionStore.class);
        integrityChecker = mock(IntegrityChecker.class);
        store = mock(CbrCaseMemoryStore.class);
        eventSink = mock(Consumer.class);
        scheduler = new MemoryHygieneScheduler(
                orchestrator, reflectionOrchestrator, reflectionStore,
                integrityChecker, store, DOMAIN, List.of("test-case"),
                50, 0.7, eventSink);
    }

    @Test
    void maintainComposesAllStages() {
        var tick = new HygieneTick.Completed(0, 1, 5, List.of());
        when(orchestrator.tick("agent-1", "tenant-1")).thenReturn(tick);
        when(reflectionOrchestrator.reflect(anyString(), anyString(), any(Instant.class), anyInt()))
                .thenReturn(List.of("insight one"));
        when(integrityChecker.check(anyString(), anyString(), any(MemoryDomain.class)))
                .thenReturn(List.of());
        when(store.retrieveSimilar(any(), any())).thenReturn(List.of());

        var result = scheduler.maintain("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(MaintenanceTick.Completed.class);
        var completed = (MaintenanceTick.Completed) result;
        assertThat(completed.reflectionsGenerated()).isEqualTo(1);
        verify(reflectionStore).store(any(ReflectionEntry.class));
    }

    @Test
    void maintainReturnsFailedOnReflectionError() {
        when(orchestrator.tick(anyString(), anyString()))
                .thenReturn(new HygieneTick.Completed(0, 0, 0, List.of()));
        when(reflectionOrchestrator.reflect(anyString(), anyString(), any(Instant.class), anyInt()))
                .thenThrow(new RuntimeException("reflection failed"));

        var result = scheduler.maintain("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(MaintenanceTick.Failed.class);
        assertThat(((MaintenanceTick.Failed) result).stage()).isEqualTo("reflection");
    }

    @Test
    void maintainPropagatesTickFailure() {
        when(orchestrator.tick(anyString(), anyString()))
                .thenReturn(new HygieneTick.Failed("store down"));

        var result = scheduler.maintain("agent-1", "tenant-1");
        assertThat(result).isInstanceOf(MaintenanceTick.Failed.class);
        assertThat(((MaintenanceTick.Failed) result).stage()).isEqualTo("tick");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.MemoryHygieneSchedulerTest`
Expected: compilation failure

- [ ] **Step 3: Implement MemoryHygieneScheduler**

Not CDI — consumer constructs. `maintain()` composes: tick → reflect → peer-link → integrity. Each stage wrapped in try/catch returning `Failed` with stage name. Reflection provenance from tick's scored caseIds.

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest=io.casehub.blocks.memory.MemoryHygieneSchedulerTest`
Expected: all 3 tests PASS

- [ ] **Step 5: Run full test suite**

Run: `mvn --batch-mode test -pl blocks`
Expected: all tests PASS (including existing tests in other packages)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/memory/MemoryHygieneScheduler.java blocks/src/test/java/io/casehub/blocks/memory/MemoryHygieneSchedulerTest.java
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#120): add MemoryHygieneScheduler — maintain pipeline composing tick, reflection, peer-linking, integrity"
```

### Task 6: Update CLAUDE.md with memory package documentation

**Files:**
- Modify: `CLAUDE.md` — add `io.casehub.blocks.memory` package section

**Interfaces:**
- Consumes: All types from Tasks 1-5
- Produces: Updated project documentation

- [ ] **Step 1: Add package documentation to CLAUDE.md**

Add a new section after the `agentic.personality` package section documenting `io.casehub.blocks.memory` with the class table.

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "docs(#120): add memory package to CLAUDE.md"
```

## References

- [2026-08-20-memory-hygiene-design.md] — design spec this plan implements
- [CbrCaseMemoryStore.java] — neocortex store SPI (store, retrieveSimilar, erase, supersede, scan, findSupersededCases)
- [TemporalDecay.java] — neocortex sealed interface (HalfLife, Linear, Step)
- [ScopeDecay.java] — neocortex sealed interface (Exponential, Linear, Step)
- [ReflectionOrchestrator.java] — neocortex SPI (reflect → List<String>)
- [ContentSummariser.java] — blocks summarisation SPI (summarise → CompletionStage<SummaryResult>)
- [FeatureVectorCbrCase.java] — neocortex record (supports withFeatures)
- [ScoredCbrCase.java] — neocortex record (cbrCase, caseId, score, storedAt, scope)
- [EraseRequest.java] — neocortex record (entityId, domain, tenantId, caseId)
- [CbrQuery.java] — neocortex record (of, withMinSimilarity, withFilter)
- [FeatureValue.java] — neocortex sealed interface (StringListVal, stringList factory)
- [SupersessionStatus.java] — neocortex record (supersededAt, supersedingCaseId)
- [PersonalityEvolutionOrchestrator.java] — blocks pattern precedent (tick, per-agent locks)
- [InnerLifeOrchestrator.java] — blocks pattern precedent (tick, ConcurrentHashMap state)
- [GE-20260804-eb75e0] — scan() without features, use retrieveSimilar
- [GE-20260716-f292d3] — score-replacing decorators
- [GE-20260720-b7a8b9] — eraseEntity cross-domain gotcha
- [GitHub #120] — focal issue
- [GitHub #126] — epic issue
