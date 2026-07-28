# Design Journal — issue-60-unified-execution-model

## 2026-07-26 — Phases 0–3D: Foundation through per-compound dispatch

### Plan correction: circular dependency in Phase 0
The plan said "add engine-common as dependency of engine-api." Common already depends on api — this creates a cycle. Fixed by moving DagPlan/DagNode/JoinType TO engine-api instead. Protocol PP-20260727-5267d2 formalises the boundary: plan-definition types in api, execution types in common.

### Phase 1: @DefaultBean pattern for LoopControl
Plan said to delete ChoreographyLoopControl and make PlanningStrategyLoopControl the sole impl. Blocked by blackboard→runtime Maven cycle — can't add blackboard as a runtime dependency. Solution: ChoreographyLoopControl becomes @DefaultBean (yields to PlanningStrategyLoopControl when planning module is on classpath). This is the standard Quarkus pattern already used by NoOpWorkerProvisioner etc. Aligns with spec invariant 4 ("no compile-time alternatives").

### Phase 3A: PlanItemDefinition naming
Plan used `NewPlanItem` as temporary name. Used `PlanItemDefinition` instead — encodes the domain concept (immutable plan definition vs mutable execution state) rather than a migration artifact. The sealed hierarchy: `PlanItemDefinition.Primitive` (dispatches a worker) and `PlanItemDefinition.Compound` (dispatches children via named strategy). `DispatchMode` enum (ORCHESTRATED/CHOREOGRAPHED/HYBRID) and `CompletionSemantics` sealed interface (ALL/M_OF_N/FIRST_WINS) are separate types in the same package.

### Phase 3B: CasePlanModel compound API
Added as default methods on the interface — existing implementations work unchanged. DefaultCasePlanModel uses ConcurrentHashMap for definitions, states, parent-child index. `evaluateCompletion()` evaluates CompletionSemantics against children's terminal status. Recursive `registerDefinition()` handles nested compounds.

### Phase 3C-3D: Evaluator and dispatch
CompoundCompletionEvaluator walks the parent chain upward, completing each compound whose CompletionSemantics is satisfied. CompoundStrategyDispatcher groups eligible bindings by containing compound, resolves strategy by name, calls the 4-arg select. Free-floating bindings use the default (choreography) strategy.

## 2026-07-28 — Phase 3C.3: Stage retirement

### Stage fully retired
12 Stage files deleted, 6 integration tests migrated to Compound. Stage.java, StageStatus.java, StageLifecycleEvaluator, StageAutocompleteEvaluator, StageResetOutcomesCleaner, 3 Stage event types, 4 Stage test files. CasePlanModel Stage methods removed. Net -2233 lines.

### Gating order fix
CompoundLifecycleEvaluator must evaluate BEFORE gating in PlanningStrategyLoopControl. Without this, compounds activated in the current cycle are still PENDING when gating runs — their scoped bindings are silently filtered out. No error, test just times out. Garden entry GE-20260728-0dc033.

### CompoundCompletionEvaluator wired into handlers
Replaced StageAutocompleteEvaluator in PlanItemCompletionHandler, WorkerRetryExhaustionHandler, WorkerOutcomeResolvedHandler. Key change: passes `item.getBindingName()` (not planItemId) because the compound parent index maps binding names, not PlanItem UUIDs.

### CompoundStrategyDispatcher case-level strategy
Free-floating bindings (not scoped to any compound) now use `CaseDefinition.getPlanningStrategy()` instead of hardcoded "default". Fixes SequentialStrategyIntegrationTest — sequential strategy was being ignored for cases without compounds.

## 2026-07-28 — Phase 4: DAG plan unification

### ExecutionPlan deleted — DagPlan is the single plan type
`ExecutionPlan<T>` (blocks, 245 lines) deleted. Blocks uses `DagPlan<LeafTask<T>>` from engine-api. The type parameter carries the blocks constraint — the plan infrastructure is generic.

### DagPlan API improved
- Renamed `sequence(List<DagNode<T>>)` → `fromNodes()` — the old name was misleading; it didn't auto-wire dependencies, just converted a list to a map.
- Added `singleton(T)` with auto-generated ID `"node-0"`.
- Added `sequence(List<? extends T>)` — auto-wires a sequential chain with generated IDs. This is the real sequence factory (from blocks' `ExecutionPlan.sequence`).
- Widened `parallel(List<T>)` → `parallel(List<? extends T>)` for subtype acceptance.
- `fromList()` alias dropped (was just `sequence()`).

### JoinType consolidated
Engine's top-level `JoinType` is the single source. Blocks' inner `ExecutionPlan.JoinType` deleted. Values identical (ALL_OF, ANY_OF).

## 2026-07-28 — Phase 5: HTN decomposition SPI promotion

### TaskNode promoted with non-sealed LeafTask
`TaskNode<T>` (sealed: `LeafTask<T>` + `CompoundTask<T>`) moved to engine-api `io.casehub.engine.plan`. `LeafTask` is `non-sealed` — blocks defines concrete implementations (`PrimitiveTask`, `PlannedTask`) without engine-api knowing about them. This is the key design decision: Java sealed interfaces require permits in the same compilation unit, but `non-sealed` within a sealed hierarchy allows cross-module extension while keeping `TaskNode` itself closed.

### DecompositionContext as interface
`DecompositionContext<T>` promoted as an interface (not record) with `state()` and `depth()`. Blocks provides `AgenticDecompositionContext<T>` adding `agents()`. Strategies cast when they need the richer context — safe because the builder always constructs the blocks-specific context. Alternative considered: `List<? extends ExecutorRef>` on the engine context — rejected because `RoutingCandidate` (which carries `AgentDescriptor` for LLM prompt building) doesn't extend `ExecutorRef`.

### DecompositionStrategy extends NamedStrategy
Ready for `StrategyResolver` wiring in Phase 6. Default id `"identity"`. `EngineStrategyResolver` will need a new `Instance<DecompositionStrategy<?>>` injection point.

### PrimitiveTask and PlannedTask promoted to top-level records
Were inner records of `TaskNode` in blocks. Now top-level records in `io.casehub.blocks.agentic.decomposition` implementing `TaskNode.LeafTask<T>`. `executor()` delegates to `agent()` field. `agent()` remains blocks-specific (not on the `LeafTask` interface).

### What remains
Phase 6 (composable strategy wiring via StrategyResolver), Phase 7 (agent dispatch + GOAP + disposition routing), Phase 8 (Quarkus Flow backend).
