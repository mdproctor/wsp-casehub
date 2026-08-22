# ObservationAccumulator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #5 — Wire ObservationAccumulator with casehub-blocks for batched LLM context
**Issue group:** #5

**Goal:** Replace raw event tail reading in character observations with accumulated, compacted, and optionally LLM-summarised context via casehub-blocks `ObservationAccumulator`.

**Architecture:** Central `ObservationService` receives all world events, routes them to per-character per-room `ObservationAccumulator<ManorEvent>` instances based on room presence. On drain, a `ManorObservationRenderer` applies mechanical compaction then delegates to `TieredObservationRenderer` for tier selection (VERBATIM / GROUPED / SUMMARISED). Cross-room memory is cached post-departure. `ObservationBuilder` assembles the final prompt from drain results.

**Tech Stack:** Java 26, casehub-blocks (summarisation package), casehub-platform (AgentProvider), Quarkus CDI, JUnit 5, AssertJ

## Global Constraints

- All code under `io.casehub.examples.manor` package
- Use `casehub-blocks` types (`ObservationAccumulator`, `TieredObservationRenderer`, `Summariser`, `LevelEvent`, `EventLevel`, `ObservationTier`) — do not reimplement
- Use `AgentProvider` from casehub-platform for LLM calls — do not use LangChain4j directly
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
- IntelliJ MCP: use `mcp__intellij-index__*` for all code navigation and structural editing
- `project_path` for examples: `/Users/mdproctor/claude/casehub/worktrees/59/examples`
- `project_path` for blocks: `/Users/mdproctor/claude/casehub/worktrees/56/blocks`

---

### Task 1: Enrich ManorEvent with structured action metadata

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/ManorEvent.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/engine/WorldStateTest.java`

**Interfaces:**
- Produces: `ManorEvent(Instant, String, String, String, String, ActionType, String, String, String)` — enriched record with `actionType`, `target`, `withItem`, `departureRoom` fields + convenience constructor `ManorEvent(Instant, String, String, String, String)` for non-action events

- [ ] **Step 1: Write failing test for enriched ManorEvent**

```java
@Test
void enrichedManorEvent_carriesActionMetadata() {
    var event = new ManorEvent(Instant.now(), "action", "hooded-claw", "kitchen",
            "Sneekly picked up something.", ActionType.TAKE, "poison", null, null);
    assertThat(event.actionType()).isEqualTo(ActionType.TAKE);
    assertThat(event.target()).isEqualTo("poison");
    assertThat(event.withItem()).isNull();
    assertThat(event.departureRoom()).isNull();
}

@Test
void convenienceConstructor_setsActionFieldsToNull() {
    var event = new ManorEvent(Instant.now(), "dialogue", "penelope", "ballroom",
            "Why, hello darlin'!");
    assertThat(event.actionType()).isNull();
    assertThat(event.target()).isNull();
    assertThat(event.withItem()).isNull();
    assertThat(event.departureRoom()).isNull();
}

