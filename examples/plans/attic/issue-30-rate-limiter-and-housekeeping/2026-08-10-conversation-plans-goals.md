# Autonomous Conversation, Persistent Plans, and Dynamic Goals — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #38 — epic: Autonomous interaction chains
**Issue group:** #38

**Goal:** Characters talk TO each other, persist multi-turn plans, generate dynamic goals, and have focused exchanges — with capability-gated overhearing and behavioral assertions replacing hardcoded game completion.

**Architecture:** Seven dependency-ordered layers. Each layer adds one capability and its tests. Response format expands first (foundation), then persistent plans, dynamic goals, directed dialogue, PULL_ASIDE exchanges, and behavioral assertions. All patterns designed for platform extraction.

**Tech Stack:** Java 26, Quarkus, Jackson, JUnit 5, AssertJ, Eidos API, Neocortex Memory API, Qhorus Channels

## Global Constraints

- Pre-release: breaking changes cost nothing
- IntelliJ MCP for all .java edits — never Edit/Write on existing source files
- TDD: failing test before implementation, always
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/examples/wacky-manor/pom.xml` for test runs
- Commit after each task with `Refs #38`

---

### Task 1: Response Format Expansion

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentResponse.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentResponseTest.java`

**Interfaces:**
- Produces: `AgentResponse.talkTo()` → `String` (nullable), `AgentResponse.newGoals()` → `List<AgentResponse.GoalEntry>` (nullable), `AgentResponse.dropGoals()` → `List<String>` (nullable). Inner record `GoalEntry(String name, String description)`.

- [ ] **Step 1: Write failing tests for new fields**

Add to `AgentResponseTest`:

```java
@Test
void parse_includes_talkTo_field() {
    var json = """
        {"thinking":"plan","dialogue":"hello","talkTo":"peter-perfect","aside":null,"action":{"type":"WAIT"}}""";
    var response = AgentResponse.parse(json);
    assertThat(response.talkTo()).isEqualTo("peter-perfect");
}

@Test
void parse_talkTo_null_when_absent() {
    var json = """
        {"thinking":"plan","dialogue":"hello","action":{"type":"WAIT"}}""";
    var response = AgentResponse.parse(json);
    assertThat(response.talkTo()).isNull();
}

@Test
void parse_includes_newGoals() {
    var json = """
        {"thinking":"t","action":{"type":"WAIT"},"newGoals":[{"name":"protect-tea","description":"Stop the poison"}]}""";
    var response = AgentResponse.parse(json);
    assertThat(response.newGoals()).hasSize(1);
    assertThat(response.newGoals().get(0).name()).isEqualTo("protect-tea");
}

@Test
void parse_includes_dropGoals() {
    var json = """
        {"thinking":"t","action":{"type":"WAIT"},"dropGoals":["old-goal"]}""";
    var response = AgentResponse.parse(json);
    assertThat(response.dropGoals()).containsExactly("old-goal");
}

@Test
void parse_logs_malformed_json_and_returns_idle() {
    var response = AgentResponse.parse("not json at all {{{");
    assertThat(response.action().type()).isEqualTo(ActionType.WAIT);
}
```

- [ ] **Step 2: Run tests to verify RED**

Run: `mvn test -f .../wacky-manor/pom.xml -Dtest=AgentResponseTest`
Expected: compilation failure — `talkTo()`, `newGoals()`, `dropGoals()` don't exist

- [ ] **Step 3: Implement — expand AgentResponse record**

Use `ide_edit_member` to replace the `AgentResponse` record declaration. Add `talkTo`, `newGoals`, `dropGoals` fields. Add inner record `GoalEntry`. Update `idle()` to include null for new fields.

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record AgentResponse(
        String thinking,
        String dialogue,
        String talkTo,
        String aside,
        Action action,
        List<GoalEntry> newGoals,
        List<String> dropGoals) {

    @JsonIgnoreProperties(ignoreUnknown = true)
    public record GoalEntry(String name, String description) {}

    // ... parse(), idle(), extractJson() unchanged except:
    public static AgentResponse idle() {
        return new AgentResponse(null, null, null, null,
            new Action(ActionType.WAIT, null, null), null, null);
    }
}
```

