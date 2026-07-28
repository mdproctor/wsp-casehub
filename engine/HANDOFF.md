# Handoff — 2026-07-28

## What's Done

- **blocks#60 Phases 0–3C.3 complete**: Stage fully retired. 12 Stage files deleted (-2233 lines), 6 integration tests migrated to Compound, 345 tests pass. Key fixes this session: gating order (CompoundLifecycleEvaluator before scoped-binding gating), CompoundCompletionEvaluator wired into 3 handlers (passes bindingName not planItemId), CompoundStrategyDispatcher uses case-level planningStrategy for free-floating bindings. Garden entry GE-20260728-0dc033 captures the gating order gotcha.

## Immediate Next Step

Phase 4: DAG plan unification in blocks repo. `ExecutionPlan<T>` in blocks is structurally equivalent to `DagPlan<T>` in engine-api. Convergence path: promote `ExecutionPlan<T>` to engine-api and merge `DagPlan<T>` into it. Start by reading `blocks/src/main/java/.../ExecutionPlan.java` and comparing with `engine/api/src/main/java/.../plan/DagPlan.java` — map the type differences, identify which fields/methods are blocks-specific vs engine-specific, and design the unified type.

## Phase 4 Detail — DAG Plan Unification

**Goal:** Single `DagPlan<T>` type in engine-api used by both blocks and engine. Currently two parallel implementations:
- `DagPlan<T>` (engine-api) — immutable validated DAG with `entryNodeIds()`, `exitNodeIds()`, `topologicalSort()`, `sequentialMerge()`. Factories: `singleton`, `sequence`, `parallel`.
- `ExecutionPlan<T>` (blocks) — blocks-specific DAG with `LeafTask` nodes.

**Approach:**
1. Read both types. Map field-by-field: what's shared, what's blocks-only, what's engine-only.
2. Design unified type — `DagPlan<T>` is already generic. `ExecutionPlan<T>` likely needs `LeafTask implements TaskDescriptor` (blocks#51) before it can use a generic node type.
3. If blocks#51 is not done, check whether it can be done as part of this phase or should block it.
4. Move unified type to engine-api. Update blocks to use it. Update engine DagDriver to use it.

**Key constraint from CLAUDE.md:** "DagPlan<T> is structurally equivalent but generic over T (no blocks coupling). Convergence path: when blocks#51 (LeafTask implements TaskDescriptor) ships, ExecutionPlan<T> can be promoted to engine-api and DagPlan<T> merges into it."

**Files to read first:**
- `blocks/src/main/java/` — find `ExecutionPlan.java`, `LeafTask.java`
- `engine/api/src/main/java/io/casehub/api/model/plan/DagPlan.java`
- `engine/common/src/main/java/io/casehub/engine/common/internal/plan/DagDriver.java`

## What's Left

- Phase 4: DAG plan unification (ExecutionPlan → DagPlan) · M · Med
- Phase 5: HTN SPI promotion to engine-api · M · Med
- Phase 6: Composable strategy wiring via StrategyResolver · S · Low
- Phase 7: Agent dispatch wiring + GOAP + disposition routing · L · Med
- Phase 8: Quarkus Flow backend for workflow patterns · M · Med
- Uncommitted formatting changes in persistence-hibernate/memory (IntelliJ auto-format) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| blocks#60 | Phase 4 — ExecutionPlan → DagPlan unification | M | Med | Next — cross-repo (blocks + engine) |
| blocks#60 | Phase 5 — HTN SPI promotion | M | Med | After Phase 4 |
| blocks#60 | Phase 6 — Composable strategy wiring | S | Low | After Phase 5 |
