# Actor State View — Design Spec

**Issue:** casehubio/parent#56
**Date:** 2026-06-01
**Status:** Approved for implementation

---

## Problem

Four platform modules each hold a complementary slice of an actor's current state:

| Source | Data | Query |
|--------|------|-------|
| `casehub-ledger` | Trust scores (global + per-capability) | `TrustGateService.findScore(actorId)` |
| `casehub-work` | Active human-task WorkItems | `WorkItemStore.scan(WorkItemQuery.inbox(actorId, ...))` |
| `casehub-qhorus` | Open agent Commitments (what the actor owes) | `CommitmentStore.findOpenByObligor(actorId, channelId)` — requires channelId today |
| `casehub-engine` | Cases where the engine currently has this worker executing | `WorkerExecutionManager.getActiveWorkCount(workerId)` — count only, no UUIDs |

No endpoint joins these. Answering "what is actor X currently doing?" requires four separate calls and manual join logic in the caller. `ActorState`: zero results across the entire codebase.

---

## Solution: `casehub-engine-actor-state` module

A new adapter module in the engine repo, following the exact pattern of `casehub-engine-work-adapter`. All four sources are queried via in-process CDI — no REST clients, no shared transaction.

### Module identity

```
casehub-engine-actor-state
  ├── casehub-engine-common    EventLogRepository, WorkerExecutionManager SPI
  ├── casehub-engine           quarkus-rest (REST endpoint hosting)
  ├── casehub-ledger           TrustGateService, ActorTrustScoreRepository
  ├── casehub-work             WorkItemStore, WorkItemQuery
  └── casehub-qhorus           CommitmentStore, ChannelStore
```

**Activation:** applications add `casehub-engine-actor-state` to their pom. AML and devtown already import all required CDI beans — this module just wires them.

---

## New SPI additions

Two small additions to existing modules. Both follow PP-20260601-81b9e5 (SPI evolution via default methods).

### 1. `CommitmentStore.findOpenByObligor(String obligor)` — qhorus

Cross-channel obligor query. The existing `findOpenByObligor(obligor, channelId)` requires a channelId, preventing cross-channel aggregation.

```java
// CommitmentStore interface — new default
default List<Commitment> findOpenByObligor(String obligor) {
    return findAllOpen().stream()
            .filter(c -> obligor != null && obligor.equals(c.obligor))
            .toList();
}
```

`JpaCommitmentStore` overrides with an indexed query:

```java
@Override
public List<Commitment> findOpenByObligor(String obligor) {
    return repo.list(
            "obligor = ?1 AND state NOT IN ?2",
            obligor, terminalStates());
}
```

Also add to: `ReactiveCommitmentStore` interface, `ReactiveJpaCommitmentStore`, `InMemoryCommitmentStore` (casehub-qhorus-testing), `InMemoryReactiveCommitmentStore`.

Contract test additions to `CommitmentStoreContractTest`:
- `findOpenByObligor_returnsAcrossMultipleChannels`
- `findOpenByObligor_excludesTerminalStates_crossChannel`
- `findOpenByObligor_nullObligorInStore_notReturnedForActualActor`
- `findOpenByObligor_emptyStore_returnsEmpty`

### 2. `WorkerExecutionManager.getActiveCaseIds(String workerId)` — engine-common SPI

Returns case UUIDs for all Quartz jobs currently executing for this worker. The Quartz job data map already stores `caseHubInstanceUuid` per job (set by `QuartzWorkerSchedulerService.scheduleJob`).

```java
// WorkerExecutionManager SPI — new default
default List<UUID> getActiveCaseIds(String workerId) {
    return List.of();
}
```

`QuartzWorkerExecutionManager` overrides: same iteration as `getActiveWorkCount`, collects `caseHubInstanceUuid` from job data where `workerId` matches.

`NoOpWorkerExecutionManager` and all other no-ops inherit the default — no changes required.

---

## Components

### `ActorStateService` — `@ApplicationScoped`

Injected: `TrustGateService`, `ActorTrustScoreRepository`, `WorkItemStore`, `CommitmentStore`, `ChannelStore`, `WorkerExecutionManager`.

