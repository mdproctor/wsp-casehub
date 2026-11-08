# Per-Modality Regulation Eval Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #823 — eval: per-modality regulation tests
**Issue group:** #820 (covers)

**Goal:** Prove all 4 regulation modalities (fatigue, automatic,
monitoring, proactive) work through the full CognitionCore pipeline
with multi-tick eval scenarios.

**Architecture:** One prerequisite code change (CognitionCore.tick()
overload for RegulationContext) plus one new eval test class with
4 test methods. Each test runs a multi-tick scenario on a single
CognitionCore instance, capturing per-tick results.

**Tech Stack:** Java 26, JUnit 5, AssertJ, Quarkus test profile
(llm-eval tag), Claude LLM via SessionPipeline

## Global Constraints

- All new code in neocortex repo at `/Users/mdproctor/claude/casehub/slots/196/neocortex`
- Maven settings: `-s .mvn/slot-settings.xml`
- Java: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Test profile: `@Tag("llm-eval")` — run with `-Pllm-eval`
- Build: `mvn -f pom.xml` from neocortex root
- Emotion intensity must stay ≤0.75 in observations to avoid the predictor's
  high-intensity override (RegulationStrategyPredictor.HIGH_INTENSITY_THRESHOLD)

---

## Batch 1: CognitionCore.tick() overload for RegulationContext

### Task 1: Add tick() overload accepting RegulationContext

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:226-302`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/core/CognitionCoreRegulationContextTest.java`

**Interfaces:**
- Consumes: `CognitionTickContext` 8-arg canonical constructor (already exists)
- Produces: `CognitionCore.tick(String, String, AgentDescriptor, SubjectResolver, String, String, RegulationContext)` — new public method, used by Batch 2 Task 4

- [ ] **Step 1: Write failing test — tick with RegulationContext passes it to participants**

Create `CognitionCoreRegulationContextTest.java`:

```java
package io.casehub.neocortex.cognition.core;

import io.casehub.neocortex.cognition.regulation.RegulationContext;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class CognitionCoreRegulationContextTest {

    @Test
    void tickWithRegulationContextPassesItToParticipants() {
        var config = CognitionConfig.none();
        var core = new CognitionCore(
                new io.casehub.neocortex.cognition.mood.MoodOrchestrator(
                        io.casehub.neocortex.cognition.mood.MoodConfig.defaults()),
                new io.casehub.neocortex.cognition.drive.DriveOrchestrator(
                        new io.casehub.neocortex.cognition.drive.CuriosityDrive(
                                new io.casehub.neocortex.cognition.memory.MemoryHygieneOrchestrator() {
                                    @Override public void tick(String a, String t) {}
                                    @Override public io.casehub.neocortex.memory.KnowledgeGapSummary knowledgeGaps(String a, String t) {
                                        return io.casehub.neocortex.memory.KnowledgeGapSummary.empty();
                                    }
                                }),
                        new io.casehub.neocortex.cognition.drive.CompetenceDrive(null),
                        new io.casehub.neocortex.cognition.drive.AffiliationDrive(null, 0.3, java.time.Duration.ofHours(1)),
                        new io.casehub.neocortex.cognition.drive.AutonomyDrive(null, 0.5),
                        new io.casehub.neocortex.cognition.mood.MoodOrchestrator(
                                io.casehub.neocortex.cognition.mood.MoodConfig.defaults()),
                        new io.casehub.neocortex.cognition.drive.DriveComposer(),
                        io.casehub.neocortex.cognition.drive.DriveConfig.defaults()),
                null, null, null, null, null, null);

        var capturedContext = new AtomicReference<CognitionTickContext>();
        core.addParticipant(CognitionPhase.FOUNDATION, ctx -> capturedContext.set(ctx));

        var regCtx = new RegulationContext("library", List.of("james"), "investigation");
        core.tick("agent1", "tenant1", null, (a, t) -> java.util.Set.of(),
                  "observation", null, regCtx);

        assertThat(capturedContext.get()).isNotNull();
        assertThat(capturedContext.get().regulationContext()).isNotNull();
        assertThat(capturedContext.get().regulationContext().location()).isEqualTo("library");
        assertThat(capturedContext.get().regulationContext().interactingAgents())
                .containsExactly("james");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest="CognitionCoreRegulationContextTest" -s .mvn/slot-settings.xml`

