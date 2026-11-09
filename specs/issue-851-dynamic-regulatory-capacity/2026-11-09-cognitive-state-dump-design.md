# Cognitive State Dump Endpoint — Design Spec

**Issue:** casehubio/neocortex#520
**Epic:** #513 (cognitive pipeline completion — diagnostic tooling section)
**Date:** 2026-11-09
**Branch:** issue-851-dynamic-regulatory-capacity

## Problem

After epic #513 wired the full drive adaptation loop (drive → CAPS → appraisal → salience → rendering), there is no way to observe the pipeline state at each stage. The existing cognitive observability module (`CognitionApi`) observes the **knowledge graph** (MindMap nodes, edges, mutations, confidence) but has zero visibility into the live cognitive pipeline state — mood, drives, appraisal, CAPS settling, regulation.

Tuning and postmortems require tracing what happened at each pipeline stage for a given agent/tenant/tick. The eval framework captures behavioral transcripts (what the agent did), but not the internal reasoning chain (why the pipeline produced that behavior).

## Solution

Three new components that extend the existing cognitive observability infrastructure:

1. **CognitionStateProvider SPI** — interface in `cognitive-observability-api` that exposes live pipeline state. CognitionCore implements it.
2. **PipelineTrace record** — sub-record of CognitionSnapshot capturing the pipeline stages currently missing.
3. **PipelineSnapshotStore** — time-series store for per-tick snapshots, enabling historical replay and before/after comparison.

## Architecture

### Dependency Direction

The SPI pattern preserves correct layer ordering:

```
cognitive-observability-api (L23)
  ├── defines: CognitionStateProvider (SPI interface)
  ├── defines: PipelineTrace (record)
  └── defines: PipelineSnapshotStore (interface)

cognitive-observability-core (L23)
  └── CognitionService implements @PlatformQuery
      └── depends on: CognitionStateProvider (SPI)
      └── depends on: PipelineSnapshotStore

cognition (L25)
  └── CognitionCore implements CognitionStateProvider
      └── depends on: cognitive-observability-api (downward: L25 → L23) ✓

cognitive-observability-sqlite (L23)
  └── SqlitePipelineSnapshotStore implements PipelineSnapshotStore

cognitive-observability-testing (L23)
  └── InMemoryPipelineSnapshotStore implements PipelineSnapshotStore
```

No upward dependency. `CognitionService` depends on the SPI interface, not on CognitionCore directly.

### Data Flow

```
CognitionCore.tick() completes (after TERMINAL phase)
  → assembles PipelineTrace from current*() methods on each participant
  → captures CAPS settling data from per-agent cache
  → renders prompt sections via promptSections()
  → constructs CognitionSnapshot with existing fields + PipelineTrace
  → stores to PipelineSnapshotStore (optional — silently skipped if not wired)

Query path (live):
  CognitionService.pipelineDump(tenantId, agentId)
    → CognitionStateProvider.currentSnapshot(agentId, tenantId)

Query path (historical):
  CognitionService.pipelineDump(tenantId, agentId, tick=50)
    → PipelineSnapshotStore.findByTick(tenantId, agentId, 50)

Query path (range):
  CognitionService.pipelineDump(tenantId, agentId, fromTick=50, toTick=80)
    → PipelineSnapshotStore.find(tenantId, agentId, 50, 80)
```

## Components

### CognitionStateProvider SPI

```java
// In cognitive-observability-api
public interface CognitionStateProvider {
    Optional<CognitionSnapshot> currentSnapshot(String agentId, String tenantId);
    Optional<PipelineTrace> currentTrace(String agentId, String tenantId);
    List<String> activeAgents(String tenantId);
}
```

- `currentSnapshot()` — full state including PipelineTrace for the most recent tick
- `currentTrace()` — lightweight: just the pipeline trace fields, no mood/drives/models
- `activeAgents()` — discovery: which agents have live state for a tenant

CognitionCore implements this by querying its existing `current*()` methods on each orchestrator and participant.

### PipelineTrace Record

