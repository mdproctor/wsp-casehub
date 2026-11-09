# Cognitive State Dump Endpoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #520 — feat: cognitive state dump endpoint — end-to-end pipeline tracing
**Issue group:** #820 (branch covers)

**Goal:** Add a @PlatformQuery endpoint that dumps the full cognitive pipeline state — live or historical — for any agent/tenant, enabling postmortem tracing from drives through CAPS to appraisal to rendered prompt sections.

**Architecture:** CognitionStateProvider SPI in cognitive-observability-api, implemented by CognitionCore. PipelineTrace sub-record groups new pipeline fields. PipelineSnapshotStore (interface + SQLite + in-memory) persists per-tick snapshots. Capture at end of CognitionCore.tick(). CAPS settling data cached by BehavioralSynthesisPhase (consolidation-time, not per-tick).

**Tech Stack:** Java 26, Quarkus (CDI), SQLite (JDBC), JUnit 5

## Global Constraints

- All new types in `io.casehub.neocortex.cognitive.observability` package (API types in `-api` module, service types in `-core`)
- Follow existing `SnapshotStore` / `SqliteSnapshotStore` / `InMemorySnapshotStore` / `SnapshotStoreContractTest` patterns exactly
- `@PlatformQuery` on `CognitionApi` interface, implementation in `CognitionService`
- CDI wiring via `CognitionBeans` using `Instance<>` for optional dependencies
- CapsEngine.settle() is called from `BehavioralSynthesisPhase` (consolidation), NOT from `CognitionCore.tick()` — CAPS data reflects last consolidation, not current tick
- Use `ide-tooling` for all structural code operations (insert/replace member, rename, move)
- CognitionSnapshot is in the `cognition` module, PipelineTrace is in `cognitive-observability-api`  — the cognition module depends on cognitive-observability-api to implement the SPI

---

## Batch 1: Foundation — Types, SPI, and Store Contract

### Task 1: PipelineTrace record + CognitionStateProvider SPI + extend CognitionSnapshot

**Files:**
- Create: `cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineTrace.java`
- Create: `cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/CognitionStateProvider.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionSnapshot.java:17-35` — add PipelineTrace field
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionDelta.java` — add PipelineTrace delta support

**Interfaces:**
- Produces: `PipelineTrace` record with fields: appraisalResult, concernSalience, capsInputActivations, capsSettlingResult, regulationResult, channelWeights, habituation, activatedDriveCategories, promptSections
- Produces: `CognitionStateProvider` SPI with: `currentSnapshot(agentId, tenantId)`, `currentTrace(agentId, tenantId)`, `activeAgents(tenantId)`
- Produces: Extended `CognitionSnapshot` with `@Nullable PipelineTrace trace` field

**Prerequisite — verify module dependency:** Before writing code, check that the `cognition` module's pom.xml depends on `cognitive-observability-api`. If not, add the dependency. CognitionSnapshot (in cognition) must reference PipelineTrace (in cognitive-observability-api).

- [ ] **Step 1: Verify module dependency**

Run: check `cognition/pom.xml` for a dependency on `cognitive-observability-api`. If missing:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognitive-observability-api</artifactId>
</dependency>
```
Check exact artifactId by reading `cognitive-observability-api/pom.xml`.

- [ ] **Step 2: Write PipelineTrace record**

Create `cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineTrace.java`:

```java
package io.casehub.neocortex.cognitive.observability;

import io.casehub.neocortex.caps.SettlingResult;
import io.casehub.neocortex.cognition.appraisal.AppraisalResult;
import io.casehub.neocortex.cognition.appraisal.ChannelWeightProfile;
import io.casehub.neocortex.cognition.appraisal.HabituationState;
import io.casehub.neocortex.cognition.regulation.EmotionRegulationResult;
import org.jspecify.annotations.Nullable;

import java.util.List;
import java.util.Map;

public record PipelineTrace(
        @Nullable AppraisalResult appraisalResult,
        Map<String, Double> concernSalience,
        Map<String, Double> capsInputActivations,
        @Nullable SettlingResult capsSettlingResult,
        @Nullable EmotionRegulationResult regulationResult,
        @Nullable ChannelWeightProfile channelWeights,
        @Nullable HabituationState habituation,
        List<String> activatedDriveCategories,
        Map<String, String> promptSections
) {}
```

**Import verification:** The imports above reference types from caps-api, cognition-api. Verify `cognitive-observability-api/pom.xml` has these dependencies. Use `ide_find_class` to confirm each type's qualified name matches.

- [ ] **Step 3: Write CognitionStateProvider SPI**

Create `cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/CognitionStateProvider.java`:

```java
package io.casehub.neocortex.cognitive.observability;

import io.casehub.neocortex.cognition.core.CognitionSnapshot;

import java.util.List;
import java.util.Optional;

public interface CognitionStateProvider {
    Optional<CognitionSnapshot> currentSnapshot(String agentId, String tenantId);
    Optional<PipelineTrace> currentTrace(String agentId, String tenantId);
    List<String> activeAgents(String tenantId);
}
```

**Import verification:** `CognitionSnapshot` is in the `cognition` module. cognitive-observability-api must NOT depend on cognition (that would create a cycle). If CognitionSnapshot is not accessible, the SPI must use a different approach — define a parallel `PipelineSnapshot` type in -api, or move CognitionSnapshot to -api. Check the dependency graph with pom.xml before proceeding. If CognitionSnapshot can't be imported, return `PipelineTrace` from `currentSnapshot()` and let the caller compose it with other state.

- [ ] **Step 4: Extend CognitionSnapshot with PipelineTrace**

Use `ide_replace_member` to modify the CognitionSnapshot record at `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionSnapshot.java:17`. Add `@Nullable PipelineTrace trace` as the last component:

```java
public record CognitionSnapshot(
        String agentId,
        String tenantId,
        int turnNumber,
        Instant capturedAt,
        @Nullable MoodState mood,
        @Nullable DriveProfile drives,
        Map<String, MentalModelSnapshot> mentalModels,
        Map<String, UserProfile> userProfiles,
        @Nullable StrategyProfile strategy,
        @Nullable NarrativeState narrative,
        List<DriveGoalProposal> goalProposals,
        @Nullable AttentionBriefing lastBriefing,
        @Nullable PipelineTrace trace
) {
    public CognitionDelta diffFrom(@Nullable CognitionSnapshot previous) {
        return CognitionDelta.compute(previous, this);
    }
}
```

Add import: `import io.casehub.neocortex.cognitive.observability.PipelineTrace;`

- [ ] **Step 5: Fix all CognitionSnapshot construction sites**

Adding a field to a record breaks all existing construction sites. Use `ide_find_references` on `CognitionSnapshot` constructor to find every call site. Each must add `null` (or a PipelineTrace) as the last argument.

Run: `ide_find_references` on `CognitionSnapshot` constructor. For each call site, add `, null` as the trace argument using `ide_replace_member` or direct edit.

- [ ] **Step 6: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml compile -s .mvn/slot-settings.xml -q`

Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineTrace.java
git add cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/CognitionStateProvider.java
git add cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionSnapshot.java
git commit -m "feat(#520): PipelineTrace record + CognitionStateProvider SPI + extend CognitionSnapshot"
```

---

### Task 2: PipelineSnapshotStore interface + InMemoryPipelineSnapshotStore + contract test

**Files:**
- Create: `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineSnapshotStore.java`
- Create: `cognitive-observability-testing/src/main/java/io/casehub/neocortex/cognitive/observability/testing/InMemoryPipelineSnapshotStore.java`
- Create: `cognitive-observability-testing/src/main/java/io/casehub/neocortex/cognitive/observability/testing/PipelineSnapshotStoreContractTest.java`
- Create: `cognitive-observability-testing/src/test/java/io/casehub/neocortex/cognitive/observability/testing/InMemoryPipelineSnapshotStoreTest.java`

**Interfaces:**
- Consumes: `CognitionSnapshot` (from Task 1, with PipelineTrace field)
- Produces: `PipelineSnapshotStore` interface with: `store(CognitionSnapshot)`, `find(tenantId, agentId, fromTick, toTick)`, `findByTick(tenantId, agentId, tick)`, `findLatest(tenantId, agentId, limit)`, `count(tenantId)`, `purge(Instant olderThan)`

- [ ] **Step 1: Write the PipelineSnapshotStore interface**

Create `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineSnapshotStore.java`:

```java
package io.casehub.neocortex.cognitive.observability;

import io.casehub.neocortex.cognition.core.CognitionSnapshot;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

public interface PipelineSnapshotStore {
    void store(CognitionSnapshot snapshot);
    List<CognitionSnapshot> find(String tenantId, String agentId, int fromTick, int toTick);
    Optional<CognitionSnapshot> findByTick(String tenantId, String agentId, int tick);
    List<CognitionSnapshot> findLatest(String tenantId, String agentId, int limit);
    long count(String tenantId);
    void purge(Instant olderThan);
}
```

- [ ] **Step 2: Write the contract test**

Create `cognitive-observability-testing/src/main/java/io/casehub/neocortex/cognitive/observability/testing/PipelineSnapshotStoreContractTest.java`:

Follow the exact pattern from `SnapshotStoreContractTest`: abstract class with `protected abstract PipelineSnapshotStore createStore()`, `@BeforeEach` creates the store, individual `@Test` methods.

```java
package io.casehub.neocortex.cognitive.observability.testing;

import io.casehub.neocortex.cognitive.observability.PipelineSnapshotStore;
import io.casehub.neocortex.cognition.core.CognitionSnapshot;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

public abstract class PipelineSnapshotStoreContractTest {

    protected abstract PipelineSnapshotStore createStore();

    private PipelineSnapshotStore store;

    @BeforeEach
    void setUp() {
        store = createStore();
    }

    // Helper to create a minimal CognitionSnapshot for testing
    protected CognitionSnapshot snapshot(String agentId, String tenantId, int tick) {
        return new CognitionSnapshot(
                agentId, tenantId, tick, Instant.now(),
                null, null, Map.of(), Map.of(),
                null, null, List.of(), null, null);
    }

    @Test
    void shouldStoreAndFindByTick() {
        store.store(snapshot("agent1", "t1", 5));
        var result = store.findByTick("t1", "agent1", 5);
        assertTrue(result.isPresent());
        assertEquals("agent1", result.get().agentId());
        assertEquals(5, result.get().turnNumber());
    }

    @Test
    void shouldFindByTickRange() {
        store.store(snapshot("agent1", "t1", 1));
        store.store(snapshot("agent1", "t1", 2));
        store.store(snapshot("agent1", "t1", 3));
        store.store(snapshot("agent1", "t1", 5));

        var results = store.find("t1", "agent1", 2, 4);
        assertEquals(2, results.size());
    }

    @Test
    void shouldFilterByAgentAndTenant() {
        store.store(snapshot("agent1", "t1", 1));
        store.store(snapshot("agent2", "t1", 1));
        store.store(snapshot("agent1", "t2", 1));

        var results = store.find("t1", "agent1", 0, 10);
        assertEquals(1, results.size());
    }

    @Test
    void shouldFindLatest() {
        store.store(snapshot("agent1", "t1", 1));
        store.store(snapshot("agent1", "t1", 5));
        store.store(snapshot("agent1", "t1", 3));

        var results = store.findLatest("t1", "agent1", 2);
        assertEquals(2, results.size());
        assertEquals(5, results.get(0).turnNumber());
        assertEquals(3, results.get(1).turnNumber());
    }

    @Test
    void shouldCountByTenant() {
        store.store(snapshot("agent1", "t1", 1));
        store.store(snapshot("agent2", "t1", 1));
        store.store(snapshot("agent1", "t2", 1));

        assertEquals(2, store.count("t1"));
        assertEquals(1, store.count("t2"));
    }

    @Test
    void shouldPurgeOlderThan() {
        store.store(snapshot("agent1", "t1", 1));
        store.store(snapshot("agent1", "t1", 2));

        store.purge(Instant.now().plusSeconds(1));

        assertEquals(0, store.count("t1"));
    }

    @Test
    void shouldReturnEmptyForMissingTick() {
        var result = store.findByTick("t1", "agent1", 99);
        assertTrue(result.isEmpty());
    }

    @Test
    void shouldHandleDuplicateTickSilently() {
        store.store(snapshot("agent1", "t1", 1));
        store.store(snapshot("agent1", "t1", 1)); // duplicate — should upsert or ignore
        assertEquals(1, store.find("t1", "agent1", 0, 10).size());
    }
}
```

- [ ] **Step 3: Write InMemoryPipelineSnapshotStore**

Create `cognitive-observability-testing/src/main/java/io/casehub/neocortex/cognitive/observability/testing/InMemoryPipelineSnapshotStore.java`:

```java
package io.casehub.neocortex.cognitive.observability.testing;

import io.casehub.neocortex.cognitive.observability.PipelineSnapshotStore;
import io.casehub.neocortex.cognition.core.CognitionSnapshot;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

public class InMemoryPipelineSnapshotStore implements PipelineSnapshotStore {

    private final Map<String, List<CognitionSnapshot>> snapshots = new ConcurrentHashMap<>();

    private String key(String tenantId, String agentId) {
        return tenantId + ":" + agentId;
    }

    @Override
    public void store(CognitionSnapshot snapshot) {
        snapshots.compute(key(snapshot.tenantId(), snapshot.agentId()), (k, list) -> {
            if (list == null) list = new ArrayList<>();
            list.removeIf(s -> s.turnNumber() == snapshot.turnNumber());
            list.add(snapshot);
            return list;
        });
    }

    @Override
    public List<CognitionSnapshot> find(String tenantId, String agentId, int fromTick, int toTick) {
        return snapshots.getOrDefault(key(tenantId, agentId), List.of()).stream()
                .filter(s -> s.turnNumber() >= fromTick && s.turnNumber() <= toTick)
                .sorted(Comparator.comparingInt(CognitionSnapshot::turnNumber))
                .toList();
    }

    @Override
    public Optional<CognitionSnapshot> findByTick(String tenantId, String agentId, int tick) {
        return snapshots.getOrDefault(key(tenantId, agentId), List.of()).stream()
                .filter(s -> s.turnNumber() == tick)
                .findFirst();
    }

    @Override
    public List<CognitionSnapshot> findLatest(String tenantId, String agentId, int limit) {
        return snapshots.getOrDefault(key(tenantId, agentId), List.of()).stream()
                .sorted(Comparator.comparingInt(CognitionSnapshot::turnNumber).reversed())
                .limit(limit)
                .toList();
    }

    @Override
    public long count(String tenantId) {
        return snapshots.entrySet().stream()
                .filter(e -> e.getKey().startsWith(tenantId + ":"))
                .mapToLong(e -> e.getValue().size())
                .sum();
    }

    @Override
    public void purge(Instant olderThan) {
        snapshots.values().forEach(list -> list.removeIf(s -> s.capturedAt().isBefore(olderThan)));
        snapshots.entrySet().removeIf(e -> e.getValue().isEmpty());
    }
}
```

- [ ] **Step 4: Write InMemoryPipelineSnapshotStoreTest**

Create `cognitive-observability-testing/src/test/java/io/casehub/neocortex/cognitive/observability/testing/InMemoryPipelineSnapshotStoreTest.java`:

```java
package io.casehub.neocortex.cognitive.observability.testing;

import io.casehub.neocortex.cognitive.observability.PipelineSnapshotStore;

public class InMemoryPipelineSnapshotStoreTest extends PipelineSnapshotStoreContractTest {
    @Override
    protected PipelineSnapshotStore createStore() {
        return new InMemoryPipelineSnapshotStore();
    }
}
```

- [ ] **Step 5: Run contract tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-observability-testing/pom.xml test -Dtest=InMemoryPipelineSnapshotStoreTest -s .mvn/slot-settings.xml`

Expected: All 8 contract tests pass.

- [ ] **Step 6: Commit**

```bash
git add cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineSnapshotStore.java
git add cognitive-observability-testing/src/main/java/io/casehub/neocortex/cognitive/observability/testing/InMemoryPipelineSnapshotStore.java
git add cognitive-observability-testing/src/main/java/io/casehub/neocortex/cognitive/observability/testing/PipelineSnapshotStoreContractTest.java
git add cognitive-observability-testing/src/test/java/io/casehub/neocortex/cognitive/observability/testing/InMemoryPipelineSnapshotStoreTest.java
git commit -m "feat(#520): PipelineSnapshotStore interface + InMemoryPipelineSnapshotStore + contract tests"
```

---

## Batch 2: Storage + Capture

### Task 3: SqlitePipelineSnapshotStore

**Files:**
- Create: `cognitive-observability-sqlite/src/main/java/io/casehub/neocortex/cognitive/observability/sqlite/SqlitePipelineSnapshotStore.java`
- Create: `cognitive-observability-sqlite/src/test/java/io/casehub/neocortex/cognitive/observability/sqlite/SqlitePipelineSnapshotStoreTest.java`

**Interfaces:**
- Consumes: `PipelineSnapshotStore` interface (from Task 2), `CognitionSnapshot` (from Task 1)
- Produces: SQLite-backed implementation of PipelineSnapshotStore

- [ ] **Step 1: Read SqliteSnapshotStore for patterns**

Read `cognitive-observability-sqlite/src/main/java/io/casehub/neocortex/cognitive/observability/sqlite/SqliteSnapshotStore.java` to understand: constructor pattern (DataSource or path), schema creation in constructor, JSON serialization approach, how queries are structured.

- [ ] **Step 2: Write SqlitePipelineSnapshotStore**

Create `cognitive-observability-sqlite/src/main/java/io/casehub/neocortex/cognitive/observability/sqlite/SqlitePipelineSnapshotStore.java`:

Follow SqliteSnapshotStore's patterns exactly:
- Constructor takes DataSource (or creates SQLite connection from path)
- Creates `pipeline_snapshot` table on construction:

```sql
CREATE TABLE IF NOT EXISTS pipeline_snapshot (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tenant_id TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    tick_number INTEGER NOT NULL,
    captured_at TEXT NOT NULL,
    snapshot_json TEXT NOT NULL,
    UNIQUE(tenant_id, agent_id, tick_number)
);
CREATE INDEX IF NOT EXISTS idx_pipeline_tenant_agent_tick
    ON pipeline_snapshot(tenant_id, agent_id, tick_number);