- [ ] **Step 4: Run tests to verify GREEN**

Run: `mvn test -f .../wacky-manor/pom.xml -Dtest=AgentResponseTest`
Expected: all pass

- [ ] **Step 5: Run full suite to verify no regressions**

Run: `mvn test -f .../wacky-manor/pom.xml`
Expected: all pass (existing callers access `thinking()`, `dialogue()`, `aside()`, `action()` — new nullable fields don't break them)

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/.../AgentResponse.java wacky-manor/src/test/java/.../AgentResponseTest.java
git commit -m "feat(#38): expand AgentResponse — talkTo, newGoals, dropGoals fields

Refs #38"
```

---

### Task 2: Persistent Plans

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/CharacterState.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/model/CharacterStateTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `AgentResponse.thinking()` from Task 1
- Produces: `CharacterState.currentPlan()` → `String`, `CharacterState.setCurrentPlan(String)`, `ObservationBuilder` "Your Current Plan" section

- [ ] **Step 1: Write failing tests**

CharacterStateTest:
```java
@Test
void currentPlan_null_by_default() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    assertThat(state.currentPlan()).isNull();
}

@Test
void currentPlan_set_and_retrieved() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    state.setCurrentPlan("Step 1: get the poison");
    assertThat(state.currentPlan()).isEqualTo("Step 1: get the poison");
}
```

ObservationBuilderTest:
```java
@Test
void observation_includes_current_plan_when_set() {
    var character = world.character("hooded-claw");
    character.setCurrentPlan("Step 1: find the poison. Step 2: poison the tea.");
    var obs = ObservationBuilder.buildObservation(
            character, world, java.util.List.of(), emptyDrain,
            java.util.List.of(), character.capabilityTags());
    assertThat(obs).contains("== Your Current Plan ==");
    assertThat(obs).contains("Step 1: find the poison");
}

@Test
void observation_omits_plan_section_when_null() {
    var character = world.character("hooded-claw");
    var obs = ObservationBuilder.buildObservation(
            character, world, java.util.List.of(), emptyDrain,
            java.util.List.of(), character.capabilityTags());
    assertThat(obs).doesNotContain("Your Current Plan");
}
```

- [ ] **Step 2: Run tests — verify RED**

- [ ] **Step 3: Implement CharacterState.currentPlan**

Use `ide_insert_member` on CharacterState: add `private volatile String currentPlan;` field, `currentPlan()` getter, `setCurrentPlan(String)` setter.

- [ ] **Step 4: Implement ObservationBuilder plan section**

Use `ide_insert_member` to add `currentPlanSection(CharacterState)` method. Returns `ObservationSection.text("Your Current Plan", plan)` when non-null, null otherwise.

In `buildObservation` (6-arg), add before `goalsSection`:
```java
var planSection = currentPlanSection(character);
if (planSection != null) { sections.add(planSection); }
```

- [ ] **Step 5: Wire orchestrator — persist thinking as currentPlan**

In `ScenarioOrchestrator.runAutonomousTicks()`, after `c.setLastActionResult(...)`:
```java
if (response.thinking() != null) {
    String priorPlan = c.currentPlan();
    c.setCurrentPlan(response.thinking());
    // Memory ingestion on plan change (if experienceService available)
}
```

- [ ] **Step 6: Update RESPONSE_FORMAT_INSTRUCTION**

Add to the format instruction text: `"thinking": "your persistent strategic plan — shown to you next turn as 'Your Current Plan'. Write strategy, not stream-of-consciousness"`

- [ ] **Step 7: Run tests — verify GREEN, full suite passes**

- [ ] **Step 8: Commit**

---

### Task 3: Dynamic Goals Model and Lifecycle

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/model/DynamicGoal.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/CharacterState.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/model/CharacterStateTest.java`

**Interfaces:**
- Consumes: `AgentResponse.newGoals()`, `AgentResponse.dropGoals()` from Task 1
- Produces: `DynamicGoal(String name, String description, int creationTick)`, `CharacterState.dynamicGoals()` → `List<DynamicGoal>`, `CharacterState.addDynamicGoal(DynamicGoal)`, `CharacterState.dropDynamicGoal(String name)`, `CharacterState.dropAllDynamicGoals()`

