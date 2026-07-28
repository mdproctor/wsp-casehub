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

### What remains
Phase 4 (blocks DAG unification), Phase 5 (HTN SPI promotion), Phase 6 (composable strategy wiring), Phase 7 (agent dispatch + GOAP), Phase 8 (Quarkus Flow backend).