```

- Serialize CognitionSnapshot to JSON using the same approach as SqliteSnapshotStore (likely Jackson ObjectMapper or a GraphSerializer)
- `store()` uses INSERT OR REPLACE for upsert on duplicate (tenant, agent, tick)
- `find()` uses `WHERE tenant_id = ? AND agent_id = ? AND tick_number BETWEEN ? AND ? ORDER BY tick_number`
- `findByTick()` uses `WHERE tenant_id = ? AND agent_id = ? AND tick_number = ?`
- `findLatest()` uses `ORDER BY tick_number DESC LIMIT ?`
- `count()` uses `SELECT COUNT(*) WHERE tenant_id = ?`
- `purge()` uses `DELETE WHERE captured_at < ?`

- [ ] **Step 3: Write SqlitePipelineSnapshotStoreTest**

Create `cognitive-observability-sqlite/src/test/java/io/casehub/neocortex/cognitive/observability/sqlite/SqlitePipelineSnapshotStoreTest.java`:

```java
package io.casehub.neocortex.cognitive.observability.sqlite;

import io.casehub.neocortex.cognitive.observability.PipelineSnapshotStore;
import io.casehub.neocortex.cognitive.observability.testing.PipelineSnapshotStoreContractTest;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Path;

public class SqlitePipelineSnapshotStoreTest extends PipelineSnapshotStoreContractTest {

    @TempDir
    Path tempDir;

    @Override
    protected PipelineSnapshotStore createStore() {
        return new SqlitePipelineSnapshotStore(tempDir.resolve("test.db").toString());
    }
}
```

Adapt constructor call to match SqliteSnapshotStore's actual constructor pattern (DataSource vs path string).

- [ ] **Step 4: Run contract tests on SQLite impl**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-observability-sqlite/pom.xml test -Dtest=SqlitePipelineSnapshotStoreTest -s .mvn/slot-settings.xml`

Expected: All 8 contract tests pass.

- [ ] **Step 5: Commit**

```bash
git add cognitive-observability-sqlite/src/main/java/io/casehub/neocortex/cognitive/observability/sqlite/SqlitePipelineSnapshotStore.java
git add cognitive-observability-sqlite/src/test/java/io/casehub/neocortex/cognitive/observability/sqlite/SqlitePipelineSnapshotStoreTest.java
git commit -m "feat(#520): SqlitePipelineSnapshotStore with contract tests"
```

---

### Task 4: CAPS caching + CognitionStateProvider impl + tick capture

**Files:**
- Modify: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/BehavioralSynthesisPhase.java` — cache last SettlingResult per agent
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java` — implement CognitionStateProvider, add tick capture
- Create: test for CAPS caching in BehavioralSynthesisPhase (or modify existing test)

**Interfaces:**
- Consumes: `CognitionStateProvider` SPI (from Task 1), `PipelineTrace` (from Task 1), `PipelineSnapshotStore` (from Task 2)
- Consumes: `AppraisalTickParticipant.currentResult()`, `.currentConcernSalience()`, `.currentChannelWeights()`, `.currentHabituation()`
- Consumes: `EmotionRegulationTickParticipant` — find the method that returns current regulation result
- Consumes: `CognitionCore.activatedDriveCategories()`, `CognitionCore.promptSections()`
- Produces: Working CognitionStateProvider returning populated PipelineTrace from live state

**IMPORTANT: Verify imports and method signatures.** Before writing code:
1. Use `ide_find_class` to locate `EmotionRegulationResult` and verify its qualified name
2. Use `ide_find_symbol` to find `currentResult` on `EmotionRegulationTickParticipant` — it may be named differently
3. Use `ide_find_symbol` to find the regulation result accessor method name

- [ ] **Step 1: Add CAPS result caching to BehavioralSynthesisPhase**

Read `BehavioralSynthesisPhase.java` fully. At line 168, the settle() call produces `result`. After the settling block, cache the result per agent:

Add a field:
```java
private final Map<String, SettlingResult> lastSettlingResults = new ConcurrentHashMap<>();
private final Map<String, Map<String, Double>> lastInputActivations = new ConcurrentHashMap<>();
```

After `var result = capsEngine.settle(state, combinedActivations);` at line 168, add:
```java
lastSettlingResults.put(agentId, result);
lastInputActivations.put(agentId, Map.copyOf(combinedActivations));
```

