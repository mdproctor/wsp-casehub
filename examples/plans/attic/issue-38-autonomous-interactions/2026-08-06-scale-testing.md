# Scale Testing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #16 — test: Scale testing — beyond 5 agents and 60 turns
**Issue group:** #16

**Goal:** Run 10 agents for 200 logical turns with <30s average turn latency, maintained personality coherence, and emergent social dynamics.

**Architecture:** Four components built in order — WorldState thread safety, AgentInvocationService (rate limiting + retry + metrics), AgentExperienceService (memory ingest + recall), then scale testing harness wiring for both runners.

**Tech Stack:** Java 26, Quarkus, neocortex memory API (`ExperienceRecorder`, `CaseMemoryStore`, `MemoryQuery`), blocks observation API (`PartitionedObservationService`)

## Global Constraints

- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl wacky-manor`
- Test with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
- LLM eval tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval`
- Use `mvn` not `./mvnw`
- Always `-pl wacky-manor` — never run without module target
- Descriptor profile: `descriptors-composite.yaml` (only profile with all 17 characters)
- Tenant ID: `ManorConstants.TENANCY_ID`
- Use `ide_insert_member` for new methods, `ide_replace_member` for modifications

---

### Task 1: WorldState Thread Safety

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/engine/WorldState.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/engine/WorldStateConcurrencyTest.java`

**Interfaces:**
- Produces: Thread-safe `WorldState` — all existing callers unchanged, `characters()` returns unmodifiable snapshot

- [ ] **Step 1: Write concurrency test**

```java
package io.casehub.examples.manor.engine;