```java
// In cognitive-observability-api (or cognitive-observability-core)
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
) {
    public PipelineTraceDelta diffFrom(@Nullable PipelineTrace previous) { ... }
}
```

CognitionSnapshot gains one field: `@Nullable PipelineTrace trace`. Existing callers unaffected.

#### Trace Chain Mapping

| Issue trace stage | Field |
|---|---|
| 1. Seeded drives | CognitionSnapshot.drives (existing) |
| 2. Adapted intensities | CognitionSnapshot.drives (existing) |
| 3. CAPS input activations | trace.capsInputActivations |
| 4. CAPS settling result | trace.capsSettlingResult |
| 5. Behavioral attractors | trace.capsSettlingResult.attractors() |
| 6. Concern salience | trace.concernSalience |
| 7. Activated drive categories | trace.activatedDriveCategories |
| 8. Rendered prompt sections | trace.promptSections |
| 9. Appraisal result | trace.appraisalResult |

### PipelineSnapshotStore

```java
// In cognitive-observability-core
public interface PipelineSnapshotStore {
    void store(CognitionSnapshot snapshot);
    List<CognitionSnapshot> find(String tenantId, String agentId, int fromTick, int toTick);
    Optional<CognitionSnapshot> findByTick(String tenantId, String agentId, int tick);
    List<CognitionSnapshot> findLatest(String tenantId, String agentId, int limit);
    long count(String tenantId);
    void purge(Instant olderThan);
}
```

No keyframe/mutation model — each snapshot is a complete point-in-time record.

#### SQLite Schema

```sql
CREATE TABLE pipeline_snapshot (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tenant_id TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    tick_number INTEGER NOT NULL,
    captured_at TEXT NOT NULL,
    snapshot_json TEXT NOT NULL,
    UNIQUE(tenant_id, agent_id, tick_number)
);
CREATE INDEX idx_pipeline_tenant_agent_tick
    ON pipeline_snapshot(tenant_id, agent_id, tick_number);
```

Unique constraint prevents duplicate captures. Index optimized for agent-scoped tick-range queries. Snapshots serialized as JSON.

#### Storage Volume

17 agents x 300 ticks = 5,100 snapshots per scenario run. Each ~2-5 KB serialized. ~10-25 MB per run. Acceptable for SQLite. `purge(Instant)` handles retention.

### API Endpoint

```java
// On CognitionApi (@McpDomain("neocortex/cognition"))
@PlatformQuery(summary = "Pipeline state dump — live or historical")
CognitivePipelineDump pipelineDump(
    String tenantId,
    String agentId,
    @Nullable Integer tick,
    @Nullable Integer fromTick,
    @Nullable Integer toTick
);
```

Return type wraps either a single snapshot or a list, plus metadata (agent ID, tenant, query type, snapshot count).

### Capture Mechanism

At the end of `CognitionCore.tick()`, after the TERMINAL phase:

1. Assemble PipelineTrace from current*() methods on each participant
2. Capture CAPS data from per-agent cache (stored when settle() was called during the tick)
3. Render prompt sections via promptSections()
4. Construct CognitionSnapshot with existing fields + PipelineTrace
5. Store to PipelineSnapshotStore (if present)

The store is optional — not wiring it does not affect tick execution. Live state is always available via the SPI.

CAPS settling data: CognitionCore stores the SettlingResult + input activations in a per-agent `ConcurrentHashMap` at the call site where `CapsEngine.settle()` is invoked. CapsEngine itself remains a pure computation interface.

### CognitionMetrics Integration

`CognitionMetrics` currently wraps `CognitionSnapshot` + `CognitionDelta` + `Map<String, String> promptSectionContent`. With PipelineTrace in the snapshot, `CognitionMetrics.promptSectionContent` can delegate to `snapshot.trace().promptSections()` — removing the redundant field.

## Testing

### Unit Tests (cognitive-observability-core)

- `PipelineTraceTest` — diffFrom() produces correct deltas for each field
- `CognitivePipelineDumpTest` — query mode dispatch (live vs historical vs range)

### Contract Tests (cognitive-observability-testing)