- [ ] **Step 1: Write failing tests**

```java
@Test
void dynamicGoals_empty_by_default() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    assertThat(state.dynamicGoals()).isEmpty();
}

@Test
void addDynamicGoal_and_retrieve() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    state.addDynamicGoal(new DynamicGoal("protect-tea", "Stop the poison", 1));
    assertThat(state.dynamicGoals()).hasSize(1);
    assertThat(state.dynamicGoals().get(0).name()).isEqualTo("protect-tea");
}

@Test
void dropDynamicGoal_removes_by_normalized_name() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    state.addDynamicGoal(new DynamicGoal("protect-tea", "Stop the poison", 1));
    state.dropDynamicGoal("Protect-Tea");
    assertThat(state.dynamicGoals()).isEmpty();
}

@Test
void dropAllDynamicGoals_clears_all() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    state.addDynamicGoal(new DynamicGoal("goal-a", "A", 1));
    state.addDynamicGoal(new DynamicGoal("goal-b", "B", 2));
    state.dropAllDynamicGoals();
    assertThat(state.dynamicGoals()).isEmpty();
}

@Test
void addDynamicGoal_replaces_existing_with_same_name() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    state.addDynamicGoal(new DynamicGoal("protect-tea", "V1", 1));
    state.addDynamicGoal(new DynamicGoal("Protect-Tea", "V2", 2));
    assertThat(state.dynamicGoals()).hasSize(1);
    assertThat(state.dynamicGoals().get(0).description()).isEqualTo("V2");
}

@Test
void dynamicGoals_capped_evicts_oldest() {
    var state = new CharacterState("test", "Test", "room", 0.5, List.of());
    for (int i = 0; i < 6; i++) {
        state.addDynamicGoal(new DynamicGoal("goal-" + i, "G" + i, i));
    }
    assertThat(state.dynamicGoals()).hasSize(5);
    assertThat(state.dynamicGoals().stream().map(DynamicGoal::name).toList())
            .doesNotContain("goal-0");
}
```

- [ ] **Step 2: Run tests — verify RED**

- [ ] **Step 3: Create DynamicGoal record**

Use `ide_create_file`:
```java
package io.casehub.examples.manor.model;

public record DynamicGoal(String name, String description, int creationTick) {
    public DynamicGoal {
        name = name.strip().toLowerCase();
    }
}
```

- [ ] **Step 4: Implement CharacterState goal storage**

Add to CharacterState via `ide_insert_member`:
- `private final CopyOnWriteArrayList<DynamicGoal> dynamicGoals = new CopyOnWriteArrayList<>();`
- `public List<DynamicGoal> dynamicGoals()` — returns `List.copyOf(dynamicGoals)`
- `public void addDynamicGoal(DynamicGoal goal)` — removes existing with same normalized name, adds new, evicts oldest if over cap (5)
- `public void dropDynamicGoal(String name)` — removes by normalized name
- `public void dropAllDynamicGoals()` — clears all

- [ ] **Step 5: Wire orchestrator — process newGoals/dropGoals**

In `runAutonomousTicks`, after plan persistence:
```java
if (response.dropGoals() != null) {
    if (response.dropGoals().contains("*")) {
        c.dropAllDynamicGoals();
    } else {
        response.dropGoals().forEach(c::dropDynamicGoal);
    }
}
if (response.newGoals() != null) {
    response.newGoals().forEach(g ->
        c.addDynamicGoal(new DynamicGoal(g.name(), g.description(), currentTick)));
}
```

- [ ] **Step 6: Run tests — verify GREEN, full suite passes**

- [ ] **Step 7: Commit**

---

### Task 4: Goals Rendering

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `CharacterState.dynamicGoals()` from Task 3, `List<AgentGoal>` from Eidos
- Produces: Modified `goalsSection()` rendering both tiers with `[PRIMARY]`/`[SECONDARY]`/`[SITUATIONAL]` prefixes

- [ ] **Step 1: Write failing test**

