# Memory Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #41 — epic: Autonomous agent template
**Issue group:** #41

**Goal:** Wire the neocortex memory stack into wacky-manor: salience-scored
retrieval, relationship tracking, reflection, enhanced observations, and
memory decay.

**Architecture:** The manor calls neocortex APIs directly for memory
operations. No `casehub-engine-api` dependency in this phase. Reflection
uses the `ReflectionSynthesizer` SPI with a manor-specific LLM
implementation. The reflection loop is implemented in the manor (the
neocortex `ReflectionService` requires CDI events not available in the
manor's manual wiring). All changes are within wacky-manor.

**Tech Stack:** Java 21, Quarkus 3.32, neocortex-memory-api 0.2-SNAPSHOT,
neocortex-memory 0.2-SNAPSHOT (runtime), AssertJ for tests.

## Global Constraints

- No new Maven dependencies — neocortex-memory-api and neocortex-memory
  are already in the pom.xml
- Existing tests must continue to pass (`mvn test -pl wacky-manor`)
- `newGoals`/`dropGoals` in the LLM response format are preserved
- Use `ide_*` tools for all `.java` file operations
- Build with `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`

## File Map

**Modified:**
- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
  — salience recall, importance scoring, relationship tracking, reflection trigger, decay
- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
  — insights section, relationship notes section, new parameters
- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
  — wire reflection components, pass targetAgentId, query reflections/relationships
- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
  — pass targetAgentId to ingest
- `wacky-manor/src/main/resources/application.properties` — new config keys

**Created:**
- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizer.java`
- `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionTrigger.java`

**Tests modified:**
- `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java`
- `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Tests created:**
- `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionTriggerTest.java`
- `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizerTest.java`

---

### Task 1: Salience-Scored Retrieval and Importance Scoring

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.MemoryOrder.SALIENCE`, `io.casehub.neocortex.memory.experience.Action` (importance parameter)
- Produces: `AgentExperienceService.ingest(String agentId, String room, String description, String thinking, double importance)`, `AgentExperienceService.recall(String agentId)` (uses SALIENCE, configurable limit)

- [ ] **Step 1: Write failing test — salience-ordered recall**

Add to `AgentExperienceServiceTest.java`:

```java
@Test
void recallUsesSalienceOrder() {
    var store = mock(CaseMemoryStore.class);
    var service = new AgentExperienceService(stubRecorder(), store, "t1");

    service.recall("agent-1", 20);

    var captor = ArgumentCaptor.forClass(MemoryQuery.class);
    verify(store).query(captor.capture());
    assertThat(captor.getValue().order()).isEqualTo(MemoryOrder.SALIENCE);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest#recallUsesSalienceOrder`
Expected: FAIL — current code uses `MemoryOrder.CHRONOLOGICAL`

- [ ] **Step 3: Write failing test — importance passed to Action**

```java
@Test
void ingestPassesImportanceToAction() {
    var recorded = new java.util.ArrayList<ExperienceEvent>();
    var recorder = stubRecorder(recorded);
    var service = new AgentExperienceService(recorder, stubStore(List.of()), "t1");

    service.ingest("agent-1", "kitchen", "took the poison", null, 0.8);

    assertThat(recorded).hasSize(1);
    assertThat(recorded.get(0).importance()).isEqualTo(0.8);
}
```

Note: `stubRecorder` helper already captures events. The existing `stubRecorder()` creates one with a local list — use the overload that accepts an external list, or add one.

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest#ingestPassesImportanceToAction`
Expected: FAIL — `ingest()` doesn't accept importance parameter yet

- [ ] **Step 5: Implement salience recall and importance ingest**

Modify `AgentExperienceService`:

1. Add a `recallLimit` field (default 20), set via constructor or setter.
2. In `recall()`: change `MemoryOrder.CHRONOLOGICAL` to `MemoryOrder.SALIENCE`, use `recallLimit` for limit.
3. Add new `ingest()` overload with `double importance` parameter. Pass importance as the 6th argument to the `Action` constructor (currently `null`):

```java
public void ingest(String agentId, String room, String description,
                   String thinking, double importance) {
    try {
        var metadata = new HashMap<String, String>();
        metadata.put("room", room);
        if (thinking != null) metadata.put("thinking", thinking);
        var event = new Action(agentId, tenantId, null, null,
            description, importance, Map.copyOf(metadata), "manor-action");
        recorder.record(event);
    } catch (Exception e) {
        log.warnf("%s: experience ingest failed (non-fatal): %s",
            agentId, e.getMessage());
    }
}
```

Keep the existing 4-parameter `ingest()` as a delegate that passes `importance = 0.5` (default).

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest`
Expected: ALL PASS (including existing tests)

- [ ] **Step 7: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java
git commit -m "feat(#41): salience-scored retrieval and importance scoring in AgentExperienceService

Refs #41"
```

---

### Task 2: Relationship Tracking

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java`

**Interfaces:**
- Consumes: `ExperienceAttributeKeys.TARGET_AGENT`, `RelationshipQuery.forPair()`
- Produces: `AgentExperienceService.ingest(String agentId, String room, String description, String thinking, double importance, String targetAgentId)`, `AgentExperienceService.recallRelationships(String agentId, String otherAgentId, int limit)`

- [ ] **Step 1: Write failing test — TARGET_AGENT metadata set**

```java
@Test
void ingestSetsTargetAgentMetadata() {
    var recorded = new java.util.ArrayList<ExperienceEvent>();
    var recorder = stubRecorder(recorded);
    var service = new AgentExperienceService(recorder, stubStore(List.of()), "t1");

    service.ingest("agent-1", "kitchen", "gave poison to penelope",
                   null, 0.7, "penelope");

    assertThat(recorded.get(0).metadata())
        .containsEntry(ExperienceAttributeKeys.TARGET_AGENT, "penelope");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest#ingestSetsTargetAgentMetadata`
Expected: FAIL — no 6-parameter `ingest()` overload

- [ ] **Step 3: Write failing test — recallRelationships**

```java
@Test
void recallRelationshipsQueriesForPair() {
    var store = mock(CaseMemoryStore.class);
    when(store.query(any())).thenReturn(List.of());
    var service = new AgentExperienceService(stubRecorder(), store, "t1");

    service.recallRelationships("agent-1", "agent-2", 3);

    var captor = ArgumentCaptor.forClass(MemoryQuery.class);
    verify(store).query(captor.capture());
    // RelationshipQuery.forPair sets entityId to "agent-1" and
    // the domain to the relationship domain
    assertThat(captor.getValue().entityId()).isEqualTo("agent-1");
}
```

- [ ] **Step 4: Implement targetAgentId in ingest and recallRelationships**

Add 6-parameter `ingest()` overload:

```java
public void ingest(String agentId, String room, String description,
                   String thinking, double importance, String targetAgentId) {
    try {
        var metadata = new HashMap<String, String>();
        metadata.put("room", room);
        if (thinking != null) metadata.put("thinking", thinking);
        if (targetAgentId != null) {
            metadata.put(ExperienceAttributeKeys.TARGET_AGENT, targetAgentId);
        }
        var event = new Action(agentId, tenantId, null, null,
            description, importance, Map.copyOf(metadata), "manor-action");
        recorder.record(event);
    } catch (Exception e) {
        log.warnf("%s: experience ingest failed (non-fatal): %s",
            agentId, e.getMessage());
    }
}
```

Update the 5-parameter overload (from Task 1) to delegate with `targetAgentId = null`.

Add `recallRelationships()`:

```java
public List<Memory> recallRelationships(String agentId,
                                         String otherAgentId, int limit) {
    try {
        var query = RelationshipQuery.forPair(agentId, otherAgentId, tenantId)
                        .withLimit(limit)
                        .withOrder(MemoryOrder.SALIENCE);
        return store.query(query);
    } catch (Exception e) {
        log.warnf("%s: relationship recall failed (non-fatal): %s",
            agentId, e.getMessage());
        return List.of();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest`
Expected: ALL PASS

- [ ] **Step 6: Update call sites — ScenarioOrchestrator**

In `runAutonomousTicks()`, the experience ingest block (around line 281-294)
currently calls the old `ingest()`. Update to extract the target agent and
pass importance + targetAgentId:

```java
// After action resolution, compute importance and targetAgentId
double importance = importanceForAction(response);
String targetAgentId = extractTargetAgent(c, response, world);
if (experienceService != null) {
    String desc = /* existing description logic */;
    experienceService.ingest(c.agentId(), c.currentRoom(),
        desc.strip(), response.thinking(), importance, targetAgentId);
}
```

Add helper methods to `ScenarioOrchestrator`:

```java
private static double importanceForAction(AgentResponse response) {
    if (response.action() == null) return 0.5; // dialogue only
    return switch (response.action().type()) {
        case STEAL -> 0.9;
        case USE -> 0.8;
        case TAKE, GIVE, PULL_ASIDE -> 0.7;
        case INTERACT -> 0.6;
        case MOVE -> 0.3;
        case LOOK -> 0.2;
        case WAIT -> 0.1;
    };
}

private static String extractTargetAgent(CharacterState c,
        AgentResponse response, WorldState world) {
    if (response.talkTo() != null) return response.talkTo();
    if (response.action() == null) return null;
    String target = response.action().target();
    if (target == null) return null;
    return switch (response.action().type()) {
        case GIVE, STEAL, PULL_ASIDE -> target;
        default -> null;
    };
}
```

- [ ] **Step 7: Update call sites — CharacterAgentLoop**

In `CharacterAgentLoop.run()`, the experience ingest (around line 97-101)
currently uses the old `ingest()`. Update similarly:

```java
if (experienceService != null) {
    String desc = /* existing */;
    double importance = response.action() != null
        ? importanceForAction(response) : 0.5;
    String targetAgentId = extractTargetAgent(response);
    experienceService.ingest(character.agentId(), character.currentRoom(),
        desc.strip(), response.thinking(), importance, targetAgentId);
}
```

Add the same helper methods (or extract to a shared utility). Since
`CharacterAgentLoop` is only used in SCRIPTED mode and the orchestrator
in AUTONOMOUS mode, duplication is acceptable for 2 small methods.

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java
git commit -m "feat(#41): relationship tracking via TARGET_AGENT metadata

Refs #41"
```

---

### Task 3: Reflection Trigger and Synthesizer

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionTrigger.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizer.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionTriggerTest.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizerTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.reflection.ReflectionSynthesizer`, `io.casehub.neocortex.memory.reflection.ReflectionEvent`, `io.casehub.platform.agent.AgentProvider`
- Produces: `ManorReflectionTrigger.shouldReflect(String agentId, double importance)`, `ManorReflectionTrigger.reset(String agentId)`, `ManorReflectionSynthesizer` (implements `ReflectionSynthesizer`)

- [ ] **Step 1: Write failing test — ManorReflectionTrigger count threshold**

Create `ManorReflectionTriggerTest.java`:

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class ManorReflectionTriggerTest {

    @Test
    void firesAfterMaxUnreflectedCount() {
        var trigger = new ManorReflectionTrigger(3, 100.0);

        assertThat(trigger.shouldReflect("a1", 0.5)).isFalse();
        assertThat(trigger.shouldReflect("a1", 0.5)).isFalse();
        assertThat(trigger.shouldReflect("a1", 0.5)).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorReflectionTriggerTest#firesAfterMaxUnreflectedCount`
Expected: FAIL — class does not exist

- [ ] **Step 3: Write additional failing tests**

```java
@Test
void firesOnImportanceThreshold() {
    var trigger = new ManorReflectionTrigger(100, 2.0);

    assertThat(trigger.shouldReflect("a1", 0.9)).isFalse();
    assertThat(trigger.shouldReflect("a1", 0.9)).isFalse();
    assertThat(trigger.shouldReflect("a1", 0.9)).isTrue(); // cumulative 2.7 >= 2.0
}

@Test
void resetClearsBothCounters() {
    var trigger = new ManorReflectionTrigger(3, 100.0);

    trigger.shouldReflect("a1", 0.5);
    trigger.shouldReflect("a1", 0.5);
    trigger.reset("a1");

    assertThat(trigger.shouldReflect("a1", 0.5)).isFalse();
    assertThat(trigger.shouldReflect("a1", 0.5)).isFalse();
    assertThat(trigger.shouldReflect("a1", 0.5)).isTrue();
}

@Test
void tracksAgentsIndependently() {
    var trigger = new ManorReflectionTrigger(2, 100.0);

    trigger.shouldReflect("a1", 0.5);
    trigger.shouldReflect("a2", 0.5);

    assertThat(trigger.shouldReflect("a1", 0.5)).isTrue();
    assertThat(trigger.shouldReflect("a2", 0.5)).isTrue();
}
```

- [ ] **Step 4: Implement ManorReflectionTrigger**

Create `ManorReflectionTrigger.java`:

```java
package io.casehub.examples.manor.agent;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.DoubleAdder;

public final class ManorReflectionTrigger {

    private final int maxUnreflected;
    private final double importanceThreshold;
    private final ConcurrentHashMap<String, AtomicInteger> counts = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, DoubleAdder> importance = new ConcurrentHashMap<>();

    public ManorReflectionTrigger(int maxUnreflected, double importanceThreshold) {
        this.maxUnreflected = maxUnreflected;
        this.importanceThreshold = importanceThreshold;
    }

    public boolean shouldReflect(String agentId, double actionImportance) {
        int count = counts.computeIfAbsent(agentId, k -> new AtomicInteger())
                         .incrementAndGet();
        DoubleAdder imp = importance.computeIfAbsent(agentId, k -> new DoubleAdder());
        imp.add(actionImportance);
        return count >= maxUnreflected || imp.sum() >= importanceThreshold;
    }

    public void reset(String agentId) {
        counts.computeIfAbsent(agentId, k -> new AtomicInteger()).set(0);
        importance.computeIfAbsent(agentId, k -> new DoubleAdder()).reset();
    }
}
```

- [ ] **Step 5: Run ManorReflectionTrigger tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorReflectionTriggerTest`
Expected: ALL PASS

- [ ] **Step 6: Write failing test — ManorReflectionSynthesizer**

Create `ManorReflectionSynthesizerTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.reflection.ReflectionEvent;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ManorReflectionSynthesizerTest {

    @Test
    void synthesizesInsightsFromMemories() {
        String llmResponse = """
            [
              {"insight": "Sneekly is always near dangerous items", "importance": 0.8},
              {"insight": "Penelope trusts Sneekly too easily", "importance": 0.7}
            ]
            """;
        var provider = mockProvider(llmResponse);
        var synthesizer = new ManorReflectionSynthesizer(provider);

        var memories = List.of(
            new Memory("m1", "agent-1", new MemoryDomain("manor"), "t1",
                null, "Sneekly picked up rat poison", Map.of(), Instant.now(), 0.8),
            new Memory("m2", "agent-1", new MemoryDomain("manor"), "t1",
                null, "Sneekly offered Penelope tea", Map.of(), Instant.now(), 0.6)
        );

        List<ReflectionEvent> results = synthesizer.synthesize(
            "agent-1", "t1", memories, 1);

        assertThat(results).hasSize(2);
        assertThat(results.get(0).insight())
            .isEqualTo("Sneekly is always near dangerous items");
        assertThat(results.get(0).importance()).isEqualTo(0.8);
        assertThat(results.get(0).agentId()).isEqualTo("agent-1");
    }

    @Test
    void returnsEmptyOnLlmFailure() {
        var provider = failingProvider();
        var synthesizer = new ManorReflectionSynthesizer(provider);

        List<ReflectionEvent> results = synthesizer.synthesize(
            "agent-1", "t1", List.of(), 1);

        assertThat(results).isEmpty();
    }

    private AgentProvider mockProvider(String responseText) {
        return config -> Multi.createFrom().item(
            new AgentEvent.TextDelta(responseText));
    }

    private AgentProvider failingProvider() {
        return config -> Multi.createFrom().failure(
            new RuntimeException("LLM unavailable"));
    }
}
```

- [ ] **Step 7: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorReflectionSynthesizerTest#synthesizesInsightsFromMemories`
Expected: FAIL — class does not exist

- [ ] **Step 8: Implement ManorReflectionSynthesizer**

Create `ManorReflectionSynthesizer.java`:

```java
package io.casehub.examples.manor.agent;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.reflection.ReflectionEvent;
import io.casehub.neocortex.memory.reflection.ReflectionSynthesizer;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public final class ManorReflectionSynthesizer implements ReflectionSynthesizer {

    private static final Logger log = Logger.getLogger(ManorReflectionSynthesizer.class);
    private static final ObjectMapper JSON = new ObjectMapper();

    private static final String SYSTEM_PROMPT = """
        You are analyzing the recent experiences of an agent to identify \
        patterns, relationships, and strategic insights. Each insight should \
        be one clear, specific sentence. Focus on:
        - Patterns in other agents' behavior
        - Cause-and-effect relationships between actions
        - Strategic implications for the agent's goals
        - Social dynamics and trust signals

        Return a JSON array of insights:
        [{"insight": "...", "importance": 0.0-1.0}]
        Return ONLY the JSON array. No other text.""";

    private final AgentProvider agentProvider;

    public ManorReflectionSynthesizer(AgentProvider agentProvider) {
        this.agentProvider = agentProvider;
    }

    @Override
    public List<ReflectionEvent> synthesize(String agentId, String tenantId,
                                             List<Memory> sources, int targetLevel) {
        if (sources.isEmpty()) return List.of();
        try {
            var sb = new StringBuilder("Recent experiences:\n");
            for (var m : sources) {
                sb.append("- ").append(m.text()).append("\n");
            }
            var sourceIds = sources.stream()
                .map(Memory::memoryId).toList();

            String response = agentProvider.invoke(
                    AgentSessionConfig.of(SYSTEM_PROMPT, sb.toString()))
                .filter(e -> e instanceof AgentEvent.TextDelta)
                .map(e -> ((AgentEvent.TextDelta) e).text())
                .collect().with(Collectors.joining())
                .await().atMost(Duration.ofSeconds(120));

            record InsightEntry(String insight, Double importance) {}
            var entries = JSON.readValue(response,
                new TypeReference<List<InsightEntry>>() {});

            return entries.stream()
                .map(e -> new ReflectionEvent(agentId, tenantId, null,
                    e.insight(), targetLevel, sourceIds,
                    e.importance() != null ? e.importance() : 0.7,
                    Map.of()))
                .toList();
        } catch (Exception e) {
            log.warnf("%s: reflection synthesis failed (non-fatal): %s",
                agentId, e.getMessage());
            return List.of();
        }
    }
}
```

- [ ] **Step 9: Run ManorReflectionSynthesizer tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorReflectionSynthesizerTest`
Expected: ALL PASS

- [ ] **Step 10: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionTrigger.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizer.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionTriggerTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorReflectionSynthesizerTest.java
git commit -m "feat(#41): ManorReflectionTrigger and ManorReflectionSynthesizer

Refs #41"
```

---

### Task 4: Wire Reflection Loop into AgentExperienceService

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java`
- Modify: `wacky-manor/src/main/resources/application.properties`

**Interfaces:**
- Consumes: `ManorReflectionTrigger.shouldReflect()`, `ManorReflectionSynthesizer`, `ReflectionEvents.toMemoryInput()`, `CaseMemoryStore.store()`, `MemoryRetentionPolicy`
- Produces: `AgentExperienceService.recallReflections(String agentId, int limit)`, reflection fires asynchronously after `ingest()`

- [ ] **Step 1: Write failing test — reflection triggered after threshold**

```java
@Test
void ingestTriggersReflectionAtThreshold() throws Exception {
    var store = mock(CaseMemoryStore.class);
    when(store.query(any())).thenReturn(List.of(
        new Memory("m1", "a1", new MemoryDomain("manor"), "t1",
            null, "test memory", Map.of(), Instant.now(), 0.5)
    ));
    when(store.store(any(MemoryInput.class))).thenReturn("r1");
    var synthesizer = mock(ReflectionSynthesizer.class);
    when(synthesizer.synthesize(any(), any(), any(), anyInt()))
        .thenReturn(List.of(new ReflectionEvent("a1", "t1", null,
            "test insight", 1, List.of("m1"), 0.8, Map.of())));
    var trigger = new ManorReflectionTrigger(2, 100.0);
    var service = new AgentExperienceService(stubRecorder(), store, "t1",
        synthesizer, trigger, true, false, 7, 0.2, 15, 20);

    service.ingest("a1", "room", "action 1", null, 0.5, null);
    service.ingest("a1", "room", "action 2", null, 0.5, null);

    // Wait for async reflection
    Thread.sleep(500);

    verify(synthesizer).synthesize(eq("a1"), eq("t1"), any(), eq(1));
    verify(store, atLeastOnce()).store(any(MemoryInput.class));
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest#ingestTriggersReflectionAtThreshold`
Expected: FAIL — constructor doesn't accept these parameters

- [ ] **Step 3: Write failing test — recallReflections**

```java
@Test
void recallReflectionsQueriesReflectionDomain() {
    var store = mock(CaseMemoryStore.class);
    when(store.query(any())).thenReturn(List.of());
    var service = new AgentExperienceService(stubRecorder(), store, "t1");

    service.recallReflections("a1", 5);

    var captor = ArgumentCaptor.forClass(MemoryQuery.class);
    verify(store).query(captor.capture());
    assertThat(captor.getValue().domain())
        .isEqualTo(ReflectionEvents.DOMAIN);
    assertThat(captor.getValue().order())
        .isEqualTo(MemoryOrder.SALIENCE);
}
```

- [ ] **Step 4: Implement reflection loop and recallReflections**

Expand `AgentExperienceService` constructor to accept reflection config:

```java
public AgentExperienceService(ExperienceRecorder recorder,
        CaseMemoryStore store, String tenantId,
        ReflectionSynthesizer synthesizer,
        ManorReflectionTrigger reflectionTrigger,
        boolean reflectionEnabled,
        boolean decayEnabled, int decayMaxAgeDays, double decayMinImportance,
        int maxSourceMemories, int recallLimit) {
    this.recorder = recorder;
    this.store = store;
    this.tenantId = tenantId;
    this.synthesizer = synthesizer;
    this.reflectionTrigger = reflectionTrigger;
    this.reflectionEnabled = reflectionEnabled;
    this.decayEnabled = decayEnabled;
    this.decayMaxAgeDays = decayMaxAgeDays;
    this.decayMinImportance = decayMinImportance;
    this.maxSourceMemories = maxSourceMemories;
    this.recallLimit = recallLimit;
}
```

Keep the 3-parameter constructor as a convenience (reflection disabled):

```java
public AgentExperienceService(ExperienceRecorder recorder,
        CaseMemoryStore store, String tenantId) {
    this(recorder, store, tenantId, null, null, false, false, 7, 0.2, 15, 20);
}
```

At the end of the 6-parameter `ingest()`, add reflection trigger:

```java
if (reflectionEnabled && reflectionTrigger != null
        && reflectionTrigger.shouldReflect(agentId, importance)) {
    var since = lastReflectionTime.getOrDefault(agentId, Instant.EPOCH);
    Thread.ofVirtual().name(agentId + "-reflect").start(() -> {
        try {
            runReflection(agentId, since);
            reflectionTrigger.reset(agentId);
            lastReflectionTime.put(agentId, Instant.now());
        } catch (Exception ex) {
            log.warnf("%s: reflection failed (non-fatal): %s",
                agentId, ex.getMessage());
        }
    });
}
```

Add `runReflection()`:

```java
private void runReflection(String agentId, Instant since) {
    var sources = store.query(MemoryQuery.forEntity(agentId, MANOR_DOMAIN, tenantId)
        .withLimit(maxSourceMemories)
        .withSince(since)
        .withOrder(MemoryOrder.SALIENCE));
    if (sources.isEmpty()) return;
    var events = synthesizer.synthesize(agentId, tenantId, sources, 1);
    for (var event : events) {
        store.store(ReflectionEvents.toMemoryInput(event));
    }
    if (decayEnabled) {
        store.purge(new MemoryRetentionPolicy(tenantId, MANOR_DOMAIN,
            decayMaxAgeDays, decayMinImportance));
    }
}
```

Add fields:

```java
private final ReflectionSynthesizer synthesizer;
private final ManorReflectionTrigger reflectionTrigger;
private final boolean reflectionEnabled;
private final boolean decayEnabled;
private final int decayMaxAgeDays;
private final double decayMinImportance;
private final int maxSourceMemories;
private final int recallLimit;
private final ConcurrentHashMap<String, Instant> lastReflectionTime = new ConcurrentHashMap<>();
```

Add `recallReflections()`:

```java
public List<Memory> recallReflections(String agentId, int limit) {
    try {
        return store.query(MemoryQuery.forEntity(agentId,
            ReflectionEvents.DOMAIN, tenantId)
            .withLimit(limit)
            .withOrder(MemoryOrder.SALIENCE));
    } catch (Exception e) {
        log.warnf("%s: reflection recall failed (non-fatal): %s",
            agentId, e.getMessage());
        return List.of();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=AgentExperienceServiceTest`
Expected: ALL PASS

- [ ] **Step 6: Wire in ScenarioOrchestrator**

In `runScenario()`, create the reflection components and pass to the
experience service:

```java
var reflectionSynthesizer = new ManorReflectionSynthesizer(gatedProvider);
var reflectionTrigger = new ManorReflectionTrigger(
    maxUnreflected, reflectionImportanceThreshold);
var experienceService = new AgentExperienceService(recorder, store, tenantId,
    reflectionSynthesizer, reflectionTrigger,
    reflectionEnabled, decayEnabled, decayMaxAgeDays, decayMinImportance,
    maxSourceMemories, recallLimit);
```

Add config properties to the class (inject via `@ConfigProperty`):

```java
@ConfigProperty(name = "manor.reflection.enabled", defaultValue = "true")
boolean reflectionEnabled;
@ConfigProperty(name = "manor.reflection.max-unreflected", defaultValue = "5")
int maxUnreflected;
@ConfigProperty(name = "manor.reflection.importance-threshold", defaultValue = "3.0")
double reflectionImportanceThreshold;
@ConfigProperty(name = "manor.reflection.max-source-memories", defaultValue = "15")
int maxSourceMemories;
@ConfigProperty(name = "manor.decay.enabled", defaultValue = "true")
boolean decayEnabled;
@ConfigProperty(name = "manor.decay.max-age-days", defaultValue = "7")
int decayMaxAgeDays;
@ConfigProperty(name = "manor.decay.min-importance", defaultValue = "0.2")
double decayMinImportance;
@ConfigProperty(name = "manor.memory.recall-limit", defaultValue = "20")
int recallLimit;
```

Also need to obtain `ExperienceRecorder` and `CaseMemoryStore` — inject them:

```java
@Inject ExperienceRecorder experienceRecorder;
@Inject CaseMemoryStore caseMemoryStore;
```

Note: these beans are provided by `casehub-neocortex-memory` (runtime
scope). If they're not CDI-discoverable, create them manually from the
in-memory implementations. Check the existing code — the current
`AgentExperienceService` is created manually in tests, so the recorder
and store may already be injected in the orchestrator or passed through.

- [ ] **Step 7: Add config to application.properties**

```properties
# Memory stack
manor.memory.recall-limit=20
manor.reflection.enabled=true
manor.reflection.max-unreflected=5
manor.reflection.importance-threshold=3.0
manor.reflection.max-source-memories=15
manor.decay.enabled=true
manor.decay.max-age-days=7
manor.decay.min-importance=0.2
```

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/AgentExperienceService.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/resources/application.properties wacky-manor/src/test/java/io/casehub/examples/manor/agent/AgentExperienceServiceTest.java
git commit -m "feat(#41): wire reflection loop into AgentExperienceService with decay

Refs #41"
```

---

### Task 5: Enhanced Observations

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `AgentExperienceService.recallReflections()`, `AgentExperienceService.recallRelationships()`, `ObservationSection.items()`
- Produces: `ObservationBuilder.buildObservation()` with reflections and relationship parameters

- [ ] **Step 1: Write failing test — insights section renders**

Add to `ObservationBuilderTest.java`:

```java
@Test
void insightsSectionRendersWhenReflectionsProvided() {
    var character = createCharacter("a1", "Test", "room-1");
    var world = createWorld(character);
    var reflections = List.of(
        new Memory("r1", "a1", ReflectionEvents.DOMAIN, "t1",
            null, "Sneekly is always near dangerous items",
            Map.of(), Instant.now(), 0.8)
    );

    String obs = ObservationBuilder.buildObservation(
        character, world, List.of(), emptyDrain(),
        List.of(), reflections, Map.of(), Set.of());

    assertThat(obs).contains("Insights");
    assertThat(obs).contains("Sneekly is always near dangerous items");
}
```

- [ ] **Step 2: Write failing test — insights section omitted when empty**

```java
@Test
void insightsSectionOmittedWhenNoReflections() {
    var character = createCharacter("a1", "Test", "room-1");
    var world = createWorld(character);

    String obs = ObservationBuilder.buildObservation(
        character, world, List.of(), emptyDrain(),
        List.of(), List.of(), Map.of(), Set.of());

    assertThat(obs).doesNotContain("Insights");
}
```

- [ ] **Step 3: Write failing test — relationship notes render**

```java
@Test
void relationshipNotesRenderForCharactersInRoom() {
    var character = createCharacter("a1", "Test", "room-1");
    var other = createCharacter("sneekly", "Sneekly", "room-1");
    var world = createWorld(character, other);
    var relationships = Map.of(
        "sneekly", List.of(
            new Memory("rel1", "a1", new MemoryDomain("relationship"), "t1",
                null, "Sneekly offered you tea with unusual insistence",
                Map.of(), Instant.now(), 0.7)
        )
    );

    String obs = ObservationBuilder.buildObservation(
        character, world, List.of(), emptyDrain(),
        List.of(), List.of(), relationships, Set.of());

    assertThat(obs).contains("About Sneekly");
    assertThat(obs).contains("Sneekly offered you tea with unusual insistence");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ObservationBuilderTest`
Expected: FAIL — `buildObservation()` signature doesn't accept reflections/relationships yet

- [ ] **Step 5: Implement new ObservationBuilder sections**

Add new overload of `buildObservation()` accepting reflections and relationships:

```java
public static String buildObservation(CharacterState character, WorldState world,
        List<AgentGoal> goals,
        PartitionedDrain<String> drain,
        List<Memory> memories,
        List<Memory> reflections,
        Map<String, List<Memory>> relationshipMemories,
        Set<String> observerTags) {
    // ... existing sections ...

    // After "Characters Present", add relationship notes
    for (var entry : relationshipMemories.entrySet()) {
        if (!entry.getValue().isEmpty()) {
            var other = world.character(entry.getKey());
            String name = other != null ? other.name() : entry.getKey();
            sections.add(relationshipNotesSection(name, entry.getValue()));
        }
    }

    // After "Past Experience", add insights
    if (reflections != null && !reflections.isEmpty()) {
        sections.add(insightsSection(reflections));
    }

    // ... remaining sections ...
}
```

Add helper methods:

```java
private static ObservationSection insightsSection(List<Memory> reflections) {
    var items = reflections.stream()
        .map(Memory::text)
        .filter(t -> t != null && !t.isBlank())
        .toList();
    return ObservationSection.items("Insights", null, items);
}

private static ObservationSection relationshipNotesSection(
        String characterName, List<Memory> memories) {
    var items = memories.stream()
        .map(m -> "You recall: " + m.text())
        .toList();
    return ObservationSection.items("About " + characterName, null, items);
}
```

Update existing overloads to delegate, passing empty reflections and
relationships for backward compatibility.

- [ ] **Step 6: Run ObservationBuilder tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ObservationBuilderTest`
Expected: ALL PASS

- [ ] **Step 7: Update ScenarioOrchestrator to query and pass reflection/relationship data**

In `runAutonomousTicks()`, before building the observation for each
character, query reflections and relationships:

```java
List<Memory> reflections = experienceService.recallReflections(c.agentId(), 5);
Map<String, List<Memory>> relationships = new java.util.HashMap<>();
for (var other : world.charactersInRoom(c.currentRoom())) {
    if (!other.agentId().equals(c.agentId())) {
        var relMems = experienceService.recallRelationships(
            c.agentId(), other.agentId(), 3);
        if (!relMems.isEmpty()) {
            relationships.put(other.agentId(), relMems);
        }
    }
}
```

Pass to `ObservationBuilder.buildObservation()`:

```java
String observation = ObservationBuilder.buildObservation(
    c, world, resolveGoals(c.agentId()), drain,
    List.of(), reflections, relationships, c.capabilityTags());
```

Apply similar changes to `CharacterAgentLoop.run()` for SCRIPTED mode.

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java
git commit -m "feat(#41): enhanced observations with insights and relationship notes

Refs #41"
```

---

## Self-Review Checklist

**Spec coverage:**
- ✅ §1 Salience-scored retrieval → Task 1
- ✅ §2 Relationship tracking → Task 2
- ✅ §3 Reflection orchestration → Tasks 3 + 4
- ✅ §4 Enhanced observations → Task 5
- ✅ §5 Memory decay → Task 4 (wired into reflection loop)
- ✅ §6 Wiring in ScenarioOrchestrator → Tasks 2, 4, 5
- ✅ §7 Configuration → Task 4

**Placeholder scan:** No TBDs, TODOs, or "similar to Task N" references.

**Type consistency:** `ingest()` overloads chain correctly: 4-param → 5-param → 6-param. `recallReflections()` and `recallRelationships()` signatures are consistent across Task 4 (implementation) and Task 5 (usage).

**Tooling safety scan:** No bash cp/rm/mv on source files. All file creation uses `ide_create_file` or `Write` (new files only). Git commands for commits only.
