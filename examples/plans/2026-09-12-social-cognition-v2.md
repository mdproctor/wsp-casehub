# Social Cognition Layer Implementation Plan (v2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #52 — Wacky Manor Phase A: social cognition layer
**Issue group:** #52

**Goal:** Wire the existing neocortex/blocks cognitive stack into wacky-manor — mindmap-native cognitive model with Thing trait facades, three-tier memory, and consolidation as a game mechanic.

**Architecture:** Three-tier memory (working → episodic buffer → mindmap). CharacterCognition per character composes CognitiveProfile queries with Thing trait projections. Consolidation during "sleep" graduates Tier 2 → Tier 3. All cognitive state rendered prompt-visible via CognitiveObservationSections.

**Tech Stack:** Java 21, Quarkus, neocortex (mindmap, CognitiveProfile with perspectival resolve/compare, CognitiveDerivationEngine, ConversationBridge, consolidation), blocks (CognitiveObservationSections, DriveProfile, normative, memory scoring), Eidos (extensionData)

## Global Constraints

- Mindmap (Tier 3) is write-on-consolidation only — never written during the tick loop
- Tier 1 (working memory) and Tier 2 (episodic buffer) are in-process, not graph-backed
- No changes to goal/plan/reflection systems — keep existing ManorGoal*, ManorPlan*
- All new types in `io.casehub.examples.manor.agent` package
- Per-concept access via Thing trait projections — no separate store classes
- PerspectivalResolver is now package-private in neocortex — use `CognitiveProfile.resolve(query.withAsSeenBy(principalId))` for perspective, `CognitiveProfile.compare(query, Set<PrincipalId>)` for multi-agent comparison
- Affect memories use `MemoryInput.ownedBy(subject, domain, tenant, content, principalId).withPad(p, a, d)` for principal-scoped storage
- TDD: failing test first, then minimal implementation
- Use `ide_find_references`, `ide_find_class` for navigation; `ide_insert_member`, `ide_replace_member` for edits

## Upstream Dependencies (not in this plan — separate issues)

- neocortex#322: cognitive node type registration + trait interfaces
- neocortex#323: `consolidateNow(tenantId)` public trigger
- blocks#260: CognitiveObservationSections — beliefs, principles, trust, norms renderers

These are NOT hard blockers. Batch 1-2 have zero upstream dependencies. Batch 3-4 can use fallback rendering (`ObservationSection.items()`) if blocks#260 hasn't landed.

---

## Batch 1: Clean foundation — refactoring (no behavior changes)

After this batch: ScenarioOrchestrator is cleaner, ObservationBuilder has a builder API, AgentExperienceService constructors are sane. All 395 existing tests still pass. No functional changes.

### Task 1: Config records — extract 30+ properties from ScenarioOrchestrator

Group the 30 `@ConfigProperty` fields on ScenarioOrchestrator into typed records. Each service takes its own config instead of receiving individual values.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorConfig.java`
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:49-113` — replace 30 properties with config records
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:146-195` — pass config records to service constructors
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorConfigTest.java`

**Interfaces:**
- Produces: `ManorConfig` containing nested records: `ReflectionConfig`, `GoalConfig`, `PlanConfig`, `ObservationConfig`, `NarratorConfig`, `TrustConfig`, `DispositionConfig`, `MemoryConfig`

- [ ] **Step 1: Create ManorConfig with nested records**

```java
package io.casehub.examples.manor.agent;

public record ManorConfig(
    int maxTurns,
    ObservationConfig observation,
    NarratorConfig narrator,
    ReflectionConfig reflection,
    GoalConfig goal,
    PlanConfig plan,
    TrustConfig trust,
    DispositionConfig disposition,
    MemoryConfig memory,
    String activeCharacters,
    int maxConcurrentAgents
) {
    public record ObservationConfig(int verbatimThreshold, int groupedThreshold) {}
    public record NarratorConfig(boolean enabled, int eventThreshold, int timerSeconds) {}
    public record ReflectionConfig(boolean enabled, int maxUnreflected, double importanceThreshold, int maxSourceMemories) {}
    public record GoalConfig(boolean enabled, int cooldownTicks, int maxNewPerReflection) {}
    public record PlanConfig(boolean enabled, int maxRevisionGeneration) {}
    public record TrustConfig(boolean enabled, double positiveWeight, double negativeWeight) {}
    public record DispositionConfig(boolean enabled, int evolutionCheckInterval) {}
    public record MemoryConfig(int recallLimit, boolean personalityWeightedRetrieval) {}
}
```

