# Actor State View — Design Spec

**Issue:** casehubio/parent#56
**Date:** 2026-06-01 (revised post-review)
**Status:** Approved for implementation
**ADR required:** Yes — first use-case-specific SPI in casehub-platform-api

---

## Problem

Four platform modules each hold a complementary slice of an actor's current state:

| Source | Data | Gap |
|--------|------|-----|
| `casehub-ledger` | Trust scores (global + per-capability) | No cross-module query API |
| `casehub-work` | Active human-task WorkItems | No actor-scoped state endpoint |
| `casehub-qhorus` | Open agent Commitments (what the actor owes) | `findOpenByObligor` requires channelId today |
| `casehub-engine` | Cases where the engine has this worker executing | Count only (`getActiveWorkCount`), no UUIDs |

No endpoint joins these. `ActorState`: zero results across the entire codebase.

---

## Architecture

### ActorStateContributor SPI — placed in `casehub-platform-api`

Two interfaces using only `java.util.*`, `java.time.*`, `java.util.UUID`. No domain types cross the boundary, satisfying the platform-api scope rule ("behaviour interfaces needed by ≥4 peer repos that cannot share them via any single domain *-api module; zero domain types").

```java
// casehub-platform-api
public interface ActorStateContributor {
    String sourceName();
    void contribute(String actorId, ActorStateAccumulator accumulator);
}

public interface ActorStateAccumulator {
    void trustScore(Double score);                         // null = no score computed yet
    void capabilityScore(String capability, double score);
    void workItem(UUID id, String title, String status,
                  String category, UUID caseId);          // caseId null if not engine-created
    void commitment(UUID commitmentId, UUID channelId,
                    UUID caseId, String state,            // caseId null if non-case channel
                    Instant expiresAt);
    void engineActiveCaseId(UUID caseId);
    void markSucceeded(String sourceName);
    void markFailed(String sourceName, String reason);
}
```

**Migration path:** Contributor implementations start in `casehub-engine-actor-state`. When the platform matures, moving `WorkActorStateContributor` into `casehub-work` is a mechanical refactor — the interface already exists in platform-api, no consumer impact.

**ADR scope:** First SPI in platform-api that captures a use-case-specific behaviour (not a generic infrastructure primitive). The decision record must justify why the accumulator/visitor shape satisfies the scope rule (stdlib types only, no domain projections in the interface).

### `casehub-engine-actor-state` module

New adapter module in the engine repo, alongside `engine-work-adapter`.

```
casehub-engine-actor-state
  ├── casehub-platform-api       ActorStateContributor, ActorStateAccumulator
  ├── casehub-engine-common      REST (quarkus-rest already on classpath)
  ├── casehub-ledger             LedgerActorStateContributor
  ├── casehub-work               WorkActorStateContributor
  └── casehub-qhorus             QhorusActorStateContributor
```

`EngineActorStateContributor` uses `WorkerExecutionManager` from `engine-common` — no extra import needed.

**Activation:** applications add `casehub-engine-actor-state` to their pom. AML and devtown already import all required CDI beans.

---

## New SPI additions to existing modules

### 1. `CommitmentStore.findOpenByObligor(String obligor)` — qhorus

Cross-channel obligor query. Existing `findOpenByObligor(obligor, channelId)` requires channelId.

```java
// CommitmentStore interface — new default
// WARNING: This default is a full table scan. JPA-backed implementations MUST override.
// The default exists only for in-memory test implementations.
default List<Commitment> findOpenByObligor(String obligor) {
    return findAllOpen().stream()
            .filter(c -> obligor != null && obligor.equals(c.obligor))
            .toList();
}
```

`JpaCommitmentStore` override:
```java
@Override
public List<Commitment> findOpenByObligor(String obligor) {
    return repo.list("obligor = ?1 AND state NOT IN ?2", obligor, terminalStates());
}
```

**Reactive default** (for `ReactiveCommitmentStore`):
```java
default Uni<List<Commitment>> findOpenByObligor(String obligor) {
    return findAllOpen().map(all -> all.stream()
            .filter(c -> obligor != null && obligor.equals(c.obligor))
            .toList());
}
```

