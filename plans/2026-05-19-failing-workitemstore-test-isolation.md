# FailingWorkItemStore Test Isolation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remove `FailingWorkItemStore` from the global `selected-alternatives` list and scope it exclusively to `HumanTaskScheduleHandlerAtomicityTest` via a `@QuarkusTestProfile`.

**Architecture:** Move the failing-store inner class and atomicity test method out of `HumanTaskScheduleHandlerTest` into a new `HumanTaskScheduleHandlerAtomicityTest` that activates the store through its own `QuarkusTestProfile`. The original test class reverts to the default profile with `InMemoryWorkItemStore` active. One application restart occurs when JUnit switches between the two profiles; all other test classes are unaffected.

**Tech Stack:** Java 21, Quarkus 3.x, JUnit 5, `io.quarkus.test.junit.QuarkusTestProfile`, AssertJ.

---

## File Map

| Action | Path |
|--------|------|
| Modify | `work-adapter/src/test/resources/application.properties` |
| Modify | `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java` |
| Create | `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java` |

---

### Task 1: Create `HumanTaskScheduleHandlerAtomicityTest`

The new class is self-contained: it owns `FailingWorkItemStore`, `Profile`, and the single atomicity test. Write it first — while `FailingWorkItemStore` still exists in both places the module compiles and both test classes can be verified independently.

**Files:**
- Create: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java`

- [ ] **Step 1: Create the test class**

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

import io.casehub.api.model.HumanTaskTarget;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.internal.event.EventBusAddresses;
import io.casehub.engine.internal.event.HumanTaskScheduleEvent;
import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import io.casehub.persistence.memory.MemoryPlanItemStore;
import io.casehub.work.runtime.repository.WorkItemQuery;
import io.casehub.work.runtime.repository.WorkItemStore;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;
import io.vertx.mutiny.core.eventbus.EventBus;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

/**
 * Verifies that a WorkItem creation failure leaves PlanItem PENDING and does not write a RUNNING
 * status to the store (atomicity guarantee, engine#273). Runs under its own QuarkusTestProfile so
 * FailingWorkItemStore is activated only for this class.
 *
 * <p>Refs engine#282, engine#273.
 */
@QuarkusTest
@TestProfile(HumanTaskScheduleHandlerAtomicityTest.Profile.class)
class HumanTaskScheduleHandlerAtomicityTest {

  public static class Profile implements QuarkusTestProfile {
    @Override
    public Set<Class<?>> getEnabledAlternatives() {
      return Set.of(FailingWorkItemStore.class);
    }
  }

  /**
   * WorkItemStore substitute that throws on put() when shouldFail is true. Activated only via
   * Profile — not listed in the global selected-alternatives.
   */
  @ApplicationScoped
  @Alternative
  @Priority(2)
  public static class FailingWorkItemStore implements WorkItemStore {

    public static final AtomicBoolean shouldFail = new AtomicBoolean(false);

    private final java.util.Map<UUID, io.casehub.work.runtime.model.WorkItem> store =
        new ConcurrentHashMap<>();

    public void clear() {
      store.clear();
    }

    @Override
    public io.casehub.work.runtime.model.WorkItem put(io.casehub.work.runtime.model.WorkItem w) {
      if (shouldFail.get()) throw new RuntimeException("Simulated WorkItemStore failure");
      if (w.id == null) w.id = UUID.randomUUID();
      store.put(w.id, w);
      return w;
    }

    @Override
    public Optional<io.casehub.work.runtime.model.WorkItem> get(UUID id) {
      return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<io.casehub.work.runtime.model.WorkItem> scan(WorkItemQuery query) {
      return new ArrayList<>(store.values());
    }
  }

  @Inject BlackboardRegistry registry;
  @Inject EventBus eventBus;
  @Inject WorkItemStore workItemStore;
  @Inject PlanItemStore planItemStore;

  private UUID caseId;
  private PlanItem planItem;

  @BeforeEach
  void setUp() {
    if (workItemStore instanceof FailingWorkItemStore failing) {
      failing.clear();
      FailingWorkItemStore.shouldFail.set(false);
    }
    if (planItemStore instanceof MemoryPlanItemStore mem) {
      mem.clear();
    }
    caseId = UUID.randomUUID();
    planItem = PlanItem.create("irb-binding", "unused-worker", 5);
    registry.getOrCreate(caseId).addPlanItem(planItem);
  }

  @Test
  void inlineMode_workItemCreationFails_planItemStaysPending_storeNotUpdated() {
    HumanTaskTarget target = HumanTaskTarget.inline().title("Review").build();

    FailingWorkItemStore.shouldFail.set(true);
    try {
      eventBus.publish(
          EventBusAddresses.HUMAN_TASK_SCHEDULE,
          new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of()));

      try {
        Thread.sleep(500);
      } catch (InterruptedException ignored) {
      }

      assertThat(planItem.getStatus()).isEqualTo(PlanItemStatus.PENDING);
      assertThat(workItemStore.scanAll()).isEmpty();
      List<PlanItemRecord> records = planItemStore.findByCaseId(caseId);
      assertThat(records)
          .noneMatch(
              r ->
                  r.planItemId().equals(planItem.getPlanItemId())
                      && r.status() == PlanItemStatus.RUNNING);
    } finally {
      FailingWorkItemStore.shouldFail.set(false);
    }
  }
}
```

