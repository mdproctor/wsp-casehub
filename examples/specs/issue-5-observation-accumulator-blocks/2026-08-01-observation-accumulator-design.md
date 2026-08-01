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
    │                              │  current room:                        │
    │                              │    accumulator.drainObservation(now)   │
    │                              │    → renderer (compact + tier select)  │
    │                              │  remembered rooms:                     │
    │                              │    return cached result (§2.3)         │
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
    ManorObservationRenderer renderer;  // shared, stateless — see §3.2

    // configuration (maps to TieredObservationRenderer thresholds)
    int verbatimThreshold;   // default: 10
    int groupedThreshold;    // default: 15
}
```

**`CharacterObservationState`** holds:
- `ConcurrentHashMap<String, ObservationAccumulator<ManorEvent>>` — one
  accumulator per room the character has visited. `ConcurrentHashMap` for
  safe concurrent access between the game loop thread (collect via
  `publishEvent`) and character virtual threads (drain). Accumulators are
  lazy-created via `computeIfAbsent()` on first event routed to a room.
- `Map<String, ObservationResult> rememberedDrainCache` — cached drain
  results for remembered rooms (see §2.3)

Room position is **not** duplicated here — `CharacterState.currentRoom` is
the single source of truth, read at drain time to identify the current-room
accumulator.

### 1.3 Lifecycle

- **Scenario start:** `init(worldState)` creates a `CharacterObservationState`
  for each character, with one accumulator seeded for their starting room.
- **During scenario:** events flow through `publishEvent()`, characters drain
  on each turn.
- **Scenario complete:** accumulators are abandoned. No explicit cleanup.
- **Scenario restart:** new `init()` call creates fresh state.

**Known limitation — buffer growth:** Event buffers grow unboundedly between
drains. If a character's virtual thread blocks (e.g., on LLM timeout), events
accumulate without compaction. At POC scale (5 characters, ~5 rooms) this is
bounded by the game loop's turn-based event generation rate and is not a memory
concern. Phase 2.9 (6 rooms) should re-evaluate if buffer growth becomes
significant.

No `EventStreamBus` — direct method call from the game loop. The bus model
assumes static, per-agent subscription predicates (`bus.subscribe(e ->
agent.canSee(e), accumulator::collect)`), but the spec's routing is
per-room with dynamic room tracking — the publisher decides which
accumulator to target based on the character's current room, which changes
during the scenario. A static predicate cannot express this. Additionally,
direct calls avoid the subscription-drop gotcha documented in
GE-20260629-e8b16d where `bus.clear()` silently kills the pipeline on
lifecycle reset.

---

## 2. Event Model and Room Partitioning

### 2.1 Event Publishing

Events are published to `observationService.publishEvent(event)` from two
sources:

1. **Game loop thread** — after each action resolution (MOVE, TAKE, USE,
   INTERACT, GIVE, LOOK). The game loop captures `departureRoom` before
   calling `ActionResolver.resolve()`, then constructs the enriched
   `ManorEvent` with structured action metadata (see §2.4).
2. **Character virtual threads** — for dialogue and aside events,
   immediately after the LLM response is parsed in `CharacterAgentLoop`.

Both paths call `publishEvent(event)` in addition to the existing
`world.addEvent()` call. `WorldState.recentEvents()` stays — used by the
WebSocket UI. The two consumers (UI and character observations) are
decoupled.

Events with `event.room() == null` (narrator events) are silently skipped
by `publishEvent()` — they have no room context for routing. Narrator
events continue to be delivered via `ManorChannels` and `ManorEventBus`
only.

### 2.2 Routing Logic

When `publishEvent(event)` is called:

1. **Guard:** if `event.room() == null`, return immediately (narrator events)
2. Determine the event's room from `event.room()`
3. For each character:
   - If `characterState.currentRoom()` equals event room → route to their
     accumulator for that room (lazy-created via `computeIfAbsent()`)
   - If character is NOT in that room → skip
4. **Movement events** (`event.actionType() == MOVE`): also route to every
   character whose `currentRoom()` equals `event.departureRoom()` — they
   saw the person leave. The departure room is carried in the event itself
   (see §2.4), not derived from mutable state — this eliminates the
   ordering dependency between state update and event routing.

### 2.3 Room Transitions and Memory Retention

When a character moves rooms:

- `ActionResolver.resolveMove()` updates `CharacterState.currentRoom` —
  this is the authoritative room state. **No duplicate room tracking** in
  `ObservationService`.
- The old room's accumulator stays in the map as a "remembered room"
  partition

**Retention mechanism:** After departure, no new events route to the
remembered room's accumulator (events only route to characters present in
the room). The accumulator buffer is static. On the first drain
post-departure, the accumulator drains normally (at-most-once) and the
`ObservationResult` is cached in `rememberedDrainCache`. Subsequent turns
return the cached result directly — the empty accumulator is not
re-drained. The cache IS the retention: the initial drain already applied
mechanical compaction and tier selection via the renderer, so the cached
result is at its optimal compression level. No progressive re-compression
is needed because the event set is static.

When the character returns to a previously visited room:

- The cached result for that room is removed from `rememberedDrainCache`
- The accumulator resumes its role as the current-room accumulator
- New events arriving in the room are collected normally

### 2.4 Event Payload

`ManorEvent` is enriched with structured action metadata to support
mechanical compaction without narrative parsing:

```java
public record ManorEvent(
    Instant timestamp,
    String type,            // "dialogue", "action", "aside", "narrator"
    String characterId,
    String room,
    String description,     // narrative text (for rendering)
    ActionType actionType,  // non-null for type="action", null otherwise
    String target,          // action target (room-id, object-id, character-id)
    String withItem,        // item used in the action
    String departureRoom    // departure room for MOVE actions only
) {
    // Convenience constructor for non-action events (dialogue, aside, narrator)
    public ManorEvent(Instant timestamp, String type, String characterId,
                      String room, String description) {
        this(timestamp, type, characterId, room, description,
             null, null, null, null);
    }
}
```

`ObservationAccumulator<ManorEvent>` — wrapped in `LevelEvent<ManorEvent>`
when collected. `LevelEvent.timestamp` is derived from
`manorEvent.timestamp().toEpochMilli()` — not from a separate
`System.currentTimeMillis()` call at wrapping time. This ensures the
`TieredObservationRenderer`'s "[N ago]" display uses the event's logical
timestamp.

---

## 3. Rendering Pipeline

### 3.1 MechanicalCompactor

A pure function — no world state dependency, no LLM. Takes
`List<LevelEvent<ManorEvent>>`, returns a reduced list with superseded
entries removed.

**Operates on structured fields** (`actionType`, `target`, `characterId`),
never on narrative text parsing.

| Rule | Supersession logic | Structured fields |
|------|-------------------|-------------------|
| **Position** | Later MOVE supersedes earlier MOVE for same character — keep only latest | `actionType == MOVE`, `characterId` |
| **Presence** | Events implying character X is present are stale when a later MOVE shows X departed | `actionType == MOVE`, `departureRoom` |
| **Inventory** | Later TAKE targeting same object supersedes earlier | `actionType == TAKE`, `target` |
| **Object state** | Later INTERACT/USE targeting same object supersedes earlier | `actionType ∈ {INTERACT, USE}`, `target` |
| **Duplicate dialogue** | Same character, same description — keep first occurrence | `type == "dialogue"`, `characterId`, `description` |

### 3.2 ManorObservationRenderer

A single `ManorObservationRenderer implements ObservationRenderer<ManorEvent>`
that composes with the blocks SPI rather than reimplementing rendering logic
at the service level:

```java
public class ManorObservationRenderer implements ObservationRenderer<ManorEvent> {
    private final MechanicalCompactor compactor;
    private final TieredObservationRenderer<ManorEvent> delegate;

