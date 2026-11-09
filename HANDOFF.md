# HANDOFF — casehub-examples (slot 196)

## Last Session

Implemented Batch 1 (Foundation) of #520 cognitive state dump. Blog "Where the Why Lives" written. Dependency cycle issues resolved.

### What was built

**1. Blog: "Where the Why Lives"**

Article/essay arguing cognitive pipeline observability is a requirement for temporal agents, not a luxury. 2 SVGs. At `docs/blog/2026-11-09-mdp02-where-the-why-lives.md`.

**2. Batch 1 — Foundation types, SPI, store contract (all passing)**

| Artifact | Module | Package |
|---|---|---|
| `PipelineTrace` record | cognition-api | `io.casehub.neocortex.cognition.trace` |
| `CognitionStateProvider` SPI | cognition-api | `io.casehub.neocortex.cognition.trace` |
| `CapsSettlingSnapshot` (nested in PipelineTrace) | cognition-api | `io.casehub.neocortex.cognition.trace` |
| `CognitionSnapshot` + `@Nullable PipelineTrace trace` | cognition | `io.casehub.neocortex.cognition.core` |
| `PipelineSnapshotStore` interface | cognitive-observability-core | `io.casehub.neocortex.cognitive.observability` |
| `PipelineSnapshot` (nested in PipelineSnapshotStore) | cognitive-observability-core | `io.casehub.neocortex.cognitive.observability` |
| `InMemoryPipelineSnapshotStore` | cognitive-observability-testing | `io.casehub.neocortex.cognitive.observability.testing` |
| `PipelineSnapshotStoreContractTest` (9 tests) | cognitive-observability-testing | `io.casehub.neocortex.cognitive.observability.testing` |
| `InMemoryPipelineSnapshotStoreTest` (runner) | cognitive-observability-testing | `io.casehub.neocortex.cognitive.observability.testing` |

**3. Dependency cycle resolution**

The original design placed PipelineTrace and CognitionStateProvider in `cognitive-observability-api`. This creates cycles:
- `cognitive-observability-api → caps-api → cognitive-index → cognitive-observability-api`
- `cognitive-observability-api → cognition-api → cognitive-index → cognitive-observability-api`

Root cause: `cognitive-index` depends on `cognitive-observability-api`, so nothing `cognitive-index` depends on can be added to `-api`.

**Resolution:** Moved PipelineTrace and CognitionStateProvider to `cognition-api` (package `io.casehub.neocortex.cognition.trace`). Added `cognition-api` as a dependency of `cognitive-observability-core` (no cycle — -core is a leaf). Replaced `SettlingResult` reference in PipelineTrace with inline `CapsSettlingSnapshot` record (avoids caps-api dependency entirely).

**4. Pre-existing build fix**

Added `casehub-platform-api` dependency to `cognitive-observability-core/pom.xml` — fixes pre-existing compile error where `McpExclude`/`McpOnly` annotations weren't resolved. Required installing `platform-api` from the slot's platform directory first.

### POM changes summary

| File | Change |
|---|---|
| `cognitive-observability-api/pom.xml` | Removed cognition-api, caps-api (cycle) |
| `cognitive-observability-core/pom.xml` | Added cognition-api, platform-api |
| `cognition/pom.xml` | Removed cognitive-observability-api (no longer needed) |

## Immediate Next Step

**Batch 2: Storage + Capture** — 2 tasks remaining in this batch:

1. **Task 3: SqlitePipelineSnapshotStore** — follow SqliteSnapshotStore pattern, run contract tests
2. **Task 4: CAPS caching + StateProvider impl + tick capture** — cache SettlingResult in BehavioralSynthesisPhase, implement CognitionStateProvider on CognitionCore

Then Batch 3: API endpoint + integration test (Tasks 5-6).

Run `work continue` → read plan at `plans/2026-11-09-cognitive-state-dump.md` → invoke `executing-plans`.

## What to Watch

1. **SqliteSnapshotStore pattern.** Read the existing `SqliteSnapshotStore.java` for constructor pattern (DataSource vs path), JSON serialization, and query patterns. The contract test already works with InMemory — SQLite should pass the same 9 tests.

2. **PipelineTrace serialization.** The SQLite store needs to serialize PipelineTrace to JSON. Check how `SqliteSnapshotStore` serializes `GraphSnapshot` — follow the same approach (likely Jackson ObjectMapper).

3. **BehavioralSynthesisPhase access from CognitionCore.** CognitionCore needs to query `lastSettlingResult(agentId)` from BehavioralSynthesisPhase. Check if CognitionCore already has a reference or needs one wired via `CognitivePipelineBuilder`.

4. **CognitionCore's current*() method names.** The plan uses placeholder names. Use `ide_file_structure` to find actual field and method names on CognitionCore, DriveOrchestrator, AppraisalTickParticipant, etc.

5. **Pre-existing compile errors.** `cognition/pom.xml` has a pre-existing error in `SearchBudgetTickParticipant.java:245` (missing `SearchMotivation` symbol). Unrelated to #520 changes. May need the full reactor installed first: `mvn install -Dmaven.test.skip=true` from the neocortex root.

## Queue Status

- [x] Foundation: PipelineTrace + SPI + extend CognitionSnapshot
- [x] Foundation: PipelineSnapshotStore + InMemory + contract test
- [ ] Storage: SqlitePipelineSnapshotStore
- [ ] Capture: CAPS caching + StateProvider impl + tick capture
- [ ] API: pipelineDump endpoint + CDI wiring
- [ ] Integration: CognitionStateProvider integration test

## References

- Spec: `specs/issue-851-dynamic-regulatory-capacity/2026-11-09-cognitive-state-dump-design.md`
- Plan: `plans/2026-11-09-cognitive-state-dump.md`
- Blog: `examples/docs/blog/2026-11-09-mdp02-where-the-why-lives.md`
- Contract tests: `InMemoryPipelineSnapshotStoreTest` (9/9 pass)
- Neocortex commits: `f15c846c` (Task 1: types+SPI), `c1e2efa1` (Task 2: store+contracts)
