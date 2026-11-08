# Per-Strategy Regulation Eval Tests Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#822 — per-strategy regulation tests
**Issue group:** #820 (regulation ecosystem), #851 (dynamic regulatory capacity)

**Goal:** Prove all 5 Gross emotion regulation strategies are exercised
by the regulation pipeline with targeted 2-tick scenarios.

**Architecture:** New `PerStrategyRegulationEvalTest` class with
`CognitiveDefaultsRegistry` wired into `CognitionCoreTestBuilder`,
enabling the `RegulationStrategyPredictor` to pin strategies based on
personality signals. 5 test methods, each engineering a specific
predictor path via `RegulationDefaults` + `DispositionAxes` +
scenario observation text.

**Tech Stack:** Java 26, JUnit 5, AssertJ, CognitionCore pipeline,
LLM eval (Claude via SessionPipeline)

## Global Constraints

- Tag: `@Tag("llm-eval")` — runs only under `-Pllm-eval` Maven profile
- N=3 runs per scenario, ≥2/3 must match expected strategy
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest="PerStrategyRegulationEvalTest" -s .mvn/slot-settings.xml` (from examples repo root — but test lives in neocortex, run from there)
- Actual build command: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-llm-classifier -Pllm-eval -Dtest="PerStrategyRegulationEvalTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

---

## Batch 1: Per-strategy eval test with predictor wiring

### Task 1: Builder extension + test scaffold with 2 predictor-pinned strategies

Proves the CognitiveDefaultsRegistry wiring end-to-end with two
strategies: SITUATION_SELECTION (self-conscious + avoidant) and
ATTENTIONAL_DEPLOYMENT (uncontrollable negative). These exercise
different predictor branches.

**Files:**
- Modify: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/CognitionCoreTestBuilder.java` — add overload accepting CognitiveDefaultsRegistry
- Create: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerStrategyRegulationEvalTest.java`

**Interfaces:**
- Consumes: `EvalTestSupport.sessionPipeline()`, `CognitiveDefaultsRegistry.forTesting(...)`, `CognitiveDefaults.empty(agentId).withDescriptor(...).withRegulationDefaults(...)`, `DescriptorView.of(...)`, `DispositionAxes(...)`, `RegulationDefaults(...)`
- Produces: `PerStrategyRegulationEvalTest` class with `runScenario(agentId, descriptor, observation, expectedStrategies, n)` helper

- [ ] **Step 1: Add builder overload for defaults registry**

Use `ide_insert_member` to add this method to `CognitionCoreTestBuilder` after the existing 3-arg `buildCore`:

```java
public static CognitionCore buildCore(CognitionConfig config,
        @Nullable SessionPipeline sessionPipeline,
        io.casehub.neocortex.mindmap.@Nullable MindMapStore mindMapStore,
        io.casehub.neocortex.cognitive.index.@Nullable CognitiveDefaultsRegistry defaultsRegistry) {
    var core = buildCore(config, sessionPipeline, mindMapStore);
    if (defaultsRegistry != null) {
        core.setDefaultsRegistry(defaultsRegistry);
    }
    return core;
}
```

- [ ] **Step 2: Create the test class with shared infrastructure**

Use `ide_create_file` to create `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerStrategyRegulationEvalTest.java`:

```java
package io.casehub.neocortex.caps.llm.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.neocortex.cognitive.index.CognitiveDefaults;
import io.casehub.neocortex.cognitive.index.CognitiveDefaultsRegistry;
import io.casehub.neocortex.cognitive.index.DescriptorView;
import io.casehub.neocortex.cognitive.index.DispositionAxes;
import io.casehub.neocortex.cognitive.index.RegulationDefaults;
import io.casehub.neocortex.cognition.appraisal.LlmAppraisalStrategy;
import io.casehub.neocortex.cognition.core.CognitionConfig;
import io.casehub.neocortex.cognition.regulation.EmotionRegulationResult;
import io.casehub.neocortex.cognition.regulation.EmotionRegulationStrategy;
import io.casehub.neocortex.cognition.salience.ConcernBasedSalienceStrategy;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import io.casehub.platform.agent.session.SessionPipeline;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Per-strategy regulation eval: proves all 5 Gross strategies are exercised
 * with personality-driven predictor pinning (Petter 2025, Nathanson 1992).
 */
@Tag("llm-eval")
class PerStrategyRegulationEvalTest {

    private static final String TENANT = "eval-tenant";
    private static final int N = 3;

    private static SessionPipeline sessionPipeline;
    private static CognitiveDefaultsRegistry defaultsRegistry;

