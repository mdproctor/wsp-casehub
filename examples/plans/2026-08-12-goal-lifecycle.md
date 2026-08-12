# Goal Lifecycle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #43 — Goal lifecycle — replace DynamicGoal with engine-backed AgentGoal
**Issue group:** #41

**Goal:** Replace the manor's hand-rolled DynamicGoal management with the engine's cognitive stack — goals emerge from reflection via GoalFormationStrategy, evolve via GoalRevisionStrategy, and live on AgentRegistry as the single source of truth.

**Architecture:** Three new components (ManorGoalFormationStrategy, ManorGoalRevisionStrategy, ManorGoalEvaluator) chain after the existing reflection pipeline. Formation runs immediately; revision is gated on engine#903 (GoalRevisionAction enum). DynamicGoal, newGoals/dropGoals on AgentResponse, and goal processing in the tick loop are removed.

**Tech Stack:** Java 21, Quarkus, Jackson, JUnit 5, AssertJ, Eidos API, Neocortex Memory API, Engine API (new dep)

## Global Constraints

- Pre-release: breaking changes cost nothing
- IntelliJ MCP for all .java edits — never Edit/Write on existing source files
- TDD: failing test before implementation, always
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml` for test runs
- Commit after each task with `Refs #43`
- Revision strategy (Task 5) is blocked on engine#903 — skip if not yet shipped

---

### Task 1: Add casehub-engine-api dependency and remove DynamicGoal

**Files:**
- Modify: `wacky-manor/pom.xml`
- Delete: `wacky-manor/src/main/java/io/casehub/examples/manor/model/DynamicGoal.java` (use `ide_refactor_safe_delete`)
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/CharacterState.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentResponse.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentResponseTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/model/CharacterStateTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Produces: Clean codebase with no DynamicGoal references, no newGoals/dropGoals on AgentResponse, engine-api types available for import

- [ ] **Step 1: Add casehub-engine-api dependency to pom.xml**

Add after the `casehub-blocks` dependency in `wacky-manor/pom.xml`:

```xml
<!-- CaseHub Engine API — goal formation/revision SPIs -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

- [ ] **Step 2: Remove newGoals/dropGoals from AgentResponse**

Use `ide_replace_member` to replace the `AgentResponse` record. New record:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record AgentResponse(
        String thinking,
        String dialogue,
        String talkTo,
        String aside,
        Action action) {

    private static final ObjectMapper JSON = new ObjectMapper();
    private static final Pattern CODE_BLOCK = Pattern.compile("```(?:json)?\\s*\\n?(\\{.*?})\\s*```", Pattern.DOTALL);

    public static AgentResponse parse(String text) {
        try {
            String json = extractJson(text);
            return JSON.readValue(json, AgentResponse.class);
        } catch (Exception e) {
            System.getLogger(AgentResponse.class.getName()).log(System.Logger.Level.WARNING, "Failed to parse agent response: " + e.getMessage());
            return idle();
        }
    }

    public static AgentResponse idle() {
        return new AgentResponse(null, null, null, null,
                                 new Action(ActionType.WAIT, null, null));
    }

    private static String extractJson(String text) {
        text = text.strip();
        if (text.startsWith("{")) return text;
        Matcher m = CODE_BLOCK.matcher(text);
        if (m.find()) return m.group(1);
        int start = text.indexOf('{');
        int end = text.lastIndexOf('}');
        if (start >= 0 && end > start) return text.substring(start, end + 1);
        return text;
    }
}
```

Removes: `GoalEntry` inner record, `newGoals` field, `dropGoals` field.

- [ ] **Step 3: Remove newGoals/dropGoals from RESPONSE_FORMAT_INSTRUCTION**

Use `ide_replace_member` on `CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION`. Remove the two lines:

```
"newGoals": [{"name": "goal-name", "description": "why this goal matters"}],
"dropGoals": ["goal-name-to-remove"]
```

Updated format instruction JSON block:

```java
public static final String RESPONSE_FORMAT_INSTRUCTION = "\n" +
    new io.casehub.blocks.summarisation.observation.affordance.AffordanceRenderer().renderActionVocabulary(
        """
        You MUST respond with ONLY a JSON object in this exact format:
        {
          "thinking": "your persistent strategic plan — shown to you next turn as 'Your Current Plan'. Write strategy, not stream-of-consciousness",
          "dialogue": "what you say aloud (or null if silent)",
          "talkTo": "character-id to direct dialogue at (or null for broadcast)",
          "aside": "private thoughts for the audience only (or null)",
          "action": {
            "type": "one of the action types below",
            "target": "room-id or object-id or character-id (or null for WAIT)",
            "withItem": "inventory-item-id to use (or null)"
          }
        }
        
        ACTION TYPES — use the right one for your intent:""",
        ACTION_DESCRIPTORS) +
    "\n\nTo get an object, use TAKE. To apply an item you're carrying, use USE.\nRespond with ONLY the JSON. No other text.";
```