```java
@Test
void goals_section_renders_dynamic_goals_with_situational_prefix() {
    var character = world.character("penelope-pitstop");
    character.addDynamicGoal(new io.casehub.examples.manor.model.DynamicGoal(
            "protect-tea", "Stop Sneekly from poisoning the tea", 1));
    var goals = java.util.List.of(
            new io.casehub.eidos.api.AgentGoal("find-diamond", "Find the Doily Diamond",
                    io.casehub.eidos.api.GoalPriority.PRIMARY, io.casehub.eidos.api.Visibility.PUBLIC, java.util.List.of()));
    var obs = ObservationBuilder.buildObservation(
            character, world, goals, emptyDrain, java.util.List.of(), java.util.Set.of());
    assertThat(obs).contains("[PRIMARY] Find the Doily Diamond");
    assertThat(obs).contains("[SITUATIONAL] Stop Sneekly from poisoning the tea");
}
```

- [ ] **Step 2: Run test — verify RED**

- [ ] **Step 3: Modify goalsSection to accept and render both tiers**

Use `ide_replace_member` on `goalsSection`. The method now reads dynamic goals from the character parameter (already passed to `buildObservation`). Render identity goals first (sorted by priority), then situational goals (sorted by creation tick, newest first) with `[SITUATIONAL]` prefix.

Signature change: `goalsSection(List<AgentGoal> goals, CharacterState character)` — add character parameter to access `dynamicGoals()`.

Update the call site in `buildObservation` to pass `character`.

- [ ] **Step 4: Update RESPONSE_FORMAT_INSTRUCTION**

Add `newGoals` and `dropGoals` to the format instruction JSON template with descriptions. Add `STEAL` to available actions if not already included from prior work.

- [ ] **Step 5: Run tests — verify GREEN, full suite passes**

- [ ] **Step 6: Commit**

---

### Task 5: Directed Dialogue

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/ManorEvent.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/NarrativeEventBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationService.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/model/ManorEventTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarrativeEventBuilderTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationServiceTest.java`

**Interfaces:**
- Consumes: `AgentResponse.talkTo()` from Task 1, `ManorEvent.detailedDescription` from prior work
- Produces: `ManorEvent.dialogueTarget()` → `String`, `NarrativeEventBuilder.describeDirectedDialogue(String speaker, String target, String dialogue)` → `NarrativeDescription`

- [ ] **Step 1: Write failing tests**

ManorEventTest:
```java
@Test
void dialogueTarget_null_by_default() {
    var event = new ManorEvent(Instant.now(), "dialogue", "hc", "hall", "Hello");
    assertThat(event.dialogueTarget()).isNull();
}
```

NarrativeEventBuilderTest:
```java
@Test
void directed_dialogue_public_is_vague() {
    var desc = NarrativeEventBuilder.describeDirectedDialogue(
            "The Hooded Claw (as Sneekly)", "peter-perfect",
            "The cellar is perfectly safe.");
    assertThat(desc.publicText()).contains("spoke quietly with");
    assertThat(desc.publicText()).contains("peter-perfect");
}

