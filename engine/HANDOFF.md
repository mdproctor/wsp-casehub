# Handoff — 2026-07-28

## What's Done

- **Phases 4–5 committed** (both repos): Engine got DagPlan factories + HTN SPI types; blocks migrated to engine-api DagPlan and deleted ExecutionPlan + old decomposition types.
- **Phase 6 complete**: `DecompositionStrategy` wired into `EngineStrategyResolver` (CDI discovery via `Instance<DecompositionStrategy<?>>`). `CaseDefinition.decompositionStrategy` field added (nullable String, builder, getter/setter). YAML schema `spec.decompositionStrategy:` parsed by `CaseDefinitionYamlMapper`. Tests: 7 resolver tests + 94 YAML mapper tests green. CLAUDE.md synced.

## Immediate Next Step

**Start Phase 7 — scoped to dispatch wiring + verifications only.** GOAP and DispositionAwareRouting deferred to separate issues (new features, not migration). Read the Phase 7 analysis below, then begin with Task 7.1.

## Phase 7 Scoping Decision

Phase 7 as planned has 5 tasks. After first-principles analysis of the blocks execution path (`AgentRef` → `AgentInvoker` → `OrchestratedDriver`), the scope was narrowed:

**In scope (this branch):**
- **7.1 — Agent dispatch wiring**: Extend `AgentInvoker` to handle all 5 `AgentRef` variants. Three are self-contained (ExternalAgent already works, ChannelAgent has handler on record, ComposedAgent is recursive). Two need engine services (WorkerAgent needs WorkerFunctionHandler chain, HumanAgent needs work-api).
- **7.2 — Verify LlmSelectedRouting** compiles after Phase 5 type changes.
- **7.5 — Verify ConvergenceTermination** works with compound PlanItems.

**Deferred (file as separate issues):**
- **7.3 — GoalOrientedDecomposition (GOAP)**: New DecompositionStrategy. Depends on agent I/O capability schemas that may not exist. Not migration — new feature.
- **7.4 — DispositionAwareRouting**: New RoutingSignalProvider. Depends on eidos disposition data. Not migration — new feature.

## Key Files for Phase 7

- `blocks/.../agentic/AgentRef.java` — sealed interface, 5 variants (WorkerAgent, ChannelAgent, HumanAgent, ExternalAgent, ComposedAgent)
- `blocks/.../agentic/model/AgentInvoker.java` — FunctionalInterface, `defaultInvoker()` only handles ExternalAgent
- `blocks/.../agentic/model/AbstractExecutionDriver.java` — five-phase loop, calls `invoker.invoke(agent, context)`
- `blocks/.../agentic/model/OrchestratedDriver.java` — while-loop subclass
- `blocks/.../agentic/routing/LlmSelectedRouting.java` — verify compiles
- `blocks/.../agentic/termination/ConvergenceTermination.java` — verify works with compounds

## What's Left

- Phase 7 (scoped): agent dispatch wiring + verifications · M · Med
- Phase 8: Quarkus Flow backend for workflow patterns · M · Med
- File issues for GOAP (blocks#TBD) and DispositionAwareRouting (blocks#TBD) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| blocks#60 | Phase 7 — Agent dispatch + verifications (scoped) | M | Med | Next — see scoping above |
| blocks#60 | Phase 8 — Quarkus Flow backend | M | Med | After Phase 7 |
| TBD | GoalOrientedDecomposition (GOAP) | M | Med | Deferred from Phase 7 — file issue |
| TBD | DispositionAwareRouting | S | Med | Deferred from Phase 7 — file issue |
