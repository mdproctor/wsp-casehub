# Lifecycle Scopes for Workers — engine#237

**Date:** 2026-07-29
**Issue:** casehubio/engine#237
**Status:** Design

## Summary

Workers in the engine currently have no structured lifecycle scope — they are activated per binding trigger and complete independently. This design introduces first-class lifecycle scoping: workers can be declared to persist for the duration of a Compound or an entire Case, receiving context changes throughout their scope's lifetime.

Three scopes: **BINDING** (current default — one activation, one execution), **COMPOUND** (worker lives for the owning Compound's lifetime), and **CASE** (worker persists for the full case lifetime).

Two execution strategies: **PERSISTENT** (long-running thread with a mailbox, receives context events) and **REINVOKED** (re-invoked on each context change with accumulated state from prior invocations).

Two participation roles: **PARTICIPANT** (counts toward scope completion) and **COMPANION** (excluded from completion, terminated when scope ends — the Kubernetes sidecar pattern).

Two activation modes: **scope-activated** (starts when the scope activates) and **binding-triggered** (starts when binding condition first matches, persists for the scope).

## Prerequisites

- Unified execution model (blocks#60) — Compound replaces Stage, sealed PlanItemDefinition hierarchy. **Delivered.**

## Design Decisions

### Scope declared on Binding, not Worker or PlanItemDefinition

The Binding is the dispatch control point — it already governs trigger conditions, input projection, outcome policy, and agent routing. Scope is a natural extension of what the binding controls: how long the dispatched worker persists.

A Worker is a reusable definition (foundation-tier record in `casehubio/worker`). The same worker can be used with BINDING scope in one case definition and COMPOUND scope in another. Scope is a property of deployment, not definition.

PlanItemDefinition was considered (Approach 2 — Compound-native). Rejected because it creates a parallel dispatch path that must reimplement or bridge every feature the binding system already provides (routing, projections, outcome handling, CBR retrieval).

### Sidecar model for completion semantics

Research across CMMN, Kubernetes, actor models (Akka/Erlang), and BPMN multi-instance confirms that the cleanest pattern separates scope lifetime from completion semantics:

- **Kubernetes sidecars** (GA v1.33): init containers with `restartPolicy: Always` run for the Pod's lifetime but don't extend Pod lifetime. Main containers determine completion.
- **CMMN required flag:** Plan items within a stage can be marked required or not. Non-required items don't block stage completion.
- **BPMN completion condition:** A boolean expression evaluated after each instance completes. Remaining active instances are terminated when it fires.
- **Akka supervision:** Parent termination cascades to all children unconditionally.

The common thread: the container's completion is determined by a subset of its contents, and the rest is terminated when the container completes.

Applied here: `Participation.COMPANION` workers are excluded from `evaluateCompletion` and terminated after the scope completes. `Participation.PARTICIPANT` workers count toward completion and must signal "done" for the scope to complete.

### One dispatch path, not two

Scope-activated workers flow into the existing scheduling infrastructure after their initial trigger. `CompoundLifecycleEvaluator` fires the initial dispatch via `WorkerScheduleEvent`, which enters the same `WorkerScheduleEventHandler` → `WorkerExecutionManager` → `QuartzWorkerExecutionJob` path as all other workers.

Binding-triggered scoped workers go through the existing `CaseContextChangedEventHandler.publishWorkerSchedule()` path with an interception point: the handler checks `ScopedWorkerRegistry` before creating a new PlanItem. If a session exists, context is routed to the existing session.

One path means all existing features (agent routing, input projection, outcome handling, CBR, action risk classification) work for scoped workers automatically on first activation.

## Core Types

### LifecycleScope

Enum in `engine-api`, package `io.casehub.api.model`.

```java
public enum LifecycleScope {
    BINDING,    // one activation, one execution (current default)
    COMPOUND,   // lives for the owning Compound's lifetime
    CASE        // lives for the entire case lifetime
}
```

### Participation

Enum in `engine-api`, package `io.casehub.api.model`.

```java
public enum Participation {
    PARTICIPANT,  // counts toward completion (default)
    COMPANION     // excluded from completion, terminated when scope ends
}
```

### ExecutionMode

Enum in `engine-api`, package `io.casehub.api.model`.

```java
public enum ExecutionMode {
    SINGLE,      // fire-and-forget (current default)
    PERSISTENT,  // long-running thread, receives context events via mailbox
    REINVOKED    // re-invoked on each context change with accumulated state
}
```

### ScopeActivatedTrigger

Record in `engine-api`, package `io.casehub.api.model`.

```java
public record ScopeActivatedTrigger() implements Trigger {}
```

Fires when the owning Compound activates (transitions to RUNNING). For CASE scope, fires when the case starts. The binding's `when` condition is still respected as a guard.

### Binding changes

`Binding` gains three fields, all defaulting to current behavior:

```java
public class Binding {
    // ... existing fields ...
    private LifecycleScope lifecycleScope;    // default BINDING
    private Participation participation;       // default PARTICIPANT
    private ExecutionMode executionMode;        // default SINGLE
}
```

Builder methods: `.lifecycleScope(LifecycleScope)`, `.participation(Participation)`, `.executionMode(ExecutionMode)`.

### Validation rules (build-time, in `Binding.Builder.build()`)

- `BINDING` scope requires `ExecutionMode.SINGLE`
- `COMPOUND` scope requires the binding to be in a Compound's `scopedBindings`
- `CASE` scope is valid on any binding (case-global, no compound membership required)
- `COMPANION` participation requires `COMPOUND` or `CASE` scope
- `ScopeActivatedTrigger` requires `COMPOUND` or `CASE` scope

## Runtime Infrastructure

### ScopedWorkerSession

Sealed interface in `runtime`, package `io.casehub.engine.internal.worker.scope`.

```java
public sealed interface ScopedWorkerSession
    permits ScopedWorkerSession.Persistent, ScopedWorkerSession.Reinvoked {

    String bindingName();
    UUID caseId();
    String planItemId();
    LifecycleScope scope();
    Participation participation();

    record Persistent(
        String bindingName, UUID caseId, String planItemId,
        LifecycleScope scope, Participation participation,
        BlockingQueue<ContextEvent> mailbox,
        Future<?> workerThread
    ) implements ScopedWorkerSession {}

    record Reinvoked(
        String bindingName, UUID caseId, String planItemId,
        LifecycleScope scope, Participation participation,
        AtomicReference<Map<String, Object>> accumulatedState
    ) implements ScopedWorkerSession {}
}
```

**Persistent sessions** have a mailbox (`BlockingQueue<ContextEvent>`) and a running virtual thread. The worker function runs in a loop: take from mailbox → process → apply output via `WorkerRuntime.signal()`.

**Reinvoked sessions** hold accumulated state from prior invocations. On each context change, the engine re-invokes the worker function with current context merged with accumulated state. Output is merged back into accumulated state.

### ContextEvent

Record in `runtime`, package `io.casehub.engine.internal.worker.scope`.

Mailbox message type for persistent workers. Carries the context snapshot, change metadata, and a shutdown sentinel.

### ScopedWorkerRegistry

`@ApplicationScoped` in `runtime`, package `io.casehub.engine.internal.worker.scope`.

```java
@ApplicationScoped
public class ScopedWorkerRegistry {

    private final ConcurrentHashMap<ScopeKey, ScopedWorkerSession> sessions;

    public Optional<ScopedWorkerSession> get(UUID caseId, String bindingName);
    public void register(ScopeKey key, ScopedWorkerSession session);
    public void terminateByScope(UUID caseId, String compoundId);
    public void terminateByCase(UUID caseId);
    public List<ScopedWorkerSession> getCompanions(UUID caseId, String compoundId);

    record ScopeKey(UUID caseId, String bindingName) {}
}
```

Keyed by `(caseId, bindingName)` — one scoped session per binding per case instance.

`terminateByScope(caseId, compoundId)` finds sessions whose binding is owned by the compound and terminates them.

`terminateByCase(caseId)` terminates all sessions for a case.

### Termination protocol

**Persistent:** Poison pill (`ContextEvent.SHUTDOWN`) on the mailbox. Worker loop reads it and exits cleanly. If the worker doesn't exit within `ExecutionPolicy.timeoutMs`, the thread is interrupted.

**Reinvoked:** Session removed from registry. Any in-flight invocation completes normally but output is discarded if scope has ended.

PlanItem transitions to COMPLETED (COMPANION) or is left in current state (PARTICIPANT — should already be terminal).

## Dispatch and Lifecycle Integration

### Scope-activated dispatch

`CompoundLifecycleEvaluator.activatePendingCompounds()` — after transitioning a compound to RUNNING, collects bindings with `ScopeActivatedTrigger` owned by the compound and publishes `WorkerScheduleEvent`s. The dispatch then enters the normal scheduling path.

For CASE-scoped scope-activated bindings, `CaseStartedEventHandler` dispatches them at case start.

### Binding-triggered scoped dispatch

`CaseContextChangedEventHandler.publishWorkerSchedule()` — before creating a new PlanItem:

1. Check `binding.lifecycleScope()` — if BINDING, proceed as current (no change).
2. If COMPOUND or CASE: check `scopedWorkerRegistry.get(caseId, bindingName)`.
3. If session exists and active: route context to existing session, return.
4. If no session: proceed with normal dispatch, then register a new `ScopedWorkerSession`.

**Routing context to an existing session:**

- Persistent: put `ContextEvent` on session's mailbox.
- Reinvoked: publish `WorkerScheduleEvent` with a re-invocation flag. `QuartzWorkerExecutionJob` reads accumulated state from session, merges with current context, invokes worker function, merges output back, applies output to case context. PlanItem stays RUNNING.

### Scope termination

`CompoundCompletionEvaluator.evaluate()` — after a compound transitions to COMPLETED, a new `ScopedWorkerTerminationHandler` (in `planning` module) consumes `CompoundCompletedEvent` and calls `scopedWorkerRegistry.terminateByScope(caseId, compoundId)`.

`CaseStatusChangedHandler` — on terminal case state, calls `scopedWorkerRegistry.terminateByCase(caseId)`.

Termination ordering matches Kubernetes: COMPANION workers terminate AFTER the compound completes, not during.

### PlanItem lifecycle for scoped workers

One PlanItem per scoped worker for the entire scope lifetime:
- Created at first activation
- Transitions to RUNNING immediately
- Stays RUNNING for scope duration
- Intermediate output applied via `signal()` (persistent) or modified completion handler (reinvoked)
- Transitions to COMPLETED when scope ends (COMPANION) or when worker signals "done" (PARTICIPANT)

### How PARTICIPANT workers signal "done"

- **Persistent:** Worker loop exits normally (returns from run method). PlanItem → COMPLETED.
- **Reinvoked:** Worker sets `_lifecycle.<bindingName>.done = true` in its output. Engine detects this, transitions PlanItem → COMPLETED. Session stays alive (can still receive events) but no longer blocks completion.

## Completion Semantics

### CompoundCompletionEvaluator changes

`evaluateCompletion` must distinguish between COMPANION and PARTICIPANT scoped bindings:

- COMPANION PlanItems are excluded from the completion check entirely.
- PARTICIPANT PlanItems count toward `CompletionSemantics` (All/MOfN/FirstWins) like any other child.

The existing `scopedBindings` on Compound tracks binding ownership. The evaluator looks up each binding's `Participation` from the `CaseDefinition` when checking completion.

### Completion truth table

| Participation | Compound completing | Worker done | Worker running |
|---------------|--------------------|----|---------|
| PARTICIPANT | Compound waits | Terminal PlanItem counts toward CompletionSemantics | Compound cannot complete |
| COMPANION | Compound ignores | No effect on completion | Worker terminated after compound completes |

### Edge cases

| Scenario | Behavior |
|----------|----------|
| COMPANION faults | Logged, session removed. Does NOT fault the compound. |
| PARTICIPANT faults | Normal fault handling — OutcomePolicy applies (REROUTE or FAULT). |
| Compound exits via exit condition | All scoped workers (both roles) terminated immediately. |
| Case cancelled | All scoped workers across all scopes terminated. |
| Repeatable compound resets | Sessions terminated on completion, new sessions created on re-activation. |

## Interaction with Existing Features

| Feature | Impact |
|---------|--------|
| Agent routing | First activation only — not on re-invocations |
| Outcome policy | First activation failure only. Re-invocation failures: logged, session state preserved |
| Input projection | First activation only. Re-invocations get full context snapshot |
| CBR retrieval | First activation only |
| Signal settlement | Scoped workers don't participate — they outlive individual signals |
| Action risk classifier | Per output application, same as current |

## YAML Schema

```yaml
bindings:
  - name: case-monitor
    capability: monitoring
    trigger: scope-activated
    lifecycleScope: CASE
    participation: COMPANION
    executionMode: PERSISTENT

  - name: analyst
    capability: analysis
    trigger:
      contextChange: ".request != null"
    lifecycleScope: COMPOUND
    participation: PARTICIPANT
    executionMode: REINVOKED

  - name: normal-worker
    capability: processing
    trigger:
      contextChange: ".input != null"
    # all defaults: BINDING / PARTICIPANT / SINGLE
```

`CaseDefinitionYamlMapper` parses `lifecycleScope`, `participation`, `executionMode` from binding nodes. `trigger: scope-activated` creates `ScopeActivatedTrigger`. All default to current behavior when absent.

## Persistence

### In-memory sessions (v1)

Scoped worker sessions are in-memory only, consistent with `CaseInstanceCache` and `BlackboardRegistry`.

On JVM restart:
- Persistent sessions lost. Recovery: `WorkerRecoveryCoordinator` detects RUNNING PlanItems with scoped bindings, re-creates sessions, re-dispatches workers (fresh start, no mailbox history).
- Reinvoked sessions lose accumulated state. Recovery re-creates with empty state. Next context change re-invokes with current context only.
- Durable accumulated state is a follow-on — requires `CaseContextStore` durability (engine#732).

### PlanItem persistence

One new field on `PlanItemRecord`/`PlanItemEntity`: `lifecycle_scope` (VARCHAR, nullable — null = BINDING).

### EventLog metadata

Worker schedule events for scoped workers carry:
- `lifecycleScope`: COMPOUND or CASE
- `participation`: COMPANION or PARTICIPANT
- `executionMode`: PERSISTENT or REINVOKED
- `scopeId`: compound ID or case ID

## Module Placement

| Type | Module | Package |
|------|--------|---------|
| `LifecycleScope` | `engine-api` | `io.casehub.api.model` |
| `Participation` | `engine-api` | `io.casehub.api.model` |
| `ExecutionMode` | `engine-api` | `io.casehub.api.model` |
| `ScopeActivatedTrigger` | `engine-api` | `io.casehub.api.model` |
| `ScopedWorkerSession` | `runtime` | `io.casehub.engine.internal.worker.scope` |
| `ScopedWorkerRegistry` | `runtime` | `io.casehub.engine.internal.worker.scope` |
| `ContextEvent` | `runtime` | `io.casehub.engine.internal.worker.scope` |
| `ScopedWorkerTerminationHandler` | `planning` | `io.casehub.engine.planning.handler` |

No changes to `casehubio/worker`, `casehub-engine-common`, `casehubio/blocks`, or `casehubio/platform`.

## Cross-Repo Impact

| Repo | Impact |
|------|--------|
| `casehubio/engine` | All changes |
| `casehubio/worker` | None |
| `casehubio/blocks` | None |
| `casehubio/platform` | None |
| Consumer repos | None until they opt into scoped bindings |

## Follow-On Issues

- Durable accumulated state for reinvoked sessions (depends on engine#732)
- Persistent session recovery with mailbox replay from EventLog
- External worker (WorkerFunction.None) lifecycle scope — Qhorus channel lifetime scoping
- YAML schema validation tooling for scope/participation/trigger consistency