    public CompletionStage<ObservationResult> render(
            List<LevelEvent<ManorEvent>> events, ObservationContext context) {
        List<LevelEvent<ManorEvent>> compacted = compactor.compact(events);
        return delegate.render(compacted, context);
    }
}
```

The `TieredObservationRenderer` is configured with:
- `eventRenderer` — renders `ManorEvent::description`
- `groupKeyExtractor` — extracts `ManorEvent::type` (Dialogue, Movement, etc.)
- `verbatimThreshold` — configurable (default 10)
- `groupedThreshold` — configurable (default 15)
- `summariser` — a `ManorLlmSummariser implements Summariser<ManorEvent, String>`
  backed by `AgentProvider`. System prompt instructs factual compression — preserve
  names, items, and facts; compress dialogue into summaries; do not invent events.
- `headerFormatter` — empty (`ctx -> ""`); section headers are added by
  `ObservationBuilder` during assembly (§5.3)

The renderer is stateless, shared across all accumulators. Tier selection
happens automatically based on post-compaction batch size:

| Post-compaction events | Tier | Typical context |
|----------------------|------|-----------------|
| ≤ verbatimThreshold (10) | VERBATIM | Current room, 1 turn of events |
| ≤ groupedThreshold (15) | GROUPED | Moderate remembered room |
| > groupedThreshold (15) | SUMMARISED | Large remembered room |

This eliminates the separate `LlmObservationRenderer` and the service-level
tier selection logic. The `AgentProvider` dependency moves from
`ObservationService` to `ManorLlmSummariser`, cleanly separating event
routing from rendering concerns.

### 3.3 Drain Flow

Rendering is fully delegated to the `ObservationRenderer` SPI via
`accumulator.drainObservation(now)`:

```
drainObservation(now)
  → atomic copy-and-clear buffer (ObservationAccumulator)
  → renderer.render(events, context)
      → MechanicalCompactor.compact(events)
      → TieredObservationRenderer selects tier by batch size
      → returns ObservationResult
      └─ On LLM failure: return GROUPED tier (over budget but complete)
