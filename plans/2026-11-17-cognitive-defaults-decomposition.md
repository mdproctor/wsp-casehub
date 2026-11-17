# CognitiveDefaults Structural Decomposition — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#999 — refactor: CognitiveDefaults structural decomposition into nested config groups
**Issue group:** #999

**Goal:** Decompose the 23-field flat CognitiveDefaults record into 4 nested config groups plus identity fields, reducing constructor args from 23 to 8 and eliminating 20 boilerplate withX() method bodies.

**Architecture:** Create 4 new group records (EmotionDefaults, CognitiveResourceDefaults, GraphDefaults, BehavioralDefaults) each with mergeWith() methods, following the existing PersonalityProfile nesting pattern. CognitiveDefaults keeps leaf-level convenience accessors and withX() methods that delegate through the groups, so the ~140 test call sites using `CognitiveDefaults.empty("id").withX()` chains continue to compile unchanged. CDE's derive() and deriveAndMerge() construct groups directly. YAML profiles nest fields under group keys (only 2 files; wacky-manor derives everything).

**Tech Stack:** Java 26, Jackson YAML, JUnit 5, AssertJ

## Global Constraints

- All new records in package `io.casehub.neocortex.cognitive.index` (same as CognitiveDefaults)
- CognitiveCapacityConfig, CognitiveResourcePoolConfig, SearchBudgetConfig, RecurringThoughtConfig, HabituationConfig live in `cognitive-api` module — the new group records import them, do not move them
- mergeWith() pattern: `this` is explicit (YAML-specified), parameter is derived; null-coalescence per leaf field
- Leaf-level convenience withX() and accessor methods MUST remain on CognitiveDefaults — 42+ test call sites depend on them
- No production code outside CDE constructs CognitiveDefaults directly — only empty() + withX() chains
- Use `ide_insert_member` for new methods, `ide_replace_member` for existing method bodies, `ide_create_file` for new files

---

## Batch 1: Group records [compiles and tests pass independently]

### Task 1: Create EmotionDefaults record

**Files:**
- Create: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/EmotionDefaults.java`
- Test: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/EmotionDefaultsTest.java`

**Interfaces:**
- Consumes: `MoodBaseline` (memory-api), `AppraisalWeights` (mindmap), `RegulationDefaults` (cognitive.index), `CopingDefaults` (cognitive.index)
- Produces: `EmotionDefaults` record with `mergeWith(EmotionDefaults derived)`, `empty()` factory. Used by Task 4 (CognitiveDefaults) and Task 5 (CDE).