Add accessor methods:
```java
public Optional<SettlingResult> lastSettlingResult(String agentId) {
    return Optional.ofNullable(lastSettlingResults.get(agentId));
}

public Map<String, Double> lastInputActivations(String agentId) {
    return lastInputActivations.getOrDefault(agentId, Map.of());
}
```

- [ ] **Step 2: Implement CognitionStateProvider on CognitionCore**

CognitionCore.java — add `implements CognitionStateProvider` to the class declaration.

Implement the three methods. `currentSnapshot()` assembles a full CognitionSnapshot + PipelineTrace from the existing `current*()` methods:

```java
@Override
public Optional<CognitionSnapshot> currentSnapshot(String agentId, String tenantId) {
    if (lastAgentId == null) return Optional.empty();
    var trace = currentTrace(agentId, tenantId).orElse(null);
    // Assemble CognitionSnapshot from existing fields + trace
    // Use the same pattern as the existing snapshot() method if one exists
    // Otherwise build from: mood, drives, mentalModels, userProfiles, strategy, narrative, goals, briefing
    return Optional.of(new CognitionSnapshot(
            agentId, tenantId, turnNumber, Instant.now(),
            moodOrchestrator != null ? moodOrchestrator.currentMood(agentId, tenantId).orElse(null) : null,
            drives != null ? drives.currentDrives(agentId, tenantId).orElse(null) : null,
            /* mentalModels */ Map.of(),
            /* userProfiles */ Map.of(),
            strategy != null ? strategy.currentProfile(agentId, tenantId).orElse(null) : null,
            narrative != null ? narrative.currentState(agentId, tenantId).orElse(null) : null,
            goals != null ? goals.currentProposals(agentId, tenantId) : List.of(),
            /* lastBriefing */ null,
            trace));
}

@Override
public Optional<PipelineTrace> currentTrace(String agentId, String tenantId) {
    if (lastAgentId == null) return Optional.empty();
    return Optional.of(new PipelineTrace(
            appraisalParticipant != null ? appraisalParticipant.currentResult(agentId, tenantId).orElse(null) : null,
            appraisalParticipant != null ? appraisalParticipant.currentConcernSalience(agentId, tenantId) : Map.of(),
            behavioralSynthesis != null ? behavioralSynthesis.lastInputActivations(agentId) : Map.of(),
            behavioralSynthesis != null ? behavioralSynthesis.lastSettlingResult(agentId).orElse(null) : null,
            /* regulationResult — find accessor */ null,
            appraisalParticipant != null ? appraisalParticipant.currentChannelWeights(agentId, tenantId).orElse(null) : null,
            appraisalParticipant != null ? appraisalParticipant.currentHabituation(agentId, tenantId) : null,
            activatedDriveCategories(),
            renderPromptSections()));

}

@Override
public List<String> activeAgents(String tenantId) {
    // Return agent IDs that have active state for this tenant
    // Implementation depends on how CognitionCore tracks agents — check existing state maps
    return List.of();
}
```

**IMPORTANT:** The field names (`moodOrchestrator`, `drives`, `appraisalParticipant`, `behavioralSynthesis`, etc.) are placeholders. Use `ide_file_structure` on CognitionCore.java to find the actual field names. The accessor method names (`currentMood`, `currentDrives`, `currentProfile`, etc.) must be verified with `ide_find_symbol`.

The `renderPromptSections()` helper assembles prompt sections into a `Map<String, String>`:
```java
private Map<String, String> renderPromptSections() {
    var sections = promptSections();
    Map<String, String> rendered = new LinkedHashMap<>();
    for (var renderer : sections) {
        String text = renderer.render(/* context */);
        if (text != null) {
            String key = renderer.blockTag() != null ? renderer.blockTag().name() : renderer.getClass().getSimpleName();
            rendered.put(key, text);
        }
    }
    return rendered;
}
```

This needs a `CognitionRenderContext` — check how `promptSections()` and renderers are used in existing code to construct the context.

- [ ] **Step 3: Wire BehavioralSynthesisPhase into CognitionCore**

Check if CognitionCore already has a reference to BehavioralSynthesisPhase. Use `ide_file_structure` on CognitionCore.java. If not, add it as an optional constructor parameter (following the existing pattern for optional dependencies).

Check the `CognitivePipelineBuilder` (or equivalent) that constructs CognitionCore to pass BehavioralSynthesisPhase.

- [ ] **Step 4: Add tick capture to CognitionCore.tick()**

At the end of the `tick()` method, after the TERMINAL phase completes, add:

```java
if (pipelineSnapshotStore != null) {
    var snapshot = currentSnapshot(agentId, tenantId);
    snapshot.ifPresent(pipelineSnapshotStore::store);
}
```