@Test
void moveEvent_carriesDepartureRoom() {
    var event = new ManorEvent(Instant.now(), "action", "penelope", "kitchen",
            "Penelope walked into the Kitchen.", ActionType.MOVE, "kitchen", null, "entrance-hall");
    assertThat(event.departureRoom()).isEqualTo("entrance-hall");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=WorldStateTest#enrichedManorEvent_carriesActionMetadata -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — ManorEvent has no 9-arg constructor

- [ ] **Step 3: Modify ManorEvent record**

Use `ide_edit_member` to replace the ManorEvent record:

```java
public record ManorEvent(
    Instant timestamp,
    String type,
    String characterId,
    String room,
    String description,
    ActionType actionType,
    String target,
    String withItem,
    String departureRoom
) {
    public ManorEvent(Instant timestamp, String type, String characterId,
                      String room, String description) {
        this(timestamp, type, characterId, room, description,
             null, null, null, null);
    }
}
```

Add `import io.casehub.examples.manor.model.ActionType;` to ManorEvent.

- [ ] **Step 4: Run tests to verify pass + no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all existing tests pass — the convenience constructor matches the old signature.

- [ ] **Step 5: Commit**

```
feat(wacky-manor): enrich ManorEvent with structured action metadata

Refs #5
```

---

### Task 2: MechanicalCompactor — deterministic supersession

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/MechanicalCompactor.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/MechanicalCompactorTest.java`

**Interfaces:**
- Consumes: `ManorEvent` enriched record (Task 1), `LevelEvent<ManorEvent>` from blocks
- Produces: `MechanicalCompactor.compact(List<LevelEvent<ManorEvent>>) → List<LevelEvent<ManorEvent>>`

- [ ] **Step 1: Write failing tests for position supersession**

```java
class MechanicalCompactorTest {

    static final EventLevel MANOR = new EventLevel("manor", 0);
    final MechanicalCompactor compactor = new MechanicalCompactor();

    private LevelEvent<ManorEvent> moveEvent(String charId, String room,
                                              String departure, long ts) {
        return new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(ts), "action", charId,
                room, charId + " moved to " + room, ActionType.MOVE, room, null, departure),
                ts, MANOR);
    }

    private LevelEvent<ManorEvent> dialogueEvent(String charId, String room,
                                                   String text, long ts) {
        return new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(ts), "dialogue", charId,
                room, charId + ": " + text), ts, MANOR);
    }

    @Test
    void positionSupersession_keepsOnlyLatestMovePerCharacter() {
        var events = List.of(
                moveEvent("penelope", "kitchen", "entrance-hall", 100),
                moveEvent("penelope", "ballroom", "kitchen", 200));
        var result = compactor.compact(events);
        assertThat(result).hasSize(1);
        assertThat(result.get(0).payload().room()).isEqualTo("ballroom");
    }

    @Test
    void positionSupersession_differentCharactersKeptSeparately() {
        var events = List.of(
                moveEvent("penelope", "kitchen", "entrance-hall", 100),
                moveEvent("hooded-claw", "ballroom", "kitchen", 200));
        var result = compactor.compact(events);
        assertThat(result).hasSize(2);
    }

    @Test
    void emptyInput_returnsEmpty() {
        assertThat(compactor.compact(List.of())).isEmpty();
    }

    @Test
    void singleEvent_passesThrough() {
        var events = List.of(dialogueEvent("penelope", "kitchen", "Hello!", 100));
        assertThat(compactor.compact(events)).hasSize(1);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=MechanicalCompactorTest`
Expected: compilation failure — MechanicalCompactor doesn't exist

- [ ] **Step 3: Implement MechanicalCompactor**

Use `ide_create_file` to create `MechanicalCompactor.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.examples.manor.model.ActionType;
import io.casehub.examples.manor.model.ManorEvent;

import java.util.*;

public final class MechanicalCompactor {

    public List<LevelEvent<ManorEvent>> compact(List<LevelEvent<ManorEvent>> events) {
        if (events.isEmpty()) return List.of();

        var latest = new LinkedHashMap<String, LevelEvent<ManorEvent>>();
        var dialogueSeen = new HashSet<String>();
        var result = new ArrayList<LevelEvent<ManorEvent>>();

        for (var event : events) {
            ManorEvent e = event.payload();
            if (e.actionType() != null) {
                String key = supersessionKey(e);
                if (key != null) {
                    latest.put(key, event);
                    continue;
                }
            }
            if ("dialogue".equals(e.type())) {
                String dedupKey = e.characterId() + "::" + e.description();
                if (!dialogueSeen.add(dedupKey)) continue;
            }
            result.add(event);
        }
        result.addAll(latest.values());
        result.sort(Comparator.comparingLong(LevelEvent::timestamp));
        return List.copyOf(result);
    }

    private String supersessionKey(ManorEvent e) {
        return switch (e.actionType()) {
            case MOVE -> "move:" + e.characterId();
            case TAKE -> "take:" + e.target();
            case INTERACT, USE -> "object-state:" + e.target();
            default -> null;
        };
    }
}
```

- [ ] **Step 4: Add remaining test cases**

Add tests for inventory supersession, object state supersession, duplicate dialogue, and mixed events:

```java
@Test
void inventorySupersession_laterTakeSameObjectWins() {
    var events = List.of(
            new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(100), "action", "penelope",
                    "kitchen", "Penelope picked up something.", ActionType.TAKE, "brass-key", null, null),
                    100, MANOR),
            new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(200), "action", "hooded-claw",
                    "kitchen", "Sneekly picked up something.", ActionType.TAKE, "brass-key", null, null),
                    200, MANOR));
    var result = compactor.compact(events);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).payload().characterId()).isEqualTo("hooded-claw");
}

@Test
void objectStateSupersession_laterInteractSameObjectWins() {
    var events = List.of(
            new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(100), "action", "penelope",
                    "kitchen", "Penelope interacted with Cabinet.", ActionType.INTERACT, "cabinet", "brass-key", null),
                    100, MANOR),
            new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(200), "action", "peter",
                    "kitchen", "Peter used something on Cabinet.", ActionType.USE, "cabinet", "oil", null),
                    200, MANOR));
    var result = compactor.compact(events);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).payload().characterId()).isEqualTo("peter");
}

@Test
void duplicateDialogue_keepsFirstOccurrence() {
    var events = List.of(
            dialogueEvent("penelope", "kitchen", "Hello darlin'!", 100),
            dialogueEvent("penelope", "kitchen", "Hello darlin'!", 200));
    var result = compactor.compact(events);
    assertThat(result).hasSize(1);
    assertThat(result.get(0).timestamp()).isEqualTo(100);
}

@Test
void mixedEvents_dialogueAndActionsCompactIndependently() {
    var events = List.of(
            dialogueEvent("penelope", "kitchen", "Hello!", 100),
            moveEvent("hooded-claw", "kitchen", "entrance-hall", 150),
            dialogueEvent("hooded-claw", "kitchen", "Greetings!", 200),
            moveEvent("hooded-claw", "ballroom", "kitchen", 300));
    var result = compactor.compact(events);
    assertThat(result).hasSize(3);
    assertThat(result.stream().filter(e -> e.payload().actionType() == ActionType.MOVE).count())
            .isEqualTo(1);
}
```

- [ ] **Step 5: Run all tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=MechanicalCompactorTest`
Expected: all pass