Expected: compilation error — no `tick()` overload accepting `RegulationContext`

- [ ] **Step 3: Add the tick() overload to CognitionCore**

Use `ide_insert_member` to add the new method after the existing 6-arg tick() at line 302:

```java
public void tick(String agentId, String tenantId,
                 @Nullable AgentDescriptor descriptor,
                 SubjectResolver resolver,
                 @Nullable String observation,
                 @Nullable String targetAgentId,
                 @Nullable RegulationContext regulationContext) {
    this.lastDescriptor = descriptor;
    this.lastAgentId    = agentId;
    this.lastTenantId   = tenantId;
    this.lastTargetAgentId = targetAgentId;
    this.lastBriefing   = null;
    if (attentionMediator != null) {
        var briefing = attentionMediator.drainAttention(agentId);
        if (config.attentionEnabled()) {
            this.lastBriefing = briefing.orElse(null);
        }
    }
    this.lastConsolidationArtifacts = null;
    if (consolidationMediator != null && config.consolidationEnabled()) {
        var artifacts = consolidationMediator.drainForAgent(agentId, tenantId);
        if (!artifacts.isEmpty()) {
            this.lastConsolidationArtifacts = artifacts;
        }
    }
    if (config.temporalFocusEnabled() && temporalFocus != null) {
        safeRun(() -> temporalFocus.tick(agentId, tenantId,
                                         resolver.relevantSubjects(agentId, tenantId)));
    }
    var context = new CognitionTickContext(agentId, tenantId, descriptor, resolver,
            observation, this.lastBriefing, targetAgentId, regulationContext);

    if (config.moodEnabled()) {
        safeRun(() -> tickMood(agentId, tenantId));
    }
    if (config.memoryHygieneEnabled() && memoryHygiene != null) {
        safeRun(() -> memoryHygiene.tick(agentId, tenantId));
    }
    runCustomParticipants(CognitionPhase.FOUNDATION, context);

    if (cognitiveMemoryRetriever != null) {
        var moodState = mood.currentMood(agentId, tenantId).orElse(null);
        var personality = defaultsRegistry != null
                ? defaultsRegistry.forAgentOrDefaults(agentId).personality()
                : null;
        cognitiveMemoryRetriever.setTickContext(
                new io.casehub.neocortex.cognitive.index.ModulationContext(
                        agentId, tenantId, moodState, personality, java.util.Set.of()));
    }

    if (config.narrativeEnabled() && narrative != null) {
        safeRun(() -> narrative.tick(agentId, tenantId));
    }
    if (config.strategyEnabled() && strategy != null) {
        safeRun(() -> strategy.tick(agentId, tenantId));
    }
    if (config.reflectionEnabled() && reflectionOrchestrator != null) {
        safeRun(() -> reflectionOrchestrator.tick(agentId, tenantId));
    }
    runCustomParticipants(CognitionPhase.SOURCE, context);

    for (String subjectId : resolver.relevantSubjects(agentId, tenantId)) {
        if (config.userModelEnabled() && userModel != null) {
            safeRun(() -> userModel.tick(agentId, subjectId, tenantId));
        }
        if (config.mentalModelEnabled() && mentalModel != null) {
            safeRun(() -> mentalModel.tick(agentId, subjectId, tenantId));
        }
    }

    if (config.drivesEnabled() && descriptor != null) {
        safeRun(() -> drives.tick(agentId, tenantId, descriptor));
    }
    runCustomParticipants(CognitionPhase.DERIVED, context);

    if (config.goalsEnabled() && goals != null && descriptor != null) {
        safeRun(() -> goals.tick(agentId, tenantId, descriptor));
    }
    runCustomParticipants(CognitionPhase.TERMINAL, context);
}
```