Add `PipelineSnapshotStore` as an optional dependency of CognitionCore (nullable, set via builder/constructor).

- [ ] **Step 5: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml compile -s .mvn/slot-settings.xml -q`

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add mindmap-intelligence/src/main/java/
git add cognition/src/main/java/
git commit -m "feat(#520): CognitionStateProvider impl + CAPS caching + tick capture"
```

---

## Batch 3: API Endpoint + Integration

### Task 5: CognitivePipelineDump + pipelineDump() endpoint + CDI wiring

**Files:**
- Create: `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/CognitivePipelineDump.java`
- Modify: `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/CognitionApi.java` — add pipelineDump()
- Modify: `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/CognitionService.java` — implement pipelineDump()
- Modify: `cognitive-observability/src/main/java/io/casehub/neocortex/cognitive/observability/CognitionBeans.java` — wire CognitionStateProvider + PipelineSnapshotStore

**Interfaces:**
- Consumes: `CognitionStateProvider` SPI (from Task 1), `PipelineSnapshotStore` (from Task 2)
- Produces: `CognitivePipelineDump` response type, working `pipelineDump()` @PlatformQuery endpoint

- [ ] **Step 1: Create CognitivePipelineDump response type**

Create `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/CognitivePipelineDump.java`:

```java
package io.casehub.neocortex.cognitive.observability;

import io.casehub.neocortex.cognition.core.CognitionSnapshot;
import org.jspecify.annotations.Nullable;

import java.util.List;

public record CognitivePipelineDump(
        String tenantId,
        String agentId,
        QueryMode queryMode,
        @Nullable CognitionSnapshot snapshot,
        List<CognitionSnapshot> snapshots,
        int snapshotCount
) {
    public enum QueryMode { LIVE, HISTORICAL_SINGLE, HISTORICAL_RANGE }

    public static CognitivePipelineDump live(String tenantId, String agentId, @Nullable CognitionSnapshot snapshot) {
        return new CognitivePipelineDump(tenantId, agentId, QueryMode.LIVE, snapshot, List.of(),
                snapshot != null ? 1 : 0);
    }

    public static CognitivePipelineDump single(String tenantId, String agentId, @Nullable CognitionSnapshot snapshot) {
        return new CognitivePipelineDump(tenantId, agentId, QueryMode.HISTORICAL_SINGLE, snapshot, List.of(),
                snapshot != null ? 1 : 0);
    }

    public static CognitivePipelineDump range(String tenantId, String agentId, List<CognitionSnapshot> snapshots) {
        return new CognitivePipelineDump(tenantId, agentId, QueryMode.HISTORICAL_RANGE, null, snapshots,
                snapshots.size());
    }
}
```

- [ ] **Step 2: Add pipelineDump to CognitionApi**

Use `ide_insert_member` to add after the last `@PlatformQuery` method in `CognitionApi.java`:

```java
@PlatformQuery("Pipeline state dump — live or historical. "
        + "No tick args = live state. tick = single historical. fromTick/toTick = range query")
CognitivePipelineDump pipelineDump(@ContextParam("tenancyId") String tenancyId,
                                    String agentId,
                                    Integer tick,
                                    Integer fromTick,
                                    Integer toTick);
```

- [ ] **Step 3: Add CognitionStateProvider + PipelineSnapshotStore to CognitionService**

CognitionService constructor needs two new parameters (both nullable):
- `@Nullable CognitionStateProvider stateProvider`
- `@Nullable PipelineSnapshotStore pipelineSnapshotStore`

Add fields and update constructor. Then implement:

```java
@Override
public CognitivePipelineDump pipelineDump(String tenancyId, String agentId,
                                           Integer tick, Integer fromTick, Integer toTick) {
    if (tick != null && pipelineSnapshotStore != null) {
        var snapshot = pipelineSnapshotStore.findByTick(tenancyId, agentId, tick);
        return CognitivePipelineDump.single(tenancyId, agentId, snapshot.orElse(null));
    }
    if (fromTick != null && toTick != null && pipelineSnapshotStore != null) {
        var snapshots = pipelineSnapshotStore.find(tenancyId, agentId, fromTick, toTick);
        return CognitivePipelineDump.range(tenancyId, agentId, snapshots);
    }
    if (stateProvider != null) {
        var snapshot = stateProvider.currentSnapshot(agentId, tenancyId);
        return CognitivePipelineDump.live(tenancyId, agentId, snapshot.orElse(null));
    }
    return CognitivePipelineDump.live(tenancyId, agentId, null);
}
```

- [ ] **Step 4: Update CognitionBeans CDI wiring**