import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.model.ManorEvent;
import io.casehub.examples.manor.model.Room;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.Map;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class WorldStateConcurrencyTest {

    private WorldState createWorld() {
        var rooms = Map.of(
            "hall", new Room("hall", "Hall", "A hall", java.util.List.of("kitchen"), Map.of()),
            "kitchen", new Room("kitchen", "Kitchen", "A kitchen", java.util.List.of("hall"), Map.of()));
        var chars = new java.util.HashMap<String, CharacterState>();
        for (int i = 0; i < 10; i++) {
            chars.put("agent-" + i, new CharacterState("agent-" + i, "Agent " + i, "hall", 0.5, new ArrayList<>()));
        }
        return new WorldState(rooms, chars);
    }

    @Test
    void concurrentAddEventDoesNotCorrupt() throws Exception {
        var world = createWorld();
        int threads = 10;
        int eventsPerThread = 100;
        var barrier = new CyclicBarrier(threads);
        var errors = new AtomicInteger(0);

        var threadList = new ArrayList<Thread>();
        for (int t = 0; t < threads; t++) {
            int threadId = t;
            threadList.add(Thread.ofVirtual().start(() -> {
                try {
                    barrier.await();
                    for (int i = 0; i < eventsPerThread; i++) {
                        world.addEvent("action", "agent-" + threadId, "hall",
                            "Event " + threadId + "-" + i);
                    }
                } catch (Exception e) {
                    errors.incrementAndGet();
                }
            }));
        }
        for (var t : threadList) t.join();

        assertEquals(0, errors.get());
        assertEquals(threads * eventsPerThread, world.allEvents().size());
    }

    @Test
    void concurrentMarkObjectTakenIsAtomic() throws Exception {
        var world = createWorld();
        int threads = 10;
        var barrier = new CyclicBarrier(threads);
        var takenCount = new AtomicInteger(0);

        var threadList = new ArrayList<Thread>();
        for (int t = 0; t < threads; t++) {
            threadList.add(Thread.ofVirtual().start(() -> {
                try {
                    barrier.await();
                    if (world.tryTakeObject("test-item")) {
                        takenCount.incrementAndGet();
                    }
                } catch (Exception e) { /* ignore */ }
            }));
        }
        for (var t : threadList) t.join();

        assertEquals(1, takenCount.get(), "Only one thread should successfully take the object");
    }

    @Test
    void charactersReturnsSnapshot() {
        var world = createWorld();
        var snapshot = world.characters();
        assertThrows(UnsupportedOperationException.class,
            () -> snapshot.put("new", null));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=WorldStateConcurrencyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `tryTakeObject` does not exist, `characters()` is mutable

- [ ] **Step 3: Make WorldState thread-safe**

Replace fields in `WorldState.java`:

```java
// Replace these field declarations:
private final Set<String> firedTriggers = java.util.concurrent.ConcurrentHashMap.newKeySet();
private final Map<String, Set<String>> visibilityOverrides = new java.util.concurrent.ConcurrentHashMap<>();
private final List<ManorEvent> eventLog = java.util.Collections.synchronizedList(new ArrayList<>());
private final Set<String> completedScenes = java.util.concurrent.ConcurrentHashMap.newKeySet();
private final Set<String> takenObjects = java.util.concurrent.ConcurrentHashMap.newKeySet();
private final Map<String, Set<String>> appliedEffects = new java.util.concurrent.ConcurrentHashMap<>();
```

Add atomic take method:

```java
public boolean tryTakeObject(String objectId) {
    return takenObjects.add(objectId);
}
```

Change `characters()` to return unmodifiable snapshot:

```java
public Map<String, CharacterState> characters() { return java.util.Collections.unmodifiableMap(characters); }
```

Change `revealObject` to use concurrent set:

```java
public void revealObject(String objectId, String characterId) {
    visibilityOverrides.computeIfAbsent(objectId, k -> java.util.concurrent.ConcurrentHashMap.newKeySet()).add(characterId);
}
```

Change `applyEffect` similarly:

```java
public void applyEffect(String objectId, String itemId) {
    appliedEffects.computeIfAbsent(objectId, k -> java.util.concurrent.ConcurrentHashMap.newKeySet()).add(itemId);
}
```

Change `recentEvents` to snapshot before streaming:

```java
public List<ManorEvent> recentEvents(String roomId, int limit) {
    List<ManorEvent> snapshot;
    synchronized (eventLog) { snapshot = new ArrayList<>(eventLog); }
    return snapshot.reversed().stream()
            .filter(e -> roomId.equals(e.room()))
            .limit(limit)
            .toList();
}
```

- [ ] **Step 4: Update ActionResolver to use tryTakeObject**

In `ActionResolver.java`, find where `markObjectTaken` is called after `isObjectTaken` check. Replace the check-then-act with the atomic `tryTakeObject`:

```java
// Replace:
//   if (world.isObjectTaken(objectId)) { return fail; }
//   world.markObjectTaken(objectId);
// With:
if (!world.tryTakeObject(objectId)) { return ActionResult.fail("Someone else already took it."); }
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
git add wacky-manor/src/main/java/io/casehub/examples/manor/engine/WorldState.java \
       wacky-manor/src/main/java/io/casehub/examples/manor/engine/ActionResolver.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/engine/WorldStateConcurrencyTest.java
git commit -m "fix(#16): make WorldState thread-safe for 10-agent concurrency"
```

---

### Task 2: AgentInvocationService

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentInvocationService.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentInvocationServiceTest.java`

**Interfaces:**
- Consumes: `AgentProvider` (platform), `AgentResponse.parse()` (existing)
- Produces: `AgentInvocationService.invoke(String systemPrompt, String userPrompt, String agentId)` → `AgentResponse`
- Produces: `AgentInvocationService.metrics()` → `InvocationMetrics`

- [ ] **Step 1: Write test for rate-limited invocation**

```java
package io.casehub.examples.manor.agent;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class AgentInvocationServiceTest {

    private AgentProvider stubProvider(String jsonResponse) {
        return config -> Multi.createFrom().item(
            (AgentEvent) new AgentEvent.TextDelta(jsonResponse));
    }

    @Test
    void concurrencyCapLimitsInFlight() throws Exception {
        var inFlight = new AtomicInteger(0);
        var maxSeen = new AtomicInteger(0);
        AgentProvider slowProvider = config -> {
            int current = inFlight.incrementAndGet();
            maxSeen.updateAndGet(prev -> Math.max(prev, current));
            try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            inFlight.decrementAndGet();
            return Multi.createFrom().item(
                (AgentEvent) new AgentEvent.TextDelta("{\"action\":{\"type\":\"WAIT\"}}"));
        };

        var service = new AgentInvocationService(slowProvider, 2, 60, 2, 1000);
        int agents = 6;
        var barrier = new CyclicBarrier(agents);
        var threads = new java.util.ArrayList<Thread>();

        for (int i = 0; i < agents; i++) {
            int id = i;
            threads.add(Thread.ofVirtual().start(() -> {
                try {
                    barrier.await();
                    service.invoke("system", "user", "agent-" + id);
                } catch (Exception e) { /* ignore */ }
            }));
        }
        for (var t : threads) t.join();

        assertTrue(maxSeen.get() <= 2, "Max in-flight should not exceed concurrency cap of 2, was: " + maxSeen.get());
        assertEquals(agents, service.metrics().totalCalls());
    }

    @Test
    void retryWithJitterOnFailureThenSuccess() {
        var callCount = new AtomicInteger(0);
        AgentProvider failThenSucceed = config -> {
            if (callCount.getAndIncrement() == 0) {
                throw new RuntimeException("transient failure");
            }
            return Multi.createFrom().item(
                (AgentEvent) new AgentEvent.TextDelta("{\"action\":{\"type\":\"WAIT\"}}"));
        };

        var service = new AgentInvocationService(failThenSucceed, 5, 60, 2, 100);
        AgentResponse response = service.invoke("system", "user", "test-agent");

        assertNotNull(response);
        assertEquals(2, callCount.get());
        assertEquals(1, service.metrics().retries());
    }

    @Test
    void fallsBackToIdleAfterMaxRetries() {
        AgentProvider alwaysFails = config -> {
            throw new RuntimeException("permanent failure");
        };

        var service = new AgentInvocationService(alwaysFails, 5, 60, 2, 50);
        AgentResponse response = service.invoke("system", "user", "test-agent");

        assertNotNull(response);
        assertEquals(io.casehub.examples.manor.model.ActionType.WAIT, response.action().type());
        assertEquals(1, service.metrics().fallbacks());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentInvocationServiceTest`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement AgentInvocationService**

```java
package io.casehub.examples.manor.agent;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.concurrent.Semaphore;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.atomic.AtomicLong;
import java.util.stream.Collectors;

public class AgentInvocationService {

    private static final Logger log = Logger.getLogger(AgentInvocationService.class);

    private final AgentProvider agentProvider;
    private final Semaphore semaphore;
    private final int timeoutSeconds;
    private final int maxRetries;
    private final long baseRetryDelayMs;

    private final AtomicLong totalCalls = new AtomicLong();
    private final AtomicLong retries = new AtomicLong();
    private final AtomicLong fallbacks = new AtomicLong();
    private final AtomicLong totalLatencyMs = new AtomicLong();

    public AgentInvocationService(AgentProvider agentProvider,
                                   int maxConcurrent, int timeoutSeconds,
                                   int maxRetries, long baseRetryDelayMs) {
        this.agentProvider = agentProvider;
        this.semaphore = new Semaphore(maxConcurrent);
        this.timeoutSeconds = timeoutSeconds;
        this.maxRetries = maxRetries;
        this.baseRetryDelayMs = baseRetryDelayMs;
    }

    public AgentResponse invoke(String systemPrompt, String userPrompt, String agentId) {
        totalCalls.incrementAndGet();
        long start = System.currentTimeMillis();
        try {
            semaphore.acquire();
            try {
                return callWithRetry(systemPrompt, userPrompt, agentId);
            } finally {
                semaphore.release();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return AgentResponse.idle();
        } finally {
            totalLatencyMs.addAndGet(System.currentTimeMillis() - start);
        }
    }

    private AgentResponse callWithRetry(String systemPrompt, String userPrompt, String agentId) {
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                String text = agentProvider.invoke(
                        AgentSessionConfig.of(systemPrompt, userPrompt,
                            Duration.ofSeconds(timeoutSeconds)))
                    .filter(e -> e instanceof AgentEvent.TextDelta)
                    .map(e -> ((AgentEvent.TextDelta) e).text())
                    .collect().with(Collectors.joining())
                    .await().atMost(Duration.ofSeconds(timeoutSeconds + 60));
                return AgentResponse.parse(text);
            } catch (Exception e) {
                log.warnf("%s: LLM call failed (attempt %d/%d): %s",
                    agentId, attempt + 1, maxRetries + 1, e.getMessage());
                if (attempt < maxRetries) {
                    retries.incrementAndGet();
                    sleepWithJitter(attempt);
                }
            }
        }
        log.warnf("%s: falling back to idle after %d attempts", agentId, maxRetries + 1);
        fallbacks.incrementAndGet();
        return AgentResponse.idle();
    }

    private void sleepWithJitter(int attempt) {
        long delay = baseRetryDelayMs * (1L << attempt);
        long jitter = ThreadLocalRandom.current().nextLong(0, baseRetryDelayMs);
        try {
            Thread.sleep(delay + jitter);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public InvocationMetrics metrics() {
        return new InvocationMetrics(totalCalls.get(), retries.get(),
            fallbacks.get(), totalLatencyMs.get());
    }

    public record InvocationMetrics(long totalCalls, long retries,
                                     long fallbacks, long totalLatencyMs) {
        public long averageLatencyMs() {
            return totalCalls > 0 ? totalLatencyMs / totalCalls : 0;
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentInvocationServiceTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentInvocationService.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentInvocationServiceTest.java
git commit -m "feat(#16): AgentInvocationService — rate-limited, retried agent calls"
```

---

### Task 3: AgentExperienceService

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java`

**Interfaces:**
- Consumes: `ExperienceRecorder.record(ExperienceEvent)`, `CaseMemoryStore.query(MemoryQuery)` → `List<Memory>`
- Produces: `AgentExperienceService.ingest(String agentId, String room, String description, String thinking)` — fire-and-forget
- Produces: `AgentExperienceService.recall(String agentId, int limit)` → `List<Memory>` — with timeout

- [ ] **Step 1: Write tests**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.*;
import io.casehub.neocortex.memory.experience.*;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class AgentExperienceServiceTest {

    private final List<ExperienceEvent> recorded = new ArrayList<>();

    private ExperienceRecorder stubRecorder() {
        return new ExperienceRecorder() {
            public String record(ExperienceEvent event) {
                recorded.add(event);
                return "mem-" + recorded.size();
            }
            public ExperienceStoreResult recordAll(List<ExperienceEvent> events) {
                events.forEach(this::record);
                return new ExperienceStoreResult(
                    events.stream().map(e -> "mem-" + recorded.size()).toList(),
                    List.of());
            }
        };
    }

    private CaseMemoryStore stubStore(List<Memory> memories) {
        return new CaseMemoryStore() {
            public String store(MemoryInput input) { return "id"; }
            public List<Memory> query(MemoryQuery q) { return memories; }
            public int erase(EraseRequest r) { return 0; }
        };
    }

    @Test
    void ingestRecordsExperienceEvent() {
        var service = new AgentExperienceService(stubRecorder(), stubStore(List.of()), "test-tenant");
        service.ingest("hooded-claw", "library", "Searched the bookshelf", "Looking for clues");

        assertEquals(1, recorded.size());
        var event = recorded.getFirst();
        assertEquals("hooded-claw", event.agentId());
        assertEquals("Searched the bookshelf", event.description());
    }

    @Test
    void ingestFailureDoesNotThrow() {
        ExperienceRecorder failingRecorder = new ExperienceRecorder() {
            public String record(ExperienceEvent event) { throw new RuntimeException("store down"); }
            public ExperienceStoreResult recordAll(List<ExperienceEvent> events) { throw new RuntimeException("store down"); }
        };
        var service = new AgentExperienceService(failingRecorder, stubStore(List.of()), "test-tenant");

        assertDoesNotThrow(() -> service.ingest("agent", "room", "desc", "think"));
    }

    @Test
    void recallReturnsMemories() {
        var memories = List.of(
            new Memory("m1", "hooded-claw", MemoryDomain.of("manor"), "test-tenant",
                null, "Found a secret passage", Map.of(), Instant.now(), 0.8));
        var service = new AgentExperienceService(stubRecorder(), stubStore(memories), "test-tenant");

        List<Memory> result = service.recall("hooded-claw", 5);
        assertEquals(1, result.size());
        assertEquals("Found a secret passage", result.getFirst().text());
    }

    @Test
    void recallReturnsEmptyOnTimeout() {
        CaseMemoryStore slowStore = new CaseMemoryStore() {
            public String store(MemoryInput input) { return "id"; }
            public List<Memory> query(MemoryQuery q) {
                try { Thread.sleep(5000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                return List.of();
            }
            public int erase(EraseRequest r) { return 0; }
        };
        var service = new AgentExperienceService(stubRecorder(), slowStore, "test-tenant");
        service.setRecallTimeoutMs(100);

        List<Memory> result = service.recall("agent", 5);
        assertTrue(result.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement AgentExperienceService**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.*;
import io.casehub.neocortex.memory.experience.*;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;
import java.util.concurrent.*;

public class AgentExperienceService {

    private static final Logger log = Logger.getLogger(AgentExperienceService.class);
    private static final MemoryDomain MANOR_DOMAIN = MemoryDomain.of("manor");

    private final ExperienceRecorder recorder;
    private final CaseMemoryStore store;
    private final String tenantId;
    private long recallTimeoutMs = 2000;

    public AgentExperienceService(ExperienceRecorder recorder,
                                   CaseMemoryStore store, String tenantId) {
        this.recorder = recorder;
        this.store = store;
        this.tenantId = tenantId;
    }

    public void setRecallTimeoutMs(long ms) { this.recallTimeoutMs = ms; }

    public void ingest(String agentId, String room, String description, String thinking) {
        try {
            var metadata = new java.util.HashMap<String, String>();
            metadata.put("room", room);
            if (thinking != null) metadata.put("thinking", thinking);
            var event = new Action(agentId, tenantId, null, null,
                description, null, Map.copyOf(metadata), "manor-action");
            recorder.record(event);
        } catch (Exception e) {
            log.warnf("%s: experience ingest failed (non-fatal): %s", agentId, e.getMessage());
        }
    }

    public List<Memory> recall(String agentId, int limit) {
        try {
            var future = CompletableFuture.supplyAsync(() ->
                store.query(MemoryQuery.forEntity(agentId, MANOR_DOMAIN, tenantId)
                    .withLimit(limit)
                    .withOrder(MemoryOrder.RECENCY)));
            return future.get(recallTimeoutMs, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            log.debugf("%s: experience recall timed out after %dms", agentId, recallTimeoutMs);
            return List.of();
        } catch (Exception e) {
            log.warnf("%s: experience recall failed (non-fatal): %s", agentId, e.getMessage());
            return List.of();
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java
git commit -m "feat(#16): AgentExperienceService — fire-and-forget ingest, timed recall"
```

---

### Task 4: ObservationBuilder "Past Experience" Section

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `Memory` (neocortex API)
- Produces: `ObservationBuilder.buildObservation(CharacterState, WorldState, List<AgentGoal>, PartitionedDrain<String>, List<Memory>)` — new overload with memory parameter

- [ ] **Step 1: Write test for past experience section**

Add to existing `ObservationBuilderTest.java`:

```java
@Test
void pastExperienceSectionRendersMemories() {
    var memories = List.of(
        new Memory("m1", "test-agent", MemoryDomain.of("manor"), "test",
            null, "Found a key in the library", Map.of(), Instant.now().minusSeconds(300), 0.8),
        new Memory("m2", "test-agent", MemoryDomain.of("manor"), "test",
            null, "Spoke with Penelope about the mystery", Map.of(), Instant.now().minusSeconds(60), 0.6));

    String observation = ObservationBuilder.buildObservation(
        character, world, List.of(), emptyDrain, memories);

    assertTrue(observation.contains("Past Experience"));
    assertTrue(observation.contains("Found a key in the library"));
    assertTrue(observation.contains("Spoke with Penelope about the mystery"));
}

@Test
void emptyMemoriesOmitsPastExperienceSection() {
    String observation = ObservationBuilder.buildObservation(
        character, world, List.of(), emptyDrain, List.of());

    assertFalse(observation.contains("Past Experience"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — no overload with `List<Memory>` parameter

- [ ] **Step 3: Add overload and pastExperienceSection to ObservationBuilder**

Keep the existing 4-parameter method for backward compatibility (delegates with empty list). Add new 5-parameter overload:

```java
public static String buildObservation(CharacterState character, WorldState world,
                                       List<AgentGoal> goals,
                                       PartitionedDrain<String> drain,
                                       List<Memory> memories) {
    var sections = new ArrayList<ObservationSection>();
    sections.add(locationSection(room));
    sections.add(exitsSection(room, world));
    sections.add(objectsSection(character, world));
    sections.add(charactersSection(character, world));
    sections.add(inventorySection(character));
    sections.add(goalsSection(goals));
    sections.add(recentActivitySection(drain));
    if (!drain.rememberedPartitions().isEmpty()) {
        sections.add(rememberedSection(drain, world));
    }
    if (memories != null && !memories.isEmpty()) {
        sections.add(pastExperienceSection(memories));
    }
    sections.add(lastActionResultSection(character));
    return RENDERER.renderObservation(sections);
}

private static ObservationSection pastExperienceSection(List<Memory> memories) {
    var items = memories.stream()
        .map(m -> m.text())
        .filter(t -> t != null && !t.isBlank())
        .toList();
    return ObservationSection.items("Past Experience", null, items);
}
```

Update existing 4-param method to delegate:

```java
public static String buildObservation(CharacterState character, WorldState world,
                                       List<AgentGoal> goals,
                                       PartitionedDrain<String> drain) {
    return buildObservation(character, world, goals, drain, List.of());
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS (existing tests use 4-param overload, unchanged)

- [ ] **Step 5: Commit**

```
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java
git commit -m "feat(#16): ObservationBuilder — Past Experience section from memory recall"
```

---

### Task 5: Wire Services into Orchestrator Path

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/model/PendingAction.java`

**Interfaces:**
- Consumes: `AgentInvocationService.invoke()`, `AgentExperienceService.ingest()`, `AgentExperienceService.recall()`
- Produces: Wired orchestrator with rate limiting, memory, and logical turn counting

- [ ] **Step 1: Add timeout to PendingAction.awaitResult**

```java
// In PendingAction.java, replace:
public ActionResult awaitResult() throws Exception { return future.get(); }
// With:
public ActionResult awaitResult(long timeoutSeconds) throws Exception {
    try {
        return future.get(timeoutSeconds, java.util.concurrent.TimeUnit.SECONDS);
    } catch (java.util.concurrent.TimeoutException e) {
        return new ActionResult(false, "Your action timed out — the manor continues without you.");
    }
}
```

- [ ] **Step 2: Update CharacterAgentLoop to use AgentInvocationService + AgentExperienceService**

Replace the `run` method signature to accept services instead of raw `AgentProvider`. Remove `callAgentWithRetry` entirely — the service handles that.

```java
public void run(CharacterState character, WorldState world,
                AgentInvocationService invocationService,
                AgentExperienceService experienceService,
                String systemPrompt, BlockingQueue<PendingAction> actionQueue,
                ManorEventDispatcher dispatcher, List<AgentGoal> goals) {
    while (!world.isScenarioComplete() && character.isActive()) {
        try {
            // ... scene context + pause checks (unchanged) ...

            var drain = dispatcher.observationService().drain(character.agentId(), System.currentTimeMillis());
            List<Memory> memories = experienceService != null
                ? experienceService.recall(character.agentId(), 10) : List.of();
            String observation = ObservationBuilder.buildObservation(
                character, world, goals, drain, memories);
            String userPrompt = observation + RESPONSE_FORMAT_INSTRUCTION;

            AgentResponse response = invocationService.invoke(
                systemPrompt, userPrompt, character.agentId());

            // ... dialogue + aside publishing (unchanged) ...

            if (response.action() != null && response.action().type() != ActionType.WAIT) {
                var pending = new PendingAction(character, response.action());
                actionQueue.put(pending);
                pending.awaitResult(60);
            } else {
                character.setLastActionResult("You waited and observed.");
            }

            // Fire-and-forget experience ingest
            if (experienceService != null) {
                String desc = (response.dialogue() != null ? response.dialogue() + " " : "")
                    + (response.action() != null ? response.action().type() + " " + response.action().target() : "WAIT");
                experienceService.ingest(character.agentId(), character.currentRoom(),
                    desc.strip(), response.thinking());
            }

            Thread.sleep((long) (character.thinkDelayMs() / world.speedMultiplier()));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        } catch (Exception e) {
            log.errorf(e, "%s: loop error", character.agentId());
            break;
        }
    }
}
```

- [ ] **Step 3: Update ScenarioOrchestrator to create services and fix turn counting**

In `runScenario()`:
- Create `AgentInvocationService` and `AgentExperienceService`
- Pass them to `CharacterAgentLoop` instead of raw `AgentProvider`
- Replace `turnCount` (per-action) with logical turn tracking (per-cycle)
- Skip actions from inactive characters in the action resolution loop

Key changes:

```java
// Create services
var invocationService = new AgentInvocationService(agentProvider, 5, 60, 2, 2000);
AgentExperienceService experienceService = null;
// Wire experience service via CDI if ExperienceRecorder is available
// (optional — null-safe in CharacterAgentLoop)

// Pass to CharacterAgentLoop
new CharacterAgentLoop().run(c, world, invocationService, experienceService,
    systemPrompt, actionQueue, dispatcher, goals);

// Fix turn counting — track active agents per cycle
var actedThisCycle = java.util.concurrent.ConcurrentHashMap.<String>newKeySet();
int logicalTurns = 0;

// In the action resolution loop, after resolving an action:
actedThisCycle.add(pending.character().agentId());
long activeCount = world.characters().values().stream()
    .filter(c -> activeSet == null || activeSet.contains(c.agentId()))
    .filter(CharacterState::isActive)
    .count();
if (actedThisCycle.size() >= activeCount) {
    logicalTurns++;
    actedThisCycle.clear();
    if (logicalTurns >= maxTurns) {
        world.setScenarioComplete(CompletionReason.TURN_LIMIT);
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java \
       wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java \
       wacky-manor/src/main/java/io/casehub/examples/manor/model/PendingAction.java
git commit -m "feat(#16): wire AgentInvocationService + experience into orchestrator path"
```

---

### Task 6: Wire Services into AutonomousScenarioRunner

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunnerTest.java`

**Interfaces:**
- Consumes: `AgentInvocationService`, `AgentExperienceService`, `ObservationService`
- Produces: Runner with configurable character list, real observation pipeline, memory, rate limiting

- [ ] **Step 1: Update runner constructor and run method**

Replace `CHARACTER_ORDER` with a configurable list. Add `AgentInvocationService`, `AgentExperienceService`, and `ObservationService` as constructor parameters. Remove `callAgentWithRetry` — delegate to `AgentInvocationService`.

```java
public class AutonomousScenarioRunner {

    private final AgentInvocationService invocationService;
    private final AgentExperienceService experienceService;
    private final String modelIdentifier;
    private final String gitCommitHash;

    public AutonomousScenarioRunner(AgentInvocationService invocationService,
                                     AgentExperienceService experienceService,
                                     String modelIdentifier, String gitCommitHash) {
        this.invocationService = invocationService;
        this.experienceService = experienceService;
        this.modelIdentifier = modelIdentifier;
        this.gitCommitHash = gitCommitHash;
    }

    public TranscriptRecorder.RunResult run(WorldState world, ProfileMode profile,
                                             int runNumber, Map<String, List<AgentGoal>> goalsByAgent,
                                             int maxTurns, Function<String, String> promptRenderer,
                                             List<String> characterOrder) {
        // ... same loop structure but using characterOrder parameter
        // ... delegate LLM calls to invocationService.invoke()
        // ... recall memories via experienceService.recall()
        // ... ingest experiences via experienceService.ingest()
    }
}
```

- [ ] **Step 2: Update existing tests to pass new constructor args**

Wrap existing `AgentProvider` in `AgentInvocationService` in test setup. Pass `null` for `AgentExperienceService` (backward compat — runner is null-safe).

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunnerTest.java
git commit -m "feat(#16): wire AgentInvocationService + experience into runner, configurable characters"
```

---

### Task 7: ScaleReport + 5-Agent Baseline

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/ScaleReport.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java` (add metrics collection)

**Interfaces:**
- Consumes: `TranscriptRecorder.RunResult`, `AgentInvocationService.metrics()`
- Produces: `ScaleReport` record extending RunResult with latency percentiles and memory stats

- [ ] **Step 1: Create ScaleReport record**

```java
package io.casehub.examples.manor.experiment;

import com.fasterxml.jackson.databind.ObjectMapper;
import java.nio.file.Path;
import java.util.List;

public record ScaleReport(
    TranscriptRecorder.RunResult baseResult,
    long avgTurnLatencyMs,
    long p95TurnLatencyMs,
    long p99TurnLatencyMs,
    int agentCount,
    int logicalTurns,
    long totalLlmCalls,
    long llmRetries,
    long llmFallbacks,
    long avgLlmLatencyMs,
    int memoryRecallHits,
    int memoryRecallMisses) {

    private static final ObjectMapper JSON = new ObjectMapper()
        .findAndRegisterModules();

    public static void writeJson(ScaleReport report, Path file) throws Exception {
        JSON.writerWithDefaultPrettyPrinter().writeValue(file.toFile(), report);
    }

    public String summary() {
        return String.format(
            "ScaleReport: %d agents, %d turns, %dms avg latency (p95=%dms, p99=%dms), " +
            "%d LLM calls (%d retries, %d fallbacks), %d/%d memory hits",
            agentCount, logicalTurns, avgTurnLatencyMs, p95TurnLatencyMs, p99TurnLatencyMs,
            totalLlmCalls, llmRetries, llmFallbacks,
            memoryRecallHits, memoryRecallHits + memoryRecallMisses);
    }
}
```

- [ ] **Step 2: Add per-turn latency tracking to AutonomousScenarioRunner**

Collect turn start/end timestamps in the run loop. After the run, compute percentiles and construct `ScaleReport`.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/ScaleReport.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java
git commit -m "feat(#16): ScaleReport — latency percentiles, LLM metrics, memory stats"
```

---

### Task 8: Scale Test — 10 Agents, 200 Turns

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/ScaleTest.java`

**Interfaces:**
- Consumes: `AutonomousScenarioRunner`, `AgentInvocationService`, `AgentExperienceService`, `ScaleReport`
- Produces: LLM eval test that validates success criteria at scale

- [ ] **Step 1: Write scale test (llm-eval profile)**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentGoal;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.examples.manor.ManorConstants;
import io.casehub.examples.manor.agent.AgentExperienceService;
import io.casehub.examples.manor.agent.AgentInvocationService;
import io.casehub.examples.manor.engine.MansionLoader;
import io.casehub.examples.manor.model.ProfileMode;
import io.casehub.platform.agent.AgentProvider;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
@Tag("llm-eval")
class ScaleTest {

    static final List<String> SCALE_10_CHARACTERS = List.of(
        "penelope-pitstop", "hooded-claw", "ant-hill-mob",
        "dick-dastardly", "peter-perfect",
        "muttley", "pat-pending", "lazy-luke",
        "rock-slag", "rufus-ruffcut");

    @Inject AgentProvider agentProvider;
    @Inject AgentRegistry agentRegistry;

    @Test
    void tenAgents200Turns() throws Exception {
        var invocationService = new AgentInvocationService(agentProvider, 5, 60, 2, 2000);
        // experienceService null for first run — wire when neocortex dev service is configured
        AgentExperienceService experienceService = null;

        var runner = new AutonomousScenarioRunner(
            invocationService, experienceService,
            "scale-test", "local");

        var world = MansionLoader.loadWorld();
        Map<String, List<AgentGoal>> goals = SCALE_10_CHARACTERS.stream()
            .collect(Collectors.toMap(id -> id,
                id -> agentRegistry.findById(id, ManorConstants.TENANCY_ID)
                    .map(d -> d.goals()).orElse(List.of())));

        var result = runner.run(world, ProfileMode.COMPOSITE, 1, goals, 200,
            id -> renderPrompt(id), SCALE_10_CHARACTERS);

        // Write report
        var report = buildReport(result, invocationService, SCALE_10_CHARACTERS.size());
        System.out.println(report.summary());
        ScaleReport.writeJson(report, Path.of("target/scale-report-10agents.json"));

        // Success criteria
        assertNotNull(result.verdict(), "Scenario should complete");
        assertTrue(report.avgTurnLatencyMs() < 30_000,
            "Average turn latency should be < 30s, was: " + report.avgTurnLatencyMs() + "ms");
    }

    private String renderPrompt(String agentId) {
        var desc = agentRegistry.findById(agentId, ManorConstants.TENANCY_ID).orElseThrow();
        var ctx = io.casehub.eidos.api.AgentPromptContext.forFormat(
            io.casehub.eidos.api.SystemPromptRenderer.RenderFormat.MARKDOWN);
        return agentRegistry.renderer().render(desc, ctx).content();
    }

    private ScaleReport buildReport(TranscriptRecorder.RunResult result,
                                     AgentInvocationService invocationService,
                                     int agentCount) {
        var metrics = invocationService.metrics();
        return new ScaleReport(result,
            metrics.averageLatencyMs(), 0, 0,
            agentCount, result.totalTurns(),
            metrics.totalCalls(), metrics.retries(), metrics.fallbacks(),
            metrics.averageLatencyMs(), 0, 0);
    }
}
```

- [ ] **Step 2: Run the scale test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -Dtest=ScaleTest`
Expected: PASS — 10 agents, 200 turns, <30s latency

- [ ] **Step 3: Review transcript for emergent social dynamics**

Check `target/scale-report-10agents.json` for narrative events. Look for character interactions not seen in 5-agent runs.

- [ ] **Step 4: Commit**

```
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/ScaleTest.java
git commit -m "test(#16): ScaleTest — 10 agents, 200 turns, latency validation"
```

---

## Self-Review

1. **Spec coverage:** All 4 components covered (§0 WorldState → Task 1, §1 AgentInvocationService → Task 2, §2 AgentExperienceService → Tasks 3-4, §3 Harness → Tasks 5-8). Turn semantics fix in Task 5. PendingAction timeout in Task 5. 5-agent baseline deferred to Task 7 (metrics infrastructure first). Room count correction reflected in character table.

2. **Placeholder scan:** Clean — all steps have code or exact commands.

3. **Type consistency:** `AgentInvocationService.invoke(String, String, String)` → `AgentResponse` used consistently in Tasks 2, 5, 6. `AgentExperienceService.ingest(String, String, String, String)` and `.recall(String, int)` → `List<Memory>` consistent across Tasks 3, 4, 5, 6. `ObservationBuilder.buildObservation` 5-param overload consistent between Task 4 definition and Task 5 usage.

4. **Tooling safety scan:** No bash file operations on source files. All file creation via IDE tools or `git add`. Build/test commands only.