Also add to: `ReactiveJpaCommitmentStore`, `InMemoryCommitmentStore`, `InMemoryReactiveCommitmentStore`.

**Contract test additions** to `CommitmentStoreContractTest`:
- `findOpenByObligor_returnsAcrossMultipleChannels`
- `findOpenByObligor_excludesTerminalStates_crossChannel`
- `findOpenByObligor_nullObligorInStore_notReturnedForActualActor`
- `findOpenByObligor_emptyStore_returnsEmpty`

### 2. `ChannelStore.findByIds(Collection<UUID>)` — qhorus

Single IN(?) batch lookup replacing per-commitment channel queries.

```java
// ChannelStore interface — new default (linear scan for in-memory impls)
default List<Channel> findByIds(Collection<UUID> ids) {
    return ids.stream()
            .map(this::find)
            .filter(Optional::isPresent)
            .map(Optional::get)
            .toList();
}
```

`JpaChannelStore` override with `WHERE id IN (:ids)` JPQL.

### 3. `WorkerExecutionManager.getActiveCaseIds(String workerId)` — engine-common SPI

Returns case UUIDs from Quartz job data. `caseHubInstanceUuid` is already stored per job by `QuartzWorkerSchedulerService.scheduleJob`.

```java
// SPI default — safe no-op, per PP-20260601-81b9e5
// NOTE: workerId here is identical to actorId at the REST layer — same string, different layer name
default List<UUID> getActiveCaseIds(String workerId) {
    return List.of();
}
```

`QuartzWorkerExecutionManager` overrides: same group/key iteration as `getActiveWorkCount`, collects `caseHubInstanceUuid` from job data where `workerId` matches.

### 4. `TrustGateService.allCapabilityScores(String actorId)` — ledger

New method on ledger's service layer. Keeps external consumers out of `ActorTrustScoreRepository`.

```java
// TrustGateService — new method
public Map<String, Double> allCapabilityScores(final String actorId) {
    return repository.findByActorIdAndScoreType(actorId, ScoreType.CAPABILITY).stream()
            .collect(Collectors.toMap(s -> s.capabilityKey, s -> s.trustScore));
}
```

### 5. Parser statics — placed in source modules

**`CaseChannel.parseCaseId(String channelName)`** — static method on existing `CaseChannel` record in `casehub-engine-api`:
```java
public static UUID parseCaseId(String channelName) {
    if (channelName == null || !channelName.startsWith(CASE_CHANNEL_PREFIX)) return null;
    String rest = channelName.substring(CASE_CHANNEL_PREFIX.length());
    String uuidStr = rest.contains("/") ? rest.substring(0, rest.indexOf('/')) : rest;
    try { return UUID.fromString(uuidStr); } catch (IllegalArgumentException e) { return null; }
}
```

**`WorkItemCallerRef.parseCaseId(String callerRef)`** — new utility class in `casehub-work-api`:
```java
public final class WorkItemCallerRef {
    // Format: "caseId:planItemId" for engine-created WorkItems; anything else → null caseId
    public static UUID parseCaseId(String callerRef) {
        if (callerRef == null || !callerRef.contains(":")) return null;
        try { return UUID.fromString(callerRef.split(":")[0]); }
        catch (IllegalArgumentException e) { return null; }
    }
}
```

---

## Components

### Four contributor implementations (all in `casehub-engine-actor-state`)

**`LedgerActorStateContributor`**
```java
@ApplicationScoped
public class LedgerActorStateContributor implements ActorStateContributor {
    @Inject TrustGateService trustGateService;

    @Override public String sourceName() { return "ledger"; }

    @Override
    public void contribute(String actorId, ActorStateAccumulator acc) {
        // findScore returns Optional<ActorTrustScore>; trustScore is primitive double on entity
        // null passed to accumulator when actor has no score yet — not 0.0
        acc.trustScore(trustGateService.findScore(actorId)
                .map(s -> s.trustScore)
                .orElse(null));
        trustGateService.allCapabilityScores(actorId)
                .forEach(acc::capabilityScore);
    }
}
```