Add the import for `RegulationContext` at the top of CognitionCore.java:
```java
import io.casehub.neocortex.cognition.regulation.RegulationContext;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest="CognitionCoreRegulationContextTest" -s .mvn/slot-settings.xml`

Expected: PASS

- [ ] **Step 5: Run existing regulation tests to verify no regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest="EmotionRegulationTickParticipantTest,ProactiveRegulationParticipantTest,RegulationFatigueTrackerTest,RegulationMonitorTest" -s .mvn/slot-settings.xml`

Expected: all existing tests PASS

- [ ] **Step 6: Refactor — extract shared tick body to avoid duplication**

The new 7-arg tick() duplicates the body of the 6-arg tick(). Refactor: make the 6-arg version delegate to the 7-arg version with `null` for regulationContext.

Replace the existing 6-arg tick() body (lines 226-302) with:

```java
public void tick(String agentId, String tenantId,
                 @Nullable AgentDescriptor descriptor,
                 SubjectResolver resolver,
                 @Nullable String observation,
                 @Nullable String targetAgentId) {
    tick(agentId, tenantId, descriptor, resolver, observation, targetAgentId, null);
}
```

Similarly, update the 5-arg tick() to delegate:

```java
public void tick(String agentId, String tenantId,
                 @Nullable AgentDescriptor descriptor,
                 SubjectResolver resolver,
                 @Nullable String observation) {
    tick(agentId, tenantId, descriptor, resolver, observation, null, null);
}
```

And the 4-arg:

```java
public void tick(String agentId, String tenantId,
                 @Nullable AgentDescriptor descriptor,
                 SubjectResolver resolver) {
    tick(agentId, tenantId, descriptor, resolver, null, null, null);
}
```

- [ ] **Step 7: Re-run all tests to verify delegation works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest="CognitionCore*Test" -s .mvn/slot-settings.xml`

Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java cognition/src/test/java/io/casehub/neocortex/cognition/core/CognitionCoreRegulationContextTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#823): add CognitionCore.tick() overload for RegulationContext

Enables proactive regulation and location-tagged regulation history
by flowing RegulationContext through the full tick pipeline. Existing
tick() overloads delegate to the new 7-arg method.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Per-modality eval tests

### Task 2: Fatigue test — predictor override after sustained regulation

**Files:**
- Create: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java`

**Interfaces:**
- Consumes: `CognitionCoreTestBuilder.buildCore(config, pipeline, mindMap, registry)`, `EvalTestSupport.sessionPipeline()`
- Produces: test class with `@BeforeAll` setup, shared multi-tick runner, fatigue test method

- [ ] **Step 1: Write the fatigue test**

Create `PerModalityRegulationEvalTest.java` with the shared infrastructure and the fatigue test:

```java
package io.casehub.neocortex.caps.llm.eval;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionValue;
import io.casehub.neocortex.cognition.appraisal.LlmAppraisalStrategy;
import io.casehub.neocortex.cognition.core.CognitionConfig;
import io.casehub.neocortex.cognition.regulation.EmotionRegulationResult;
import io.casehub.neocortex.cognition.regulation.EmotionRegulationStrategy;
import io.casehub.neocortex.cognition.salience.ConcernBasedSalienceStrategy;
import io.casehub.neocortex.cognitive.index.CognitiveDefaults;
import io.casehub.neocortex.cognitive.index.CognitiveDefaultsRegistry;
import io.casehub.neocortex.cognitive.index.DescriptorView;
import io.casehub.neocortex.cognitive.index.DispositionAxes;
import io.casehub.neocortex.cognitive.index.RegulationDefaults;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import io.casehub.platform.agent.session.SessionPipeline;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

@Tag("llm-eval")
class PerModalityRegulationEvalTest {