Method: `ActorState forActor(String actorId, String tenancyId)`.

Each source is wrapped in an independent try-catch per PP-20260601-c60b28 (cross-backend aggregation). Failure of one source does not affect others. `tenancyId` is passed where the store requires it (WorkItemStore context; engine EventLog is Quartz-in-memory and not tenant-scoped).

### `ActorStateResource` — JAX-RS `GET /actors/{actorId}/state`

Thin delegate to `ActorStateService`. Returns `ActorStateResponse` as JSON.

### Response shape

```json
{
  "actorId": "aml-sar-agent-v2",
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
  "activeCases": ["3fa85f64-...", "2cb96a12-..."],
  "sources": ["ledger", "work", "qhorus", "engine"]
}
```

`sources` lists backends that successfully responded. A backend that throws is omitted from `sources` and its field(s) are empty/null. Response is always 200.

---

## Data flow per source

### Ledger (trust)
```
TrustGateService.findScore(actorId)                      → global trustScore
ActorTrustScoreRepository.findByActorIdAndScoreType(
    actorId, CAPABILITY)                                 → capabilityScores map
```
Both synchronous CDI calls. No tenancy filter (trust scores are per-actorId, not per-tenant).

### Work (human tasks)
```
WorkItemStore.scan(
    WorkItemQuery.inbox(actorId, null, null)
        .toBuilder()
        .statusIn(List.of(ASSIGNED, IN_PROGRESS, SUSPENDED))
        .build())                                         → List<WorkItem>
```
Status filter is intentional: PENDING items where the actor is a candidate but hasn't claimed are not "active work". For each WorkItem, parse `caseId` from `callerRef`:
```java
static UUID parseCaseId(String callerRef) {
    if (callerRef == null || !callerRef.contains(":")) return null;
    try { return UUID.fromString(callerRef.split(":")[0]); }
    catch (IllegalArgumentException e) { return null; }
}
```
Returns null for externally-created WorkItems — acceptable, not an error.

### Qhorus (commitments)
```
CommitmentStore.findOpenByObligor(actorId)               → List<Commitment>
```
For each Commitment, derive `caseId` from channel name:
```java
static UUID parseCaseIdFromChannel(String channelName) {
    if (channelName == null || !channelName.startsWith(CaseChannel.CASE_CHANNEL_PREFIX))
        return null;
    String rest = channelName.substring(CaseChannel.CASE_CHANNEL_PREFIX.length());
    String uuidStr = rest.contains("/") ? rest.substring(0, rest.indexOf('/')) : rest;
    try { return UUID.fromString(uuidStr); }
    catch (IllegalArgumentException e) { return null; }
}
```
Channel lookup: deduplicate channelIds before calling `ChannelStore.find()` to avoid N+1. Build a `Map<UUID, String>` (channelId → channelName) with one lookup per distinct channelId.

**Improvement opportunity (deferred):** replace with a JOIN projection `findOpenByObligorWithChannelName(actorId)` once query volume justifies it.

### Engine (active Quartz jobs)
```
WorkerExecutionManager.getActiveCaseIds(actorId)         → List<UUID>
```
In-memory Quartz scan. Best-effort snapshot — a job completing between this call and the HTTP response means `activeCases` may transiently list a case whose work just finished. Documented in Javadoc; acceptable for a monitoring endpoint.

---

## Error handling

Per PP-20260601-c60b28:

```java
// Each source in ActorStateService.forActor() follows this pattern:
List<WorkItem> workItems = List.of();
boolean workSucceeded = false;
try {
    workItems = workItemStore.scan(...);
    workSucceeded = true;
} catch (Exception e) {
    LOG.warnf("work source failed for actorId=%s: %s", actorId, e.getMessage());
}
```

`sources` is built from which backends succeeded. Never a 500 — always 200 with whatever was retrieved.

**Unknown actorId:** all sources return empty/null (no trust score computed yet, no WorkItems, no Commitments, no active Quartz jobs). Response: valid empty response with all four sources listed (they all responded with empty, not errors).

