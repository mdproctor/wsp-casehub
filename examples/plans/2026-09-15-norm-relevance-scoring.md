# Dynamic Norm Relevance Scoring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #57 — Context-based ManorNormFilter
**Issue group:** #54, #55, #56, #57, #59, #60

**Goal:** Replace ManorNormFilter's static sort-by-priority with cognitive
context-driven relevance scoring and budget-gated selection.

**Architecture:** ManorNormFilter.score() replaces filter(). Each norm is
scored as `priority + contextBoost(10) + inventoryBoost(10)`, where boosts
fire when case-insensitive word tokens (≥3 chars) from active cognitive
subjects or inventory items appear in the norm's rule text. Results are
sorted by score descending and truncated to CognitiveBudget.maxNorms().
ManorContextStrategy.selectNorms() wires the budget through.
CharacterCognition.renderCognitiveSections() passes entity display names
from the existing agentNames map.

**Tech Stack:** Java 26, JUnit 5, AssertJ, Quarkus

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
- All 457+ existing tests must continue to pass
- No new dependencies — everything needed is already on the classpath
- Follow DSL Style Guide YAML/Java parity where applicable

---

## Batch 1: Scoring + Wiring

### Task 1: ManorNormFilter — scoring with cognitive context

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorNormFilter.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorNormFilterTest.java`

**Interfaces:**
- Consumes: `SocialConfig.NormEntry(String rule, int priority)` — existing record
- Produces: `ManorNormFilter.score(List<NormEntry> allNorms, Collection<String> activeCognitiveSubjects, Collection<String> activeInventory, int maxNorms)` → `List<NormEntry>` — scored, sorted, budget-gated

- [ ] **Step 1: Replace existing tests with new scoring tests**

Rewrite `ManorNormFilterTest` entirely — the old `filter()` method is being replaced. The new tests exercise the scoring model.

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;

import java.util.Collection;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ManorNormFilterTest {

    @Test
    void contextRelevantNormsRankedAboveGeneral() {
        var norms = List.of(
                new SocialConfig.NormEntry("Maintain charm", 5),
                new SocialConfig.NormEntry("Never help Penelope directly", 5));
        var result = ManorNormFilter.score(norms, List.of("Penelope Pitstop"), List.of(), 10);
        assertThat(result.get(0).rule()).isEqualTo("Never help Penelope directly");
        assertThat(result.get(1).rule()).isEqualTo("Maintain charm");
    }

    @Test
    void generalNormsIncludedWithinBudget() {
        var norms = List.of(
                new SocialConfig.NormEntry("General rule", 5),
                new SocialConfig.NormEntry("Help Penelope", 3));
        var result = ManorNormFilter.score(norms, List.of("Penelope Pitstop"), List.of(), 10);
        assertThat(result).hasSize(2);
    }

    @Test
    void budgetGatesNormCount() {
        var norms = List.of(
                new SocialConfig.NormEntry("Rule A", 10),
                new SocialConfig.NormEntry("Rule B", 8),
                new SocialConfig.NormEntry("Rule C", 5));
        var result = ManorNormFilter.score(norms, List.of(), List.of(), 2);
        assertThat(result).hasSize(2);
        assertThat(result.get(0).rule()).isEqualTo("Rule A");
        assertThat(result.get(1).rule()).isEqualTo("Rule B");
    }

    @Test
    void inventoryMatchBoostsRelevance() {
        var norms = List.of(
                new SocialConfig.NormEntry("General rule", 8),
                new SocialConfig.NormEntry("Protect the treasure", 3));
        var result = ManorNormFilter.score(norms, List.of(), List.of("treasure"), 10);
        assertThat(result.get(0).rule()).isEqualTo("Protect the treasure");
    }

    @Test
    void shortWordsIgnored() {
        var norms = List.of(
                new SocialConfig.NormEntry("Do no harm", 5));
        // "HC" is 2 chars — below threshold
        var result = ManorNormFilter.score(norms, List.of("HC"), List.of(), 10);
        // No boost — "HC" is too short, score stays at priority 5
        assertThat(result).hasSize(1);
        assertThat(result.get(0).priority()).isEqualTo(5);
    }

    @Test
    void caseInsensitiveMatching() {
        var norms = List.of(
                new SocialConfig.NormEntry("never help penelope", 3),
                new SocialConfig.NormEntry("General rule", 5));
        var result = ManorNormFilter.score(norms, List.of("Penelope Pitstop"), List.of(), 10);
        assertThat(result.get(0).rule()).isEqualTo("never help penelope");
    }

    @Test
    void emptyContextReturnsTopByPriority() {
        var norms = List.of(
                new SocialConfig.NormEntry("Low", 1),
                new SocialConfig.NormEntry("High", 10),
                new SocialConfig.NormEntry("Mid", 5));
        var result = ManorNormFilter.score(norms, List.of(), List.of(), 10);
        assertThat(result.get(0).rule()).isEqualTo("High");
        assertThat(result.get(1).rule()).isEqualTo("Mid");
        assertThat(result.get(2).rule()).isEqualTo("Low");
    }

    @Test
    void multipleCognitiveSubjectsMatchIndependently() {
        var norms = List.of(
                new SocialConfig.NormEntry("Protect Penelope always", 3),
                new SocialConfig.NormEntry("Beware of Dastardly", 3),
                new SocialConfig.NormEntry("Stay calm", 3));
        var result = ManorNormFilter.score(norms,
                List.of("Penelope Pitstop", "Dick Dastardly"), List.of(), 10);
        assertThat(result.get(0).rule()).isIn("Protect Penelope always", "Beware of Dastardly");
        assertThat(result.get(1).rule()).isIn("Protect Penelope always", "Beware of Dastardly");
        assertThat(result.get(2).rule()).isEqualTo("Stay calm");
    }

    @Test
    void priorityBreaksTiesAmongBoosted() {
        var norms = List.of(
                new SocialConfig.NormEntry("Protect Penelope always", 9),
                new SocialConfig.NormEntry("Never help Penelope directly", 3));
        var result = ManorNormFilter.score(norms, List.of("Penelope Pitstop"), List.of(), 10);
        assertThat(result.get(0).rule()).isEqualTo("Protect Penelope always");
        assertThat(result.get(1).rule()).isEqualTo("Never help Penelope directly");
    }

    @Test
    void emptyNormsReturnsEmpty() {
        var result = ManorNormFilter.score(List.of(), List.of("Penelope"), List.of("key"), 10);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorNormFilterTest -s slot-settings.xml`
