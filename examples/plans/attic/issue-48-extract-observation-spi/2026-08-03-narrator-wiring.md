# Phase 2.7 Narrator Wiring — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #9 — Phase 2.7: Wire live LLM narrator to accumulated event stream
**Issue group:** #9

**Goal:** Wire NarratorAgent to consume accumulated ManorEvents via
SummarisationRunner and produce live LLM-generated narration dispatched
to the narrator panel and Qhorus audience channel in AUTONOMOUS mode.

**Architecture:** NarratorAgent owns a `SummarisationRunner<ManorEvent, String>`
with a hybrid trigger (5 events OR 15s). A new ManorEventDispatcher
centralizes all event fan-out. MechanicalCompactor implements the blocks
`Compactor<ManorEvent>` SPI. NarratorSummariser implements `Summariser`
and calls the LLM.

**Tech Stack:** casehub-blocks (SummarisationRunner, Compactor, WindowPolicy,
EventStreamBus), casehub-platform-agent-api (AgentProvider), Quarkus, JUnit 5

## Global Constraints

- Java 26, Quarkus
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
- casehub-blocks 0.2-SNAPSHOT must include Compactor SPI (blocks#83 — CLOSED, available)
- casehub-blocks 0.2-SNAPSHOT must include flush() (blocks#90 — OPEN, hard prerequisite for Task 3 shutdown tests)
- Narrator runs in AUTONOMOUS mode only — SCRIPTED mode unchanged
- Use IntelliJ MCP for all code navigation and structural editing

## Prerequisites

**blocks#90 — Add flush() to SummarisationRunner.** Must be implemented
and `mvn install`ed before Task 3's shutdown tests can be written. Tasks
1–2 can proceed without it.

---

### Task 1: MechanicalCompactor implements Compactor\<ManorEvent\>

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/MechanicalCompactor.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorObservationRenderer.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/MechanicalCompactorTest.java` (may exist — verify)

**Interfaces:**
- Consumes: `io.casehub.blocks.summarisation.Compactor<E>` (blocks SPI)
- Produces: `MechanicalCompactor implements Compactor<ManorEvent>` — used by Task 3 (NarratorAgent constructor)

- [ ] **Step 1: Verify existing compactor tests exist**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=MechanicalCompactorTest`

If tests exist and pass, proceed. If no test class exists, check
`ManorObservationRendererTest` for compactor coverage — the compactor
may be tested indirectly.

- [ ] **Step 2: Add `implements Compactor<ManorEvent>` to MechanicalCompactor**

Use `ide_edit_member` to update the class declaration:

```java
public final class MechanicalCompactor implements Compactor<ManorEvent> {
```

Add the import:
```java
import io.casehub.blocks.summarisation.Compactor;
```

The existing `compact()` method signature already matches
`Compactor<ManorEvent>.compact(List<LevelEvent<ManorEvent>>)` — no
method changes needed.

- [ ] **Step 3: Update ManorObservationRenderer field type**

Change the field type from `MechanicalCompactor` to `Compactor<ManorEvent>`:

```java
private final Compactor<ManorEvent> compactor;
```

Update constructor parameter type to match. Update import.

- [ ] **Step 4: Build and verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`

Expected: all existing tests pass. The interface addition is purely
additive — no behavioral change.

- [ ] **Step 5: Commit**

```bash
git -C <PROJECT> add wacky-manor/src/main/java/io/casehub/examples/manor/agent/MechanicalCompactor.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorObservationRenderer.java
git -C <PROJECT> commit -m "refactor: MechanicalCompactor implements Compactor<ManorEvent>

Refs #9"
```

---

### Task 2: NarratorSummariser

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/NarratorSummariser.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarratorSummariserTest.java`

**Interfaces:**
- Consumes: `io.casehub.blocks.summarisation.Summariser<ManorEvent, String>` (blocks SPI),
  `io.casehub.platform.agent.AgentProvider`, `io.casehub.platform.agent.AgentSessionConfig`
- Produces: `NarratorSummariser` — used by Task 3 (NarratorAgent creates it internally)

- [ ] **Step 1: Write failing test — prompt formatting**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class NarratorSummariserTest {

    static final EventLevel NARRATOR = new EventLevel("narrator", 0);

    @Test
    void formats_events_by_room_and_calls_llm() {
        var capturedPrompt = new AtomicReference<String>();
        AgentProvider mockProvider = config -> {
            capturedPrompt.set(config.userMessage());
            return Multi.createFrom().item(new AgentEvent.TextDelta("DRAMATIC narration!"));
        };

        var summariser = new NarratorSummariser(mockProvider);
        var events = List.of(
                new LevelEvent<>(new ManorEvent(Instant.now(), "action", "hooded-claw",
                        "kitchen", "The Hooded Claw picked up the Rat Poison"), Instant.now().toEpochMilli(), NARRATOR),
                new LevelEvent<>(new ManorEvent(Instant.now(), "dialogue", "penelope",
                        "ballroom", "Penelope Pitstop: \"Why, this is simply darlin'!\""), Instant.now().toEpochMilli(), NARRATOR)
        );

        List<String> result = summariser.summarise(events).toCompletableFuture().join();

        assertThat(result).hasSize(1);
        assertThat(result.get(0)).isEqualTo("DRAMATIC narration!");
        assertThat(capturedPrompt.get()).contains("[Kitchen]");
        assertThat(capturedPrompt.get()).contains("[Ballroom]");
        assertThat(capturedPrompt.get()).contains("Rat Poison");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=NarratorSummariserTest`

Expected: FAIL — `NarratorSummariser` does not exist.

- [ ] **Step 3: Implement NarratorSummariser**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.Summariser;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.stream.Collectors;

public final class NarratorSummariser implements Summariser<ManorEvent, String> {

    private static final Logger log = Logger.getLogger(NarratorSummariser.class);

    static final String SYSTEM_PROMPT = """
        You are the narrator of a Wacky Races cartoon special set in a haunted mansion.
        Your style is breathless, alliterative, dramatic, and omniscient — like the
        original Wacky Races narrator.

        Rules:
        - Use CAPITAL LETTERS for dramatic emphasis
        - Be alliterative when possible
        - Use exclamation marks liberally
        - You see everything and know everyone's secrets
        - Address the audience directly
        - Keep each narration to 2-3 sentences maximum

        Example: "And so our heroes GATHER in the dusty entrance of Doily Manor,
        UTTERLY UNAWARE that DANGER lurks behind every cobweb! The Hooded Claw
        adjusts his disguise and flashes a smile SO sinister it could curdle MILK!"
        """;

    private final AgentProvider agentProvider;

    public NarratorSummariser(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    @Override
    public CompletionStage<List<String>> summarise(List<LevelEvent<ManorEvent>> batch) {
        if (batch.isEmpty()) {
            return CompletableFuture.completedFuture(List.of());
        }
        String prompt = formatPrompt(batch);
        try {
            String narration = agentProvider.invoke(
                            AgentSessionConfig.of(SYSTEM_PROMPT, prompt, Duration.ofSeconds(30)))
                    .filter(e -> e instanceof AgentEvent.TextDelta)
                    .map(e -> ((AgentEvent.TextDelta) e).text())
                    .collect().with(Collectors.joining())
                    .await().atMost(Duration.ofSeconds(60));
            return CompletableFuture.completedFuture(List.of(narration));
        } catch (Exception e) {
            log.warnf("Narrator LLM call failed: %s", e.getMessage());
            return CompletableFuture.failedFuture(e);
        }
    }

    String formatPrompt(List<LevelEvent<ManorEvent>> batch) {
        var byRoom = new LinkedHashMap<String, java.util.ArrayList<String>>();
        for (var event : batch) {
            ManorEvent e = event.payload();
            String room = e.room() != null ? e.room() : "General";
            byRoom.computeIfAbsent(room, k -> new java.util.ArrayList<>())
                    .add("- " + e.description());
        }
        var sb = new StringBuilder("Narrate what just happened in 2-3 sentences:\n\n");
        for (var entry : byRoom.entrySet()) {
            sb.append("[").append(capitalize(entry.getKey())).append("]\n");
            for (String line : entry.getValue()) {
                sb.append(line).append("\n");
            }
            sb.append("\n");
        }
        return sb.toString().stripTrailing();
    }

    private static String capitalize(String s) {
        if (s.isEmpty()) return s;
        return Character.toUpperCase(s.charAt(0)) + s.substring(1);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=NarratorSummariserTest`

Expected: PASS

- [ ] **Step 5: Write test — null room defensive grouping**

```java
@Test
void null_room_grouped_under_general() {
    AgentProvider mockProvider = config ->
            Multi.createFrom().item(new AgentEvent.TextDelta("narration"));
    var summariser = new NarratorSummariser(mockProvider);
    var events = List.of(
            new LevelEvent<>(new ManorEvent(Instant.now(), "action", "x",
                    null, "something happened"), Instant.now().toEpochMilli(), NARRATOR)
    );

    summariser.summarise(events).toCompletableFuture().join();
    // No NPE — null room handled
}
```

- [ ] **Step 6: Write test — LLM failure returns failed future**

```java
@Test
void llm_failure_returns_failed_future() {
    AgentProvider failingProvider = config -> {
        throw new RuntimeException("API timeout");
    };
    var summariser = new NarratorSummariser(failingProvider);
    var events = List.of(
            new LevelEvent<>(new ManorEvent(Instant.now(), "action", "x",
                    "kitchen", "event"), Instant.now().toEpochMilli(), NARRATOR)
    );

    var future = summariser.summarise(events).toCompletableFuture();
    assertThat(future).isCompletedExceptionally();
}
```

- [ ] **Step 7: Run all summariser tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=NarratorSummariserTest`

Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C <PROJECT> add wacky-manor/src/main/java/io/casehub/examples/manor/agent/NarratorSummariser.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarratorSummariserTest.java
git -C <PROJECT> commit -m "feat: add NarratorSummariser — Summariser SPI for LLM narration

Refs #9"
```

---

### Task 3: NarratorAgent rewrite

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/NarratorAgent.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarratorAgentTest.java`

**Interfaces:**
- Consumes: `SummarisationRunner<ManorEvent, String>` (blocks),
  `Compactor<ManorEvent>` (Task 1), `NarratorSummariser` (Task 2),
  `WindowPolicy`, `EventStreamBus<String>`, `EventLevel` (all blocks),
  `ManorChannels`, `ManorEventBus` (wacky-manor)
- Produces: `NarratorAgent` with `collect(ManorEvent)`, `start(WorldState)`,
  `stop()` — used by Task 4 (ManorEventDispatcher) and Task 5 (ScenarioOrchestrator)

**Prerequisite:** blocks#90 (flush) must be available for the shutdown
tests. If not yet landed, implement the class and write tests for
collect/tick/threshold — skip the final-flush test and add a TODO.

- [ ] **Step 1: Write failing test — collect and tick dispatches narration**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.EventStreamBus;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.WindowPolicy;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

class NarratorAgentTest {

    AgentProvider echoProvider = config ->
            io.smallrye.mutiny.Multi.createFrom().item(
                    new AgentEvent.TextDelta("DRAMATIC narration!"));

    @Test
    void collect_5_events_then_tick_dispatches_narration() {
        var dispatched = new ArrayList<String>();
        var agent = new NarratorAgent(
                new MechanicalCompactor(), echoProvider,
                null, null, 5, 15);
        agent.testSubscribe(dispatched::add);

        for (int i = 0; i < 5; i++) {
            agent.collect(new ManorEvent(Instant.now(), "action", "char-" + i,
                    "kitchen", "event " + i));
        }
        agent.tickNow();

        assertThat(dispatched).hasSize(1);
        assertThat(dispatched.get(0)).isEqualTo("DRAMATIC narration!");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=NarratorAgentTest`

Expected: FAIL — NarratorAgent constructor doesn't match.

- [ ] **Step 3: Implement NarratorAgent**

Rewrite the entire file. Use `ide_create_file` or complete rewrite via
Write tool since the class structure changes completely:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.Compactor;
import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.EventStreamBus;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.SummarisationRunner;
import io.casehub.blocks.summarisation.WindowPolicy;
import io.casehub.examples.manor.engine.WorldState;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.examples.manor.web.ManorEventBus;
import io.casehub.examples.manor.web.ManorWebSocketEvent;
import io.casehub.platform.agent.AgentProvider;
import org.jboss.logging.Logger;

import java.util.concurrent.TimeUnit;
import java.util.function.Consumer;

public final class NarratorAgent {

    private static final Logger log = Logger.getLogger(NarratorAgent.class);
    static final EventLevel NARRATOR_LEVEL = new EventLevel("narrator", 0);

    private final SummarisationRunner<ManorEvent, String> runner;
    private final EventStreamBus<String> outputBus;
    private volatile boolean stopped;
    private Thread narratorThread;

    public NarratorAgent(Compactor<ManorEvent> compactor,
                         AgentProvider agentProvider,
                         ManorChannels manorChannels,
                         ManorEventBus webEventBus,
                         int eventThreshold,
                         int timerSeconds) {
        this.outputBus = new EventStreamBus<>();
        if (manorChannels != null || webEventBus != null) {
            outputBus.subscribe(s -> true, event -> {
                if (manorChannels != null) {
                    try { manorChannels.dispatchNarration(event.payload()); }
                    catch (Exception e) { log.warn("Qhorus narration dispatch failed", e); }
                }
                if (webEventBus != null) {
                    try { webEventBus.broadcast(ManorWebSocketEvent.narrator(event.payload())); }
                    catch (Exception e) { log.warn("WebSocket narration dispatch failed", e); }
                }
            });
        }
        var policy = WindowPolicy.of(timerSeconds * 1000L, eventThreshold);
        var summariser = new NarratorSummariser(agentProvider);
        this.runner = new SummarisationRunner<>(policy, compactor, summariser,
                outputBus, NARRATOR_LEVEL,
                batch -> log.warnf("Narrator summarisation failed, batch size=%d", batch.size()));
    }

    public void collect(ManorEvent event) {
        runner.collect(new LevelEvent<>(event, event.timestamp().toEpochMilli(), NARRATOR_LEVEL));
    }

    public void start(WorldState world) {
        this.stopped = false;
        this.narratorThread = Thread.ofVirtual().name("narrator-loop")
                .start(() -> runLoop(world));
    }

    public void stop() {
        this.stopped = true;
        if (narratorThread != null) {
            narratorThread.interrupt();
        }
    }

    public Thread thread() {
        return narratorThread;
    }

    private void runLoop(WorldState world) {
        while (!stopped && !world.isScenarioComplete()) {
            try {
                runner.tick(System.currentTimeMillis())
                        .toCompletableFuture()
                        .orTimeout(90, TimeUnit.SECONDS)
                        .exceptionally(ex -> { log.warn("Narrator tick timed out"); return null; })
                        .join();
                Thread.sleep(1_000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
        runner.flush()
                .toCompletableFuture()
                .orTimeout(90, TimeUnit.SECONDS)
                .exceptionally(ex -> { log.warn("Narrator flush timed out"); return null; })
                .join();
    }

    // Test support — allows tests to subscribe to output without ManorChannels/ManorEventBus
    void testSubscribe(Consumer<String> callback) {
        outputBus.subscribe(s -> true, event -> callback.accept(event.payload()));
    }

    // Test support — manual tick for deterministic testing
    void tickNow() {
        runner.tick(System.currentTimeMillis() + 60_000)
                .toCompletableFuture().join();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=NarratorAgentTest`

Expected: PASS

- [ ] **Step 5: Write test — below threshold does not narrate**

```java
@Test
void below_threshold_does_not_narrate() {
    var dispatched = new ArrayList<String>();
    var agent = new NarratorAgent(
            new MechanicalCompactor(), echoProvider,
            null, null, 5, 15);
    agent.testSubscribe(dispatched::add);

    for (int i = 0; i < 3; i++) {
        agent.collect(new ManorEvent(Instant.now(), "action", "char-" + i,
                "kitchen", "event " + i));
    }
    // tick with current time — timer hasn't expired, count below threshold
    runner_tick_with_current_time(agent);

    assertThat(dispatched).isEmpty();
}

private void runner_tick_with_current_time(NarratorAgent agent) {
    // Access runner directly via reflection or add a test method
    // For now, tickNow() uses future timestamp — we need a tick with real time
    // This test verifies the WindowPolicy gate
}
```

Note: the test for below-threshold requires either exposing the runner
for test access or using `tickNow()` with careful timestamp control.
Simplest approach: add a `tickAt(long now)` test method to NarratorAgent.

- [ ] **Step 6: Write test — collect after stop is safe**

```java
@Test
void collect_after_stop_does_not_throw() {
    var agent = new NarratorAgent(
            new MechanicalCompactor(), echoProvider,
            null, null, 5, 15);
    agent.stop();
    // Should not throw
    agent.collect(new ManorEvent(Instant.now(), "action", "x", "kitchen", "event"));
}
```

- [ ] **Step 7: Run all NarratorAgent tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=NarratorAgentTest`

Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git -C <PROJECT> add wacky-manor/src/main/java/io/casehub/examples/manor/agent/NarratorAgent.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarratorAgentTest.java
git -C <PROJECT> commit -m "feat: rewrite NarratorAgent — stateful class with SummarisationRunner

Refs #9"
```

---

### Task 4: ManorEventDispatcher

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorEventDispatcher.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorEventDispatcherTest.java`

**Interfaces:**
- Consumes: `WorldState.addEvent(ManorEvent)`, `ObservationService.publishEvent(ManorEvent)`,
  `NarratorAgent.collect(ManorEvent)` (nullable), `ManorChannels`, `ManorEventBus`
- Produces: `ManorEventDispatcher.publish(ManorEvent)` — used by Task 5
  (ScenarioOrchestrator and CharacterAgentLoop)

- [ ] **Step 1: Write failing test — fan-out to all targets**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.engine.WorldState;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.examples.manor.web.ManorEventBus;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ManorEventDispatcherTest {

    @Test
    void publish_fans_out_to_all_targets() {
        var world = mock(WorldState.class);
        var observationService = mock(ObservationService.class);
        var narratorAgent = mock(NarratorAgent.class);
        var manorChannels = mock(ManorChannels.class);
        var webEventBus = mock(ManorEventBus.class);

        var dispatcher = new ManorEventDispatcher(
                world, observationService, narratorAgent,
                manorChannels, webEventBus);

        var event = new ManorEvent(Instant.now(), "dialogue", "penelope",
                "ballroom", "Penelope: \"Darlin'!\"");

        dispatcher.publishDialogue(event);

        verify(world).addEvent(event);
        verify(observationService).publishEvent(event);
        verify(narratorAgent).collect(event);
        verify(manorChannels).dispatchDialogue("penelope", "ballroom", "Penelope: \"Darlin'!\"");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorEventDispatcherTest`

Expected: FAIL — `ManorEventDispatcher` does not exist.

- [ ] **Step 3: Implement ManorEventDispatcher**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.engine.WorldState;
import io.casehub.examples.manor.model.ActionType;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.examples.manor.web.ManorEventBus;
import io.casehub.examples.manor.web.ManorWebSocketEvent;
import org.jboss.logging.Logger;

public final class ManorEventDispatcher {

    private static final Logger log = Logger.getLogger(ManorEventDispatcher.class);

    private final WorldState world;
    private final ObservationService observationService;
    private final NarratorAgent narratorAgent;
    private final ManorChannels manorChannels;
    private final ManorEventBus webEventBus;

    public ManorEventDispatcher(WorldState world,
                                ObservationService observationService,
                                NarratorAgent narratorAgent,
                                ManorChannels manorChannels,
                                ManorEventBus webEventBus) {
        this.world = world;
        this.observationService = observationService;
        this.narratorAgent = narratorAgent;
        this.manorChannels = manorChannels;
        this.webEventBus = webEventBus;
    }

    public void publishAction(ManorEvent event, String characterId, String room, double x) {
        world.addEvent(event);
        observationService.publishEvent(event);
        collectNarrator(event);
        if (event.actionType() == ActionType.MOVE) {
            manorChannels.dispatchPositionEvent(characterId, room);
            webEventBus.broadcast(ManorWebSocketEvent.position(characterId, room, x));
        }
    }

    public void publishDialogue(ManorEvent event) {
        world.addEvent(event);
        observationService.publishEvent(event);
        collectNarrator(event);
        manorChannels.dispatchDialogue(event.characterId(), event.room(), event.description());
        webEventBus.broadcast(ManorWebSocketEvent.dialogue(
                event.characterId(), event.room(), extractContent(event)));
    }

    public void publishAside(ManorEvent event) {
        world.addEvent(event);
        observationService.publishEvent(event);
        collectNarrator(event);
        manorChannels.dispatchAside(event.characterId(), extractContent(event));
        webEventBus.broadcast(ManorWebSocketEvent.aside(
                event.characterId(), extractContent(event)));
    }

    private void collectNarrator(ManorEvent event) {
        if (narratorAgent != null) {
            narratorAgent.collect(event);
        }
    }

    private String extractContent(ManorEvent event) {
        String desc = event.description();
        int colonIdx = desc.indexOf(": ");
        return colonIdx >= 0 ? desc.substring(colonIdx + 2) : desc;
    }
}
```

Note: the exact dispatch signatures will need refinement during
implementation based on how CharacterAgentLoop currently constructs
events and calls ManorChannels. Use `ide_find_references` on
`dispatchDialogue`, `dispatchAside`, `dispatchPositionEvent` to
confirm the call patterns.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorEventDispatcherTest`

Expected: PASS

- [ ] **Step 5: Write test — null narrator does not NPE**

```java
@Test
void null_narrator_does_not_npe() {
    var world = mock(WorldState.class);
    var observationService = mock(ObservationService.class);
    var manorChannels = mock(ManorChannels.class);
    var webEventBus = mock(ManorEventBus.class);

    var dispatcher = new ManorEventDispatcher(
            world, observationService, null,
            manorChannels, webEventBus);

    var event = new ManorEvent(Instant.now(), "dialogue", "penelope",
            "ballroom", "Penelope: test");

    dispatcher.publishDialogue(event);

    verify(world).addEvent(event);
    verify(observationService).publishEvent(event);
}
```

- [ ] **Step 6: Run all dispatcher tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorEventDispatcherTest`

Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C <PROJECT> add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorEventDispatcher.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorEventDispatcherTest.java
git -C <PROJECT> commit -m "feat: add ManorEventDispatcher — centralized event fan-out

Refs #9"
```

---

### Task 5: Wire ScenarioOrchestrator + CharacterAgentLoop

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/main/resources/application.properties`

**Interfaces:**
- Consumes: `ManorEventDispatcher` (Task 4), `NarratorAgent` (Task 3),
  `MechanicalCompactor` (Task 1)
- Produces: wired scenario with narrator support in AUTONOMOUS mode

- [ ] **Step 1: Add config properties to application.properties**

```properties
# Narrator
manor.narrator.enabled=true
manor.narrator.event-threshold=5
manor.narrator.timer-seconds=15
```

- [ ] **Step 2: Add config injection to ScenarioOrchestrator**

Use `ide_insert_member` to add after the existing `groupedThreshold` field:

```java
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.narrator.enabled", defaultValue = "true")
boolean narratorEnabled;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.narrator.event-threshold", defaultValue = "5")
int narratorEventThreshold;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.narrator.timer-seconds", defaultValue = "15")
int narratorTimerSeconds;
```

- [ ] **Step 3: Create dispatcher and narrator in runScenario()**

Modify `ScenarioOrchestrator.runScenario()` using `ide_replace_member`.
After creating `observationService` and before launching character threads:

```java
NarratorAgent narratorAgent = null;
if (narratorEnabled && mode == io.casehub.examples.manor.model.ScenarioMode.AUTONOMOUS) {
    narratorAgent = new NarratorAgent(
            compactor, agentProvider, manorChannels, webEventBus,
            narratorEventThreshold, narratorTimerSeconds);
    narratorAgent.start(world);
}

var dispatcher = new ManorEventDispatcher(
        world, observationService, narratorAgent,
        manorChannels, webEventBus);
```

- [ ] **Step 4: Update CharacterAgentLoop signature**

Change `CharacterAgentLoop.run()` to accept `ManorEventDispatcher` instead
of `ObservationService`, `ManorChannels`, and `ManorEventBus`.

Use `ide_change_signature` or `ide_edit_member` on `CharacterAgentLoop.run()`.

New signature:
```java
public void run(CharacterState character, WorldState world,
                AgentProvider agentProvider, String systemPrompt,
                BlockingQueue<PendingAction> actionQueue,
                ManorEventDispatcher dispatcher,
                java.util.List<io.casehub.eidos.api.AgentGoal> goals,
                ObservationService observationService)
```

Note: `observationService` is still needed for `drain()` calls in the
loop. Only event publishing moves to the dispatcher. Use
`ide_find_references` to trace all uses of `manorChannels` and
`webEventBus` in CharacterAgentLoop and replace them with
`dispatcher.publishDialogue(event)` / `dispatcher.publishAside(event)`.

- [ ] **Step 5: Replace event publishing in CharacterAgentLoop**

Replace the inline dialogue publishing block:
```java
// Before (dialogue)
world.addEvent(dialogueEvent);
observationService.publishEvent(dialogueEvent);
manorChannels.dispatchDialogue(...);
webEventBus.broadcast(...);

// After
dispatcher.publishDialogue(dialogueEvent);
```

Same for aside events:
```java
// Before (aside)
world.addEvent(asideEvent);
observationService.publishEvent(asideEvent);
manorChannels.dispatchAside(...);
webEventBus.broadcast(...);

// After
dispatcher.publishAside(asideEvent);
```

- [ ] **Step 6: Replace event publishing in ScenarioOrchestrator game loop**

Replace the action event publishing in the game loop:
```java
// Before
world.addEvent(enrichedEvent);
observationService.publishEvent(enrichedEvent);

// After
dispatcher.publishAction(enrichedEvent, pending.character().agentId(),
        pending.character().currentRoom(), pending.character().x());
```

Remove the separate `manorChannels.dispatchPositionEvent()` and
`webEventBus.broadcast(position(...))` calls — the dispatcher handles them.

- [ ] **Step 7: Add shutdown ordering**

After the game loop exits, before the existing character thread join:

```java
// After existing character join loop:
if (narratorAgent != null) {
    narratorAgent.stop();
    try {
        narratorAgent.thread().join(Duration.ofSeconds(120));
        if (narratorAgent.thread().isAlive()) {
            log.warn("Narrator thread did not terminate");
            narratorAgent.thread().interrupt();
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```

- [ ] **Step 8: Build and run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`

Expected: all existing tests pass. The refactor changes call sites but
not behavior for SCRIPTED mode (no narrator created, dispatcher has
null narrator).

- [ ] **Step 9: Commit**

```bash
git -C <PROJECT> add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/main/resources/application.properties
git -C <PROJECT> commit -m "feat: wire NarratorAgent and ManorEventDispatcher into ScenarioOrchestrator

Narrator starts in AUTONOMOUS mode, dispatches via ManorEventDispatcher.
CharacterAgentLoop uses dispatcher for all event publishing.

Refs #9"
```

---

### Task 6: Integration test (verdict gate)

**Files:**
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarratorIntegrationTest.java`

**Interfaces:**
- Consumes: full wired scenario in AUTONOMOUS mode
- Produces: verdict gate evidence

- [ ] **Step 1: Write integration test**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.web.ManorWebSocket;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class NarratorIntegrationTest {

    @Inject
    ScenarioOrchestrator orchestrator;

    @Test
    void narrator_generates_commentary_in_autonomous_mode() throws Exception {
        // Configure for autonomous mode with narrator
        // Start scenario, wait for narrator events on the event bus
        // Assert narrator output contains dramatic language
        // This is the verdict gate test — the exact assertions depend on
        // the WebSocket event capture mechanism already in place
        //
        // Implementation note: use the ManorEventBus to capture narrator
        // events directly rather than opening a WebSocket connection.
        // Follow the pattern of existing tests.
    }
}
```

The exact test implementation depends on the existing test infrastructure
for WebSocket event capture. Use `ide_find_references` on
`ManorEventBus` and `ManorWebSocket` to find existing test patterns.

- [ ] **Step 2: Run with llm-eval profile**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -Dtest=NarratorIntegrationTest`

- [ ] **Step 3: Commit**

```bash
git -C <PROJECT> add wacky-manor/src/test/java/io/casehub/examples/manor/agent/NarratorIntegrationTest.java
git -C <PROJECT> commit -m "test: add narrator integration test — verdict gate for Phase 2.7

Refs #9"
```

---

## Task Dependency Graph

```
Task 1 (Compactor) ──┐
                     ├── Task 3 (NarratorAgent) ──┐
Task 2 (Summariser) ─┘                           ├── Task 5 (Wiring)── Task 6 (Integration)
                      Task 4 (Dispatcher) ────────┘
```

Tasks 1, 2, 4 are independent of each other. Task 3 depends on 1 and 2.
Task 5 depends on 3 and 4. Task 6 depends on 5.

## Post-implementation

- File casehubio/examples issue for WorldState.addEvent() thread safety
  (pre-existing, documented in spec)
- Update issue #9 acceptance criteria to reflect the design decision
  (direct collection vs PartitionedObservationService)