- `PipelineSnapshotStoreContractTest` — store/find/findByTick/findLatest/purge
- Both InMemoryPipelineSnapshotStore and SqlitePipelineSnapshotStore run the same contract

### Integration Test (cognition module)

- `CognitionStateProviderIntegrationTest` — run a CognitionCore tick, query the SPI, verify the snapshot contains populated PipelineTrace with non-null appraisal, CAPS, and prompt sections

## Follow-Up: LLM Pipeline Quality Tests

Not in scope for #520, but the infrastructure enables a new class of eval tests:

**Pipeline coherence eval** — LLM-classified tests that assert on PipelineTrace content rather than behavioral outcomes. Examples:
- Given a shame-inducing observation, does the trace show: high shame in appraisalResult → elevated CAPS activation on suppression nodes → RESPONSE_MODULATION selected → suppression-related prompt sections rendered?
- Given a curiosity trigger, does the trace show: curiosity drive intensified → EXPLORATION attractor activated → curiosity prompt section rendered → explorative goal proposed?
- Cross-tick coherence: does drive adaptation across 10 ticks show a trajectory consistent with the experience history?

These tests validate that the pipeline stages are internally coherent — the "why" matches the "what." They require the pipeline dump infrastructure from #520 and should be filed as a follow-up issue.

## Module Placement Summary

| Artifact | Module | Rationale |
|---|---|---|
| CognitionStateProvider | cognitive-observability-api | SPI contract — L23 |
| PipelineTrace, PipelineTraceDelta | cognitive-observability-api | Record types used by SPI |
| CognitivePipelineDump | cognitive-observability-core | Response type for @PlatformQuery |
| PipelineSnapshotStore | cognitive-observability-core | Store interface |
| pipelineDump() on CognitionApi | cognitive-observability-core | Endpoint definition |
| pipelineDump() impl on CognitionService | cognitive-observability-core | Endpoint implementation |
| SqlitePipelineSnapshotStore | cognitive-observability-sqlite | Persistent storage impl |
| InMemoryPipelineSnapshotStore | cognitive-observability-testing | Test impl |
| PipelineSnapshotStoreContractTest | cognitive-observability-testing | Contract test |
| CognitionStateProvider impl | cognition (CognitionCore) | Provides live state |
| Tick capture logic | cognition (CognitionCore) | Assembles and stores snapshots |
| CognitionStateProviderIntegrationTest | cognition | Pipeline integration test |

## Decisions

D15–D19 in `decisions.md` (this branch). Key decisions:
- D15: PipelineTrace sub-record (not flat extension) — revised after R1-08
- D16: CognitionStateProvider SPI (not direct injection) — revised after R1-04 layer violation
- D17: Live + per-tick snapshot storage (not live-only)
- D18: New PipelineSnapshotStore with SQLite impl (not extend SnapshotStore)
- D19: CognitionCore captures CAPS data, CapsEngine stays pure — revised after R1-09

## References

- CognitionSnapshot.java — existing snapshot record (cognition module)
- CognitionMetrics.java — snapshot + delta + prompt sections wrapper
- CognitionDelta.java — diffFrom for existing snapshot fields
- CognitionApi.java — @McpDomain("neocortex/cognition") interface
- CognitionService.java — @PlatformQuery implementation
- CognitionCore.java — pipeline orchestrator, tick() method
- AppraisalTickParticipant.java — currentResult(), currentConcernSalience()
- CapsEngine.java / CapsSettler.java — settle() pure computation
- SnapshotStore.java — existing graph snapshot pattern (model for PipelineSnapshotStore)
- SqliteSnapshotStore.java — existing SQLite pattern
- InMemorySnapshotStore.java — existing in-memory pattern
- SnapshotStoreContractTest.java — existing contract test pattern
- CognitionBeans.java — CDI wiring for observability module
- Issue #520 — cognitive state dump endpoint
- Epic #513 — cognitive pipeline completion, diagnostic tooling section
- Decision review: R1-04 (layer violation), R1-08 (record growth), R1-09 (interface pollution)