- [ ] **Step 4: Remove dynamicGoals from CharacterState**

Use `ide_edit_member` to remove from `CharacterState`:
- Field: `MAX_DYNAMIC_GOALS`
- Field: `dynamicGoals`
- Method: `dynamicGoals()`
- Method: `addDynamicGoal(DynamicGoal)`
- Method: `dropDynamicGoal(String)`
- Method: `dropAllDynamicGoals()`

- [ ] **Step 5: Delete DynamicGoal.java**

Use `ide_refactor_safe_delete` on `DynamicGoal.java`.

- [ ] **Step 6: Simplify ObservationBuilder.goalsSection()**

Use `ide_replace_member` on `goalsSection`. Remove the `CharacterState character` parameter. New method:

```java
private static io.casehub.blocks.summarisation.observation.affordance.ObservationSection goalsSection(
        java.util.List<io.casehub.eidos.api.AgentGoal> goals) {
    var items = new java.util.ArrayList<String>();
    goals.stream()
         .sorted(java.util.Comparator.comparing(io.casehub.eidos.api.AgentGoal::priority)
                                     .thenComparing(io.casehub.eidos.api.AgentGoal::name))
         .map(g -> "[" + g.priority().name() + "] " + g.description())
         .forEach(items::add);
    if (items.isEmpty()) {
        return io.casehub.blocks.summarisation.observation.affordance.ObservationSection.items(
                "Your Goals", "No specific goals.", java.util.List.of());
    }
    return io.casehub.blocks.summarisation.observation.affordance.ObservationSection.items(
            "Your Goals", null, items);
}
```

Update all call sites: `goalsSection(goals, character)` → `goalsSection(goals)`. There are two in `buildObservation()` (6-arg and 8-arg overloads).

- [ ] **Step 7: Remove goal processing from ScenarioOrchestrator**

Use `ide_edit_member` to remove the newGoals/dropGoals block in `runAutonomousTicks()`. Delete the block that starts with `if (response.dropGoals() != null)` and ends after the `response.newGoals().forEach(...)` call.

- [ ] **Step 8: Update tests**

**AgentResponseTest** — use `ide_edit_member` to:
- Remove `parse_includes_newGoals` test
- Remove `parse_includes_dropGoals` test
- Update `idle_has_null_for_new_fields` — remove assertions for `newGoals()` and `dropGoals()`, keep `talkTo()` assertion

**CharacterStateTest** — use `ide_edit_member` to remove:
- `dynamicGoals_empty_by_default`
- `addDynamicGoal_and_retrieve`
- `dropDynamicGoal_removes_by_normalized_name`
- `dropAllDynamicGoals_clears_all`
- `addDynamicGoal_replaces_existing_with_same_name`
- `dynamicGoals_capped_evicts_oldest`

**ObservationBuilderTest** — use `ide_edit_member` to:
- Remove `goals_section_renders_dynamic_goals_with_situational_prefix` test
- Remove/update any test that calls `character.addDynamicGoal()`
- Update any `goalsSection` call site that passes `character` parameter

- [ ] **Step 9: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml`
Expected: all tests pass with no DynamicGoal references remaining

- [ ] **Step 10: Verify no remaining DynamicGoal references**

```bash
grep -rn "DynamicGoal\|dynamicGoal\|newGoals\|dropGoals\|GoalEntry\|SITUATIONAL" /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/src/
```
Expected: no matches (except possibly comments referencing the old design)

- [ ] **Step 11: Commit**

```bash
git add -A wacky-manor/
git commit -m "feat(#43): remove DynamicGoal, add casehub-engine-api dependency

Remove DynamicGoal record, CharacterState.dynamicGoals, AgentResponse
newGoals/dropGoals, and goal processing from tick loop. Add engine-api
dependency for GoalFormationStrategy/GoalRevisionStrategy SPIs.
Goals now come from AgentRegistry only.

Refs #43"
```

---

### Task 2: ManorGoalFormationStrategy

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalFormationStrategy.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorGoalFormationStrategyTest.java`

**Interfaces:**
- Consumes: `GoalFormationStrategy` (engine-api SPI), `GoalFormationContext`, `GoalFormationProposal`, `AgentProvider`
- Produces: `ManorGoalFormationStrategy` CDI bean implementing `GoalFormationStrategy.propose(GoalFormationContext) → GoalFormationProposal`

- [ ] **Step 1: Write failing tests**

Create `ManorGoalFormationStrategyTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.api.spi.routing.GoalFormationContext;
import io.casehub.api.spi.routing.GoalFormationProposal;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class ManorGoalFormationStrategyTest {

    private ManorGoalFormationStrategy strategy;
    private String lastPrompt;

    @BeforeEach
    void setUp() {
        AgentProvider mockProvider = config -> {
            lastPrompt = config.userMessage();
            String response = """
                {"goals": [{"name": "protect-tea", "description": "Prevent poisoning",
                  "suggestedPriority": "SECONDARY", "formationReason": "Observed suspicious behavior"}],
                 "rationale": "Agent noticed danger signs"}
                """;
            return Multi.createFrom().item(new AgentEvent.TextDelta(response));
        };
        strategy = new ManorGoalFormationStrategy(mockProvider);
    }

    @Test
    void propose_returns_goals_from_llm_response() {
        var context = new GoalFormationContext("hc", "wacky-manor",
                List.of("Sneekly is acting suspiciously"),
                List.of(new AgentGoal("find-diamond", "Find the diamond",
                        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of())),
                List.of(), 5);
        GoalFormationProposal proposal = strategy.propose(context);
        assertThat(proposal.goals()).hasSize(1);
        assertThat(proposal.goals().get(0).name()).isEqualTo("protect-tea");
        assertThat(proposal.goals().get(0).formationReason()).isEqualTo("Observed suspicious behavior");
    }

    @Test
    void propose_prompt_includes_insights_and_goals() {
        var context = new GoalFormationContext("hc", "wacky-manor",
                List.of("HC spotted poison"),
                List.of(new AgentGoal("eliminate", "Eliminate Penelope",
                        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of())),
                List.of(), 3);
        strategy.propose(context);
        assertThat(lastPrompt).contains("HC spotted poison");
        assertThat(lastPrompt).contains("eliminate");
        assertThat(lastPrompt).contains("Remaining goal capacity: 3");
    }

    @Test
    void propose_returns_empty_on_malformed_response() {
        AgentProvider badProvider = config ->
                Multi.createFrom().item(new AgentEvent.TextDelta("not json"));
        var badStrategy = new ManorGoalFormationStrategy(badProvider);
        var context = new GoalFormationContext("hc", "wacky-manor",
                List.of("insight"), List.of(), List.of(), 5);
        GoalFormationProposal proposal = badStrategy.propose(context);
        assertThat(proposal.goals()).isEmpty();
    }

    @Test
    void id_returns_manor_llm() {
        assertThat(strategy.id()).isEqualTo("manor-llm");
    }
}
```

- [ ] **Step 2: Run tests to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorGoalFormationStrategyTest`
Expected: compilation failure — `ManorGoalFormationStrategy` doesn't exist

- [ ] **Step 3: Implement ManorGoalFormationStrategy**

Create `ManorGoalFormationStrategy.java`:

```java
package io.casehub.examples.manor.agent;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.RetrievedMemory;
import io.casehub.api.spi.routing.GoalFormationContext;
import io.casehub.api.spi.routing.GoalFormationProposal;
import io.casehub.api.spi.routing.GoalFormationStrategy;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.GoalPriority;
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
public class ManorGoalFormationStrategy implements GoalFormationStrategy {

    private static final Logger log = Logger.getLogger(ManorGoalFormationStrategy.class);
    private static final ObjectMapper JSON = new ObjectMapper();

    private static final String SYSTEM_PROMPT = """
        You are a goal discovery analyst for an autonomous agent. Given the agent's \
        recent reflection insights, current goals, and relevant memories, identify \
        new goals the agent should pursue. Only propose goals that represent genuinely \
        new objectives — not refinements of existing goals. Each goal must be specific, \
        actionable, and distinct from existing goals.
        Return ONLY a JSON object: {"goals": [{"name": "...", "description": "...", \
        "suggestedPriority": "PRIMARY"|"SECONDARY"|null, "formationReason": "..."}], \
        "rationale": "..."}""";

    private final AgentProvider agentProvider;

    @Inject
    public ManorGoalFormationStrategy(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    @Override
    public String id() { return "manor-llm"; }

    @Override
    public GoalFormationProposal propose(GoalFormationContext context) {
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
            log.warnf("Goal formation failed (non-fatal): %s", e.getMessage());
            return new GoalFormationProposal(List.of(), "");
        }
    }

    private String buildPrompt(GoalFormationContext context) {
        var sb = new StringBuilder();
        sb.append("Agent: ").append(context.agentId()).append("\n");
        sb.append("Remaining goal capacity: ").append(context.remainingCapacity()).append("\n");
        sb.append("\nCurrent goals:\n");
        for (AgentGoal goal : context.existingGoals()) {
            sb.append("- ").append(goal.name()).append(": ").append(goal.description())
              .append(" (priority: ").append(goal.priority()).append(")\n");
        }
        sb.append("\nRecent reflection insights:\n");
        for (String insight : context.reflectionInsights()) {
            sb.append("- ").append(insight).append("\n");
        }
        if (!context.recentMemories().isEmpty()) {
            sb.append("\nRelevant memories:\n");
            for (RetrievedMemory memory : context.recentMemories()) {
                sb.append("- ").append(memory.text()).append("\n");
            }
        }
        sb.append("\nRespond with JSON only.");
        return sb.toString();
    }