**`WorkActorStateContributor`**
```java
@ApplicationScoped
public class WorkActorStateContributor implements ActorStateContributor {
    @Inject WorkItemStore workItemStore;

    @Override public String sourceName() { return "work"; }

    @Override
    public void contribute(String actorId, ActorStateAccumulator acc) {
        // scan(WorkItemQuery.inbox(actorId, null, null)) — the two nulls are:
        //   candidateGroups: null = no group filter
        //   candidateUserId: null = no candidateUsers field filter
        // PENDING excluded: actor is a candidate but hasn't claimed — not "active work"
        // SUSPENDED included: actor is obligated to complete (paused, not released)
        workItemStore.scan(WorkItemQuery.inbox(actorId, null, null)
                .toBuilder()
                .statusIn(List.of(ASSIGNED, IN_PROGRESS, SUSPENDED))
                .build())
            .forEach(wi -> acc.workItem(
                    wi.id, wi.title,
                    wi.status != null ? wi.status.name() : null,
                    wi.category,
                    WorkItemCallerRef.parseCaseId(wi.callerRef)));
    }
}
```

**`QhorusActorStateContributor`**
```java
@ApplicationScoped
public class QhorusActorStateContributor implements ActorStateContributor {
    @Inject CommitmentStore commitmentStore;
    @Inject ChannelStore channelStore;

    @Override public String sourceName() { return "qhorus"; }

    @Override
    public void contribute(String actorId, ActorStateAccumulator acc) {
        List<Commitment> open = commitmentStore.findOpenByObligor(actorId);
        // Batch channel lookup — one IN(?) query, not N queries
        Set<UUID> channelIds = open.stream().map(c -> c.channelId).collect(toSet());
        Map<UUID, String> channelNames = channelStore.findByIds(channelIds).stream()
                .collect(toMap(ch -> ch.id, ch -> ch.name));
        open.forEach(c -> acc.commitment(
                c.id, c.channelId,
                CaseChannel.parseCaseId(channelNames.get(c.channelId)),
                c.state.name(), c.expiresAt));
    }
}
```

**`EngineActorStateContributor`**
```java
@ApplicationScoped
public class EngineActorStateContributor implements ActorStateContributor {
    @Inject WorkerExecutionManager executionManager;

    @Override public String sourceName() { return "engine"; }

    @Override
    public void contribute(String actorId, ActorStateAccumulator acc) {
        // actorId == workerId — same string, different layer naming convention
        // Quartz in-memory scan; best-effort snapshot (TOCTOU — documented in Javadoc)
        executionManager.getActiveCaseIds(actorId).forEach(acc::engineActiveCaseId);
    }
}
```

### `ActorStateAggregator`

```java
@ApplicationScoped
class ActorStateAggregator {
    @Inject @Any Instance<ActorStateContributor> contributors;

    ActorStateResponse forActor(String actorId) {
        var accumulator = new ActorStateAccumulatorImpl(actorId);
        // Parallel execution — blocking path uses virtual threads per contributor
        // Total latency = max(source latencies), not sum
        List<CompletableFuture<Void>> futures = contributors.stream()
                .map(c -> CompletableFuture.runAsync(() -> {
                    try {
                        c.contribute(actorId, accumulator);
                        accumulator.markSucceeded(c.sourceName());
                    } catch (Exception e) {
                        LOG.warnf("source %s failed for actor %s: %s",
                                c.sourceName(), actorId, e.getMessage());
                        accumulator.markFailed(c.sourceName(), e.getMessage());
                    }
                }, VIRTUAL_THREAD_EXECUTOR))
                .toList();
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        return accumulator.build();
    }
}
```

`VIRTUAL_THREAD_EXECUTOR` = `Executors.newVirtualThreadPerTaskExecutor()` — avoids dedicated pool config, works with blocking JDBC calls.

### `ActorStateResource`

`GET /actors/{actorId}/state` — delegates to `ActorStateAggregator`.

**Authorization:** inherits application-level security. No platform-level role restriction — applications apply `@Authenticated`, `@RolesAllowed`, or other constraints via their existing security configuration. The resource itself carries no `@RolesAllowed` annotation.