- [ ] **Step 6: Commit**

```
feat(wacky-manor): add MechanicalCompactor for deterministic event supersession

Refs #5
```

---

### Task 3: ManorObservationRenderer and ManorLlmSummariser

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorObservationRenderer.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorLlmSummariser.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorObservationRendererTest.java`

**Interfaces:**
- Consumes: `MechanicalCompactor` (Task 2), `TieredObservationRenderer` from blocks, `Summariser` from blocks, `AgentProvider` from platform
- Produces: `ManorObservationRenderer implements ObservationRenderer<ManorEvent>` — composes compaction + tiered rendering + LLM fallback

- [ ] **Step 1: Write failing tests**

```java
class ManorObservationRendererTest {

    static final EventLevel MANOR = new EventLevel("manor", 0);

    private LevelEvent<ManorEvent> dialogue(String charId, String text, long ts) {
        return new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(ts), "dialogue", charId,
                "kitchen", charId + ": " + text), ts, MANOR);
    }

    @Test
    void belowVerbatimThreshold_rendersVerbatim() {
        var renderer = new ManorObservationRenderer(new MechanicalCompactor(), 10, 15, null);
        var events = List.of(dialogue("penelope", "Hello!", 1000));
        var result = renderer.render(events, new ObservationContext(2000, 1000))
                .toCompletableFuture().join();
        assertThat(result.tier()).isEqualTo(ObservationTier.VERBATIM);
        assertThat(result.renderedText()).contains("Hello!");
    }

    @Test
    void aboveVerbatimBelowGrouped_rendersGrouped() {
        var renderer = new ManorObservationRenderer(new MechanicalCompactor(), 2, 15, null);
        var events = List.of(
                dialogue("penelope", "Hello!", 1000),
                dialogue("hooded-claw", "Greetings!", 1100),
                dialogue("peter", "Jolly good!", 1200));
        var result = renderer.render(events, new ObservationContext(2000, 1000))
                .toCompletableFuture().join();
        assertThat(result.tier()).isEqualTo(ObservationTier.GROUPED);
    }

    @Test
    void empty_returnsEmptyResult() {
        var renderer = new ManorObservationRenderer(new MechanicalCompactor(), 10, 15, null);
        var result = renderer.render(List.of(), new ObservationContext(2000, 1000))
                .toCompletableFuture().join();
        assertThat(result.eventCount()).isZero();
    }

    @Test
    void compactionReducesBelowThreshold_rendersVerbatim() {
        var renderer = new ManorObservationRenderer(new MechanicalCompactor(), 2, 15, null);
        var events = List.of(
                new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(100), "action", "penelope",
                        "kitchen", "Penelope moved.", ActionType.MOVE, "kitchen", null, "entrance-hall"), 100, MANOR),
                new LevelEvent<>(new ManorEvent(Instant.ofEpochMilli(200), "action", "penelope",
                        "ballroom", "Penelope moved.", ActionType.MOVE, "ballroom", null, "kitchen"), 200, MANOR),
                dialogue("hooded-claw", "Nyah!", 300));
        var result = renderer.render(events, new ObservationContext(1000, 500))
                .toCompletableFuture().join();
        assertThat(result.tier()).isEqualTo(ObservationTier.VERBATIM);
        assertThat(result.eventCount()).isEqualTo(2);
    }

    @Test
    void llmFailure_fallsBackToGroupedText() {
        Summariser<ManorEvent, String> failingSummariser = batch ->
                CompletableFuture.failedFuture(new RuntimeException("LLM timeout"));
        var renderer = new ManorObservationRenderer(new MechanicalCompactor(), 1, 2, failingSummariser);
        var events = List.of(
                dialogue("penelope", "Hello!", 1000),
                dialogue("hooded-claw", "Greetings!", 1100),
                dialogue("peter", "Jolly good!", 1200));
        var result = renderer.render(events, new ObservationContext(2000, 1000))
                .toCompletableFuture().join();
        assertThat(result.tier()).isEqualTo(ObservationTier.GROUPED);
        assertThat(result.renderedText()).contains("Hello!");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorObservationRendererTest`
Expected: compilation failure

- [ ] **Step 3: Implement ManorLlmSummariser**

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
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.stream.Collectors;

public final class ManorLlmSummariser implements Summariser<ManorEvent, String> {

    private static final Logger log = Logger.getLogger(ManorLlmSummariser.class);

    private static final String SYSTEM_PROMPT = """
            You are a concise event summariser for a cartoon mansion game.
            Summarise the following events into a brief narrative paragraph.
            Preserve all character names, item names, and factual details.
            Compress dialogue exchanges into summaries.
            Do not invent events that did not happen.
            Respond with ONLY the summary text, no formatting.""";

    private final AgentProvider agentProvider;

    public ManorLlmSummariser(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    @Override
    public CompletionStage<List<String>> summarise(List<LevelEvent<ManorEvent>> batch) {
        String eventText = batch.stream()
                .map(e -> e.payload().description())
                .collect(Collectors.joining("\n"));
        try {
            String summary = agentProvider.invoke(
                            AgentSessionConfig.of(SYSTEM_PROMPT, eventText, Duration.ofSeconds(30)))
                    .filter(e -> e instanceof AgentEvent.TextDelta)
                    .map(e -> ((AgentEvent.TextDelta) e).text())
                    .collect().with(Collectors.joining())
                    .await().atMost(Duration.ofSeconds(60));
            return CompletableFuture.completedFuture(List.of(summary));
        } catch (Exception e) {
            log.warnf("LLM summarisation failed, falling back to raw text: %s", e.getMessage());
            return CompletableFuture.completedFuture(
                    batch.stream().map(ev -> ev.payload().description()).toList());
        }
    }
}
```

- [ ] **Step 4: Implement ManorObservationRenderer**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.Summariser;
import io.casehub.blocks.summarisation.observation.*;
import io.casehub.examples.manor.model.ManorEvent;

import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.CompletionStage;
import java.util.stream.Collectors;

public final class ManorObservationRenderer implements ObservationRenderer<ManorEvent> {

    private final MechanicalCompactor compactor;
    private final TieredObservationRenderer<ManorEvent> delegate;

    public ManorObservationRenderer(MechanicalCompactor compactor,
                                     int verbatimThreshold,
                                     int groupedThreshold,
                                     Summariser<ManorEvent, String> summariser) {
        this.compactor = compactor;
        var builder = new TieredObservationRenderer<ManorEvent>(
                e -> e.description(),
                e -> e.type(),
                verbatimThreshold,
                groupedThreshold,
                summariser)
                .withHeaderFormatter(ctx -> "");
        this.delegate = builder;
    }

    @Override
    public CompletionStage<ObservationResult> render(
            List<LevelEvent<ManorEvent>> events, ObservationContext context) {
        if (events.isEmpty()) {
            return CompletableFuture.completedFuture(
                    ObservationResult.empty(context.timeSinceLastDrain()));
        }
        List<LevelEvent<ManorEvent>> compacted = compactor.compact(events);
        return delegate.render(compacted, context)
                .exceptionally(ex -> renderFallback(compacted, context));
    }

    private ObservationResult renderFallback(
            List<LevelEvent<ManorEvent>> compacted, ObservationContext context) {
        String text = compacted.stream()
                .map(e -> e.payload().description())
                .collect(Collectors.joining("\n"));
        return new ObservationResult(text, List.of(), compacted.size(),
                context.timeSinceLastDrain(), ObservationTier.GROUPED);
    }
}
```

- [ ] **Step 5: Run tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorObservationRendererTest`
Expected: all pass

- [ ] **Step 6: Commit**

```
feat(wacky-manor): add ManorObservationRenderer with two-tier compaction

Refs #5
```

---

### Task 4: ObservationService, CharacterObservationState, ObservationDrain

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationService.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterObservationState.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationDrain.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/RememberedRoom.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationServiceTest.java`

**Interfaces:**
- Consumes: `ManorObservationRenderer` (Task 3), `ObservationAccumulator` from blocks, `CharacterState` (existing), `WorldState` (existing)
- Produces:
  - `ObservationService.init(WorldState)` — initialise per-character state
  - `ObservationService.publishEvent(ManorEvent)` — route events to accumulators
  - `ObservationService.drain(String characterId, long now) → ObservationDrain`
  - `ObservationDrain(ObservationResult currentRoom, Map<String, RememberedRoom> rememberedRooms)`
  - `RememberedRoom(ObservationResult result, long cachedAt)`

- [ ] **Step 1: Write failing tests for event routing**

```java
class ObservationServiceTest {

    static final EventLevel MANOR = new EventLevel("manor", 0);

    WorldState createWorld() {
        // Use existing WorldState test helper pattern from WorldStateTest
        var rooms = WorldStateTestHelper.createRooms();
        var characters = WorldStateTestHelper.createCharacters();
        return new WorldState(rooms, characters);
    }

    ObservationService createService() {
        var compactor = new MechanicalCompactor();
        var renderer = new ManorObservationRenderer(compactor, 10, 15, null);
        return new ObservationService(renderer);
    }

    @Test
    void publishEvent_routesToCharactersInSameRoom() {
        var world = createWorld();
        var service = createService();
        service.init(world);

        var event = new ManorEvent(Instant.now(), "dialogue", "penelope",
                "entrance-hall", "Penelope: Hello!");
        service.publishEvent(event);

        var drain = service.drain("hooded-claw", System.currentTimeMillis());
        assertThat(drain.currentRoom().eventCount()).isEqualTo(1);
    }

    @Test
    void publishEvent_skipsCharactersInDifferentRoom() {
        var world = createWorld();
        var service = createService();
        service.init(world);

        world.moveCharacter("penelope", "kitchen");
        var event = new ManorEvent(Instant.now(), "dialogue", "penelope",
                "kitchen", "Penelope: Hello from the Kitchen!");
        service.publishEvent(event);

        var drain = service.drain("hooded-claw", System.currentTimeMillis());
        assertThat(drain.currentRoom().eventCount()).isZero();
    }

    @Test
    void publishEvent_nullRoom_silentlySkipped() {
        var world = createWorld();
        var service = createService();
        service.init(world);

        var event = new ManorEvent(Instant.now(), "narrator", null, null,
                "The chandelier creaks ominously!");
        service.publishEvent(event);

        var drain = service.drain("penelope", System.currentTimeMillis());
        assertThat(drain.currentRoom().eventCount()).isZero();
    }

    @Test
    void publishEvent_asideOnlyRoutesToSpeaker() {
        var world = createWorld();
        var service = createService();
        service.init(world);

        var event = new ManorEvent(Instant.now(), "aside", "hooded-claw",
                "entrance-hall", "Nyah-ha-ha!");
        service.publishEvent(event);

        var hcDrain = service.drain("hooded-claw", System.currentTimeMillis());
        assertThat(hcDrain.currentRoom().eventCount()).isEqualTo(1);

        var penelopeDrain = service.drain("penelope", System.currentTimeMillis());
        assertThat(penelopeDrain.currentRoom().eventCount()).isZero();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: compilation failure

- [ ] **Step 3: Implement RememberedRoom, ObservationDrain, CharacterObservationState, ObservationService**

Create four files. `RememberedRoom`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.ObservationResult;

public record RememberedRoom(ObservationResult result, long cachedAt) {}
```

`ObservationDrain`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.ObservationResult;
import java.util.Map;

public record ObservationDrain(
    ObservationResult currentRoom,
    Map<String, RememberedRoom> rememberedRooms) {}
```

`CharacterObservationState`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.ObservationAccumulator;
import io.casehub.blocks.summarisation.observation.ObservationRenderer;
import io.casehub.examples.manor.model.ManorEvent;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public final class CharacterObservationState {

    private final ConcurrentHashMap<String, ObservationAccumulator<ManorEvent>> accumulators
            = new ConcurrentHashMap<>();
    private final LinkedHashMap<String, RememberedRoom> rememberedDrainCache
            = new LinkedHashMap<>();
    private final ObservationRenderer<ManorEvent> renderer;

    public CharacterObservationState(String startRoom, ObservationRenderer<ManorEvent> renderer) {
        this.renderer = renderer;
        accumulators.computeIfAbsent(startRoom, r -> new ObservationAccumulator<>(renderer));
    }

    public ObservationAccumulator<ManorEvent> accumulatorFor(String roomId) {
        return accumulators.computeIfAbsent(roomId, r -> new ObservationAccumulator<>(renderer));
    }

    public Map<String, ObservationAccumulator<ManorEvent>> accumulators() {
        return accumulators;
    }

    public LinkedHashMap<String, RememberedRoom> rememberedDrainCache() {
        return rememberedDrainCache;
    }
}
```

`ObservationService`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.blocks.summarisation.observation.ObservationResult;
import io.casehub.examples.manor.engine.WorldState;
import io.casehub.examples.manor.model.ActionType;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.model.ManorEvent;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public final class ObservationService {

    static final EventLevel MANOR = new EventLevel("manor", 0);

    private final ManorObservationRenderer renderer;
    private final ConcurrentHashMap<String, CharacterObservationState> characterStates
            = new ConcurrentHashMap<>();
    private WorldState worldState;

    public ObservationService(ManorObservationRenderer renderer) {
        this.renderer = renderer;
    }

    public void init(WorldState worldState) {
        this.worldState = worldState;
        characterStates.clear();
        for (var entry : worldState.characters().entrySet()) {
            characterStates.put(entry.getKey(),
                    new CharacterObservationState(entry.getValue().currentRoom(), renderer));
        }
    }

    public void publishEvent(ManorEvent event) {
        if (event.room() == null) return;

        for (var entry : characterStates.entrySet()) {
            String charId = entry.getKey();
            CharacterObservationState charState = entry.getValue();
            CharacterState character = worldState.character(charId);
            String charRoom = character.currentRoom();

            if (charRoom.equals(event.room())) {
                if ("aside".equals(event.type()) && !charId.equals(event.characterId())) {
                    continue;
                }
                charState.accumulatorFor(charRoom).collect(
                        new LevelEvent<>(event, event.timestamp().toEpochMilli(), MANOR));
            }

            if (event.actionType() == ActionType.MOVE
                    && event.departureRoom() != null
                    && charRoom.equals(event.departureRoom())
                    && !charId.equals(event.characterId())) {
                charState.accumulatorFor(charRoom).collect(
                        new LevelEvent<>(event, event.timestamp().toEpochMilli(), MANOR));
            }
        }
    }

    public ObservationDrain drain(String characterId, long now) {
        CharacterObservationState charState = characterStates.get(characterId);
        if (charState == null) {
            return new ObservationDrain(ObservationResult.empty(0), Map.of());
        }
        CharacterState character = worldState.character(characterId);
        String currentRoom = character.currentRoom();

        ObservationResult currentRoomResult = charState.accumulatorFor(currentRoom)
                .drainObservation(now).toCompletableFuture().join();

        var remembered = new LinkedHashMap<String, RememberedRoom>();
        for (var accEntry : charState.accumulators().entrySet()) {
            String roomId = accEntry.getKey();
            if (roomId.equals(currentRoom)) continue;

            var cached = charState.rememberedDrainCache().get(roomId);
            if (cached != null) {
                remembered.put(roomId, cached);
            } else {
                var roomResult = accEntry.getValue()
                        .drainObservation(now).toCompletableFuture().join();
                if (roomResult.eventCount() > 0) {
                    var rememberedRoom = new RememberedRoom(roomResult, now);
                    charState.rememberedDrainCache().put(roomId, rememberedRoom);
                    remembered.put(roomId, rememberedRoom);
                }
            }
        }

        return new ObservationDrain(currentRoomResult, remembered);
    }
}
```

- [ ] **Step 4: Add tests for remembered rooms and movement events**

```java
@Test
void roomTransition_previousRoomBecomeRemembered() {
    var world = createWorld();
    var service = createService();
    service.init(world);

    var event = new ManorEvent(Instant.now(), "dialogue", "penelope",
            "entrance-hall", "Penelope: Hello!");
    service.publishEvent(event);

    world.moveCharacter("hooded-claw", "kitchen");

    var drain = service.drain("hooded-claw", System.currentTimeMillis());
    assertThat(drain.currentRoom().eventCount()).isZero();
    assertThat(drain.rememberedRooms()).containsKey("entrance-hall");
    assertThat(drain.rememberedRooms().get("entrance-hall").result().eventCount()).isEqualTo(1);
}

@Test
void moveEvent_routesToDepartureRoomObservers() {
    var world = createWorld();
    var service = createService();
    service.init(world);

    var moveEvent = new ManorEvent(Instant.now(), "action", "penelope",
            "kitchen", "Penelope walked to the Kitchen.",
            ActionType.MOVE, "kitchen", null, "entrance-hall");
    world.moveCharacter("penelope", "kitchen");
    service.publishEvent(moveEvent);

    var drain = service.drain("hooded-claw", System.currentTimeMillis());
    assertThat(drain.currentRoom().eventCount()).isEqualTo(1);
    assertThat(drain.currentRoom().renderedText()).contains("walked");
}
```

- [ ] **Step 5: Run all tests to verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ObservationServiceTest`
Expected: all pass

- [ ] **Step 6: Run full suite for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all pass

- [ ] **Step 7: Commit**

```
feat(wacky-manor): add ObservationService with per-character room-partitioned accumulators

Refs #5
```

---

### Task 5: Wire ObservationBuilder and CharacterAgentLoop to use ObservationService

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/CharacterState.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/engine/WorldState.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`
- Modify: `wacky-manor/src/main/resources/application.properties`

**Interfaces:**
- Consumes: `ObservationService` (Task 4), `ObservationDrain` (Task 4)
- Produces: Updated `ObservationBuilder.buildObservation(CharacterState, WorldState, List<AgentGoal>, ObservationDrain)` with "Recent Activity" and "Remembered" sections

- [ ] **Step 1: Write failing test for ObservationBuilder with ObservationDrain**

Add to existing `ObservationBuilderTest`:

```java
@Test
void buildObservation_withDrain_rendersRecentActivityFromDrain() {
    var currentRoom = new ObservationResult("- [1s ago] Penelope: Hello!\n",
            List.of(), 1, 1000, ObservationTier.VERBATIM);
    var drain = new ObservationDrain(currentRoom, Map.of());
    String obs = ObservationBuilder.buildObservation(character, world, goals, drain);
    assertThat(obs).contains("== Recent Activity ==");
    assertThat(obs).contains("Penelope: Hello!");
    assertThat(obs).doesNotContain("== Remembered ==");
}

@Test
void buildObservation_withRememberedRooms_rendersRememberedSection() {
    var currentRoom = ObservationResult.empty(0);
    var rememberedResult = new ObservationResult("Sneekly examined the cabinet.\n",
            List.of(), 1, 5000, ObservationTier.GROUPED);
    var remembered = new LinkedHashMap<String, RememberedRoom>();
    remembered.put("kitchen", new RememberedRoom(rememberedResult, System.currentTimeMillis() - 30000));
    var drain = new ObservationDrain(currentRoom, remembered);
    String obs = ObservationBuilder.buildObservation(character, world, goals, drain);
    assertThat(obs).contains("== Remembered ==");
    assertThat(obs).contains("Kitchen");
    assertThat(obs).contains("Sneekly examined the cabinet.");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: compilation failure — `buildObservation` doesn't accept `ObservationDrain`

- [ ] **Step 3: Add ObservationDrain parameter to ObservationBuilder**

Use `ide_edit_member` to modify `buildObservation`:

```java
public static String buildObservation(CharacterState character, WorldState world,
                                      List<AgentGoal> goals, ObservationDrain drain) {
    Room room = world.room(character.currentRoom());
    var sections = new ArrayList<ObservationSection>();

    sections.add(locationSection(room));
    sections.add(exitsSection(room, world));
    sections.add(objectsSection(character, world));
    sections.add(charactersSection(character, world));
    sections.add(inventorySection(character));
    sections.add(goalsSection(goals));
    sections.add(recentActivitySection(drain));
    if (!drain.rememberedRooms().isEmpty()) {
        sections.add(rememberedSection(drain, world));
    }
    sections.add(lastActionResultSection(character));

    return RENDERER.renderObservation(sections);
}
```

Replace `recentActivitySection(CharacterState, WorldState)` with:

```java
private static ObservationSection recentActivitySection(ObservationDrain drain) {
    String text = drain.currentRoom().renderedText();
    if (text == null || text.isBlank()) {
        return ObservationSection.items("Recent Activity", "The room is quiet.", List.of());
    }
    return ObservationSection.text("Recent Activity", text.strip());
}

private static ObservationSection rememberedSection(ObservationDrain drain, WorldState world) {
    var items = new ArrayList<String>();
    var entries = new ArrayList<>(drain.rememberedRooms().entrySet());
    Collections.reverse(entries);
    long now = System.currentTimeMillis();
    for (var entry : entries) {
        String roomId = entry.getKey();
        RememberedRoom remembered = entry.getValue();
        Room room = world.room(roomId);
        long elapsed = now - remembered.cachedAt();
        String timeAgo = formatElapsed(elapsed);
        String text = remembered.result().renderedText();
        if (text != null && !text.isBlank()) {
            items.add(room.name() + " (" + timeAgo + " ago): " + text.strip());
        }
    }
    return ObservationSection.items("Remembered", null, items);
}

private static String formatElapsed(long millis) {
    long seconds = millis / 1000;
    if (seconds < 60) return seconds + "s";
    long minutes = seconds / 60;
    return minutes + " minute" + (minutes == 1 ? "" : "s");
}
```

- [ ] **Step 4: Mark CharacterState.currentRoom as volatile**

Use `ide_edit_member` on `CharacterState.java` to change `private currentRoom String` to `private volatile currentRoom String`.

- [ ] **Step 5: Add addEvent(ManorEvent) overload to WorldState**

Use `ide_insert_member` to add after `addEvent(String, String, String, String)`:

```java
public void addEvent(ManorEvent event) {
    eventLog.add(event);
}
```

- [ ] **Step 6: Modify CharacterAgentLoop to use ObservationService**

The `run()` method gains an `ObservationService` parameter. Change the observation building line:

```java
// Before
String observation = ObservationBuilder.buildObservation(character, world, goals);

// After
ObservationDrain drain = observationService.drain(character.agentId(), System.currentTimeMillis());
String observation = ObservationBuilder.buildObservation(character, world, goals, drain);
```

After dialogue dispatch, publish to ObservationService:

```java
if (response.dialogue() != null) {
    var dialogueEvent = new ManorEvent(Instant.now(), "dialogue", character.agentId(),
            character.currentRoom(), character.name() + ": " + response.dialogue());
    world.addEvent(dialogueEvent);
    observationService.publishEvent(dialogueEvent);
    // ... existing channel/websocket dispatch
}
if (response.aside() != null) {
    var asideEvent = new ManorEvent(Instant.now(), "aside", character.agentId(),
            character.currentRoom(), response.aside());
    world.addEvent(asideEvent);
    observationService.publishEvent(asideEvent);
    // ... existing channel/websocket dispatch
}
```

- [ ] **Step 7: Modify ScenarioOrchestrator to init and wire ObservationService**

In `runScenario()`, after `manorChannels.initChannels()`:

```java
var compactor = new MechanicalCompactor();
var summariser = new ManorLlmSummariser(agentProvider);
var obsRenderer = new ManorObservationRenderer(compactor, verbatimThreshold, groupedThreshold, summariser);
var observationService = new ObservationService(obsRenderer);
observationService.init(world);
```

Pass `observationService` to `CharacterAgentLoop.run()`.

In the game loop, after action resolution and narrative event creation, publish enriched events:

```java
String departureRoom = pending.character().currentRoom();
ActionResult result = actionResolver.resolve(pending.character(), pending.action(), world);

// ... existing narrative + websocket dispatch ...

var enrichedEvent = new ManorEvent(Instant.now(), "action",
        pending.character().agentId(), pending.character().currentRoom(),
        narrative != null ? narrative : result.text(),
        pending.action().type(), pending.action().target(),
        pending.action().withItem(),
        pending.action().type() == ActionType.MOVE ? departureRoom : null);
observationService.publishEvent(enrichedEvent);
```

Add config fields:

```java
@ConfigProperty(name = "manor.observation.verbatim-threshold", defaultValue = "10")
int verbatimThreshold;

@ConfigProperty(name = "manor.observation.grouped-threshold", defaultValue = "15")
int groupedThreshold;
```

- [ ] **Step 8: Add config to application.properties**

```properties
manor.observation.verbatim-threshold=10
manor.observation.grouped-threshold=15
```

- [ ] **Step 9: Update existing ObservationBuilder tests**

Existing tests that call the old 3-arg `buildObservation` need updating to pass an empty `ObservationDrain`:

```java
var emptyDrain = new ObservationDrain(ObservationResult.empty(0), Map.of());
String obs = ObservationBuilder.buildObservation(character, world, goals, emptyDrain);
```

- [ ] **Step 10: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all pass

- [ ] **Step 11: Commit**

```
feat(wacky-manor): wire ObservationService into game loop and character agent

Refs #5
```

---

### Task 6: Integration tests — accumulator scenario, room return, budget threshold

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AccumulatorScenarioTest.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/DialogueAsideRoutingTest.java`

**Interfaces:**
- Consumes: `ObservationService` (Task 4), `ObservationBuilder` (Task 5), `WorldState` (existing)

- [ ] **Step 1: Write AccumulatorScenarioTest**

```java
class AccumulatorScenarioTest {

    @Test
    void hcMovesToKitchen_entranceHallBecomesRemembered() {
        var world = WorldStateTestHelper.createDefaultWorld();
        var service = createService();
        service.init(world);

        // Dialogue in entrance hall before HC moves
        service.publishEvent(new ManorEvent(Instant.now(), "dialogue", "penelope",
                "entrance-hall", "Penelope: What a lovely foyer!"));

        // HC moves to kitchen
        world.moveCharacter("hooded-claw", "kitchen");
        service.publishEvent(new ManorEvent(Instant.now(), "action", "hooded-claw",
                "kitchen", "Sneekly walked to the Kitchen.",
                ActionType.MOVE, "kitchen", null, "entrance-hall"));

        // Kitchen event
        service.publishEvent(new ManorEvent(Instant.now(), "dialogue", "hooded-claw",
                "kitchen", "Sneekly: What have we here..."));

        var drain = service.drain("hooded-claw", System.currentTimeMillis());

        // Current room (kitchen) has kitchen events
        assertThat(drain.currentRoom().eventCount()).isGreaterThan(0);
        assertThat(drain.currentRoom().renderedText()).contains("What have we here");

        // Remembered room (entrance-hall) has pre-move events
        assertThat(drain.rememberedRooms()).containsKey("entrance-hall");
        assertThat(drain.rememberedRooms().get("entrance-hall").result().renderedText())
                .contains("lovely foyer");
    }

    @Test
    void roomReturn_cachedMemoryPersists() {
        var world = WorldStateTestHelper.createDefaultWorld();
        var service = createService();
        service.init(world);

        // Dialogue in entrance hall
        service.publishEvent(new ManorEvent(Instant.now(), "dialogue", "penelope",
                "entrance-hall", "Penelope: Hello!"));

        // HC leaves entrance hall
        world.moveCharacter("hooded-claw", "kitchen");

        // Drain to cache entrance hall
        service.drain("hooded-claw", System.currentTimeMillis());

        // HC returns to entrance hall
        world.moveCharacter("hooded-claw", "entrance-hall");

        // Drain should have entrance-hall as current (empty, no new events)
        // and no remembered rooms (entrance-hall is current again)
        var drain = service.drain("hooded-claw", System.currentTimeMillis());
        assertThat(drain.currentRoom().eventCount()).isZero();
    }

    private ObservationService createService() {
        return new ObservationService(
                new ManorObservationRenderer(new MechanicalCompactor(), 10, 15, null));
    }
}
```

- [ ] **Step 2: Write DialogueAsideRoutingTest**

```java
class DialogueAsideRoutingTest {

    @Test
    void dialogueRoutesToAllInRoom() {
        var world = WorldStateTestHelper.createDefaultWorld();
        var service = createService();
        service.init(world);

        service.publishEvent(new ManorEvent(Instant.now(), "dialogue", "penelope",
                "entrance-hall", "Penelope: Hello everyone!"));

        for (String charId : List.of("hooded-claw", "ant-hill-mob", "peter-perfect", "dick-dastardly")) {
            var drain = service.drain(charId, System.currentTimeMillis());
            assertThat(drain.currentRoom().eventCount())
                    .as("Character %s should see dialogue", charId)
                    .isEqualTo(1);
        }
    }

    @Test
    void asideRoutesOnlyToSpeaker() {
        var world = WorldStateTestHelper.createDefaultWorld();
        var service = createService();
        service.init(world);

        service.publishEvent(new ManorEvent(Instant.now(), "aside", "hooded-claw",
                "entrance-hall", "Nyah-ha-ha! My fiendish plan!"));

        var hcDrain = service.drain("hooded-claw", System.currentTimeMillis());
        assertThat(hcDrain.currentRoom().eventCount()).isEqualTo(1);

        var penelopeDrain = service.drain("penelope", System.currentTimeMillis());
        assertThat(penelopeDrain.currentRoom().eventCount()).isZero();
    }

    private ObservationService createService() {
        return new ObservationService(
                new ManorObservationRenderer(new MechanicalCompactor(), 10, 15, null));
    }
}
```

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest="AccumulatorScenarioTest,DialogueAsideRoutingTest"`
Expected: all pass

- [ ] **Step 4: Run full suite for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all pass

- [ ] **Step 5: Commit**

```
feat(wacky-manor): add integration tests for observation accumulator

Closes #5
```

---
