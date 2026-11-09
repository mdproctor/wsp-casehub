# HANDOFF — casehub-examples (slot 196)

## Last Session

Implemented Batch 1 (Foundation) of #520 cognitive state dump. Blog "Where the Why Lives" written. Neocortex research doc updated with Stream 9. Dependency cycle issues resolved.

### What was built

**1. Blog: "Where the Why Lives"**

Article/essay arguing cognitive pipeline observability is a requirement for temporal agents. 2 non-inline SVGs. At `examples/docs/blog/2026-11-09-mdp02-where-the-why-lives.md`. Also fixed SVG paths in "The Test That Broke the Math" blog.

**2. Batch 1 — Foundation types, SPI, store contract (9/9 tests pass)**

| Artifact | Module | Package |
|---|---|---|
| `PipelineTrace` record | cognition-api | `io.casehub.neocortex.cognition.trace` |
| `CognitionStateProvider` SPI | cognition-api | `io.casehub.neocortex.cognition.trace` |
| `CapsSettlingSnapshot` (nested in PipelineTrace) | cognition-api | `io.casehub.neocortex.cognition.trace` |
| `CognitionSnapshot` + `@Nullable PipelineTrace trace` | cognition | `io.casehub.neocortex.cognition.core` |
| `PipelineSnapshotStore` interface + `PipelineSnapshot` record | cognitive-observability-core | `io.casehub.neocortex.cognitive.observability` |
| `InMemoryPipelineSnapshotStore` | cognitive-observability-testing | `io.casehub.neocortex.cognitive.observability.testing` |
| `PipelineSnapshotStoreContractTest` (9 tests) | cognitive-observability-testing | same |

**3. Neocortex research doc updated**

Added Stream 9 (cognitive pipeline observability) to `docs/research/2026-11-05-post-era6-regulation-and-identity.md`. Covers SPI architecture, dependency cycle finding, CAPS consolidation-time discovery, Batch 1 status. Blog coverage table updated with both new entries.

**4. Dependency cycle resolution**

Original design placed PipelineTrace/CognitionStateProvider in `cognitive-observability-api`. Cycles through `cognitive-index`:
- `observability-api → caps-api → cognitive-index → observability-api`
- `observability-api → cognition-api → cognitive-index → observability-api`

**Fix:** Moved both types to `cognition-api` (package `io.casehub.neocortex.cognition.trace`). `SettlingResult` replaced with inline `CapsSettlingSnapshot` record. Added `cognition-api` + `platform-api` to `cognitive-observability-core/pom.xml`.

**5. Pre-existing build fix**

Added `casehub-platform-api` to `cognitive-observability-core/pom.xml` — fixes `McpExclude`/`McpOnly` resolution. Required installing `platform-api` from the slot's platform directory:
```
mvn -f platform/pom.xml install -pl platform-api -am -Dmaven.test.skip=true -s slot-settings.xml
```

### POM changes summary

| File | Change |
|---|---|
| `cognitive-observability-api/pom.xml` | No new deps (cycle prevention) |
| `cognitive-observability-core/pom.xml` | Added `cognition-api`, `platform-api` |
| `cognition/pom.xml` | No changes (cognition-api already a dep) |

## Immediate Next Step

**Execute Batch 2 of the implementation plan.**

Run `work continue` → read plan at `plans/2026-11-09-cognitive-state-dump.md` → invoke `executing-plans`.

Remaining tasks:

| Batch | Task | What |
|-------|------|------|
| 2 | Task 3 | `SqlitePipelineSnapshotStore` — follow `SqliteSnapshotStore` pattern, run 9 contract tests |
| 2 | Task 4 | CAPS caching in `BehavioralSynthesisPhase` + `CognitionStateProvider` impl on `CognitionCore` + tick capture |
| 3 | Task 5 | `CognitivePipelineDump` + `pipelineDump()` `@PlatformQuery` + `CognitionBeans` wiring |
| 3 | Task 6 | `CognitionStateProviderIntegrationTest` |

**Note:** The plan's code examples for Task 4 (CognitionCore field names, accessor methods) are approximate. Use `ide_file_structure` on `CognitionCore.java` to find actual names before writing code.

## What to Watch

1. **`PipelineSnapshotStore` stores `PipelineTrace`, not `CognitionSnapshot`.** Design was adjusted during implementation — the store persists the transient pipeline data only. Durable state (mood, drives, models) already lives in MindMap/SnapshotStore. The `PipelineSnapshot` record wraps tenant/agent/tick/timestamp + `PipelineTrace`.

2. **`SqliteSnapshotStore` pattern.** Read it for: constructor (DataSource), JSON serialization approach, query patterns. The `PipelineTrace` needs Jackson serialization. Check if `AppraisalResult`, `ChannelWeightProfile` etc. are already Jackson-annotated.

3. **`BehavioralSynthesisPhase` at line 168** is the sole `CapsEngine.settle()` call site. Cache `SettlingResult` + `combinedActivations` per agent in `ConcurrentHashMap`s. Add `lastSettlingResult(agentId)` and `lastInputActivations(agentId)` accessors.

4. **CognitionCore → BehavioralSynthesisPhase wiring.** Check if CognitionCore already holds a reference. If not, add it via constructor/builder (`CognitivePipelineBuilder`).

5. **Pre-existing compile error.** `SearchBudgetTickParticipant.java:245` — missing `SearchMotivation` symbol. Unrelated to #520. May need full reactor install from neocortex root.

6. **Platform-api must be installed first.** If the local Maven repo is stale, run: `mvn -f platform/pom.xml install -pl platform-api -am -Dmaven.test.skip=true -s slot-settings.xml`

## Queue Status

- [x] Foundation: PipelineTrace + SPI + extend CognitionSnapshot
- [x] Foundation: PipelineSnapshotStore + InMemory + contract test
- [ ] Storage: SqlitePipelineSnapshotStore
- [ ] Capture: CAPS caching + StateProvider impl + tick capture
- [ ] API: pipelineDump endpoint + CDI wiring
- [ ] Integration: CognitionStateProvider integration test

## References

- Spec: `wsp specs/issue-851-dynamic-regulatory-capacity/2026-11-09-cognitive-state-dump-design.md`
- Plan: `wsp plans/2026-11-09-cognitive-state-dump.md`
- Decisions: `wsp specs/issue-851-dynamic-regulatory-capacity/decisions.md` (D15-D19)
- Research: `neocortex docs/research/2026-11-05-post-era6-regulation-and-identity.md` (Stream 9)
- Blog: `examples docs/blog/2026-11-09-mdp02-where-the-why-lives.md`
- Blog fix: `examples docs/blog/2026-11-09-mdp01-the-test-that-broke-the-math.md` (SVG paths)
- Contract tests: `InMemoryPipelineSnapshotStoreTest` (9/9 pass)
- Neocortex commits: `f15c846c` (Task 1), `c1e2efa1` (Task 2), `26ea157c` (research update)
- Decision review: `/Users/mdproctor/reviews/casehub-slots/issue-851-decision-20261109-021336/`
