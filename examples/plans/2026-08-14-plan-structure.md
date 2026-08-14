# Plan Structure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #44 — Plan structure — replace currentPlan string with structured plans
**Issue group:** #41

**Goal:** Replace the manor's `String currentPlan` with per-goal structured plans that decompose goals into named steps with status tracking, forming from goal events and revising on action failure or reflection.

**Architecture:** Three new components (AgentPlan/PlanStep data model, ManorPlanFormationStrategy, ManorPlanRevisionStrategy) plus a ManorPlanEvaluator orchestrator. Plans form when goals form (LLM-driven), revise on action failure (reactive) and during reflection (deliberative). The thinking field becomes a reactive scratchpad; plans are shown separately in observations. ManorPlanRevisionStrategy implements the engine's PlanRevisionStrategy SPI directly, passing nulls for case-specific fields.

**Tech Stack:** Java 21, Quarkus, Jackson, JUnit 5, AssertJ, Eidos API, Engine API (PlanRevisionStrategy SPI), Neocortex Memory API

## Global Constraints

- Pre-release: breaking changes cost nothing
- IntelliJ MCP for all .java edits — never Edit/Write on existing source files
- TDD: failing test before implementation, always
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml` for test runs
- Commit after each task with `Refs #44`

---

### Task 1: AgentPlan data model and CharacterState migration

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/model/AgentPlan.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/model/PlanStep.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/model/PlanStepStatus.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/CharacterState.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/model/CharacterStateTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Produces: `AgentPlan(String goalName, List<PlanStep> steps, String rationale, int creationTick, int lastRevisionTick, int revisionGeneration)`, `PlanStep(String id, String description, PlanStepStatus status)`, `PlanStepStatus` enum (`PENDING`, `IN_PROGRESS`, `COMPLETED`, `FAILED`), `CharacterState.plans()`, `CharacterState.setPlan(String, AgentPlan)`, `CharacterState.removePlan(String)`, `CharacterState.currentThinking()`, `CharacterState.setCurrentThinking(String)`

- [ ] **Step 1: Create PlanStepStatus enum**

Create `wacky-manor/src/main/java/io/casehub/examples/manor/model/PlanStepStatus.java`:

```java
package io.casehub.examples.manor.model;

public enum PlanStepStatus {
    PENDING, IN_PROGRESS, COMPLETED, FAILED
}
```

- [ ] **Step 2: Create PlanStep record**

Create `wacky-manor/src/main/java/io/casehub/examples/manor/model/PlanStep.java`:

```java
package io.casehub.examples.manor.model;

public record PlanStep(String id, String description, PlanStepStatus status) {

    public PlanStep withStatus(PlanStepStatus newStatus) {
        return new PlanStep(id, description, newStatus);
    }
}
```

- [ ] **Step 3: Create AgentPlan record**

Create `wacky-manor/src/main/java/io/casehub/examples/manor/model/AgentPlan.java`:

```java
package io.casehub.examples.manor.model;

import java.util.List;

public record AgentPlan(
        String goalName,
        List<PlanStep> steps,
        String rationale,
        int creationTick,
        int lastRevisionTick,
        int revisionGeneration) {

    public AgentPlan withSteps(List<PlanStep> newSteps) {
        return new AgentPlan(goalName, newSteps, rationale,
                creationTick, lastRevisionTick, revisionGeneration);
    }

    public AgentPlan withRevision(List<PlanStep> newSteps, String newRationale, int tick) {
        return new AgentPlan(goalName, newSteps, newRationale,
                creationTick, tick, revisionGeneration + 1);
    }
}
```

- [ ] **Step 4: Write failing tests for CharacterState plan storage**

Add to `CharacterStateTest.java` using `ide_insert_member`:

```java
@Test
void plans_empty_by_default() {
    assertThat(state.plans()).isEmpty();
}

@Test
void setPlan_and_retrieve() {
    var step = new PlanStep("s1", "Find the poison", PlanStepStatus.PENDING);
    var plan = new AgentPlan("protect-penelope", List.of(step), "need to protect", 1, 1, 0);
    state.setPlan("protect-penelope", plan);
    assertThat(state.plans()).containsKey("protect-penelope");
    assertThat(state.plans().get("protect-penelope").steps()).hasSize(1);
}

@Test
void removePlan_removes_by_goalName() {
    var plan = new AgentPlan("goal-a", List.of(), "r", 1, 1, 0);
    state.setPlan("goal-a", plan);
    state.removePlan("goal-a");
    assertThat(state.plans()).isEmpty();
}

@Test
void currentThinking_null_by_default() {
    assertThat(state.currentThinking()).isNull();
}

@Test
void currentThinking_set_and_retrieved() {
    state.setCurrentThinking("I see the poison on the shelf");
    assertThat(state.currentThinking()).isEqualTo("I see the poison on the shelf");
}
```

