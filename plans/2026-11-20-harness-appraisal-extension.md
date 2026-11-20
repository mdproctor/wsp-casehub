# Harness Appraisal Extension Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1188 — extend CognitionTestHarness with lightweight appraisal for tick-level divergence
**Issue group:** #1188

**Goal:** Wire a real rule-based appraisal participant into the test harness so divergence tests can assert on tick-level `PipelineTrace` fields.

**Architecture:** Create a `TestAppraisalStrategy` that computes personality-sensitive SEC dimensions from `AppraisalWeights` and `DispositionAxes`. Add `withAppraisalEnabled(true)` to the harness builder, wiring through `CognitivePipelineBuilder.appraisal()` to create a real `AppraisalTickParticipant`. Upgrade divergence tests to use tick-level assertions.

**Tech Stack:** Java 21, JUnit 5, AssertJ

## Global Constraints

- `TestAppraisalStrategy` lives in test sources only (`cognition/src/test/java/`)
- No LLM or NLP — purely mechanical personality-sensitive rules
- `withAppraisalEnabled(true)` and `withAppraisalResult()` are mutually exclusive

---

## Batch 1: TestAppraisalStrategy + harness wiring

### Task 1: TestAppraisalStrategy

**Files:**
- Create: `cognition/src/test/java/io/casehub/neocortex/cognition/eval/TestAppraisalStrategy.java`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/eval/TestAppraisalStrategyTest.java`

**Interfaces:**
- Consumes: `AppraisalStrategy` (functional interface), `AppraisalContext`, `AppraisalWeights`, `DispositionAxes`, `SecDimensions`, `ActionReadiness`, `ActionTendency`, `HabituationState`, `CognitiveEmotion`, `EmotionType`, `EmotionSource`, `AlmaPadTable`
- Produces: `TestAppraisalStrategy implements AppraisalStrategy` — used by Task 2 in harness builder

- [ ] **Step 1: Write failing test — SEC dimensions differ for OPEN vs CAUTIOUS**

```java
package io.casehub.neocortex.cognition.eval;