    private GoalFormationProposal parseResponse(String response) {
        try {
            JsonNode root = JSON.readTree(response);
            JsonNode goalsNode = root.get("goals");
            String rationale = root.has("rationale") ? root.get("rationale").asText() : "";
            List<GoalFormationProposal.ProposedGoal> goals = new ArrayList<>();
            if (goalsNode != null && goalsNode.isArray()) {
                for (JsonNode node : goalsNode) {
                    String name = node.get("name").asText();
                    String description = node.get("description").asText();
                    GoalPriority priority = null;
                    if (node.has("suggestedPriority") && !node.get("suggestedPriority").isNull()) {
                        try {
                            priority = GoalPriority.valueOf(node.get("suggestedPriority").asText());
                        } catch (IllegalArgumentException e) {
                            priority = GoalPriority.SECONDARY;
                        }
                    }
                    String reason = node.has("formationReason") ? node.get("formationReason").asText() : "";
                    goals.add(new GoalFormationProposal.ProposedGoal(name, description, priority, reason));
                }
            }
            return new GoalFormationProposal(goals, rationale);
        } catch (Exception e) {
            log.warnf("Failed to parse goal formation response: %s", e.getMessage());
            return new GoalFormationProposal(List.of(), "");
        }
    }
}
```

- [ ] **Step 4: Run tests to verify GREEN**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorGoalFormationStrategyTest`
Expected: all pass

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/.../ManorGoalFormationStrategy.java wacky-manor/src/test/java/.../ManorGoalFormationStrategyTest.java
git commit -m "feat(#43): ManorGoalFormationStrategy — LLM-driven goal formation

Implements GoalFormationStrategy SPI. Receives reflection insights,
existing goals, and memories. Calls LLM to propose new goals.
Parses GoalFormationProposal from JSON response.

Refs #43"
```

---

### Task 3: ManorGoalEvaluator

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalEvaluator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorGoalEvaluatorTest.java`

**Interfaces:**
- Consumes: `GoalFormationStrategy.propose(GoalFormationContext) → GoalFormationProposal`, `AgentRegistry.findById(String, String) → Optional<AgentDescriptor>`, `AgentRegistry.register(AgentDescriptor)`, `CaseMemoryStore.query(MemoryQuery) → List<Memory>`
- Produces: `ManorGoalEvaluator.evaluate(String agentId, List<String> insights, Map<String, GoalOutcomeCounts> goalOutcomes)` — reads descriptor, calls formation strategy, validates results, registers updated descriptor

- [ ] **Step 1: Write failing tests**

Create `ManorGoalEvaluatorTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.api.spi.routing.GoalFormationContext;
import io.casehub.api.spi.routing.GoalFormationProposal;
import io.casehub.api.spi.routing.GoalFormationStrategy;
import io.casehub.api.spi.routing.GoalRevisionStrategy;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.MemoryQuery;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class ManorGoalEvaluatorTest {

    private ManorGoalEvaluator evaluator;
    private AtomicReference<AgentDescriptor> registeredDescriptor;
    private GoalFormationProposal nextProposal;

    @BeforeEach
    void setUp() {
        registeredDescriptor = new AtomicReference<>();
        nextProposal = new GoalFormationProposal(List.of(
                new GoalFormationProposal.ProposedGoal("protect-tea",
                        "Prevent poisoning", GoalPriority.SECONDARY, "Danger observed")),
                "rationale");

        GoalFormationStrategy formationStrategy = ctx -> nextProposal;
        GoalRevisionStrategy revisionStrategy = null;

        var descriptor = AgentDescriptor.builder()
                .agentId("hc").name("Hooded Claw").tenancyId("wacky-manor")
                .slot("manor-character")
                .goals(List.of(new AgentGoal("eliminate", "Eliminate Penelope",
                        GoalPriority.PRIMARY, Visibility.PUBLIC, List.of())))
                .build();

        AgentRegistry registry = new AgentRegistry() {
            private AgentDescriptor current = descriptor;
            @Override public Optional<AgentDescriptor> findById(String id, String tenancy) {
                return Optional.of(current);
            }
            @Override public void register(AgentDescriptor d) {
                current = d;
                registeredDescriptor.set(d);
            }
            // other methods throw UnsupportedOperationException
        };

        CaseMemoryStore memoryStore = new CaseMemoryStore() {
            @Override public java.util.List<io.casehub.neocortex.memory.Memory> query(MemoryQuery q) {
                return List.of();
            }
            // other methods throw UnsupportedOperationException
        };

        evaluator = new ManorGoalEvaluator(formationStrategy, revisionStrategy,
                registry, memoryStore, "wacky-manor", 5, 2, 10);
    }

    @Test
    void evaluate_registers_new_goals_on_descriptor() {
        evaluator.evaluate("hc", List.of("danger observed"), Map.of());
        assertThat(registeredDescriptor.get()).isNotNull();
        assertThat(registeredDescriptor.get().goals()).hasSize(2);
        assertThat(registeredDescriptor.get().goals().stream()
                .map(AgentGoal::name).toList())
                .containsExactlyInAnyOrder("eliminate", "protect-tea");
    }

    @Test
    void evaluate_sets_private_visibility_on_formed_goals() {
        evaluator.evaluate("hc", List.of("insight"), Map.of());
        var formedGoal = registeredDescriptor.get().goals().stream()
                .filter(g -> g.name().equals("protect-tea")).findFirst().orElseThrow();
        assertThat(formedGoal.visibility()).isEqualTo(Visibility.PRIVATE);
    }

    @Test
    void evaluate_defaults_priority_to_secondary() {
        nextProposal = new GoalFormationProposal(List.of(
                new GoalFormationProposal.ProposedGoal("new-goal",
                        "A goal", null, "reason")), "");
        evaluator.evaluate("hc", List.of("insight"), Map.of());
        var formedGoal = registeredDescriptor.get().goals().stream()
                .filter(g -> g.name().equals("new-goal")).findFirst().orElseThrow();
        assertThat(formedGoal.priority()).isEqualTo(GoalPriority.SECONDARY);
    }

    @Test
    void evaluate_rejects_duplicate_goal_names() {
        nextProposal = new GoalFormationProposal(List.of(
                new GoalFormationProposal.ProposedGoal("eliminate",
                        "Duplicate", GoalPriority.SECONDARY, "reason")), "");
        evaluator.evaluate("hc", List.of("insight"), Map.of());
        assertThat(registeredDescriptor.get().goals()).hasSize(1);
    }

    @Test
    void evaluate_respects_capacity_cap() {
        var manyGoals = new ArrayList<GoalFormationProposal.ProposedGoal>();
        for (int i = 0; i < 15; i++) {
            manyGoals.add(new GoalFormationProposal.ProposedGoal(
                    "goal-" + i, "G" + i, GoalPriority.SECONDARY, "reason"));
        }
        nextProposal = new GoalFormationProposal(manyGoals, "");
        evaluator.evaluate("hc", List.of("insight"), Map.of());
        assertThat(registeredDescriptor.get().goals().size()).isLessThanOrEqualTo(10);
    }

    @Test
    void evaluate_respects_max_new_per_reflection() {
        var threeGoals = List.of(
                new GoalFormationProposal.ProposedGoal("a", "A", GoalPriority.SECONDARY, "r"),
                new GoalFormationProposal.ProposedGoal("b", "B", GoalPriority.SECONDARY, "r"),
                new GoalFormationProposal.ProposedGoal("c", "C", GoalPriority.SECONDARY, "r"));
        nextProposal = new GoalFormationProposal(threeGoals, "");
        evaluator.evaluate("hc", List.of("insight"), Map.of());
        // max-new-per-reflection = 2, so only 2 new goals added to the existing 1
        assertThat(registeredDescriptor.get().goals()).hasSize(3);
    }

    @Test
    void evaluate_skips_when_within_cooldown() {
        evaluator.evaluate("hc", List.of("first"), Map.of());
        registeredDescriptor.set(null);
        evaluator.evaluate("hc", List.of("second"), Map.of());
        assertThat(registeredDescriptor.get()).isNull();
    }

    @Test
    void evaluate_does_not_register_when_no_new_goals() {
        nextProposal = new GoalFormationProposal(List.of(), "nothing new");
        evaluator.evaluate("hc", List.of("insight"), Map.of());
        assertThat(registeredDescriptor.get()).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorGoalEvaluatorTest`
Expected: compilation failure — `ManorGoalEvaluator` doesn't exist

- [ ] **Step 3: Implement ManorGoalEvaluator**