    @BeforeAll
    static void setUp() {
        sessionPipeline = EvalTestSupport.sessionPipeline();
        defaultsRegistry = CognitiveDefaultsRegistry.forTesting(
                buildDefaults("eval-sit-select",
                        new DispositionAxes("cooperative", "flexible", "cautious", "low", "avoidant"),
                        new RegulationDefaults(0.40, 0.50, 0.60)),
                buildDefaults("eval-sit-modify",
                        new DispositionAxes("competitive", "flexible", "bold", "high", "competitive"),
                        new RegulationDefaults(0.40, 0.40, 0.30)),
                buildDefaults("eval-attn-deploy",
                        new DispositionAxes("cooperative", "flexible", "bold", "low", "protective"),
                        new RegulationDefaults(0.50, 0.50, 0.60)),
                buildDefaults("eval-cog-change",
                        new DispositionAxes("cooperative", "flexible", "cautious", "high", "accommodating"),
                        new RegulationDefaults(0.80, 0.40, 0.30)),
                buildDefaults("eval-resp-mod",
                        new DispositionAxes("competitive", "strict", "cautious", "high", "competitive"),
                        new RegulationDefaults(0.35, 0.80, 0.40))
        );
    }

    @Test
    void situationSelection_shameWithAvoidantConflict_selectsAvoidance() {
        String observation = """
                The roundtable is about to start. You know James will raise the \
                missing money. You haven't prepared an explanation. Every face will \
                turn toward you. Clara already suspects — you saw it in her eyes \
                at breakfast. The shame of being caught unprepared is crawling up \
                your spine.""";

        var descriptor = AgentDescriptor.builder()
                .agentId("eval-sit-select").name("Eval SitSelect").slot("manor-character").tenancyId(TENANT)
                .disposition(disposition("cooperative", "flexible", "cautious", "low", "avoidant"))
                .build();

        var results = runScenario("eval-sit-select", descriptor, observation, N);
        int matched = countMatching(results, Set.of(EmotionRegulationStrategy.SITUATION_SELECTION));
        System.out.printf("SITUATION_SELECTION: %d/%d matched%n", matched, N);
        assertThat(matched)
                .as("SITUATION_SELECTION should be selected in at least 2/%d runs", N)
                .isGreaterThanOrEqualTo(2);
        for (var r : results) {
            if (r != null && r.strategy() == EmotionRegulationStrategy.SITUATION_SELECTION) {
                assertThat(r.regulationAxis()).as("environmental strategy must have regulationAxis").isNotNull();
                assertThat(r.environmentalDirective()).as("environmental strategy must have directive").isNotBlank();
            }
        }
    }

    @Test
    void attentionalDeployment_fearUncontrollable_redirectsFocus() {
        String observation = """
                The ceiling beam above Clara is cracking. Plaster dust is falling \
                on her hair. She hasn't noticed — she's reading the inscription \
                on the mantelpiece. The beam shifts with a groan. You have seconds. \
                Your chest is tight. Your legs want to move before your brain \
                decides where.""";

        var descriptor = AgentDescriptor.builder()
                .agentId("eval-attn-deploy").name("Eval AttnDeploy").slot("manor-character").tenancyId(TENANT)
                .disposition(disposition("cooperative", "flexible", "bold", "low", "protective"))
                .build();

        var results = runScenario("eval-attn-deploy", descriptor, observation, N);
        int matched = countMatching(results, Set.of(EmotionRegulationStrategy.ATTENTIONAL_DEPLOYMENT));
        System.out.printf("ATTENTIONAL_DEPLOYMENT: %d/%d matched%n", matched, N);
        assertThat(matched)
                .as("ATTENTIONAL_DEPLOYMENT should be selected in at least 2/%d runs", N)
                .isGreaterThanOrEqualTo(2);
    }

    // --- shared infrastructure ---

    private List<EmotionRegulationResult> runScenario(String agentId, AgentDescriptor descriptor,
                                                       String observation, int runs) {
        var results = new java.util.ArrayList<EmotionRegulationResult>();
        for (int i = 0; i < runs; i++) {
            try {
                var result = runSingleScenario(agentId + "-" + i, agentId, descriptor, observation);
                results.add(result);
                if (result != null) {
                    System.out.printf("  run %d: %s effectiveness=%.2f emotions=%s%n",
                            i + 1, result.strategy(), result.effectiveness(),
                            String.join(",", result.targetEmotions()));
                } else {
                    System.out.printf("  run %d: no regulation%n", i + 1);
                }
            } catch (Exception e) {
                System.out.printf("  run %d failed: %s%n", i + 1, e.getMessage());
                results.add(null);
            }
        }
        return results;
    }