- [ ] **Step 5: Run tests to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=CharacterStateTest`
Expected: compilation failures — plans()/setPlan()/removePlan()/currentThinking() don't exist

- [ ] **Step 6: Migrate CharacterState — remove currentPlan, add plans + currentThinking**

Use `ide_edit_member` on `CharacterState` to:
- Remove field: `currentPlan`
- Remove method: `currentPlan()`
- Remove method: `setCurrentPlan(String)`
- Add field: `private final java.util.concurrent.ConcurrentHashMap<String, io.casehub.examples.manor.model.AgentPlan> plans = new java.util.concurrent.ConcurrentHashMap<>();`
- Add field: `private volatile String currentThinking;`
- Add method: `public java.util.Map<String, io.casehub.examples.manor.model.AgentPlan> plans() { return java.util.Collections.unmodifiableMap(plans); }`
- Add method: `public void setPlan(String goalName, io.casehub.examples.manor.model.AgentPlan plan) { plans.put(goalName, plan); }`
- Add method: `public void removePlan(String goalName) { plans.remove(goalName); }`
- Add method: `public String currentThinking() { return currentThinking; }`
- Add method: `public void setCurrentThinking(String thinking) { this.currentThinking = thinking; }`

- [ ] **Step 7: Update ScenarioOrchestrator — setCurrentPlan → setCurrentThinking**

Use `ide_search_text` to find `setCurrentPlan` in `ScenarioOrchestrator.java`.
Use `ide_replace_member` or structural search/replace to change:
```java
c.setCurrentPlan(response.thinking());
```
to:
```java
c.setCurrentThinking(response.thinking());
```

- [ ] **Step 8: Update ExchangeRunner — setCurrentPlan → setCurrentThinking**

Use `ide_search_text` to find `setCurrentPlan` in `ExchangeRunner.java`.
Change:
```java
responder.setCurrentPlan(response.thinking());
```
to:
```java
responder.setCurrentThinking(response.thinking());
```

- [ ] **Step 9: Update ObservationBuilder — currentPlanSection → currentThinkingSection + planSections**

Use `ide_replace_member` on `ObservationBuilder.currentPlanSection`:

```java
private static io.casehub.blocks.summarisation.observation.affordance.ObservationSection currentThinkingSection(CharacterState character) {
    String thinking = character.currentThinking();
    if (thinking == null || thinking.isBlank()) {return null;}
    return io.casehub.blocks.summarisation.observation.affordance.ObservationSection.text("Your Current Thinking", thinking);
}
```

Use `ide_insert_member` to add:

```java
private static java.util.List<io.casehub.blocks.summarisation.observation.affordance.ObservationSection> planSections(CharacterState character) {
    if (character.plans().isEmpty()) return java.util.List.of();
    return character.plans().entrySet().stream()
        .sorted(java.util.Map.Entry.comparingByKey())
        .map(e -> {
            var plan = e.getValue();
            var items = plan.steps().stream()
                .map(s -> "[" + s.status().name() + "] " + s.description())
                .toList();
            return io.casehub.blocks.summarisation.observation.affordance.ObservationSection.items(
                "Plan: " + e.getKey(), plan.rationale(), items);
        })
        .toList();
}
```

The 4-arg `buildObservation` delegates to the 6-arg, which delegates to the 8-arg. Only the 6-arg and 8-arg call `currentPlanSection(character)`. Replace those two calls with `currentThinkingSection(character)`. Add `planSections(character)` rendering after `goalsSection(goals)` in the same two overloads — iterate the returned list and add each non-null section to the sections list.

- [ ] **Step 10: Update RESPONSE_FORMAT_INSTRUCTION**

Use `ide_replace_member` on `CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION`. Change the thinking field description from:
```
"thinking": "your persistent strategic plan — shown to you next turn as 'Your Current Plan'. Write strategy, not stream-of-consciousness"
```
to:
```
"thinking": "your immediate reasoning about the current situation — what you notice, how it affects your plans, what to do next. Shown to you next turn as 'Your Current Thinking'"
```

- [ ] **Step 11: Update ObservationBuilderTest**

Use `ide_edit_member` on `ObservationBuilderTest` to:
- Update `current_plan_section` tests: `setCurrentPlan` → `setCurrentThinking`, "Your Current Plan" → "Your Current Thinking"
- Add test for `planSections`:

```java
@Test
void plan_sections_render_per_goal_with_status() {
    var step1 = new PlanStep("s1", "Find the poison", PlanStepStatus.COMPLETED);
    var step2 = new PlanStep("s2", "Dispose of it", PlanStepStatus.PENDING);
    var plan = new AgentPlan("protect-penelope", List.of(step1, step2), "protect her", 1, 1, 0);
    character.setPlan("protect-penelope", plan);
    String observation = ObservationBuilder.buildObservation(character, world, goals, drain);
    assertThat(observation).contains("Plan: protect-penelope");
    assertThat(observation).contains("[COMPLETED] Find the poison");
    assertThat(observation).contains("[PENDING] Dispose of it");
}

@Test
void plan_sections_empty_when_no_plans() {
    String observation = ObservationBuilder.buildObservation(character, world, goals, drain);
    assertThat(observation).doesNotContain("Plan:");
}
```

- [ ] **Step 12: Update remaining CharacterState tests**

Remove old `currentPlan_null_by_default` and `currentPlan_set_and_retrieved` tests.

- [ ] **Step 13: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml`
Expected: all tests pass with no currentPlan references remaining

- [ ] **Step 14: Verify no remaining currentPlan references**

```bash
grep -rn "currentPlan\|setCurrentPlan\|Your Current Plan" /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/src/
```
Expected: no matches

- [ ] **Step 15: Commit**

```bash
git add -A wacky-manor/
git commit -m "feat(#44): AgentPlan data model, replace currentPlan with structured plans + currentThinking

Introduce AgentPlan/PlanStep/PlanStepStatus records. Replace
String currentPlan on CharacterState with Map<String, AgentPlan> plans
and String currentThinking. Update ObservationBuilder to render per-goal
plan sections and 'Your Current Thinking'. Update response format
instruction.

Refs #44"
```

---

