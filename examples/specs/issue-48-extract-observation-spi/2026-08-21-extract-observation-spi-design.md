# Extract Observation SPI — Decouple Cognitive Sections from World-Specific Rendering

**Date:** 2026-08-21
**Status:** Approved
**Branch:** issue-48-extract-observation-spi
**Issue:** casehubio/examples#48
**Depends on:** casehubio/blocks#127 (WorldObservationProvider — complete)

## Goal

Refactor `ObservationBuilder` to accept a `WorldObservationProvider` instead of
`WorldState` directly. Extract platform-type-only cognitive section formatters
to a blocks utility class so any agent can reuse them. Manor-specific methods
stay in the manor.

**Verdict gate:** ObservationBuilder assembles observations from
`WorldObservationProvider` + blocks cognitive utility + manor-local character
sections. No direct WorldState/Room/GameObject references remain in the
assembly logic.

---

## 1. Three-Way Method Split

### 1.1 World Sections → ManorWorldObservationProvider (in manor)

Implements `WorldObservationProvider` from blocks. Captures `CharacterState`,
`WorldState`, `PartitionedDrain`, and `Set<String> observerTags` at
construction time. Returns world-perception sections:

| Method | Section header |
|--------|---------------|
| `locationSection` | Current Location |
| `exitsSection` | Exits |
| `objectsSection` + `toObservableEntity` | Visible Objects |
| `charactersSection` | Characters Present |
| `keenObservationsSection` | Keen Observations |
| `directedDialogueSection` | Directed to You |
| `rememberedSection` | Remembered |

These methods move verbatim from `ObservationBuilder` into
`ManorWorldObservationProvider`. The provider's `worldSections()` method
calls each in order and returns the collected list, applying the same
conditional logic currently in `buildObservation` (e.g., keen vs directed
based on observer tags, remembered only when non-empty).

### 1.2 Cognitive Sections → CognitiveObservationSections (in blocks)

New utility class in `io.casehub.blocks.summarisation.observation.affordance`.
Static methods producing `ObservationSection` from platform types:

| Method | Input type | Section header |
|--------|-----------|---------------|
| `goalsSection` | `List<AgentGoal>` | Your Goals |
| `recentActivitySection` | `PartitionedDrain<String>` | Recent Activity |
| `pastExperienceSection` | `List<Memory>` | Past Experience |
| `insightsSection` | `List<Memory>` | Insights |
| `relationshipNotesSection` | `String, List<Memory>` | About {name} |

These methods move verbatim from `ObservationBuilder`. No behavioural
changes — pure extraction.

### 1.3 CharacterState-Dependent → Stay in Manor's ObservationBuilder

These depend on `CharacterState` (a manor type) and cannot move to blocks
without abstracting it. They stay as private methods in `ObservationBuilder`:

| Method | Section header |
|--------|---------------|
| `inventorySection` | Your Inventory |
| `currentThinkingSection` | Your Current Thinking |
| `planSections` | Plan: {goal} |
| `lastActionResultSection` | Last Action Result |

---

## 2. ManorWorldObservationProvider

```java
package io.casehub.examples.manor.agent;

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

    // private methods: locationSection, exitsSection, objectsSection,
    // toObservableEntity, charactersSection, keenObservationsSection,
    // directedDialogueSection, rememberedSection, formatElapsed
    // — moved verbatim from ObservationBuilder
}
```

### 2.1 ManorExchangeObservationProvider

A minimal provider for the exchange (two-character dialogue) path:

```java
public class ManorExchangeObservationProvider implements WorldObservationProvider {

    private final CharacterState character;
    private final String otherDialogue;
    private final WorldState world;

    @Override
    public List<ObservationSection> worldSections() {
        var sections = new ArrayList<ObservationSection>();
        Room room = world.room(character.currentRoom());
        sections.add(ObservationSection.text("Location", room.name()));

        List<CharacterState> others = world.charactersInRoom(character.currentRoom())
                .stream().filter(c -> !c.agentId().equals(character.agentId())).toList();
        if (!others.isEmpty()) {
            sections.add(ObservationSection.items("Others Present", null,
                    others.stream().map(CharacterState::name).toList()));
        }
        sections.add(ObservationSection.text("They said", otherDialogue));
        return sections;
    }
}
```

---

## 3. Refactored ObservationBuilder

After extraction, `ObservationBuilder` becomes a pure assembly point:

```java
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
}
```

**Key changes to the public API:**
- `WorldState world` parameter replaced by `WorldObservationProvider worldProvider`
- The four overloaded `buildObservation` signatures collapse — the provider
  encapsulates world state, observer tags, and drain context. Callers construct
  the appropriate provider and pass it in.
- `buildExchangeObservation` follows the same pattern with
  `ManorExchangeObservationProvider`.

**Section ordering changes** from the current interleaved layout to a
grouped layout:

```
── World perception (from provider) ──
  Current Location, Exits, Visible Objects, Characters Present,
  Remembered, Keen Observations / Directed to You

── Character state (manor-local) ──
  Relationship Notes, Your Inventory, Your Current Thinking

── Cognitive state (from blocks utility + manor-local) ──
  Your Goals, Plan: {goal}, Recent Activity,
  Past Experience, Insights, Last Action Result
```

