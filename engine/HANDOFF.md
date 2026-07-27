# Handoff — 2026-07-28

## What's Done

- **blocks#60 Phases 0–3D + compound wiring**: PlanningStrategyLoopControl now uses Compound-based gating (scopedBindings + definition status), CompoundLifecycleEvaluator for activation, and CompoundStrategyDispatcher for per-compound strategy dispatch. Stage-based gating removed from the dispatch path. 6 commits this session: Compound.builder() + scopedBindings, design review fixes (index bug, dispatchMode, HYBRID), CompoundLifecycleEvaluator, PlanningStrategyLoopControl rewiring, compound completion for scoped bindings. Design review subagent caught a critical index conflation bug before wiring. PlanItemDefinition naming validated against HTN/BPMN/CMMN/Temporal/Airflow — follows the established Definition/Instance pattern. Unit tests pass (394+). 10 integration tests fail (Stage-using @QuarkusTest tests need migration).

## Immediate Next Step

Migrate 10 failing integration tests from Stage to Compound. The tests are in `planning/src/test/java/io/casehub/engine/planning/it/` — StageBlackboardTest (5 errors), ExitConditionBlackboardTest, LambdaEntryConditionBlackboardTest, PlanConfigurerBlackboardTest, SequentialStagesBlackboardTest, SequentialStrategyIntegrationTest. Pattern: replace `Stage.alwaysActivate()/builder()` + `addStage()` with `Compound.builder()` + `registerDefinition()`. Replace `stage.getStatus() == StageStatus.ACTIVE` assertions with `plan.getDefinitionStatus(id) == TaskStatus.RUNNING`. After migration, delete Stage and related files (Stage, StageStatus, StageLifecycleEvaluator, StageAutocompleteEvaluator, StageResetOutcomesCleaner, Stage events, Stage methods on CasePlanModel).

## Plan Corrections Discovered This Session

1. **evaluateCompletion gap**: Compounds with only `scopedBindings` (no structural children) need completion to check PlanItem statuses of scoped bindings. Fixed — `evaluateCompletion()` now counts both structural children (definition status) AND scoped bindings (PlanItem status).
2. **DispatchMode.HYBRID removed**: Undefined enum value with no semantics.
3. **dispatchMode moved to Compound only**: Primitives are atomic — no dispatch semantics.
4. **PlanItemDefinition naming validated**: Research against HTN/BPMN/CMMN/Temporal/Airflow confirmed Definition/Instance is the dominant naming pattern. Keep PlanItemDefinition/PlanItem.

## What's Left

- Migrate 10 integration tests from Stage to Compound · M · Med
- Delete Stage and related files (Stage, StageStatus, evaluators, events, CasePlanModel Stage methods) · S · Low
- Phase 4: DAG plan unification in blocks (ExecutionPlan → DagPlan) · M · Med
- Phase 5: HTN SPI promotion to engine-api · M · Med
- Phase 6: Composable strategy wiring via StrategyResolver · S · Low
- Phase 7: Agent dispatch wiring + GOAP + disposition routing · L · Med
- Phase 8: Quarkus Flow backend for workflow patterns · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| blocks#60 | Migrate integration tests + delete Stage | M | Med | Next — clears the path |
| blocks#60 | Phase 4 — ExecutionPlan → DagPlan in blocks | M | Med | Independent, blocks repo |
| blocks#60 | Phase 5 — HTN SPI promotion | M | Med | After Phase 4 |
