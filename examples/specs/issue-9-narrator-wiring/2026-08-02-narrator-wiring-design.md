# Phase 2.7: Live LLM Narrator — Design Spec

**Date:** 2026-08-02
**Issue:** casehubio/examples#9
**Branch:** issue-9-narrator-wiring
**Stacked on:** feat/wacky-manor-poc
**Verdict gate:** Narrator panel shows entertaining LLM-generated running commentary

## Goal

Wire `NarratorAgent` to consume the accumulated event stream and produce live
LLM-generated narration in the Wacky Races narrator style. The narrator is
omniscient — it sees all events across all rooms and generates batched commentary
dispatched to the narrator panel and Qhorus audience channel.

## Architecture

### Component diagram

```
CharacterAgentLoop ──publishEvent──→ ObservationService (characters)
        │
        └──collect──→ SummarisationRunner<ManorEvent, String>
                        ├── WindowPolicy.of(15_000ms, 5 events)
                        ├── Compactor<ManorEvent> (wraps MechanicalCompactor)
                        ├── NarratorSummariser (Summariser SPI → LLM call)
                        └── EventStreamBus<String> (narrator output)
                                 │
                                 └──subscriber──→ ManorChannels.dispatchNarration()
                                                  ManorEventBus.broadcast(narrator(...))

ScenarioOrchestrator ──publishEvent──→ ObservationService (characters)
        │
        └──collect──→ (same SummarisationRunner above)
```

### NarratorAgent

Stateful class (not a CDI bean) created by `ScenarioOrchestrator` in
`runScenario()`. Lifecycle is scenario-scoped — same as `ObservationService`
and `CharacterAgentLoop`.

Responsibilities:
1. Owns a `SummarisationRunner<ManorEvent, String>` from casehub-blocks
2. Runs a virtual thread that calls `runner.tick(now)` in a loop
3. Dispatches narrator output to ManorChannels and ManorEventBus via
   an EventStreamBus subscriber

Constructor parameters:
- `MechanicalCompactor compactor`
- `AgentProvider agentProvider`
- `ManorChannels manorChannels`
- `ManorEventBus webEventBus`
- `int eventThreshold` (default 5)
- `long timerMillis` (default 15_000)

Public API:
- `collect(ManorEvent event)` — wraps as LevelEvent, delegates to runner
- `start(WorldState world)` — starts the narrator virtual thread
- `stop()` — interrupts the narrator thread (called at scenario end)

### NarratorSummariser

Implements `Summariser<ManorEvent, String>`. The narrator's LLM call.

Receives compacted events (compactor already applied by SummarisationRunner).
Formats events into a prompt, calls `AgentProvider.invoke()` with the narrator
system prompt, returns `List.of(narrationText)`.

Prompt format — chronological, room-grouped:

```
Narrate what just happened in 2-3 sentences:

[Kitchen]
- The Hooded Claw picked up the Rat Poison
- The Hooded Claw: "Oh, what a lovely kitchen..."

[Ballroom]
- Penelope Pitstop: "Why, this tea service is simply darlin'!"
- Peter Perfect entered the Ballroom
```

System prompt — the existing narrator prompt from `NarratorAgent.java`:
breathless, alliterative, dramatic, omniscient Wacky Races narrator style
with CAPITAL LETTERS for emphasis, 2-3 sentences max.

### Compactor integration

