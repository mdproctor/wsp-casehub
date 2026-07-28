# Handoff — 2026-07-28

## What's Done

- **blocks#60 Phases 4–5 complete**: DAG plan unified (`ExecutionPlan` deleted, `DagPlan` is the single plan type). HTN decomposition SPI promoted to engine-api (`TaskNode`, `DecompositionStrategy extends NamedStrategy`, `DecompositionMethod`, `DecompositionContext`). `LeafTask` is `non-sealed` — blocks defines `PrimitiveTask`/`PlannedTask` without engine-api coupling. CLAUDE.md updated. Design journal updated. Blog entry written.

## Immediate Next Step

**Commit the uncommitted changes in both repos** (engine + blocks), then start Phase 6.

Engine has: 4 new files in `api/src/.../plan/` (TaskNode, DecompositionStrategy, DecompositionMethod, DecompositionContext), modified DagPlan + tests, CLAUDE.md update. Blocks has: deleted ExecutionPlan + old decomposition types, 3 new files (PrimitiveTask, PlannedTask, AgenticDecompositionContext), migrated 8 production + 7 test files.

## Phase 6 Detail — Composable Strategy Wiring

**Goal:** Wire `DecompositionStrategy` into `EngineStrategyResolver` so YAML case definitions can select a decomposition strategy by name, the same way they select routing and matching strategies.

**What's already in place (from Phase 5):**
- `DecompositionStrategy<T> extends NamedStrategy` — has `id()`, discoverable via CDI `Instance<>`
- `EngineStrategyResolver` (`runtime/internal/routing/`) — already resolves `AgentRoutingStrategy`, `ImplementationRoutingStrategy`, `CandidateMatchingStrategy`, `HumanTaskRoutingStrategy` via per-type `Instance<>` injection
- `CaseDefinition` already has `agentRouting`, `implementationRouting`, `candidateMatching`, `humanTaskRouting` (nullable String strategy IDs)

**What Phase 6 adds:**
1. Add `Instance<DecompositionStrategy<?>>` to `EngineStrategyResolver` constructor — follows existing pattern
2. Add `decompositionStrategy` field to `CaseDefinition` (nullable String) + builder method
3. YAML: `decompositionStrategy:` key parsed by `CaseDefinitionYamlMapper`
4. Integration point: wherever decomposition is invoked, resolve via `StrategyResolver` instead of direct construction

**Key files to read first:**
- `engine/runtime/src/.../routing/EngineStrategyResolver.java` — see how existing strategies are wired
- `engine/api/src/.../engine/CaseDefinition.java` — existing strategy ID fields
- `engine/schema/src/.../CaseDefinitionYamlMapper.java` — YAML parsing for strategy fields
- `blocks/src/.../pattern/HtnBuilder.java` — current decomposition strategy consumer

**Known constraint from CLAUDE.md:** "Adding new strategy SPI types requires updating this resolver's constructor."

## What's Left

- Phase 6: Composable strategy wiring via StrategyResolver · S · Low
- Phase 7: Agent dispatch wiring + GOAP + disposition routing · L · Med
- Phase 8: Quarkus Flow backend for workflow patterns · M · Med
- Uncommitted changes in engine + blocks from Phases 4–5 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| blocks#60 | Phase 6 — Composable strategy wiring | S | Low | Next — mechanical, pattern exists |
| blocks#60 | Phase 7 — Agent dispatch + GOAP | L | Med | After Phase 6 |
| blocks#60 | Phase 8 — Quarkus Flow backend | M | Med | After Phase 7 |