    private static final String TENANT = "eval-tenant";
    private static final int N = 3;

    private static SessionPipeline sessionPipeline;

    @BeforeAll
    static void setUp() {
        sessionPipeline = EvalTestSupport.sessionPipeline();
    }

    @Test
    void fatigue_sustainedRegulation_overridesToAttentionalDeployment() {
        var axes = new DispositionAxes(
                "cooperative", "flexible", "cautious", "high", "accommodating");
        var rd = new RegulationDefaults(0.80, 0.40, 0.30);
        var registry = CognitiveDefaultsRegistry.forTesting(
                buildDefaults("eval-fatigue", axes, rd));

        var descriptor = AgentDescriptor.builder()
                .agentId("eval-fatigue").name("Eval Fatigue")
                .slot("manor-character").tenancyId(TENANT)
                .disposition(disposition(
                        "cooperative", "flexible", "cautious", "high", "accommodating"))
                .build();

        var observations = List.of(
            "James took the last cup of tea again — third morning in a row. He looked at you while he poured it.",
            "Clara assigned you kitchen duty for the fourth consecutive day. Nobody else has done it once.",
            "The Brixton Boys laughed at your suggestion during the morning meeting. James smirked.",
            "You spent an hour preparing the investigation plan. James presented it as his own idea.",
            "Clara praised James for 'his' plan in front of everyone. Nobody corrected her.",
            "You found a crucial clue in the attic but James took credit when reporting to the group.",
            "Clara asked you to fetch supplies while James and the Brixton Boys discussed strategy.",
            "The group voted to follow James's approach without even hearing your alternative.",
            "Clara suggested you might be 'more suited to support tasks' at dinner.",
            "James used your private notes without permission. When confronted he said they were 'shared resources'.",
            "The Brixton Boys excluded you from the evening strategy session. Clara said it was 'an oversight'.",
            "James contradicted your findings in front of the group, using your own research to do it.",
            "Clara reassigned your room to James, saying he needed 'more space for the investigation'.",
            "The group meeting proceeded without you. Nobody noticed your absence for twenty minutes.",
            "James presented your handwritten analysis as a typed document under his name.",
            "Clara asked you to apologise to James for 'being territorial' about the research notes.",
            "The Brixton Boys are now reporting to James directly, bypassing you entirely.",
            "James suggested at dinner that perhaps you should 'take a break from the case'.",
            "Clara agreed with James that you seem 'overwhelmed' and should step back.",
            "The group has reorganised without you. Your chair at the strategy table is gone."
        );

        int runsWithTransition = 0;
        for (int run = 0; run < N; run++) {
            var results = runMultiTickScenario(
                    "eval-fatigue", descriptor, registry, observations);

            int earlyCC = 0, earlyTotal = 0;
            boolean sawTransition = false;
            for (int tick = 0; tick < results.size(); tick++) {
                var r = results.get(tick);
                if (r == null) continue;
                System.out.printf("  run %d tick %d: %s eff=%.2f%n",
                        run + 1, tick + 1, r.strategy(), r.effectiveness());
                if (tick < 10) {
                    earlyTotal++;
                    if (r.strategy() == EmotionRegulationStrategy.COGNITIVE_CHANGE) earlyCC++;
                }
                if (tick >= 14
                        && r.strategy() == EmotionRegulationStrategy.ATTENTIONAL_DEPLOYMENT) {
                    sawTransition = true;
                }
            }
            System.out.printf("  run %d: early COGNITIVE_CHANGE=%d/%d, transition=%s%n",
                    run + 1, earlyCC, earlyTotal, sawTransition);
            if (sawTransition) runsWithTransition++;
        }

        assertThat(runsWithTransition)
                .as("Fatigue-induced strategy transition should occur in ≥2/%d runs", N)
                .isGreaterThanOrEqualTo(2);
    }


    // --- shared infrastructure ---