- [ ] **Step 2: Write test verifying config record construction**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ManorConfigTest {
    @Test
    void configRecordsConstruct() {
        var config = new ManorConfig(300,
            new ManorConfig.ObservationConfig(10, 15),
            new ManorConfig.NarratorConfig(true, 5, 15),
            new ManorConfig.ReflectionConfig(true, 5, 3.0, 15),
            new ManorConfig.GoalConfig(true, 10, 2),
            new ManorConfig.PlanConfig(true, 5),
            new ManorConfig.TrustConfig(true, 1.0, -2.0),
            new ManorConfig.DispositionConfig(true, 5),
            new ManorConfig.MemoryConfig(20, true),
            "", 5);
        assertThat(config.maxTurns()).isEqualTo(300);
        assertThat(config.reflection().maxUnreflected()).isEqualTo(5);
        assertThat(config.goal().cooldownTicks()).isEqualTo(10);
    }
}
```

- [ ] **Step 3: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorConfigTest`
Expected: PASS

- [ ] **Step 4: Replace @ConfigProperty fields in ScenarioOrchestrator**

Use `ide_replace_member` on ScenarioOrchestrator. Replace the 30 `@ConfigProperty` fields (lines 49–113) with a single `@Inject ManorConfig config` field. Create a CDI producer that constructs `ManorConfig` from `@ConfigProperty` values.

Create `src/main/java/io/casehub/examples/manor/agent/ManorConfigProducer.java`:

```java
package io.casehub.examples.manor.agent;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;

@ApplicationScoped
public class ManorConfigProducer {
    @org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.scenario.max-turns", defaultValue = "300")
    int maxTurns;
    // ... all 30 properties moved here ...

    @Produces @ApplicationScoped
    public ManorConfig produce() {
        return new ManorConfig(maxTurns,
            new ManorConfig.ObservationConfig(verbatimThreshold, groupedThreshold),
            // ... construct all sub-records ...
        );
    }
}
```

Update all references in `runScenario()` and `runAutonomousTicks()` from `this.verbatimThreshold` to `config.observation().verbatimThreshold()` etc.

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all 395 tests PASS (1 pre-existing error in PersonalityCompositionVerificationTest — unrelated)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorConfig.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorConfigProducer.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorConfigTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "refactor(#52): extract ManorConfig records from ScenarioOrchestrator"
```

---

### Task 2: ObservationBuilder → builder pattern

Convert the static 9-parameter method into an instance builder. Exchange path (which doesn't need full cognition) calls `.build()` without `.withCognition()`.

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java` — convert to instance with builder API
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:294` — update caller
- Modify: `src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java:71` — update caller
- Modify: `src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java:65` — update caller
- Test: `src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Produces: `new ObservationBuilder(worldProvider, pipeline, observerTags).withCharacter(c).withGoals(goals).withDrain(drain).withMemories(memories, reflections, relationships).withCognitiveSections(sections).build()` returning `String`