**`obligor = null` Commitments:** broadcast COMMAND/QUERY messages have `obligor = null`. The `WHERE obligor = ?1` query never returns null-obligor rows — correct, they are not "owed" by any specific actor.

**DELEGATED Commitments:** terminal state, excluded by `terminalStates()` in JpaCommitmentStore. When a HANDOFF fires, the child Commitment with the new agent as obligor IS returned. Correct.

**callerRef identity gap:** documented at endpoint level — the `actorId` path parameter must be the same string used as `workerId` in engine YAML, `obligor` in qhorus messages, `actorId` in ledger, and `assigneeId` in work. Silent empty results from a source indicate identity misalignment.

---

## Reactive dual-path

Per PP-20260519-39a9a5 Consumer Contract section:

When `casehub.qhorus.reactive.enabled=true`, `CommitmentStore` (blocking) is not the active CDI bean. Following the `A2AResource` / `ReactiveA2AResource` split in qhorus, the resource is also split:

- `ActorStateService` — `@UnlessBuildProperty(name = "casehub.qhorus.reactive.enabled", ...)` — injects `CommitmentStore`
- `ReactiveActorStateService` — `@IfBuildProperty(name = "casehub.qhorus.reactive.enabled", stringValue = "true")` — injects `ReactiveCommitmentStore`, all methods return `Uni<T>`
- `ActorStateResource` — `@UnlessBuildProperty(...)` — injects `ActorStateService`
- `ReactiveActorStateResource` — `@IfBuildProperty(...)` — injects `ReactiveActorStateService`

Both resources are `@Path("/actors/{actorId}/state")` — only one is active at build time.

Parity rule applies: every method on `ActorStateService` must have a `Uni<T>` equivalent on `ReactiveActorStateService`.

---

## Testing

### `ActorStateServiceTest` — plain JUnit, no CDI

All stores supplied as lambdas. Tests:
- All sources healthy → complete response, all fields populated, all 4 in `sources`
- Ledger throws → `sources` excludes "ledger", trustScore null, capabilityScores empty, others intact
- Work throws → `sources` excludes "work", workItems empty, others intact
- Qhorus throws → `sources` excludes "qhorus", commitments empty, others intact
- Engine throws → `sources` excludes "engine", activeCases empty, others intact
- WorkItem with `callerRef = "3fa85f64-...:pi-001"` → caseId parsed
- WorkItem with `callerRef = "not-a-uuid:something"` → caseId null, no throw
- WorkItem with null callerRef → caseId null, no throw
- WorkItem with PENDING status → excluded by status filter
- Commitment on `case-{uuid}/work` channel → caseId extracted
- Commitment on non-case channel name → caseId null, no throw
- Commitment with `obligor = null` → not returned by `findOpenByObligor`
- Unknown actorId → valid empty response, all 4 in `sources`

### `ActorStateResourceTest` — `@QuarkusTest` with `@InjectMock ActorStateService`

- 200 response with correct JSON field names
- `sources` array present in response
- Partial `sources` when one source returns empty

### `CommitmentStoreContractTest` additions (casehub-qhorus-testing)

New abstract methods in `CommitmentStoreContractTest`:
- `findOpenByObligor_returnsAcrossMultipleChannels` — actor has open commitments on two channels; both returned
- `findOpenByObligor_excludesTerminalStates_crossChannel` — terminal commitments not returned
- `findOpenByObligor_nullObligorInStore_notReturnedForActualActor` — null-obligor broadcast not returned
- `findOpenByObligor_emptyStore_returnsEmpty`

`InMemoryCommitmentStoreTest` and `JpaCommitmentStoreTest` run these automatically via the abstract base.

### `QuartzWorkerExecutionManagerTest` additions (engine-scheduler-quartz)

Mock `Scheduler` via Mockito. Test `getActiveCaseIds`:
- Job with `workerId="agent-x"` and `caseHubInstanceUuid="uuid1"` → `getActiveCaseIds("agent-x")` returns `[uuid1]`
- Two jobs for same worker on different cases → both UUIDs returned
- `getActiveCaseIds("other-agent")` with no matching jobs → empty list
- `Scheduler.getJobGroupNames()` throws → returns empty list (no exception propagated)

