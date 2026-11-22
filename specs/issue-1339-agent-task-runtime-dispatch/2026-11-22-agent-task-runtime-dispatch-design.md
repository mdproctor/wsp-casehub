# Design: Agent-Task Runtime Dispatch Handler for AgentRegistry Capability Matching

**Issue:** casehubio/engine#1339
**Date:** 2026-11-22
**Branch:** `issue-1339-agent-task-runtime-dispatch`

## Summary

Wire the runtime dispatch path for `AgentTaskTarget` bindings. When a binding targets an `AgentTaskTarget`, the dispatch handler queries `AgentRegistry.find(AgentQuery.byCapabilityAndDomain(...))` to select an agent, then delegates execution to the existing `AgentWorkerFunctionHandler` via a new identity-based deferred resolution path.

## Motivation

`AgentTaskTarget` was added to the sealed `BindingTarget` hierarchy in #1333 along with `BindingExecutorResolver` support, but no runtime dispatch handler exists. `CaseContextChangedEventHandler.publishByTarget()` — the central switch that dispatches bindings by target type — has no case arm for `AgentTaskTarget`. The binding type compiles but doesn't dispatch.

## Design

### Core Principle: Selection vs Mechanics

The design separates two responsibilities:

- **Dispatch handler (selection):** Resolves capability → agent identity. Answers "which agent handles this?"
- **Execution handler (mechanics):** Resolves agent identity → descriptor → prompt → invoker → LLM call. Answers "how to invoke it?"

This avoids both the double-lookup-on-different-keys problem (capability→slot→slot-query) and the eager-resolution problem (prompt rendering in the dispatch handler).

### 1. AgentTaskConfig — New API Record

New record in `api` module, package `io.casehub.api.model.ai`:

```java
public record AgentTaskConfig(
    String agentId,
    String tenancyId,
    String inputProjection,
    String outputProjection) {
  public AgentTaskConfig {
    Objects.requireNonNull(agentId, "agentId is required");
    Objects.requireNonNull(tenancyId, "tenancyId is required");
  }
}
```

Analogous to `AgentSlotConfig` but identity-based: carries the `agentId` of the selected agent rather than a slot reference. The `inputProjection` and `outputProjection` are JQ strings from the `AgentTaskTarget`'s mappings.

### 2. AgentWorkerFunction — Third Construction Path

Extend `AgentWorkerFunction` with a third mode alongside `eager(invoker)` and `ofSlot(slotConfig)`:

```java
public record AgentWorkerFunction(
    WorkerAgentInvoker invoker,
    AgentSlotConfig slotConfig,
    AgentTaskConfig agentTaskConfig)
    implements WorkerFunction<Map<String, Object>, Map<String, Object>> {

  // existing: eager(invoker), ofSlot(slotConfig)

  public static AgentWorkerFunction ofAgent(AgentTaskConfig config) {
    return new AgentWorkerFunction(null, null, Objects.requireNonNull(config));
  }

  public boolean isAgentTask() {
    return agentTaskConfig != null;
  }
}
```

### 3. Dispatch Handler — publishAgentTaskSchedule()

New private method in `CaseContextChangedEventHandler`, called from the `publishByTarget()` switch:

```java
case AgentTaskTarget at ->
    publishAgentTaskSchedule(caseInstance, caseDefinition, binding, at);
```

The method:

1. Queries `AgentRegistry.find(AgentQuery.byCapabilityAndDomain(at.capability(), at.domain(), caseInstance.tenancyId))`
2. If empty → log warning, return (no matching agent)
3. Takes first match (best quality) → extracts `descriptor.agentId()`
4. Builds `AgentTaskConfig` with agentId, tenancyId, and input/output projections from the target's `ExpressionEvaluator` mappings
5. Constructs synthetic `Worker` with `AgentWorkerFunction.ofAgent(config)` and a synthetic `Capability`
6. Dispatches via `WorkerScheduleEvent`

`CaseContextChangedEventHandler` needs `AgentRegistry` as a new constructor dependency. It is already a CDI bean in `runtime-core` (with `NoOpAgentRegistry` as a fallback).

### 4. Execution Handler — executeAgentTask()

Extend `AgentWorkerFunctionHandler.execute()` to handle the `isAgentTask()` path:

```java
if (agent.isAgentTask()) {
  return executeAgentTask(agent.agentTaskConfig(), worker, inputData, metadata);
}
```

The `executeAgentTask` method:

1. `agentRegistry.findById(config.agentId(), config.tenancyId())` → `Optional<AgentDescriptor>`
2. If empty → `HandlerResult(WorkerResult.failed("Agent not found: " + agentId))`
3. `systemPromptRenderer.render(descriptor, AgentPromptContext.forFormat(MARKDOWN))` → system prompt
4. Build input/output transformers from `config.inputProjection()` / `config.outputProjection()` (same as `executeDeferred()`)
5. Construct `WorkerAgentInvoker` with rendered prompt, transformers
6. Delegate to existing `executeWithInvoker(invoker, worker, inputData)`