Create `ManorGoalEvaluator.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.api.model.RetrievedMemory;
import io.casehub.api.spi.routing.GoalFormationContext;
import io.casehub.api.spi.routing.GoalFormationProposal;
import io.casehub.api.spi.routing.GoalFormationStrategy;
import io.casehub.api.spi.routing.GoalRevisionStrategy;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.GoalOutcomeCounts;
import io.casehub.eidos.api.GoalPriority;
import io.casehub.eidos.api.Visibility;
import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.MemoryOrder;
import io.casehub.neocortex.memory.MemoryQuery;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

public class ManorGoalEvaluator {

    private static final Logger log = Logger.getLogger(ManorGoalEvaluator.class);

    private final GoalFormationStrategy formationStrategy;
    private final GoalRevisionStrategy revisionStrategy;
    private final AgentRegistry agentRegistry;
    private final CaseMemoryStore memoryStore;
    private final String tenancyId;
    private final long cooldownMinutes;
    private final int maxNewPerReflection;
    private final int maxGoals;
    private final ConcurrentHashMap<String, Instant> lastFormationTime = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, ReentrantLock> agentLocks = new ConcurrentHashMap<>();

    public ManorGoalEvaluator(GoalFormationStrategy formationStrategy,
                               GoalRevisionStrategy revisionStrategy,
                               AgentRegistry agentRegistry,
                               CaseMemoryStore memoryStore,
                               String tenancyId,
                               long cooldownMinutes,
                               int maxNewPerReflection,
                               int maxGoals) {
        this.formationStrategy = formationStrategy;
        this.revisionStrategy = revisionStrategy;
        this.agentRegistry = agentRegistry;
        this.memoryStore = memoryStore;
        this.tenancyId = tenancyId;
        this.cooldownMinutes = cooldownMinutes;
        this.maxNewPerReflection = maxNewPerReflection;
        this.maxGoals = maxGoals;
    }

    public void evaluate(String agentId, List<String> insights,
                         Map<String, GoalOutcomeCounts> goalOutcomes) {
        var lock = agentLocks.computeIfAbsent(agentId, k -> new ReentrantLock());
        lock.lock();
        try {
            var last = lastFormationTime.get(agentId);
            if (last != null && Duration.between(last, Instant.now()).toMinutes() < cooldownMinutes) {
                return;
            }

            var descriptorOpt = agentRegistry.findById(agentId, tenancyId);
            if (descriptorOpt.isEmpty()) return;
            var descriptor = descriptorOpt.get();

            List<RetrievedMemory> memories = retrieveMemories(agentId);
            int remaining = maxGoals - descriptor.goals().size();
            if (remaining <= 0 && revisionStrategy == null) return;

            List<AgentGoal> finalGoals = new ArrayList<>(descriptor.goals());

            if (remaining > 0) {
                var context = new GoalFormationContext(agentId, tenancyId,
                        insights, descriptor.goals(), memories, remaining);
                var proposal = formationStrategy.propose(context);
                if (proposal != null && !proposal.goals().isEmpty()) {
                    var newGoals = validateAndConvert(proposal.goals(), descriptor, remaining);
                    finalGoals.addAll(newGoals);
                }
            }

            // Revision placeholder — blocked on engine#903
            // if (revisionStrategy != null) { ... }

            if (finalGoals.size() == descriptor.goals().size()) return;

            var updated = descriptor.toBuilder().goals(finalGoals).build();
            agentRegistry.register(updated);
            lastFormationTime.put(agentId, Instant.now());
        } catch (Exception e) {
            log.warnf(e, "Goal evaluation failed for agent %s", agentId);
        } finally {
            lock.unlock();
        }
    }

    private List<AgentGoal> validateAndConvert(
            List<GoalFormationProposal.ProposedGoal> proposed,
            AgentDescriptor descriptor, int remaining) {
        var existingNames = new HashSet<String>();
        for (AgentGoal g : descriptor.goals()) {
            existingNames.add(g.name());
        }
        List<AgentGoal> validated = new ArrayList<>();
        for (var p : proposed) {
            if (validated.size() >= maxNewPerReflection || validated.size() >= remaining) break;
            if (p.name() == null || p.name().isBlank()) continue;
            if (p.description() == null || p.description().isBlank()) continue;
            if (existingNames.contains(p.name())) continue;
            GoalPriority priority = p.suggestedPriority() != null
                    ? p.suggestedPriority() : GoalPriority.SECONDARY;
            try {
                var goal = new AgentGoal(p.name(), p.description(), priority,
                        Visibility.PRIVATE, List.of());
                validated.add(goal);
                existingNames.add(p.name());
            } catch (Exception e) {
                log.debugf("Rejected proposed goal %s: %s", p.name(), e.getMessage());
            }
        }
        return validated;
    }

    private List<RetrievedMemory> retrieveMemories(String agentId) {
        try {
            var memories = memoryStore.query(MemoryQuery.forEntity(agentId,
                    new MemoryDomain("manor"), tenancyId)
                    .withLimit(20).withOrder(MemoryOrder.SALIENCE));
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

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorGoalEvaluatorTest`
Expected: all pass

- [ ] **Step 5: Run full suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml`
Expected: all pass

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/.../ManorGoalEvaluator.java wacky-manor/src/test/java/.../ManorGoalEvaluatorTest.java
git commit -m "feat(#43): ManorGoalEvaluator — orchestrates goal formation

Reads descriptor from AgentRegistry, calls GoalFormationStrategy,
validates proposals (capacity, duplicates, defaults), registers
updated descriptor. Per-agent lock prevents concurrent clobbering.
Cooldown skips re-formation within configurable window.
Revision placeholder for engine#903.

Refs #43"
```

---

