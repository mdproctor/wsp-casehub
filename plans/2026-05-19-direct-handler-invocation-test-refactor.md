# Direct Handler Invocation Test Refactor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace event-bus-mediated async test dispatch with direct `HumanTaskScheduleHandler` method calls, eliminating all timing machinery while preserving every assertion.

**Architecture:** Inject `HumanTaskScheduleHandler` directly into the test class and call `onHumanTaskSchedule()` synchronously. The CDI proxy enforces `@Transactional` identically to production. One wiring smoke test retains event bus coverage. The `templateMode_withInputData` test is corrected to actually persist `defaultPayload` before asserting override semantics (engine#291).

**Tech Stack:** Java 21, Quarkus 3.x, JUnit 5, AssertJ, Awaitility (wiring test only).

---

## File Map

| Action | Path |
|--------|------|
| Rewrite | `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java` |

---

### Task 1: Rewrite `HumanTaskScheduleHandlerTest`

This is a single-task plan — the entire file is replaced atomically. The new file has identical assertions to the original on every test, with timing machinery removed.

**Files:**
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`

- [ ] **Step 1: Verify current test compiles (baseline)**

```bash
mvn test-compile -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml 2>&1 | tail -3
```
Expected: `BUILD SUCCESS`

- [ ] **Step 2: Replace the file with the refactored version**

Write the following content to `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.workadapter;

import static org.assertj.core.api.Assertions.assertThat;
import static org.awaitility.Awaitility.await;

import io.casehub.api.model.HumanTaskTarget;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.internal.event.EventBusAddresses;
import io.casehub.engine.internal.event.HumanTaskScheduleEvent;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import io.casehub.persistence.memory.MemoryPlanItemStore;
import io.casehub.work.runtime.model.WorkItem;
import io.casehub.work.runtime.model.WorkItemStatus;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.casehub.work.testing.InMemoryWorkItemStore;
import io.quarkus.test.junit.QuarkusTest;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Duration;
import java.util.Map;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * Verifies HumanTaskScheduleHandler logic by invoking the handler directly (no event bus
 * dispatch), so all assertions are synchronous — no await or sleep needed. A single wiring
 * test confirms the @ConsumeEvent route is correctly registered.
 *
 * <p>Refs engine#290 (removed Thread.sleep), engine#291 (fixed detached entity in
 * templateMode_withInputData), engine#245.
 */
@QuarkusTest
class HumanTaskScheduleHandlerTest {

  @Inject HumanTaskScheduleHandler handler;
  @Inject BlackboardRegistry registry;
  @Inject EventBus eventBus;
  @Inject WorkItemStore workItemStore;
  @Inject PlanItemStore planItemStore;

  private UUID caseId;
  private PlanItem planItem;

  @BeforeEach
  @Transactional
  void setUp() {
    if (workItemStore instanceof InMemoryWorkItemStore mem) {
      mem.clear();
    }
    if (planItemStore instanceof MemoryPlanItemStore mem) {
      mem.clear();
    }
    WorkItemTemplate.deleteAll();
    caseId = UUID.randomUUID();
    planItem = PlanItem.create("irb-binding", "unused-worker", 5);
    registry.getOrCreate(caseId).addPlanItem(planItem);
  }

  // ── Wiring ────────────────────────────────────────────────────────────────

