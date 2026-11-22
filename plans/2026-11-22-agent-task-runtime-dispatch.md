# Agent-Task Runtime Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #1339 — agent-task: runtime dispatch handler for AgentRegistry capability matching
**Issue group:** #1339

**Goal:** Wire the runtime dispatch path so `AgentTaskTarget` bindings resolve agents via `AgentRegistry` capability matching and execute them through the existing worker pipeline.

**Architecture:** Dispatch handler resolves capability→agentId (selection), then delegates to `AgentWorkerFunctionHandler` via a new `AgentWorkerFunction.ofAgent()` construction path. Execution handler resolves agentId→descriptor→prompt→invoker (mechanics). Projections are carried as `ExpressionEvaluator` instances and applied at execution time only.

**Tech Stack:** Java 21, Quarkus 3.32.2, eidos-api (AgentRegistry, SystemPromptRenderer), platform-agent-api (AgentProvider)

## Global Constraints

- Quarkus version: `3.32.2`
- eidos-api version: `0.4-SNAPSHOT`
- Sealed `BindingTarget` interface — compiler enforces exhaustive switches
- `@DefaultBean` MUST NOT be used on multi-instance SPIs (protocol PP-20260921-b7c277)
- Execution infrastructure types stay in `engine-common`/`runtime-core`; definition types in `api` (protocol PP-20260727-5267d2)
- Input/output mappings carried as `ExpressionEvaluator` (not String) to avoid lossy JQ assumption
- No `tenancyId` on `AgentTaskConfig` — use `ExecutionMetadata.tenancyId()` at execution time
- Projections applied at execution time only (not schedule time) to avoid double-projection

---

## Batch 1: API Foundation — AgentTaskConfig + AgentWorkerFunction extension

### Task 1: AgentTaskConfig record and AgentWorkerFunction third construction path

**Files:**
- Create: `api/src/main/java/io/casehub/api/model/ai/AgentTaskConfig.java`
- Modify: `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java`
- Modify: `api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java`

**Interfaces:**
- Consumes: `ExpressionEvaluator` (from `io.casehub.platform.api.expression`)
- Produces: `AgentTaskConfig(String agentId, ExpressionEvaluator inputMapping, ExpressionEvaluator outputMapping)`, `AgentWorkerFunction.ofAgent(AgentTaskConfig)`, `AgentWorkerFunction.isAgentTask()`, `AgentWorkerFunction.agentTaskConfig()`

- [ ] **Step 1: Write failing tests for AgentTaskConfig**

Add tests in `AgentWorkerFunctionTest.java` (the existing test class already covers `AgentSlotConfig`):

```java
@Test
void agentTaskConfig_requiresAgentId() {
  assertThrows(NullPointerException.class,
      () -> new AgentTaskConfig(null, null, null));
}

@Test
void agentTaskConfig_acceptsNullMappings() {
  var config = new AgentTaskConfig("agent-123", null, null);
  assertEquals("agent-123", config.agentId());
  assertNull(config.inputMapping());
  assertNull(config.outputMapping());
}

@Test
void ofAgent_createsAgentTaskMode() {
  var config = new AgentTaskConfig("agent-123", null, null);
  var fn = AgentWorkerFunction.ofAgent(config);
  assertTrue(fn.isAgentTask());
  assertFalse(fn.isDeferred());
  assertNull(fn.invoker());
  assertNull(fn.slotConfig());
  assertSame(config, fn.agentTaskConfig());
}

@Test
void mutualExclusivity_rejectsMultipleNonNull() {
  var invoker = testInvoker();
  var slotConfig = new AgentSlotConfig("slot", null, null, null, null);
  var taskConfig = new AgentTaskConfig("agent-123", null, null);
  assertThrows(IllegalArgumentException.class,
      () -> new AgentWorkerFunction(invoker, slotConfig, null));
  assertThrows(IllegalArgumentException.class,
      () -> new AgentWorkerFunction(null, slotConfig, taskConfig));
  assertThrows(IllegalArgumentException.class,
      () -> new AgentWorkerFunction(invoker, null, taskConfig));
}

@Test
void mutualExclusivity_rejectsAllNull() {
  assertThrows(IllegalArgumentException.class,
      () -> new AgentWorkerFunction(null, null, null));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl api -Dtest=AgentWorkerFunctionTest -q`