### Task 4: Expanded ManorReflectionSynthesizer and wiring

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizer.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizerTest.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`

**Interfaces:**
- Consumes: `ManorGoalEvaluator.evaluate(String, List<String>, Map<String, GoalOutcomeCounts>)` from Task 3, `AgentRegistry` (CDI), `ManorGoalFormationStrategy` (CDI)
- Produces: `ManorReflectionSynthesizer.ReflectionWithGoalAssessment(List<ReflectionEvent>, Map<String, GoalOutcomeCounts>)`, `ManorReflectionSynthesizer.synthesizeWithGoalAssessment(String, String, List<Memory>, List<AgentGoal>, int) → ReflectionWithGoalAssessment`

- [ ] **Step 1: Write failing test for synthesizeWithGoalAssessment**

Add to `ManorReflectionSynthesizerTest.java`:

```java
@Test
void synthesizeWithGoalAssessment_returns_insights_and_counts() {
    String response = """
        {"insights": [{"insight": "HC is suspicious", "importance": 0.8}],
         "goalAssessments": {"eliminate": {"success": 2, "failure": 1}}}
        """;
    AgentProvider mockProvider = config ->
            Multi.createFrom().item(new AgentEvent.TextDelta(response));
    var synth = new ManorReflectionSynthesizer(mockProvider);
    var goals = List.of(new AgentGoal("eliminate", "Eliminate Penelope",
            GoalPriority.PRIMARY, Visibility.PUBLIC, List.of()));
    var result = synth.synthesizeWithGoalAssessment("hc", "wacky-manor",
            List.of(/* mock memories */), goals, 1);
    assertThat(result.insights()).hasSize(1);
    assertThat(result.goalOutcomes()).containsKey("eliminate");
    assertThat(result.goalOutcomes().get("eliminate").successCount()).isEqualTo(2);
    assertThat(result.goalOutcomes().get("eliminate").failureCount()).isEqualTo(1);
}
```

- [ ] **Step 2: Run test to verify RED**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorReflectionSynthesizerTest#synthesizeWithGoalAssessment_returns_insights_and_counts`
Expected: compilation failure — method doesn't exist

- [ ] **Step 3: Implement synthesizeWithGoalAssessment**

Use `ide_insert_member` on `ManorReflectionSynthesizer` to add:

```java
public record ReflectionWithGoalAssessment(
    List<ReflectionEvent> insights,
    java.util.Map<String, io.casehub.eidos.api.GoalOutcomeCounts> goalOutcomes
) {}

public ReflectionWithGoalAssessment synthesizeWithGoalAssessment(
        String agentId, String tenantId,
        List<Memory> sources, List<io.casehub.eidos.api.AgentGoal> currentGoals, int targetLevel) {
    if (sources.isEmpty()) return new ReflectionWithGoalAssessment(List.of(), Map.of());
    try {
        var sb = new StringBuilder("Recent experiences:\n");
        for (var m : sources) {
            sb.append("- ").append(m.text()).append("\n");
        }
        if (!currentGoals.isEmpty()) {
            sb.append("\nCurrent goals:\n");
            for (var g : currentGoals) {
                sb.append("- ").append(g.name()).append(": ").append(g.description()).append("\n");
            }
        }
        var sourceIds = sources.stream().map(Memory::memoryId).toList();

        String assessmentPrompt = SYSTEM_PROMPT + "\n\nAlso assess goal progress. " +
            "Return JSON: {\"insights\": [{\"insight\": \"...\", \"importance\": 0.0-1.0}], " +
            "\"goalAssessments\": {\"goal-name\": {\"success\": N, \"failure\": M}}}";

        String response = agentProvider.invoke(
                AgentSessionConfig.of(assessmentPrompt, sb.toString()))
            .filter(e -> e instanceof AgentEvent.TextDelta)
            .map(e -> ((AgentEvent.TextDelta) e).text())
            .collect().with(java.util.stream.Collectors.joining())
            .await().atMost(java.time.Duration.ofSeconds(120));

        var root = JSON.readTree(response);
        var insights = parseInsights(root, agentId, tenantId, targetLevel, sourceIds);
        var goalOutcomes = parseGoalAssessments(root);

        return new ReflectionWithGoalAssessment(insights, goalOutcomes);
    } catch (Exception e) {
        log.warnf("%s: reflection with goal assessment failed (non-fatal): %s",
                agentId, e.getMessage());
        return new ReflectionWithGoalAssessment(List.of(), Map.of());
    }
}

private List<ReflectionEvent> parseInsights(com.fasterxml.jackson.databind.JsonNode root,
        String agentId, String tenantId, int targetLevel, List<String> sourceIds) {
    var insightsNode = root.get("insights");
    if (insightsNode == null || !insightsNode.isArray()) return List.of();
    return java.util.stream.StreamSupport.stream(insightsNode.spliterator(), false)
        .map(node -> new ReflectionEvent(agentId, tenantId, null,
                node.get("insight").asText(), targetLevel, sourceIds,
                node.has("importance") ? node.get("importance").asDouble() : 0.7, Map.of()))
        .toList();
}

private java.util.Map<String, io.casehub.eidos.api.GoalOutcomeCounts> parseGoalAssessments(
        com.fasterxml.jackson.databind.JsonNode root) {
    var assessmentsNode = root.get("goalAssessments");
    if (assessmentsNode == null || !assessmentsNode.isObject()) return Map.of();
    var result = new java.util.HashMap<String, io.casehub.eidos.api.GoalOutcomeCounts>();
    assessmentsNode.fields().forEachRemaining(entry -> {
        int success = entry.getValue().has("success") ? entry.getValue().get("success").asInt() : 0;
        int failure = entry.getValue().has("failure") ? entry.getValue().get("failure").asInt() : 0;
        result.put(entry.getKey(), new io.casehub.eidos.api.GoalOutcomeCounts(success, failure));
    });
    return result;
}
```