**tenancyId:** not an explicit parameter. Each store handles tenancy implicitly:
- `WorkItemStore.scan()`: Panache/security context applies tenant scoping per `no-conditional-tenancy-filtering` protocol
- `CommitmentStore`: qhorus is single-tenant per deployment
- `TrustGateService`: scores are by actorId, not tenant-scoped in query
- `WorkerExecutionManager.getActiveCaseIds()`: Quartz in-memory, not tenant-scoped

### `ActorStateAccumulatorImpl`

Implements `ActorStateAccumulator`. Builds `ActorStateResponse`. Thread-safe (contributors run concurrently) — uses `ConcurrentLinkedQueue` for collections, `AtomicReference` for scalar fields.

### Response shape

```json
{
  "actorId": "aml-sar-agent-v2",
  "retrievedAt": "2026-06-01T12:34:56.789Z",
  "trustScore": 0.82,
  "capabilityScores": {
    "sar-drafting": 0.79,
    "osint-screening": 0.85
  },
  "activeWorkItems": [
    {
      "id": "...",
      "title": "...",
      "status": "IN_PROGRESS",
      "category": "...",
      "caseId": "3fa85f64-..."
    }
  ],
  "openCommitments": [
    {
      "commitmentId": "...",
      "channelId": "...",
      "caseId": "3fa85f64-...",
      "state": "OPEN",
      "expiresAt": "..."
    }
  ],
  "engineActiveCaseIds": ["3fa85f64-...", "2cb96a12-..."],
  "sources": ["ledger", "work", "qhorus", "engine"],
  "sourceWarnings": {
    "qhorus": "Connection refused after 5000ms"
  }
}
```

**Null contract:** `trustScore: null` when actor has no computed score yet (not `0.0`, which would mean zero trust). `capabilityScores: {}` when no capability scores exist. `caseId: null` on workItems/commitments when the item was not created by the engine or the channel doesn't follow case naming. `sourceWarnings` is absent (not an empty object) when all sources succeeded.

**`engineActiveCaseIds` naming:** explicitly scoped to Quartz in-flight jobs. Does not claim to be an exhaustive view of all cases the actor is working on — an actor can have active WorkItems and open Commitments for a case without an active Quartz job for it (the engine may have already dispatched and be waiting on the Qhorus reply). Callers must treat the four sources as independent slices, not as a unified case list.

---

## Reactive path

Contributors are always blocking (`void contribute()`). The reactive aggregator wraps each contributor on the blocking executor — no reactive contributor interface needed, no parity problem, no doubled SPI surface:

```java
@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")
@ApplicationScoped
class ReactiveActorStateAggregator {
    @Inject @Any Instance<ActorStateContributor> contributors;

    Uni<ActorStateResponse> forActor(String actorId) {
        var accumulator = new ActorStateAccumulatorImpl(actorId);
        List<Uni<Void>> unis = contributors.stream()
                .map(c -> Uni.createFrom().voidItem()
                        .invoke(() -> {
                            try { c.contribute(actorId, accumulator); accumulator.markSucceeded(c.sourceName()); }
                            catch (Exception e) { accumulator.markFailed(c.sourceName(), e.getMessage()); }
                        })
                        .runSubscriptionOn(Infrastructure.getDefaultBlockingExecutor()))
                .toList();
        return Uni.combine().all().unis(unis).discardItems()
                .map(v -> accumulator.build());
    }
}
```

`ReactiveActorStateResource` (gated by same `@IfBuildProperty`) injects `ReactiveActorStateAggregator` and returns `Uni<ActorStateResponse>`.

---

## Testing

### `ActorStateServiceTest` — plain JUnit, no CDI