@Test
void directed_dialogue_detailed_includes_content() {
    var desc = NarrativeEventBuilder.describeDirectedDialogue(
            "The Hooded Claw (as Sneekly)", "peter-perfect",
            "The cellar is perfectly safe.");
    assertThat(desc.detailedText()).contains("cellar is perfectly safe");
}
```

ObservationServiceTest:
```java
@Test
void directed_dialogue_visible_to_target_with_full_detail() {
    var world = createWorld();
    var service = createService();
    service.init(world);

    var event = new ManorEvent(Instant.now(), "dialogue", "hooded-claw", "entrance-hall",
            "Sneekly spoke quietly with Peter.", null, null, null, null,
            "Sneekly, speaking to Peter: 'The cellar is safe.'", false);
    // dialogueTarget constructor needed
    service.publishEvent(event);

    var drain = service.drain("peter-perfect", System.currentTimeMillis());
    assertThat(drain.currentPartition().eventCount()).isEqualTo(1);
}
```

- [ ] **Step 2: Run tests — verify RED**

- [ ] **Step 3: Add dialogueTarget to ManorEvent**

Use `ide_replace_text_in_file` to add `String dialogueTarget` field to ManorEvent record and update constructors. Existing constructors pass `null` for dialogueTarget.

- [ ] **Step 4: Add describeDirectedDialogue to NarrativeEventBuilder**

Use `ide_insert_member`:
```java
public static NarrativeDescription describeDirectedDialogue(String speakerName, String targetId, String dialogue) {
    String publicText = speakerName + " spoke quietly with " + targetId + ".";
    String detailedText = speakerName + ", speaking to " + targetId + ": '" + dialogue + "'";
    return new NarrativeDescription(publicText, detailedText);
}
```

- [ ] **Step 5: Wire orchestrator — talkTo validation and directed dialogue events**

In `runAutonomousTicks`, during dialogue processing: if `response.talkTo()` is set, validate target is in same room. If valid, create ManorEvent with `dialogueTarget` set and use `describeDirectedDialogue()`. If invalid, null out talkTo and use broadcast.

- [ ] **Step 6: Run tests — verify GREEN, full suite passes**

- [ ] **Step 7: Commit**

---

### Task 6: PULL_ASIDE Action and Exchange Runner

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/ActionType.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ExchangeRunnerTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `AgentInvocationService`, `ObservationBuilder`, `ManorEventDispatcher`, `CharacterState.currentPlan()`
- Produces: `ExchangeRunner.run(CharacterState initiator, CharacterState target, String openingDialogue, WorldState world, AgentInvocationService invocationService, ManorEventDispatcher dispatcher, Function<String, String> systemPromptRenderer)` → `List<ManorEvent>`, `ObservationBuilder.buildExchangeObservation(CharacterState character, String otherDialogue, WorldState world)` → `String`

- [ ] **Step 1: Write failing tests**

ExchangeRunnerTest — test with a mock/stub InvocationService:
```java
@Test
void exchange_runs_capped_round_trips() { ... }

@Test
void exchange_terminates_when_participant_produces_wait() { ... }

@Test
void exchange_terminates_when_participant_produces_action() { ... }

@Test
void exchange_publishes_directed_dialogue_events() { ... }

@Test
void exchange_updates_both_participants_plans() { ... }
```

ObservationBuilderTest:
```java
@Test
void exchange_observation_includes_room_and_characters_only() {
    var obs = ObservationBuilder.buildExchangeObservation(
            world.character("hooded-claw"), "What do you need, boss?", world);
    assertThat(obs).contains("Entrance Hall");
    assertThat(obs).doesNotContain("== Visible Objects ==");
    assertThat(obs).doesNotContain("== Exits ==");
    assertThat(obs).contains("What do you need, boss?");
}
```

- [ ] **Step 2: Run tests — verify RED**

- [ ] **Step 3: Add PULL_ASIDE to ActionType**

```java
MOVE, INTERACT, TAKE, GIVE, USE, LOOK, WAIT, STEAL, PULL_ASIDE
```

Update all switch statements in NarrativeEventBuilder and ActionResolver to handle PULL_ASIDE (returns null narrative / not resolved by ActionResolver — orchestrator handles it directly).

- [ ] **Step 4: Implement buildExchangeObservation**

Use `ide_insert_member` on ObservationBuilder. Minimal observation: room name, characters present, other's dialogue, own current plan. No objects, exits, inventory, affordances.

- [ ] **Step 5: Implement ExchangeRunner**

Use `ide_create_file`. Core logic: alternating LLM calls, cap at N round-trips, termination on WAIT/action, timeout. Each turn: build exchange observation, call LLM, parse response, persist thinking, publish directed dialogue event.

Exchange format instruction (simplified JSON: thinking, dialogue, action only).

- [ ] **Step 6: Wire into orchestrator tick pipeline**

Add PULL_ASIDE resolution phase between response gathering and dialogue/action processing. Process in iteration order, mark both participants as suppressed, run exchanges sequentially. Add PULL_ASIDE to ACTION_DESCRIPTORS.

- [ ] **Step 7: Run tests — verify GREEN, full suite passes**

- [ ] **Step 8: Commit**

---

### Task 7: Behavioral Assertions and Completion

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/model/TickSnapshot.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AssertionRegistry.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/CompletionReason.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AssertionRegistryTest.java`

