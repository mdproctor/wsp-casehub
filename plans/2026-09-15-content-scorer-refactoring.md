# ContentScorer Refactoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #58 — Wire blocks scoring (SurpriseScorer/ArousalScorer) into consolidation importance
**Issue group:** #58

**Goal:** Introduce a domain-agnostic ContentScorer abstraction that lets blocks scoring heuristics (surprise, arousal) compose into neocortex consolidation graduation scoring.

**Architecture:** Three-layer change — new types in neocortex-memory-api (ScoreableContent, ContentScorer, WeightedContentScorer), dual-interface refactoring of existing scorers in blocks (SurpriseScorer, ArousalScorer implement both ContentScorer and ConfidenceScorer), and game-specific graduation scoring in examples (ActionImportanceScorer, ManorGraduationScorer).

**Tech Stack:** Java 26, Quarkus, Maven multi-module

## Global Constraints

- All scores return values in [0.0, 1.0], clamped
- ContentScorer is `@FunctionalInterface`
- No changes to ConfidenceScorer, GraduationScorer, or any of their existing consumers
- Dependency direction: neocortex-memory-api ← blocks ← examples
- Use `slot-settings.xml` for all Maven builds
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` for all commands

---

## Batch 1: Foundation — ScoreableContent + ContentScorer in neocortex-memory-api

### Task 1: ScoreableContent, ContentScorer, WeightedContentScorer

**Files:**
- Create: `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/experience/ScoreableContent.java`
- Create: `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/experience/ContentScorer.java`
- Create: `neocortex/memory-api/src/main/java/io/casehub/neocortex/memory/experience/WeightedContentScorer.java`
- Create: `neocortex/memory-api/src/test/java/io/casehub/neocortex/memory/experience/ScoreableContentTest.java`
- Create: `neocortex/memory-api/src/test/java/io/casehub/neocortex/memory/experience/ContentScorerTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.Memory` (existing record in memory-api)
- Produces: `ScoreableContent.fromMemory(Memory)`, `ContentScorer.score(ScoreableContent)`, `ContentScorer.composite(List<WeightedContentScorer>)`, `ContentScorer.weighted(ContentScorer, double)`, `WeightedContentScorer(ContentScorer, double)`

- [ ] **Step 1: Write ScoreableContent test**

```java
package io.casehub.neocortex.memory.experience;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class ScoreableContentTest {

    @Test
    void constructionWithAllFields() {
        var content = new ScoreableContent("hello", Map.of("k", "v"), Instant.EPOCH);
        assertThat(content.text()).isEqualTo("hello");
        assertThat(content.metadata()).containsEntry("k", "v");
        assertThat(content.timestamp()).isEqualTo(Instant.EPOCH);
    }

    @Test
    void nullTextThrows() {
        assertThatNullPointerException()
            .isThrownBy(() -> new ScoreableContent(null, Map.of(), Instant.EPOCH));
    }

    @Test
    void nullMetadataDefaultsToEmpty() {
        var content = new ScoreableContent("text", null, Instant.EPOCH);
        assertThat(content.metadata()).isEmpty();
    }

    @Test
    void metadataIsDefensivelyCopied() {
        var mutable = new java.util.HashMap<String, String>();
        mutable.put("a", "b");
        var content = new ScoreableContent("text", mutable, Instant.EPOCH);
        mutable.put("c", "d");
        assertThat(content.metadata()).doesNotContainKey("c");
    }

    @Test
    void fromMemory() {
        var memory = new Memory(
            "mem-1",
            io.casehub.neocortex.memory.Subject.of("agent", "entity-1"),
            io.casehub.neocortex.memory.MemoryDomain.of("test"),
            "tenant-1", "case-1", "event text",
            Map.of("event-type", "observation"),
            Instant.parse("2026-01-01T00:00:00Z"),
            null, null, null, null, null, null);

        var content = ScoreableContent.fromMemory(memory);
        assertThat(content.text()).isEqualTo("event text");
        assertThat(content.metadata()).containsEntry("event-type", "observation");
        assertThat(content.timestamp()).isEqualTo(Instant.parse("2026-01-01T00:00:00Z"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-api -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ScoreableContentTest -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml`
Expected: FAIL — ScoreableContent class does not exist

- [ ] **Step 3: Implement ScoreableContent**

```java
package io.casehub.neocortex.memory.experience;

import io.casehub.neocortex.memory.Memory;
import java.time.Instant;
import java.util.Map;
import java.util.Objects;

public record ScoreableContent(String text, Map<String, String> metadata, Instant timestamp) {
    public ScoreableContent {
        Objects.requireNonNull(text, "text required");
        metadata = metadata != null ? Map.copyOf(metadata) : Map.of();
    }

    public static ScoreableContent fromMemory(Memory memory) {
        return new ScoreableContent(
            memory.text(),
            memory.attributes(),
            memory.createdAt());
    }
}
```

- [ ] **Step 4: Run ScoreableContent tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-api -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ScoreableContentTest -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml`
Expected: PASS — all 5 tests green

- [ ] **Step 5: Write ContentScorer + WeightedContentScorer tests**

```java
package io.casehub.neocortex.memory.experience;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class ContentScorerTest {

    private static final ScoreableContent SAMPLE =
        new ScoreableContent("text", Map.of(), Instant.EPOCH);

    @Test
    void lambdaScorer() {
        ContentScorer scorer = c -> 0.42;
        assertThat(scorer.score(SAMPLE)).isCloseTo(0.42, within(0.001));
    }

    @Test
    void weightedCreation() {
        ContentScorer scorer = c -> 0.5;
        var weighted = ContentScorer.weighted(scorer, 0.3);
        assertThat(weighted.weight()).isEqualTo(0.3);
        assertThat(weighted.scorer().score(SAMPLE)).isCloseTo(0.5, within(0.001));
    }

    @Test
    void weightedRejectsNonPositiveWeight() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ContentScorer.weighted(c -> 0.5, 0.0));
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ContentScorer.weighted(c -> 0.5, -1.0));
    }

    @Test
    void weightedRejectsNullScorer() {
        assertThatNullPointerException()
            .isThrownBy(() -> ContentScorer.weighted(null, 1.0));
    }

    @Test
    void compositeWeightedMean() {
        ContentScorer fixed80 = c -> 0.8;
        ContentScorer fixed20 = c -> 0.2;
        var composite = ContentScorer.composite(List.of(
            ContentScorer.weighted(fixed80, 0.3),
            ContentScorer.weighted(fixed20, 0.7)));
        // (0.8*0.3 + 0.2*0.7) / (0.3+0.7) = (0.24 + 0.14) / 1.0 = 0.38
        assertThat(composite.score(SAMPLE)).isCloseTo(0.38, within(0.001));
    }

    @Test
    void compositeSingleScorerReturnsItsValue() {
        ContentScorer fixed = c -> 0.42;
        var composite = ContentScorer.composite(List.of(
            ContentScorer.weighted(fixed, 1.0)));
        assertThat(composite.score(SAMPLE)).isCloseTo(0.42, within(0.001));
    }

    @Test
    void compositeClamps() {
        ContentScorer overOne = c -> 1.5;
        var composite = ContentScorer.composite(List.of(
            ContentScorer.weighted(overOne, 1.0)));
        assertThat(composite.score(SAMPLE)).isEqualTo(1.0);
    }

    @Test
    void compositeRejectsEmpty() {
        assertThatIllegalArgumentException()
            .isThrownBy(() -> ContentScorer.composite(List.of()));
    }
}
```

- [ ] **Step 6: Implement ContentScorer and WeightedContentScorer**

WeightedContentScorer:
```java
package io.casehub.neocortex.memory.experience;

import java.util.Objects;

public record WeightedContentScorer(ContentScorer scorer, double weight) {
    public WeightedContentScorer {
        Objects.requireNonNull(scorer, "scorer required");
        if (weight <= 0.0) {
            throw new IllegalArgumentException("weight must be positive, got " + weight);
        }
    }
}
```

ContentScorer:
```java
package io.casehub.neocortex.memory.experience;

import java.util.List;
import java.util.Objects;

@FunctionalInterface
public interface ContentScorer {
    double score(ScoreableContent content);

    static WeightedContentScorer weighted(ContentScorer scorer, double weight) {
        return new WeightedContentScorer(scorer, weight);
    }

    static ContentScorer composite(List<WeightedContentScorer> scorers) {
        Objects.requireNonNull(scorers, "scorers required");
        if (scorers.isEmpty()) {
            throw new IllegalArgumentException("at least one scorer required");
        }
        var copy = List.copyOf(scorers);
        return content -> {
            double weightedSum = 0.0;
            double totalWeight = 0.0;
            for (var ws : copy) {
                weightedSum += ws.scorer().score(content) * ws.weight();
                totalWeight += ws.weight();
            }
            return Math.clamp(weightedSum / totalWeight, 0.0, 1.0);
        };
    }
}
```

- [ ] **Step 7: Run all ContentScorer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-api -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ContentScorerTest -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml`
Expected: PASS — all 8 tests green

- [ ] **Step 8: Install memory-api to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl memory-api -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add memory-api/src/main/java/io/casehub/neocortex/memory/experience/ScoreableContent.java memory-api/src/main/java/io/casehub/neocortex/memory/experience/ContentScorer.java memory-api/src/main/java/io/casehub/neocortex/memory/experience/WeightedContentScorer.java memory-api/src/test/java/io/casehub/neocortex/memory/experience/ScoreableContentTest.java memory-api/src/test/java/io/casehub/neocortex/memory/experience/ContentScorerTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#58): ScoreableContent + ContentScorer — domain-agnostic scoring abstraction in memory-api

Refs casehubio/examples#58"
```

---

## Batch 2: Dual-interface refactoring — SurpriseScorer + ArousalScorer in blocks

### Task 2: Refactor SurpriseScorer and ArousalScorer to implement ContentScorer

**Files:**
- Modify: `blocks/src/main/java/io/casehub/blocks/memory/SurpriseScorer.java`
- Modify: `blocks/src/main/java/io/casehub/blocks/memory/ArousalScorer.java`
- Modify: `blocks/src/test/java/io/casehub/blocks/memory/ConfidenceScorerTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.experience.ContentScorer` (from Task 1), `io.casehub.neocortex.memory.experience.ScoreableContent` (from Task 1)
- Produces: `SurpriseScorer implements ContentScorer, ConfidenceScorer`, `ArousalScorer implements ContentScorer, ConfidenceScorer`

- [ ] **Step 1: Write ContentScorer-interface tests for SurpriseScorer**

Add to the existing `ConfidenceScorerTest.java`:

```java
@Test
void surpriseScorer_contentScorer_emptyMetadata() {
    var scorer = new SurpriseScorer();
    var content = new ScoreableContent("text", Map.of(), Instant.EPOCH);
    assertThat(scorer.score(content)).isCloseTo(0.5, within(0.001));
}

@Test
void surpriseScorer_contentScorer_withMetadata() {
    var scorer = new SurpriseScorer();
    var content = new ScoreableContent("text",
        Map.of("feature1", "longvalue12345", "feature2", "another"),
        Instant.EPOCH);
    double score = scorer.score(content);
    assertThat(score).isBetween(0.0, 1.0);
    assertThat(score).isGreaterThan(0.0);
}

@Test
void surpriseScorer_dualInterface_consistency() {
    var scorer = new SurpriseScorer();
    var content = new ScoreableContent("problem solution",
        Map.of("key", "value"), Instant.EPOCH);
    double contentScore = scorer.score(content);
    assertThat(contentScore).isBetween(0.0, 1.0);
}
```

- [ ] **Step 2: Write ContentScorer-interface tests for ArousalScorer**

Add to `ConfidenceScorerTest.java`:

```java
@Test
void arousalScorer_contentScorer_noHighArousalWords() {
    var scorer = new ArousalScorer();
    var content = new ScoreableContent("a peaceful calm day",
        Map.of(), Instant.EPOCH);
    assertThat(scorer.score(content)).isCloseTo(0.0, within(0.001));
}

@Test
void arousalScorer_contentScorer_withHighArousalWords() {
    var scorer = new ArousalScorer();
    var content = new ScoreableContent("critical emergency alert failure",
        Map.of(), Instant.EPOCH);
    double score = scorer.score(content);
    assertThat(score).isGreaterThan(0.0);
    assertThat(score).isLessThanOrEqualTo(1.0);
}

@Test
void arousalScorer_dualInterface_consistency() {
    var scorer = new ArousalScorer();
    var content = new ScoreableContent("critical error in system",
        Map.of(), Instant.EPOCH);
    double contentScore = scorer.score(content);
    assertThat(contentScore).isBetween(0.0, 1.0);
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl . -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ConfidenceScorerTest -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`
Expected: FAIL — SurpriseScorer/ArousalScorer don't implement ContentScorer yet

- [ ] **Step 4: Refactor SurpriseScorer**

The class implements both `ContentScorer` and `ConfidenceScorer`. Real logic in `score(ScoreableContent)`. The `score(ScoredCbrCase, Instant)` method extracts ScoreableContent and delegates.

```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import io.casehub.neocortex.memory.cbr.FeatureValue;
import io.casehub.neocortex.memory.experience.ContentScorer;
import io.casehub.neocortex.memory.experience.ScoreableContent;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

public final class SurpriseScorer implements ContentScorer, ConfidenceScorer {

    @Override
    public double score(ScoreableContent content) {
        Map<String, String> metadata = content.metadata();
        if (metadata == null || metadata.isEmpty()) {return 0.5;}
        int distinctValues = 0;
        for (var value : metadata.values()) {
            distinctValues += value.length();
        }
        return Math.clamp(Math.log1p(distinctValues) / 10.0, 0.0, 1.0);
    }

    @Override
    public double score(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        return score(toScoreableContent(memory, now));
    }

    static ScoreableContent toScoreableContent(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        CbrCase c = memory.cbrCase();
        String text = c.problem();
        if (c.solution() != null) {
            text = text + " " + c.solution();
        }
        Map<String, String> metadata = new HashMap<>();
        if (c.features() != null) {
            for (var entry : c.features().entrySet()) {
                metadata.put(entry.getKey(), featureValueToString(entry.getValue()));
            }
        }
        Instant timestamp = memory.storedAt() != null ? memory.storedAt() : now;
        return new ScoreableContent(text != null ? text : "", metadata, timestamp);
    }

    private static String featureValueToString(FeatureValue fv) {
        if (fv instanceof FeatureValue.StringVal sv) {return sv.value();}
        if (fv instanceof FeatureValue.NumberVal nv) {return String.valueOf(nv.value());}
        if (fv instanceof FeatureValue.StringListVal sl) {return String.join(",", sl.values());}
        return fv.toString();
    }
}
```

- [ ] **Step 5: Refactor ArousalScorer**

```java
package io.casehub.blocks.memory;

import io.casehub.neocortex.memory.cbr.CbrCase;
import io.casehub.neocortex.memory.cbr.ScoredCbrCase;
import io.casehub.neocortex.memory.experience.ContentScorer;
import io.casehub.neocortex.memory.experience.ScoreableContent;

import java.time.Instant;
import java.util.Set;

public final class ArousalScorer implements ContentScorer, ConfidenceScorer {

    private static final Set<String> HIGH_AROUSAL = Set.of(
            "critical", "emergency", "urgent", "failure", "crisis", "error",
            "escalation", "breach", "violation", "fatal", "severe", "alarm",
            "panic", "catastrophe", "danger", "threat", "attack", "outage",
            "incident", "alert", "warning", "shutdown", "corrupt", "exploit");

    @Override
    public double score(ScoreableContent content) {
        var text = content.text().toLowerCase();
        var words = text.split("\\W+");
        if (words.length == 0) {return 0.0;}
        int hits = 0;
        for (var word : words) {
            if (HIGH_AROUSAL.contains(word)) {hits++;}
        }
        return Math.clamp((double) hits / words.length * 5.0, 0.0, 1.0);
    }

    @Override
    public double score(ScoredCbrCase<? extends CbrCase> memory, Instant now) {
        return score(SurpriseScorer.toScoreableContent(memory, now));
    }
}
```

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl . -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ConfidenceScorerTest -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`
Expected: PASS — all existing tests + 6 new tests green

- [ ] **Step 7: Run full blocks test suite to confirm no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl . -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`
Expected: BUILD SUCCESS — no regressions in MemoryHygieneOrchestrator, CompositeConfidenceScorer, etc.

- [ ] **Step 8: Install blocks to local Maven repo**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl . -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add src/main/java/io/casehub/blocks/memory/SurpriseScorer.java src/main/java/io/casehub/blocks/memory/ArousalScorer.java src/test/java/io/casehub/blocks/memory/ConfidenceScorerTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#58): SurpriseScorer/ArousalScorer dual-interface — implement ContentScorer + ConfidenceScorer adapter

Refs casehubio/examples#58"
```

---

## Batch 3: Game-specific scoring — ActionImportanceScorer + ManorGraduationScorer

### Task 3: ActionImportanceScorer and ManorGraduationScorer in wacky-manor

**Files:**
- Create: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/engine/ActionImportanceScorer.java`
- Create: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/engine/ManorGraduationScorer.java`
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/engine/ActionImportanceScorerTest.java`
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/engine/ManorGraduationScorerTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.experience.ContentScorer` (from Task 1), `io.casehub.neocortex.memory.experience.ScoreableContent` (from Task 1), `io.casehub.blocks.memory.ArousalScorer` (from Task 2), `io.casehub.neocortex.memory.experience.GraduationScorer` (existing), `io.casehub.neocortex.memory.Memory` (existing)
- Produces: `ActionImportanceScorer implements ContentScorer`, `ManorGraduationScorer implements GraduationScorer`

- [ ] **Step 1: Write ActionImportanceScorer test**

```java
package io.casehub.examples.manor.engine;

import io.casehub.neocortex.memory.experience.ScoreableContent;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class ActionImportanceScorerTest {

    private final ActionImportanceScorer scorer = new ActionImportanceScorer();

    @Test
    void conflictResolution_highScore() {
        var content = new ScoreableContent("resolved",
            Map.of("event-type", "conflict_resolution"), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.9, within(0.001));
    }

    @Test
    void trustChange_highScore() {
        var content = new ScoreableContent("trust shifted",
            Map.of("event-type", "trust_change"), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.8, within(0.001));
    }

    @Test
    void socialInteraction_mediumScore() {
        var content = new ScoreableContent("talked",
            Map.of("event-type", "social_interaction"), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.6, within(0.001));
    }

    @Test
    void observation_lowScore() {
        var content = new ScoreableContent("saw something",
            Map.of("event-type", "observation"), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.4, within(0.001));
    }

    @Test
    void idle_veryLowScore() {
        var content = new ScoreableContent("nothing",
            Map.of("event-type", "idle"), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.1, within(0.001));
    }

    @Test
    void unknownEventType_defaultScore() {
        var content = new ScoreableContent("something",
            Map.of("event-type", "unexpected_type"), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.3, within(0.001));
    }

    @Test
    void missingEventType_defaultScore() {
        var content = new ScoreableContent("no type", Map.of(), Instant.EPOCH);
        assertThat(scorer.score(content)).isCloseTo(0.3, within(0.001));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ActionImportanceScorerTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement ActionImportanceScorer**

```java
package io.casehub.examples.manor.engine;

import io.casehub.neocortex.memory.experience.ContentScorer;
import io.casehub.neocortex.memory.experience.ScoreableContent;
import java.util.Map;

public final class ActionImportanceScorer implements ContentScorer {

    private static final Map<String, Double> EVENT_WEIGHTS = Map.of(
        "conflict_resolution", 0.9,
        "trust_change", 0.8,
        "social_interaction", 0.6,
        "observation", 0.4,
        "idle", 0.1);

    private static final double DEFAULT_WEIGHT = 0.3;

    @Override
    public double score(ScoreableContent content) {
        String eventType = content.metadata().getOrDefault("event-type", "unknown");
        return EVENT_WEIGHTS.getOrDefault(eventType, DEFAULT_WEIGHT);
    }
}
```

- [ ] **Step 4: Run ActionImportanceScorer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ActionImportanceScorerTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: PASS — all 7 tests green

- [ ] **Step 5: Write ManorGraduationScorer test**

```java
package io.casehub.examples.manor.engine;

import io.casehub.neocortex.memory.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;

import static org.assertj.core.api.Assertions.*;

class ManorGraduationScorerTest {

    private final ManorGraduationScorer scorer = new ManorGraduationScorer();

    @Test
    void highArousal_highImportance_highConfidence() {
        var memory = new Memory("m1", Subject.of("agent", "e1"),
            MemoryDomain.of("test"), "t1", "c1",
            "critical emergency failure alert",
            Map.of("event-type", "conflict_resolution"),
            Instant.EPOCH, Confidence.of(0.9), null, null, null, null, null);
        double score = scorer.score(memory);
        assertThat(score).isBetween(0.5, 1.0);
    }

    @Test
    void lowArousal_lowImportance_lowConfidence() {
        var memory = new Memory("m2", Subject.of("agent", "e1"),
            MemoryDomain.of("test"), "t1", "c1",
            "a peaceful calm day passed",
            Map.of("event-type", "idle"),
            Instant.EPOCH, Confidence.of(0.1), null, null, null, null, null);
        double score = scorer.score(memory);
        assertThat(score).isBetween(0.0, 0.3);
    }

    @Test
    void nullConfidence_fallbackToHalf() {
        var memory = new Memory("m3", Subject.of("agent", "e1"),
            MemoryDomain.of("test"), "t1", "c1",
            "critical failure detected",
            Map.of("event-type", "trust_change"),
            Instant.EPOCH, null, null, null, null, null, null);
        double score = scorer.score(memory);
        assertThat(score).isBetween(0.0, 1.0);
    }

    @Test
    void formulaVerification() {
        // arousal=0, actionImportance=0.9 (conflict_resolution), confidence=1.0
        // (0*0.3 + 0.9*0.4) / 0.7 * 0.7 + 1.0*0.3 = 0.36 + 0.3 = 0.66
        var memory = new Memory("m4", Subject.of("agent", "e1"),
            MemoryDomain.of("test"), "t1", "c1",
            "peaceful conflict resolution",
            Map.of("event-type", "conflict_resolution"),
            Instant.EPOCH, Confidence.of(1.0), null, null, null, null, null);
        double score = scorer.score(memory);
        assertThat(score).isCloseTo(0.66, within(0.05));
    }

    @Test
    void scoreAlwaysClamped() {
        var memory = new Memory("m5", Subject.of("agent", "e1"),
            MemoryDomain.of("test"), "t1", "c1", "text",
            Map.of("event-type", "observation"),
            Instant.EPOCH, Confidence.of(0.5), null, null, null, null, null);
        double score = scorer.score(memory);
        assertThat(score).isBetween(0.0, 1.0);
    }
}
```

- [ ] **Step 6: Implement ManorGraduationScorer**

```java
package io.casehub.examples.manor.engine;

import io.casehub.blocks.memory.ArousalScorer;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.experience.ContentScorer;
import io.casehub.neocortex.memory.experience.GraduationScorer;
import io.casehub.neocortex.memory.experience.ScoreableContent;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;

@ApplicationScoped
public class ManorGraduationScorer implements GraduationScorer {

    private final ContentScorer compositeScorer;

    public ManorGraduationScorer() {
        this.compositeScorer = ContentScorer.composite(List.of(
            ContentScorer.weighted(new ArousalScorer(), 0.3),
            ContentScorer.weighted(new ActionImportanceScorer(), 0.4)));
    }

    @Override
    public double score(Memory memory) {
        ScoreableContent content = ScoreableContent.fromMemory(memory);
        double contentScore = compositeScorer.score(content);
        double confidence = memory.confidence() != null ? memory.confidence().value() : 0.5;
        return Math.clamp(contentScore * 0.7 + confidence * 0.3, 0.0, 1.0);
    }
}
```

- [ ] **Step 7: Run ManorGraduationScorer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ManorGraduationScorerTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: PASS — all 5 tests green

- [ ] **Step 8: Run full wacky-manor test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: BUILD SUCCESS — ManorGraduationScorer overrides DefaultGraduationScorer via CDI, no regressions

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/main/java/io/casehub/examples/manor/engine/ActionImportanceScorer.java wacky-manor/src/main/java/io/casehub/examples/manor/engine/ManorGraduationScorer.java wacky-manor/src/test/java/io/casehub/examples/manor/engine/ActionImportanceScorerTest.java wacky-manor/src/test/java/io/casehub/examples/manor/engine/ManorGraduationScorerTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#58): ManorGraduationScorer — game-specific consolidation scoring

Composes ArousalScorer (0.3) + ActionImportanceScorer (0.4) + confidence (0.3)
for Tier 2 → Tier 3 graduation importance.

Refs #58"
```

---

## References

- [2026-09-15-content-scorer-refactoring-design.md] — design spec this plan implements
- `io.casehub.blocks.memory.ConfidenceScorer` — existing interface preserved as adapter
- `io.casehub.neocortex.memory.experience.GraduationScorer` — existing SPI consumed by ManorGraduationScorer
- `io.casehub.blocks.memory.SurpriseScorer` — refactored to dual-interface
- `io.casehub.blocks.memory.ArousalScorer` — refactored to dual-interface
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ExperienceConsolidationPhase` — consumer of GraduationScorer (unchanged)
- `io.casehub.neocortex.mindmap.intelligence.consolidation.DefaultGraduationScorer` — @DefaultBean fallback (overridden by ManorGraduationScorer)
- [GE-20260912-be7c74] — three-tier memory model: Tier 2 → Tier 3 graduation via importance scoring
- [GE-20260820-aa31ab] — composite retention score masking with weighted means
- [GitHub #58] — Wire blocks scoring into consolidation importance
- [GitHub #53] — Phase B: mindmap-native cognitive integration (parent)