The current code interleaves world and cognitive sections (e.g.,
`recentActivity` appears before `remembered`). The new layout groups
all world perception together, then all internal state. This is
intentionally cleaner — the LLM sees "what's around you" before
"what you're thinking/planning." Section headers are unchanged, so
any prompt-level section referencing works the same.

### 3.1 Relationship Notes Positioning

`relationshipNotesSection` is called from blocks (CognitiveObservationSections)
but positioned between world sections (after characters) and cognitive sections
in the assembly. The builder handles this ordering — the utility just produces
the section. The character name resolution (`world.character(id).name()`) must
happen at the caller (ObservationBuilder), since the utility doesn't see
WorldState. The current 8-arg overload already resolves names before calling
the section method — this stays.

---

## 4. Caller Changes

### 4.1 ScenarioOrchestrator

```java
// Before
String observation = ObservationBuilder.buildObservation(
        character, world, goals, drain, memories, reflections,
        relationshipMemories, observerTags);

// After
var provider = new ManorWorldObservationProvider(character, world, drain, observerTags);
String observation = ObservationBuilder.buildObservation(
        provider, character, goals, drain, memories, reflections, relationshipMemories);
```

### 4.2 CharacterAgentLoop

```java
// Before
String observation = ObservationBuilder.buildObservation(
        character, world, goals, drain, memories);

// After
var provider = new ManorWorldObservationProvider(
        character, world, drain, character.capabilityTags());
String observation = ObservationBuilder.buildObservation(
        provider, character, goals, drain, memories,
        List.of(), Map.of());
```

### 4.3 ExchangeRunner

```java
// Before
String observation = ObservationBuilder.buildExchangeObservation(
        responder, lastDialogue, world);

// After
var provider = new ManorExchangeObservationProvider(
        responder, lastDialogue, world);
String observation = ObservationBuilder.buildObservation(
        provider, responder, List.of(), emptyDrain,
        List.of(), List.of(), Map.of());
```

### 4.4 LiveScenarioTest + AutonomousScenarioRunner

Same pattern as CharacterAgentLoop — construct provider, pass to builder.

---

## 5. Cross-Repo Work

### 5.1 blocks (casehubio/blocks)

**New class:** `CognitiveObservationSections` in
`io.casehub.blocks.summarisation.observation.affordance`

5 static methods moved from manor's `ObservationBuilder`:
- `goalsSection(List<AgentGoal>)` → `ObservationSection`
- `recentActivitySection(PartitionedDrain<String>)` → `ObservationSection`
- `pastExperienceSection(List<Memory>)` → `ObservationSection`
- `insightsSection(List<Memory>)` → `ObservationSection`
- `relationshipNotesSection(String, List<Memory>)` → `ObservationSection`

No new dependencies — blocks already has eidos-api and neocortex-memory-api.

`WorldObservationProvider` is already in place (blocks#127).

### 5.2 examples/wacky-manor (this repo)

**New classes:**
- `ManorWorldObservationProvider` — implements `WorldObservationProvider`
- `ManorExchangeObservationProvider` — implements `WorldObservationProvider`

**Modified:**
- `ObservationBuilder` — assembly refactored per §3
- `ScenarioOrchestrator` — caller change per §4.1
- `CharacterAgentLoop` — caller change per §4.2
- `ExchangeRunner` — caller change per §4.3
- `LiveScenarioTest` — caller change per §4.4
- `AutonomousScenarioRunner` — caller change per §4.4
- `ObservationBuilderTest` — update all 30+ test methods to construct providers

**Dependency update:** bump `casehub-blocks` SNAPSHOT to pick up
`CognitiveObservationSections` and `WorldObservationProvider`.

---

## 6. Test Plan

### Unit Tests

| Test class | Coverage |
|-----------|----------|
| `CognitiveObservationSectionsTest` (in blocks) | Goals rendering, empty goals, priority sorting. Recent activity rendering, empty drain. Past experience with memories, empty memories. Insights rendering. Relationship notes rendering. |
| `ManorWorldObservationProviderTest` (in manor) | Location, exits, objects, characters sections. Keen observations for perception-tagged observers. Directed dialogue for non-perception observers. Remembered section with elapsed time. Empty room handling. |
| `ManorExchangeObservationProviderTest` (in manor) | Location, others present, dialogue text. Solo exchange (no others). |
| `ObservationBuilderTest` (in manor) | Full assembly with mocked WorldObservationProvider. Section ordering preserved. All existing assertions pass against new signatures. |

### Integration Tests (standard suite)

Existing tests (`AccumulatorScenarioTest`, `RoomReturnMemoryTest`, etc.) continue
to pass — they exercise the full observation pipeline end-to-end. The refactoring
is structural, not behavioural.

---

## References

- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java` — current monolith being refactored
- `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/WorldObservationProvider.java` — SPI from blocks#127
- `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationSection.java` — sealed interface for section types
- `docs/specs/issue-5-observation-accumulator-blocks/2026-08-01-observation-accumulator-design.md` — ObservationAccumulator wiring, ObservationBuilder integration context
- casehubio/blocks#127 — WorldObservationProvider SPI definition
- casehubio/examples#48 — this issue