Expected: compilation failures (AgentTaskConfig doesn't exist, AgentWorkerFunction has 2-arg constructor)

- [ ] **Step 3: Create AgentTaskConfig record**

Create `api/src/main/java/io/casehub/api/model/ai/AgentTaskConfig.java`:

```java
package io.casehub.api.model.ai;

import io.casehub.platform.api.expression.ExpressionEvaluator;
import java.util.Objects;

public record AgentTaskConfig(
    String agentId,
    ExpressionEvaluator inputMapping,
    ExpressionEvaluator outputMapping) {
  public AgentTaskConfig {
    Objects.requireNonNull(agentId, "agentId is required");
  }
}
```

- [ ] **Step 4: Modify AgentWorkerFunction to add third construction path**

Change the record to three fields with mutual exclusivity validation:

```java
@SuppressWarnings("unchecked")
public record AgentWorkerFunction(WorkerAgentInvoker invoker, AgentSlotConfig slotConfig,
    AgentTaskConfig agentTaskConfig)
    implements WorkerFunction<Map<String, Object>, Map<String, Object>> {

  public AgentWorkerFunction {
    int count = (invoker != null ? 1 : 0)
        + (slotConfig != null ? 1 : 0)
        + (agentTaskConfig != null ? 1 : 0);
    if (count != 1) {
      throw new IllegalArgumentException(
          "Exactly one of invoker, slotConfig, agentTaskConfig must be non-null");
    }
  }

  public AgentWorkerFunction(WorkerAgentInvoker invoker) {
    this(Objects.requireNonNull(invoker), null, null);
  }

  public static AgentWorkerFunction eager(WorkerAgentInvoker invoker) {
    return new AgentWorkerFunction(Objects.requireNonNull(invoker), null, null);
  }

  public static AgentWorkerFunction ofSlot(AgentSlotConfig slotConfig) {
    return new AgentWorkerFunction(null, Objects.requireNonNull(slotConfig), null);
  }

  public static AgentWorkerFunction ofAgent(AgentTaskConfig config) {
    return new AgentWorkerFunction(null, null, Objects.requireNonNull(config));
  }

  public boolean isDeferred() {
    return slotConfig != null;
  }

  public boolean isAgentTask() {
    return agentTaskConfig != null;
  }

  @Override
  public Class<Map<String, Object>> inputType() {
    return (Class) Map.class;
  }

  @Override
  public Class<Map<String, Object>> outputType() {
    return (Class) Map.class;
  }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn test -pl api -Dtest=AgentWorkerFunctionTest -q`
Expected: all tests pass

- [ ] **Step 6: Compile full project to check for breakage**

Run: `mvn install -DskipTests -q`
Expected: clean build. The 2-arg `AgentWorkerFunction(invoker, slotConfig)` calls that existed previously will fail because the canonical constructor is now 3-arg. Find and fix any callers — the `AgentWorkerFunction(WorkerAgentInvoker)` convenience constructor should handle most cases; `ofSlot()` handles the rest.

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/api/model/ai/AgentTaskConfig.java api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java
git commit -m "feat(#1339): add AgentTaskConfig and AgentWorkerFunction.ofAgent() construction path

Refs #1339"
```

---

## Batch 2: Execution Handler — AgentWorkerFunctionHandler agentTask path

### Task 2: Extend AgentWorkerFunctionHandler for agentTask execution

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java:92-106`
- Modify: `runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java`

**Interfaces:**
- Consumes: `AgentTaskConfig.agentId()`, `AgentTaskConfig.inputMapping()`, `AgentTaskConfig.outputMapping()`, `AgentWorkerFunction.isAgentTask()`, `AgentRegistry.findById(String, String)`, `SystemPromptRenderer.render(AgentDescriptor, AgentPromptContext)`, `ExecutionMetadata.tenancyId()`
- Produces: `HandlerResult` from agent-task execution (same contract as `executeDeferred` and `executeWithInvoker`)

- [ ] **Step 1: Write failing test for agentTask execution path**

Add to `AgentWorkerFunctionHandlerTest.java`:

```java
@Test
void executeAgentTask_resolvesById_andInvokes() {
  var descriptor = AgentDescriptor.builder()
      .agentId("agent-123")
      .name("test-agent")
      .slot("test-slot")
      .tenancyId("tenant-1")
      .build();

  when(agentRegistry.findById("agent-123", "tenant-1"))
      .thenReturn(Optional.of(descriptor));
  when(systemPromptRenderer.render(eq(descriptor), any()))
      .thenReturn(new SystemPromptRenderer.RenderedPrompt("You are a test agent"));

  var config = new AgentTaskConfig("agent-123", null, null);
  var fn = AgentWorkerFunction.ofAgent(config);
  var worker = Worker.builder()
      .name("agent-task:test-binding")
      .function(fn)
      .capability(Capability.builder().name("test-cap").build())
      .build();
  var metadata = new ExecutionMetadata(UUID.randomUUID(), "tenant-1", null);

  HandlerResult result = handler.execute(worker,
      worker.capabilities().iterator().next(),
      Map.of("key", "value"), null, metadata);

  assertNotNull(result);
  verify(agentRegistry).findById("agent-123", "tenant-1");
  verify(systemPromptRenderer).render(eq(descriptor), any());
}

@Test
void executeAgentTask_agentNotFound_returnsFailed() {
  when(agentRegistry.findById("gone-agent", "tenant-1"))
      .thenReturn(Optional.empty());

  var config = new AgentTaskConfig("gone-agent", null, null);
  var fn = AgentWorkerFunction.ofAgent(config);
  var worker = Worker.builder()
      .name("agent-task:test-binding")
      .function(fn)
      .capability(Capability.builder().name("test-cap").build())
      .build();
  var metadata = new ExecutionMetadata(UUID.randomUUID(), "tenant-1", null);

  HandlerResult result = handler.execute(worker,
      worker.capabilities().iterator().next(),
      Map.of(), null, metadata);

  assertTrue(result.result().isFailed());
  assertThat(result.result().failureReason()).contains("Agent not found");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest -q`
Expected: FAIL — `executeAgentTask` method doesn't exist

- [ ] **Step 3: Implement executeAgentTask in AgentWorkerFunctionHandler**

Add dispatch in the `execute()` method after line 102:

```java
if (agent.isAgentTask()) {
  return executeAgentTask(agent.agentTaskConfig(), worker, inputData, metadata);
}
```

Add the new method:

```java
private HandlerResult executeAgentTask(
    AgentTaskConfig config, Worker worker, Object inputData, ExecutionMetadata metadata) {

  String tenancyId = metadata.tenancyId();
  java.util.Optional<AgentDescriptor> opt =
      agentRegistry.findById(config.agentId(), tenancyId);
  if (opt.isEmpty()) {
    return new HandlerResult(
        WorkerResult.failed("Agent not found: " + config.agentId()));
  }

  AgentDescriptor descriptor = opt.get();
  SystemPromptRenderer.RenderedPrompt rendered =
      systemPromptRenderer.render(
          descriptor, AgentPromptContext.forFormat(SystemPromptRenderer.RenderFormat.MARKDOWN));

  UnaryOperator<com.fasterxml.jackson.databind.JsonNode> inputTransformer =
      config.inputMapping() != null
          ? node -> config.inputMapping().apply(node)
          : UnaryOperator.identity();
  UnaryOperator<com.fasterxml.jackson.databind.JsonNode> outputTransformer =
      config.outputMapping() != null
          ? node -> config.outputMapping().apply(node)
          : UnaryOperator.identity();

  WorkerAgentInvoker invoker =
      new WorkerAgentInvoker(
          agentProvider,
          rendered.content(),
          null,
          inputTransformer,
          outputTransformer,
          null,
          null,
          null);

  return executeWithInvoker(invoker, worker, inputData);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl runtime -Dtest=AgentWorkerFunctionHandlerTest -q`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java
git commit -m "feat(#1339): add agentTask execution path to AgentWorkerFunctionHandler

Refs #1339"
```

---

## Batch 3: Dispatch Handler + Constructor Wiring + Switch Sites

### Task 3: Add publishAgentTaskSchedule() to CaseContextChangedEventHandler and wire constructor

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java:99-200` (fields + constructor), `:503-532` (publishByTarget switch), new method after `:844`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java:884-960` (CDI producer)
- Modify: `runtime-spring/src/main/java/io/casehub/engine/runtime/spring/RuntimeManualConfig.java:207-290` (Spring config)

**Interfaces:**
- Consumes: `AgentRegistry.find(AgentQuery)`, `AgentMatch.descriptor()`, `AgentDescriptor.agentId()`, `AgentTaskTarget.capability()`, `AgentTaskTarget.domain()`, `AgentTaskTarget.inputMapping()`, `AgentTaskTarget.outputMapping()`, `AgentWorkerFunction.ofAgent(AgentTaskConfig)`, `WorkerScheduleEvent`
- Produces: `WorkerScheduleEvent` dispatched via `eventDispatcher.dispatch()` with synthetic Worker

- [ ] **Step 1: Add AgentRegistry field and constructor parameter to CaseContextChangedEventHandler**

Add field after line 136 (after `activityTracker`):

```java
private final io.casehub.eidos.api.AgentRegistry agentRegistry;
```

Add parameter to the constructor and assign it. Add import for `io.casehub.eidos.api.AgentRegistry`.

- [ ] **Step 2: Add AgentTaskTarget case to publishByTarget() switch**

In the `publishByTarget()` method (around line 503), the switch on `binding.target()` currently ends with `HumanTaskTarget`. Before the closing brace, the compiler may already require `AgentTaskTarget` handling. Add:

```java
case AgentTaskTarget at ->
    publishAgentTaskSchedule(caseInstance, caseDefinition, binding, at);
```

Add import for `io.casehub.api.model.AgentTaskTarget`.

- [ ] **Step 3: Implement publishAgentTaskSchedule()**

Add after `publishJudgmentSchedule()` (around line 940):

```java
private void publishAgentTaskSchedule(
    final CaseInstance caseInstance,
    final CaseDefinition caseDefinition,
    final Binding binding,
    final AgentTaskTarget target) {

  java.util.List<io.casehub.eidos.api.AgentMatch> matches =
      agentRegistry.find(
          io.casehub.eidos.api.AgentQuery.byCapabilityAndDomain(
              target.capability(), target.domain(), caseInstance.tenancyId));

  if (matches.isEmpty()) {
    LOG.warnf(
        "No agent found for capability '%s' domain '%s' binding '%s' caseId=%s",
        target.capability(), target.domain(), binding.getName(), caseInstance.getUuid());
    return;
  }

  io.casehub.eidos.api.AgentDescriptor descriptor = matches.get(0).descriptor();
  var config = new io.casehub.api.model.ai.AgentTaskConfig(
      descriptor.agentId(), target.inputMapping(), target.outputMapping());

  io.casehub.worker.api.Capability syntheticCapability =
      io.casehub.worker.api.Capability.builder()
          .name(target.capability())
          .build();

  io.casehub.worker.api.Worker syntheticWorker =
      io.casehub.worker.api.Worker.builder()
          .name("agent-task:" + binding.getName())
          .function(AgentWorkerFunction.ofAgent(config))
          .capability(syntheticCapability)
          .build();

  quiescenceTracker.onWorkerDispatched(caseInstance.getUuid());

  eventDispatcher.dispatch(
      new io.casehub.engine.common.internal.event.WorkerScheduleEvent(
          caseInstance,
          syntheticWorker,
          syntheticCapability,
          binding.getName(),
          null,
          null,
          io.casehub.api.model.event.ExecutionOrigin.BINDING_DISPATCH,
          java.util.List.of(),
          null,
          null,
          null,
          null,
          null));
}
```

- [ ] **Step 4: Update RuntimeBeans CDI producer**

In `RuntimeBeans.caseContextChangedEventHandler()` (around line 884), add `AgentRegistry` parameter and pass it to the constructor:

```java
io.casehub.eidos.api.AgentRegistry agentRegistry,
```

Pass `agentRegistry` as the new constructor argument in the `new CaseContextChangedEventHandler(...)` call.

- [ ] **Step 5: Update RuntimeManualConfig Spring config**

In `RuntimeManualConfig.caseContextChangedEventHandler()` (around line 207), add `AgentRegistry` parameter and pass it to the constructor.

- [ ] **Step 6: Compile to verify wiring**

Run: `mvn install -DskipTests -q`
Expected: clean build

- [ ] **Step 7: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java runtime-spring/src/main/java/io/casehub/engine/runtime/spring/RuntimeManualConfig.java
git commit -m "feat(#1339): add AgentTaskTarget dispatch handler and constructor wiring

Refs #1339"
```

### Task 4: Switch-site updates (compiler-guided)

**Files:**
- Modify: `runtime-core/src/main/java/io/casehub/engine/runtime/scheduler/SchedulerService.java:78-103` and `:209-228`
- Modify: `scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java:366-385`
- Modify: `runtime/src/main/java/io/casehub/engine/runtime/memory/CbrCaseRetainObserver.java:325-333`
- Modify: `planning-core/src/main/java/io/casehub/engine/planning/snapshot/PlanningCasePlanModelSnapshotProvider.java:171-184`

**Interfaces:**
- Consumes: `AgentTaskTarget` (sealed permit on `BindingTarget`)
- Produces: no new interfaces — these are exhaustive-switch completions

- [ ] **Step 1: Add AgentTaskTarget to SchedulerService.registerScheduledTriggers()**

In the switch at line 78, add before the closing brace:

```java
case AgentTaskTarget ignored ->
    LOG.warnf("Schedule binding '%s' has AgentTask target — skipping", binding.getName());
```

- [ ] **Step 2: Add AgentTaskTarget to SchedulerService.createJobData()**

In the switch at line 209, add:

```java
case io.casehub.api.model.AgentTaskTarget ignored ->
    throw new IllegalStateException(
        "createJobData called with AgentTaskTarget binding '" + binding.getName() + "'");
```

- [ ] **Step 3: Add AgentTaskTarget to QuartzWorkerExecutionManager.createTriggerJobData()**

In the switch at line 366, add:

```java
case io.casehub.api.model.AgentTaskTarget ignored ->
    throw new IllegalStateException(
        "Schedule-triggered binding '" + binding.getName() + "' must target a Capability");
```

- [ ] **Step 4: Add AgentTaskTarget to CbrCaseRetainObserver.buildRoutingKeyMap()**

In the switch at line 325, add within the existing `default` arm or replace it:

```java
case io.casehub.api.model.AgentTaskTarget at ->
    map.put(binding.getName(), at.capability());
```

- [ ] **Step 5: Add AgentTaskTarget to PlanningCasePlanModelSnapshotProvider.mapTargetType()**

In the switch at line 175, add:

```java
case AgentTaskTarget at -> "WORKER";
```

- [ ] **Step 6: Compile to verify all switch sites are covered**

Run: `mvn install -DskipTests -q`
Expected: clean build — no remaining exhaustive switch errors

- [ ] **Step 7: Run full test suite to check for regressions**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: all tests pass

- [ ] **Step 8: Commit**

```bash
git add runtime-core/src/main/java/io/casehub/engine/runtime/scheduler/SchedulerService.java scheduler-quartz/src/main/java/io/casehub/engine/scheduler/quartz/QuartzWorkerExecutionManager.java runtime/src/main/java/io/casehub/engine/runtime/memory/CbrCaseRetainObserver.java planning-core/src/main/java/io/casehub/engine/planning/snapshot/PlanningCasePlanModelSnapshotProvider.java
git commit -m "feat(#1339): add AgentTaskTarget to all exhaustive BindingTarget switches

Refs #1339"
```

---

## Batch 4: Integration Test — End-to-End Dispatch

### Task 5: Integration test for agent-task dispatch flow

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/AgentTaskTargetDispatchTest.java`

**Interfaces:**
- Consumes: `CaseHub`, `CaseDefinition.builder()`, `Binding.builder()`, `AgentTaskTarget.builder()`, `ContextChangeTrigger`, `AgentRegistry`, `AgentDescriptor`, `WorkerScheduleEvent`
- Produces: verified end-to-end dispatch test

- [ ] **Step 1: Write integration test**

Model after `JudgmentTargetDispatchTest.java`. Create a `@QuarkusTest` that:
1. Defines a `CaseHub` subclass with an `AgentTaskTarget` binding
2. Provides a test `AgentRegistry` with a registered agent
3. Starts a case with context data that triggers the binding
4. Verifies the `WorkerScheduleEvent` was dispatched with correct synthetic Worker

```java
package io.casehub.engine;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.api.engine.CaseHub;
import io.casehub.api.model.AgentTaskTarget;
import io.casehub.api.model.Binding;
import io.casehub.api.model.CaseDefinition;
import io.casehub.api.model.ContextChangeTrigger;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentMatch;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.ResolvedCapability;
import io.casehub.engine.common.internal.event.WorkerScheduleEvent;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

@QuarkusTest
class AgentTaskTargetDispatchTest {

  @Inject AgentTaskCaseBean caseBean;

  @BeforeEach
  void reset() {
    RecordingWorkerScheduleObserver.events.clear();
  }

  @Test
  void agentTaskBinding_dispatchesSyntheticWorker_withCorrectAgentId() {
    UUID caseId = caseBean.startCase(Map.of("ticket", Map.of("text", "hello")));

    await()
        .atMost(5, TimeUnit.SECONDS)
        .untilAsserted(
            () -> assertThat(RecordingWorkerScheduleObserver.events).isNotEmpty());

    WorkerScheduleEvent event = RecordingWorkerScheduleObserver.events.get(0);
    assertThat(event.worker().name()).isEqualTo("agent-task:analyse-ticket");
    assertThat(event.capability().name()).isEqualTo("sentiment-analysis");
    assertThat(event.bindingName()).isEqualTo("analyse-ticket");

    var fn = (io.casehub.api.model.AgentWorkerFunction) event.worker().function();
    assertThat(fn.isAgentTask()).isTrue();
    assertThat(fn.agentTaskConfig().agentId()).isEqualTo("sentiment-agent-1");
  }

  @ApplicationScoped
  static class AgentTaskCaseBean extends CaseHub {

    @Override
    public CaseDefinition getDefinition() {
      return CaseDefinition.builder()
          .namespace("test")
          .name("agent-task-dispatch-test")
          .version("1.0.0")
          .bindings(
              Binding.builder()
                  .name("analyse-ticket")
                  .target(
                      AgentTaskTarget.builder()
                          .capability("sentiment-analysis")
                          .domain("support")
                          .inputMapping(".ticket")
                          .build())
                  .on(new ContextChangeTrigger(".ticket != null"))
                  .build())
          .build();
    }
  }

  @Alternative
  @jakarta.annotation.Priority(1)
  @ApplicationScoped
  static class TestAgentRegistry implements AgentRegistry {

    @Override
    public void register(AgentDescriptor descriptor) {}

    @Override
    public Optional<AgentDescriptor> findById(String agentId, String tenancyId) {
      if ("sentiment-agent-1".equals(agentId)) {
        return Optional.of(testDescriptor());
      }
      return Optional.empty();
    }

    @Override
    public List<AgentMatch> find(AgentQuery query) {
      if ("sentiment-analysis".equals(query.capabilityName())) {
        return List.of(new AgentMatch(testDescriptor(), null));
      }
      return List.of();
    }

    private AgentDescriptor testDescriptor() {
      return AgentDescriptor.builder()
          .agentId("sentiment-agent-1")
          .name("Sentiment Analyst")
          .slot("sentiment-analyst")
          .tenancyId("test")
          .build();
    }
  }
}
```

Note: `RecordingWorkerScheduleObserver` may already exist in the test infrastructure. If not, create a simple CDI observer that captures `WorkerScheduleEvent` instances to a static list. Check existing test helpers first.

- [ ] **Step 2: Run the integration test**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=AgentTaskTargetDispatchTest -q`
Expected: PASS — the synthetic worker event is dispatched with the correct agent ID and capability

- [ ] **Step 3: Run full test suite**

Run: `TESTCONTAINERS_RYUK_DISABLED=true mvn clean test -q`
Expected: all tests pass, no regressions

- [ ] **Step 4: Commit**

```bash
git add runtime/src/test/java/io/casehub/engine/AgentTaskTargetDispatchTest.java
git commit -m "feat(#1339): add integration test for AgentTaskTarget dispatch flow

Closes #1339"
```

## References

- [2026-11-22-agent-task-runtime-dispatch-design.md] — design spec this plan implements
- `api/src/main/java/io/casehub/api/model/AgentTaskTarget.java` — binding target type
- `api/src/main/java/io/casehub/api/model/AgentWorkerFunction.java:25` — existing record
- `api/src/main/java/io/casehub/api/model/ai/AgentSlotConfig.java:20` — parallel config record
- `runtime/src/main/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandler.java:63` — execution handler
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java:503` — dispatch switch
- `runtime-core/src/main/java/io/casehub/engine/runtime/engine/handler/CaseContextChangedEventHandler.java:846` — judgment dispatch pattern
- `runtime/src/main/java/io/casehub/engine/runtime/quarkus/RuntimeBeans.java:884` — CDI producer
- `runtime-spring/src/main/java/io/casehub/engine/runtime/spring/RuntimeManualConfig.java:207` — Spring config
- `runtime/src/test/java/io/casehub/engine/JudgmentTargetDispatchTest.java` — integration test pattern
- `api/src/test/java/io/casehub/api/model/AgentWorkerFunctionTest.java` — existing unit test
- `runtime/src/test/java/io/casehub/engine/runtime/executor/AgentWorkerFunctionHandlerTest.java` — existing handler test
- casehubio/engine#1339 — focal issue
- casehubio/engine#1333 — parent issue
- PP-20260921-b7c277 — no @DefaultBean on multi-instance SPIs
- PP-20260727-5267d2 — plan-type module boundary
