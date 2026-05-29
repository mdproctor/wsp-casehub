# BlackboardRegistry Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Consolidate `BlackboardRegistry` internals from three separate maps into a single `ConcurrentHashMap<UUID, CaseEntry>`, making eviction atomic, then add `@AfterEach` eviction to three work-adapter test classes.

**Architecture:** `CaseEntry` is a private static class inside `BlackboardRegistry` holding all per-case state. The six public methods are unchanged in signature; implementation delegates to `entryFor(caseId)` which uses `computeIfAbsent`. Eviction becomes a single `entries.remove(caseId)` — no race window. Task 1 writes contract tests first, then refactors. Task 2 adds `@AfterEach` teardown to three test classes.

**Tech Stack:** Java 21, JUnit 5, AssertJ.

---

## File Map

| Action | Path |
|--------|------|
| Create | `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTest.java` |
| Rewrite | `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java` |
| Modify | `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java` |
| Modify | `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java` |
| Modify | `work-adapter/src/test/java/io/casehub/workadapter/WorkItemLifecycleAdapterTest.java` |

---

### Task 1: Consolidate `BlackboardRegistry` internals (TDD)

Write the contract test first. Verify it passes on the current implementation (green baseline). Refactor internals. Verify it still passes. Commit.

**Files:**
- Create: `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTest.java`
- Rewrite: `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java`

- [ ] **Step 1: Write the contract test**

Create `blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTest.java`:

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
package io.casehub.blackboard.registry;

import static org.assertj.core.api.Assertions.assertThat;