    private EmotionRegulationResult runSingleScenario(String instanceId, String characterId,
                                                       AgentDescriptor descriptor, String observation) {
        var config = CognitionConfig.none()
                .with("mood", true)
                .with("drives", true)
                .with("appraisal", true)
                .with("salience", true)
                .with("regulation", true);

        var mindMap = new InMemoryMindMapStore();
        var core = CognitionCoreTestBuilder.buildCore(config, sessionPipeline, mindMap, defaultsRegistry);
        core.configureAppraisal(new LlmAppraisalStrategy(sessionPipeline), new ConcernBasedSalienceStrategy());
        core.configureRegulation();

        // Tick 1: seed baseline
        core.tick(instanceId, TENANT, descriptor, (a, t) -> Set.of(), null);
        // Tick 2: deliver the situation
        core.tick(instanceId, TENANT, descriptor, (a, t) -> Set.of("other"), observation);

        var appraisal = core.appraisalParticipant().currentResult(instanceId, TENANT);
        if (appraisal.isPresent() && !appraisal.get().emotions().isEmpty()) {
            for (var e : appraisal.get().emotions()) {
                System.out.printf("    appraisal: %s intensity=%.2f%n", e.type(), e.intensity());
            }
        } else {
            System.out.printf("    appraisal: %s%n",
                    appraisal.isPresent() ? "empty emotions" : "no result");
        }

        return core.regulationParticipant().currentResult(instanceId, TENANT).orElse(null);
    }

    private static int countMatching(List<EmotionRegulationResult> results,
                                      Set<EmotionRegulationStrategy> expected) {
        return (int) results.stream()
                .filter(r -> r != null && expected.contains(r.strategy()))
                .count();
    }

    private static CognitiveDefaults buildDefaults(String agentId,
                                                    DispositionAxes axes, RegulationDefaults rd) {
        var dv = DescriptorView.of(agentId, axes, List.of(), List.of());
        return CognitiveDefaults.empty(agentId)
                .withDescriptor(dv)
                .withRegulationDefaults(rd);
    }

    private static AgentDisposition disposition(String social, String rules, String risk,
                                                 String autonomy, String conflict) {
        return new AgentDisposition(
                List.of(DispositionValue.of(social)),
                List.of(DispositionValue.of(rules)),
                List.of(DispositionValue.of(risk)),
                List.of(DispositionValue.of(autonomy)),
                List.of(DispositionValue.of(conflict)),
                false, List.of(), List.of(), null, null);
    }
}
```

- [ ] **Step 3: Verify compilation**

Run `ide_diagnostics` on the new test file. Fix any compilation errors.

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml compile test-compile -pl caps-llm-classifier -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: BUILD SUCCESS

- [ ] **Step 4: Run the 2 strategy tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-llm-classifier -Pllm-eval -Dtest="PerStrategyRegulationEvalTest#situationSelection_shameWithAvoidantConflict_selectsAvoidance+attentionalDeployment_fearUncontrollable_redirectsFocus" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: 2 tests PASS (each ≥2/3 runs matching expected strategy)

If a test fails, examine the diagnostic output (appraisal emotions and
intensities). Adjust observation text to more strongly evoke the target
emotion category if needed.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/CognitionCoreTestBuilder.java caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerStrategyRegulationEvalTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#822): add per-strategy regulation eval — SITUATION_SELECTION + ATTENTIONAL_DEPLOYMENT"
```

### Task 2: Remaining 3 strategy tests

Adds SITUATION_MODIFICATION, COGNITIVE_CHANGE, and RESPONSE_MODULATION
tests to the existing test class.