`MechanicalCompactor` already exists in wacky-manor. It implements the new
`Compactor<ManorEvent>` interface from blocks (casehubio/blocks#83). The
compactor applies deterministic supersession:
- Multiple MOVE events for the same character collapse to the final position
- Multiple WAIT events collapse to one

SummarisationRunner applies the compactor between drain and summarise
automatically.

### Hybrid trigger

`WindowPolicy.of(15_000, 5)` — narrate after 5 events accumulate OR after
15 seconds since the oldest buffered event, whichever comes first.

- **Bursts:** 5 events accumulate quickly during action → narrate promptly
- **Quiet periods:** Timer fires after 15s → narrate whatever is buffered
- **Empty buffer:** `shouldEmit()` returns false → no wasted LLM calls

Configurable via `application.properties`:
- `manor.narrator.event-threshold` — default `5`
- `manor.narrator.timer-seconds` — default `15`
- `manor.narrator.enabled` — default `true`

### Failure handling

SummarisationRunner's `onFailure` handler (casehubio/blocks#84) logs
the failed batch and continues. A missed narration is acceptable — the
next batch covers new events. No retry, no dead-letter.

## Integration with ScenarioOrchestrator

### Creation

In `runScenario()`, after creating `ObservationService` and before launching
character threads:

```java
var narratorAgent = new NarratorAgent(
        compactor, agentProvider, manorChannels, webEventBus,
        narratorEventThreshold, narratorTimerSeconds);
narratorAgent.start(world);
```

### Dual publish

At every existing `observationService.publishEvent(event)` call site, add
`narratorAgent.collect(event)`. Three call sites:

1. `ScenarioOrchestrator.runScenario()` — after action resolution (enriched
   action events)
2. `CharacterAgentLoop.run()` — dialogue events
3. `CharacterAgentLoop.run()` — aside events

### Thread lifecycle

Narrator thread starts after `observationService.init()`, before character
threads launch. Stopped at scenario end via `narratorAgent.stop()`, then
joined alongside character threads.

The narrator thread loop:

```java
while (!world.isScenarioComplete()) {
    runner.tick(System.currentTimeMillis())
          .toCompletableFuture().join();
    Thread.sleep(1_000);  // poll interval — WindowPolicy decides when to actually narrate
}
// Final drain for any remaining buffered events
runner.tick(System.currentTimeMillis())
      .toCompletableFuture().join();
```

### Aside filtering

The narrator sees aside events (villain monologues, character thoughts).
This is correct — the narrator is omniscient and should narrate these
dramatically. The narrator prompt already says "you know everyone's secrets."

### Interaction with scripted narration

In SCRIPTED mode, trigger-fired narrator events (static text from
`triggers.yaml`) and scene beat narration still dispatch directly via
`manorChannels.dispatchNarration()`. The live narrator runs alongside.
If double-narration is noticeable, trigger narrator events can be removed
in a follow-up.

## Event flow

```
1. Character acts → CharacterAgentLoop
2. Action resolved → ScenarioOrchestrator game loop
3. ManorEvent created with enriched metadata
4. Event published to:
   a. ObservationService (character observations — room-scoped)
   b. NarratorAgent.collect() (narrator — omniscient)
5. NarratorAgent's SummarisationRunner buffers the event
6. On next tick() where WindowPolicy fires:
   a. Events drained atomically (drainIfReady)
   b. MechanicalCompactor applied (supersession)
   c. NarratorSummariser formats + calls LLM
   d. Narration published to EventStreamBus<String>
7. Bus subscriber dispatches to:
   a. ManorChannels.dispatchNarration() (Qhorus /manor/audience)
   b. ManorEventBus.broadcast(narrator(...)) (WebSocket → UI)
8. Narrator panel renders the narration
```

## Files changed

### New files
- `NarratorSummariser.java` — `Summariser<ManorEvent, String>` implementation
- `NarratorSummariserTest.java` — unit test
- `NarratorAgentTest.java` — unit test with real SummarisationRunner, mock LLM

### Modified files
- `NarratorAgent.java` — rewrite from static utility to stateful class
- `MechanicalCompactor.java` — implement `Compactor<ManorEvent>`
- `ScenarioOrchestrator.java` — create narrator, add collect() calls
- `CharacterAgentLoop.java` — add narrator collect() calls (signature change: accept NarratorAgent)
- `application.properties` — narrator config properties

### Test files
- `NarratorIntegrationTest.java` — `@QuarkusTest` + `@Tag("llm-eval")`, verdict gate

## Testing

### Unit tests (standard suite)

**NarratorSummariserTest** — no CDI, mock AgentProvider.
- Verify prompt formatting: events grouped by room, chronological order
- Verify LLM response returned as `List.of(narrationText)`
- Verify compacted events produce correct prompt (no duplicate MOVEs)
- Verify failure path: AgentProvider throws → CompletionStage fails gracefully

**NarratorAgentTest** — no CDI, real SummarisationRunner, mock AgentProvider.
- Publish 5 events → call tick() → verify narration dispatched to test subscriber
- Publish 2 events → tick() with fresh timestamp → no narration (below threshold)
- Publish 2 events → tick() with timestamp 16s later → narration fires (timer)
- Verify collect() after stop() is safe (no exception)

### Integration test (llm-eval profile)

**NarratorIntegrationTest** — `@QuarkusTest`, `@Tag("llm-eval")`.
- Start scenario with narrator enabled
- Wait for narrator events on WebSocket
- Assert narrator output contains dramatic language, character references
- Verdict gate: output reads like Wacky Races narrator, not a dry event log

## Config

```properties
# Narrator
manor.narrator.enabled=true
manor.narrator.event-threshold=5
manor.narrator.timer-seconds=15
```

## Dependencies

No new dependencies. Uses existing:
- `casehub-blocks` (SummarisationRunner, Compactor, WindowPolicy, EventStreamBus)
- `casehub-platform-agent-api` (AgentProvider)

## Open questions

None — all design decisions resolved during brainstorming.

## References

- POC spec: `wacky-manor/docs/POC-SPEC.md` (Section 1.6 — Narrator)
- Vision: `wacky-manor/docs/VISION.md` (The Narrator section)
- Phase 2.6 spec: observation accumulator design (workspace)
- blocks hardening: casehubio/blocks#82 (Compactor SPI, failure recovery, atomic drain)
- Garden: GE-20260629-e8b16d (EventStreamBus.clear() gotcha)
- Garden: GE-20260724-7b07f5 (direct injection over event bus)