In `CognitionBeans.cognitionService()`, add parameters:

```java
Instance<CognitionStateProvider> stateProvider,
Instance<PipelineSnapshotStore> pipelineSnapshotStore
```

Pass to constructor:
```java
stateProvider.isResolvable() ? stateProvider.get() : null,
pipelineSnapshotStore.isResolvable() ? pipelineSnapshotStore.get() : null
```

- [ ] **Step 5: Verify compilation of all observability modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-observability/pom.xml compile -s .mvn/slot-settings.xml -q`

Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add cognitive-observability-core/src/main/java/
git add cognitive-observability/src/main/java/
git commit -m "feat(#520): pipelineDump @PlatformQuery endpoint + CDI wiring"
```

---

### Task 6: CognitionStateProviderIntegrationTest

**Files:**
- Create: `cognition/src/test/java/io/casehub/neocortex/cognition/core/CognitionStateProviderIntegrationTest.java`

**Interfaces:**
- Consumes: `CognitionStateProvider` (implemented by CognitionCore), `CognitionCoreTestBuilder`, `PipelineTrace`

- [ ] **Step 1: Write integration test**

Create `cognition/src/test/java/io/casehub/neocortex/cognition/core/CognitionStateProviderIntegrationTest.java`:

```java
package io.casehub.neocortex.cognition.core;

import io.casehub.neocortex.cognitive.observability.CognitionStateProvider;
import io.casehub.neocortex.cognitive.observability.PipelineTrace;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class CognitionStateProviderIntegrationTest {

    @Test
    void tickProducesSnapshotWithPipelineTrace() {
        // Build CognitionCore using CognitionCoreTestBuilder
        // Use the same pattern as existing tests in this package
        // Run one tick with a simple observation
        // Query CognitionCore as CognitionStateProvider
        // Assert: snapshot is present
        // Assert: trace is not null
        // Assert: promptSections is non-empty
        // Assert: activatedDriveCategories is populated (if drives configured)
    }

    @Test
    void currentTraceContainsAppraisalAfterTick() {
        // Build CognitionCore with appraisal enabled
        // Run one tick with an emotional observation
        // Query currentTrace()
        // Assert: appraisalResult is not null
        // Assert: concernSalience is non-empty
    }

    @Test
    void activeAgentsReturnsTickedAgent() {
        // Build CognitionCore, tick one agent
        // Query activeAgents(tenantId)
        // Assert: contains the ticked agentId
    }
}
```

**IMPORTANT:** Use `ide_find_class` to locate `CognitionCoreTestBuilder` and read it to understand the builder pattern. Read 2-3 existing tests in `cognition/src/test/java/.../core/` for the setup pattern (descriptor, resolver, observation construction).

The test code above is structural — fill in concrete implementation by studying the existing test patterns.

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=CognitionStateProviderIntegrationTest -s .mvn/slot-settings.xml`

Expected: All tests pass.

- [ ] **Step 3: Run full fast test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -s .mvn/slot-settings.xml -q`

Expected: BUILD SUCCESS (all existing tests still pass)

- [ ] **Step 4: Commit**

```bash
git add cognition/src/test/java/
git commit -m "feat(#520): CognitionStateProvider integration test"
```

---

## References

- [2026-11-09-cognitive-state-dump-design.md] — design spec this plan implements
- [CognitionSnapshot.java:17] — existing snapshot record (12 fields, adding PipelineTrace)
- [CognitionApi.java:10-55] — @McpDomain interface with @PlatformQuery methods
- [CognitionService.java] — @PlatformQuery implementation
- [CognitionBeans.java:14-34] — CDI producer with Instance<> optional pattern
- [CognitionCore.java:234-321] — tick() pipeline flow (FOUNDATION → SOURCE → DERIVED → TERMINAL)
- [CognitionCore.java:595-619] — activatedDriveCategories() method
- [CognitionCore.java:622-704] — promptSections() method
- [BehavioralSynthesisPhase.java:168] — sole CapsEngine.settle() call site (consolidation-time)
- [SnapshotStore.java] — interface pattern to follow for PipelineSnapshotStore
- [SqliteSnapshotStore.java] — SQLite impl pattern
- [InMemorySnapshotStore.java] — in-memory impl pattern
- [SnapshotStoreContractTest.java] — abstract contract test pattern
- [AppraisalTickParticipant.java] — currentResult(), currentConcernSalience(), currentChannelWeights(), currentHabituation()
- [GitHub #520] — feat: cognitive state dump endpoint
- [GitHub #513] — epic: cognitive pipeline completion (diagnostic tooling section)
- [decisions.md D15-D19] — design decisions (revised after review)