import io.casehub.neocortex.cognition.appraisal.*;
import io.casehub.neocortex.cognition.eval.divergence.DivergenceFixtures;
import io.casehub.neocortex.cognitive.index.DispositionAxes;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class TestAppraisalStrategyTest {

    private final TestAppraisalStrategy strategy = new TestAppraisalStrategy();

    @Test
    void secDimensionsDivergeByPersonality() {
        var openCtx = contextFor("open",
                DivergenceFixtures.openDefaults().appraisalWeights(),
                DivergenceFixtures.OPEN_AXES);
        var cautiousCtx = contextFor("cautious",
                DivergenceFixtures.cautiousDefaults().appraisalWeights(),
                DivergenceFixtures.CAUTIOUS_AXES);

        var openResult = strategy.appraise(openCtx);
        var cautiousResult = strategy.appraise(cautiousCtx);

        assertThat(openResult.secDimensions()).isNotEmpty();
        assertThat(cautiousResult.secDimensions()).isNotEmpty();

        boolean anyDiffers = false;
        for (var key : openResult.secDimensions().keySet()) {
            if (!openResult.secDimensions().get(key).equals(
                    cautiousResult.secDimensions().get(key))) {
                anyDiffers = true;
                break;
            }
        }
        assertThat(anyDiffers)
                .as("SEC dimensions should differ — open=%s vs cautious=%s",
                        openResult.secDimensions(), cautiousResult.secDimensions())
                .isTrue();
    }

    private static AppraisalContext contextFor(String agentId,
            io.casehub.neocortex.mindmap.AppraisalWeights weights,
            DispositionAxes axes) {
        return new AppraisalContext(agentId,
                PerceivedSituation.passThrough("test observation"),
                List.of(), weights, null, HabituationState.empty(), null, axes);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="TestAppraisalStrategyTest" -s .mvn/slot-settings.xml -pl .`
Expected: compilation error — `TestAppraisalStrategy` class not found

- [ ] **Step 3: Write TestAppraisalStrategy implementation**

```java
package io.casehub.neocortex.cognition.eval;

import io.casehub.neocortex.cognition.appraisal.*;
import io.casehub.neocortex.cognitive.*;
import io.casehub.neocortex.cognitive.index.DispositionAxes;
import io.casehub.neocortex.mindmap.AppraisalWeights;
import java.time.Instant;
import java.util.List;
import java.util.Map;

public class TestAppraisalStrategy implements AppraisalStrategy {

    @Override
    public AppraisalResult appraise(AppraisalContext context) {
        var weights = context.weights();
        var axes = context.dispositionAxes();

        var secDimensions = computeSecDimensions(weights);
        var tendencies = deriveTendencies(axes);
        var emotions = deriveEmotions(secDimensions);

        return new AppraisalResult(emotions, tendencies,
                HabituationState.empty(), null, secDimensions);
    }

    private static Map<String, Double> computeSecDimensions(AppraisalWeights weights) {
        if (weights == null) {
            return Map.of(
                    SecDimensions.RELEVANCE, 0.8,
                    SecDimensions.NOVELTY, 0.5,
                    SecDimensions.CONDUCIVENESS, 0.0,
                    SecDimensions.CONTROLLABILITY, 0.5);
        }
        double conduciveness = (weights.fearOnsetThreshold() - 1.0) * 0.5;
        double controllability = Math.clamp(1.0 - (weights.urgencyWeight() - 1.0) * 0.3, 0.0, 1.0);

        return Map.of(
                SecDimensions.RELEVANCE, 0.8,
                SecDimensions.NOVELTY, 0.5,
                SecDimensions.CONDUCIVENESS, conduciveness,
                SecDimensions.CONTROLLABILITY, controllability);
    }

    private static List<ActionTendency> deriveTendencies(DispositionAxes axes) {
        if (axes == null) {
            return List.of(new ActionTendency(ActionReadiness.ATTENDING, 0.4, ""));
        }
        return switch (axes.conflictMode()) {
            case "cooperative" -> List.of(new ActionTendency(ActionReadiness.APPROACH, 0.6, ""));
            case "avoidant" -> List.of(new ActionTendency(ActionReadiness.AVOIDANCE, 0.6, ""));
            default -> List.of(new ActionTendency(ActionReadiness.ATTENDING, 0.4, ""));
        };
    }

    private static List<CognitiveEmotion> deriveEmotions(Map<String, Double> sec) {
        double conduciveness = sec.getOrDefault(SecDimensions.CONDUCIVENESS, 0.0);
        var type = conduciveness >= 0 ? EmotionType.INTEREST : EmotionType.FEAR;
        double intensity = Math.clamp(Math.abs(conduciveness) + 0.3, 0.1, 0.9);
        var pad = AlmaPadTable.project(type, intensity);
        return List.of(new CognitiveEmotion(type, intensity, "appraisal",
                Instant.now(), EmotionSource.INTRINSIC, pad));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="TestAppraisalStrategyTest" -s .mvn/slot-settings.xml -pl .`
Expected: PASS

- [ ] **Step 5: Add second test — action tendencies differ by disposition**

```java
@Test
void actionTendenciesDivergeByDisposition() {
    var openCtx = contextFor("open",
            DivergenceFixtures.openDefaults().appraisalWeights(),
            DivergenceFixtures.OPEN_AXES);
    var cautiousCtx = contextFor("cautious",
            DivergenceFixtures.cautiousDefaults().appraisalWeights(),
            DivergenceFixtures.CAUTIOUS_AXES);

    var openResult = strategy.appraise(openCtx);
    var cautiousResult = strategy.appraise(cautiousCtx);

    assertThat(openResult.actionTendencies()).isNotEmpty();
    assertThat(cautiousResult.actionTendencies()).isNotEmpty();

    assertThat(openResult.actionTendencies().getFirst().readiness())
            .as("cooperative disposition → APPROACH")
            .isEqualTo(io.casehub.neocortex.cognition.appraisal.ActionReadiness.APPROACH);
    assertThat(cautiousResult.actionTendencies().getFirst().readiness())
            .as("avoidant disposition → AVOIDANCE")
            .isEqualTo(io.casehub.neocortex.cognition.appraisal.ActionReadiness.AVOIDANCE);
}
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="TestAppraisalStrategyTest" -s .mvn/slot-settings.xml -pl .`
Expected: 2 tests PASS

- [ ] **Step 7: Commit**

```bash
git -C neocortex add cognition/src/test/java/io/casehub/neocortex/cognition/eval/TestAppraisalStrategy.java cognition/src/test/java/io/casehub/neocortex/cognition/eval/TestAppraisalStrategyTest.java
git -C neocortex commit -m "feat(#1188): TestAppraisalStrategy — rule-based personality-sensitive appraisal for test harness"
```

---

### Task 2: Harness `withAppraisalEnabled` wiring

**Files:**
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/eval/CognitionTestHarness.java` — Builder class

**Interfaces:**
- Consumes: `TestAppraisalStrategy` (from Task 1), `NoOpSalienceStrategy`, `RuleBasedSituationClassifier`, `CognitivePipelineBuilder.appraisal()`, `CognitivePipelineBuilder.situationClassifier()`
- Produces: `Builder.withAppraisalEnabled(boolean)` — used by Task 3 in divergence tests

- [ ] **Step 1: Write failing test — appraisal result populated after tick**

Create test file:
```java
package io.casehub.neocortex.cognition.eval;

import io.casehub.neocortex.cognition.eval.divergence.DivergenceFixtures;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CognitionTestHarnessAppraisalTest {

    @Test
    void appraisalEnabledPopulatesTraceAfterTick() {
        var harness = DivergenceFixtures.pairedBuilder()
                .withAppraisalEnabled(true)
                .build();

        harness.observe("open", DivergenceFixtures.TENANT, "test observation");
        harness.tick("open", DivergenceFixtures.TENANT);

        var trace = harness.currentTrace("open", DivergenceFixtures.TENANT);
        assertThat(trace).isPresent();
        assertThat(trace.get().appraisalResult())
                .as("appraisal result should be populated after tick")
                .isNotNull();
        assertThat(trace.get().appraisalResult().secDimensions())
                .as("SEC dimensions should be populated")
                .isNotEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="CognitionTestHarnessAppraisalTest" -s .mvn/slot-settings.xml -pl .`
Expected: compilation error — `withAppraisalEnabled` not found

- [ ] **Step 3: Add `withAppraisalEnabled` to Builder and wire in `build()`**

Add field and method to `CognitionTestHarness.Builder`:

```java
private boolean appraisalEnabled;

public Builder withAppraisalEnabled(boolean enabled) {
    this.appraisalEnabled = enabled;
    return this;
}
```

In `Builder.build()`, modify the pipeline builder chain (around line 445-452) to add appraisal when enabled:

```java
var pipelineBuilder = io.casehub.neocortex.cognition.core.CognitivePipelineBuilder
    .create(moodOrchestrator, driveOrchestrator)
    .userModel(userModel).mentalModel(mentalModel).strategy(strategy)
    .narrative(narrativeOrch).goals(goals).memoryHygiene(hygiene)
    .innerLife(innerLife).config(config).mindMapStore(store)
    .consolidationMediator(mediator).sessionPipeline(sessionPipeline)
    .effectiveDefaults(registry);

if (appraisalEnabled) {
    if (!appraisalResults.isEmpty() || !habituationStates.isEmpty()) {
        throw new IllegalStateException(
                "withAppraisalEnabled and withAppraisalResult/withHabituationState are mutually exclusive");
    }
    pipelineBuilder.appraisal(new TestAppraisalStrategy(),
            new io.casehub.neocortex.cognition.appraisal.NoOpSalienceStrategy());
}

var core = pipelineBuilder.build();
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="CognitionTestHarnessAppraisalTest" -s .mvn/slot-settings.xml -pl .`
Expected: PASS

- [ ] **Step 5: Add test — mutual exclusion throws**

```java
@Test
void appraisalEnabledAndMockThrows() {
    org.junit.jupiter.api.Assertions.assertThrows(IllegalStateException.class, () ->
        DivergenceFixtures.pairedBuilder()
                .withAppraisalEnabled(true)
                .withAppraisalResult("open",
                        io.casehub.neocortex.cognition.appraisal.AppraisalResult.empty())
                .build());
}
```

- [ ] **Step 6: Run all harness tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="CognitionTestHarnessAppraisalTest,TestAppraisalStrategyTest" -s .mvn/slot-settings.xml -pl .`
Expected: all PASS

- [ ] **Step 7: Run existing divergence tests for regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="*DivergenceTest" -s .mvn/slot-settings.xml -pl .`
Expected: all 40 existing tests PASS

- [ ] **Step 8: Commit**

```bash
git -C neocortex add cognition/src/test/java/io/casehub/neocortex/cognition/eval/CognitionTestHarness.java cognition/src/test/java/io/casehub/neocortex/cognition/eval/CognitionTestHarnessAppraisalTest.java
git -C neocortex commit -m "feat(#1188): withAppraisalEnabled on CognitionTestHarness — wires real AppraisalTickParticipant via pipeline builder"
```

---

## Batch 2: Upgraded divergence tests

### Task 3: Tick-level divergence assertions in AppraisalDivergenceTest

**Files:**
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/eval/divergence/AppraisalDivergenceTest.java`

**Interfaces:**
- Consumes: `Builder.withAppraisalEnabled(true)` (from Task 2), `PipelineTrace.appraisalResult()`, `PipelineTrace.capsInputActivations()`, `DivergenceFixtures`

- [ ] **Step 1: Add new test — appraisal results diverge at tick level**

```java
@Test
void appraisalResultDivergesAtTickLevel() {
    var harness = DivergenceFixtures.pairedBuilder()
            .withAppraisalEnabled(true)
            .build();

    harness.observe("open", TENANT, "someone expressed strong disagreement");
    harness.tick("open", TENANT);
    harness.observe("cautious", TENANT, "someone expressed strong disagreement");
    harness.tick("cautious", TENANT);

    var openTrace = harness.currentTrace("open", TENANT);
    var cautiousTrace = harness.currentTrace("cautious", TENANT);

    assertThat(openTrace).isPresent();
    assertThat(cautiousTrace).isPresent();
    assertThat(openTrace.get().appraisalResult()).isNotNull();
    assertThat(cautiousTrace.get().appraisalResult()).isNotNull();

    var openSec = openTrace.get().appraisalResult().secDimensions();
    var cautiousSec = cautiousTrace.get().appraisalResult().secDimensions();

    boolean secDiffers = openSec.entrySet().stream()
            .anyMatch(e -> !e.getValue().equals(cautiousSec.get(e.getKey())));

    assertThat(secDiffers)
            .as("SEC dimensions should differ at tick level — open=%s vs cautious=%s",
                    openSec, cautiousSec)
            .isTrue();
}
```

- [ ] **Step 2: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="AppraisalDivergenceTest#appraisalResultDivergesAtTickLevel" -s .mvn/slot-settings.xml -pl .`
Expected: PASS

- [ ] **Step 3: Add new test — action tendencies diverge at tick level**

```java
@Test
void actionTendenciesDivergeAtTickLevel() {
    var harness = DivergenceFixtures.pairedBuilder()
            .withAppraisalEnabled(true)
            .build();

    harness.observe("open", TENANT, "someone expressed strong disagreement");
    harness.tick("open", TENANT);
    harness.observe("cautious", TENANT, "someone expressed strong disagreement");
    harness.tick("cautious", TENANT);

    var openTrace = harness.currentTrace("open", TENANT);
    var cautiousTrace = harness.currentTrace("cautious", TENANT);

    assertThat(openTrace).isPresent();
    assertThat(cautiousTrace).isPresent();

    var openTendencies = openTrace.get().appraisalResult().actionTendencies();
    var cautiousTendencies = cautiousTrace.get().appraisalResult().actionTendencies();

    assertThat(openTendencies).isNotEmpty();
    assertThat(cautiousTendencies).isNotEmpty();

    assertThat(openTendencies.getFirst().readiness())
            .as("action tendencies should differ — open=%s vs cautious=%s",
                    openTendencies, cautiousTendencies)
            .isNotEqualTo(cautiousTendencies.getFirst().readiness());
}
```

- [ ] **Step 4: Run all divergence tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="*DivergenceTest" -s .mvn/slot-settings.xml -pl .`
Expected: all PASS (40 existing + 2 new = 42)

- [ ] **Step 5: Commit**

```bash
git -C neocortex add cognition/src/test/java/io/casehub/neocortex/cognition/eval/divergence/AppraisalDivergenceTest.java
git -C neocortex commit -m "feat(#1188): tick-level appraisal divergence tests — SEC dimensions + action tendencies"
```

---

## References

- [2026-11-20-harness-appraisal-extension-design.md] — design spec this plan implements
- [CognitionTestHarness.java:445-512] — harness builder build() method
- [CognitivePipelineBuilder.java:184,282-284] — `.appraisal()` method and wiring
- [AppraisalTickParticipant.java:124-215] — real `tick()` method
- [AppraisalStrategy.java] — functional interface
- [AppraisalWeights.java] — personality-derived weights record
- [PipelineTrace.java:13-26] — trace fields
- [AppraisalDivergenceTest.java] — tests to upgrade
- [DivergenceFixtures.java] — OPEN/CAUTIOUS fixture data
- [GitHub #1188] — focal issue