### 5. Switch-Site Updates

`BindingTarget` is sealed, so the compiler identifies every exhaustive switch that needs updating. Beyond the dispatch handler:

| Site | Module | Change |
|------|--------|--------|
| `SchedulerService.registerScheduledTriggers()` | `runtime-core` | Add `AgentTaskTarget` case — log warning, skip (scheduled agent-task not supported initially) |
| `SchedulerService.createJobData()` | `runtime-core` | Add `AgentTaskTarget` case — throw `IllegalStateException` |
| `QuartzWorkerExecutionManager.createTriggerJobData()` | `scheduler-quartz` | Add `AgentTaskTarget` case — throw `IllegalStateException` |
| `CbrCaseRetainObserver.buildRoutingKeyMap()` | `runtime` | Add `AgentTaskTarget` case — map binding name to capability name |
| `BindingExecutorResolver.resolve()` | `common-core` | Already has the case arm — no change needed |

### 6. Constructor Wiring

`CaseContextChangedEventHandler` constructor gains `AgentRegistry` parameter. Update in:

- `RuntimeBeans.caseContextChangedEventHandler()` (Quarkus CDI producer in `runtime`)
- `RuntimeManualConfig.caseContextChangedEventHandler()` (Spring config in `runtime-spring`)

## Module Placement

| Artifact | Module |
|----------|--------|
| `AgentTaskConfig` | `api` |
| `AgentWorkerFunction` changes | `api` |
| `publishAgentTaskSchedule()` | `runtime-core` |
| `AgentWorkerFunctionHandler` changes | `runtime` |
| `SchedulerService` changes | `runtime-core` |
| `RuntimeBeans` wiring | `runtime` |
| `RuntimeManualConfig` wiring | `runtime-spring` |
| Switch-site updates | various (compiler-guided) |

## Error Handling

- **No matching agent at dispatch:** `AgentRegistry.find()` returns empty list → log warning with capability, binding name, case ID. Binding not dispatched. PlanItem stays PENDING.
- **Agent deregistered between dispatch and execution:** `findById()` returns `Optional.empty()` → `HandlerResult(WorkerResult.failed("Agent not found: " + agentId))`. PlanItem transitions to FAILED.
- **LLM invocation failure:** Existing `executeWithInvoker()` handles timeouts, exceptions, and empty responses — produces appropriate `HandlerResult`.

## Testing Strategy

- **Unit: `AgentTaskConfigTest`** — record construction, null agentId rejected
- **Unit: `AgentWorkerFunctionTest`** — extend existing test for `ofAgent()` path, `isAgentTask()` predicate, mutual exclusivity of construction modes
- **Unit: `AgentWorkerFunctionHandlerTest`** — extend existing test for agentTask path: mock `findById()`, mock `SystemPromptRenderer`, verify invoker construction and execution
- **Integration: `AgentTaskTargetDispatchTest`** — `@QuarkusTest` end-to-end: CaseDefinition with `AgentTaskTarget` binding, test `AgentRegistry` with registered agent, verify dispatch-to-execution flow

## References

- `AgentTaskTarget.java` (api) — the binding target type (from #1333)
- `CaseContextChangedEventHandler.publishByTarget()` (runtime-core:503) — central dispatch switch
- `CaseContextChangedEventHandler.publishJudgmentSchedule()` (runtime-core:846) — judgment dispatch pattern
- `AgentWorkerFunctionHandler.executeDeferred()` (runtime:108) — slot-based deferred resolution pattern
- `AgentWorkerFunction.java` (api) — existing eager/slot construction paths
- `AgentSlotConfig.java` (api) — slot-based config record (parallel to new AgentTaskConfig)
- `AgentRegistry.find()`, `AgentRegistry.findById()` (eidos-api) — registry query APIs
- `AgentQuery.byCapabilityAndDomain()` (eidos-api) — capability+domain query factory
- `SystemPromptRenderer.render()` (eidos-api) — prompt rendering
- `WorkerScheduleEvent.java` (common-core) — event for worker execution pipeline
- `BindingExecutorResolver.java` (common-core) — executor resolution switch
- `SchedulerService.java` (runtime-core) — schedule-triggered dispatch
- `NoOpAgentRegistry.java` (runtime-core) — fallback when eidos not on classpath
- casehubio/engine#1333 — parent issue, agent-task binding model
- `docs/specs/issue-1333-agent-worker-function-provider/2026-11-20-agent-worker-function-provider-design.md` — #1333 design spec