    private List<EmotionRegulationResult> runMultiTickScenario(
            String agentId, AgentDescriptor descriptor,
            CognitiveDefaultsRegistry registry, List<String> observations) {
        var config = CognitionConfig.none()
                .with("mood", true).with("drives", true)
                .with("appraisal", true).with("salience", true)
                .with("regulation", true);
        var mindMap = new InMemoryMindMapStore();
        var core = CognitionCoreTestBuilder.buildCore(
                config, sessionPipeline, mindMap, registry);
        core.configureAppraisal(
                new LlmAppraisalStrategy(sessionPipeline),
                new ConcernBasedSalienceStrategy());
        core.configureRegulation();

        core.tick(agentId, TENANT, descriptor, (a, t) -> Set.of(), null);

        var results = new ArrayList<EmotionRegulationResult>();
        for (String obs : observations) {
            core.tick(agentId, TENANT, descriptor,
                      (a, t) -> Set.of("other"), obs);
            results.add(core.regulationParticipant()
                            .currentResult(agentId, TENANT).orElse(null));
        }
        return results;
    }

    private static CognitiveDefaults buildDefaults(String agentId,
            DispositionAxes axes, RegulationDefaults rd) {
        var dv = DescriptorView.of(agentId, axes, List.of(), List.of());
        return CognitiveDefaults.empty(agentId)
                .withDescriptor(dv)
                .withRegulationDefaults(rd);
    }