```

For remembered rooms with cached results (§2.3), `ObservationService`
returns the cache directly without draining.

**LLM failure fallback:** The `ManorLlmSummariser` must catch
`AgentProvider` exceptions internally and degrade gracefully. If LLM
summarisation fails (timeout, quota exhaustion, malformed response), the
summariser returns the mechanically-compacted event descriptions as plain
text — the `TieredObservationRenderer.renderSummarised()` path falls back
to GROUPED-equivalent output. Data is never lost due to a summarisation
failure. This is a contract on the summariser implementation, not a change
to `ObservationAccumulator` (which correctly provides at-most-once delivery
to the renderer per the blocks API contract documented in ARC42STORIES).

---

## 4. Configuration

Quarkus `@ConfigProperty` with defaults. Overridable in
`application.properties` or via system properties for testing.

```properties
manor.observation.verbatim-threshold=10
manor.observation.grouped-threshold=15
```

These map directly to `TieredObservationRenderer` constructor parameters:
- `verbatim-threshold` — batch sizes at or below this render as VERBATIM
  (one timestamped line per event). Current room events typically fall here.
- `grouped-threshold` — batch sizes above verbatim and at or below this
  render as GROUPED (events partitioned by type). Moderate remembered rooms
  fall here.
- Batch sizes above `grouped-threshold` trigger the `ManorLlmSummariser`
  and render as SUMMARISED. No separate config — it's implicit.

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
rooms and those rooms have cached drain results (§2.3) or non-empty
accumulators. Rooms ordered by most recently visited.

### 5.4 CharacterAgentLoop Change

```java
// Before
String observation = ObservationBuilder.buildObservation(character, world, goals);

