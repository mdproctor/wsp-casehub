# Extract Observation SPI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #48 — Extract Observation SPI — decouple cognitive sections from world-specific rendering
**Issue group:** #48

**Goal:** Refactor ObservationBuilder to accept WorldObservationProvider, extract cognitive section formatters to blocks utility class, and implement manor-specific world providers.

**Architecture:** Three-way split: world perception sections move into ManorWorldObservationProvider (implements WorldObservationProvider from blocks#127); platform-type cognitive formatters move to CognitiveObservationSections in blocks; CharacterState-dependent methods stay in manor's ObservationBuilder. Section ordering regroups from interleaved to perception-first layout.

**Tech Stack:** Java 26, Quarkus, casehub-blocks, casehub-eidos-api, casehub-neocortex-memory-api

## Global Constraints

- blocks SNAPSHOT must be published before manor work begins
- No new dependencies in blocks (eidos-api and neocortex-memory-api already present)
- WorldObservationProvider interface from blocks#127 is `@FunctionalInterface` with `List<ObservationSection> worldSections()`
- Section headers must not change (LLM prompts reference them by name)
- All methods move verbatim — no behavioural changes except section ordering

---

## Batch 1: CognitiveObservationSections in blocks (cross-repo)

### Task 1: Create CognitiveObservationSections utility class with tests

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/CognitiveObservationSections.java`
- Test: `blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/CognitiveObservationSectionsTest.java`

**Interfaces:**
- Consumes: `ObservationSection` (blocks), `AgentGoal` (eidos-api), `Memory` (neocortex-memory-api), `PartitionedDrain` (blocks)
- Produces:
  - `static ObservationSection goalsSection(List<AgentGoal> goals)`
  - `static ObservationSection recentActivitySection(PartitionedDrain<String> drain)`
  - `static ObservationSection pastExperienceSection(List<Memory> memories)`
  - `static ObservationSection insightsSection(List<Memory> reflections)`
  - `static ObservationSection relationshipNotesSection(String characterName, List<Memory> memories)`

- [ ] **Step 1: Write failing tests for goalsSection**

```java
@Test
void goalsSection_renders_sorted_by_priority_then_name() {
    var goals = List.of(
            new AgentGoal("low-goal", "Low priority goal", AgentGoal.Priority.LOW, "agent-1"),
            new AgentGoal("high-goal", "High priority goal", AgentGoal.Priority.HIGH, "agent-1"),
            new AgentGoal("a-high-goal", "A high priority goal", AgentGoal.Priority.HIGH, "agent-1"));
    var section = CognitiveObservationSections.goalsSection(goals);
    assertThat(section).isInstanceOf(ObservationSection.ItemList.class);
    var items = ((ObservationSection.ItemList) section).items();
    assertThat(items.get(0)).contains("HIGH").contains("A high priority goal");
    assertThat(items.get(1)).contains("HIGH").contains("High priority goal");
    assertThat(items.get(2)).contains("LOW").contains("Low priority goal");
}

@Test
void goalsSection_empty_goals_shows_no_specific_goals() {
    var section = CognitiveObservationSections.goalsSection(List.of());
    assertThat(section).isInstanceOf(ObservationSection.ItemList.class);
    assertThat(((ObservationSection.ItemList) section).emptyMessage()).isEqualTo("No specific goals.");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=CognitiveObservationSectionsTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement goalsSection**

Use `ide_create_file` to create `CognitiveObservationSections.java`:

```java
package io.casehub.blocks.summarisation.observation.affordance;

import io.casehub.eidos.api.AgentGoal;
import io.casehub.neocortex.memory.Memory;

import java.util.ArrayList;
import java.util.Comparator;
import java.util.List;

public final class CognitiveObservationSections {

    private CognitiveObservationSections() {}

    public static ObservationSection goalsSection(List<AgentGoal> goals) {
        var items = new ArrayList<String>();
        goals.stream()
             .sorted(Comparator.comparing(AgentGoal::priority)
                                .thenComparing(AgentGoal::name))
             .map(g -> "[" + g.priority().name() + "] " + g.description())
             .forEach(items::add);
        if (items.isEmpty()) {
            return ObservationSection.items("Your Goals", "No specific goals.", List.of());
        }
        return ObservationSection.items("Your Goals", null, items);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=CognitiveObservationSectionsTest`
Expected: PASS

- [ ] **Step 5: Write failing tests for recentActivitySection**

```java
@Test
void recentActivitySection_renders_drain_text() {
    var result = new ObservationResult("Penelope entered the room.\nDastardly laughed.", List.of(), 2, 5000L, ObservationTier.VERBATIM);
    var drain = new PartitionedDrain<String>(result, Map.of());
    var section = CognitiveObservationSections.recentActivitySection(drain);
    assertThat(section).isInstanceOf(ObservationSection.TextBlock.class);
    assertThat(((ObservationSection.TextBlock) section).content()).contains("Penelope entered");
}

@Test
void recentActivitySection_empty_drain_shows_quiet_room() {
    var drain = new PartitionedDrain<String>(ObservationResult.empty(0), Map.of());
    var section = CognitiveObservationSections.recentActivitySection(drain);
    assertThat(section).isInstanceOf(ObservationSection.ItemList.class);
    assertThat(((ObservationSection.ItemList) section).emptyMessage()).isEqualTo("The room is quiet.");
}
```

- [ ] **Step 6: Implement recentActivitySection**

Use `ide_insert_member` to add to `CognitiveObservationSections`:

```java
public static ObservationSection recentActivitySection(PartitionedDrain<String> drain) {
    String text = drain.currentPartition().renderedText();
    if (text == null || text.isBlank()) {
        return ObservationSection.items("Recent Activity", "The room is quiet.", List.of());
    }
    return ObservationSection.text("Recent Activity", text.strip());
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=CognitiveObservationSectionsTest`
Expected: PASS

- [ ] **Step 8: Write failing tests for memory-based sections**

```java
@Test
void pastExperienceSection_renders_memory_texts() {
    var memories = List.of(
            memory("Saw Dastardly near the kitchen"),
            memory("Heard a suspicious noise"));
    var section = CognitiveObservationSections.pastExperienceSection(memories);
    assertThat(section).isInstanceOf(ObservationSection.ItemList.class);
    var items = ((ObservationSection.ItemList) section).items();
    assertThat(items).containsExactly("Saw Dastardly near the kitchen", "Heard a suspicious noise");
}

@Test
void insightsSection_renders_reflection_texts() {
    var reflections = List.of(memory("Dastardly is planning something"));
    var section = CognitiveObservationSections.insightsSection(reflections);
    assertThat(((ObservationSection.ItemList) section).header()).isEqualTo("Insights");
    assertThat(((ObservationSection.ItemList) section).items()).containsExactly("Dastardly is planning something");
}

@Test
void relationshipNotesSection_renders_with_recall_prefix() {
    var memories = List.of(memory("Helped me escape the trap"));
    var section = CognitiveObservationSections.relationshipNotesSection("Penelope", memories);
    assertThat(((ObservationSection.ItemList) section).header()).isEqualTo("About Penelope");
    assertThat(((ObservationSection.ItemList) section).items()).containsExactly("You recall: Helped me escape the trap");
}
```

The `memory(String)` helper creates a `Memory` record with the text and sensible defaults. Check the `Memory` record's constructor to determine required fields and write the helper accordingly.

- [ ] **Step 9: Implement pastExperienceSection, insightsSection, relationshipNotesSection**

Use `ide_insert_member` to add each method:

```java
public static ObservationSection pastExperienceSection(List<Memory> memories) {
    var items = memories.stream()
                        .map(Memory::text)
                        .filter(t -> t != null && !t.isBlank())
                        .toList();
    return ObservationSection.items("Past Experience", null, items);
}

public static ObservationSection insightsSection(List<Memory> reflections) {
    var items = reflections.stream()
                           .map(Memory::text)
                           .filter(t -> t != null && !t.isBlank())
                           .toList();
    return ObservationSection.items("Insights", null, items);
}

public static ObservationSection relationshipNotesSection(String characterName, List<Memory> memories) {
    var items = memories.stream()
                        .map(m -> "You recall: " + m.text())
                        .toList();
    return ObservationSection.items("About " + characterName, null, items);
}
```

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=CognitiveObservationSectionsTest`
Expected: PASS (all tests)

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/CognitiveObservationSections.java blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/CognitiveObservationSectionsTest.java
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#127): add CognitiveObservationSections utility for reusable agent observation formatting"
```

- [ ] **Step 12: Install blocks SNAPSHOT locally**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl blocks -s slot-settings.xml -DskipTests`

This publishes the SNAPSHOT to the local Maven repository so wacky-manor can pick it up.

---

## Batch 2: Manor — Providers, Builder Refactor, Caller Updates

### Task 2: Create ManorWorldObservationProvider and ManorExchangeObservationProvider

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorWorldObservationProvider.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorExchangeObservationProvider.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorWorldObservationProviderTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorExchangeObservationProviderTest.java`

**Interfaces:**
- Consumes: `WorldObservationProvider` (blocks), `ObservationSection` (blocks), `WorldState`, `CharacterState`, `Room`, `GameObject`, `ManorEvent`, `PartitionedDrain` (blocks)
- Produces:
  - `ManorWorldObservationProvider(CharacterState, WorldState, PartitionedDrain<String>, Set<String>)` implementing `WorldObservationProvider`
  - `ManorExchangeObservationProvider(CharacterState, String, WorldState)` implementing `WorldObservationProvider`

- [ ] **Step 1: Write failing test for ManorWorldObservationProvider**

```java
@Test
void worldSections_includes_location_exits_objects_characters() {
    var character = world.character("penelope-pitstop");
    var provider = new ManorWorldObservationProvider(character, world, emptyDrain, Set.of());
    var sections = provider.worldSections();
    var rendered = new AffordanceRenderer().renderObservation(sections);
    assertThat(rendered).contains("== Current Location ==");
    assertThat(rendered).contains("Entrance Hall");
    assertThat(rendered).contains("== Exits ==");
    assertThat(rendered).contains("== Visible Objects ==");
    assertThat(rendered).contains("== Characters Present ==");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorWorldObservationProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class not found

- [ ] **Step 3: Implement ManorWorldObservationProvider**

Use `ide_create_file` to create the class. Move the 7 world-specific methods verbatim from `ObservationBuilder` plus `toObservableEntity` and `formatElapsed` helpers:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.RememberedPartition;
import io.casehub.blocks.summarisation.observation.PartitionedDrain;
import io.casehub.blocks.summarisation.observation.affordance.Affordance;
import io.casehub.blocks.summarisation.observation.affordance.ObservableEntity;
import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import io.casehub.blocks.summarisation.observation.affordance.WorldObservationProvider;
import io.casehub.examples.manor.engine.WorldState;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.model.GameObject;
import io.casehub.examples.manor.model.Room;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Set;

public class ManorWorldObservationProvider implements WorldObservationProvider {

    private final CharacterState character;
    private final WorldState world;
    private final PartitionedDrain<String> drain;
    private final Set<String> observerTags;

    public ManorWorldObservationProvider(CharacterState character, WorldState world,
                                         PartitionedDrain<String> drain,
                                         Set<String> observerTags) {
        this.character = character;
        this.world = world;
        this.drain = drain;
        this.observerTags = observerTags;
    }

    @Override
    public List<ObservationSection> worldSections() {
        var sections = new ArrayList<ObservationSection>();
        Room room = world.room(character.currentRoom());

        sections.add(locationSection(room));
        sections.add(exitsSection(room, world));
        sections.add(objectsSection(character, world));
        sections.add(charactersSection(character, world));
        if (!drain.rememberedPartitions().isEmpty()) {
            sections.add(rememberedSection(drain, world));
        }
        if (observerTags.contains("perception")) {
            var keen = keenObservationsSection(character, world);
            if (keen != null) { sections.add(keen); }
        } else {
            var directed = directedDialogueSection(character, world);
            if (directed != null) { sections.add(directed); }
        }
        return sections;
    }

    // Move verbatim from ObservationBuilder:
    // locationSection, exitsSection, objectsSection, toObservableEntity,
    // charactersSection, keenObservationsSection, directedDialogueSection,
    // rememberedSection, formatElapsed
}
```

The private methods (`locationSection`, `exitsSection`, `objectsSection`, `toObservableEntity`, `charactersSection`, `keenObservationsSection`, `directedDialogueSection`, `rememberedSection`, `formatElapsed`) are copied verbatim from `ObservationBuilder.java` lines 129–301.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorWorldObservationProviderTest`
Expected: PASS

- [ ] **Step 5: Write additional tests for edge cases**

```java
@Test
void worldSections_perceptive_observer_includes_keen_observations() {
    var character = world.character("penelope-pitstop");
    character.addCapabilityTag("perception");
    // Add a detailed-description event to the world
    var provider = new ManorWorldObservationProvider(character, world, emptyDrain, Set.of("perception"));
    var sections = provider.worldSections();
    var rendered = new AffordanceRenderer().renderObservation(sections);
    assertThat(rendered).doesNotContain("== Directed to You ==");
}

@Test
void worldSections_non_perceptive_observer_gets_directed_dialogue() {
    var character = world.character("penelope-pitstop");
    var provider = new ManorWorldObservationProvider(character, world, emptyDrain, Set.of());
    var sections = provider.worldSections();
    // Directed section only appears if matching events exist — verify no keen section
    var rendered = new AffordanceRenderer().renderObservation(sections);
    assertThat(rendered).doesNotContain("== Keen Observations ==");
}
```

- [ ] **Step 6: Write failing test for ManorExchangeObservationProvider**

```java
@Test
void exchangeProvider_includes_location_others_dialogue() {
    var character = world.character("penelope-pitstop");
    var provider = new ManorExchangeObservationProvider(character, "Hehehehe!", world);
    var sections = provider.worldSections();
    var rendered = new AffordanceRenderer().renderObservation(sections);
    assertThat(rendered).contains("== Location ==");
    assertThat(rendered).contains("Entrance Hall");
    assertThat(rendered).contains("== They said ==");
    assertThat(rendered).contains("Hehehehe!");
}
```

- [ ] **Step 7: Implement ManorExchangeObservationProvider**

Use `ide_create_file`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import io.casehub.blocks.summarisation.observation.affordance.WorldObservationProvider;
import io.casehub.examples.manor.engine.WorldState;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.model.Room;

import java.util.ArrayList;
import java.util.List;

public class ManorExchangeObservationProvider implements WorldObservationProvider {

    private final CharacterState character;
    private final String otherDialogue;
    private final WorldState world;

    public ManorExchangeObservationProvider(CharacterState character, String otherDialogue, WorldState world) {
        this.character = character;
        this.otherDialogue = otherDialogue;
        this.world = world;
    }

    @Override
    public List<ObservationSection> worldSections() {
        var sections = new ArrayList<ObservationSection>();
        Room room = world.room(character.currentRoom());
        sections.add(ObservationSection.text("Location", room.name()));

        List<CharacterState> others = world.charactersInRoom(character.currentRoom()).stream()
                .filter(c -> !c.agentId().equals(character.agentId())).toList();
        if (!others.isEmpty()) {
            sections.add(ObservationSection.items("Others Present", null,
                    others.stream().map(CharacterState::name).toList()));
        }
        sections.add(ObservationSection.text("They said", otherDialogue));
        return sections;
    }
}
```

- [ ] **Step 8: Run all provider tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest="ManorWorldObservationProviderTest,ManorExchangeObservationProviderTest"`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorWorldObservationProvider.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorExchangeObservationProvider.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorWorldObservationProviderTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorExchangeObservationProviderTest.java
git commit -m "feat(#48): add ManorWorldObservationProvider and ManorExchangeObservationProvider

Refs #48"
```

### Task 3: Refactor ObservationBuilder, update callers and tests

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java:69`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:292`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java:64`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/engine/LiveScenarioTest.java:144`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java:63`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `ManorWorldObservationProvider` (Task 2), `ManorExchangeObservationProvider` (Task 2), `CognitiveObservationSections` (Task 1), `WorldObservationProvider` (blocks#127)
- Produces: Refactored `ObservationBuilder.buildObservation(WorldObservationProvider, CharacterState, List<AgentGoal>, PartitionedDrain<String>, List<Memory>, List<Memory>, Map<String, List<Memory>>)` — single public entry point

- [ ] **Step 1: Refactor ObservationBuilder**

Replace all four `buildObservation` overloads and `buildExchangeObservation` with a single method that accepts `WorldObservationProvider`. Remove the 7 world-specific methods and 5 cognitive methods (now in providers and blocks utility). Keep 4 CharacterState-dependent methods as private.

Use `ide_edit_member` for the class declaration to update imports, then `ide_replace_member` for each method body.

The refactored class:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.PartitionedDrain;
import io.casehub.blocks.summarisation.observation.affordance.AffordanceRenderer;
import io.casehub.blocks.summarisation.observation.affordance.CognitiveObservationSections;
import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import io.casehub.blocks.summarisation.observation.affordance.WorldObservationProvider;
import io.casehub.eidos.api.AgentGoal;
import io.casehub.neocortex.memory.Memory;
import io.casehub.examples.manor.model.CharacterState;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public final class ObservationBuilder {

    private static final AffordanceRenderer RENDERER = new AffordanceRenderer();

    public static String buildObservation(WorldObservationProvider worldProvider,
                                          CharacterState character,
                                          List<AgentGoal> goals,
                                          PartitionedDrain<String> drain,
                                          List<Memory> memories,
                                          List<Memory> reflections,
                                          Map<String, List<Memory>> relationshipMemories) {
        var sections = new ArrayList<ObservationSection>();

        // World perception (from provider)
        sections.addAll(worldProvider.worldSections());

        // Character state (manor-local)
        for (var entry : relationshipMemories.entrySet()) {
            if (!entry.getValue().isEmpty()) {
                sections.add(CognitiveObservationSections.relationshipNotesSection(
                        entry.getKey(), entry.getValue()));
            }
        }
        sections.add(inventorySection(character));
        var thinking = currentThinkingSection(character);
        if (thinking != null) { sections.add(thinking); }

        // Cognitive state (blocks utility + manor-local)
        sections.add(CognitiveObservationSections.goalsSection(goals));
        planSections(character).forEach(sections::add);
        sections.add(CognitiveObservationSections.recentActivitySection(drain));
        if (memories != null && !memories.isEmpty()) {
            sections.add(CognitiveObservationSections.pastExperienceSection(memories));
        }
        if (reflections != null && !reflections.isEmpty()) {
            sections.add(CognitiveObservationSections.insightsSection(reflections));
        }
        sections.add(lastActionResultSection(character));

        return RENDERER.renderObservation(sections);
    }

    // Keep these 4 private methods unchanged:
    // inventorySection, currentThinkingSection, planSections, lastActionResultSection
}
```

- [ ] **Step 2: Update CharacterAgentLoop**

Read the current call site at line 69. Replace:

```java
// Before
String observation = ObservationBuilder.buildObservation(character, world, goals, drain, memories);

// After
var worldProvider = new ManorWorldObservationProvider(character, world, drain, character.capabilityTags());
String observation = ObservationBuilder.buildObservation(worldProvider, character, goals, drain, memories, List.of(), Map.of());
```

Use `ide_replace_member` or Edit tool for the specific lines. Add imports for `ManorWorldObservationProvider`.

- [ ] **Step 3: Update ScenarioOrchestrator**

Read the current call site at line 292. Replace:

```java
// Before
String observation = ObservationBuilder.buildObservation(
        character, world, goals, drain, memories, reflections,
        relationshipMemories, observerTags);

// After
var worldProvider = new ManorWorldObservationProvider(character, world, drain, observerTags);
String observation = ObservationBuilder.buildObservation(
        worldProvider, character, goals, drain, memories, reflections, relationshipMemories);
```

Add import for `ManorWorldObservationProvider`.

- [ ] **Step 4: Update ExchangeRunner**

Read the current call site at line 64. Replace:

```java
// Before
String observation = ObservationBuilder.buildExchangeObservation(responder, lastDialogue, world);

// After
var worldProvider = new ManorExchangeObservationProvider(responder, lastDialogue, world);
String observation = ObservationBuilder.buildObservation(
        worldProvider, responder, List.of(), emptyDrain, List.of(), List.of(), Map.of());
```

The `emptyDrain` needs to be available — either construct inline (`new PartitionedDrain<>(ObservationResult.empty(0), Map.of())`) or add a field. Read the current ExchangeRunner class to determine the best approach.

Add imports for `ManorExchangeObservationProvider`, `PartitionedDrain`, `ObservationResult`.

- [ ] **Step 5: Update LiveScenarioTest**

Read the current call site at line 144. Apply the same provider pattern as CharacterAgentLoop.

- [ ] **Step 6: Update AutonomousScenarioRunner**

Read the current call site at line 63. Apply the same provider pattern as CharacterAgentLoop.

- [ ] **Step 7: Update ObservationBuilderTest**

This is the largest change — 30+ test methods. Each test that called `ObservationBuilder.buildObservation(character, world, goals, drain, ...)` needs to construct a `ManorWorldObservationProvider` first. Each test that called `buildExchangeObservation` needs a `ManorExchangeObservationProvider`.

Pattern for each test:

```java
// Before
var obs = ObservationBuilder.buildObservation(character, world, List.of(), emptyDrain);

// After
var provider = new ManorWorldObservationProvider(character, world, emptyDrain, Set.of());
var obs = ObservationBuilder.buildObservation(provider, character, List.of(), emptyDrain, List.of(), List.of(), Map.of());
```

Consider extracting a helper method in the test class:

```java
private String buildObs(CharacterState character, Set<String> tags) {
    var provider = new ManorWorldObservationProvider(character, world, emptyDrain, tags);
    return ObservationBuilder.buildObservation(provider, character, List.of(), emptyDrain, List.of(), List.of(), Map.of());
}
```

Update all 30+ test methods to use either the helper or direct provider construction (for tests that pass goals, memories, custom drains, etc.).

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: ALL PASS

- [ ] **Step 9: Run `ide_diagnostics` to verify no compilation errors**

Check for any missed import updates or reference issues across the modified files.

- [ ] **Step 10: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java wacky-manor/src/test/java/io/casehub/examples/manor/engine/LiveScenarioTest.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java
git commit -m "feat(#48): refactor ObservationBuilder to use WorldObservationProvider

ObservationBuilder now accepts WorldObservationProvider instead of WorldState.
World sections delegated to ManorWorldObservationProvider.
Cognitive formatters delegated to CognitiveObservationSections in blocks.
Section ordering regrouped: perception → character state → cognitive state.

Closes #48"
```

---

## References

- [2026-08-21-extract-observation-spi-design.md] — design spec this plan implements
- [wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java] — current monolith being refactored
- [blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/WorldObservationProvider.java] — SPI from blocks#127
- [blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationSection.java] — sealed interface for section types
- [GitHub #48] — focal issue
- [GitHub blocks#127] — WorldObservationProvider SPI (complete)