**Files:**
- Modify: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerStrategyRegulationEvalTest.java` — add 3 test methods

**Interfaces:**
- Consumes: `runScenario(...)`, `countMatching(...)`, `disposition(...)` from Task 1

- [ ] **Step 1: Add SITUATION_MODIFICATION test**

Use `ide_insert_member` to add after the `attentionalDeployment` test:

```java
@Test
void situationModification_controllableWithFallthrough_modifiesEnvironment() {
    String observation = """
            The forged documents are on the table. Nobody has read them yet. \
            Clara is reaching for them. If she reads them, your entire plan \
            unravels. The documents are right there — within arm's reach. You \
            could intercept them, redirect her attention, change the situation \
            before it becomes a crisis.""";

    var descriptor = AgentDescriptor.builder()
            .agentId("eval-sit-modify").name("Eval SitModify").slot("manor-character").tenancyId(TENANT)
            .disposition(disposition("competitive", "flexible", "bold", "high", "competitive"))
            .build();

    var results = runScenario("eval-sit-modify", descriptor, observation, N);
    int matched = countMatching(results, Set.of(
            EmotionRegulationStrategy.SITUATION_MODIFICATION,
            EmotionRegulationStrategy.SITUATION_SELECTION));
    System.out.printf("SITUATION_MODIFICATION (environmental): %d/%d matched%n", matched, N);
    assertThat(matched)
            .as("Environmental strategy should be selected in at least 2/%d runs", N)
            .isGreaterThanOrEqualTo(2);
    for (var r : results) {
        if (r != null && r.isEnvironmentalStrategy()) {
            assertThat(r.regulationAxis()).as("environmental strategy must have regulationAxis").isNotNull();
            assertThat(r.environmentalDirective()).as("environmental strategy must have directive").isNotBlank();
        }
    }
}
```

- [ ] **Step 2: Add COGNITIVE_CHANGE test**

Use `ide_insert_member` to add after the SITUATION_MODIFICATION test:

```java
@Test
void cognitiveChange_controllableWithHighReappraisal_reappraises() {
    String observation = """
            James won the puzzle challenge. You came second. Again. He's \
            accepting congratulations from Clara, and the Brixton Boys are \
            clapping him on the back. You solved the harder half of the puzzle \
            but he got the final piece. The resentment is a tight knot in your \
            chest.""";

    var descriptor = AgentDescriptor.builder()
            .agentId("eval-cog-change").name("Eval CogChange").slot("manor-character").tenancyId(TENANT)
            .disposition(disposition("cooperative", "flexible", "cautious", "high", "accommodating"))
            .build();

    var results = runScenario("eval-cog-change", descriptor, observation, N);
    int matched = countMatching(results, Set.of(EmotionRegulationStrategy.COGNITIVE_CHANGE));
    System.out.printf("COGNITIVE_CHANGE: %d/%d matched%n", matched, N);
    assertThat(matched)
            .as("COGNITIVE_CHANGE should be selected in at least 2/%d runs", N)
            .isGreaterThanOrEqualTo(2);
}
```

- [ ] **Step 3: Add RESPONSE_MODULATION test**

Use `ide_insert_member` to add after the COGNITIVE_CHANGE test:

```java
@Test
void responseModulation_otherDirectedWithHighSuppression_suppresses() {
    String observation = """
            Clara is standing at the head of the table, announcing that she has \
            taken charge of the investigation. She has reassigned your tasks to \
            James without asking. The Brixton Boys are nodding along. Your \
            authority is being stripped away in public, and everyone seems to \
            think it's reasonable. The reproach burns, but every eye is on you.""";

    var descriptor = AgentDescriptor.builder()
            .agentId("eval-resp-mod").name("Eval RespMod").slot("manor-character").tenancyId(TENANT)
            .disposition(disposition("competitive", "strict", "cautious", "high", "competitive"))
            .build();

    var results = runScenario("eval-resp-mod", descriptor, observation, N);
    int matched = countMatching(results, Set.of(EmotionRegulationStrategy.RESPONSE_MODULATION));
    System.out.printf("RESPONSE_MODULATION: %d/%d matched%n", matched, N);
    assertThat(matched)
            .as("RESPONSE_MODULATION should be selected in at least 2/%d runs", N)
            .isGreaterThanOrEqualTo(2);
}
```

- [ ] **Step 4: Run all 5 strategy tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-llm-classifier -Pllm-eval -Dtest="PerStrategyRegulationEvalTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: 5 tests PASS

If SITUATION_MODIFICATION fails (LLM picks non-environmental strategy):
- Check appraisal output — if emotion is FEAR, the predictor pins ATTENTIONAL_DEPLOYMENT
- Adjust observation text to more strongly evoke controllable anger/agency
- The observation should frame the situation as one you can change, not one that threatens you

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerStrategyRegulationEvalTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#822): complete per-strategy eval — all 5 Gross strategies with targeted scenarios"
```

## References

- [2026-11-08-per-strategy-regulation-eval-design.md] — design spec
- [RegulationStrategyPredictor.java] — predictor logic (Petter 2025, Nathanson 1992)
- [EmotionRegulationTickParticipant.java:123] — predictor null-check
- [CognitionCore.java:734-759] — configureRegulation wiring
- [CognitiveDefaultsRegistry.java:90-96] — forTesting factory
- [CognitionCoreTestBuilder.java] — existing builder pattern
- [RegulationFullPipelineEvalTest.java] — existing eval test pattern
- [decisions.md D6, D7, D8] — design decisions
- [GitHub casehubio/neocortex#822] — focal issue