---

## Files requiring updates per repo

### `casehub-engine` (engine repo)

New files:
- `engine-actor-state/pom.xml`
- `engine-actor-state/src/main/java/.../ActorStateService.java` (`@UnlessBuildProperty`)
- `engine-actor-state/src/main/java/.../ReactiveActorStateService.java` (`@IfBuildProperty`)
- `engine-actor-state/src/main/java/.../ActorStateResource.java` (`@UnlessBuildProperty`)
- `engine-actor-state/src/main/java/.../ReactiveActorStateResource.java` (`@IfBuildProperty`)
- `engine-actor-state/src/main/java/.../ActorStateResponse.java` (+ sub-DTOs: `WorkItemSummary`, `CommitmentSummary`)
- `engine-actor-state/src/test/java/.../ActorStateServiceTest.java`
- `engine-actor-state/src/test/java/.../ActorStateResourceTest.java`

Modified files:
- `common/src/main/java/.../WorkerExecutionManager.java` — add `getActiveCaseIds` default
- `scheduler-quartz/src/main/java/.../QuartzWorkerExecutionManager.java` — implement `getActiveCaseIds`
- `scheduler-quartz/src/test/java/.../QuartzWorkerExecutionManagerTest.java` — add contract cases
- Engine root `pom.xml` — add `engine-actor-state` module
- `CLAUDE.md` — document new module in module listing
- `docs/repos/casehub-engine.md` (in parent) — add actor-state module to repo deep-dive

### `casehub-qhorus` (qhorus repo)

Modified files:
- `runtime/src/main/java/.../store/CommitmentStore.java` — add `findOpenByObligor(obligor)` default
- `runtime/src/main/java/.../store/jpa/JpaCommitmentStore.java` — override with indexed query
- `runtime/src/main/java/.../store/ReactiveCommitmentStore.java` — add `findOpenByObligor(obligor)` returning `Uni<List<Commitment>>`
- `runtime/src/main/java/.../store/jpa/ReactiveJpaCommitmentStore.java` — reactive implementation
- `testing/src/main/java/.../InMemoryCommitmentStore.java` — implement new method
- `testing/src/main/java/.../InMemoryReactiveCommitmentStore.java` — implement new method
- `testing/src/test/java/.../contract/CommitmentStoreContractTest.java` — add 4 new test cases
- `testing/src/test/java/.../InMemoryCommitmentStoreTest.java` — inherits new tests via contract
- `runtime/src/test/java/.../store/JpaCommitmentStoreTest.java` — inherits new tests; may need `@TestTransaction` wrappers
- `CLAUDE.md` — document new query method
- `docs/repos/casehub-qhorus.md` (in parent) — update CommitmentStore section

### `casehub-ledger` (ledger repo)

No code changes. The `TrustGateService` and `ActorTrustScoreRepository` are queried as-is. No file updates required.

### Applications (AML, devtown)

Each application adds `casehub-engine-actor-state` to their `app/pom.xml`. No other changes. Their existing imports of engine, work, qhorus, and ledger already provide all required CDI beans.

---

## Identity invariant (documented, not enforced)

The `actorId` parameter in `GET /actors/{actorId}/state` must be consistent across all four systems:
- Engine YAML `worker.name` (e.g. `"sar-drafting-agent-v1"`)
- Qhorus `Commitment.obligor` (set when the COMMAND is sent to the agent)
- Ledger `ActorTrustScore.actorId`
- Work `WorkItem.assigneeId`

Silent empty results from a source indicate identity misalignment, not a system error. The endpoint cannot validate this at runtime — it is a precondition on the caller.

---

## Out of scope (deferred)

- JOIN projection for `findOpenByObligorWithChannelName` (N+1 improvement) — address if query volume warrants
- `GET /actors/{actorId}/state` paging or filtering
- Caching / TTL (endpoint is best-effort snapshot by design)
- `ActorStateContributor` SPI (issue's long-term vision) — valid future direction once actor state becomes a platform-wide concern