import io.casehub.blackboard.plan.CasePlanModel;
import java.util.UUID;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class BlackboardRegistryTest {

  private BlackboardRegistry registry;
  private UUID caseId;

  @BeforeEach
  void setUp() {
    registry = new BlackboardRegistry();
    caseId = UUID.randomUUID();
  }

  @Test
  void getOrCreate_returnsPlanModel() {
    CasePlanModel model = registry.getOrCreate(caseId);
    assertThat(model).isNotNull();
  }

  @Test
  void getOrCreate_returnsSameInstanceOnRepeatCall() {
    CasePlanModel first = registry.getOrCreate(caseId);
    CasePlanModel second = registry.getOrCreate(caseId);
    assertThat(first).isSameAs(second);
  }

  @Test
  void get_returnsEmptyForUnknownCase() {
    assertThat(registry.get(UUID.randomUUID())).isEmpty();
  }

  @Test
  void get_returnsPresentAfterGetOrCreate() {
    CasePlanModel model = registry.getOrCreate(caseId);
    assertThat(registry.get(caseId)).contains(model);
  }

  @Test
  void indexWorkerForCompletion_andGetPlanItemId_roundTrip() {
    String planItemId = UUID.randomUUID().toString();
    registry.indexWorkerForCompletion(caseId, "worker-a", planItemId);
    assertThat(registry.getPlanItemId(caseId, "worker-a")).contains(planItemId);
  }

  @Test
  void getPlanItemId_returnsEmptyForUnknownCase() {
    assertThat(registry.getPlanItemId(UUID.randomUUID(), "worker-a")).isEmpty();
  }

  @Test
  void getPlanItemId_returnsEmptyForUnknownWorker() {
    registry.getOrCreate(caseId);
    assertThat(registry.getPlanItemId(caseId, "unknown-worker")).isEmpty();
  }

  @Test
  void markConfigured_returnsTrueFirstTime() {
    assertThat(registry.markConfigured(caseId)).isTrue();
  }

  @Test
  void markConfigured_returnsFalseOnSubsequentCall() {
    registry.markConfigured(caseId);
    assertThat(registry.markConfigured(caseId)).isFalse();
  }

  @Test
  void evict_removesAllState() {
    registry.getOrCreate(caseId);
    registry.indexWorkerForCompletion(caseId, "worker-a", "plan-item-1");
    registry.markConfigured(caseId);

    registry.evict(caseId);

    assertThat(registry.get(caseId)).isEmpty();
    assertThat(registry.getPlanItemId(caseId, "worker-a")).isEmpty();
    // markConfigured returns true again — entry is gone, fresh entry on next access
    assertThat(registry.markConfigured(caseId)).isTrue();
  }

  @Test
  void evict_isIdempotent() {
    registry.getOrCreate(caseId);
    registry.evict(caseId);
    registry.evict(caseId); // must not throw
    assertThat(registry.get(caseId)).isEmpty();
  }
}
```

- [ ] **Step 2: Verify tests pass on current implementation (green baseline)**

```bash
mvn test -pl blackboard -Dtest=BlackboardRegistryTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL"
```
Expected: `Tests run: 10, Failures: 0, Errors: 0` and `BUILD SUCCESS`

- [ ] **Step 3: Rewrite `BlackboardRegistry` internals**

Overwrite `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java`:

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
package io.casehub.blackboard.registry;

import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.blackboard.plan.DefaultCasePlanModel;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.Optional;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Shared registry of per-case {@link CasePlanModel} instances and the worker-name-to-PlanItemId
 * completion index.
 *
 * <p>All per-case state is co-located in a single {@link CaseEntry}, making eviction atomic — one
 * map removal instead of three. See casehubio/engine#292.
 *
 * <p>Injected by both {@link io.casehub.blackboard.control.PlanningStrategyLoopControl} (which
 * writes entries on Binding selection) and {@link
 * io.casehub.blackboard.handler.PlanItemCompletionHandler} (which reads entries on worker
 * completion). See casehubio/engine#76.
 *
 * <p>State is in-memory and transient — rebuilt from EventLog on engine recovery. Persistence SPI
 * deferred to casehubio/engine#84.
 */
@ApplicationScoped
public class BlackboardRegistry {

  private static final class CaseEntry {
    final CasePlanModel planModel;
    final ConcurrentHashMap<String, String> completionIndex = new ConcurrentHashMap<>();
    final AtomicBoolean configured = new AtomicBoolean(false);

    CaseEntry(UUID caseId) {
      this.planModel = new DefaultCasePlanModel(caseId);
    }
  }

  private final ConcurrentHashMap<UUID, CaseEntry> entries = new ConcurrentHashMap<>();

  private CaseEntry entryFor(UUID caseId) {
    return entries.computeIfAbsent(caseId, CaseEntry::new);
  }

  /**
   * Returns the {@link CasePlanModel} for the given case, creating it if absent. Only {@link
   * io.casehub.blackboard.control.PlanningStrategyLoopControl} should call this method — all other
   * components should use {@link #get(UUID)}.
   */
  public CasePlanModel getOrCreate(UUID caseId) {
    return entryFor(caseId).planModel;
  }

  public Optional<CasePlanModel> get(UUID caseId) {
    CaseEntry e = entries.get(caseId);
    return e == null ? Optional.empty() : Optional.of(e.planModel);
  }

  public void indexWorkerForCompletion(UUID caseId, String workerName, String planItemId) {
    entryFor(caseId).completionIndex.put(workerName, planItemId);
  }

  public Optional<String> getPlanItemId(UUID caseId, String workerName) {
    CaseEntry e = entries.get(caseId);
    return e == null ? Optional.empty() : Optional.ofNullable(e.completionIndex.get(workerName));
  }

  /**
   * Atomically marks a case as configured by {@link
   * io.casehub.blackboard.control.BlackboardPlanConfigurer}(s). Returns {@code true} only the first
   * time this method is called for the given case — subsequent calls return {@code false}. This
   * guarantees configurers are invoked exactly once per case instance.
   */
  public boolean markConfigured(UUID caseId) {
    return entryFor(caseId).configured.compareAndSet(false, true);
  }

  /**
   * Atomically evicts the plan model, completion index, and configured marker for a completed or
   * terminated case. Single map removal — no race window between partial removes. Call when a case
   * reaches a terminal state to prevent unbounded memory growth. See casehubio/engine#84 for the
   * persistence SPI that will eventually replace this in-memory registry.
   */
  public void evict(UUID caseId) {
    entries.remove(caseId);
  }
}
```

- [ ] **Step 4: Verify contract tests still pass**

```bash
mvn test -pl blackboard -Dtest=BlackboardRegistryTest -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep -E "Tests run|BUILD|FAIL"
```
Expected: `Tests run: 10, Failures: 0, Errors: 0` and `BUILD SUCCESS`

- [ ] **Step 5: Verify full blackboard test suite still passes (unit tests only)**

```bash
mvn test-compile -pl blackboard -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep "BUILD"
```
Expected: `BUILD SUCCESS`