Tests `ActorStateAggregator` with contributor stubs (lambdas implementing `ActorStateContributor`). Tests:
- All sources healthy → complete response, all 4 in `sources`, `sourceWarnings` absent
- One contributor throws → that source in `sourceWarnings`, excluded from `sources`, others intact
- Ledger returns no score → `trustScore: null` in response (not `0.0`)
- WorkItem with `callerRef = "3fa85f64-...:pi-001"` → caseId parsed via `WorkItemCallerRef.parseCaseId`
- WorkItem with `callerRef = "not-a-uuid:something"` → `caseId: null`, no throw
- WorkItem with null callerRef → `caseId: null`, no throw
- WorkItem with PENDING status → excluded (status filter in contributor)
- Commitment on `case-{uuid}/work` channel → caseId extracted via `CaseChannel.parseCaseId`
- Commitment on non-case channel → `caseId: null`, no throw
- Unknown actorId → valid empty response, all 4 in `sources`, `sourceWarnings` absent

### `ActorStateResourceTest` — `@QuarkusTest` with `@InjectMock ActorStateAggregator`

- 200 with correct JSON field names including `retrievedAt`, `engineActiveCaseIds`, `sourceWarnings`
- `sources` array present
- `sourceWarnings` present only when a source failed

### `ReactiveActorStateServiceTest` — plain JUnit, no CDI

Mirrors `ActorStateServiceTest` for the reactive aggregator. Same contributor stubs, all methods verify `Uni` results via `.await().indefinitely()`.

**Build-time parity:** add `casehub-engine-actor-state` to the scope of `BlockingReactiveParityTest` (ArchUnit) or a local equivalent asserting `ReactiveActorStateAggregator` has a `Uni<ActorStateResponse>` method for every `ActorStateResponse` method on `ActorStateAggregator`.

### `CommitmentStoreContractTest` additions

New abstract methods, run by both `InMemoryCommitmentStoreTest` and `JpaCommitmentStoreTest`:
- `findOpenByObligor_returnsAcrossMultipleChannels`
- `findOpenByObligor_excludesTerminalStates_crossChannel`
- `findOpenByObligor_nullObligorInStore_notReturnedForActualActor`
- `findOpenByObligor_emptyStore_returnsEmpty`

### `QuartzWorkerExecutionManagerTest` additions

Mock `Scheduler` via Mockito:
- Job with `workerId="agent-x"`, `caseHubInstanceUuid="uuid1"` → `getActiveCaseIds("agent-x")` returns `[uuid1]`
- Two jobs for same worker on different cases → both UUIDs returned
- `getActiveCaseIds("other-agent")` → empty list
- `getJobGroupNames()` throws → empty list (no exception propagated)

### `CaseChannel.parseCaseId` unit tests (engine-api)

- `"case-3fa85f64-.../work"` → correct UUID
- `"case-3fa85f64-..."` (no purpose segment) → correct UUID
- `"not-a-case-channel"` → null
- `"case-not-a-uuid/work"` → null
- null → null

### `WorkItemCallerRef.parseCaseId` unit tests (casehub-work-api)

- `"3fa85f64-...:pi-001"` → correct UUID
- `"not-a-uuid:planItem"` → null
- no colon → null
- null → null

---

## Files requiring updates per repo

### `casehub-platform` (platform repo)

New files:
- `platform-api/src/main/java/.../ActorStateContributor.java`
- `platform-api/src/main/java/.../ActorStateAccumulator.java`

Modified:
- `platform-api/pom.xml` — no new deps (stdlib only)
- `CLAUDE.md` — document new interfaces in platform-api section
- ADR required documenting placement rationale (stdlib types only, needed by ≥4 repos)

### `casehub-engine` (engine repo)

New files:
- `engine-actor-state/pom.xml`
- `engine-actor-state/src/main/java/.../LedgerActorStateContributor.java`
- `engine-actor-state/src/main/java/.../WorkActorStateContributor.java`
- `engine-actor-state/src/main/java/.../QhorusActorStateContributor.java`
- `engine-actor-state/src/main/java/.../EngineActorStateContributor.java`
- `engine-actor-state/src/main/java/.../ActorStateAggregator.java`
- `engine-actor-state/src/main/java/.../ReactiveActorStateAggregator.java`
- `engine-actor-state/src/main/java/.../ActorStateAccumulatorImpl.java`
- `engine-actor-state/src/main/java/.../ActorStateResource.java`
- `engine-actor-state/src/main/java/.../ReactiveActorStateResource.java`
- `engine-actor-state/src/main/java/.../ActorStateResponse.java` (+ `WorkItemSummary`, `CommitmentSummary`)
- `engine-actor-state/src/test/java/.../ActorStateServiceTest.java`
- `engine-actor-state/src/test/java/.../ActorStateResourceTest.java`
- `engine-actor-state/src/test/java/.../ReactiveActorStateServiceTest.java`
- `api/src/main/java/.../CaseChannel.java` — add `parseCaseId(String)` static