- [ ] **Step 4: Run test to verify GREEN**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml -Dtest=ManorReflectionSynthesizerTest`
Expected: all pass

- [ ] **Step 5: Wire AgentExperienceService to chain goal evaluation**

Change the `synthesizer` field type from `ReflectionSynthesizer` to `ManorReflectionSynthesizer`. Add `ManorGoalEvaluator goalEvaluator` field (nullable) and `AgentRegistry agentRegistry` field. Update constructors.

Modify `runReflection()` to branch: if `goalEvaluator != null`, call `synthesizeWithGoalAssessment()` and chain `goalEvaluator.evaluate()`. Otherwise use the existing `synthesize()` path.

(Exact code matches the spec's `AgentExperienceService` section.)

- [ ] **Step 6: Wire ScenarioOrchestrator to inject and construct evaluator**

Add `@Inject` fields for `ManorGoalFormationStrategy`. Add config properties for goal settings. In `runScenario()`, construct `ManorGoalEvaluator` with the injected strategy, `agentRegistry`, `caseMemoryStore`, and config values. Pass to `AgentExperienceService`.

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/slots/107/examples/wacky-manor/pom.xml`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git add -A wacky-manor/src/
git commit -m "feat(#43): wire goal lifecycle — synthesizer, evaluator, orchestrator

Expand ManorReflectionSynthesizer with goal outcome assessments.
Chain ManorGoalEvaluator after reflection in AgentExperienceService.
ScenarioOrchestrator injects ManorGoalFormationStrategy and constructs
evaluator with config. Goals now form from reflection insights and
register on AgentDescriptor.

Refs #43"
```

---

### Task 5: ManorGoalRevisionStrategy (blocked on engine#903)

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalRevisionStrategy.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorGoalRevisionStrategyTest.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalEvaluator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`

**Interfaces:**
- Consumes: `GoalRevisionStrategy` (engine-api), `GoalRevisionContext`, `GoalRevisionProposal`, `GoalRevisionAction` (engine#903), `AgentProvider`
- Produces: `ManorGoalRevisionStrategy` CDI bean implementing `GoalRevisionStrategy.revise(GoalRevisionContext) → GoalRevisionProposal`

**⚠️ This task is blocked on casehubio/engine#903 shipping. Skip if GoalRevisionAction is not yet available. Implement after `mvn install` of engine with #903.**

- [ ] **Step 1: Write failing tests**

Create `ManorGoalRevisionStrategyTest.java` — tests for REVISE, ABANDON, COMPLETE action parsing from LLM response.

- [ ] **Step 2: Implement ManorGoalRevisionStrategy**

Similar pattern to ManorGoalFormationStrategy — build prompt with goals + outcome counts, parse GoalRevisionProposal with GoalRevisionAction.

- [ ] **Step 3: Wire into ManorGoalEvaluator**

Uncomment the revision block in `evaluate()`. After formation, build `GoalRevisionContext` and call `revisionStrategy.revise()`. Process actions:
- `REVISE`: update goal description via `goal.toBuilder().description(revised).build()`
- `ABANDON`: remove from goals list, ingest "Abandoned goal: [name]" memory
- `COMPLETE`: remove from goals list, ingest "Completed goal: [name]" memory

- [ ] **Step 4: Wire ScenarioOrchestrator**

Add `@Inject ManorGoalRevisionStrategy`, pass to `ManorGoalEvaluator` constructor.

- [ ] **Step 5: Run full suite, commit**

```bash
git commit -m "feat(#43): ManorGoalRevisionStrategy — revision, abandonment, completion

Implements GoalRevisionStrategy SPI with GoalRevisionAction enum.
Handles REVISE (update description), ABANDON (remove + memory),
COMPLETE (remove + memory). Wired into ManorGoalEvaluator and
ScenarioOrchestrator.

Refs #43"
```