- [ ] **Step 2: Verify the module compiles with both classes present**

```bash
mvn test-compile -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml
```

Expected: `BUILD SUCCESS`. Both `HumanTaskScheduleHandlerTest` (still has `FailingWorkItemStore`) and the new class coexist cleanly.

---

### Task 2: Remove `FailingWorkItemStore` from `HumanTaskScheduleHandlerTest` and global alternatives

Now that the atomicity class owns `FailingWorkItemStore`, strip it from the original class and from `application.properties`.

**Files:**
- Modify: `work-adapter/src/test/resources/application.properties`
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`

- [ ] **Step 1: Remove from `application.properties`**

In `work-adapter/src/test/resources/application.properties`, change the `selected-alternatives` block from:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.persistence.memory.InMemoryCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryEventLogRepository,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository,\
  io.casehub.persistence.memory.MemoryPlanItemStore,\
  io.casehub.workadapter.NoOpLedgerEntryRepository,\
  io.casehub.workadapter.NoOpReactiveLedgerEntryRepository,\
  io.casehub.workadapter.HumanTaskScheduleHandlerTest$FailingWorkItemStore
```

to:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.persistence.memory.InMemoryCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryEventLogRepository,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository,\
  io.casehub.persistence.memory.MemoryPlanItemStore,\
  io.casehub.workadapter.NoOpLedgerEntryRepository,\
  io.casehub.workadapter.NoOpReactiveLedgerEntryRepository
```

- [ ] **Step 2: Remove `FailingWorkItemStore` inner class from `HumanTaskScheduleHandlerTest`**

Delete lines 67–99 (the entire `FailingWorkItemStore` static inner class with its Javadoc).

- [ ] **Step 3: Remove the atomicity test method from `HumanTaskScheduleHandlerTest`**

Delete the `inlineMode_workItemCreationFails_planItemStaysPending_storeNotUpdated` test method (lines 327–353).

- [ ] **Step 4: Simplify `setUp()` in `HumanTaskScheduleHandlerTest`**

Remove the dead `instanceof FailingWorkItemStore` arm. The `setUp()` becomes:

```java
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
```

- [ ] **Step 5: Remove unused imports from `HumanTaskScheduleHandlerTest`**

Remove these imports (no longer referenced after the inner class and test are gone):

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;
```

Also remove `import io.casehub.engine.internal.model.PlanItemRecord;` if it was only used in the deleted test. Check: `PlanItemRecord` was used only in the atomicity test — remove it.

- [ ] **Step 6: Verify compilation**

```bash
mvn test-compile -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7: Full install (skip tests — Docker not available locally)**

```bash
mvn install -DskipTests -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml
```

Expected: `BUILD SUCCESS`.

---

### Task 3: Commit

- [ ] **Step 1: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  work-adapter/src/test/resources/application.properties \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java

git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
test(work-adapter): scope FailingWorkItemStore to atomicity test via QuarkusTestProfile (engine#282)

FailingWorkItemStore was registered globally in selected-alternatives,
making it the active WorkItemStore for every @QuarkusTest class in the
module. Move it to HumanTaskScheduleHandlerAtomicityTest with its own
QuarkusTestProfile.getEnabledAlternatives(), where it activates only for
the single test that needs it. HumanTaskScheduleHandlerTest reverts to the
default profile with InMemoryWorkItemStore active.

Closes #282
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ Remove from global `selected-alternatives` → Task 2 Step 1
- ✅ Simplify `HumanTaskScheduleHandlerTest.setUp()` → Task 2 Steps 4–5
- ✅ Create `HumanTaskScheduleHandlerAtomicityTest` with `Profile`, `FailingWorkItemStore`, atomicity test → Task 1
- ✅ Remove `FailingWorkItemStore` from `HumanTaskScheduleHandlerTest` → Task 2 Steps 2–3

**Placeholder scan:** None found.

**Type consistency:** `FailingWorkItemStore`, `Profile`, `shouldFail`, `clear()`, `setUp()` used consistently across tasks.