Modified:
- `common/src/main/java/.../WorkerExecutionManager.java` — add `getActiveCaseIds` default
- `scheduler-quartz/src/main/java/.../QuartzWorkerExecutionManager.java` — implement `getActiveCaseIds`
- `scheduler-quartz/src/test/java/.../QuartzWorkerExecutionManagerTest.java` — add cases
- Engine root `pom.xml` — add `engine-actor-state` module
- `CLAUDE.md` — document new module and `getActiveCaseIds` in SPI section
- `docs/repos/casehub-engine.md` (in parent) — add actor-state module, SPI additions

### `casehub-qhorus` (qhorus repo)

Modified:
- `runtime/.../store/CommitmentStore.java` — add `findOpenByObligor(obligor)` default with table-scan warning
- `runtime/.../store/jpa/JpaCommitmentStore.java` — override with indexed query
- `runtime/.../store/ReactiveCommitmentStore.java` — add `findOpenByObligor(obligor)` returning `Uni<List<Commitment>>`
- `runtime/.../store/jpa/ReactiveJpaCommitmentStore.java` — reactive implementation
- `runtime/.../store/ChannelStore.java` — add `findByIds(Collection<UUID>)` default
- `runtime/.../store/jpa/JpaChannelStore.java` — override with IN(?) query
- `testing/.../InMemoryCommitmentStore.java` — implement `findOpenByObligor(obligor)`
- `testing/.../InMemoryReactiveCommitmentStore.java` — implement reactive variant
- `testing/.../InMemoryChannelStore.java` (if exists) — implement `findByIds`
- `testing/.../contract/CommitmentStoreContractTest.java` — add 4 new abstract test cases
- `testing/.../InMemoryCommitmentStoreTest.java` — inherits new tests via contract base
- `runtime/.../store/JpaCommitmentStoreTest.java` — inherits new tests; add `@TestTransaction`
- `CLAUDE.md` — document new query methods
- `docs/repos/casehub-qhorus.md` (in parent) — update CommitmentStore and ChannelStore sections

### `casehub-ledger` (ledger repo)

Modified:
- `runtime/.../service/TrustGateService.java` — add `allCapabilityScores(actorId)` method
- `CLAUDE.md` — document new method
- `docs/repos/casehub-ledger.md` (in parent) — update TrustGateService section

### `casehub-work` (work repo)

New file:
- `api/src/main/java/.../WorkItemCallerRef.java` — `parseCaseId(String)` utility

Modified:
- `CLAUDE.md` — document new utility class
- `docs/repos/casehub-work.md` (in parent) — document callerRef format explicitly

### Applications (AML, devtown)

Each adds `casehub-engine-actor-state` to their `app/pom.xml`. No other changes.

---

## Identity invariant (documented, not enforced)

The `actorId` path parameter must be the same string across all four systems:
- Engine YAML `worker.name` (e.g. `"sar-drafting-agent-v1"`) — used as `workerId` in Quartz
- Qhorus `Commitment.obligor` (set when COMMAND is dispatched to the agent)
- Ledger `ActorTrustScore.actorId`
- Work `WorkItem.assigneeId`

Silent empty results from a source indicate identity misalignment, not a system error. Cannot be validated at runtime.

---

## Out of scope (deferred)

- JOIN projection `findOpenByObligorWithChannelName` — `findByIds` batch is sufficient for now
- `GET /actors/{actorId}/state` paging or filtering
- Caching / TTL (best-effort snapshot by design)
- Moving contributor implementations into their home modules (work, qhorus, ledger) — migration path preserved via platform-api SPI