### Task 2: ManorPlanFormationStrategy

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorPlanFormationStrategy.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorPlanFormationStrategyTest.java`

**Interfaces:**
- Consumes: `AgentPlan`, `PlanStep`, `PlanStepStatus` from Task 1, `AgentProvider` (platform), `AgentGoal` (eidos-api)
- Produces: `ManorPlanFormationStrategy.formPlan(String agentId, String tenancyId, AgentGoal goal, List<AgentGoal> allGoals, List<RetrievedMemory> memories, int currentTick) → AgentPlan`

- [ ] **Step 1: Write failing tests**

Create `ManorPlanFormationStrategyTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.examples.manor.model.AgentPlan;
import io.casehub.examples.manor.model.PlanStepStatus;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ManorPlanFormationStrategyTest {

    private ManorPlanFormationStrategy strategy;
    private String lastPrompt;

    @BeforeEach
    void setUp() {
        AgentProvider mockProvider = config -> {
            lastPrompt = config.userMessage();
            String response = """
                {"steps": [
                  {"id": "find-poison", "description": "Locate the poison bottle in the kitchen"},
                  {"id": "take-poison", "description": "Pick up the poison"}
                ], "rationale": "Need to remove the poison before HC uses it"}
                """;
            return Multi.createFrom().item(new AgentEvent.TextDelta(response));
        };
        strategy = new ManorPlanFormationStrategy(mockProvider);
    }

    @Test
    void formPlan_returns_plan_with_steps() {
        var goal = new AgentGoal("protect-penelope", "Prevent poisoning",
                GoalPriority.PRIMARY, Visibility.PRIVATE, List.of());
        AgentPlan plan = strategy.formPlan("pp", "wacky-manor", goal, List.of(goal), List.of(), 5);
        assertThat(plan.goalName()).isEqualTo("protect-penelope");
        assertThat(plan.steps()).hasSize(2);
        assertThat(plan.steps().get(0).id()).isEqualTo("find-poison");
        assertThat(plan.steps().get(0).status()).isEqualTo(PlanStepStatus.PENDING);
        assertThat(plan.rationale()).isEqualTo("Need to remove the poison before HC uses it");
        assertThat(plan.creationTick()).isEqualTo(5);
    }

    @Test
    void formPlan_prompt_includes_goal_and_context() {
        var goal = new AgentGoal("find-diamond", "Find the hidden diamond",
                GoalPriority.SECONDARY, Visibility.PRIVATE, List.of());
        var otherGoal = new AgentGoal("protect-penelope", "Protect Penelope",
                GoalPriority.PRIMARY, Visibility.PRIVATE, List.of());
        strategy.formPlan("pp", "wacky-manor", goal, List.of(goal, otherGoal), List.of(), 3);
        assertThat(lastPrompt).contains("find-diamond");
        assertThat(lastPrompt).contains("Find the hidden diamond");
        assertThat(lastPrompt).contains("protect-penelope");
    }

    @Test
    void formPlan_returns_empty_on_malformed_response() {
        AgentProvider badProvider = config ->
                Multi.createFrom().item(new AgentEvent.TextDelta("not json"));
        var badStrategy = new ManorPlanFormationStrategy(badProvider);
        var goal = new AgentGoal("g", "desc", GoalPriority.SECONDARY, Visibility.PRIVATE, List.of());
        AgentPlan plan = badStrategy.formPlan("id", "t", goal, List.of(), List.of(), 1);
        assertThat(plan).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorPlanFormationStrategyTest`
Expected: compilation failure — `ManorPlanFormationStrategy` doesn't exist

- [ ] **Step 3: Implement ManorPlanFormationStrategy**

Create `ManorPlanFormationStrategy.java`:

```java
package io.casehub.examples.manor.agent;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.RetrievedMemory;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.examples.manor.model.AgentPlan;
import io.casehub.examples.manor.model.PlanStep;
import io.casehub.examples.manor.model.PlanStepStatus;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

public class ManorPlanFormationStrategy {

    private static final Logger log = Logger.getLogger(ManorPlanFormationStrategy.class);
    private static final ObjectMapper JSON = new ObjectMapper();

    private static final String SYSTEM_PROMPT = """
        You are a tactical planner for an autonomous agent. Given a goal the agent \
        wants to achieve, decompose it into 2-5 concrete, actionable steps. Each step \
        should be achievable in 1-3 turns using available actions (MOVE, TAKE, USE, \
        INTERACT, LOOK, GIVE, STEAL, WAIT). Steps must be specific and ordered.
        Return ONLY a JSON object: {"steps": [{"id": "step-slug", "description": "what to do"}], \
        "rationale": "why this plan"}""";

    private final AgentProvider agentProvider;

    public ManorPlanFormationStrategy(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    public AgentPlan formPlan(String agentId, String tenancyId, AgentGoal goal,
                              List<AgentGoal> allGoals, List<RetrievedMemory> memories,
                              int currentTick) {
        try {
            String userPrompt = buildPrompt(agentId, goal, allGoals, memories);
            String response = agentProvider.invoke(
                    AgentSessionConfig.of(SYSTEM_PROMPT, userPrompt))
                .filter(e -> e instanceof AgentEvent.TextDelta)
                .map(e -> ((AgentEvent.TextDelta) e).text())
                .collect().with(Collectors.joining())
                .await().atMost(Duration.ofSeconds(120));
            return parseResponse(response, goal.name(), currentTick);
        } catch (Exception e) {
            log.warnf("Plan formation failed for goal %s (non-fatal): %s",
                    goal.name(), e.getMessage());
            return null;
        }
    }

    private String buildPrompt(String agentId, AgentGoal goal,
                                List<AgentGoal> allGoals, List<RetrievedMemory> memories) {
        var sb = new StringBuilder();
        sb.append("Agent: ").append(agentId).append("\n");
        sb.append("Goal to plan: ").append(goal.name()).append(" — ").append(goal.description()).append("\n");
        sb.append("Priority: ").append(goal.priority()).append("\n");
        if (allGoals.size() > 1) {
            sb.append("\nOther active goals (for awareness, not planning):\n");
            for (AgentGoal g : allGoals) {
                if (!g.name().equals(goal.name())) {
                    sb.append("- ").append(g.name()).append(": ").append(g.description()).append("\n");
                }
            }
        }
        if (!memories.isEmpty()) {
            sb.append("\nRelevant memories:\n");
            for (RetrievedMemory m : memories) {
                sb.append("- ").append(m.text()).append("\n");
            }
        }
        sb.append("\nRespond with JSON only.");
        return sb.toString();
    }

    private AgentPlan parseResponse(String response, String goalName, int currentTick) {
        try {
            JsonNode root = JSON.readTree(response);
            JsonNode stepsNode = root.get("steps");
            String rationale = root.has("rationale") ? root.get("rationale").asText() : "";
            if (stepsNode == null || !stepsNode.isArray() || stepsNode.isEmpty()) return null;
            List<PlanStep> steps = new ArrayList<>();
            for (JsonNode node : stepsNode) {
                String id = node.get("id").asText();
                String description = node.get("description").asText();
                steps.add(new PlanStep(id, description, PlanStepStatus.PENDING));
            }
            return new AgentPlan(goalName, steps, rationale, currentTick, currentTick, 0);
        } catch (Exception e) {
            log.warnf("Failed to parse plan formation response: %s", e.getMessage());
            return null;
        }
    }
}
```

- [ ] **Step 4: Run tests to verify GREEN**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorPlanFormationStrategyTest`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/.../ManorPlanFormationStrategy.java wacky-manor/src/test/java/.../ManorPlanFormationStrategyTest.java
git commit -m "feat(#44): ManorPlanFormationStrategy — LLM-driven goal decomposition

Decomposes a goal into 2-5 actionable plan steps via AgentProvider.
Parses structured JSON response into AgentPlan with PlanStep list.
All steps start as PENDING.

Refs #44"
```

---

### Task 3: ManorPlanRevisionStrategy (PlanRevisionStrategy SPI)

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorPlanRevisionStrategy.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ActionFailureCause.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ReflectionCause.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorPlanRevisionStrategyTest.java`

**Interfaces:**
- Consumes: `PlanRevisionStrategy` (engine-api SPI), `RevisionContext`, `AdaptationContext`, `RevisedPlan`, `PlanStepDescriptor`, `CompletedStep`, `AdaptationCause` (engine-api), `AgentProvider`, `PlanStep`/`PlanStepStatus` from Task 1
- Produces: `ManorPlanRevisionStrategy` CDI bean implementing `PlanRevisionStrategy.revise(RevisionContext) → RevisedPlan`, `ActionFailureCause(String actionType, String target, String failureReason)`, `ReflectionCause(List<String> insights, int ticksSinceLastRevision)`

- [ ] **Step 1: Create AdaptationCause implementations**

Create `ActionFailureCause.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.engine.plan.adaptation.AdaptationCause;

public record ActionFailureCause(
        String actionType,
        String target,
        String failureReason) implements AdaptationCause {}
```

Create `ReflectionCause.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.engine.plan.adaptation.AdaptationCause;

import java.util.List;

public record ReflectionCause(
        List<String> insights,
        int ticksSinceLastRevision) implements AdaptationCause {}
```

- [ ] **Step 2: Write failing tests**

Create `ManorPlanRevisionStrategyTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.engine.plan.adaptation.AdaptationContext;
import io.casehub.engine.plan.adaptation.PlanStepDescriptor;
import io.casehub.engine.plan.adaptation.RevisedPlan;
import io.casehub.engine.plan.adaptation.RevisionContext;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ManorPlanRevisionStrategyTest {

    private ManorPlanRevisionStrategy strategy;
    private String lastPrompt;

    @BeforeEach
    void setUp() {
        AgentProvider mockProvider = config -> {
            lastPrompt = config.userMessage();
            String response = """
                {"steps": [
                  {"id": "alt-route", "description": "Find another way to the kitchen"},
                  {"id": "take-poison", "description": "Pick up the poison"}
                ], "rationale": "Door is locked, need alternative route"}
                """;
            return Multi.createFrom().item(new AgentEvent.TextDelta(response));
        };
        strategy = new ManorPlanRevisionStrategy(mockProvider);
    }

    @Test
    void revise_returns_revised_plan() {
        var pending = List.of(
                new PlanStepDescriptor("go-kitchen", "Go to the kitchen", null),
                new PlanStepDescriptor("take-poison", "Pick up the poison", null));
        var cause = new ActionFailureCause("MOVE", "kitchen", "The door is locked");
        var adaptCtx = new AdaptationContext(null, "wacky-manor", null,
                "protect-penelope", List.of(), pending, List.of(),
                null, null, null, null, 0);
        var ctx = new RevisionContext(adaptCtx, cause, List.of(), List.of());

        RevisedPlan result = strategy.revise(ctx);

        assertThat(result.steps()).hasSize(2);
        assertThat(result.steps().get(0).id()).isEqualTo("alt-route");
        assertThat(result.rationale()).contains("locked");
    }

    @Test
    void revise_prompt_includes_failure_context() {
        var pending = List.of(new PlanStepDescriptor("s1", "Do something", null));
        var cause = new ActionFailureCause("TAKE", "poison", "Someone already took it");
        var adaptCtx = new AdaptationContext(null, "wacky-manor", null,
                "eliminate", List.of(), pending, List.of(),
                null, null, null, null, 1);
        var ctx = new RevisionContext(adaptCtx, cause, List.of(), List.of());

        strategy.revise(ctx);

        assertThat(lastPrompt).contains("eliminate");
        assertThat(lastPrompt).contains("Someone already took it");
    }

    @Test
    void revise_returns_empty_on_malformed_response() {
        AgentProvider badProvider = config ->
                Multi.createFrom().item(new AgentEvent.TextDelta("not json"));
        var badStrategy = new ManorPlanRevisionStrategy(badProvider);
        var adaptCtx = new AdaptationContext(null, "wacky-manor", null,
                "goal", List.of(), List.of(), List.of(),
                null, null, null, null, 0);
        var ctx = new RevisionContext(adaptCtx, new ReflectionCause(List.of(), 5),
                List.of(), List.of());

        RevisedPlan result = badStrategy.revise(ctx);

        assertThat(result.steps()).isEmpty();
    }

    @Test
    void id_returns_manor_llm() {
        assertThat(strategy.id()).isEqualTo("manor-llm");
    }
}
```

- [ ] **Step 3: Run tests to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorPlanRevisionStrategyTest`
Expected: compilation failure — `ManorPlanRevisionStrategy` doesn't exist

- [ ] **Step 4: Implement ManorPlanRevisionStrategy**

Create `ManorPlanRevisionStrategy.java`:

```java
package io.casehub.examples.manor.agent;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.engine.plan.adaptation.AdaptationCause;
import io.casehub.engine.plan.adaptation.CompletedStep;
import io.casehub.engine.plan.adaptation.PlanRevisionStrategy;
import io.casehub.engine.plan.adaptation.PlanStepDescriptor;
import io.casehub.engine.plan.adaptation.RevisedPlan;
import io.casehub.engine.plan.adaptation.RevisionContext;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

@ApplicationScoped
public class ManorPlanRevisionStrategy implements PlanRevisionStrategy {

    private static final Logger log = Logger.getLogger(ManorPlanRevisionStrategy.class);
    private static final ObjectMapper JSON = new ObjectMapper();

    private static final String SYSTEM_PROMPT = """
        You are a tactical replanner for an autonomous agent. Given the agent's \
        current plan (with completed and pending steps) and a reason for revision \
        (action failure or strategic reassessment), propose a revised plan. \
        Keep completed steps as context. Revise or replace pending steps as needed. \
        Each step should be achievable in 1-3 turns.
        Return ONLY a JSON object: {"steps": [{"id": "step-slug", "description": "what to do"}], \
        "rationale": "why this revision"}""";

    private final AgentProvider agentProvider;

    @Inject
    public ManorPlanRevisionStrategy(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    @Override
    public String id() { return "manor-llm"; }

    @Override
    public RevisedPlan revise(RevisionContext context) {
        try {
            String userPrompt = buildPrompt(context);
            String response = agentProvider.invoke(
                    AgentSessionConfig.of(SYSTEM_PROMPT, userPrompt))
                .filter(e -> e instanceof AgentEvent.TextDelta)
                .map(e -> ((AgentEvent.TextDelta) e).text())
                .collect().with(Collectors.joining())
                .await().atMost(Duration.ofSeconds(120));
            return parseResponse(response);
        } catch (Exception e) {
            log.warnf("Plan revision failed (non-fatal): %s", e.getMessage());
            return new RevisedPlan(List.of(), "");
        }
    }

    private String buildPrompt(RevisionContext context) {
        var adaptCtx = context.adaptationContext();
        var sb = new StringBuilder();
        sb.append("Goal: ").append(adaptCtx.goalName()).append("\n");
        sb.append("Revision #").append(adaptCtx.adaptationGeneration() + 1).append("\n");

        if (!adaptCtx.completedSteps().isEmpty()) {
            sb.append("\nCompleted steps:\n");
            for (CompletedStep step : adaptCtx.completedSteps()) {
                sb.append("- [DONE] ").append(step.description()).append("\n");
            }
        }
        if (!adaptCtx.pendingSteps().isEmpty()) {
            sb.append("\nPending steps:\n");
            for (PlanStepDescriptor step : adaptCtx.pendingSteps()) {
                sb.append("- [PENDING] ").append(step.id()).append(": ").append(step.description()).append("\n");
            }
        }

        AdaptationCause cause = context.cause();
        if (cause instanceof ActionFailureCause failure) {
            sb.append("\nRevision trigger: ACTION FAILURE\n");
            sb.append("  Action: ").append(failure.actionType()).append(" ").append(failure.target()).append("\n");
            sb.append("  Reason: ").append(failure.failureReason()).append("\n");
        } else if (cause instanceof ReflectionCause reflection) {
            sb.append("\nRevision trigger: STRATEGIC REASSESSMENT\n");
            sb.append("  Ticks since last revision: ").append(reflection.ticksSinceLastRevision()).append("\n");
            if (!reflection.insights().isEmpty()) {
                sb.append("  Recent insights:\n");
                for (String insight : reflection.insights()) {
                    sb.append("  - ").append(insight).append("\n");
                }
            }
        }

        if (!context.memories().isEmpty()) {
            sb.append("\nRelevant memories:\n");
            for (var m : context.memories()) {
                sb.append("- ").append(m.text()).append("\n");
            }
        }
        sb.append("\nRespond with JSON only.");
        return sb.toString();
    }

    private RevisedPlan parseResponse(String response) {
        try {
            JsonNode root = JSON.readTree(response);
            JsonNode stepsNode = root.get("steps");
            String rationale = root.has("rationale") ? root.get("rationale").asText() : "";
            List<PlanStepDescriptor> steps = new ArrayList<>();
            if (stepsNode != null && stepsNode.isArray()) {
                for (JsonNode node : stepsNode) {
                    steps.add(new PlanStepDescriptor(
                            node.get("id").asText(),
                            node.get("description").asText(),
                            null));
                }
            }
            return new RevisedPlan(steps, rationale);
        } catch (Exception e) {
            log.warnf("Failed to parse plan revision response: %s", e.getMessage());
            return new RevisedPlan(List.of(), "");
        }
    }
}
```

- [ ] **Step 5: Run tests to verify GREEN**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorPlanRevisionStrategyTest`
Expected: all pass

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/.../ManorPlanRevisionStrategy.java wacky-manor/src/main/java/.../ActionFailureCause.java wacky-manor/src/main/java/.../ReflectionCause.java wacky-manor/src/test/java/.../ManorPlanRevisionStrategyTest.java
git commit -m "feat(#44): ManorPlanRevisionStrategy — PlanRevisionStrategy SPI implementation

Implements engine PlanRevisionStrategy SPI. Builds AdaptationContext
with manor data (goalName, completed/pending steps) and nulls for
case-specific fields. Parses RevisedPlan from LLM JSON response.
Two AdaptationCause types: ActionFailureCause and ReflectionCause.

Refs #44"
```

---

### Task 4: ManorPlanEvaluator and wiring

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorPlanEvaluator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalEvaluator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorPlanEvaluatorTest.java`

**Interfaces:**
- Consumes: `ManorPlanFormationStrategy.formPlan()` from Task 2, `ManorPlanRevisionStrategy.revise()` from Task 3, `AgentPlan`/`PlanStep`/`PlanStepStatus` from Task 1, `CharacterState.plans()`/`setPlan()`/`removePlan()` from Task 1, `ManorGoalEvaluator.evaluate()`, `ActionResult.Failed`
- Produces: `ManorPlanEvaluator.formPlanForGoal(String, AgentGoal, List<AgentGoal>, int)`, `ManorPlanEvaluator.removePlanForGoal(String, String)`, `ManorPlanEvaluator.reviseOnFailure(String, String, String, ActionResult.Failed, int)`, `ManorPlanEvaluator.reviseOnReflection(String, List<String>, int)`

- [ ] **Step 1: Write failing tests**

Create `ManorPlanEvaluatorTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.api.model.RetrievedMemory;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.engine.plan.adaptation.PlanStepDescriptor;
import io.casehub.engine.plan.adaptation.RevisedPlan;
import io.casehub.examples.manor.model.ActionResult;
import io.casehub.examples.manor.model.AgentPlan;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.model.PlanStep;
import io.casehub.examples.manor.model.PlanStepStatus;
import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.MemoryQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.function.Function;

import static org.assertj.core.api.Assertions.assertThat;

class ManorPlanEvaluatorTest {

    private ManorPlanEvaluator evaluator;
    private CharacterState character;
    private AgentPlan nextFormationResult;
    private RevisedPlan nextRevisionResult;

    @BeforeEach
    void setUp() {
        character = new CharacterState("hc", "Hooded Claw", "ballroom", 0, List.of());

        nextFormationResult = new AgentPlan("goal-a",
                List.of(new PlanStep("s1", "Step 1", PlanStepStatus.PENDING)),
                "rationale", 1, 1, 0);

        nextRevisionResult = new RevisedPlan(
                List.of(new PlanStepDescriptor("s2", "Revised step", null)),
                "revised because failure");

        ManorPlanFormationStrategy formationStrategy = new ManorPlanFormationStrategy(null) {
            @Override
            public AgentPlan formPlan(String agentId, String tenancyId, AgentGoal goal,
                    java.util.List<AgentGoal> allGoals, java.util.List<RetrievedMemory> memories, int tick) {
                return nextFormationResult;
            }
        };

        ManorPlanRevisionStrategy revisionStrategy = new ManorPlanRevisionStrategy(null) {
            @Override
            public RevisedPlan revise(io.casehub.engine.plan.adaptation.RevisionContext ctx) {
                return nextRevisionResult;
            }
        };

        CaseMemoryStore memoryStore = new CaseMemoryStore() {
            @Override
            public java.util.List<io.casehub.neocortex.memory.Memory> query(MemoryQuery q) {
                return List.of();
            }
        };

        Function<String, CharacterState> lookup = id -> character;

        evaluator = new ManorPlanEvaluator(formationStrategy, revisionStrategy,
                memoryStore, "wacky-manor", lookup, 5);
    }

    @Test
    void formPlanForGoal_stores_plan_on_character() {
        var goal = new AgentGoal("goal-a", "Do something",
                GoalPriority.PRIMARY, Visibility.PRIVATE, List.of());
        evaluator.formPlanForGoal("hc", goal, List.of(goal), 1);
        assertThat(character.plans()).containsKey("goal-a");
        assertThat(character.plans().get("goal-a").steps()).hasSize(1);
    }

    @Test
    void removePlanForGoal_removes_plan() {
        character.setPlan("goal-a", nextFormationResult);
        evaluator.removePlanForGoal("hc", "goal-a");
        assertThat(character.plans()).isEmpty();
    }

    @Test
    void reviseOnFailure_updates_plan() {
        character.setPlan("goal-a", nextFormationResult);
        var failure = new ActionResult.Failed("The door is locked");
        evaluator.reviseOnFailure("hc", "MOVE", "kitchen", failure, 5);
        var plan = character.plans().get("goal-a");
        assertThat(plan.steps().get(0).id()).isEqualTo("s2");
        assertThat(plan.revisionGeneration()).isEqualTo(1);
    }

    @Test
    void reviseOnFailure_skips_when_no_plans() {
        var failure = new ActionResult.Failed("Something failed");
        evaluator.reviseOnFailure("hc", "TAKE", "poison", failure, 5);
        assertThat(character.plans()).isEmpty();
    }

    @Test
    void reviseOnFailure_respects_max_generation() {
        var maxedPlan = new AgentPlan("goal-a",
                List.of(new PlanStep("s1", "Step 1", PlanStepStatus.PENDING)),
                "r", 1, 4, 5);
        character.setPlan("goal-a", maxedPlan);
        var failure = new ActionResult.Failed("fail");
        evaluator.reviseOnFailure("hc", "MOVE", "library", failure, 6);
        assertThat(character.plans().get("goal-a").revisionGeneration()).isEqualTo(5);
    }
}
```

- [ ] **Step 2: Run tests to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorPlanEvaluatorTest`
Expected: compilation failure — `ManorPlanEvaluator` doesn't exist

- [ ] **Step 3: Implement ManorPlanEvaluator**

Create `ManorPlanEvaluator.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.api.model.RetrievedMemory;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.engine.plan.adaptation.AdaptationContext;
import io.casehub.engine.plan.adaptation.CompletedStep;
import io.casehub.engine.plan.adaptation.PlanStepDescriptor;
import io.casehub.engine.plan.adaptation.RevisionContext;
import io.casehub.examples.manor.model.ActionResult;
import io.casehub.examples.manor.model.AgentPlan;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.model.PlanStep;
import io.casehub.examples.manor.model.PlanStepStatus;
import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.MemoryOrder;
import io.casehub.neocortex.memory.MemoryQuery;
import org.jboss.logging.Logger;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.function.Function;

public class ManorPlanEvaluator {

    private static final Logger log = Logger.getLogger(ManorPlanEvaluator.class);

    private final ManorPlanFormationStrategy formationStrategy;
    private final ManorPlanRevisionStrategy revisionStrategy;
    private final CaseMemoryStore memoryStore;
    private final String tenancyId;
    private final Function<String, CharacterState> characterLookup;
    private final int maxRevisionGeneration;

    public ManorPlanEvaluator(ManorPlanFormationStrategy formationStrategy,
                               ManorPlanRevisionStrategy revisionStrategy,
                               CaseMemoryStore memoryStore,
                               String tenancyId,
                               Function<String, CharacterState> characterLookup,
                               int maxRevisionGeneration) {
        this.formationStrategy = formationStrategy;
        this.revisionStrategy = revisionStrategy;
        this.memoryStore = memoryStore;
        this.tenancyId = tenancyId;
        this.characterLookup = characterLookup;
        this.maxRevisionGeneration = maxRevisionGeneration;
    }

    public void formPlanForGoal(String agentId, AgentGoal goal,
                                 List<AgentGoal> allGoals, int currentTick) {
        try {
            List<RetrievedMemory> memories = retrieveMemories(agentId);
            AgentPlan plan = formationStrategy.formPlan(
                    agentId, tenancyId, goal, allGoals, memories, currentTick);
            if (plan != null && !plan.steps().isEmpty()) {
                characterLookup.apply(agentId).setPlan(goal.name(), plan);
            }
        } catch (Exception e) {
            log.warnf(e, "Plan formation failed for agent %s goal %s", agentId, goal.name());
        }
    }

    public void removePlanForGoal(String agentId, String goalName) {
        characterLookup.apply(agentId).removePlan(goalName);
    }

    public void reviseOnFailure(String agentId, String actionType, String actionTarget,
                                ActionResult.Failed failure, int currentTick) {
        var character = characterLookup.apply(agentId);
        if (character.plans().isEmpty()) return;

        var cause = new ActionFailureCause(
                actionType, actionTarget, failure.reason());
        List<RetrievedMemory> memories = retrieveMemories(agentId);

        for (Map.Entry<String, AgentPlan> entry : character.plans().entrySet()) {
            var plan = entry.getValue();
            if (plan.revisionGeneration() >= maxRevisionGeneration) continue;
            revisePlan(character, entry.getKey(), plan, cause, memories, currentTick);
        }
    }

    public void reviseOnReflection(String agentId, List<String> insights, int currentTick) {
        var character = characterLookup.apply(agentId);
        if (character.plans().isEmpty()) return;

        List<RetrievedMemory> memories = retrieveMemories(agentId);

        for (Map.Entry<String, AgentPlan> entry : character.plans().entrySet()) {
            var plan = entry.getValue();
            if (plan.revisionGeneration() >= maxRevisionGeneration) continue;
            int ticksSince = currentTick - plan.lastRevisionTick();
            var cause = new ReflectionCause(insights, ticksSince);
            revisePlan(character, entry.getKey(), plan, cause, memories, currentTick);
        }
    }

    private void revisePlan(CharacterState character, String goalName, AgentPlan plan,
                             io.casehub.engine.plan.adaptation.AdaptationCause cause,
                             List<RetrievedMemory> memories, int currentTick) {
        try {
            List<CompletedStep> completed = plan.steps().stream()
                    .filter(s -> s.status() == PlanStepStatus.COMPLETED)
                    .map(s -> new CompletedStep(s.id(), null, s.description(), Map.of(), null))
                    .toList();
            List<PlanStepDescriptor> pending = plan.steps().stream()
                    .filter(s -> s.status() != PlanStepStatus.COMPLETED)
                    .map(s -> new PlanStepDescriptor(s.id(), s.description(), null))
                    .toList();

            var adaptCtx = new AdaptationContext(null, tenancyId, null,
                    goalName, completed, pending, List.of(),
                    null, null, null, null, plan.revisionGeneration());
            var ctx = new RevisionContext(adaptCtx, cause, List.of(), memories);

            var revised = revisionStrategy.revise(ctx);
            if (revised != null && !revised.steps().isEmpty()) {
                List<PlanStep> newSteps = revised.steps().stream()
                        .map(s -> new PlanStep(s.id(), s.description(), PlanStepStatus.PENDING))
                        .toList();
                character.setPlan(goalName, plan.withRevision(newSteps, revised.rationale(), currentTick));
            }
        } catch (Exception e) {
            log.warnf(e, "Plan revision failed for goal %s", goalName);
        }
    }

    private List<RetrievedMemory> retrieveMemories(String agentId) {
        try {
            var memories = memoryStore.query(MemoryQuery.forEntity(agentId,
                    new MemoryDomain("manor"), tenancyId)
                    .withLimit(10).withOrder(MemoryOrder.SALIENCE));
            return memories.stream()
                    .map(m -> new RetrievedMemory(m.memoryId(), m.text(),
                            m.domain().name(), m.createdAt(), m.attributes()))
                    .toList();
        } catch (Exception e) {
            log.debugf("Failed to retrieve memories for %s: %s", agentId, e.getMessage());
            return List.of();
        }
    }
}
```

- [ ] **Step 4: Run tests to verify GREEN**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorPlanEvaluatorTest`
Expected: all pass

- [ ] **Step 5: Wire ManorGoalEvaluator to delegate plan events**

Use `ide_edit_member` on `ManorGoalEvaluator`:
- Add field: `private final ManorPlanEvaluator planEvaluator;`
- Update constructor to accept `ManorPlanEvaluator planEvaluator` (nullable)
- In `evaluate()`, after registering new goals, add plan formation:
  ```java
  if (planEvaluator != null) {
      for (AgentGoal newGoal : validated) {
          planEvaluator.formPlanForGoal(agentId, newGoal, finalGoals, currentTick);
      }
  }
  ```
- In `evaluate()`, after goal abandonment/completion (inside the revision block), add plan removal:
  ```java
  if (planEvaluator != null) {
      planEvaluator.removePlanForGoal(agentId, goalName);
  }
  ```

- [ ] **Step 6: Wire AgentExperienceService**

Use `ide_edit_member` on `AgentExperienceService`:
- Add field: `private final ManorPlanEvaluator planEvaluator;`
- Add a new constructor overload that accepts `ManorPlanEvaluator` (nullable), chaining to existing constructor
- In `runReflection()`, after goal evaluation, add:
  ```java
  if (planEvaluator != null) {
      var insightTexts = events.stream()
          .map(io.casehub.neocortex.memory.ReflectionEvent::insight).toList();
      planEvaluator.reviseOnReflection(agentId, insightTexts, currentTick);
  }
  ```

- [ ] **Step 7: Wire ScenarioOrchestrator**

Use `ide_edit_member` on `ScenarioOrchestrator`:
- Add config properties:
  ```java
  @ConfigProperty(name = "manor.plan.enabled", defaultValue = "true")
  boolean planEnabled;

  @ConfigProperty(name = "manor.plan.revision.max-generation", defaultValue = "5")
  int planMaxRevisionGeneration;
  ```
- In `runScenario()`, construct `ManorPlanEvaluator` (if `planEnabled`) with injected strategies
- Pass `planEvaluator` to `ManorGoalEvaluator` constructor and `AgentExperienceService` constructor
- In `runAutonomousTicks()`, after the action resolution block, add:
  ```java
  if (result instanceof ActionResult.Failed failure && planEvaluator != null) {
      String aType = response.action() != null ? response.action().type().name() : "WAIT";
      String aTarget = response.action() != null ? response.action().target() : null;
      planEvaluator.reviseOnFailure(c.agentId(), aType, aTarget, failure, currentTick);
  }
  ```

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml`
Expected: all tests pass

- [ ] **Step 9: Commit**

```bash
git add -A wacky-manor/src/
git commit -m "feat(#44): ManorPlanEvaluator — plan lifecycle orchestration and wiring

ManorPlanEvaluator orchestrates plan formation, revision, and removal.
Plans form when goals form (via ManorGoalEvaluator), revise on action
failure (reactive, via ScenarioOrchestrator) and during reflection
(deliberative, via AgentExperienceService). Max revision generation
caps infinite revision loops.

Refs #44"
```

---

### Task 5: LLM eval tests for plan lifecycle

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/PlanLifecycleEvalTest.java`

**Interfaces:**
- Consumes: all components from Tasks 1-4

- [ ] **Step 1: Write LLM eval tests**

Create `PlanLifecycleEvalTest.java` under the `llm-eval` profile (same pattern as goal lifecycle eval tests):

```java
package io.casehub.examples.manor.agent;

import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.examples.manor.model.PlanStepStatus;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIfSystemProperty;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

@EnabledIfSystemProperty(named = "llm.eval", matches = "true")
class PlanLifecycleEvalTest {

    @Test
    void plan_formation_produces_actionable_steps() {
        // Use real AgentProvider (from test config)
        // Form a plan for "poison Penelope's tea"
        // Verify: 2-5 steps, all PENDING, steps reference concrete actions
        var goal = new AgentGoal("poison-tea", "Poison Penelope's tea",
                GoalPriority.PRIMARY, Visibility.PRIVATE, List.of());
        // ... real provider invocation, assert step count and structure
    }

    @Test
    void plan_revision_adapts_to_failure() {
        // Create a plan, simulate a failure, revise
        // Verify: revised plan addresses the failure, steps are different
    }
}
```

(Exact implementation follows the pattern from `GoalLifecycleEvalTest` — wire real `AgentProvider` from test configuration, invoke strategies, assert structural properties of LLM output.)

- [ ] **Step 2: Run eval tests (optional — requires API key)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Pllm-eval`

- [ ] **Step 3: Commit**

```bash
git add wacky-manor/src/test/java/.../PlanLifecycleEvalTest.java
git commit -m "test(#44): LLM eval tests for plan lifecycle

Eval tests for plan formation (produces actionable steps) and
plan revision (adapts to failure context). Requires API key,
runs under llm-eval profile.

Refs #44"
```

---

## Amendments

**A1: Step status assessment in reviseOnReflection (Task 4)**

The plan's `reviseOnReflection()` calls `PlanRevisionStrategy.revise()`
but never updates step statuses (PENDING → COMPLETED/FAILED). Without
this, `completedSteps` in `AdaptationContext` is always empty.

Add a step status assessment LLM call at the start of
`reviseOnReflection()`. The prompt receives current plans + recent
memories and returns status updates per step. Steps whose status
changes are updated on `CharacterState` before revision proceeds.
This ensures `AdaptationContext.completedSteps()` reflects actual
progress. The assessment and revision can share the same LLM call
by combining prompts — assess status and propose revisions in one
response.

**A2: Goal revision triggers plan revision (Task 4)**

When `ManorGoalRevisionStrategy` revises a goal (changes description),
the goal's plan may be stale. Add a `reviseOnGoalChange` method to
`ManorPlanEvaluator` called from `ManorGoalEvaluator` after goal
revision. Uses `ReflectionCause` with the revision reason as context.

**A3: Tick-count cooldown for plan revision (Task 4)**

Following the pattern from #43 amendment A1 (tick-count cooldown for
goal formation), plan revision should use tick-based cooldown rather
than wall-clock. The `reviseOnReflection()` method already receives
`currentTick` and can check `ticksSince = currentTick - plan.lastRevisionTick()`.
Add a `planRevisionCooldownTicks` config property (default: 3) and skip
revision if `ticksSince < cooldownTicks`.