- [ ] **Step 1: Write tests for EmotionDefaults**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.memory.mood.MoodBaseline;
import io.casehub.neocortex.mindmap.AppraisalWeights;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class EmotionDefaultsTest {

    @Test
    void empty_allFieldsNull() {
        var empty = EmotionDefaults.empty();
        assertThat(empty.moodBaseline()).isNull();
        assertThat(empty.appraisalWeights()).isNull();
        assertThat(empty.regulationDefaults()).isNull();
        assertThat(empty.copingDefaults()).isNull();
    }

    @Test
    void mergeWith_explicitWins() {
        var explicitMood = new MoodBaseline(0.5, 0.3, 0.1);
        var derivedMood = new MoodBaseline(-0.2, 0.0, 0.4);
        var derivedRegulation = new RegulationDefaults(0.6, 0.4, 0.3, 0.5);

        var explicit = new EmotionDefaults(explicitMood, null, null, null);
        var derived = new EmotionDefaults(derivedMood, null, derivedRegulation, null);

        var merged = explicit.mergeWith(derived);
        assertThat(merged.moodBaseline()).isSameAs(explicitMood);
        assertThat(merged.regulationDefaults()).isSameAs(derivedRegulation);
    }

    @Test
    void mergeWith_bothNull_returnsNull() {
        var merged = EmotionDefaults.empty().mergeWith(EmotionDefaults.empty());
        assertThat(merged.moodBaseline()).isNull();
        assertThat(merged.regulationDefaults()).isNull();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -Dtest="EmotionDefaultsTest" -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: compilation failure (EmotionDefaults doesn't exist)

- [ ] **Step 3: Create EmotionDefaults record**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.memory.mood.MoodBaseline;
import io.casehub.neocortex.mindmap.AppraisalWeights;

public record EmotionDefaults(
        MoodBaseline moodBaseline,
        AppraisalWeights appraisalWeights,
        RegulationDefaults regulationDefaults,
        CopingDefaults copingDefaults
) {
    public static EmotionDefaults empty() {
        return new EmotionDefaults(null, null, null, null);
    }

    public EmotionDefaults mergeWith(EmotionDefaults derived) {
        if (derived == null) return this;
        return new EmotionDefaults(
                moodBaseline != null ? moodBaseline : derived.moodBaseline(),
                appraisalWeights != null ? appraisalWeights : derived.appraisalWeights(),
                regulationDefaults != null ? regulationDefaults : derived.regulationDefaults(),
                copingDefaults != null ? copingDefaults : derived.copingDefaults());
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -Dtest="EmotionDefaultsTest" -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: 3 tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/EmotionDefaults.java cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/EmotionDefaultsTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#999): add EmotionDefaults config group record  Refs casehubio/neocortex#999"
```

### Task 2: Create CognitiveResourceDefaults record

**Files:**
- Create: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveResourceDefaults.java`
- Test: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveResourceDefaultsTest.java`

**Interfaces:**
- Consumes: `CognitiveCapacityConfig` (cognitive-api), `CognitiveResourcePoolConfig` (cognitive-api), `SearchBudgetConfig` (cognitive-api), `RecurringThoughtConfig` (cognitive-api), `HabituationConfig` (cognitive-api)
- Produces: `CognitiveResourceDefaults` record with `mergeWith()`, `empty()`. Note: `capacityConfig` merge uses the existing `CognitiveCapacityConfig.mergeWith()` for sub-field-level merge, while other fields use null-coalescence.

- [ ] **Step 1: Write tests for CognitiveResourceDefaults**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.cognitive.CognitiveCapacityConfig;
import io.casehub.neocortex.cognitive.CognitiveResourcePoolConfig;
import io.casehub.neocortex.cognitive.HabituationConfig;
import io.casehub.neocortex.cognitive.RecurringThoughtConfig;
import io.casehub.neocortex.cognitive.SearchBudgetConfig;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class CognitiveResourceDefaultsTest {

    @Test
    void empty_allFieldsNull() {
        var empty = CognitiveResourceDefaults.empty();
        assertThat(empty.capacityConfig()).isNull();
        assertThat(empty.cognitiveResourcePoolConfig()).isNull();
        assertThat(empty.searchBudgetConfig()).isNull();
        assertThat(empty.recurringThoughtConfig()).isNull();
        assertThat(empty.habituationConfig()).isNull();
    }

    @Test
    void mergeWith_explicitWins() {
        var explicitCapacity = new CognitiveCapacityConfig(0.9, null, null);
        var derivedCapacity = new CognitiveCapacityConfig(0.5, 0.6, 0.7);
        var derivedPool = CognitiveResourcePoolConfig.defaults();

        var explicit = new CognitiveResourceDefaults(explicitCapacity, null, null, null, null);
        var derived = new CognitiveResourceDefaults(derivedCapacity, derivedPool, null, null, null);

        var merged = explicit.mergeWith(derived);
        // capacityConfig uses sub-field merge via CognitiveCapacityConfig.mergeWith()
        assertThat(merged.capacityConfig().relationalComplexity()).isEqualTo(0.9);
        assertThat(merged.capacityConfig().associativeReach()).isEqualTo(0.6);
        assertThat(merged.capacityConfig().workingMemorySpan()).isEqualTo(0.7);
        // other fields use null-coalescence
        assertThat(merged.cognitiveResourcePoolConfig()).isSameAs(derivedPool);
    }

    @Test
    void mergeWith_capacityBothNull() {
        var merged = CognitiveResourceDefaults.empty().mergeWith(CognitiveResourceDefaults.empty());
        assertThat(merged.capacityConfig()).isNull();
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -Dtest="CognitiveResourceDefaultsTest" -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: compilation failure

- [ ] **Step 3: Create CognitiveResourceDefaults record**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.cognitive.CognitiveCapacityConfig;
import io.casehub.neocortex.cognitive.CognitiveResourcePoolConfig;
import io.casehub.neocortex.cognitive.HabituationConfig;
import io.casehub.neocortex.cognitive.RecurringThoughtConfig;
import io.casehub.neocortex.cognitive.SearchBudgetConfig;

public record CognitiveResourceDefaults(
        CognitiveCapacityConfig capacityConfig,
        CognitiveResourcePoolConfig cognitiveResourcePoolConfig,
        SearchBudgetConfig searchBudgetConfig,
        RecurringThoughtConfig recurringThoughtConfig,
        HabituationConfig habituationConfig
) {
    public static CognitiveResourceDefaults empty() {
        return new CognitiveResourceDefaults(null, null, null, null, null);
    }

    public CognitiveResourceDefaults mergeWith(CognitiveResourceDefaults derived) {
        if (derived == null) return this;
        return new CognitiveResourceDefaults(
                mergeCapacity(capacityConfig, derived.capacityConfig()),
                cognitiveResourcePoolConfig != null ? cognitiveResourcePoolConfig : derived.cognitiveResourcePoolConfig(),
                searchBudgetConfig != null ? searchBudgetConfig : derived.searchBudgetConfig(),
                recurringThoughtConfig != null ? recurringThoughtConfig : derived.recurringThoughtConfig(),
                habituationConfig != null ? habituationConfig : derived.habituationConfig());
    }

    private static CognitiveCapacityConfig mergeCapacity(CognitiveCapacityConfig explicit, CognitiveCapacityConfig derived) {
        if (explicit == null) return derived;
        if (derived == null) return explicit;
        return explicit.mergeWith(derived);
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -Dtest="CognitiveResourceDefaultsTest" -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: 3 tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveResourceDefaults.java cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveResourceDefaultsTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#999): add CognitiveResourceDefaults config group record  Refs casehubio/neocortex#999"
```

### Task 3: Create GraphDefaults and BehavioralDefaults records

**Files:**
- Create: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/GraphDefaults.java`
- Create: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/BehavioralDefaults.java`
- Test: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/GraphDefaultsTest.java`
- Test: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/BehavioralDefaultsTest.java`

**Interfaces:**
- GraphDefaults consumes: `MindMapVocabulary` (mindmap), `DeclarativeTraitRule` (mindmap), `DeclarativeDerivedEdgeRule` (mindmap), `GraphStructureDefaults` (cognitive.index), `ExtractionBiasDefaults` (cognitive.index)
- GraphDefaults produces: record with `mergeWith()`, `empty()`. Special merge: vocabulary, services, traitRules, derivedEdgeRules always come from explicit (never derived) — matches existing CDE behavior.
- BehavioralDefaults consumes: `CuriosityConfig` (mindmap), `TemporalFocusConfig` (cognitive.index), `SocialCognitionDefaults` (cognitive.index), `CbrStrategyDefaults` (cognitive.index)
- BehavioralDefaults produces: record with `mergeWith()`, `empty()`.

- [ ] **Step 1: Write tests for GraphDefaults**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.mindmap.MindMapVocabulary;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class GraphDefaultsTest {

    @Test
    void empty_allFieldsNull_servicesEmpty() {
        var empty = GraphDefaults.empty();
        assertThat(empty.vocabulary()).isNull();
        assertThat(empty.services()).isEmpty();
        assertThat(empty.traitRules()).isNull();
        assertThat(empty.derivedEdgeRules()).isNull();
        assertThat(empty.graphStructure()).isNull();
        assertThat(empty.extractionBias()).isNull();
    }

    @Test
    void mergeWith_explicitOnlyForNonDerived() {
        var derivedGraph = GraphStructureDefaults.defaults();
        var derivedExtraction = ExtractionBiasDefaults.defaults();

        var explicit = new GraphDefaults(null, Map.of("k", "v"), null, null, null, null);
        var derived = new GraphDefaults(null, Map.of(), null, null, derivedGraph, derivedExtraction);

        var merged = explicit.mergeWith(derived);
        // vocabulary, services, traitRules, derivedEdgeRules come from explicit
        assertThat(merged.services()).containsEntry("k", "v");
        // graphStructure, extractionBias use explicit-wins
        assertThat(merged.graphStructure()).isSameAs(derivedGraph);
        assertThat(merged.extractionBias()).isSameAs(derivedExtraction);
    }

    @Test
    void services_defensivelyCopied() {
        var gd = new GraphDefaults(null, Map.of("a", "b"), null, null, null, null);
        assertThat(gd.services()).isUnmodifiable();
    }
}
```

- [ ] **Step 2: Write tests for BehavioralDefaults**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.mindmap.CuriosityConfig;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class BehavioralDefaultsTest {

    @Test
    void empty_allFieldsNull() {
        var empty = BehavioralDefaults.empty();
        assertThat(empty.curiosity()).isNull();
        assertThat(empty.temporalFocus()).isNull();
        assertThat(empty.socialCognition()).isNull();
        assertThat(empty.cbrStrategy()).isNull();
    }

    @Test
    void mergeWith_explicitWins() {
        var explicitCuriosity = CuriosityConfig.defaults();
        var derivedSocial = SocialCognitionDefaults.defaults();

        var explicit = new BehavioralDefaults(explicitCuriosity, null, null, null);
        var derived = new BehavioralDefaults(null, null, derivedSocial, null);

        var merged = explicit.mergeWith(derived);
        assertThat(merged.curiosity()).isSameAs(explicitCuriosity);
        assertThat(merged.socialCognition()).isSameAs(derivedSocial);
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -Dtest="GraphDefaultsTest,BehavioralDefaultsTest" -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: compilation failure

- [ ] **Step 4: Create GraphDefaults record**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.mindmap.DeclarativeDerivedEdgeRule;
import io.casehub.neocortex.mindmap.DeclarativeTraitRule;
import io.casehub.neocortex.mindmap.MindMapVocabulary;

import java.util.List;
import java.util.Map;

public record GraphDefaults(
        MindMapVocabulary vocabulary,
        Map<String, String> services,
        List<DeclarativeTraitRule> traitRules,
        List<DeclarativeDerivedEdgeRule> derivedEdgeRules,
        GraphStructureDefaults graphStructure,
        ExtractionBiasDefaults extractionBias
) {
    public GraphDefaults {
        services = services != null ? Map.copyOf(services) : Map.of();
        traitRules = traitRules != null ? List.copyOf(traitRules) : null;
        derivedEdgeRules = derivedEdgeRules != null ? List.copyOf(derivedEdgeRules) : null;
    }

    public static GraphDefaults empty() {
        return new GraphDefaults(null, Map.of(), null, null, null, null);
    }

    public GraphDefaults mergeWith(GraphDefaults derived) {
        if (derived == null) return this;
        return new GraphDefaults(
                vocabulary,
                services,
                traitRules,
                derivedEdgeRules,
                graphStructure != null ? graphStructure : derived.graphStructure(),
                extractionBias != null ? extractionBias : derived.extractionBias());
    }
}
```

- [ ] **Step 5: Create BehavioralDefaults record**

```java
package io.casehub.neocortex.cognitive.index;

import io.casehub.neocortex.mindmap.CuriosityConfig;

public record BehavioralDefaults(
        CuriosityConfig curiosity,
        TemporalFocusConfig temporalFocus,
        SocialCognitionDefaults socialCognition,
        CbrStrategyDefaults cbrStrategy
) {
    public static BehavioralDefaults empty() {
        return new BehavioralDefaults(null, null, null, null);
    }

    public BehavioralDefaults mergeWith(BehavioralDefaults derived) {
        if (derived == null) return this;
        return new BehavioralDefaults(
                curiosity != null ? curiosity : derived.curiosity(),
                temporalFocus != null ? temporalFocus : derived.temporalFocus(),
                socialCognition != null ? socialCognition : derived.socialCognition(),
                cbrStrategy != null ? cbrStrategy : derived.cbrStrategy());
    }
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -Dtest="GraphDefaultsTest,BehavioralDefaultsTest" -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: 5 tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/GraphDefaults.java cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/BehavioralDefaults.java cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/GraphDefaultsTest.java cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/BehavioralDefaultsTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#999): add GraphDefaults and BehavioralDefaults config group records  Refs casehubio/neocortex#999"
```

---

## Batch 2: Core refactoring [CognitiveDefaults, CDE, YAML]

### Task 4: Refactor CognitiveDefaults record to use nested groups

**Files:**
- Modify: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDefaults.java`

**Interfaces:**
- Consumes: EmotionDefaults, CognitiveResourceDefaults, GraphDefaults, BehavioralDefaults (from Tasks 1-3)
- Produces: Refactored CognitiveDefaults with 8 record components instead of 23. All existing accessor methods and withX() methods preserved as delegates. `empty()` simplified.

The record changes from 23 flat components to 8:
```
agentId, tenantId, personalityProfile, descriptor,
emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults
```

All 20 existing withX() methods become delegates through the group, and all leaf accessors (e.g., `moodBaseline()`, `curiosity()`) become explicit delegate methods. This is the SAME pattern already used for `personality()`, `personalityDominance()`, and `expressionStyles()`.

**IMPORTANT:** This task changes the record structure, which means Tasks 4, 5, and 6 must all complete before the module can compile. They form one atomic change within Batch 2.

- [ ] **Step 1: Rewrite CognitiveDefaults record**

Replace the entire record definition. The new record has:
1. 8 record components (down from 23)
2. Compact constructor validates agentId only (services/rules validation moves to GraphDefaults)
3. `empty()` factory with 8 args (5 nulls + 3 group empties)
4. All leaf accessor methods as delegates (e.g., `moodBaseline()` delegates to `emotionDefaults.moodBaseline()`)
5. All withX() methods as delegates (e.g., `withMoodBaseline(mb)` constructs a new EmotionDefaults and calls `withEmotionDefaults()`)
6. Group-level withX() methods (e.g., `withEmotionDefaults(ed)` reconstructs the 8-arg record)

```java
public record CognitiveDefaults(
        String agentId,
        String tenantId,
        PersonalityProfile personalityProfile,
        DescriptorView descriptor,
        EmotionDefaults emotionDefaults,
        CognitiveResourceDefaults cognitiveResourceDefaults,
        GraphDefaults graphDefaults,
        BehavioralDefaults behavioralDefaults
) {
    public CognitiveDefaults {
        Objects.requireNonNull(agentId, "agentId required");
        if (emotionDefaults == null) emotionDefaults = EmotionDefaults.empty();
        if (cognitiveResourceDefaults == null) cognitiveResourceDefaults = CognitiveResourceDefaults.empty();
        if (graphDefaults == null) graphDefaults = GraphDefaults.empty();
        if (behavioralDefaults == null) behavioralDefaults = BehavioralDefaults.empty();
    }

    public static CognitiveDefaults empty(String agentId) {
        return new CognitiveDefaults(agentId, null, null, null, null, null, null, null);
    }

    // --- Leaf accessors (delegates) ---

    public PersonalityWeights personality() {
        return personalityProfile != null ? personalityProfile.personality() : null;
    }

    public Double personalityDominance() {
        return personalityProfile != null ? personalityProfile.personalityDominance() : null;
    }

    public ExpressionStyleDefaults expressionStyles() {
        return personalityProfile != null ? personalityProfile.expressionStyles() : null;
    }

    // Emotion group
    public MoodBaseline moodBaseline() { return emotionDefaults.moodBaseline(); }
    public AppraisalWeights appraisalWeights() { return emotionDefaults.appraisalWeights(); }
    public RegulationDefaults regulationDefaults() { return emotionDefaults.regulationDefaults(); }
    public CopingDefaults copingDefaults() { return emotionDefaults.copingDefaults(); }

    // Resource group
    public CognitiveCapacityConfig capacityConfig() { return cognitiveResourceDefaults.capacityConfig(); }
    public CognitiveResourcePoolConfig cognitiveResourcePoolConfig() { return cognitiveResourceDefaults.cognitiveResourcePoolConfig(); }
    public SearchBudgetConfig searchBudgetConfig() { return cognitiveResourceDefaults.searchBudgetConfig(); }
    public RecurringThoughtConfig recurringThoughtConfig() { return cognitiveResourceDefaults.recurringThoughtConfig(); }
    public HabituationConfig habituationConfig() { return cognitiveResourceDefaults.habituationConfig(); }

    // Graph group
    public MindMapVocabulary vocabulary() { return graphDefaults.vocabulary(); }
    public Map<String, String> services() { return graphDefaults.services(); }
    public List<DeclarativeTraitRule> traitRules() { return graphDefaults.traitRules(); }
    public List<DeclarativeDerivedEdgeRule> derivedEdgeRules() { return graphDefaults.derivedEdgeRules(); }
    public GraphStructureDefaults graphStructure() { return graphDefaults.graphStructure(); }
    public ExtractionBiasDefaults extractionBias() { return graphDefaults.extractionBias(); }

    // Behavioral group
    public CuriosityConfig curiosity() { return behavioralDefaults.curiosity(); }
    public TemporalFocusConfig temporalFocus() { return behavioralDefaults.temporalFocus(); }
    public SocialCognitionDefaults socialCognition() { return behavioralDefaults.socialCognition(); }
    public CbrStrategyDefaults cbrStrategy() { return behavioralDefaults.cbrStrategy(); }

    // --- Group-level withX() methods ---

    public CognitiveDefaults withTenantId(String tenantId) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    public CognitiveDefaults withPersonalityProfile(PersonalityProfile personalityProfile) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    public CognitiveDefaults withDescriptor(DescriptorView descriptor) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    public CognitiveDefaults withEmotionDefaults(EmotionDefaults emotionDefaults) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    public CognitiveDefaults withCognitiveResourceDefaults(CognitiveResourceDefaults cognitiveResourceDefaults) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    public CognitiveDefaults withGraphDefaults(GraphDefaults graphDefaults) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    public CognitiveDefaults withBehavioralDefaults(BehavioralDefaults behavioralDefaults) {
        return new CognitiveDefaults(agentId, tenantId, personalityProfile, descriptor, emotionDefaults, cognitiveResourceDefaults, graphDefaults, behavioralDefaults);
    }

    // --- Leaf-level withX() methods (delegates) ---

    public CognitiveDefaults withPersonality(PersonalityWeights personality) {
        var pp = personalityProfile != null ? personalityProfile : new PersonalityProfile(null, null, null, null, List.of());
        return withPersonalityProfile(new PersonalityProfile(personality, pp.personalityDominance(), pp.expressionStyles(), pp.attachment(), pp.facets()));
    }

    public CognitiveDefaults withPersonalityDominance(Double personalityDominance) {
        var pp = personalityProfile != null ? personalityProfile : new PersonalityProfile(null, null, null, null, List.of());
        return withPersonalityProfile(new PersonalityProfile(pp.personality(), personalityDominance, pp.expressionStyles(), pp.attachment(), pp.facets()));
    }

    public CognitiveDefaults withExpressionStyles(ExpressionStyleDefaults expressionStyles) {
        var pp = personalityProfile != null ? personalityProfile : new PersonalityProfile(null, null, null, null, List.of());
        return withPersonalityProfile(new PersonalityProfile(pp.personality(), pp.personalityDominance(), expressionStyles, pp.attachment(), pp.facets()));
    }

    // Emotion delegates
    public CognitiveDefaults withMoodBaseline(MoodBaseline moodBaseline) {
        return withEmotionDefaults(new EmotionDefaults(moodBaseline, emotionDefaults.appraisalWeights(), emotionDefaults.regulationDefaults(), emotionDefaults.copingDefaults()));
    }

    public CognitiveDefaults withAppraisalWeights(AppraisalWeights appraisalWeights) {
        return withEmotionDefaults(new EmotionDefaults(emotionDefaults.moodBaseline(), appraisalWeights, emotionDefaults.regulationDefaults(), emotionDefaults.copingDefaults()));
    }

    public CognitiveDefaults withRegulationDefaults(RegulationDefaults regulationDefaults) {
        return withEmotionDefaults(new EmotionDefaults(emotionDefaults.moodBaseline(), emotionDefaults.appraisalWeights(), regulationDefaults, emotionDefaults.copingDefaults()));
    }

    public CognitiveDefaults withCopingDefaults(CopingDefaults copingDefaults) {
        return withEmotionDefaults(new EmotionDefaults(emotionDefaults.moodBaseline(), emotionDefaults.appraisalWeights(), emotionDefaults.regulationDefaults(), copingDefaults));
    }

    public CognitiveDefaults withHabituationConfig(HabituationConfig habituationConfig) {
        var r = cognitiveResourceDefaults;
        return withCognitiveResourceDefaults(new CognitiveResourceDefaults(r.capacityConfig(), r.cognitiveResourcePoolConfig(), r.searchBudgetConfig(), r.recurringThoughtConfig(), habituationConfig));
    }

    // Resource delegates
    public CognitiveDefaults withSearchBudgetConfig(SearchBudgetConfig searchBudgetConfig) {
        var r = cognitiveResourceDefaults;
        return withCognitiveResourceDefaults(new CognitiveResourceDefaults(r.capacityConfig(), r.cognitiveResourcePoolConfig(), searchBudgetConfig, r.recurringThoughtConfig(), r.habituationConfig()));
    }

    public CognitiveDefaults withRecurringThoughtConfig(RecurringThoughtConfig recurringThoughtConfig) {
        var r = cognitiveResourceDefaults;
        return withCognitiveResourceDefaults(new CognitiveResourceDefaults(r.capacityConfig(), r.cognitiveResourcePoolConfig(), r.searchBudgetConfig(), recurringThoughtConfig, r.habituationConfig()));
    }

    public CognitiveDefaults withCapacityConfig(CognitiveCapacityConfig capacityConfig) {
        var r = cognitiveResourceDefaults;
        return withCognitiveResourceDefaults(new CognitiveResourceDefaults(capacityConfig, r.cognitiveResourcePoolConfig(), r.searchBudgetConfig(), r.recurringThoughtConfig(), r.habituationConfig()));
    }

    public CognitiveDefaults withCognitiveResourcePoolConfig(CognitiveResourcePoolConfig cognitiveResourcePoolConfig) {
        var r = cognitiveResourceDefaults;
        return withCognitiveResourceDefaults(new CognitiveResourceDefaults(r.capacityConfig(), cognitiveResourcePoolConfig, r.searchBudgetConfig(), r.recurringThoughtConfig(), r.habituationConfig()));
    }

    // Graph delegates
    public CognitiveDefaults withVocabulary(MindMapVocabulary vocabulary) {
        var g = graphDefaults;
        return withGraphDefaults(new GraphDefaults(vocabulary, g.services(), g.traitRules(), g.derivedEdgeRules(), g.graphStructure(), g.extractionBias()));
    }

    public CognitiveDefaults withServices(Map<String, String> services) {
        var g = graphDefaults;
        return withGraphDefaults(new GraphDefaults(g.vocabulary(), services, g.traitRules(), g.derivedEdgeRules(), g.graphStructure(), g.extractionBias()));
    }

    public CognitiveDefaults withTraitRules(List<DeclarativeTraitRule> traitRules) {
        var g = graphDefaults;
        return withGraphDefaults(new GraphDefaults(g.vocabulary(), g.services(), traitRules, g.derivedEdgeRules(), g.graphStructure(), g.extractionBias()));
    }

    public CognitiveDefaults withDerivedEdgeRules(List<DeclarativeDerivedEdgeRule> derivedEdgeRules) {
        var g = graphDefaults;
        return withGraphDefaults(new GraphDefaults(g.vocabulary(), g.services(), g.traitRules(), derivedEdgeRules, g.graphStructure(), g.extractionBias()));
    }

    public CognitiveDefaults withGraphStructure(GraphStructureDefaults graphStructure) {
        var g = graphDefaults;
        return withGraphDefaults(new GraphDefaults(g.vocabulary(), g.services(), g.traitRules(), g.derivedEdgeRules(), graphStructure, g.extractionBias()));
    }

    public CognitiveDefaults withExtractionBias(ExtractionBiasDefaults extractionBias) {
        var g = graphDefaults;
        return withGraphDefaults(new GraphDefaults(g.vocabulary(), g.services(), g.traitRules(), g.derivedEdgeRules(), g.graphStructure(), extractionBias));
    }

    // Behavioral delegates
    public CognitiveDefaults withCuriosity(CuriosityConfig curiosity) {
        var b = behavioralDefaults;
        return withBehavioralDefaults(new BehavioralDefaults(curiosity, b.temporalFocus(), b.socialCognition(), b.cbrStrategy()));
    }

    public CognitiveDefaults withTemporalFocus(TemporalFocusConfig temporalFocus) {
        var b = behavioralDefaults;
        return withBehavioralDefaults(new BehavioralDefaults(b.curiosity(), temporalFocus, b.socialCognition(), b.cbrStrategy()));
    }

    public CognitiveDefaults withSocialCognition(SocialCognitionDefaults socialCognition) {
        var b = behavioralDefaults;
        return withBehavioralDefaults(new BehavioralDefaults(b.curiosity(), b.temporalFocus(), socialCognition, b.cbrStrategy()));
    }

    public CognitiveDefaults withCbrStrategy(CbrStrategyDefaults cbrStrategy) {
        var b = behavioralDefaults;
        return withBehavioralDefaults(new BehavioralDefaults(b.curiosity(), b.temporalFocus(), b.socialCognition(), cbrStrategy));
    }
}
```

NOTE: Imports must include all the types from both cognitive-api and mindmap modules. The existing imports are mostly correct; add `EmotionDefaults`, `CognitiveResourceDefaults`, `GraphDefaults`, `BehavioralDefaults` and remove any that are no longer directly referenced at the record component level (they'll still be referenced in method signatures).

- [ ] **Step 2: Do not compile yet — Task 5 (CDE) must also change before compilation succeeds**

### Task 5: Refactor CognitiveDerivationEngine

**Files:**
- Modify: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java`

**Interfaces:**
- Consumes: Refactored CognitiveDefaults (Task 4), EmotionDefaults, CognitiveResourceDefaults, GraphDefaults, BehavioralDefaults (Tasks 1-3)
- Produces: Updated `derive()` and `deriveAndMerge()` that construct groups directly

- [ ] **Step 1: Refactor derive() method (lines 287-304)**

Replace the withX() chain at the end of `derive()` with group construction:

```java
var emotionDefaults = new EmotionDefaults(mood, appraisal, regulation, coping);
var resourceDefaults = new CognitiveResourceDefaults(capacity, poolConfig, searchBudget, recurringThought, habituation);
var graphDefaults = GraphDefaults.empty();
var behavioralDefaults = new BehavioralDefaults(curiosity, temporalFocus, socialCognition, cbrStrategy);

observer.onPipelineComplete(descriptor.agentId(), eventRef);

return new CognitiveDefaults(descriptor.agentId(), null, personalityProf, descriptor,
        emotionDefaults, resourceDefaults, graphDefaults, behavioralDefaults);
```

The `graphDefaults` is `empty()` because `derive()` never derives vocabulary, services, traitRules, or derivedEdgeRules — only graphStructure and extractionBias. Update to:

```java
var graphDefaults = new GraphDefaults(null, Map.of(), null, null, graphStructure, extractionBias);
```

- [ ] **Step 2: Refactor deriveAndMerge() method (lines 308-337)**

Replace the per-field merge chain with group-level merging:

```java
public static CognitiveDefaults deriveAndMerge(CognitiveDefaults explicit) {
    if (explicit.descriptor() == null) {
        return explicit;
    }
    CognitiveDefaults derived = derive(explicit.descriptor());
    var mergedProfile = mergePersonalityProfile(explicit.personalityProfile(), derived.personalityProfile());
    var mergedEmotion = explicit.emotionDefaults().mergeWith(derived.emotionDefaults());
    var mergedResources = explicit.cognitiveResourceDefaults().mergeWith(derived.cognitiveResourceDefaults());
    var mergedGraph = explicit.graphDefaults().mergeWith(derived.graphDefaults());
    var mergedBehavioral = explicit.behavioralDefaults().mergeWith(derived.behavioralDefaults());

    return new CognitiveDefaults(explicit.agentId(), explicit.tenantId(), mergedProfile,
            explicit.descriptor(), mergedEmotion, mergedResources, mergedGraph, mergedBehavioral);
}
```

Remove the now-unused `mergeCapacity()` static method (lines 400-404) — it moved to `CognitiveResourceDefaults.mergeCapacity()`.

- [ ] **Step 3: Compile the module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml compile -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: BUILD SUCCESS (Tasks 4 + 5 combined make the module compile)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDefaults.java cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "refactor(#999): decompose CognitiveDefaults into nested config groups  Refs casehubio/neocortex#999"
```

### Task 6: Update YAML profiles and parser tests

**Files:**
- Modify: `cognitive-index/src/test/resources/cognitive-profiles/alice.yaml`
- Modify: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDefaultsParserTest.java`

**Interfaces:**
- Consumes: Refactored CognitiveDefaults record (Task 4)
- Produces: Updated YAML structure matching the nested record. `minimal.yaml` needs no change (only has `agentId`).

- [ ] **Step 1: Update alice.yaml to nested structure**

```yaml
agentId: alice
tenantId: family-1

personalityProfile:
  personality:
    experience: 1.5
    relationship: 0.8
    mood: 1.2

emotionDefaults:
  moodBaseline:
    pleasure: 0.0
    arousal: -0.2
    dominance: 0.3

behavioralDefaults:
  curiosity:
    proximityScale: 7.0
    staleDaysThreshold: 90
    maxBfsDepth: 4
    topCentrality: 3
    volatilityThreshold: 0.3
    maxBoostFactor: 1.0
    minDampenFactor: 0.1
    improvingDampenCap: 0.7
    volatilityBoostCap: 0.5
    trajectoryLimit: 20
  temporalFocus:
    proximityScale: 7.0
    worseningBoostCap: 1.0
    improvingDampenFactor: 0.5
    volatilityBoostCap: 0.5

graphDefaults:
  vocabulary:
    edgeTypes:
      - canonical: knows
        aliases: [knows-about, familiar-with]
        defaultDecayHalfLifeDays: 365
      - canonical: child-of
  services:
    reflectionSynthesizer: llm
    explanationRenderer: detailed
  traitRules:
    - trait: FamilyMember
      when:
        hasEdgeTypes: [child-of, parent-of, sibling-of]
  derivedEdgeRules:
    - name: inverse-knows
      on:
        edgeType: knows
      derive:
        - edgeType: known-by
          source: trigger.target
          target: trigger.source
          properties:
            perspective: alice
```

- [ ] **Step 2: Update CognitiveDefaultsParserTest assertions**

The test accesses fields via convenience methods (`defaults.moodBaseline()`, `defaults.curiosity()`, etc.), which still work. The main change needed: verify group-level accessors work too.

Add a test verifying the group structure:
```java
@Test
void fullProfile_parsesGroupStructure() throws IOException {
    var defaults = parse("cognitive-profiles/alice.yaml");

    assertThat(defaults.emotionDefaults()).isNotNull();
    assertThat(defaults.emotionDefaults().moodBaseline()).isNotNull();
    assertThat(defaults.behavioralDefaults()).isNotNull();
    assertThat(defaults.behavioralDefaults().curiosity()).isNotNull();
    assertThat(defaults.graphDefaults()).isNotNull();
    assertThat(defaults.graphDefaults().vocabulary()).isNotNull();
    assertThat(defaults.graphDefaults().services()).containsEntry("reflectionSynthesizer", "llm");
}
```

Also verify the existing tests still pass — they use `defaults.moodBaseline()` etc. which now delegates through the group.

- [ ] **Step 3: Run all cognitive-index tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognitive-index/pom.xml test -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: All tests pass

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/src/test/resources/cognitive-profiles/alice.yaml cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDefaultsParserTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "refactor(#999): update YAML profiles and parser tests for nested groups  Refs casehubio/neocortex#999"
```

---

## Batch 3: Cross-module verification [full build passes]

### Task 7: Full build verification and fix cross-module compilation

**Files:**
- Potentially modify: test files across cognition, mindmap-intelligence, mindmap-core, cognitive-observability-core, cognitive-eidos, caps-llm-classifier modules (only if compilation breaks)

**Interfaces:**
- Consumes: All changes from Tasks 1-6
- Produces: Green full build across all neocortex modules

Since CognitiveDefaults retains all leaf-level accessor and withX() convenience methods, most cross-module code should compile unchanged. The main risk areas:
1. Code that imports types that were previously record components but are now only in method signatures (should still resolve via import)
2. Any code that relies on CognitiveDefaults being a record with specific component names for reflection/serialization

- [ ] **Step 1: Build the full neocortex project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml compile -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: BUILD SUCCESS (or compilation errors to fix)

- [ ] **Step 2: If compilation errors, fix them**

Likely fixes:
- Missing imports in files that accessed now-grouped fields directly on the record object (instead of via accessor methods)
- Pattern match / deconstruction on CognitiveDefaults record (unlikely but check)

- [ ] **Step 3: Run fast tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -Pfast -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
(If no fast profile exists on neocortex, run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`)
Expected: All tests pass

- [ ] **Step 4: Build examples/wacky-manor to verify downstream**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/examples/wacky-manor/pom.xml compile -s /Users/mdproctor/claude/casehub/slots/196/.mvn/slot-settings.xml`
Expected: BUILD SUCCESS (wacky-manor uses CognitiveDefaults only as a pass-through)

- [ ] **Step 5: Commit any fixes**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add -A
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "fix(#999): cross-module compilation fixes for CognitiveDefaults decomposition  Refs casehubio/neocortex#999"
```

---

## References

- [casehubio/neocortex#999] — focal issue: CognitiveDefaults structural decomposition
- [CognitiveDefaults.java] — 23-field record (the target)
- [CognitiveDerivationEngine.java] — derive() and deriveAndMerge() (the main producer)
- [PersonalityProfile.java] — existing nested group pattern
- [CognitiveCapacityConfig.java:51-56] — existing mergeWith() pattern
- [alice.yaml] — full YAML profile (must nest)
- [minimal.yaml] — minimal profile (unchanged)
- [CognitiveDefaultsParserTest.java] — YAML parsing tests
- [CognitiveDefaultsRegistryTest.java] — registry tests
- [CognitiveDerivationEngineTest.java] — derivation tests