Expected: compilation failure — `score()` method does not exist

- [ ] **Step 3: Implement ManorNormFilter.score()**

Replace the entire body of `ManorNormFilter.java`:

```java
package io.casehub.examples.manor.agent;

import java.util.Collection;
import java.util.List;
import java.util.Locale;
import java.util.Set;
import java.util.stream.Collectors;

public final class ManorNormFilter {

    static final int CONTEXT_BOOST = 10;
    static final int MIN_WORD_LENGTH = 3;

    private ManorNormFilter() {}

    public static List<SocialConfig.NormEntry> score(
            List<SocialConfig.NormEntry> allNorms,
            Collection<String> activeCognitiveSubjects,
            Collection<String> activeInventory,
            int maxNorms) {

        Set<String> matchWords = extractMatchWords(activeCognitiveSubjects);
        Set<String> inventoryWords = extractMatchWords(activeInventory);

        record Scored(SocialConfig.NormEntry norm, int score) {}

        return allNorms.stream()
                .map(norm -> {
                    String ruleLower = norm.rule().toLowerCase(Locale.ROOT);
                    int score = norm.priority();
                    if (matchWords.stream().anyMatch(ruleLower::contains)) {
                        score += CONTEXT_BOOST;
                    }
                    if (inventoryWords.stream().anyMatch(ruleLower::contains)) {
                        score += CONTEXT_BOOST;
                    }
                    return new Scored(norm, score);
                })
                .sorted((a, b) -> Integer.compare(b.score(), a.score()))
                .limit(maxNorms)
                .map(Scored::norm)
                .toList();
    }

    private static Set<String> extractMatchWords(Collection<String> names) {
        return names.stream()
                .flatMap(name -> java.util.Arrays.stream(name.split("\\s+")))
                .filter(word -> word.length() >= MIN_WORD_LENGTH)
                .map(word -> word.toLowerCase(Locale.ROOT))
                .collect(Collectors.toSet());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorNormFilterTest -s slot-settings.xml`
Expected: all 10 tests PASS

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorNormFilter.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorNormFilterTest.java
git commit -m "feat(#57): ManorNormFilter.score() — cognitive context relevance scoring with budget gating

Replaces filter() with score(). Norms scored by priority + context boost
(+10 when entity display name words appear in rule text) + inventory boost.
Budget-gated via maxNorms parameter.