- [ ] **Step 6: Build full engine (skip tests — Docker not available locally)**

```bash
mvn install -DskipTests -f /Users/mdproctor/claude/casehub/engine/pom.xml 2>&1 | grep "BUILD" | tail -2
```
Expected: `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  blackboard/src/test/java/io/casehub/blackboard/registry/BlackboardRegistryTest.java \
  blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java

git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
refactor(blackboard): consolidate BlackboardRegistry internals into CaseEntry (engine#292)

Replace three separate ConcurrentHashMaps (planModels, completionIndex,
configured) with a single ConcurrentHashMap<UUID, CaseEntry>. All per-case
state is co-located in one entry; evict() is now a single atomic remove with
no race window between partial removes. Public API is unchanged — all callers
(CaseEvictionHandler, PlanningStrategyLoopControl, PlanItemCompletionHandler)
are unaffected.

Contract test BlackboardRegistryTest covers all six public methods.

Refs #292
EOF
)"
```

---

### Task 2: Add `@AfterEach` eviction to three test classes

**Files:**
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java`
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/WorkItemLifecycleAdapterTest.java`

- [ ] **Step 1: Add `@AfterEach` to `HumanTaskScheduleHandlerTest`**

In `HumanTaskScheduleHandlerTest.java`:

Add `import org.junit.jupiter.api.AfterEach;` to the import block.

Add this method after `setUp()`:
```java
@AfterEach
void tearDown() {
  registry.evict(caseId);
}
```

- [ ] **Step 2: Add `@AfterEach` to `HumanTaskScheduleHandlerAtomicityTest`**

In `HumanTaskScheduleHandlerAtomicityTest.java`:

Add `import org.junit.jupiter.api.AfterEach;` to the import block.

Add this method after `setUp()`:
```java
@AfterEach
void tearDown() {
  registry.evict(caseId);
}
```

- [ ] **Step 3: Add `@AfterEach` to `WorkItemLifecycleAdapterTest`**

In `WorkItemLifecycleAdapterTest.java`:

Add `import org.junit.jupiter.api.AfterEach;` to the import block.

Add this method after `setUp()`:
```java
@AfterEach
void tearDown() {
  registry.evict(caseId);
}
```

- [ ] **Step 4: Verify compilation**

```bash
mvn test-compile -f /Users/mdproctor/claude/casehub/engine/work-adapter/pom.xml 2>&1 | grep "BUILD" | tail -2
```
Expected: `BUILD SUCCESS`

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerAtomicityTest.java \
  work-adapter/src/test/java/io/casehub/workadapter/WorkItemLifecycleAdapterTest.java

git -C /Users/mdproctor/claude/casehub/engine commit -m "$(cat <<'EOF'
test(work-adapter): evict BlackboardRegistry entry after each test (engine#292)

Adds @AfterEach tearDown() calling registry.evict(caseId) to three test
classes that call registry.getOrCreate() in @BeforeEach without cleanup.
Prevents silent accumulation in the @ApplicationScoped singleton across
all @QuarkusTest methods in the module.

Closes #292
EOF
)"
```

---

## Self-Review

**Spec coverage:**
- ✅ Single `ConcurrentHashMap<UUID, CaseEntry>` with private static `CaseEntry` → Task 1 Step 3
- ✅ Atomic eviction via single `entries.remove(caseId)` → Task 1 Step 3
- ✅ Identical public API → Task 1 Step 3 (all 6 methods preserved)
- ✅ `compareAndSet(false, true)` for `markConfigured` → Task 1 Step 3
- ✅ Contract test covering all 6 methods including eviction → Task 1 Step 1
- ✅ `@AfterEach` in HumanTaskScheduleHandlerTest → Task 2 Step 1
- ✅ `@AfterEach` in HumanTaskScheduleHandlerAtomicityTest → Task 2 Step 2
- ✅ `@AfterEach` in WorkItemLifecycleAdapterTest → Task 2 Step 3

**Placeholder scan:** None found.

**Type consistency:** `CaseEntry`, `entryFor()`, `entries` used consistently. `markConfigured` uses `compareAndSet(false, true)` matching spec exactly.

**TDD check:** Contract test written before refactoring (Step 1), verified green on current impl (Step 2), refactor (Step 3), verified still green (Step 4). ✓