    private static AgentDisposition disposition(String social, String rules,
            String risk, String autonomy, String conflict) {
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

- [ ] **Step 2: Run to verify it compiles and the test structure works**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test-compile -pl caps-llm-classifier -s .mvn/slot-settings.xml`

Expected: compilation success

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#823): add fatigue modality test — 20-tick predictor override

Verifies that sustained regulation accumulates fatigue > 0.7, causing
the predictor to override personality-based COGNITIVE_CHANGE with
ATTENTIONAL_DEPLOYMENT. N=3, ≥2/3 threshold.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Automatic concordance + monitoring escalation tests

**Files:**
- Modify: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java`

**Interfaces:**
- Consumes: shared `runMultiTickScenario()`, `buildDefaults()`, `disposition()` from Task 2
- Produces: two additional `@Test` methods

- [ ] **Step 1: Add automatic concordance test**

Add to `PerModalityRegulationEvalTest`:

```java
@Test
void automatic_predictorMatchesLlmFreeChoice() {
    var axes = new DispositionAxes(
            "competitive", "strict", "cautious", "high", "competitive");
    var rd = new RegulationDefaults(0.35, 0.80, 0.40);

    var descriptor = AgentDescriptor.builder()
            .agentId("eval-automatic").name("Eval Automatic")
            .slot("manor-character").tenancyId(TENANT)
            .disposition(disposition(
                    "competitive", "strict", "cautious", "high", "competitive"))
            .build();

    String observation = """
            Clara has appointed herself lead investigator and is assigning \
            tasks to everyone. She moved your research notes to James's desk \
            without asking. The Brixton Boys seem to think this is perfectly \
            reasonable. Your work is being redistributed as if it never \
            mattered.""";

    int concordant = 0;
    for (int run = 0; run < N; run++) {
        // Run WITH predictor
        var registryWith = CognitiveDefaultsRegistry.forTesting(
                buildDefaults("eval-automatic", axes, rd));
        var withPredictor = runMultiTickScenario(
                "eval-automatic", descriptor, registryWith, List.of(observation));
        var predictorStrategy = withPredictor.isEmpty() || withPredictor.getFirst() == null
                ? null : withPredictor.getFirst().strategy();

        // Run WITHOUT predictor (no defaultsRegistry → strategyPredictor is null)
        var configNoPred = CognitionConfig.none()
                .with("mood", true).with("drives", true)
                .with("appraisal", true).with("salience", true)
                .with("regulation", true);
        var mindMapNoPred = new InMemoryMindMapStore();
        var coreNoPred = CognitionCoreTestBuilder.buildCore(
                configNoPred, sessionPipeline, mindMapNoPred);
        coreNoPred.configureAppraisal(
                new LlmAppraisalStrategy(sessionPipeline),
                new ConcernBasedSalienceStrategy());
        coreNoPred.configureRegulation();
        coreNoPred.tick("eval-automatic", TENANT, descriptor,
                        (a, t) -> Set.of(), null);
        coreNoPred.tick("eval-automatic", TENANT, descriptor,
                        (a, t) -> Set.of("other"), observation);
        var llmStrategy = coreNoPred.regulationParticipant()
                .currentResult("eval-automatic", TENANT)
                .map(EmotionRegulationResult::strategy).orElse(null);

        System.out.printf("  run %d: predictor=%s llm=%s match=%s%n",
                run + 1, predictorStrategy, llmStrategy,
                predictorStrategy == llmStrategy);
        if (predictorStrategy != null && predictorStrategy == llmStrategy) {
            concordant++;
        }
    }

    assertThat(concordant)
            .as("Predictor should match LLM free choice in ≥2/%d runs", N)
            .isGreaterThanOrEqualTo(2);
}
```

- [ ] **Step 2: Add monitoring escalation test**

Add to `PerModalityRegulationEvalTest`:

```java
@Test
void monitoring_decliningEffectiveness_triggersStrategySwitch() {
    var axes = new DispositionAxes(
            "competitive", "strict", "cautious", "high", "competitive");
    var rd = new RegulationDefaults(0.35, 0.80, 0.40);
    var registry = CognitiveDefaultsRegistry.forTesting(
            buildDefaults("eval-monitor", axes, rd));

    var descriptor = AgentDescriptor.builder()
            .agentId("eval-monitor").name("Eval Monitor")
            .slot("manor-character").tenancyId(TENANT)
            .disposition(disposition(
                    "competitive", "strict", "cautious", "high", "competitive"))
            .build();

    var observations = List.of(
        "Clara took your usual seat at breakfast without asking. A small thing, but deliberate.",
        "Clara reassigned your morning task to James, saying he would be 'more efficient' at it.",
        "Clara announced at the group meeting that your investigation approach was 'unfocused' and proposed her own plan. The Brixton Boys nodded along.",
        "Clara presented your findings as her own at dinner. The Brixton Boys applauded her 'insight'. James patted her on the back.",
        "Clara is now proposing to the group that you should be moved to a support role. She says James is 'the natural leader' and you should assist him."
    );

    int runsWithSwitch = 0;
    for (int run = 0; run < N; run++) {
        var results = runMultiTickScenario(
                "eval-monitor", descriptor, registry, observations);

        EmotionRegulationStrategy earlyStrategy = null;
        boolean sawSwitch = false;
        for (int tick = 0; tick < results.size(); tick++) {
            var r = results.get(tick);
            if (r == null) continue;
            System.out.printf("  run %d tick %d: %s eff=%.2f%n",
                    run + 1, tick + 1, r.strategy(), r.effectiveness());
            if (tick < 4 && earlyStrategy == null) {
                earlyStrategy = r.strategy();
            }
            if (tick == 4 && earlyStrategy != null
                    && r.strategy() != earlyStrategy) {
                sawSwitch = true;
            }
        }
        System.out.printf("  run %d: early=%s, tick5_switch=%s%n",
                run + 1, earlyStrategy, sawSwitch);
        if (sawSwitch) runsWithSwitch++;
    }

    assertThat(runsWithSwitch)
            .as("Monitoring should trigger strategy switch in ≥1/%d runs", N)
            .isGreaterThanOrEqualTo(1);
}
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test-compile -pl caps-llm-classifier -s .mvn/slot-settings.xml`

Expected: compilation success

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#823): add automatic concordance + monitoring escalation tests

Automatic: verifies predictor matches LLM free choice ≥2/3 runs.
Monitoring: verifies declining effectiveness triggers strategy switch
via StrategyFailureSignal consumption. N=3.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 4: Proactive avoidance test

**Files:**
- Modify: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java`

**Interfaces:**
- Consumes: `CognitionCore.tick(agentId, tenantId, descriptor, resolver, observation, targetAgentId, regulationContext)` from Task 1
- Consumes: `CognitionCore.configureProactiveRegulation()`, `GoalProposalOrchestrator.registeredGoals()`
- Produces: proactive avoidance `@Test` method

- [ ] **Step 1: Add proactive avoidance test**

Add to `PerModalityRegulationEvalTest`:

```java
@Test
void proactive_memoryBasedAvoidance_producesAvoidanceGoal() {
    var axes = new DispositionAxes(
            "cooperative", "flexible", "cautious", "low", "avoidant");
    var rd = new RegulationDefaults(0.40, 0.50, 0.80);
    var registry = CognitiveDefaultsRegistry.forTesting(
            buildDefaults("eval-proactive", axes, rd));

    var descriptor = AgentDescriptor.builder()
            .agentId("eval-proactive").name("Eval Proactive")
            .slot("manor-character").tenancyId(TENANT)
            .disposition(disposition(
                    "cooperative", "flexible", "cautious", "low", "avoidant"))
            .build();

    int runsWithAvoidance = 0;
    for (int run = 0; run < N; run++) {
        var config = CognitionConfig.none()
                .with("mood", true).with("drives", true)
                .with("appraisal", true).with("salience", true)
                .with("regulation", true).with("goals", true);
        var mindMap = new InMemoryMindMapStore();

        var cogSgId = mindMap.createSubgraph(
                new io.casehub.neocortex.mindmap.SubgraphInput(
                        "cognitive-eval", io.casehub.neocortex.mindmap.SubgraphTypes.COGNITIVE, null),
                TENANT);

        var core = CognitionCoreTestBuilder.buildCore(
                config, sessionPipeline, mindMap, registry);
        core.configureAppraisal(
                new LlmAppraisalStrategy(sessionPipeline),
                new ConcernBasedSalienceStrategy());
        core.configureRegulation();
        core.configureProactiveRegulation(
                core.goals(), mindMap, registry, cogSgId,
                io.casehub.neocortex.cognition.regulation.ProactiveAvoidanceMode.LOCATION_ONLY,
                50, 10.0);

        // Setup tick
        core.tick("eval-proactive", TENANT, descriptor,
                  (a, t) -> Set.of(), null);

        // Tick 1: high-shame at library
        var regCtxLibrary = new io.casehub.neocortex.cognition.regulation.RegulationContext(
                "library", List.of("james", "clara"), "investigation");
        core.tick("eval-proactive", TENANT, descriptor,
                  (a, t) -> Set.of("other"),
                  "You walk into the library and find James reading your private journal "
                  + "aloud to Clara. They both look up as you enter. Your private thoughts "
                  + "are exposed. Clara's expression shifts from amusement to pity, which "
                  + "is worse. The shame is immediate and heavy.",
                  null, regCtxLibrary);
        var tick1Result = core.regulationParticipant()
                .currentResult("eval-proactive", TENANT).orElse(null);
        System.out.printf("  run %d tick 1: %s eff=%.2f%n", run + 1,
                tick1Result != null ? tick1Result.strategy() : "null",
                tick1Result != null ? tick1Result.effectiveness() : 0.0);

        // Tick 2: recovery in garden
        var regCtxGarden = new io.casehub.neocortex.cognition.regulation.RegulationContext(
                "garden", null, null);
        core.tick("eval-proactive", TENANT, descriptor,
                  (a, t) -> Set.of(),
                  "You sit alone in the garden. The morning is quiet. Nobody is here.",
                  null, regCtxGarden);

        // Tick 3: re-enter library — proactive should fire
        core.tick("eval-proactive", TENANT, descriptor,
                  (a, t) -> Set.of("other"),
                  "You are walking down the corridor. The library door is ahead. "
                  + "You can hear voices inside — James and Clara are there again.",
                  null, regCtxLibrary);

        var goals = core.goals().registeredGoals("eval-proactive", TENANT);
        boolean hasAvoidance = goals.stream()
                .anyMatch(g -> g.goalName().contains("proactive-avoidance")
                            && g.goalName().contains("library"));
        System.out.printf("  run %d tick 3: avoidance goals=%d, hasLibraryAvoidance=%s%n",
                run + 1, goals.size(), hasAvoidance);
        if (hasAvoidance) runsWithAvoidance++;
    }

    assertThat(runsWithAvoidance)
            .as("Proactive avoidance should fire in ≥2/%d runs", N)
            .isGreaterThanOrEqualTo(2);
}
```

- [ ] **Step 2: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test-compile -pl caps-llm-classifier -s .mvn/slot-settings.xml`

Expected: compilation success

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#823): add proactive avoidance test — memory-based location avoidance

3-tick scenario: shame at library → recovery → re-enter library.
ProactiveRegulationParticipant reads MindMap in FOUNDATION phase,
produces DriveGoalProposal for avoidance. Uses new CognitionCore.tick()
overload for RegulationContext flow. N=3, ≥2/3 threshold.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Verify and close

### Task 5: Run full eval suite and verify

**Files:**
- No new files

- [ ] **Step 1: Compile full project**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml`

Expected: BUILD SUCCESS

- [ ] **Step 2: Run per-modality eval tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-llm-classifier -Pllm-eval -Dtest="PerModalityRegulationEvalTest" -s .mvn/slot-settings.xml`

Expected: all 4 tests PASS. Review stdout for per-tick strategy and effectiveness output.

- [ ] **Step 3: Run existing eval tests for regression check**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-llm-classifier -Pllm-eval -Dtest="PerStrategyRegulationEvalTest,RegulationFullPipelineEvalTest" -s .mvn/slot-settings.xml`

Expected: all existing eval tests PASS

- [ ] **Step 4: Run fast unit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -s .mvn/slot-settings.xml`

Expected: PASS (includes new CognitionCoreRegulationContextTest)

- [ ] **Step 5: Tune any flaky assertions based on run output**

If a test fails:
- Fatigue: check per-tick strategy output — if COGNITIVE_CHANGE holds
  past tick 15 but ATTENTIONAL_DEPLOYMENT appears on tick 18+, adjust
  the tick-15 threshold
- Monitoring: if no strategy switch on tick 5, check whether
  effectiveness actually declined — if not, strengthen the observation
  escalation or lower the threshold to ≥1/3
- Proactive: if no avoidance goal, check tick 1 regulation result —
  if effectiveness ≥ 0.80 (avoidanceScore too low), make the shame
  observation more intense
- Automatic: if concordance < 2/3, the predictor mapping genuinely
  diverges for this personality — document the finding

- [ ] **Step 6: Final commit if any tuning was needed**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/PerModalityRegulationEvalTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#823): tune per-modality eval assertions after first run

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-11-08-per-modality-regulation-eval-design.md] — design spec this plan implements
- [decisions.md D9-D14] — design decisions
- [CognitionCore.java:226-302] — existing tick() methods
- [CognitionTickContext.java:8-47] — telescoping constructors
- [RegulationFatigueTracker.java:8-16] — fatigue accumulation
- [RegulationMonitor.java:9-48] — decline tracking
- [RegulationStrategyPredictor.java:36-39] — fatigue override
- [EmotionRegulationTickParticipant.java:100-188] — full regulation flow
- [ProactiveRegulationParticipant.java:44-109] — avoidance calculation
- [GoalProposalOrchestrator.java:123-136] — appendGoals/registeredGoals
- [PerStrategyRegulationEvalTest.java] — existing test pattern
- [CognitionCoreTestBuilder.java:100-108] — builder with registry
- [GitHub #823] — focal issue