// After
ObservationDrain drain = observationService.drain(character.agentId(), now);
String observation = ObservationBuilder.buildObservation(character, world, goals, drain);
```

`ObservationService.drain()` internally:

1. Reads `CharacterState.currentRoom` to identify the current-room accumulator
2. Calls `drainObservation(now)` on the current-room accumulator →
   `CompletionStage<ObservationResult>`
3. For each remembered room: returns cached result if available (§2.3),
   otherwise calls `drainObservation(now)` →
   `CompletionStage<ObservationResult>`
4. Awaits all `CompletionStages` in parallel via `CompletableFuture.allOf()`
   — drain runs on a character virtual thread where blocking is appropriate
5. On partial failure: any room whose summarisation fails returns its
   GROUPED-tier fallback (per §3.3). The overall drain succeeds as long as
   at least the current-room drain produces a result.
6. Builds `ObservationDrain` from the collected results

### 5.5 WorldState.recentEvents() Unchanged

Stays for the WebSocket UI room chat panels. The two consumers are decoupled.

---

## 6. Thread Safety

- `publishEvent()` — called from **both** the game loop thread (action
  events) and character virtual threads (dialogue/aside events)
- `drain()` — called from each character's virtual thread
- `ObservationAccumulator` is `synchronized` on collect/drain — safe for
  concurrent access from any combination of threads
- `CharacterObservationState` uses `ConcurrentHashMap<String,
  ObservationAccumulator>` — concurrent iteration during drain is safe but
  **weakly consistent**: a drain in progress might not see a room
  accumulator that was just created by a concurrent `publishEvent`. This
  is acceptable — the new room's events will be captured on the next drain
  cycle.
- Room position: `CharacterState.currentRoom` is the single source of
  truth. Written by the game loop thread (via `ActionResolver`), read by
  character virtual threads (via `drain()` to determine current vs
  remembered rooms). Visibility is guaranteed by the happens-before chain
  through `PendingAction`: the game loop writes `currentRoom`, then calls
  `pending.complete(result)` (`CompletableFuture.complete()`). The character
  thread calls `pending.awaitResult()` (`CompletableFuture.get()`) before
  the next `drain()`. Per JMM, `complete()` happens-before `get()`, so
  the `currentRoom` write is visible to the character thread.

---

## 7. File Changes Summary

| File | Change |
|------|--------|
| `ObservationService.java` | **New** — central event distribution + per-character accumulators + drain assembly |
| `CharacterObservationState.java` | **New** — per-character room→accumulator map + remembered drain cache |
| `ObservationDrain.java` | **New** — record holding drain results |
| `MechanicalCompactor.java` | **New** — deterministic supersession compaction (pure function) |
| `ManorObservationRenderer.java` | **New** — `ObservationRenderer<ManorEvent>` composing compaction + `TieredObservationRenderer` |
| `ManorLlmSummariser.java` | **New** — `Summariser<ManorEvent, String>` backed by `AgentProvider` |
| `ManorEvent.java` | **Modified** — enriched with `ActionType actionType`, `String target`, `String withItem`, `String departureRoom` + convenience constructor |
| `WorldState.java` | **Modified** — new `addEvent(ManorEvent)` overload accepting pre-constructed events |
| `ObservationBuilder.java` | **Modified** — new `ObservationDrain` parameter, replace `recentActivitySection` with `recentActivity` + `remembered` sections |
| `CharacterAgentLoop.java` | **Modified** — drain from `ObservationService`, publish dialogue/aside events through `ObservationService` |
| `ManorEvent.java` | **Modified** — enriched with `ActionType actionType`, `String target`, `String withItem`, `String departureRoom` fields + convenience constructor for non-action events |
| `WorldState.java` | **Modified** — new `addEvent(ManorEvent)` overload accepting pre-constructed events |
| `ScenarioOrchestrator.java` | **Modified** — init `ObservationService` at scenario start, capture departure room before resolve, publish enriched events after action resolution |
| `CharacterAgentLoop.java` | **Modified** — drain from `ObservationService`, publish dialogue/aside events through `ObservationService` |
| `application.properties` | **Modified** — add observation threshold config properties |

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