- [ ] **Step 1: Write test for builder API**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ObservationBuilderTest {
    @Test
    void builderProducesNonEmptyObservation() {
        var builder = new ObservationBuilder(null, null, java.util.Set.of());
        String result = builder
            .withCharacter(TestFixtures.createCharacter("test-agent", "Test Room"))
            .withGoals(List.of())
            .build();
        assertThat(result).isNotBlank();
        assertThat(result).contains("Your Inventory");
    }

    @Test
    void cognitiveSectionsAppearInOutput() {
        var section = ObservationSection.text("Your Beliefs", "You believe the poison is in the Kitchen");
        var builder = new ObservationBuilder(null, null, java.util.Set.of());
        String result = builder
            .withCharacter(TestFixtures.createCharacter("test-agent", "Test Room"))
            .withGoals(List.of())
            .withCognitiveSections(List.of(section))
            .build();
        assertThat(result).contains("Your Beliefs");
        assertThat(result).contains("poison is in the Kitchen");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ObservationBuilderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — builder API doesn't exist yet

- [ ] **Step 3: Convert ObservationBuilder to instance with builder methods**

Replace the static `buildObservation()` with instance methods. Each `with*` method stores parameters. `build()` assembles sections and renders. Default all optional parameters to empty/null so minimal callers work.

Use `ide_replace_member` to rewrite the class body. Key change: the static method's section assembly logic moves into `build()`. Add `withCognitiveSections(List<ObservationSection> sections)` — these are inserted after inventory, before goals.

- [ ] **Step 4: Update 3 call sites**

Use `ide_find_references` on `buildObservation` to find all callers. Update each:

- `ScenarioOrchestrator.java:294` → `new ObservationBuilder(worldProvider, pipeline, observerTags).withCharacter(c).withGoals(goals).withDrain(drain).withMemories(memories, reflections, relationships).build()`
- `CharacterAgentLoop.java:71` → same pattern, simpler (no memories)
- `ExchangeRunner.java:65` → same pattern, minimal (no cognition)

- [ ] **Step 5: Create TestFixtures helper if not present**

```java
package io.casehub.examples.manor.agent;

public final class TestFixtures {
    public static io.casehub.examples.manor.model.CharacterState createCharacter(String agentId, String room) {
        return new io.casehub.examples.manor.model.CharacterState(agentId, agentId, room, 0, 0, 1000);
    }
}
```

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS — refactoring only, no behavior change

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/TestFixtures.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "refactor(#52): ObservationBuilder — static method to builder pattern"
```

---

### Task 3: AgentExperienceService constructor cleanup

Replace 4 telescoping constructors (3→11→12→13 params) with a config record.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/ExperienceConfig.java`
- Modify: `src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java:41-91` — single constructor taking config
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — update construction site
- Test: existing tests via full suite

**Interfaces:**
- Produces: `new AgentExperienceService(ExperienceConfig config)` replacing 4 constructors

- [ ] **Step 1: Create ExperienceConfig**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.experience.ExperienceRecorder;

public record ExperienceConfig(
    ExperienceRecorder recorder,
    CaseMemoryStore store,
    String tenantId,
    io.casehub.neocortex.memory.reflection.ReflectionSynthesizer synthesizer,
    ManorReflectionTrigger reflectionTrigger,
    boolean reflectionEnabled,
    boolean decayEnabled,
    int decayMaxAgeDays,
    double decayMinImportance,
    int maxSourceMemories,
    int recallLimit,
    ManorGoalEvaluator goalEvaluator,
    ManorPlanEvaluator planEvaluator
) {
    public static ExperienceConfig minimal(ExperienceRecorder recorder, CaseMemoryStore store, String tenantId) {
        return new ExperienceConfig(recorder, store, tenantId, null, null, false, false, 7, 0.2, 15, 20, null, null);
    }
}
```

- [ ] **Step 2: Replace constructors with single constructor**

Use `ide_replace_member` on `AgentExperienceService` to replace all 4 constructors with:

```java
public AgentExperienceService(ExperienceConfig config) {
    this.recorder           = config.recorder();
    this.store              = config.store();
    this.tenantId           = config.tenantId();
    this.synthesizer        = config.synthesizer();
    this.reflectionTrigger  = config.reflectionTrigger();
    this.reflectionEnabled  = config.reflectionEnabled();
    this.decayEnabled       = config.decayEnabled();
    this.decayMaxAgeDays    = config.decayMaxAgeDays();
    this.decayMinImportance = config.decayMinImportance();
    this.maxSourceMemories  = config.maxSourceMemories();
    this.recallLimit        = config.recallLimit();
    this.goalEvaluator      = config.goalEvaluator();
    this.planEvaluator      = config.planEvaluator();
}
```

- [ ] **Step 3: Update call sites**

Use `ide_find_references` on `AgentExperienceService` constructor. Update ScenarioOrchestrator and any test constructors to use `ExperienceConfig`.

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExperienceConfig.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "refactor(#52): AgentExperienceService — telescoping constructors to ExperienceConfig record"
```

---

## Batch 2: CharacterCognition extraction

After this batch: per-character CharacterCognition object exists. ScenarioOrchestrator delegates cognitive operations to it. Old ManorTrustProvider, ManorDispositionRecorder, ManorPersonalityEvolution removed. All tests pass.

### Task 4: Extract CharacterCognition from ScenarioOrchestrator

The core extraction. Creates a per-character cognitive object that owns Tier 2 state (trust events, experience tracking) and will later wire to Tier 3 (mindmap). For now, it encapsulates the existing behavior.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java`
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:241-434` — delegate to CharacterCognition
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorTrustProvider.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorDispositionRecorder.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorPersonalityEvolution.java` (use `ide_refactor_safe_delete`)
- Test: `src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: `ManorConfig`, `ExperienceConfig` from Tasks 1+3
- Produces: `CharacterCognition` with:
  - `recordTrustEvent(String targetId, ActionType action)` — buffers trust events (Tier 2)
  - `recordExperience(String room, String description, String thinking, double importance, String targetAgentId, int tick)` — delegates to AgentExperienceService
  - `computeImportance(ActionType action)` — replaces static `importanceForAction()`
  - `recallMemories(int limit)`, `recallReflections(int limit)`, `recallRelationships(String otherId, int limit)` — memory queries
  - `renderCognitiveSections(CharacterState character, Collection<String> nearbyAgentIds, Map<String, String> agentNames)` — returns `List<ObservationSection>` (empty for now — wiring adds content in Batch 3)

- [ ] **Step 1: Write CharacterCognition test**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.model.ActionType;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class CharacterCognitionTest {

    @Test
    void computeImportanceMatchesExistingBehavior() {
        var cognition = createMinimalCognition("test-agent");
        assertThat(cognition.computeImportance(ActionType.STEAL)).isEqualTo(0.9);
        assertThat(cognition.computeImportance(ActionType.WAIT)).isEqualTo(0.1);
        assertThat(cognition.computeImportance(ActionType.MOVE)).isEqualTo(0.3);
        assertThat(cognition.computeImportance(null)).isEqualTo(0.5);
    }

    @Test
    void cognitiveSectionsEmptyBeforeWiring() {
        var cognition = createMinimalCognition("test-agent");
        var sections = cognition.renderCognitiveSections(
            TestFixtures.createCharacter("test-agent", "Room"),
            List.of(), Map.of());
        assertThat(sections).isEmpty();
    }

    private CharacterCognition createMinimalCognition(String agentId) {
        return new CharacterCognition(agentId, null);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=CharacterCognitionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure

- [ ] **Step 3: Implement CharacterCognition**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import io.casehub.examples.manor.model.ActionType;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.neocortex.memory.Memory;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Map;

public final class CharacterCognition {

    private final String agentId;
    private final AgentExperienceService experienceService;

    public CharacterCognition(String agentId, AgentExperienceService experienceService) {
        this.agentId = agentId;
        this.experienceService = experienceService;
    }

    public String agentId() { return agentId; }

    public double computeImportance(ActionType action) {
        if (action == null) return 0.5;
        return switch (action) {
            case STEAL -> 0.9;
            case USE -> 0.8;
            case TAKE, GIVE, PULL_ASIDE -> 0.7;
            case INTERACT -> 0.6;
            case MOVE -> 0.3;
            case LOOK -> 0.2;
            case WAIT -> 0.1;
        };
    }

    public void recordExperience(String room, String description, String thinking,
                                  double importance, String targetAgentId, int tick) {
        if (experienceService != null) {
            experienceService.ingest(agentId, room, description, thinking, importance, targetAgentId, tick);
        }
    }

    public List<Memory> recallMemories(int limit) {
        return experienceService != null ? experienceService.recall(agentId, limit) : List.of();
    }

    public List<Memory> recallReflections(int limit) {
        return experienceService != null ? experienceService.recallReflections(agentId, limit) : List.of();
    }

    public List<Memory> recallRelationships(String otherId, int limit) {
        return experienceService != null ? experienceService.recallRelationships(agentId, otherId, limit) : List.of();
    }

    public List<ObservationSection> renderCognitiveSections(
            CharacterState character,
            Collection<String> nearbyAgentIds,
            Map<String, String> agentNames) {
        return List.of();
    }

    public void recordTrustEvent(String targetId, ActionType action) {
        // Tier 2 buffer — will be wired to mindmap edges in Batch 3
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=CharacterCognitionTest`
Expected: all tests PASS

- [ ] **Step 5: Wire into ScenarioOrchestrator**

In `runScenario()`: create `Map<String, CharacterCognition>` for active characters. In `runAutonomousTicks()`:

1. Replace `experienceService.recall()` calls with `cognition.recallMemories()`
2. Replace `experienceService.recallReflections()` with `cognition.recallReflections()`
3. Replace `experienceService.recallRelationships()` with `cognition.recallRelationships()`
4. Replace `experienceService.ingest()` with `cognition.recordExperience()`
5. Replace `importanceForAction(response)` with `cognition.computeImportance(response.action().type())`
6. Replace `trustProvider.recordPositive/Negative()` with `cognition.recordTrustEvent(target, action)`
7. Replace `dispositionRecorder.record()` — remove, covered by `computeImportance()`
8. Replace `personalityEvolution.checkAndEvolve()` — remove for now
9. Pass `cognition.renderCognitiveSections()` to ObservationBuilder `.withCognitiveSections()`

Remove local variable creation for `ManorTrustProvider`, `ManorDispositionRecorder`, `ManorPersonalityEvolution`.

- [ ] **Step 6: Delete old types via ide_refactor_safe_delete**

Delete these files using `ide_refactor_safe_delete`:
- `ManorTrustProvider.java`
- `ManorDispositionRecorder.java`
- `ManorPersonalityEvolution.java`

If safe delete reports remaining references, fix them first.

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): extract CharacterCognition — per-character cognitive object, remove old trust/disposition types"
```

---

## Batch 3: Mindmap wiring — connect neocortex cognitive stack

After this batch: characters have mindmap-backed cognition. CognitiveDerivationEngine derives defaults from Eidos. CognitiveProfile queries mindmap for beliefs/trust/principles. ConversationBridge processes dialogue into knowledge. Social config in character descriptors.

### Task 5: Register game-world types + wire CognitiveDerivationEngine + character descriptors

Set up the cognitive infrastructure: register mindmap types, derive cognitive defaults from Eidos personality, add social config to character YAML.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSetup.java`
- Modify: `src/main/resources/META-INF/eidos/descriptors-composite.yaml` — add `extensionData.social` to 5 characters
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSetupTest.java`

**Interfaces:**
- Consumes: neocortex `TypeRegistry`, `CognitiveDerivationEngine`, Eidos `AgentRegistry`
- Produces: `ManorCognitiveSetup.init(String tenantId)` — registers ITEM, LOCATION, CHARACTER types. `ManorCognitiveSetup.deriveDefaults(AgentDescriptor descriptor)` — returns `CognitiveDefaults`

> **Pre-step: Check CognitiveDefaultsRegistry** — neocortex now has `CognitiveDefaultsRegistry` for YAML-driven per-agent cognitive config. Before implementing custom `SocialConfig` / `SocialCognitionLoader` / `ManorCognitiveSetup.parseSocialConfig()`, check whether CognitiveDefaultsRegistry already supports drives/norms/beliefs configuration. If it does, use it instead — our custom parsing becomes unnecessary. If it only covers trust formation rate / conflict interpretation (from `CognitiveDerivationEngine.deriveSocialCognition()`), then keep custom parsing for drives/norms/beliefs and wire CognitiveDefaultsRegistry for the personality-derived defaults.

- [ ] **Step 1: Write test for type registration**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ManorCognitiveSetupTest {

    @Test
    void gameWorldTypesAreRegistered() {
        var types = ManorCognitiveSetup.gameWorldTypes();
        assertThat(types).containsExactlyInAnyOrder("item", "location", "character");
    }

    @Test
    void socialConfigParsesFromExtensionData() {
        var ext = java.util.Map.<String, Object>of("social", java.util.Map.of(
            "drives", java.util.List.of(
                java.util.Map.of("type", "scheming", "intensity", 0.9, "description", "Compelled to scheme")
            ),
            "norms", java.util.List.of(
                java.util.Map.of("rule", "Never help Penelope", "priority", 10)
            ),
            "initial-beliefs", java.util.List.of(
                java.util.Map.of("key", "penelope-awareness", "value", "Penelope is naive")
            )
        ));
        var config = ManorCognitiveSetup.parseSocialConfig(ext);
        assertThat(config.drives()).hasSize(1);
        assertThat(config.norms()).hasSize(1);
        assertThat(config.initialBeliefs()).hasSize(1);
    }
}
```

- [ ] **Step 2: Implement ManorCognitiveSetup**

```java
package io.casehub.examples.manor.agent;

import java.util.List;
import java.util.Map;

public final class ManorCognitiveSetup {

    public static List<String> gameWorldTypes() {
        return List.of("item", "location", "character");
    }

    @SuppressWarnings("unchecked")
    public static SocialConfig parseSocialConfig(Map<String, Object> extensionData) {
        return SocialCognitionLoader.parseExtensionData(extensionData);
    }

    // Called at scenario start with TypeRegistry from neocortex mindmap-intelligence
    // typeRegistry.registerType("item", "general", tenantId);
    // typeRegistry.registerType("location", "general", tenantId);
    // typeRegistry.registerType("character", "person", tenantId);
}
```

- [ ] **Step 3: Add social config to 5 character descriptors**

Edit `src/main/resources/META-INF/eidos/descriptors-composite.yaml`. Add `extensionData:` with `social:` section to each character. Example for hooded-claw:

```yaml
  - agentId: hooded-claw
    # ... existing fields ...
    extensionData:
      social:
        drives:
          - type: scheming
            intensity: 0.9
            description: "Compelled to hatch elaborate plans against Penelope"
          - type: self-preservation
            intensity: 0.7
            description: "Avoids direct confrontation, prefers subterfuge"
          - type: dominance
            intensity: 0.6
            description: "Must be the most powerful person in every room"
        norms:
          - rule: "Never help Penelope directly"
            priority: 10
          - rule: "Maintain a veneer of charm in public"
            priority: 5
          - rule: "Protect personal schemes from discovery"
            priority: 8
        initial-beliefs:
          - key: "penelope-awareness"
            value: "Penelope is naive and trusts too easily"
          - key: "peter-threat"
            value: "Peter Perfect is protective but predictable"
```

Repeat for penelope-pitstop, peter-perfect, muttley, dick-dastardly with character-appropriate drives/norms/beliefs per the spec.

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): ManorCognitiveSetup + social config in 5 character descriptors"
```

---

### Task 6: Wire CognitiveProfile + ConversationBridge + observation rendering

The main wiring task. CharacterCognition queries mindmap via CognitiveProfile + Thing trait projections. Dialogue flows through ConversationBridge. Cognitive sections rendered into observations.

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java` — wire CognitiveProfile, ConversationBridge, render cognitive sections
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorTrustEvents.java`
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorNormFilter.java`
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — inject neocortex services, pass to CharacterCognition
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorTrustEventsTest.java`
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorNormFilterTest.java`

**Interfaces:**
- Consumes: neocortex `CognitiveProfile` (with `resolve(query.withAsSeenBy(principalId))` and `compare(query, Set<PrincipalId>)`), `ConversationBridge`, `CognitiveDerivationEngine`, blocks `CognitiveObservationSections`
- Note: `PerspectivalResolver` is now package-private — perspective is applied via `CognitiveProfile.resolve()` with `withAsSeenBy()`, not a separate service
- Produces: `CharacterCognition.renderCognitiveSections()` now returns populated sections (drives, beliefs, norms, trust, principles). `CharacterCognition.processDialogue(String text)` feeds ConversationBridge.

- [ ] **Step 1: Write ManorTrustEvents — personality-modulated**

`CognitiveDerivationEngine.derive(descriptorView).socialCognition()` returns `SocialCognitionDefaults(trustFormationRate, conflictInterpretation)` per character. ManorTrustEvents should modulate base weights by `trustFormationRate` so that a character with high trust formation (0.7) gains/loses trust faster than one with low trust formation (0.3). `conflictInterpretation` (REPAIR/INFORMATION/NEUTRAL/DISENGAGE) can modulate negative weights — a REPAIR interpreter takes less damage from theft than a DISENGAGE interpreter.

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.model.ActionType;
import java.util.Map;

public final class ManorTrustEvents {
    private static final Map<ActionType, Double> BASE_WEIGHTS = Map.of(
        ActionType.STEAL, -0.4,
        ActionType.GIVE, 0.15,
        ActionType.INTERACT, 0.05,
        ActionType.PULL_ASIDE, 0.0
    );

    public static double weightFor(ActionType action, double trustFormationRate) {
        return BASE_WEIGHTS.getOrDefault(action, 0.0) * trustFormationRate;
    }

    public static double weightFor(ActionType action) {
        return BASE_WEIGHTS.getOrDefault(action, 0.0);
    }

    public static boolean isRelevant(ActionType action) {
        return BASE_WEIGHTS.containsKey(action) && BASE_WEIGHTS.get(action) != 0.0;
    }
}
```

- [ ] **Step 2: Write ManorNormFilter**

```java
package io.casehub.examples.manor.agent;

import java.util.Collection;
import java.util.List;
import java.util.stream.Collectors;

public final class ManorNormFilter {
    public static List<SocialConfig.NormEntry> filter(
            List<SocialConfig.NormEntry> allNorms,
            Collection<String> nearbyCharacterNames,
            Collection<String> inventory) {
        return allNorms.stream()
            .sorted((a, b) -> Integer.compare(b.priority(), a.priority()))
            .toList();
    }
}
```

Initial implementation returns all norms sorted by priority. Context-based filtering (matching norm text against nearby characters/inventory) is an enhancement.

- [ ] **Step 3: Write tests for ManorTrustEvents and ManorNormFilter**

```java
// ManorTrustEventsTest.java
package io.casehub.examples.manor.agent;
import io.casehub.examples.manor.model.ActionType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ManorTrustEventsTest {
    @Test void stealIsNegative() { assertThat(ManorTrustEvents.weightFor(ActionType.STEAL)).isLessThan(0); }
    @Test void giveIsPositive() { assertThat(ManorTrustEvents.weightFor(ActionType.GIVE)).isGreaterThan(0); }
    @Test void waitIsIrrelevant() { assertThat(ManorTrustEvents.isRelevant(ActionType.WAIT)).isFalse(); }
}

// ManorNormFilterTest.java
package io.casehub.examples.manor.agent;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ManorNormFilterTest {
    @Test
    void sortsByPriorityDescending() {
        var norms = List.of(
            new SocialConfig.NormEntry("low", 1),
            new SocialConfig.NormEntry("high", 10),
            new SocialConfig.NormEntry("mid", 5));
        var filtered = ManorNormFilter.filter(norms, List.of(), List.of());
        assertThat(filtered.get(0).rule()).isEqualTo("high");
        assertThat(filtered.get(2).rule()).isEqualTo("low");
    }
}
```

- [ ] **Step 4: Wire CharacterCognition to neocortex services**

Update `CharacterCognition` constructor to accept neocortex services. Update `renderCognitiveSections()` to:

1. Load social config from Eidos extensionData (cached per character)
2. Load principles from Eidos constraints
3. Query mindmap via `CognitiveProfile.resolve(query.withAsSeenBy(PrincipalId.agent(agentId)))` for belief/trust/judgment nodes (Tier 3) — uses `node.as(traitInterface)` if neocortex#322 has landed, otherwise reads node properties directly. Perspective is applied internally by CognitiveProfile (PerspectivalResolver is package-private).
4. For social context rendering ("how does this character perceive nearby characters"), use `CognitiveProfile.compare(query, nearbyAgentPrincipalIds)` for batched multi-agent comparison
5. Filter norms via ManorNormFilter
6. Render all sections via `CognitiveObservationSections` methods (if blocks#260 has landed) or `ObservationSection.items()` fallback

Update `recordTrustEvent()` to buffer events in Tier 2 with `ManorTrustEvents.weightFor(action, socialCognitionDefaults.trustFormationRate())` — modulated per character personality via `CognitiveDerivationEngine.derive(descriptorView).socialCognition()`.

Store affect memories using `MemoryInput.ownedBy(subject, domain, tenant, content, principalId).withPad(p, a, d)` for principal-scoped storage (follows neocortex walkthrough pattern).

Add `processDialogue(String text, String tenantId)` that calls `ConversationBridge.process()` for knowledge extraction.

- [ ] **Step 5: Update ScenarioOrchestrator to inject neocortex services**

Add `@Inject` fields for neocortex services: `CognitiveProfile`, `ConversationBridge`, `CognitiveDerivationEngine`, `TypeRegistry`. Pass to `CharacterCognition` construction. Cache `SocialCognitionDefaults` per character (derived once at scenario start via `CognitiveDerivationEngine.derive(descriptorView).socialCognition()`). In the tick loop, after dialogue processing, call `cognition.processDialogue()` with the dialogue text.

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): wire CognitiveProfile + ConversationBridge + cognitive observation rendering"
```

---

## Batch 4: Consolidation + integration

After this batch: full social cognition running. Sleep game mechanic triggers consolidation. Integration tests verify end-to-end.

### Task 7: Wire consolidation (sleep game mechanic) + integration tests

Add the "night falls" consolidation trigger and verify the full cognitive stack end-to-end.

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — add sleep cycle between acts
- Modify: `src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java` — add `consolidate()` method
- Test: `src/test/java/io/casehub/examples/manor/agent/SocialCognitionIntegrationTest.java`

**Interfaces:**
- Consumes: neocortex `ConsolidationScheduler.consolidateNow(tenantId)` (neocortex#323)
- Produces: Sleep cycle in the tick loop. Full cognitive stack integration verified.

- [ ] **Step 1: Add consolidate() to CharacterCognition**

```java
public void consolidate() {
    // If neocortex#323 has landed:
    // consolidationScheduler.consolidateNow(tenantId + ":" + agentId);

    // Fallback: directly run accessible consolidation phases
    // This is a placeholder until consolidateNow() is available
    log.infof("Consolidation triggered for %s", agentId);
}
```

- [ ] **Step 2: Add sleep cycle to ScenarioOrchestrator**

In `runAutonomousTicks()`, add a consolidation check at configurable intervals (e.g., every 50 ticks):

```java
if (tick > 0 && tick % config.consolidationInterval() == 0) {
    log.info("Night falls on the mansion. Characters rest and reflect...");
    webEventBus.broadcast(ManorWebSocketEvent.narrator("Night falls. The characters rest and reflect on the day's events..."));
    for (var cognition : cognitions.values()) {
        cognition.consolidate();
    }
    webEventBus.broadcast(ManorWebSocketEvent.narrator("Dawn breaks. A new day begins..."));
}
```

Add `consolidationInterval` to ManorConfig (default: 50 ticks).

- [ ] **Step 3: Write integration test**

```java
package io.casehub.examples.manor.agent;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class SocialCognitionIntegrationTest {

    @Inject ScenarioOrchestrator orchestrator;
    @Inject io.casehub.eidos.api.AgentRegistry agentRegistry;

    @Test
    void socialConfigLoadsFromDescriptors() {
        var desc = agentRegistry.findById("hooded-claw", "wacky-manor").orElseThrow();
        var social = ManorCognitiveSetup.parseSocialConfig(desc.extensionData());
        assertThat(social.drives()).isNotEmpty();
        assertThat(social.drives().stream().anyMatch(d -> d.type().equals("scheming"))).isTrue();
        assertThat(social.norms()).isNotEmpty();
        assertThat(social.initialBeliefs()).isNotEmpty();
    }

    @Test
    void characterCognitionRendersSections() {
        var desc = agentRegistry.findById("hooded-claw", "wacky-manor").orElseThrow();
        var cognition = new CharacterCognition("hooded-claw", null);
        // After wiring, renderCognitiveSections returns non-empty for characters with social config
        var sections = cognition.renderCognitiveSections(
            TestFixtures.createCharacter("hooded-claw", "Grand Hallway"),
            java.util.List.of("penelope-pitstop"),
            java.util.Map.of("penelope-pitstop", "Penelope Pitstop"));
        // Sections will be populated once CognitiveProfile is wired
    }
}
```

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): wire consolidation sleep mechanic + integration tests"
```

---

## Deferred

- **LLM eval tests:** Add once cognitive stack is verified with manual runs. Tag `@Tag("llm-eval")`.
- **Context-based norm filtering:** ManorNormFilter currently returns all norms sorted. Context matching (checking norm text against nearby characters/items) is an enhancement.
- **SocialComparison integration:** Use `SocialComparison.compare()` with results from `CognitiveProfile.compare()` to render inter-character perception divergence (PAD distance, trajectory alignment) in observation sections. Natural extension after basic CognitiveProfile queries are validated.
- **DomainActivation:** Cross-domain DTW correlation — not relevant to current room-based model but could track cross-room emotional patterns if rooms become affect-tracked subgraphs.
- **Blocks scoring integration:** SurpriseScorer/ArousalScorer for consolidation importance scoring — wire after basic consolidation works.
- **ExperienceConsolidationPhase:** The Tier 2 → Tier 3 graduation phase is a neocortex contribution, not a wacky-manor task. Filed as a gap in the spec; implementation belongs in a neocortex issue.

## References

- [2026-09-11-social-cognition-design.md] — design spec (revised)
- [decisions.md] — D1-D7 captured decisions
- [D7-debate/mediator-synthesis-1.md] — unified vs separate stores debate
- [ScenarioOrchestrator.java:241-434] — autonomous tick loop
- [ObservationBuilder.java:1-88] — observation rendering
- [AgentExperienceService.java:41-91] — telescoping constructors
- [descriptors-composite.yaml] — Eidos character descriptors
- [CognitiveProfile] (neocortex cognitive-index) — entity knowledge queries, perspectival resolve via `withAsSeenBy()`, batched `compare()`
- [CognitiveDerivationEngine] (neocortex cognitive-index) — `deriveSocialCognition()` for personality-modulated trust formation
- [SocialComparison] (neocortex cognitive-index) — PAD distance, pairwise differences, trajectory alignment (deferred)
- [CognitiveIndexWalkthroughTest] (neocortex examples) — canonical consumer pattern: overlays, affect memories, perspectival resolve
- [ConversationBridge] (neocortex mindmap-intelligence) — text → mindmap
- [CognitiveObservationSections] (blocks) — observation section rendering
- neocortex#322 — cognitive node types (still open)
- neocortex#323 — consolidateNow trigger (still open)
- blocks#260 — cognitive observation renderers
- GitHub #52 — focal issue
