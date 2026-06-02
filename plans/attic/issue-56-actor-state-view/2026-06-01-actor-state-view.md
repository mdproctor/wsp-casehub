# Actor State View Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement `GET /actors/{actorId}/state` — a unified view of an actor's active workload (trust scores, WorkItems, Commitments, engine cases) assembled from four independent backends with per-source failure isolation.

**Architecture:** `ActorStateContributor` SPI + `ActorStateAccumulator` interface in `casehub-platform-api` (stdlib types only). Four contributor implementations in a new `casehub-engine-actor-state` module. Two small SPI additions each to qhorus, ledger, work, and engine. Blocking aggregator uses `@Inject ManagedExecutor` for CDI-context-safe parallel execution. Reactive aggregator wraps blocking contributors on Mutiny's blocking executor.

**Tech Stack:** Java 21, Quarkus 3.32.2, Panache (blocking and reactive), MicroProfile Context Propagation (`ManagedExecutor`), Jakarta CDI `@Any Instance<T>`, Jackson (`@JsonInclude(NON_NULL)`).

**Repos touched (in order):**
1. `casehub-platform` — two interfaces
2. `casehub-qhorus` — SPI additions + contract tests
3. `casehub-ledger` — service method addition
4. `casehub-work` — utility class
5. `casehub-engine` — SPI addition + new module

**Spec:** `~/claude/public/casehub/specs/2026-06-01-actor-state-view-design.md`

---

## Phase 1 — Platform API (`casehub-platform` repo)

### Task 1: ActorStateContributor and ActorStateAccumulator interfaces

**Repo:** `/Users/mdproctor/claude/casehub/platform`

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/actor/ActorStateContributor.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/actor/ActorStateAccumulator.java`

- [ ] **Step 1: Create `ActorStateAccumulator.java`**

```java
// platform-api/src/main/java/io/casehub/platform/api/actor/ActorStateAccumulator.java
package io.casehub.platform.api.actor;

import java.time.Instant;
import java.util.UUID;

/**
 * Visitor interface for assembling actor state data from independent backends.
 *
 * <p>All parameters use only {@code java.util.*}, {@code java.time.*}, and {@code java.util.UUID} —
 * no domain types cross the platform-api boundary.
 *
 * <p>Thread-safety: implementations must be thread-safe — contributors run concurrently.
 */
public interface ActorStateAccumulator {

    /** Global trust score. {@code null} when actor has no computed score yet (not 0.0). */
    void trustScore(Double score);

    /** Per-capability trust score. */
    void capabilityScore(String capability, double score);

    /**
     * An active WorkItem assigned to or owned by the actor.
     *
     * @param title    may be null for externally created WorkItems
     * @param category may be null for externally created WorkItems
     * @param caseId   may be null when WorkItem was not created by the engine
     */
    void workItem(UUID id, String title, String status, String category, UUID caseId);

    /**
     * An open Commitment where this actor is the obligor.
     *
     * @param caseId may be null when channel does not follow {@code case-{caseId}/...} naming
     */
    void commitment(UUID commitmentId, UUID channelId, UUID caseId, String state, Instant expiresAt);

    /**
     * A Quartz job case UUID where this actor is currently executing.
     * Best-effort snapshot — may include cases whose work completed since the scan started.
     */
    void engineActiveCaseId(UUID caseId);
}
```

- [ ] **Step 2: Create `ActorStateContributor.java`**

```java
// platform-api/src/main/java/io/casehub/platform/api/actor/ActorStateContributor.java
package io.casehub.platform.api.actor;

/**
 * SPI for contributing to a unified actor state view.
 *
 * <p>Implementations must collect all results before calling any accumulator method
 * (atomic contribution contract). If contribute() throws, the aggregator records
 * the failure and excludes this source — no partial data reaches the accumulator.
 *
 * <p>CDI discovery: annotate implementations {@code @ApplicationScoped}.
 * The aggregator collects via {@code @Inject @Any Instance<ActorStateContributor>}.
 */
public interface ActorStateContributor {

    /** Stable name for this source — appears in the response {@code sources} and {@code sourceWarnings} fields. */
    String sourceName();

    /**
     * Contribute actor state data to the accumulator.
     *
     * @param actorId     the actor identity string — must be the same string across all backends
     * @param accumulator the thread-safe accumulator collecting this session's state
     */
    void contribute(String actorId, ActorStateAccumulator accumulator);
}
```

- [ ] **Step 3: Build platform-api to verify compilation**

```bash
mvn --batch-mode compile -pl platform-api -f /Users/mdproctor/claude/casehub/platform/pom.xml -q
```
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add platform-api/src/main/java/io/casehub/platform/api/actor/
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(parent#56): add ActorStateContributor + ActorStateAccumulator SPI to platform-api

First use-case-specific SPI in platform-api — stdlib types only (no domain objects).
Satisfies platform-api scope rule: needed by ≥4 peer repos, zero domain types.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 2 — Qhorus SPI additions (`casehub-qhorus` repo)

**Repo:** `/Users/mdproctor/claude/casehub/qhorus`

### Task 2: CommitmentStore.findOpenByObligor (cross-channel)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/CommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/ReactiveCommitmentStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/ReactiveJpaCommitmentStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryCommitmentStore.java`
- Modify: `testing/src/main/java/io/casehub/qhorus/testing/InMemoryReactiveCommitmentStore.java`
- Modify: `testing/src/test/java/io/casehub/qhorus/testing/contract/CommitmentStoreContractTest.java`

- [ ] **Step 1: Add 4 @Test methods to CommitmentStoreContractTest**

In `CommitmentStoreContractTest`, add abstract method declaration and 4 tests after the existing tests:

```java
// Add to CommitmentStoreContractTest — after existing tests

protected abstract List<Commitment> findOpenByObligor(String obligor);

@Test
void findOpenByObligor_returnsAcrossMultipleChannels() {
    UUID ch1 = UUID.randomUUID();
    UUID ch2 = UUID.randomUUID();
    save(openCommitment("corr-cross-1", "req", "agent-x", ch1));
    save(openCommitment("corr-cross-2", "req", "agent-x", ch2));
    save(openCommitment("corr-cross-3", "req", "agent-y", ch1));  // different obligor

    List<Commitment> result = findOpenByObligor("agent-x");
    assertThat(result).hasSize(2);
    assertThat(result).extracting(c -> c.correlationId)
            .containsExactlyInAnyOrder("corr-cross-1", "corr-cross-2");
}

@Test
void findOpenByObligor_excludesTerminalStates_crossChannel() {
    UUID ch1 = UUID.randomUUID();
    UUID ch2 = UUID.randomUUID();
    save(openCommitment("corr-term-open", "req", "agent-x", ch1));
    Commitment fulfilled = save(openCommitment("corr-term-fulfilled", "req", "agent-x", ch2));
    fulfilled.state = CommitmentState.FULFILLED;
    fulfilled.resolvedAt = Instant.now();
    save(fulfilled);

    List<Commitment> result = findOpenByObligor("agent-x");
    assertThat(result).hasSize(1);
    assertEquals("corr-term-open", result.get(0).correlationId);
}

@Test
void findOpenByObligor_nullObligorInStore_notReturnedForActualActor() {
    UUID ch = UUID.randomUUID();
    // null obligor = broadcast message
    Commitment broadcast = openCommitment("corr-broadcast", "req", null, ch);
    broadcast.obligor = null;
    save(broadcast);
    save(openCommitment("corr-real", "req", "agent-x", ch));

    List<Commitment> result = findOpenByObligor("agent-x");
    assertThat(result).hasSize(1);
    assertEquals("corr-real", result.get(0).correlationId);
}

@Test
void findOpenByObligor_emptyStore_returnsEmpty() {
    assertThat(findOpenByObligor("agent-x")).isEmpty();
}
```

Also add the import for `assertThat` if missing:
```java
import static org.assertj.core.api.Assertions.assertThat;
```