**Interfaces:**
- Consumes: `AgentResponse` (all fields), `WorldState`, `List<ManorEvent>`
- Produces: `TickSnapshot(int tick, Map<String, AgentResponse> responses, List<ManorEvent> events, WorldState worldView)`, `AssertionRegistry.register(String id, Predicate<TickSnapshot>)`, `AssertionRegistry.evaluate(TickSnapshot)`, `AssertionRegistry.wasSatisfied(String id)` → `boolean`, `AssertionRegistry.firstSatisfiedTick(String id)` → `int`

- [ ] **Step 1: Write failing tests**

```java
@Test
void assertion_not_satisfied_initially() {
    var registry = new AssertionRegistry();
    registry.register("test", snap -> false);
    assertThat(registry.wasSatisfied("test")).isFalse();
}

@Test
void assertion_satisfied_after_matching_tick() {
    var registry = new AssertionRegistry();
    registry.register("test", snap -> snap.tick() == 3);
    registry.evaluate(new TickSnapshot(1, Map.of(), List.of(), null));
    registry.evaluate(new TickSnapshot(3, Map.of(), List.of(), null));
    assertThat(registry.wasSatisfied("test")).isTrue();
    assertThat(registry.firstSatisfiedTick("test")).isEqualTo(3);
}

@Test
void unknown_assertion_returns_false() {
    var registry = new AssertionRegistry();
    assertThat(registry.wasSatisfied("nonexistent")).isFalse();
}
```

- [ ] **Step 2: Run tests — verify RED**

- [ ] **Step 3: Create TickSnapshot record**

```java
package io.casehub.examples.manor.model;

public record TickSnapshot(int tick, java.util.Map<String, io.casehub.examples.manor.agent.AgentResponse> responses,
                           java.util.List<ManorEvent> events, io.casehub.examples.manor.engine.WorldState worldView) {}
```

- [ ] **Step 4: Create AssertionRegistry**

```java
@ApplicationScoped
public class AssertionRegistry {
    private final Map<String, Predicate<TickSnapshot>> predicates = new LinkedHashMap<>();
    private final Map<String, List<Boolean>> history = new LinkedHashMap<>();

    public void register(String id, Predicate<TickSnapshot> predicate) { ... }
    public void evaluate(TickSnapshot snapshot) { ... }
    public boolean wasSatisfied(String id) { ... }
    public int firstSatisfiedTick(String id) { ... }
}
```

- [ ] **Step 5: Update CompletionReason**

Replace `POISONED, TURN_LIMIT` with `DAWN`. Update all references.

- [ ] **Step 6: Wire into orchestrator**

Remove hardcoded `hasEffect("tea-service", "rat-poison")` check. Replace with:
```java
if (tick >= maxTurns) {
    world.setScenarioComplete(CompletionReason.DAWN);
}
```

Build `TickSnapshot` each tick and call `assertionRegistry.evaluate(snapshot)`.

Register example assertions at scenario start (poison-related goal, mob intervention, etc.).

- [ ] **Step 7: Run tests — verify GREEN, full suite passes**

- [ ] **Step 8: Commit**

---

### Task 8: LLM Eval Tests

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/voice/CapabilityGatedObservationTest.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/voice/ConversationTest.java`

**Interfaces:**
- Consumes: All prior tasks

- [ ] **Step 1: Write directed dialogue eval test**

```java
@QuarkusTest @Tag("llm-eval")
class ConversationTest {
    // Test: HC talks to Peter with deceptive intent
    // Verify: response includes talkTo field targeting a character
    // Verify: dialogue content is in-character
}
```

- [ ] **Step 2: Write dynamic goal generation eval test**

Give a character a danger scenario. Verify the response includes `newGoals` with a protective/responsive goal.

- [ ] **Step 3: Write persistent plan eval test**

Give a character a plan section in their observation. Verify the response's thinking builds on the prior plan rather than starting fresh.

- [ ] **Step 4: Run eval tests**

Run: `mvn test -f .../wacky-manor/pom.xml -Pllm-eval -Dtest=ConversationTest`

- [ ] **Step 5: Commit**

```bash
git commit -m "test(#38): LLM eval tests for conversation, plans, and dynamic goals

Refs #38"
```
