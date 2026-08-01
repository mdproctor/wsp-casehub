# Phase 2.6 — ObservationAccumulator Wiring with casehub-blocks

**Date:** 2026-08-01
**Status:** Approved
**Branch:** issue-5-observation-accumulator-blocks
**Issue:** casehubio/examples#5
**Depends on:** Phase 2.5 (complete)

## Goal

Replace raw event tail reading in character observations with accumulated,
compacted, and optionally LLM-summarised context via casehub-blocks
`ObservationAccumulator`. Characters should act on batched context, not raw
event tails, and retain cross-room memory that is mechanically compacted
and semantically compressed when it exceeds a configurable budget.

**Verdict gate:** Characters act on batched context, not raw event tails.
Cross-room memory is retained and compacted. Observation size stays bounded.

---

## 1. ObservationService

Central `@ApplicationScoped` service that owns event distribution and
per-character accumulation.

### 1.1 Data Flow

```
Game Loop                    ObservationService                     CharacterAgentLoop
    │                              │                                       │
    ├─ resolveAction() ──────────► publishEvent(event) ──┐                 │
    │                              │                     │                 │
    │                              │  for each character: │                │
    │                              │    visibility check  │                │
    │                              │    route to room     │                │
    │                              │    partition         │                │
    │                              │◄────────────────────┘                 │
    │                              │                                       │
    │                              │◄──── drain(charId, now) ──────────────┤
    │                              │                                       │
    │                              │  1. mechanical compaction             │
    │                              │  2. LLM summarisation (if over budget)│
    │                              │                                       │
    │                              ├───── ObservationDrain ────────────────►│
    │                              │                                       │
```

### 1.2 Internal State

```java
@ApplicationScoped
public class ObservationService {

    Map<String, CharacterObservationState> characterStates;
    WorldState worldState;
    AgentProvider agentProvider;

    // configuration
    int currentRoomMaxLines;      // default: 10
    int rememberedRoomsMaxLines;  // default: 5
    int summarisationThreshold;   // default: 15
}
```

**`CharacterObservationState`** holds:
- `Map<String, ObservationAccumulator<ManorEvent>>` — one accumulator per
  room the character has visited
- `String currentRoom` — updated when the character moves

### 1.3 Lifecycle

- **Scenario start:** `init(worldState)` creates a `CharacterObservationState`
  for each character, with one accumulator seeded for their starting room.
- **During scenario:** events flow through `publishEvent()`, characters drain
  on each turn.
- **Scenario complete:** accumulators are abandoned. No explicit cleanup.
- **Scenario restart:** new `init()` call creates fresh state.

No `EventStreamBus` — direct method call from the game loop. This avoids
the subscription-drop gotcha documented in GE-20260629-e8b16d where
`bus.clear()` silently kills the pipeline on lifecycle reset.

---

## 2. Event Model and Room Partitioning

### 2.1 Event Publishing

The game loop calls `observationService.publishEvent(event)` after each
action resolution, in addition to the existing `world.addEvent()` call.
`WorldState.recentEvents()` stays — used by the WebSocket UI. The two
consumers (UI and character observations) are decoupled.

Movement events must also be published so the service can track position
changes for supersession compaction.

### 2.2 Routing Logic

When `publishEvent(event)` is called:

1. Determine the event's room from `event.room()`
2. For each character:
   - If character is in that room → route to their **current-room** accumulator
   - If character is NOT in that room → skip
3. Movement events about other characters are also routed to every character
   who was in the **departure** room — they saw the person leave

### 2.3 Room Transitions

When a character moves rooms:

- The service updates `characterState.currentRoom`
- The old room's accumulator stays in the map as a "remembered room"
  partition
- If the character returns later, that accumulator still has their
  compacted history

### 2.4 Event Payload

`ObservationAccumulator<ManorEvent>` — the existing `ManorEvent` record
is the payload, wrapped in `LevelEvent<ManorEvent>` when collected.

---

## 3. Two-Tier Compaction

### 3.1 Tier 1: Mechanical Compaction

A `MechanicalCompactor` — pure function, no world state dependency, no LLM.
Takes `List<LevelEvent<ManorEvent>>`, returns a reduced list with superseded
entries removed.

| Rule | Supersession logic |
|------|-------------------|
| **Position** | "X moved to Kitchen" superseded by later "X moved to Ballroom" — keep only the latest position per character |
| **Presence** | "X is in the Kitchen" stale when we have "X moved to Ballroom" |
| **Inventory** | "Brass key is on the floor" superseded by "Y picked up something" targeting the same object |
| **Object state** | "Cabinet is locked" superseded by "Y interacted with Cabinet" successfully |
| **Duplicate dialogue** | Same character saying the same thing — keep only the first occurrence |