- [ ] **Step 2: Run tests — expect compile errors (method doesn't exist yet)**

```bash
mvn --batch-mode test -pl testing -am -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -20
```
Expected: COMPILE ERROR — `findOpenByObligor` not found.

- [ ] **Step 3: Add default method to `CommitmentStore` interface**

In `CommitmentStore.java`, add after `findAllOpen()`:

```java
/**
 * All non-terminal commitments where this actor is the obligor, across all channels.
 *
 * <p>WARNING: This default is a full table scan over all open commitments —
 * JPA-backed implementations MUST override with an indexed query.
 * This default is correct only for in-memory test implementations.
 *
 * @param obligor the actor identity string; null returns empty list
 */
default List<Commitment> findOpenByObligor(String obligor) {
    if (obligor == null) return java.util.List.of();
    return findAllOpen().stream()
            .filter(c -> obligor.equals(c.obligor))
            .toList();
}
```

- [ ] **Step 4: Override in `JpaCommitmentStore`**

In `JpaCommitmentStore.java`, add override after the existing `findOpenByObligor(String, UUID)` method:

```java
@Override
public List<Commitment> findOpenByObligor(String obligor) {
    if (obligor == null) return List.of();
    return repo.list(
            "obligor = ?1 AND state NOT IN ?2",
            obligor, terminalStates());
}
```

- [ ] **Step 5: Add to `ReactiveCommitmentStore` interface**

In `ReactiveCommitmentStore.java`, add after `findAllOpen()`:

```java
/**
 * Reactive equivalent of {@link CommitmentStore#findOpenByObligor(String)}.
 * JPA-backed implementations MUST override — default is a full scan of all open commitments.
 */
default io.smallrye.mutiny.Uni<List<Commitment>> findOpenByObligor(String obligor) {
    if (obligor == null) return io.smallrye.mutiny.Uni.createFrom().item(java.util.List.of());
    return findAllOpen().map(all -> all.stream()
            .filter(c -> obligor != null && obligor.equals(c.obligor))
            .toList());
}
```

- [ ] **Step 6: Override in `ReactiveJpaCommitmentStore`**

In `ReactiveJpaCommitmentStore.java`, add override:

```java
@Override
public Uni<List<Commitment>> findOpenByObligor(String obligor) {
    if (obligor == null) return Uni.createFrom().item(List.of());
    return repo.list("obligor = ?1 AND state NOT IN ?2",
            obligor, terminalStates());
}
```

Where `terminalStates()` returns `List.of(CommitmentState.FULFILLED, CommitmentState.DECLINED, CommitmentState.FAILED, CommitmentState.DELEGATED, CommitmentState.EXPIRED)`. Add this helper if not already present in `ReactiveJpaCommitmentStore`.

- [ ] **Step 7: Implement in `InMemoryCommitmentStore`**

In `InMemoryCommitmentStore.java`, add override:

```java
@Override
public List<Commitment> findOpenByObligor(String obligor) {
    if (obligor == null) return List.of();
    return byId.values().stream()
            .filter(c -> !c.state.isTerminal())
            .filter(c -> obligor.equals(c.obligor))
            .toList();
}
```

- [ ] **Step 8: Implement in `InMemoryReactiveCommitmentStore`**

In `InMemoryReactiveCommitmentStore.java`, add override (it delegates to blocking store):

```java
@Override
public Uni<List<Commitment>> findOpenByObligor(String obligor) {
    return Uni.createFrom().item(() -> blocking.findOpenByObligor(obligor));
}
```

Where `blocking` is the injected or composed `InMemoryCommitmentStore` (check how other reactive methods delegate — follow the same pattern).

- [ ] **Step 9: Implement abstract method in CommitmentStoreContractTest subclasses**

In `InMemoryCommitmentStoreTest` (and any other direct subclasses), add:
```java
@Override
protected List<Commitment> findOpenByObligor(String obligor) {
    return store.findOpenByObligor(obligor);
}
```

In `JpaCommitmentStoreTest`:
```java
@Override
protected List<Commitment> findOpenByObligor(String obligor) {
    return store.findOpenByObligor(obligor);
}
```

- [ ] **Step 10: Run tests — all pass**

```bash
mvn --batch-mode test -pl testing,runtime -am -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL|ERROR" | tail -10
```
Expected: all tests pass, BUILD SUCCESS.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(parent#56): add CommitmentStore.findOpenByObligor(obligor) cross-channel query

Default scans findAllOpen() (in-memory only). JpaCommitmentStore and
ReactiveJpaCommitmentStore override with indexed queries.
Contract test: 4 new @Test methods inherited by InMemory and JPA test classes.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: ChannelStore.findByIds batch lookup

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/ChannelStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelStore.java`
- Modify: `testing/src/test/java/io/casehub/qhorus/testing/contract/ChannelStoreContractTest.java`

Note: `InMemoryChannelStore` inherits the default — no override needed.

- [ ] **Step 1: Add 4 @Test methods to ChannelStoreContractTest**

```java
// Add to ChannelStoreContractTest — requires findByIds abstract method

protected abstract List<io.casehub.qhorus.runtime.channel.Channel> findByIds(
        java.util.Collection<java.util.UUID> ids);

@Test
void findByIds_allPresent() {
    Channel ch1 = put(channel("ch-ids-1-" + UUID.randomUUID(), ChannelSemantic.APPEND));
    Channel ch2 = put(channel("ch-ids-2-" + UUID.randomUUID(), ChannelSemantic.COLLECT));
    List<Channel> result = findByIds(List.of(ch1.id, ch2.id));
    assertThat(result).hasSize(2);
    assertThat(result).extracting(c -> c.id).containsExactlyInAnyOrder(ch1.id, ch2.id);
}

@Test
void findByIds_partiallyPresent_missingIdsOmitted() {
    Channel ch = put(channel("ch-partial-" + UUID.randomUUID(), ChannelSemantic.APPEND));
    List<Channel> result = findByIds(List.of(ch.id, UUID.randomUUID()));
    assertThat(result).hasSize(1);
    assertEquals(ch.id, result.get(0).id);
}

@Test
void findByIds_emptyCollection_returnsEmpty() {
    assertThat(findByIds(List.of())).isEmpty();
}

@Test
void findByIds_unknownIds_returnsEmpty() {
    assertThat(findByIds(List.of(UUID.randomUUID(), UUID.randomUUID()))).isEmpty();
}
```

- [ ] **Step 2: Run to confirm compile error**

```bash
mvn --batch-mode test-compile -pl testing -am -f /Users/mdproctor/claude/casehub/qhorus/pom.xml 2>&1 | tail -10
```
Expected: compile error — `findByIds` not found.

- [ ] **Step 3: Add default method to `ChannelStore` interface**

In `ChannelStore.java`, add after existing `find(UUID id)` method:

```java
/**
 * Batch lookup of channels by ID set. Returns only channels that exist — missing IDs are silently omitted.
 * Empty collection input returns an empty list without querying the store.
 *
 * <p>Default uses N individual {@link #find(UUID)} calls — JPA-backed implementations
 * should override with a single {@code WHERE id IN (:ids)} query.
 */
default java.util.List<io.casehub.qhorus.runtime.channel.Channel> findByIds(
        java.util.Collection<java.util.UUID> ids) {
    if (ids == null || ids.isEmpty()) return java.util.List.of();
    return ids.stream()
            .map(this::find)
            .filter(java.util.Optional::isPresent)
            .map(java.util.Optional::get)
            .toList();
}
```

- [ ] **Step 4: Override in `JpaChannelStore` with a single IN query**

In `JpaChannelStore.java`, add override:

```java
@Override
public List<Channel> findByIds(Collection<UUID> ids) {
    if (ids == null || ids.isEmpty()) return List.of();
    return Channel.list("id IN ?1", new ArrayList<>(ids));
}
```

Add import `import java.util.Collection;` and `import java.util.ArrayList;` if missing.

- [ ] **Step 5: Implement abstract method in contract test subclasses**

In `InMemoryChannelStoreTest`:
```java
@Override
protected List<Channel> findByIds(Collection<UUID> ids) {
    return store.findByIds(ids);
}
```

In `JpaChannelStoreTest` (if it exists — find via `find /Users/mdproctor/claude/casehub/qhorus -name "JpaChannelStoreTest.java"`):
```java
@Override
protected List<Channel> findByIds(Collection<UUID> ids) {
    return store.findByIds(ids);
}
```

- [ ] **Step 6: Run tests**

```bash
mvn --batch-mode test -pl testing,runtime -am -f /Users/mdproctor/claude/casehub/qhorus/pom.xml -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/qhorus add -A
git -C /Users/mdproctor/claude/casehub/qhorus commit -m "feat(parent#56): add ChannelStore.findByIds(Collection<UUID>) batch lookup

Default uses N individual find() calls (correct for in-memory).
JpaChannelStore overrides with a single WHERE id IN (?1) Panache query.
Contract test: 4 new @Test methods.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 3 — Ledger service addition (`casehub-ledger` repo)

**Repo:** `/Users/mdproctor/claude/casehub/ledger`

### Task 4: TrustGateService.allCapabilityScores

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustGateService.java`
- Modify (test): find existing `TrustGateService` test class via `find /Users/mdproctor/claude/casehub/ledger -name "TrustGateService*Test*" -o -name "*TrustGate*Test*"`

- [ ] **Step 1: Find the existing TrustGateService test**

```bash
find /Users/mdproctor/claude/casehub/ledger -name "*TrustGate*" 2>/dev/null
```

If no test exists, create `runtime/src/test/java/io/casehub/ledger/runtime/service/TrustGateServiceTest.java`.

- [ ] **Step 2: Write failing test for allCapabilityScores**

In the TrustGateService test class (or create it):

```java
@Test
void allCapabilityScores_returnsAllCapabilityRows() {
    // Use Mockito to mock ActorTrustScoreRepository
    var repo = mock(ActorTrustScoreRepository.class);
    var service = new TrustGateService(repo);

    var cap1 = new io.casehub.ledger.runtime.model.ActorTrustScore();
    cap1.capabilityKey = "sar-drafting";
    cap1.trustScore = 0.79;

    var cap2 = new io.casehub.ledger.runtime.model.ActorTrustScore();
    cap2.capabilityKey = "osint-screening";
    cap2.trustScore = 0.85;

    when(repo.findByActorIdAndScoreType("agent-x", ScoreType.CAPABILITY))
            .thenReturn(List.of(cap1, cap2));

    Map<String, Double> result = service.allCapabilityScores("agent-x");

    assertEquals(2, result.size());
    assertEquals(0.79, result.get("sar-drafting"), 0.001);
    assertEquals(0.85, result.get("osint-screening"), 0.001);
}

@Test
void allCapabilityScores_noScores_returnsEmptyMap() {
    var repo = mock(ActorTrustScoreRepository.class);
    when(repo.findByActorIdAndScoreType(any(), eq(ScoreType.CAPABILITY))).thenReturn(List.of());
    var service = new TrustGateService(repo);
    assertTrue(service.allCapabilityScores("unknown").isEmpty());
}
```

Imports needed: `io.casehub.ledger.api.model.ActorTrustScore.ScoreType`, `org.mockito.Mockito.*`, `java.util.Map`.

- [ ] **Step 3: Run test — expect FAIL (method doesn't exist)**

```bash
mvn --batch-mode test -pl runtime -am -f /Users/mdproctor/claude/casehub/ledger/pom.xml -Dtest="TrustGateServiceTest#allCapabilityScores*" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -15
```
Expected: compile or runtime error.

- [ ] **Step 4: Add `allCapabilityScores` to TrustGateService**

In `TrustGateService.java`, add after existing `dimensionScores` method:

```java
/**
 * Returns all CAPABILITY trust scores for the actor, keyed by capability tag.
 * Empty map if no capability scores have been computed yet.
 */
public Map<String, Double> allCapabilityScores(final String actorId) {
    return repository.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY).stream()
            .collect(Collectors.toMap(s -> s.capabilityKey, s -> s.trustScore));
}
```

Add import `import io.casehub.ledger.api.model.ActorTrustScore.ScoreType;` if missing.

- [ ] **Step 5: Run tests — expect PASS**

```bash
mvn --batch-mode test -pl runtime -am -f /Users/mdproctor/claude/casehub/ledger/pom.xml -Dtest="TrustGateServiceTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD" | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add -A
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(parent#56): add TrustGateService.allCapabilityScores(actorId)

Returns Map<String, Double> of all CAPABILITY-scoped trust scores.
Keeps external consumers out of ActorTrustScoreRepository.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 4 — Work API utility (`casehub-work` repo)

**Repo:** `/Users/mdproctor/claude/casehub/work`

### Task 5: WorkItemCallerRef.parseCaseId

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/WorkItemCallerRef.java`
- Create: `api/src/test/java/io/casehub/work/api/WorkItemCallerRefTest.java`

- [ ] **Step 1: Write failing test**

```java
// api/src/test/java/io/casehub/work/api/WorkItemCallerRefTest.java
package io.casehub.work.api;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class WorkItemCallerRefTest {

    private static final UUID CASE_UUID = UUID.fromString("3fa85f64-5717-4562-b3fc-2c963f66afa6");

    @Test
    void parseCaseId_validEngineFormat_returnsCaseId() {
        String callerRef = CASE_UUID + ":pi-001";
        assertEquals(CASE_UUID, WorkItemCallerRef.parseCaseId(callerRef));
    }

    @Test
    void parseCaseId_notAUuid_returnsNull() {
        assertNull(WorkItemCallerRef.parseCaseId("not-a-uuid:planItem"));
    }

    @Test
    void parseCaseId_noColon_returnsNull() {
        assertNull(WorkItemCallerRef.parseCaseId("justsomething"));
    }

    @Test
    void parseCaseId_null_returnsNull() {
        assertNull(WorkItemCallerRef.parseCaseId(null));
    }

    @Test
    void parseCaseId_emptyString_returnsNull() {
        assertNull(WorkItemCallerRef.parseCaseId(""));
    }
}
```

- [ ] **Step 2: Run — expect compile error**

```bash
mvn --batch-mode test -pl api -am -f /Users/mdproctor/claude/casehub/work/pom.xml -Dtest="WorkItemCallerRefTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | tail -10
```
Expected: compile error — `WorkItemCallerRef` not found.

- [ ] **Step 3: Create `WorkItemCallerRef.java`**

```java
// api/src/main/java/io/casehub/work/api/WorkItemCallerRef.java
package io.casehub.work.api;

import java.util.UUID;

/**
 * Utilities for parsing the {@code WorkItem.callerRef} field.
 *
 * <p>The engine sets {@code callerRef} to {@code "{caseId}:{planItemId}"} when spawning
 * WorkItems from a case binding. Externally created WorkItems may use any format —
 * parsing returns {@code null} rather than throwing when the format is unrecognised.
 */
public final class WorkItemCallerRef {

    private WorkItemCallerRef() {}

    /**
     * Parses the case UUID from a callerRef string.
     *
     * @param callerRef the WorkItem callerRef; may be null
     * @return the case UUID, or null if callerRef is null, has no colon, or the first segment is not a valid UUID
     */
    public static UUID parseCaseId(final String callerRef) {
        if (callerRef == null || !callerRef.contains(":")) return null;
        try {
            return UUID.fromString(callerRef.split(":")[0]);
        } catch (final IllegalArgumentException e) {
            return null;
        }
    }
}
```

- [ ] **Step 4: Run — expect PASS**

```bash
mvn --batch-mode test -pl api -am -f /Users/mdproctor/claude/casehub/work/pom.xml -Dtest="WorkItemCallerRefTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD" | tail -5
```
Expected: BUILD SUCCESS, Tests run: 5.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/work add api/src/main/java/io/casehub/work/api/WorkItemCallerRef.java api/src/test/java/io/casehub/work/api/WorkItemCallerRefTest.java
git -C /Users/mdproctor/claude/casehub/work commit -m "feat(parent#56): add WorkItemCallerRef.parseCaseId utility

Safely parses caseId from engine-created WorkItem.callerRef ('caseId:planItemId').
Returns null for non-engine callerRefs without throwing.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 5 — Engine SPI and API additions (`casehub-engine` repo)

**Repo:** `/Users/mdproctor/claude/casehub/engine`

### Task 6: WorkerExecutionManager.getActiveCaseIds + QuartzWorkerExecutionManager override

**Files:**
- Modify: `common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionManager.java`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java`
- Modify: `scheduler-quartz/src/test/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManagerTest.java`

- [ ] **Step 1: Write failing tests for getActiveCaseIds in QuartzWorkerExecutionManagerTest**

In `QuartzWorkerExecutionManagerTest.java`, add import and test methods:

```java
import org.mockito.Mockito;
import org.quartz.JobDataMap;
import org.quartz.JobDetail;
import org.quartz.JobKey;
import org.quartz.Scheduler;
import org.quartz.impl.matchers.GroupMatcher;
import java.util.List;
import java.util.Set;
import java.util.UUID;

// Add to QuartzWorkerExecutionManagerTest:

@Test
void getActiveCaseIds_returnsUuidForMatchingWorker() throws Exception {
    UUID caseUuid = UUID.randomUUID();
    Scheduler scheduler = Mockito.mock(Scheduler.class);
    JobKey jobKey = JobKey.jobKey("key-1", "group-1");
    JobDetail detail = Mockito.mock(JobDetail.class);
    JobDataMap dataMap = new JobDataMap();
    dataMap.put("workerId", "agent-x");
    dataMap.put("caseHubInstanceUuid", caseUuid.toString());

    Mockito.when(scheduler.getJobGroupNames()).thenReturn(List.of("group-1"));
    Mockito.when(scheduler.getJobKeys(GroupMatcher.groupEquals("group-1")))
            .thenReturn(Set.of(jobKey));
    Mockito.when(scheduler.getJobDetail(jobKey)).thenReturn(detail);
    Mockito.when(detail.getJobDataMap()).thenReturn(dataMap);

    QuartzWorkerExecutionManager manager = new QuartzWorkerExecutionManager(scheduler);
    List<UUID> result = manager.getActiveCaseIds("agent-x");
    assertEquals(1, result.size());
    assertEquals(caseUuid, result.get(0));
}

@Test
void getActiveCaseIds_wrongWorker_returnsEmpty() throws Exception {
    Scheduler scheduler = Mockito.mock(Scheduler.class);
    JobKey jobKey = JobKey.jobKey("key-2", "group-1");
    JobDetail detail = Mockito.mock(JobDetail.class);
    JobDataMap dataMap = new JobDataMap();
    dataMap.put("workerId", "agent-other");
    dataMap.put("caseHubInstanceUuid", UUID.randomUUID().toString());

    Mockito.when(scheduler.getJobGroupNames()).thenReturn(List.of("group-1"));
    Mockito.when(scheduler.getJobKeys(GroupMatcher.groupEquals("group-1")))
            .thenReturn(Set.of(jobKey));
    Mockito.when(scheduler.getJobDetail(jobKey)).thenReturn(detail);
    Mockito.when(detail.getJobDataMap()).thenReturn(dataMap);

    QuartzWorkerExecutionManager manager = new QuartzWorkerExecutionManager(scheduler);
    assertTrue(manager.getActiveCaseIds("agent-x").isEmpty());
}

@Test
void getActiveCaseIds_schedulerThrows_returnsEmpty() throws Exception {
    Scheduler scheduler = Mockito.mock(Scheduler.class);
    Mockito.when(scheduler.getJobGroupNames()).thenThrow(new org.quartz.SchedulerException("test"));

    QuartzWorkerExecutionManager manager = new QuartzWorkerExecutionManager(scheduler);
    assertTrue(manager.getActiveCaseIds("agent-x").isEmpty());
}
```

- [ ] **Step 2: Run — expect compile error (method not found)**

```bash
mvn --batch-mode test-compile -pl scheduler-quartz -am -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | tail -10
```

- [ ] **Step 3: Add default method to WorkerExecutionManager SPI**

In `WorkerExecutionManager.java`, add after `getActiveWorkCount`:

```java
/**
 * Returns the case UUIDs for all Quartz jobs currently executing for this worker.
 *
 * <p>This is a best-effort snapshot — a job completing between this call and the
 * HTTP response means the list may transiently contain cases whose work just finished.
 * Acceptable for a monitoring/observability endpoint.
 *
 * <p>NOTE: workerId here equals actorId at the REST layer — same string, different naming convention.
 *
 * @param workerId the worker name from the case definition YAML
 * @return list of case UUIDs currently executing; empty list if none or on error
 */
default java.util.List<java.util.UUID> getActiveCaseIds(String workerId) {
    return java.util.List.of();
}
```

- [ ] **Step 4: Override in QuartzWorkerExecutionManager**

In `QuartzWorkerExecutionManager.java`, add override method (mirroring `getActiveWorkCount` structure):

```java
@Override
public List<UUID> getActiveCaseIds(String workerId) {
    try {
        List<String> groups = scheduler.getJobGroupNames();
        List<UUID> caseIds = new java.util.ArrayList<>();
        for (String group : groups) {
            Set<JobKey> keys = scheduler.getJobKeys(GroupMatcher.groupEquals(group));
            for (JobKey key : keys) {
                JobDetail detail = scheduler.getJobDetail(key);
                if (detail != null) {
                    Object wId = detail.getJobDataMap().get("workerId");
                    Object caseUuid = detail.getJobDataMap().get("caseHubInstanceUuid");
                    if (workerId.equals(wId) && caseUuid instanceof String s) {
                        try {
                            caseIds.add(UUID.fromString(s));
                        } catch (IllegalArgumentException ignored) {
                            // malformed UUID in job data — skip
                        }
                    }
                }
            }
        }
        return java.util.Collections.unmodifiableList(caseIds);
    } catch (org.quartz.SchedulerException e) {
        LOG.warnf("Failed to collect active case IDs for worker '%s' — returning empty: %s",
                workerId, e.getMessage());
        return List.of();
    }
}
```

Add `import java.util.UUID;` and `import java.util.List;` if missing.

- [ ] **Step 5: Run tests**

```bash
mvn --batch-mode test -pl scheduler-quartz -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dtest="QuartzWorkerExecutionManagerTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD" | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src/main/java/io/casehub/engine/common/spi/scheduler/WorkerExecutionManager.java scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java scheduler-quartz/src/test/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManagerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add WorkerExecutionManager.getActiveCaseIds(workerId)

Default returns List.of() (safe no-op per PP-20260601-81b9e5).
QuartzWorkerExecutionManager collects caseHubInstanceUuid from Quartz job data
for matching workerId. Error-safe: SchedulerException returns empty list.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 7: CaseChannel.parseCaseId static method

**Files:**
- Modify: `api/src/main/java/io/casehub/api/model/CaseChannel.java`
- Modify or Create: `api/src/test/java/io/casehub/api/model/CaseChannelTest.java`

- [ ] **Step 1: Find existing CaseChannel test or create it**

```bash
find /Users/mdproctor/claude/casehub/engine/api/src/test -name "CaseChannelTest.java" 2>/dev/null
```

If not found, create `api/src/test/java/io/casehub/api/model/CaseChannelTest.java`.

- [ ] **Step 2: Write failing tests**

```java
package io.casehub.api.model;

import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class CaseChannelTest {

    private static final UUID CASE_UUID = UUID.fromString("3fa85f64-5717-4562-b3fc-2c963f66afa6");

    @Test
    void parseCaseId_standardFormat_withPurpose() {
        assertEquals(CASE_UUID, CaseChannel.parseCaseId("case-" + CASE_UUID + "/work"));
    }

    @Test
    void parseCaseId_noPurposeSegment() {
        assertEquals(CASE_UUID, CaseChannel.parseCaseId("case-" + CASE_UUID));
    }

    @Test
    void parseCaseId_notCaseChannel_returnsNull() {
        assertNull(CaseChannel.parseCaseId("some-other-channel"));
    }

    @Test
    void parseCaseId_invalidUuid_returnsNull() {
        assertNull(CaseChannel.parseCaseId("case-not-a-uuid/work"));
    }

    @Test
    void parseCaseId_null_returnsNull() {
        assertNull(CaseChannel.parseCaseId(null));
    }
}
```

- [ ] **Step 3: Run — expect compile error**

```bash
mvn --batch-mode test-compile -pl api -am -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | tail -10
```

- [ ] **Step 4: Add parseCaseId to CaseChannel**

In `CaseChannel.java`, add after the existing `channelName(UUID, String)` static method:

```java
/**
 * Extracts the case UUID from a channel name that follows the {@code "case-{caseId}/{purpose}"} convention.
 *
 * @param channelName the channel name; may be null
 * @return the case UUID, or null if channelName is null, does not start with {@link #CASE_CHANNEL_PREFIX},
 *         or the UUID segment is malformed
 */
public static java.util.UUID parseCaseId(final String channelName) {
    if (channelName == null || !channelName.startsWith(CASE_CHANNEL_PREFIX)) return null;
    final String rest = channelName.substring(CASE_CHANNEL_PREFIX.length());
    final String uuidStr = rest.contains("/") ? rest.substring(0, rest.indexOf('/')) : rest;
    try {
        return java.util.UUID.fromString(uuidStr);
    } catch (final IllegalArgumentException e) {
        return null;
    }
}
```

- [ ] **Step 5: Run tests**

```bash
mvn --batch-mode test -pl api -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dtest="CaseChannelTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD" | tail -5
```
Expected: BUILD SUCCESS, Tests run: 5.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/main/java/io/casehub/api/model/CaseChannel.java api/src/test/java/io/casehub/api/model/CaseChannelTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add CaseChannel.parseCaseId(channelName) static utility

Safely extracts case UUID from 'case-{caseId}/{purpose}' channel naming convention.
Returns null for non-case channels or malformed UUIDs without throwing.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 6 — Actor-state module (`casehub-engine` repo)

### Task 8: Module structure and pom.xml

**Files:**
- Create: `actor-state/pom.xml`
- Modify: `pom.xml` (engine root — add `<module>actor-state</module>`)

- [ ] **Step 1: Add module to engine root pom.xml**

In `/Users/mdproctor/claude/casehub/engine/pom.xml`, add `<module>actor-state</module>` after `<module>work-adapter</module>`:

```xml
        <module>work-adapter</module>
        <module>actor-state</module>
```

- [ ] **Step 2: Create actor-state/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-engine-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-engine-actor-state</artifactId>
    <name>Case Hub :: Engine :: Actor State</name>
    <description>GET /actors/{actorId}/state — unified actor workload view assembled from ledger, work, qhorus, and engine via ActorStateContributor SPI</description>

    <dependencies>
        <!-- SPI interfaces (ActorStateContributor, ActorStateAccumulator) -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
            <version>${casehub.version}</version>
        </dependency>

        <!-- WorkerExecutionManager SPI -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-common</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- quarkus-rest + casehub-engine-api (CaseChannel) transitively -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- LedgerActorStateContributor — TrustGateService, allCapabilityScores -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ledger</artifactId>
        </dependency>

        <!-- WorkActorStateContributor — WorkItemStore, WorkItemCallerRef -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work</artifactId>
        </dependency>

        <!-- QhorusActorStateContributor — CommitmentStore, ChannelStore -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-qhorus</artifactId>
        </dependency>

        <!-- ManagedExecutor — CDI-context-safe parallel contributor execution -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-context-propagation</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit-mockito</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <parameters>true</parameters>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

Note: `${casehub.version}` is used for platform-api (cross-repo). Check engine parent pom for the property name — it may be `${version.io.casehub}` or `${casehub.version}`. Run:
```bash
grep "casehub.version\|version.io.casehub" /Users/mdproctor/claude/casehub/engine/pom.xml | head -5
```
Use whichever is defined. If neither, check how `casehub-ledger` dependency is declared in other engine modules and follow the same pattern.

- [ ] **Step 3: Create package skeleton**

```bash
mkdir -p /Users/mdproctor/claude/casehub/engine/actor-state/src/main/java/io/casehub/actorstate
mkdir -p /Users/mdproctor/claude/casehub/engine/actor-state/src/test/java/io/casehub/actorstate
```

- [ ] **Step 4: Verify engine parent compiles with new module**

```bash
mvn --batch-mode compile -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -q 2>&1 | tail -10
```
Expected: BUILD SUCCESS (module exists, no sources yet).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/ pom.xml
git -C /Users/mdproctor/claude/casehub/engine commit -m "chore(parent#56): scaffold casehub-engine-actor-state module

Empty module with pom.xml, package directories. Platform-api, engine-common,
engine (runtime), ledger, work, qhorus, smallrye-context-propagation deps.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 9: ActorStateResponse DTOs

**Files:**
- Create: `actor-state/src/main/java/io/casehub/actorstate/ActorStateResponse.java`

These types are needed before tests can compile. Define them first.

- [ ] **Step 1: Create ActorStateResponse**

```java
// actor-state/src/main/java/io/casehub/actorstate/ActorStateResponse.java
package io.casehub.actorstate;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Response body for GET /actors/{actorId}/state.
 *
 * <p>{@code trustScore: null} means the actor has no computed score yet (not zero trust).
 * {@code sourceWarnings} is absent (not {}) when all sources succeeded — use @JsonInclude(NON_NULL).
 * {@code engineActiveCaseIds} is scoped to active Quartz jobs only — not an exhaustive case list.
 */
public record ActorStateResponse(
        String actorId,
        Instant retrievedAt,
        Double trustScore,
        Map<String, Double> capabilityScores,
        List<WorkItemSummary> activeWorkItems,
        List<CommitmentSummary> openCommitments,
        List<UUID> engineActiveCaseIds,
        List<String> sources,
        @JsonInclude(JsonInclude.Include.NON_NULL) Map<String, String> sourceWarnings) {

    public record WorkItemSummary(
            UUID id,
            String title,
            String status,
            String category,
            UUID caseId) {}

    public record CommitmentSummary(
            UUID commitmentId,
            UUID channelId,
            UUID caseId,
            String state,
            Instant expiresAt) {}
}
```

- [ ] **Step 2: Compile**

```bash
mvn --batch-mode compile -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/main/java/io/casehub/actorstate/ActorStateResponse.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add ActorStateResponse DTOs

WorkItemSummary and CommitmentSummary nested records.
sourceWarnings @JsonInclude(NON_NULL) — absent not {} when all sources succeed.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 10: ActorStateAccumulatorImpl

**Files:**
- Create: `actor-state/src/main/java/io/casehub/actorstate/ActorStateAccumulatorImpl.java`

- [ ] **Step 1: Create ActorStateAccumulatorImpl**

```java
// actor-state/src/main/java/io/casehub/actorstate/ActorStateAccumulatorImpl.java
package io.casehub.actorstate;

import io.casehub.platform.api.actor.ActorStateAccumulator;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Thread-safe accumulator for actor state data. Contributors run concurrently via ManagedExecutor.
 *
 * <p>markSucceeded() and markFailed() are package-private — called by ActorStateAggregator only.
 * Contributors never call these methods; they are not on the ActorStateAccumulator interface.
 */
class ActorStateAccumulatorImpl implements ActorStateAccumulator {

    private final String actorId;
    private final AtomicReference<Double> trustScore = new AtomicReference<>(null);
    private final ConcurrentHashMap<String, Double> capabilityScores = new ConcurrentHashMap<>();
    private final CopyOnWriteArrayList<ActorStateResponse.WorkItemSummary> workItems = new CopyOnWriteArrayList<>();
    private final CopyOnWriteArrayList<ActorStateResponse.CommitmentSummary> commitments = new CopyOnWriteArrayList<>();
    private final CopyOnWriteArrayList<UUID> engineActiveCaseIds = new CopyOnWriteArrayList<>();
    private final CopyOnWriteArrayList<String> sources = new CopyOnWriteArrayList<>();
    private final ConcurrentHashMap<String, String> warnings = new ConcurrentHashMap<>();

    ActorStateAccumulatorImpl(final String actorId) {
        this.actorId = actorId;
    }

    @Override public void trustScore(final Double score) { trustScore.set(score); }
    @Override public void capabilityScore(final String capability, final double score) { capabilityScores.put(capability, score); }
    @Override public void workItem(final UUID id, final String title, final String status, final String category, final UUID caseId) {
        workItems.add(new ActorStateResponse.WorkItemSummary(id, title, status, category, caseId));
    }
    @Override public void commitment(final UUID commitmentId, final UUID channelId, final UUID caseId, final String state, final Instant expiresAt) {
        commitments.add(new ActorStateResponse.CommitmentSummary(commitmentId, channelId, caseId, state, expiresAt));
    }
    @Override public void engineActiveCaseId(final UUID caseId) { engineActiveCaseIds.add(caseId); }

    /** Called by ActorStateAggregator after contributor.contribute() succeeds. */
    void markSucceeded(final String sourceName) { sources.add(sourceName); }

    /** Called by ActorStateAggregator when contributor.contribute() throws. */
    void markFailed(final String sourceName, final String reason) { warnings.put(sourceName, reason); }

    ActorStateResponse build() {
        return new ActorStateResponse(
                actorId,
                Instant.now(),
                trustScore.get(),
                Collections.unmodifiableMap(new HashMap<>(capabilityScores)),
                new ArrayList<>(workItems),
                new ArrayList<>(commitments),
                new ArrayList<>(engineActiveCaseIds),
                new ArrayList<>(sources),
                warnings.isEmpty() ? null : Collections.unmodifiableMap(new HashMap<>(warnings)));
    }
}
```

- [ ] **Step 2: Compile**

```bash
mvn --batch-mode compile -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -q 2>&1 | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/main/java/io/casehub/actorstate/ActorStateAccumulatorImpl.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add ActorStateAccumulatorImpl — thread-safe CDI contributor output collector

ConcurrentHashMap + CopyOnWriteArrayList for concurrent contributor writes.
markSucceeded/markFailed package-private — aggregator only, not on SPI interface.
sourceWarnings null (not empty map) when all sources succeed.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 11: Contributor implementations (4 classes)

**Files:**
- Create: `actor-state/src/main/java/io/casehub/actorstate/LedgerActorStateContributor.java`
- Create: `actor-state/src/main/java/io/casehub/actorstate/WorkActorStateContributor.java`
- Create: `actor-state/src/main/java/io/casehub/actorstate/QhorusActorStateContributor.java`
- Create: `actor-state/src/main/java/io/casehub/actorstate/EngineActorStateContributor.java`

- [ ] **Step 1: Create LedgerActorStateContributor**

```java
// actor-state/src/main/java/io/casehub/actorstate/LedgerActorStateContributor.java
package io.casehub.actorstate;

import io.casehub.ledger.runtime.service.TrustGateService;
import io.casehub.platform.api.actor.ActorStateAccumulator;
import io.casehub.platform.api.actor.ActorStateContributor;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class LedgerActorStateContributor implements ActorStateContributor {

    @Inject TrustGateService trustGateService;

    @Override
    public String sourceName() { return "ledger"; }

    @Override
    public void contribute(final String actorId, final ActorStateAccumulator acc) {
        // Atomic: collect all data before calling accumulator
        // findScore().map(s -> s.trustScore) — primitive double boxed to Double; null when no score
        final Double globalScore = trustGateService.findScore(actorId)
                .map(s -> s.trustScore)
                .orElse(null);
        final Map<String, Double> capScores = trustGateService.allCapabilityScores(actorId);
        acc.trustScore(globalScore);
        capScores.forEach(acc::capabilityScore);
    }
}
```

- [ ] **Step 2: Create WorkActorStateContributor**

```java
// actor-state/src/main/java/io/casehub/actorstate/WorkActorStateContributor.java
package io.casehub.actorstate;

import io.casehub.platform.api.actor.ActorStateAccumulator;
import io.casehub.platform.api.actor.ActorStateContributor;
import io.casehub.work.api.WorkItemCallerRef;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.repository.WorkItemQuery;
import io.casehub.work.runtime.repository.WorkItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.List;

@ApplicationScoped
public class WorkActorStateContributor implements ActorStateContributor {

    @Inject WorkItemStore workItemStore;

    @Override
    public String sourceName() { return "work"; }

    @Override
    public void contribute(final String actorId, final ActorStateAccumulator acc) {
        // inbox(actorId, null, null): first null = candidateGroups, second = candidateUserId
        // PENDING excluded: candidate but hasn't claimed — not active work
        // SUSPENDED included: still obligated to complete (paused, not released)
        // Atomic: scan() returns eager List<WorkItem> — collect fully before calling acc
        final var items = workItemStore.scan(
                WorkItemQuery.inbox(actorId, null, null)
                        .toBuilder()
                        .statusIn(List.of(
                                WorkItemStatus.ASSIGNED,
                                WorkItemStatus.IN_PROGRESS,
                                WorkItemStatus.SUSPENDED))
                        .build());
        items.forEach(wi -> acc.workItem(
                wi.id,
                wi.title,
                wi.status != null ? wi.status.name() : null,
                wi.category,
                WorkItemCallerRef.parseCaseId(wi.callerRef)));
    }
}
```

- [ ] **Step 3: Create QhorusActorStateContributor**

```java
// actor-state/src/main/java/io/casehub/actorstate/QhorusActorStateContributor.java
package io.casehub.actorstate;

import io.casehub.api.model.CaseChannel;
import io.casehub.platform.api.actor.ActorStateAccumulator;
import io.casehub.platform.api.actor.ActorStateContributor;
import io.casehub.qhorus.runtime.store.ChannelStore;
import io.casehub.qhorus.runtime.store.CommitmentStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class QhorusActorStateContributor implements ActorStateContributor {

    @Inject CommitmentStore commitmentStore;
    @Inject ChannelStore channelStore;

    @Override
    public String sourceName() { return "qhorus"; }

    @Override
    public void contribute(final String actorId, final ActorStateAccumulator acc) {
        // Atomic: collect all data before calling accumulator
        final var open = commitmentStore.findOpenByObligor(actorId);
        final Set<UUID> channelIds = open.stream()
                .map(c -> c.channelId)
                .collect(Collectors.toSet());
        // Batch channel lookup — one IN(?) query via findByIds
        final Map<UUID, String> channelNames = channelStore.findByIds(channelIds).stream()
                .collect(Collectors.toMap(ch -> ch.id, ch -> ch.name));
        open.forEach(c -> acc.commitment(
                c.id,
                c.channelId,
                CaseChannel.parseCaseId(channelNames.get(c.channelId)),
                c.state.name(),
                c.expiresAt));
    }
}
```

- [ ] **Step 4: Create EngineActorStateContributor**

```java
// actor-state/src/main/java/io/casehub/actorstate/EngineActorStateContributor.java
package io.casehub.actorstate;

import io.casehub.engine.common.spi.scheduler.WorkerExecutionManager;
import io.casehub.platform.api.actor.ActorStateAccumulator;
import io.casehub.platform.api.actor.ActorStateContributor;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class EngineActorStateContributor implements ActorStateContributor {

    @Inject WorkerExecutionManager executionManager;

    @Override
    public String sourceName() { return "engine"; }

    @Override
    public void contribute(final String actorId, final ActorStateAccumulator acc) {
        // actorId == workerId — same string, different naming convention per layer
        // Quartz in-memory scan; best-effort snapshot (TOCTOU acceptable for monitoring)
        final var caseIds = executionManager.getActiveCaseIds(actorId);
        caseIds.forEach(acc::engineActiveCaseId);
    }
}
```

- [ ] **Step 5: Compile all contributors**

```bash
mvn --batch-mode compile -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | tail -10
```
Expected: BUILD SUCCESS. If compile errors on imports, verify exact package paths using:
```bash
find /Users/mdproctor/claude/casehub/ledger -name "TrustGateService.java" | head -1
find /Users/mdproctor/claude/casehub/qhorus -name "CommitmentStore.java" | head -1
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/main/java/io/casehub/actorstate/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add four ActorStateContributor implementations

LedgerActorStateContributor: global + capability scores via TrustGateService.
WorkActorStateContributor: ASSIGNED/IN_PROGRESS/SUSPENDED WorkItems via WorkItemStore.
QhorusActorStateContributor: open Commitments + batch channel name lookup.
EngineActorStateContributor: active Quartz job case IDs via WorkerExecutionManager.
All follow atomic contribution contract (collect before accumulate).

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 12: ActorStateAggregator (blocking) + tests

**Files:**
- Create: `actor-state/src/main/java/io/casehub/actorstate/ActorStateAggregator.java`
- Create: `actor-state/src/test/java/io/casehub/actorstate/ActorStateAggregatorTest.java`

- [ ] **Step 1: Write ActorStateAggregatorTest (plain JUnit, no CDI)**

```java
// actor-state/src/test/java/io/casehub/actorstate/ActorStateAggregatorTest.java
package io.casehub.actorstate;

import io.casehub.platform.api.actor.ActorStateContributor;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class ActorStateAggregatorTest {

    private static ActorStateContributor contributor(String name, ThrowingContributor body) {
        return new ActorStateContributor() {
            @Override public String sourceName() { return name; }
            @Override public void contribute(String actorId, io.casehub.platform.api.actor.ActorStateAccumulator acc) {
                body.contribute(actorId, acc);
            }
        };
    }

    @FunctionalInterface
    interface ThrowingContributor {
        void contribute(String actorId, io.casehub.platform.api.actor.ActorStateAccumulator acc);
    }

    @Test
    void allSources_healthy_completesResponse() {
        UUID caseId = UUID.randomUUID();
        var agg = new ActorStateAggregator(List.of(
                contributor("ledger", (id, acc) -> {
                    acc.trustScore(0.82);
                    acc.capabilityScore("sar-drafting", 0.79);
                }),
                contributor("work", (id, acc) ->
                        acc.workItem(UUID.randomUUID(), "title", "IN_PROGRESS", "aml", caseId)),
                contributor("qhorus", (id, acc) ->
                        acc.commitment(UUID.randomUUID(), UUID.randomUUID(), caseId, "OPEN", Instant.now())),
                contributor("engine", (id, acc) ->
                        acc.engineActiveCaseId(caseId))));

        var resp = agg.forActor("agent-x");

        assertEquals(0.82, resp.trustScore(), 0.001);
        assertEquals(0.79, resp.capabilityScores().get("sar-drafting"), 0.001);
        assertEquals(1, resp.activeWorkItems().size());
        assertEquals(1, resp.openCommitments().size());
        assertEquals(1, resp.engineActiveCaseIds().size());
        assertEquals(List.of("ledger", "work", "qhorus", "engine"), resp.sources().stream().sorted().toList());
        assertNull(resp.sourceWarnings());
        assertNotNull(resp.retrievedAt());
    }

    @Test
    void oneSourceThrows_excludedFromSourcesAndInWarnings_othersIntact() {
        var agg = new ActorStateAggregator(List.of(
                contributor("ledger", (id, acc) -> acc.trustScore(0.7)),
                contributor("work", (id, acc) -> { throw new RuntimeException("DB down"); }),
                contributor("qhorus", (id, acc) -> {}),
                contributor("engine", (id, acc) -> {})));

        var resp = agg.forActor("agent-x");

        assertEquals(0.7, resp.trustScore(), 0.001);
        assertFalse(resp.sources().contains("work"));
        assertTrue(resp.sources().contains("ledger"));
        assertNotNull(resp.sourceWarnings());
        assertTrue(resp.sourceWarnings().containsKey("work"));
        assertTrue(resp.activeWorkItems().isEmpty());
    }

    @Test
    void noScore_trustScoreNull_notZero() {
        var agg = new ActorStateAggregator(List.of(
                contributor("ledger", (id, acc) -> acc.trustScore(null))));

        var resp = agg.forActor("unknown-actor");
        assertNull(resp.trustScore());
    }

    @Test
    void workItemWithNullCallerRef_caseIdNull_noThrow() {
        var agg = new ActorStateAggregator(List.of(
                contributor("work", (id, acc) ->
                        acc.workItem(UUID.randomUUID(), null, "ASSIGNED", null, null))));

        var resp = agg.forActor("agent-x");
        assertEquals(1, resp.activeWorkItems().size());
        assertNull(resp.activeWorkItems().get(0).caseId());
        assertNull(resp.activeWorkItems().get(0).title());
    }

    @Test
    void unknownActor_validEmptyResponse_allSourcesPresent() {
        var agg = new ActorStateAggregator(List.of(
                contributor("ledger", (id, acc) -> {}),
                contributor("work", (id, acc) -> {}),
                contributor("qhorus", (id, acc) -> {}),
                contributor("engine", (id, acc) -> {})));

        var resp = agg.forActor("unknown");

        assertNull(resp.trustScore());
        assertTrue(resp.activeWorkItems().isEmpty());
        assertTrue(resp.openCommitments().isEmpty());
        assertTrue(resp.engineActiveCaseIds().isEmpty());
        assertEquals(4, resp.sources().size());
        assertNull(resp.sourceWarnings());
    }
}
```

- [ ] **Step 2: Run — expect compile error (ActorStateAggregator not found)**

```bash
mvn --batch-mode test-compile -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | tail -10
```

- [ ] **Step 3: Create ActorStateAggregator**

```java
// actor-state/src/main/java/io/casehub/actorstate/ActorStateAggregator.java
package io.casehub.actorstate;

import io.casehub.platform.api.actor.ActorStateContributor;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import org.eclipse.microprofile.context.ManagedExecutor;
import org.jboss.logging.Logger;

/**
 * Assembles actor state from all registered {@link ActorStateContributor} beans in parallel.
 *
 * <p>Uses {@link ManagedExecutor} (MicroProfile Context Propagation) to propagate CDI request
 * context and Hibernate session across threads — required for Panache-backed stores.
 *
 * <p>Each contributor is independent: a contributor failure excludes that source from
 * {@code sources} and adds a warning to {@code sourceWarnings} without affecting other sources.
 */
@ApplicationScoped
public class ActorStateAggregator {

    private static final Logger LOG = Logger.getLogger(ActorStateAggregator.class);

    private final List<ActorStateContributor> contributors;
    private final ManagedExecutor executor;

    /** CDI constructor — injects all ActorStateContributor beans and ManagedExecutor. */
    @Inject
    public ActorStateAggregator(@Any final Instance<ActorStateContributor> contributors,
                                 final ManagedExecutor executor) {
        this.contributors = contributors.stream().toList();
        this.executor = executor;
    }

    /** Test constructor — accepts an explicit contributor list and no-thread execution. */
    ActorStateAggregator(final List<ActorStateContributor> contributors) {
        this.contributors = contributors;
        this.executor = null; // test path — uses sequential execution
    }

    /**
     * Assembles actor state from all contributors.
     *
     * @param actorId the actor identity string (must be consistent across all backends)
     * @return assembled response; always returns 200 even when some sources fail
     */
    public ActorStateResponse forActor(final String actorId) {
        final var accumulator = new ActorStateAccumulatorImpl(actorId);

        if (executor != null) {
            // CDI production path — parallel via ManagedExecutor
            final List<CompletableFuture<Void>> futures = new ArrayList<>();
            for (final ActorStateContributor c : contributors) {
                futures.add(CompletableFuture.runAsync(
                        () -> runContributor(c, actorId, accumulator), executor));
            }
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        } else {
            // Test path — sequential (no executor injection in plain JUnit)
            contributors.forEach(c -> runContributor(c, actorId, accumulator));
        }

        return accumulator.build();
    }

    private void runContributor(final ActorStateContributor c,
                                 final String actorId,
                                 final ActorStateAccumulatorImpl accumulator) {
        try {
            c.contribute(actorId, accumulator);
            accumulator.markSucceeded(c.sourceName());
        } catch (final Exception e) {
            LOG.warnf("source %s failed for actorId=%s: %s", c.sourceName(), actorId, e.getMessage());
            accumulator.markFailed(c.sourceName(), e.getMessage());
        }
    }
}
```

- [ ] **Step 4: Run tests**

```bash
mvn --batch-mode test -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dtest="ActorStateAggregatorTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -10
```
Expected: BUILD SUCCESS, all 5 tests pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/main/java/io/casehub/actorstate/ActorStateAggregator.java actor-state/src/test/java/io/casehub/actorstate/ActorStateAggregatorTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add ActorStateAggregator — parallel multi-source actor state assembly

ManagedExecutor for CDI-context-safe parallel contributor execution.
Per-source failure isolation: failed contributors go to sourceWarnings, not 500.
Test constructor (executor=null) runs sequentially for plain JUnit tests.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 13: ActorStateResource (blocking) + resource test

**Files:**
- Create: `actor-state/src/main/java/io/casehub/actorstate/ActorStateResource.java`
- Create: `actor-state/src/test/java/io/casehub/actorstate/ActorStateResourceTest.java`

- [ ] **Step 1: Create ActorStateResource**

```java
// actor-state/src/main/java/io/casehub/actorstate/ActorStateResource.java
package io.casehub.actorstate;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

/**
 * REST endpoint for GET /actors/{actorId}/state.
 *
 * <p>Authorization: inherits application-level security. No platform-level role restriction.
 * Applications add {@code @Authenticated} or {@code @RolesAllowed} via their security config.
 *
 * <p>tenancyId is not an explicit parameter — each store handles tenant scoping implicitly
 * (Panache via security context; qhorus is single-tenant; trust scores are by actorId;
 * Quartz is in-memory without tenancy).
 *
 * <p>Active only when {@code casehub.qhorus.reactive.enabled} is false or absent (default).
 */
@Path("/actors")
@Produces(MediaType.APPLICATION_JSON)
@ApplicationScoped
@io.quarkus.arc.properties.UnlessBuildProperty(
        name = "casehub.qhorus.reactive.enabled",
        stringValue = "true",
        enableIfMissing = true)
public class ActorStateResource {

    @Inject
    ActorStateAggregator aggregator;

    @GET
    @Path("/{actorId}/state")
    public ActorStateResponse getActorState(@PathParam("actorId") final String actorId) {
        return aggregator.forActor(actorId);
    }
}
```

- [ ] **Step 2: Create ActorStateResourceTest**

This test uses `@QuarkusTest` with `@InjectMock` on the aggregator.

```java
// actor-state/src/test/java/io/casehub/actorstate/ActorStateResourceTest.java
package io.casehub.actorstate;

import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;
import static org.mockito.ArgumentMatchers.anyString;
import static org.mockito.Mockito.when;

@QuarkusTest
class ActorStateResourceTest {

    @InjectMock
    ActorStateAggregator aggregator;

    private static ActorStateResponse successResponse() {
        return new ActorStateResponse(
                "agent-x",
                Instant.now(),
                0.82,
                Map.of("sar-drafting", 0.79),
                List.of(new ActorStateResponse.WorkItemSummary(UUID.randomUUID(), "title", "IN_PROGRESS", "aml", UUID.randomUUID())),
                List.of(),
                List.of(UUID.randomUUID()),
                List.of("ledger", "work", "qhorus", "engine"),
                null);
    }

    @Test
    void get_returnsOk_withCorrectShape() {
        when(aggregator.forActor("agent-x")).thenReturn(successResponse());

        given().when().get("/actors/agent-x/state")
                .then()
                .statusCode(200)
                .contentType("application/json")
                .body("actorId", equalTo("agent-x"))
                .body("trustScore", equalTo(0.82f))
                .body("sources", hasItems("ledger", "work", "qhorus", "engine"))
                .body("retrievedAt", notNullValue())
                .body("engineActiveCaseIds", hasSize(1));
    }

    @Test
    void get_sourceWarnings_absentWhenAllSucceeded() {
        when(aggregator.forActor(anyString())).thenReturn(successResponse());

        given().when().get("/actors/agent-x/state")
                .then()
                .statusCode(200)
                .body("sourceWarnings", nullValue());
    }

    @Test
    void get_sourceWarnings_presentWhenSourceFailed() {
        var resp = new ActorStateResponse("agent-x", Instant.now(), null, Map.of(),
                List.of(), List.of(), List.of(), List.of("ledger", "qhorus", "engine"),
                Map.of("work", "DB timeout"));
        when(aggregator.forActor(anyString())).thenReturn(resp);

        given().when().get("/actors/agent-x/state")
                .then()
                .statusCode(200)
                .body("sourceWarnings.work", equalTo("DB timeout"))
                .body("sources", not(hasItem("work")));
    }
}
```

Note: `ActorStateResourceTest` needs `quarkus-test-h2` or similar datasource config in `src/test/resources/application.properties`. Check if existing engine test infrastructure can be reused, or add a minimal test application.properties:

```properties
# actor-state/src/test/resources/application.properties
quarkus.http.test-port=0
```

- [ ] **Step 3: Run resource test**

```bash
mvn --batch-mode test -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dtest="ActorStateResourceTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL|ERROR" | tail -15
```

If test fails due to missing CDI beans (unsatisfied dependencies), add `quarkus.arc.exclude-types` in test application.properties to exclude unnecessary contributors. The mock already covers `ActorStateAggregator`.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/main/java/io/casehub/actorstate/ActorStateResource.java actor-state/src/test/java/io/casehub/actorstate/ActorStateResourceTest.java actor-state/src/test/resources/
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add ActorStateResource GET /actors/{actorId}/state

Blocked when casehub.qhorus.reactive.enabled=true (reactive path takes over).
No platform role restriction — applications add auth via their own security config.
Test: @QuarkusTest + @InjectMock ActorStateAggregator verifies JSON shape.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 14: Reactive path (ReactiveActorStateAggregator + ReactiveActorStateResource)

**Files:**
- Create: `actor-state/src/main/java/io/casehub/actorstate/ReactiveActorStateAggregator.java`
- Create: `actor-state/src/main/java/io/casehub/actorstate/ReactiveActorStateResource.java`
- Create: `actor-state/src/test/java/io/casehub/actorstate/ReactiveActorStateAggregatorTest.java`

- [ ] **Step 1: Create ReactiveActorStateAggregator**

```java
// actor-state/src/main/java/io/casehub/actorstate/ReactiveActorStateAggregator.java
package io.casehub.actorstate;

import io.casehub.platform.api.actor.ActorStateContributor;
import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.List;
import org.jboss.logging.Logger;

/**
 * Reactive variant — active when {@code casehub.qhorus.reactive.enabled=true}.
 *
 * <p>Contributors are always blocking ({@code void contribute()}). This aggregator wraps each on
 * Mutiny's blocking executor — no reactive contributor interface needed, no parity problem.
 * All four contributors work in both reactive and non-reactive modes (qhorus reactive flag is
 * additive: JpaCommitmentStore remains available alongside ReactiveJpaCommitmentStore).
 */
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
public class ReactiveActorStateAggregator {

    private static final Logger LOG = Logger.getLogger(ReactiveActorStateAggregator.class);

    private final List<ActorStateContributor> contributors;

    @Inject
    public ReactiveActorStateAggregator(@Any final Instance<ActorStateContributor> contributors) {
        this.contributors = contributors.stream().toList();
    }

    /** Test constructor. */
    ReactiveActorStateAggregator(final List<ActorStateContributor> contributors) {
        this.contributors = contributors;
    }

    public Uni<ActorStateResponse> forActor(final String actorId) {
        final var accumulator = new ActorStateAccumulatorImpl(actorId);
        final var unis = contributors.stream()
                .map(c -> Uni.createFrom().voidItem()
                        .invoke(() -> {
                            try {
                                c.contribute(actorId, accumulator);
                                accumulator.markSucceeded(c.sourceName());
                            } catch (final Exception e) {
                                LOG.warnf("source %s failed for actorId=%s: %s",
                                        c.sourceName(), actorId, e.getMessage());
                                accumulator.markFailed(c.sourceName(), e.getMessage());
                            }
                        })
                        .runSubscriptionOn(Infrastructure.getDefaultBlockingExecutor()))
                .toList();
        return Uni.combine().all().unis(unis).discardItems()
                .map(v -> accumulator.build());
    }
}
```

- [ ] **Step 2: Create ReactiveActorStateResource**

```java
// actor-state/src/main/java/io/casehub/actorstate/ReactiveActorStateResource.java
package io.casehub.actorstate;

import io.quarkus.arc.properties.IfBuildProperty;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

/** Reactive variant — same path as blocking resource; only one is active at build time. */
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@Path("/actors")
@Produces(MediaType.APPLICATION_JSON)
@ApplicationScoped
public class ReactiveActorStateResource {

    @Inject
    ReactiveActorStateAggregator aggregator;

    @GET
    @Path("/{actorId}/state")
    public Uni<ActorStateResponse> getActorState(@PathParam("actorId") final String actorId) {
        return aggregator.forActor(actorId);
    }
}
```

- [ ] **Step 3: Create ReactiveActorStateAggregatorTest**

```java
// actor-state/src/test/java/io/casehub/actorstate/ReactiveActorStateAggregatorTest.java
package io.casehub.actorstate;

import io.casehub.platform.api.actor.ActorStateContributor;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class ReactiveActorStateAggregatorTest {

    private static ActorStateContributor contributor(String name, ThrowingContributor body) {
        return new ActorStateContributor() {
            @Override public String sourceName() { return name; }
            @Override public void contribute(String actorId, io.casehub.platform.api.actor.ActorStateAccumulator acc) {
                body.contribute(actorId, acc);
            }
        };
    }

    @FunctionalInterface
    interface ThrowingContributor {
        void contribute(String actorId, io.casehub.platform.api.actor.ActorStateAccumulator acc);
    }

    @Test
    void allSources_healthy_completesUni() {
        var agg = new ReactiveActorStateAggregator(List.of(
                contributor("ledger", (id, acc) -> acc.trustScore(0.75)),
                contributor("work", (id, acc) -> {}),
                contributor("qhorus", (id, acc) -> {}),
                contributor("engine", (id, acc) -> acc.engineActiveCaseId(UUID.randomUUID()))));

        var resp = agg.forActor("agent-x").await().indefinitely();

        assertEquals(0.75, resp.trustScore(), 0.001);
        assertEquals(4, resp.sources().size());
        assertNull(resp.sourceWarnings());
    }

    @Test
    void oneSourceThrows_excludedFromSources_othersIntact() {
        var agg = new ReactiveActorStateAggregator(List.of(
                contributor("ledger", (id, acc) -> acc.trustScore(0.7)),
                contributor("work", (id, acc) -> { throw new RuntimeException("timeout"); })));

        var resp = agg.forActor("agent-x").await().indefinitely();

        assertEquals(0.7, resp.trustScore(), 0.001);
        assertTrue(resp.sources().contains("ledger"));
        assertFalse(resp.sources().contains("work"));
        assertNotNull(resp.sourceWarnings());
    }
}
```

- [ ] **Step 4: Run reactive tests**

```bash
mvn --batch-mode test -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dtest="ReactiveActorStateAggregatorTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/main/java/io/casehub/actorstate/ReactiveActorStateAggregator.java actor-state/src/main/java/io/casehub/actorstate/ReactiveActorStateResource.java actor-state/src/test/java/io/casehub/actorstate/ReactiveActorStateAggregatorTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(parent#56): add ReactiveActorStateAggregator + ReactiveActorStateResource

Active when casehub.qhorus.reactive.enabled=true (build-time gate).
Wraps blocking contributors on Mutiny's blocking executor — no reactive contributor
interface needed. qhorus reactive flag is additive (JpaCommitmentStore stays active).

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 15: ActorStateParityTest (local ArchUnit / JUnit reflection)

**Files:**
- Create: `actor-state/src/test/java/io/casehub/actorstate/ActorStateParityTest.java`

- [ ] **Step 1: Create parity test**

```java
// actor-state/src/test/java/io/casehub/actorstate/ActorStateParityTest.java
package io.casehub.actorstate;

import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;
import java.lang.reflect.Method;
import java.util.Arrays;
import java.util.Set;
import java.util.stream.Collectors;

import static org.junit.jupiter.api.Assertions.*;

/**
 * Verifies that ReactiveActorStateAggregator has a Uni<ActorStateResponse> equivalent
 * for every ActorStateResponse-returning method on ActorStateAggregator.
 *
 * Local test — do not add casehub-engine-actor-state to casehub-ledger's test classpath
 * (would invert the dependency direction).
 */
class ActorStateParityTest {

    @Test
    void reactiveAggregatorHasUniEquivalentForEveryBlockingMethod() {
        final Set<String> blockingMethodNames = Arrays.stream(ActorStateAggregator.class.getMethods())
                .filter(m -> m.getReturnType().equals(ActorStateResponse.class))
                .filter(m -> m.getDeclaringClass().equals(ActorStateAggregator.class))
                .map(Method::getName)
                .collect(Collectors.toSet());

        assertFalse(blockingMethodNames.isEmpty(),
                "ActorStateAggregator must have at least one ActorStateResponse-returning method");

        for (final String name : blockingMethodNames) {
            final Method reactiveMethod = Arrays.stream(ReactiveActorStateAggregator.class.getMethods())
                    .filter(m -> m.getName().equals(name))
                    .findFirst()
                    .orElseGet(() -> fail("ReactiveActorStateAggregator missing method: " + name));
            assertEquals(Uni.class, reactiveMethod.getReturnType(),
                    "Method " + name + " in ReactiveActorStateAggregator must return Uni<ActorStateResponse>");
        }
    }
}
```

- [ ] **Step 2: Run parity test**

```bash
mvn --batch-mode test -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dtest="ActorStateParityTest" -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -5
```
Expected: BUILD SUCCESS.

- [ ] **Step 3: Run all actor-state tests together**

```bash
mvn --batch-mode test -pl actor-state -am -f /Users/mdproctor/claude/casehub/engine/pom.xml -Dsurefire.failIfNoSpecifiedTests=false 2>&1 | grep -E "Tests run|BUILD|FAIL" | tail -10
```
Expected: all tests pass, BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add actor-state/src/test/java/io/casehub/actorstate/ActorStateParityTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(parent#56): add ActorStateParityTest — local blocking/reactive method parity check

Verifies ReactiveActorStateAggregator has Uni<T> equivalent for every
ActorStateResponse-returning method on ActorStateAggregator.
Local test — avoids inverting the ledger→engine dependency direction.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 7 — Application integration

### Task 16: AML and devtown pom additions

**Repos:** `/Users/mdproctor/claude/casehub/aml`, `/Users/mdproctor/claude/casehub/devtown`

- [ ] **Step 1: Check current branch in both repos**

```bash
git -C /Users/mdproctor/claude/casehub/aml branch --show-current
git -C /Users/mdproctor/claude/casehub/devtown branch --show-current
```
Both must be on `main` before modifying.

- [ ] **Step 2: Add dependency to AML app/pom.xml**

In `/Users/mdproctor/claude/casehub/aml/app/pom.xml`, add after the `casehub-engine-ledger` dependency:

```xml
<!-- Actor state view — GET /actors/{actorId}/state -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-actor-state</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

Verify the version property name from the AML pom before adding.

- [ ] **Step 3: Build AML to verify no new compile errors**

```bash
mvn --batch-mode compile -pl app -f /Users/mdproctor/claude/casehub/aml/pom.xml -q 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 4: Add dependency to devtown app/pom.xml**

Same dep in `/Users/mdproctor/claude/casehub/devtown/app/pom.xml`.

- [ ] **Step 5: Build devtown**

```bash
mvn --batch-mode compile -pl app -f /Users/mdproctor/claude/casehub/devtown/pom.xml -q 2>&1 | tail -10
```
Expected: BUILD SUCCESS.

- [ ] **Step 6: Commit both**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/pom.xml
git -C /Users/mdproctor/claude/casehub/aml commit -m "feat(parent#56): add casehub-engine-actor-state dep — enables GET /actors/{actorId}/state

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"

git -C /Users/mdproctor/claude/casehub/devtown add app/pom.xml
git -C /Users/mdproctor/claude/casehub/devtown commit -m "feat(parent#56): add casehub-engine-actor-state dep — enables GET /actors/{actorId}/state

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Phase 8 — Documentation

### Task 17: CLAUDE.md updates per repo

- [ ] **Step 1: casehub-engine CLAUDE.md**

Add `casehub-engine-actor-state` to the module listing. Add a note that `WorkerExecutionManager.getActiveCaseIds(workerId)` was added. Check what section documents modules and add an entry.

```bash
grep -n "actor-state\|work-adapter\|module" /Users/mdproctor/claude/casehub/engine/CLAUDE.md | head -10
```

- [ ] **Step 2: casehub-qhorus CLAUDE.md**

Document new `CommitmentStore.findOpenByObligor(obligor)` and `ChannelStore.findByIds(ids)` methods.

- [ ] **Step 3: casehub-ledger CLAUDE.md**

Document new `TrustGateService.allCapabilityScores(actorId)` method.

- [ ] **Step 4: casehub-work CLAUDE.md**

Document new `WorkItemCallerRef` utility class and callerRef format.

- [ ] **Step 5: Commit CLAUDE.md updates**

Each repo gets its own commit:
```bash
git -C <repo> add CLAUDE.md
git -C <repo> commit -m "docs(parent#56): document new SPI additions and actor-state module

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 18: docs/repos/ deep-dive updates in parent repo

The parent repo (`/Users/mdproctor/claude/casehub/parent`) has deep-dive docs for each repo.

- [ ] **Step 1: Update casehub-engine.md**

```bash
grep -n "module\|actor-state\|getActive" /Users/mdproctor/claude/casehub/parent/docs/repos/casehub-engine.md | head -10
```
Add actor-state module entry and `getActiveCaseIds` SPI addition.

- [ ] **Step 2: Update casehub-qhorus.md**

Add `findOpenByObligor(obligor)` and `findByIds` to CommitmentStore and ChannelStore sections.

- [ ] **Step 3: Update casehub-ledger.md**

Add `allCapabilityScores` to TrustGateService section.

- [ ] **Step 4: Update casehub-work.md**

Add `WorkItemCallerRef` and document callerRef format.

- [ ] **Step 5: Commit parent docs**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(parent#56): update repo deep-dives for actor state SPI additions

casehub-engine: actor-state module, getActiveCaseIds SPI.
casehub-qhorus: CommitmentStore.findOpenByObligor, ChannelStore.findByIds.
casehub-ledger: TrustGateService.allCapabilityScores.
casehub-work: WorkItemCallerRef utility, callerRef format.

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Self-Review

**Spec coverage check:**
- ✅ ActorStateContributor + ActorStateAccumulator in platform-api (Task 1)
- ✅ CommitmentStore.findOpenByObligor + contract tests (Task 2)
- ✅ ChannelStore.findByIds + contract tests (Task 3)
- ✅ TrustGateService.allCapabilityScores (Task 4)
- ✅ WorkItemCallerRef.parseCaseId (Task 5)
- ✅ WorkerExecutionManager.getActiveCaseIds (Task 6)
- ✅ CaseChannel.parseCaseId (Task 7)
- ✅ casehub-engine-actor-state module (Tasks 8–15)
- ✅ trustScore null contract (AccumulatorImpl.build() + response test)
- ✅ sourceWarnings absent when all succeed (AccumulatorImpl.build() → null)
- ✅ Parallel execution via ManagedExecutor (ActorStateAggregator)
- ✅ Reactive path (Tasks 14, 15)
- ✅ Parity test (Task 15)
- ✅ Application integration AML + devtown (Task 16)
- ✅ CLAUDE.md + docs updates (Tasks 17, 18)
- ✅ retrievedAt timestamp (in AccumulatorImpl.build(), Instant.now())
- ✅ Atomic contribution contract (Javadoc on interface + contributor implementations)
- ✅ engineActiveCaseIds naming (field name in response record)

**Type consistency:** `ActorStateAccumulatorImpl` implements `ActorStateAccumulator`. `ActorStateAggregator` constructor takes `Instance<ActorStateContributor>`. Test constructor takes `List<ActorStateContributor>`. `forActor()` is `public` in both aggregators. `markSucceeded`/`markFailed` are package-private on impl, not on interface — consistent throughout.

**Missing from plan:** None identified.