Refs #57

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: ManorContextStrategy + CharacterCognition wiring

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorContextStrategy.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:94-128`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: `ManorNormFilter.score(List<NormEntry>, Collection<String>, Collection<String>, int)` from Task 1
- Consumes: `CognitiveBudget.maxNorms()` — existing accessor on existing record
- Produces: `ManorContextStrategy.selectNorms(List<NormEntry>, Collection<String>, Collection<String>, CognitiveBudget)` → `List<NormEntry>`

- [ ] **Step 1: Write test for ManorContextStrategy.selectNorms()**

Add a new test to verify `selectNorms` delegates correctly:

```java
// In a new file or appended to existing ManorContextStrategyTest if it exists
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ManorContextStrategyTest {

    @Test
    void selectNormsDelegatesToScoringWithBudget() {
        var strategy = new ManorContextStrategy();
        var norms = List.of(
                new SocialConfig.NormEntry("Never help Penelope directly", 10),
                new SocialConfig.NormEntry("Maintain charm", 5),
                new SocialConfig.NormEntry("Protect schemes", 8));
        var budget = CognitiveBudget.forSituation(0, 0.3, 0); // maxNorms = 4
        var result = strategy.selectNorms(norms, List.of("Penelope Pitstop"), List.of(), budget);
        assertThat(result).hasSize(3);
        assertThat(result.get(0).rule()).isEqualTo("Never help Penelope directly");
    }

    @Test
    void selectNormsRespectsMaxNormsFromBudget() {
        var strategy = new ManorContextStrategy();
        var norms = List.of(
                new SocialConfig.NormEntry("A", 10),
                new SocialConfig.NormEntry("B", 8),
                new SocialConfig.NormEntry("C", 6),
                new SocialConfig.NormEntry("D", 4),
                new SocialConfig.NormEntry("E", 2));
        var budget = new CognitiveBudget(5, 2, 3, 3); // maxNorms = 3
        var result = strategy.selectNorms(norms, List.of(), List.of(), budget);
        assertThat(result).hasSize(3);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorContextStrategyTest -s slot-settings.xml`
Expected: compilation failure — `selectNorms()` does not exist

- [ ] **Step 3: Implement ManorContextStrategy.selectNorms() and remove filterNorms()**

Replace the entire body of `ManorContextStrategy.java`:

```java
package io.casehub.examples.manor.agent;

import java.util.Collection;
import java.util.List;

public final class ManorContextStrategy {

    public CognitiveBudget budgetFor(int nearbyCount, double arousal, int activeGoals) {
        return CognitiveBudget.forSituation(nearbyCount, arousal, activeGoals);
    }

    public List<SocialConfig.NormEntry> selectNorms(List<SocialConfig.NormEntry> norms,
                                                      Collection<String> activeCognitiveSubjects,
                                                      Collection<String> activeInventory,
                                                      CognitiveBudget budget) {
        return ManorNormFilter.score(norms, activeCognitiveSubjects, activeInventory, budget.maxNorms());
    }
}
```

- [ ] **Step 4: Run ManorContextStrategyTest to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorContextStrategyTest -s slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Update CharacterCognition.renderCognitiveSections() norm wiring**

In `CharacterCognition.java`, replace the norm filtering block (lines 122-128):

Before:
```java
var filteredNorms = contextStrategy.filterNorms(socialConfig.norms(), nearbyAgentIds, character.inventory());
if (!filteredNorms.isEmpty()) {
    var items = filteredNorms.stream()
                             .map(SocialConfig.NormEntry::rule)
                             .toList();
    sections.add(ObservationSection.items("Social Rules", null, items));
}
```

After:
```java
var budget = contextStrategy.budgetFor(nearbyAgentIds.size(), 0.5, 0);
var selectedNorms = contextStrategy.selectNorms(socialConfig.norms(), agentNames.values(), character.inventory(), budget);
if (!selectedNorms.isEmpty()) {
    var items = selectedNorms.stream()
                              .map(SocialConfig.NormEntry::rule)
                              .toList();
    sections.add(ObservationSection.items("Social Rules", null, items));
}
```

- [ ] **Step 6: Update CharacterCognitionTest — beliefsAndNormsRendered**

The test at line 82-90 passes empty nearbyAgentIds and empty agentNames. With budget gating (base maxNorms=4), all 3 Penelope norms still fit. No logic change needed — verify it still passes.

- [ ] **Step 7: Update CharacterCognitionTest — allFourSectionsForFullyConfiguredCharacter**

The test at line 93-105 already passes `Map.of("penelope-pitstop", "Penelope Pitstop")` as agentNames. This means "Penelope Pitstop" will now be an active cognitive subject. Hooded Claw's norm "Never help Penelope directly" will be boosted. All 3 norms still fit within budget (maxNorms=4 for 1 nearby). Verify it still passes.

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS (457+ tests, no regressions)

- [ ] **Step 9: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorContextStrategy.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorContextStrategyTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java
git commit -m "feat(#57): wire ManorContextStrategy.selectNorms + budget gating into CharacterCognition

ManorContextStrategy.selectNorms() replaces filterNorms(), passes budget
through. CharacterCognition.renderCognitiveSections() passes entity display
names from agentNames map and computes situational budget for norm selection.

Refs #57

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-15-norm-relevance-scoring-design.md] — design spec this plan implements
- ManorNormFilter.java:6-18 — current filter() implementation being replaced
- ManorContextStrategy.java:12-16 — current filterNorms() delegation
- CharacterCognition.java:94-128 — renderCognitiveSections with norm rendering
- ScenarioOrchestrator.java:243-257 — tick loop context (nearbyIds, agentNameMap)
- CognitiveBudget.java:3-22 — maxNorms with situational adaptation
- SocialConfig.java:31-34 — NormEntry record
- social-config.yaml — actual norm data
- ManorNormFilterTest.java — existing tests being replaced
- CharacterCognitionTest.java:82-105 — existing tests being updated
- GE-20260912-ff141b — neocortex cognitive stack
- GE-20260914-e3cb03 — profile data over prose directives
- GitHub #57 — Context-based ManorNormFilter
- GitHub #53 — Phase B: mindmap-native cognitive integration (epic)