Compaction runs lazily — at drain time, not on collect. This gives the
compactor more context (the character's current room is known at drain time).

### 3.2 Tier 2: LLM Summarisation

After mechanical compaction, if the result exceeds `summarisationThreshold`,
an `LlmObservationRenderer` implementing `ObservationRenderer<ManorEvent>`
fires:

1. Takes the mechanically-compacted events
2. Sends to `AgentProvider` with a system prompt instructing factual
   compression — preserve names, items, and facts; compress dialogue into
   summaries; do not invent events
3. Returns `ObservationResult` with tier `SUMMARISED`

If the budget is not exceeded, events return as-is with tier `VERBATIM`
(current room) or `GROUPED` (remembered rooms after mechanical compaction).

### 3.3 Drain Flow

```
Raw buffer (N events)
  → Mechanical compaction (remove superseded)
  → Count lines
  → If lines ≤ budget: return as GROUPED tier
  → If lines > budget: LLM summarise → return as SUMMARISED tier
```

---

## 4. Configuration

Quarkus `@ConfigProperty` with defaults. Overridable in
`application.properties` or via system properties for testing.

```properties
manor.observation.current-room.max-lines=10
manor.observation.remembered-rooms.max-lines=5
manor.observation.summarisation-threshold=15
```

---

## 5. Integration with ObservationBuilder

### 5.1 New Parameter

```java
public static String buildObservation(
    CharacterState character, WorldState world,
    List<AgentGoal> goals, ObservationDrain drain)
```

### 5.2 ObservationDrain Record

```java
public record ObservationDrain(
    ObservationResult currentRoom,
    Map<String, ObservationResult> rememberedRooms) {}
```

### 5.3 New Observation Sections

The `recentActivitySection()` (which called `world.recentEvents(roomId, 5)`)
is replaced by two sections:

```
== Recent Activity ==
{current-room accumulator drain — verbatim or grouped}

== Remembered ==
Kitchen (3 turns ago): Sneekly was examining the cabinet. You saw rat poison on the shelf.
Entrance Hall (5 turns ago): Dastardly gave Muttley a fake medal.
```

The "Remembered" section only appears if the character has visited other
rooms and those accumulators are non-empty after compaction. Rooms ordered
by most recently visited.

### 5.4 CharacterAgentLoop Change

```java
// Before
String observation = ObservationBuilder.buildObservation(character, world, goals);

// After
ObservationDrain drain = observationService.drain(character.agentId(), now);
String observation = ObservationBuilder.buildObservation(character, world, goals, drain);
```

### 5.5 WorldState.recentEvents() Unchanged

Stays for the WebSocket UI room chat panels. The two consumers are decoupled.

---

## 6. Thread Safety

- `publishEvent()` — called from the game loop thread (single-threaded)
- `drain()` — called from each character's virtual thread
- `ObservationAccumulator` is `synchronized` on collect/drain — safe for
  this access pattern
- `CharacterObservationState.currentRoom` updates happen from the game
  loop thread only (via `publishEvent` processing movement events) — no
  race with drain calls

---

## 7. File Changes Summary

| File | Change |
|------|--------|
| `ObservationService.java` | **New** — central event distribution + per-character accumulators |
| `CharacterObservationState.java` | **New** — per-character room→accumulator map |
| `ObservationDrain.java` | **New** — record holding drain results |
| `MechanicalCompactor.java` | **New** — deterministic supersession compaction |
| `LlmObservationRenderer.java` | **New** — `ObservationRenderer<ManorEvent>` impl using AgentProvider |
| `ObservationBuilder.java` | **Modified** — new `ObservationDrain` parameter, replace `recentActivitySection` with `recentActivity` + `remembered` sections |
| `CharacterAgentLoop.java` | **Modified** — drain from `ObservationService` instead of reading world events directly |
| `ScenarioOrchestrator.java` | **Modified** — init `ObservationService` at scenario start, pass to character loops, publish events after action resolution |
| `application.properties` | **Modified** — add observation budget config properties |

---

## 8. Test Plan

### Unit Tests

| Test class | Coverage |
|-----------|----------|
| `MechanicalCompactorTest` | Position, inventory, object state, presence supersession. Duplicate dialogue removal. Mixed event types. Empty input. Single event passthrough. |
| `ObservationServiceTest` | Event routing by room. Visibility filtering. Room transition partitioning. Movement events to departure room observers. Drain returns current + remembered rooms. |
| `ObservationDrainIntegrationTest` | Full flow: publish → drain → verify compacted output. Current room verbatim, remembered room compacted. Budget threshold triggers summarisation (mocked AgentProvider). |
| `ObservationBuilderTest` | New `ObservationDrain` parameter renders Recent Activity and Remembered sections. Empty remembered rooms omits section. Existing sections unchanged (regression). |

### Integration Tests (standard suite)

| Test | Validation |
|------|-----------|
| `AccumulatorScenarioTest` | Autonomous scenario with mock LLM. HC moves to Kitchen, events accumulate in Entrance Hall as remembered. HC's observation shows verbatim Kitchen events + compacted Entrance Hall memory. |
| `RoomReturnMemoryTest` | Character leaves, events happen, character returns. "While you were away" events present in current-room accumulator. |
| `BudgetThresholdTest` | 20+ events into a room accumulator. Verify mechanical compaction runs. If over threshold, verify LLM summarisation fires (mock AgentProvider). |
| `ScriptedModeUnchangedTest` | Scripted mode regression — triggers and scenes still work with accumulator active. |

### LLM Evaluation (llm-eval profile)

| Test | Verdict |
|------|---------|
| `AccumulatedObservationEvalTest` | Full autonomous scenario with live LLM + accumulator. Manual verdict: do characters reference remembered events? Does HC remember the poison when in the Ballroom? |