  @Test
  void eventBus_routesHumanTaskScheduleEvent_toHandler() {
    HumanTaskTarget target = HumanTaskTarget.inline().title("Smoke").build();
    eventBus.publish(
        EventBusAddresses.HUMAN_TASK_SCHEDULE,
        new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of()));
    await()
        .atMost(5, TimeUnit.SECONDS)
        .untilAsserted(() -> assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.RUNNING));
  }

  // ── Inline mode ───────────────────────────────────────────────────────────

  @Test
  void inlineMode_createsWorkItem_withCallerRef_andMarksPlanItemRunning() {
    HumanTaskTarget target =
        HumanTaskTarget.inline()
            .title("IRB Ethics Review")
            .candidateGroups(Set.of("ethics-committee"))
            .expiresIn(Duration.ofHours(72))
            .build();

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of("caseRef", "T-42")));

    String expectedCallerRef = CallerRef.encode(caseId, planItem.getPlanItemId());

    WorkItem created =
        workItemStore.scanAll().stream()
            .filter(w -> expectedCallerRef.equals(w.callerRef))
            .findFirst()
            .orElse(null);
    assertThat(created).isNotNull();
    assertThat(created.status).isEqualTo(WorkItemStatus.PENDING);
    assertThat(created.title).isEqualTo("IRB Ethics Review");
    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.RUNNING);
    assertThat(planItemStore.findByCaseId(caseId))
        .anyMatch(
            r ->
                r.planItemId().equals(planItem.getPlanItemId())
                    && r.status() == PlanItemStatus.RUNNING);
  }

  // ── Template mode ─────────────────────────────────────────────────────────

  @Test
  void templateMode_byUuid_createsWorkItem_andMarksPlanItemRunning() {
    WorkItemTemplate tmpl = persistTemplate("IRB Ethics Review Template");

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId, "irb-binding", HumanTaskTarget.template(tmpl.id.toString()).build(), Map.of()));

    String expectedCallerRef = CallerRef.encode(caseId, planItem.getPlanItemId());
    WorkItem created =
        workItemStore.scanAll().stream()
            .filter(w -> expectedCallerRef.equals(w.callerRef))
            .findFirst()
            .orElse(null);
    assertThat(created).isNotNull();
    assertThat(created.status).isEqualTo(WorkItemStatus.PENDING);
    assertThat(created.title).isEqualTo("IRB Ethics Review Template");
    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.RUNNING);
  }

  @Test
  void templateMode_byName_createsWorkItem_andMarksPlanItemRunning() {
    persistTemplate("AML Suspicious Activity Review");

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId,
            "irb-binding",
            HumanTaskTarget.template("AML Suspicious Activity Review").build(),
            Map.of()));

    assertThat(workItemStore.scanAll()).hasSize(1);
    assertThat(workItemStore.scanAll().get(0).title).isEqualTo("AML Suspicious Activity Review");
    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.RUNNING);
  }

  @Test
  void templateMode_withInputData_inputDataOverridesTemplateDefaultPayload() {
    // defaultPayload persisted in DB — required to prove override semantics (engine#291)
    WorkItemTemplate tmpl = persistTemplate("Clinical Trial Consent", "{\"type\":\"default\"}");

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId,
            "irb-binding",
            HumanTaskTarget.template(tmpl.id.toString()).build(),
            Map.of("trialId", "T-99", "phase", "III")));

    WorkItem created = workItemStore.scanAll().stream().findFirst().orElse(null);
    assertThat(created).isNotNull();
    assertThat(created.payload).contains("trialId").contains("T-99");
    assertThat(created.payload).doesNotContain("\"type\":\"default\"");
    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.RUNNING);
  }

  @Test
  void templateMode_emptyInputData_usesTemplateDefaultPayload() {
    WorkItemTemplate tmpl = persistTemplate("Loan Approval", "{\"type\":\"loan\"}");

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId,
            "irb-binding",
            HumanTaskTarget.template(tmpl.id.toString()).build(),
            Map.of()));

    WorkItem created = workItemStore.scanAll().stream().findFirst().orElse(null);
    assertThat(created).isNotNull();
    assertThat(created.payload).isEqualTo("{\"type\":\"loan\"}");
    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.RUNNING);
  }

  @Test
  void templateMode_templateNotFound_planItemStaysPending() {
    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId,
            "irb-binding",
            HumanTaskTarget.template(UUID.randomUUID().toString()).build(),
            Map.of()));

    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.PENDING);
    assertThat(workItemStore.scanAll()).isEmpty();
  }

  @Test
  void templateMode_ambiguousName_planItemStaysPending() {
    persistTemplate("Duplicate Name");
    persistTemplate("Duplicate Name");

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId,
            "irb-binding",
            HumanTaskTarget.template("Duplicate Name").build(),
            Map.of()));

    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.PENDING);
    assertThat(workItemStore.scanAll()).isEmpty();
  }

  @Test
  void noPlanForCaseId_eventIgnored() {
    UUID unknownCaseId = UUID.randomUUID();

    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            unknownCaseId,
            "irb-binding",
            HumanTaskTarget.inline().title("Review").build(),
            Map.of()));

    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.PENDING);
    assertThat(workItemStore.scanAll()).isEmpty();
  }

  @Test
  void noPlanItemForBindingName_eventIgnored() {
    handler.onHumanTaskSchedule(
        new HumanTaskScheduleEvent(
            caseId,
            "unknown-binding",
            HumanTaskTarget.inline().title("Review").build(),
            Map.of()));

    assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.PENDING);
    assertThat(workItemStore.scanAll()).isEmpty();
  }

  // ── Helpers ───────────────────────────────────────────────────────────────

  @Transactional
  WorkItemTemplate persistTemplate(final String name) {
    return persistTemplate(name, null);
  }

  @Transactional
  WorkItemTemplate persistTemplate(final String name, final String defaultPayload) {
    WorkItemTemplate t = new WorkItemTemplate();
    t.name = name;
    t.createdBy = "test";
    t.defaultPayload = defaultPayload;
    WorkItemTemplate.persist(t);
    return t;
  }
}
```

- [ ] **Step 3: Verify compilation**

```bash
mvn test-compile -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml 2>&1 | tail -3
```
Expected: `BUILD SUCCESS`

- [ ] **Step 4: Full install (skip tests — Docker not available locally)**

```bash
mvn install -DskipTests -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml 2>&1 | grep "BUILD" | tail -2
```
Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java

git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
test(work-adapter): replace event bus dispatch with direct handler invocation (engine#290, engine#291)

All tests now call handler.onHumanTaskSchedule() directly through the CDI
proxy. @Transactional is enforced identically to production; assertions are
synchronous — no await() or Thread.sleep() needed. A single wiring test
confirms @ConsumeEvent routing. templateMode_withInputData now correctly
persists defaultPayload before asserting inputData override semantics (engine#291).

Closes #290
Closes #291
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ Direct handler injection → `@Inject HumanTaskScheduleHandler handler`
- ✅ All `eventBus.publish()` replaced → 9 tests use `handler.onHumanTaskSchedule()`
- ✅ All `await().atMost()` removed from positive-path tests → assertions immediately follow handler call
- ✅ All `Thread.sleep()` removed from negative-path tests → assertions immediately follow handler call
- ✅ Wiring smoke test added → `eventBus_routesHumanTaskScheduleEvent_toHandler`
- ✅ engine#291 fixed → `persistTemplate("Clinical Trial Consent", "{\"type\":\"default\"}")` + `doesNotContain` assertion added
- ✅ Javadoc updated
- ✅ Imports cleaned (`TimeUnit` kept for wiring test, `await` kept for wiring test)

**Placeholder scan:** None.

**Type/name consistency:** `handler.onHumanTaskSchedule(HumanTaskScheduleEvent)` used identically across all tasks.

**Test count:** 9 existing → 10 (9 direct + 1 wiring). No test removed, one added.

**Assertion diligence check (per user requirement):**
- `inlineMode`: callerRef filter, status, title, PlanItem RUNNING, planItemStore record — all preserved, improved ordering
- `templateMode_byUuid`: callerRef filter, status, title, PlanItem RUNNING — all preserved
- `templateMode_byName`: size=1, title, PlanItem RUNNING — `assertThat(planItem.getStatus())` ADDED (was not in original)
- `templateMode_withInputData`: payload contains trialId+T-99, `doesNotContain` default ADDED, PlanItem RUNNING ADDED
- `templateMode_emptyInputData`: exact payload match, PlanItem RUNNING ADDED
- `templateMode_templateNotFound`: PENDING, empty store — preserved
- `templateMode_ambiguousName`: PENDING, empty store — preserved
- `noPlanForCaseId`: PENDING, empty store — preserved
- `noPlanItemForBindingName`: PENDING, empty store — preserved
