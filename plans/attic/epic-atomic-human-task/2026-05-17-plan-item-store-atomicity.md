# PlanItemStore — Durable PlanItem Status and Handler Atomicity

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `PlanItem` status durable via the platform Store SPI pattern and fix `HumanTaskScheduleHandler` so that `markRunning()` and WorkItem creation are atomic — neither advances without the other.

**Architecture:** Extract `PlanItemStatus` to `casehub-engine-common` so the `PlanItemStore`/`ReactivePlanItemStore` SPIs (also in `common`) can reference it without pulling in the blackboard module. Blocking JPA impl (`JpaPlanItemStore` + `PlanItemEntity`) lives in `work-adapter` to share its blocking Hibernate datasource. `NoOpPlanItemStore` in `blackboard` is the `@DefaultBean` fallback. Handler gains `@Transactional`; execution order is: guard PENDING → create WorkItem → `planItemStore.updateStatus()` → `item.markRunning()`.

**Tech Stack:** Quarkus 3.x, Hibernate ORM (blocking, `work-adapter`), Hibernate Reactive (Panache, `persistence-hibernate`), Mutiny `Uni<>`, H2 (tests), CDI `@DefaultBean` / `@Alternative`.

---

## File Map

| Action | File |
|--------|------|
| Create | `common/src/main/java/io/casehub/engine/internal/model/PlanItemStatus.java` |
| Create | `common/src/main/java/io/casehub/engine/spi/PlanItemStore.java` |
| Create | `common/src/main/java/io/casehub/engine/spi/ReactivePlanItemStore.java` |
| Modify | `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java` |
| Create | `blackboard/src/main/java/io/casehub/blackboard/store/NoOpPlanItemStore.java` |
| Create | `blackboard/src/main/java/io/casehub/blackboard/store/NoOpReactivePlanItemStore.java` |
| Modify | `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java` |
| Modify | `blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java` |
| Create | `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java` |
| Create | `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java` |
| Create | `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java` |
| Create | `work-adapter/src/main/java/io/casehub/workadapter/PlanItemEntity.java` |
| Create | `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java` |
| Modify | `work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java` |
| Modify | `work-adapter/src/test/resources/application.properties` |
| Modify | `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java` |

---

### Task 1: `PlanItemStatus` standalone enum in `casehub-engine-common`

**Files:**
- Create: `common/src/main/java/io/casehub/engine/internal/model/PlanItemStatus.java`

- [ ] **Step 1.1: Create the enum**

```java
package io.casehub.engine.internal.model;

/** Lifecycle states for a {@link io.casehub.blackboard.plan.PlanItem}. */
public enum PlanItemStatus {
  PENDING,
  RUNNING,
  COMPLETED,
  FAULTED,
  CANCELLED
}
```

- [ ] **Step 1.2: Compile to verify**

```bash
mvn --batch-mode compile -pl common -q
```
Expected: BUILD SUCCESS

- [ ] **Step 1.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add common/src/main/java/io/casehub/engine/internal/model/PlanItemStatus.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(common): add standalone PlanItemStatus enum

Extracted from PlanItem nested enum so PlanItemStore SPI
can reference it from casehub-engine-common without pulling
in the blackboard module.

Refs #273"
```

---

### Task 2: `PlanItemStore` and `ReactivePlanItemStore` SPIs

**Files:**
- Create: `common/src/main/java/io/casehub/engine/spi/PlanItemStore.java`
- Create: `common/src/main/java/io/casehub/engine/spi/ReactivePlanItemStore.java`

- [ ] **Step 2.1: Write failing contract test for `PlanItemStore`**

Create `common/src/test/java/io/casehub/engine/spi/PlanItemStoreContractTest.java`:

```java
package io.casehub.engine.spi;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.engine.internal.model.PlanItemStatus;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

/** Contract test — extend with a concrete impl to verify the Store SPI. */
public abstract class PlanItemStoreContractTest {

  protected abstract PlanItemStore store();

  @Test
  void save_and_findByCaseId() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store().save(caseId, planItemId, "my-binding", PlanItemStatus.PENDING, Instant.now());
    List<io.casehub.engine.internal.model.PlanItemRecord> results = store().findByCaseId(caseId);
    assertThat(results).hasSize(1);
    assertThat(results.get(0).planItemId()).isEqualTo(planItemId);
    assertThat(results.get(0).status()).isEqualTo(PlanItemStatus.PENDING);
  }

  @Test
  void updateStatus_changes_stored_status() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store().save(caseId, planItemId, "my-binding", PlanItemStatus.PENDING, Instant.now());
    store().updateStatus(planItemId, PlanItemStatus.RUNNING);
    List<io.casehub.engine.internal.model.PlanItemRecord> results = store().findByCaseId(caseId);
    assertThat(results.get(0).status()).isEqualTo(PlanItemStatus.RUNNING);
  }
}
```

- [ ] **Step 2.2: Run to verify it fails**

```bash
mvn --batch-mode test -pl common -q 2>&1 | tail -5
```
Expected: compilation error — `PlanItemStore`, `PlanItemRecord` not found

- [ ] **Step 2.3: Create `PlanItemRecord` value object**

Create `common/src/main/java/io/casehub/engine/internal/model/PlanItemRecord.java`:

```java
package io.casehub.engine.internal.model;

import java.time.Instant;
import java.util.UUID;

/** Lightweight read model returned by {@link io.casehub.engine.spi.PlanItemStore#findByCaseId}. */
public record PlanItemRecord(
    UUID caseId,
    String planItemId,
    String bindingName,
    PlanItemStatus status,
    Instant createdAt) {}
```

- [ ] **Step 2.4: Create `PlanItemStore` SPI**

Create `common/src/main/java/io/casehub/engine/spi/PlanItemStore.java`:

```java
package io.casehub.engine.spi;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

/**
 * Blocking SPI for durable {@link io.casehub.blackboard.plan.PlanItem} status persistence.
 *
 * <p>Used by {@code HumanTaskScheduleHandler} (blocking context) to write PlanItem status
 * in the same JTA transaction as WorkItem creation. The default no-op implementation
 * ({@code NoOpPlanItemStore}) is active when no real store is on the classpath.
 *
 * @see ReactivePlanItemStore for the Uni-returning mirror
 */
public interface PlanItemStore {

  /** Record a new PlanItem. Called from {@code DefaultCasePlanModel.addPlanItem()}. */
  void save(UUID caseId, String planItemId, String bindingName, PlanItemStatus status, Instant createdAt);

  /** Update the stored status for an existing PlanItem. */
  void updateStatus(String planItemId, PlanItemStatus status);

  /** Return all PlanItemRecords for the given case. */
  List<PlanItemRecord> findByCaseId(UUID caseId);
}
```

- [ ] **Step 2.5: Create `ReactivePlanItemStore` SPI**

Create `common/src/main/java/io/casehub/engine/spi/ReactivePlanItemStore.java`:

```java
package io.casehub.engine.spi;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.smallrye.mutiny.Uni;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

/**
 * Reactive mirror of {@link PlanItemStore} — method signatures identical, return types wrapped
 * in {@link Uni}. For engine runtime handlers on Vert.x IO threads.
 */
public interface ReactivePlanItemStore {

  Uni<Void> save(UUID caseId, String planItemId, String bindingName, PlanItemStatus status, Instant createdAt);

  Uni<Void> updateStatus(String planItemId, PlanItemStatus status);

  Uni<List<PlanItemRecord>> findByCaseId(UUID caseId);
}
```

- [ ] **Step 2.6: Run contract test — should still fail (no concrete impl yet)**

```bash
mvn --batch-mode test -pl common -q 2>&1 | tail -5
```
Expected: tests skipped or fail — `PlanItemStoreContractTest` is abstract, no concrete subclass yet. That's fine — confirm compilation succeeds.

- [ ] **Step 2.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  common/src/main/java/io/casehub/engine/internal/model/PlanItemRecord.java \
  common/src/main/java/io/casehub/engine/spi/PlanItemStore.java \
  common/src/main/java/io/casehub/engine/spi/ReactivePlanItemStore.java \
  common/src/test/java/io/casehub/engine/spi/PlanItemStoreContractTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(common): add PlanItemStore + ReactivePlanItemStore SPIs

Blocking and reactive store SPIs for durable PlanItem status,
following the ledger dual-variant pattern. PlanItemRecord is the
lightweight read model returned by findByCaseId.

Refs #273"
```

---

### Task 3: Update `PlanItem` to use `PlanItemStatus` from common

**Files:**
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java`
- Modify: `blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java` (if it references nested enum)

- [ ] **Step 3.1: Use IntelliJ to move `PlanItem.PlanItemStatus` nested enum**

Use IntelliJ Refactor → Move to move the nested `PlanItemStatus` enum OUT of `PlanItem`. But since we already have a standalone enum in `common`, we'll instead delete the nested enum and update `PlanItem` to import from common.

In `PlanItem.java`, replace the nested enum declaration:

```java
// DELETE this block:
public enum PlanItemStatus {
  PENDING,
  RUNNING,
  COMPLETED,
  FAULTED,
  CANCELLED
}
```

Add import at top of `PlanItem.java`:

```java
import io.casehub.engine.internal.model.PlanItemStatus;
```

Change the field type from `PlanItem.PlanItemStatus` to `PlanItemStatus`:

```java
// Before:
private volatile PlanItem.PlanItemStatus status;

// After:
private volatile PlanItemStatus status;
```

Change constructor initialisation:

```java
// Before:
this.status = PlanItem.PlanItemStatus.PENDING;

// After:
this.status = PlanItemStatus.PENDING;
```

Update all method bodies in `PlanItem.java` that reference `PlanItem.PlanItemStatus.X` to use `PlanItemStatus.X`. The affected methods are `markRunning()`, `markCompleted()`, `markFaulted()`, `markCancelled()`. Example:

```java
// markRunning() — before:
if (status != PlanItem.PlanItemStatus.PENDING) {

// after:
if (status != PlanItemStatus.PENDING) {
```

- [ ] **Step 3.2: Compile blackboard to catch all broken references**

```bash
mvn --batch-mode install -DskipTests -q -pl common && \
mvn --batch-mode compile -pl blackboard -q 2>&1 | grep "error:"
```

Fix any remaining `PlanItem.PlanItemStatus` references in `DefaultCasePlanModel.java` and any test files — replace `PlanItem.PlanItemStatus.X` with `PlanItemStatus.X` and add import `io.casehub.engine.internal.model.PlanItemStatus`.

- [ ] **Step 3.3: Run blackboard tests**

```bash
mvn --batch-mode install -DskipTests -q -pl common && \
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -q
```
Expected: all tests pass

- [ ] **Step 3.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  blackboard/src/main/java/io/casehub/blackboard/plan/PlanItem.java \
  blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java \
  blackboard/src/test/java/io/casehub/blackboard/plan/PlanItemTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(blackboard): use PlanItemStatus from casehub-engine-common

Removes the nested PlanItem.PlanItemStatus enum in favour of the
standalone io.casehub.engine.internal.model.PlanItemStatus so the
PlanItemStore SPI in common can reference it without depending on
the blackboard module.

Refs #273"
```

---

### Task 4: No-op defaults + `BlackboardRegistry` + `DefaultCasePlanModel` wiring

**Files:**
- Create: `blackboard/src/main/java/io/casehub/blackboard/store/NoOpPlanItemStore.java`
- Create: `blackboard/src/main/java/io/casehub/blackboard/store/NoOpReactivePlanItemStore.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java`
- Modify: `blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java`

- [ ] **Step 4.1: Write failing test for DefaultCasePlanModel store integration**

Add to `blackboard/src/test/java/io/casehub/blackboard/plan/DefaultCasePlanModelTest.java` (create if it doesn't exist):

```java
package io.casehub.blackboard.plan;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

class DefaultCasePlanModelTest {

  /** Minimal recording store for unit tests — no CDI, no Quarkus. */
  static class RecordingPlanItemStore implements PlanItemStore {
    final List<PlanItemRecord> saved = new ArrayList<>();

    @Override
    public void save(UUID caseId, String planItemId, String bindingName,
                     PlanItemStatus status, Instant createdAt) {
      saved.add(new PlanItemRecord(caseId, planItemId, bindingName, status, createdAt));
    }

    @Override public void updateStatus(String planItemId, PlanItemStatus status) {}

    @Override public List<PlanItemRecord> findByCaseId(UUID caseId) { return saved; }
  }

  @Test
  void addPlanItem_saves_to_store() {
    UUID caseId = UUID.randomUUID();
    RecordingPlanItemStore store = new RecordingPlanItemStore();
    DefaultCasePlanModel model = new DefaultCasePlanModel(caseId, store);

    PlanItem item = PlanItem.create("my-binding", "my-worker", 5);
    model.addPlanItem(item);

    assertThat(store.saved).hasSize(1);
    assertThat(store.saved.get(0).planItemId()).isEqualTo(item.getPlanItemId());
    assertThat(store.saved.get(0).status()).isEqualTo(PlanItemStatus.PENDING);
  }

  @Test
  void addPlanItemIfAbsent_saves_to_store_when_added() {
    UUID caseId = UUID.randomUUID();
    RecordingPlanItemStore store = new RecordingPlanItemStore();
    DefaultCasePlanModel model = new DefaultCasePlanModel(caseId, store);

    PlanItem item = PlanItem.create("my-binding", "my-worker", 5);
    boolean added = model.addPlanItemIfAbsent(item);

    assertThat(added).isTrue();
    assertThat(store.saved).hasSize(1);
  }

  @Test
  void addPlanItemIfAbsent_does_not_save_when_already_active() {
    UUID caseId = UUID.randomUUID();
    RecordingPlanItemStore store = new RecordingPlanItemStore();
    DefaultCasePlanModel model = new DefaultCasePlanModel(caseId, store);

    PlanItem item = PlanItem.create("my-binding", "my-worker", 5);
    model.addPlanItem(item);
    store.saved.clear(); // reset

    boolean added = model.addPlanItemIfAbsent(PlanItem.create("my-binding", "my-worker", 5));
    assertThat(added).isFalse();
    assertThat(store.saved).isEmpty();
  }
}
```

- [ ] **Step 4.2: Run to verify failure**

```bash
mvn --batch-mode install -DskipTests -q -pl common && \
mvn --batch-mode test -pl blackboard -Dtest=DefaultCasePlanModelTest -q 2>&1 | tail -8
```
Expected: compilation error — `DefaultCasePlanModel(UUID, PlanItemStore)` constructor not found

- [ ] **Step 4.3: Create `NoOpPlanItemStore`**

Create `blackboard/src/main/java/io/casehub/blackboard/store/NoOpPlanItemStore.java`:

```java
package io.casehub.blackboard.store;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

/**
 * No-op {@link PlanItemStore} — active when no real store implementation is on the classpath.
 * PlanItem status is tracked in-memory only (via {@link io.casehub.blackboard.plan.PlanItem}).
 */
@DefaultBean
@ApplicationScoped
public class NoOpPlanItemStore implements PlanItemStore {

  @Override
  public void save(UUID caseId, String planItemId, String bindingName,
                   PlanItemStatus status, Instant createdAt) {}

  @Override
  public void updateStatus(String planItemId, PlanItemStatus status) {}

  @Override
  public List<PlanItemRecord> findByCaseId(UUID caseId) {
    return List.of();
  }
}
```

- [ ] **Step 4.4: Create `NoOpReactivePlanItemStore`**

Create `blackboard/src/main/java/io/casehub/blackboard/store/NoOpReactivePlanItemStore.java`:

```java
package io.casehub.blackboard.store;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.ReactivePlanItemStore;
import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

/** No-op reactive mirror of {@link NoOpPlanItemStore}. */
@DefaultBean
@ApplicationScoped
public class NoOpReactivePlanItemStore implements ReactivePlanItemStore {

  @Override
  public Uni<Void> save(UUID caseId, String planItemId, String bindingName,
                        PlanItemStatus status, Instant createdAt) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Void> updateStatus(String planItemId, PlanItemStatus status) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<List<PlanItemRecord>> findByCaseId(UUID caseId) {
    return Uni.createFrom().item(List.of());
  }
}
```

- [ ] **Step 4.5: Update `DefaultCasePlanModel` constructor and `addPlanItem` methods**

In `DefaultCasePlanModel.java`, change the constructor from `(UUID caseId)` to `(UUID caseId, PlanItemStore store)` and add a `store` field:

```java
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;

// ...

private final PlanItemStore store;

public DefaultCasePlanModel(UUID caseId, PlanItemStore store) {
  this.caseId = caseId;
  this.store = store;
}
```

Update `addPlanItem`:

```java
@Override
public void addPlanItem(PlanItem item) {
  agenda.add(item);
  itemsById.put(item.getPlanItemId(), item);
  activeByBinding.put(item.getBindingName(), item);
  store.save(caseId, item.getPlanItemId(), item.getBindingName(),
             PlanItemStatus.PENDING, item.getCreatedAt());
}
```

Update `addPlanItemIfAbsent` — call `store.save()` inside the compute lambda only when `added[0]` is set to true:

```java
@Override
public boolean addPlanItemIfAbsent(PlanItem item) {
  boolean[] added = {false};
  activeByBinding.compute(
      item.getBindingName(),
      (k, existing) -> {
        if (existing != null) {
          PlanItem.PlanItemStatus s = existing.getStatus();
          if (s == PlanItemStatus.PENDING || s == PlanItemStatus.RUNNING) {
            return existing;
          }
        }
        agenda.add(item);
        itemsById.put(item.getPlanItemId(), item);
        added[0] = true;
        return item;
      });
  if (added[0]) {
    store.save(caseId, item.getPlanItemId(), item.getBindingName(),
               PlanItemStatus.PENDING, item.getCreatedAt());
  }
  return added[0];
}
```

- [ ] **Step 4.6: Update `BlackboardRegistry` to inject `PlanItemStore` and pass it**

In `BlackboardRegistry.java`, add:

```java
import io.casehub.engine.spi.PlanItemStore;
import jakarta.inject.Inject;

@Inject PlanItemStore planItemStore;
```

Change `getOrCreate` to pass the store:

```java
public CasePlanModel getOrCreate(UUID caseId) {
  return planModels.computeIfAbsent(caseId, id -> new DefaultCasePlanModel(id, planItemStore));
}
```

- [ ] **Step 4.7: Run tests**

```bash
mvn --batch-mode install -DskipTests -q -pl common && \
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl blackboard -q
```
Expected: all pass including `DefaultCasePlanModelTest`

- [ ] **Step 4.8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  blackboard/src/main/java/io/casehub/blackboard/store/ \
  blackboard/src/main/java/io/casehub/blackboard/plan/DefaultCasePlanModel.java \
  blackboard/src/main/java/io/casehub/blackboard/registry/BlackboardRegistry.java \
  blackboard/src/test/java/io/casehub/blackboard/plan/DefaultCasePlanModelTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(blackboard): wire PlanItemStore into DefaultCasePlanModel

NoOpPlanItemStore (@DefaultBean) is the safe fallback when no real
store is deployed. BlackboardRegistry injects the store and passes
it to DefaultCasePlanModel. addPlanItem and addPlanItemIfAbsent
call store.save() so every PlanItem has a durable record from
creation.

Refs #273"
```

---

### Task 5: `MemoryPlanItemStore` + `MemoryReactivePlanItemStore` in `persistence-memory`

**Files:**
- Create: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java`
- Create: `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java`

Note: `persistence-memory` depends on `casehub-engine-common` — the SPIs and `PlanItemStatus` are available without any new pom changes.

- [ ] **Step 5.1: Add contract test subclass to persistence-memory test sources**

Create `persistence-memory/src/test/java/io/casehub/persistence/memory/MemoryPlanItemStoreContractTest.java`:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.spi.PlanItemStore;
import io.casehub.engine.spi.PlanItemStoreContractTest;

class MemoryPlanItemStoreContractTest extends PlanItemStoreContractTest {

  private final MemoryPlanItemStore store = new MemoryPlanItemStore();

  @Override
  protected PlanItemStore store() {
    return store;
  }
}
```

- [ ] **Step 5.2: Run to verify failure**

```bash
mvn --batch-mode install -DskipTests -q -pl common && \
mvn --batch-mode test -pl persistence-memory -Dtest=MemoryPlanItemStoreContractTest -q 2>&1 | tail -5
```
Expected: compilation error — `MemoryPlanItemStore` not found

- [ ] **Step 5.3: Create `MemoryPlanItemStore`**

Create `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java`:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

/**
 * In-memory {@link PlanItemStore} for use in engine unit tests.
 * Activated via {@code quarkus.arc.selected-alternatives} — never active in production.
 */
@Alternative
@ApplicationScoped
public class MemoryPlanItemStore implements PlanItemStore {

  private final ConcurrentHashMap<String, PlanItemRecord> records = new ConcurrentHashMap<>();

  public void clear() {
    records.clear();
  }

  @Override
  public void save(UUID caseId, String planItemId, String bindingName,
                   PlanItemStatus status, Instant createdAt) {
    records.put(planItemId,
        new PlanItemRecord(caseId, planItemId, bindingName, status, createdAt));
  }

  @Override
  public void updateStatus(String planItemId, PlanItemStatus status) {
    records.computeIfPresent(planItemId, (k, r) ->
        new PlanItemRecord(r.caseId(), r.planItemId(), r.bindingName(), status, r.createdAt()));
  }

  @Override
  public List<PlanItemRecord> findByCaseId(UUID caseId) {
    return records.values().stream()
        .filter(r -> caseId.equals(r.caseId()))
        .collect(Collectors.toList());
  }
}
```

- [ ] **Step 5.4: Create `MemoryReactivePlanItemStore`**

Create `persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java`:

```java
package io.casehub.persistence.memory;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.ReactivePlanItemStore;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

/** Reactive wrapper around {@link MemoryPlanItemStore} for test use. */
@Alternative
@ApplicationScoped
public class MemoryReactivePlanItemStore implements ReactivePlanItemStore {

  @Inject MemoryPlanItemStore delegate;

  @Override
  public Uni<Void> save(UUID caseId, String planItemId, String bindingName,
                        PlanItemStatus status, Instant createdAt) {
    delegate.save(caseId, planItemId, bindingName, status, createdAt);
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Void> updateStatus(String planItemId, PlanItemStatus status) {
    delegate.updateStatus(planItemId, status);
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<List<PlanItemRecord>> findByCaseId(UUID caseId) {
    return Uni.createFrom().item(delegate.findByCaseId(caseId));
  }
}
```

- [ ] **Step 5.5: Run contract tests**

```bash
mvn --batch-mode install -DskipTests -q -pl common && \
mvn --batch-mode test -pl persistence-memory -q
```
Expected: all pass

- [ ] **Step 5.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryPlanItemStore.java \
  persistence-memory/src/main/java/io/casehub/persistence/memory/MemoryReactivePlanItemStore.java \
  persistence-memory/src/test/java/io/casehub/persistence/memory/MemoryPlanItemStoreContractTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(persistence-memory): add MemoryPlanItemStore + reactive mirror

@Alternative @ApplicationScoped in-memory implementations for
engine tests. Activated via quarkus.arc.selected-alternatives.
Contract test subclass verifies all PlanItemStore SPI operations.

Refs #273"
```

---

### Task 6: `JpaReactivePlanItemStore` in `persistence-hibernate`

**Files:**
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java`
- Create: `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java`

Note: `persistence-hibernate` uses Panache reactive (Hibernate Reactive). This is the reactive impl for engine runtime handlers.

- [ ] **Step 6.1: Create `PlanItemEntity` (reactive Panache)**

Create `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java`:

```java
package io.casehub.persistence.jpa;

import io.casehub.engine.internal.model.PlanItemStatus;
import io.quarkus.hibernate.reactive.panache.PanacheEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.Table;
import java.time.Instant;
import java.util.UUID;
import org.hibernate.annotations.DynamicUpdate;

@Entity
@DynamicUpdate
@Table(name = "plan_item")
public class PlanItemEntity extends PanacheEntity {

  @Column(name = "plan_item_id", nullable = false, unique = true, length = 36)
  public String planItemId;

  @Column(name = "case_id", nullable = false)
  public UUID caseId;

  @Column(name = "binding_name", nullable = false, length = 255)
  public String bindingName;

  @Enumerated(EnumType.STRING)
  @Column(name = "status", nullable = false, length = 50)
  public PlanItemStatus status;

  @Column(name = "created_at", nullable = false)
  public Instant createdAt;
}
```

- [ ] **Step 6.2: Create `JpaReactivePlanItemStore`**

Create `persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java`:

```java
package io.casehub.persistence.jpa;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.ReactivePlanItemStore;
import io.quarkus.hibernate.reactive.panache.Panache;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class JpaReactivePlanItemStore extends AbstractJpaRepository
    implements ReactivePlanItemStore {

  @Override
  public Uni<Void> save(UUID caseId, String planItemId, String bindingName,
                        PlanItemStatus status, Instant createdAt) {
    PlanItemEntity e = new PlanItemEntity();
    e.caseId = caseId;
    e.planItemId = planItemId;
    e.bindingName = bindingName;
    e.status = status;
    e.createdAt = createdAt;
    return withSafeContext(() -> Panache.withTransaction(e::persist).replaceWithVoid());
  }

  @Override
  public Uni<Void> updateStatus(String planItemId, PlanItemStatus status) {
    return withSafeContext(() ->
        Panache.withTransaction(() ->
            PlanItemEntity.<PlanItemEntity>find("planItemId", planItemId)
                .firstResult()
                .invoke(e -> { if (e != null) e.status = status; })
                .replaceWithVoid()));
  }

  @Override
  public Uni<List<PlanItemRecord>> findByCaseId(UUID caseId) {
    return withSafeContext(() ->
        Panache.withSession(() ->
            PlanItemEntity.<PlanItemEntity>find("caseId", caseId)
                .list()
                .map(list -> list.stream()
                    .map(e -> new PlanItemRecord(e.caseId, e.planItemId, e.bindingName,
                                                e.status, e.createdAt))
                    .collect(Collectors.toList()))));
  }
}
```

- [ ] **Step 6.3: Compile persistence-hibernate**

```bash
mvn --batch-mode install -DskipTests -q -pl common,blackboard,persistence-memory && \
mvn --batch-mode compile -pl persistence-hibernate -q
```
Expected: BUILD SUCCESS

- [ ] **Step 6.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/PlanItemEntity.java \
  persistence-hibernate/src/main/java/io/casehub/persistence/jpa/JpaReactivePlanItemStore.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(persistence-hibernate): add JpaReactivePlanItemStore

Reactive Panache implementation of ReactivePlanItemStore.
PlanItemEntity is the JPA backing entity (plan_item table).
Schema managed by drop-and-create; no Flyway.

Refs #273"
```

---

### Task 7: `PlanItemEntity` + `JpaPlanItemStore` (blocking) in `work-adapter`

**Files:**
- Create: `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java`

Note: `PlanItemEntity` is shared with `persistence-hibernate` only if both are on the classpath. In practice the blocking `JpaPlanItemStore` uses `EntityManager` (standard JPA, not Panache reactive) and lives entirely in `work-adapter` which is in the blocking JPA world of casehub-work. We use a separate entity class here (`WorkAdapterPlanItemEntity`) to avoid class loading conflicts with the reactive `PlanItemEntity` from `persistence-hibernate`.

Actually — both entity classes map to the same `plan_item` table. Only ONE should be on the classpath in a given deployment. In the work-adapter test context (H2 blocking), the `persistence-hibernate` module is NOT on the classpath (it's a reactive module). So there's no conflict. We name the class distinctly to avoid confusion.

- [ ] **Step 7.1: Write failing test for `JpaPlanItemStore`**

Create `work-adapter/src/test/java/io/casehub/workadapter/JpaPlanItemStoreTest.java`:

```java
package io.casehub.workadapter;

import static org.assertj.core.api.Assertions.assertThat;
import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import org.junit.jupiter.api.Test;

@QuarkusTest
class JpaPlanItemStoreTest {

  @Inject JpaPlanItemStore store;

  @Test
  @Transactional
  void save_and_findByCaseId() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store.save(caseId, planItemId, "test-binding", PlanItemStatus.PENDING, Instant.now());
    List<PlanItemRecord> found = store.findByCaseId(caseId);
    assertThat(found).hasSize(1);
    assertThat(found.get(0).status()).isEqualTo(PlanItemStatus.PENDING);
  }

  @Test
  @Transactional
  void updateStatus_updates_stored_value() {
    UUID caseId = UUID.randomUUID();
    String planItemId = UUID.randomUUID().toString();
    store.save(caseId, planItemId, "test-binding", PlanItemStatus.PENDING, Instant.now());
    store.updateStatus(planItemId, PlanItemStatus.RUNNING);
    List<PlanItemRecord> found = store.findByCaseId(caseId);
    assertThat(found.get(0).status()).isEqualTo(PlanItemStatus.RUNNING);
  }
}
```

- [ ] **Step 7.2: Run to verify failure**

```bash
mvn --batch-mode install -DskipTests -q -pl common,blackboard,persistence-memory && \
mvn --batch-mode test -pl work-adapter -Dtest=JpaPlanItemStoreTest -q 2>&1 | tail -8
```
Expected: compilation error — `JpaPlanItemStore` not found

- [ ] **Step 7.3: Create the JPA entity (blocking)**

Create `work-adapter/src/main/java/io/casehub/workadapter/WorkAdapterPlanItemEntity.java`:

```java
package io.casehub.workadapter;

import io.casehub.engine.internal.model.PlanItemStatus;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "plan_item")
public class WorkAdapterPlanItemEntity {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  public Long id;

  @Column(name = "plan_item_id", nullable = false, unique = true, length = 36)
  public String planItemId;

  @Column(name = "case_id", nullable = false)
  public UUID caseId;

  @Column(name = "binding_name", nullable = false, length = 255)
  public String bindingName;

  @Enumerated(EnumType.STRING)
  @Column(name = "status", nullable = false, length = 50)
  public PlanItemStatus status;

  @Column(name = "created_at", nullable = false)
  public Instant createdAt;
}
```

- [ ] **Step 7.4: Create `JpaPlanItemStore`**

Create `work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java`:

```java
package io.casehub.workadapter;

import io.casehub.engine.internal.model.PlanItemRecord;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.persistence.TypedQuery;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import java.util.stream.Collectors;

/**
 * Blocking JPA {@link PlanItemStore} for use in the work-adapter context.
 *
 * <p>Uses the same blocking persistence unit as casehub-work, so writes participate
 * in the same JTA transaction as {@link io.casehub.work.runtime.service.WorkItemService}.
 */
@ApplicationScoped
public class JpaPlanItemStore implements PlanItemStore {

  @Inject EntityManager em;

  @Override
  public void save(UUID caseId, String planItemId, String bindingName,
                   PlanItemStatus status, Instant createdAt) {
    WorkAdapterPlanItemEntity e = new WorkAdapterPlanItemEntity();
    e.caseId = caseId;
    e.planItemId = planItemId;
    e.bindingName = bindingName;
    e.status = status;
    e.createdAt = createdAt;
    em.persist(e);
  }

  @Override
  public void updateStatus(String planItemId, PlanItemStatus status) {
    em.createQuery(
            "UPDATE WorkAdapterPlanItemEntity e SET e.status = :status WHERE e.planItemId = :planItemId")
        .setParameter("status", status)
        .setParameter("planItemId", planItemId)
        .executeUpdate();
  }

  @Override
  public List<PlanItemRecord> findByCaseId(UUID caseId) {
    return em.createQuery(
            "SELECT e FROM WorkAdapterPlanItemEntity e WHERE e.caseId = :caseId",
            WorkAdapterPlanItemEntity.class)
        .setParameter("caseId", caseId)
        .getResultList()
        .stream()
        .map(e -> new PlanItemRecord(e.caseId, e.planItemId, e.bindingName, e.status, e.createdAt))
        .collect(Collectors.toList());
  }
}
```

- [ ] **Step 7.5: Run `JpaPlanItemStoreTest`**

```bash
mvn --batch-mode install -DskipTests -q -pl common,blackboard,persistence-memory && \
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter \
  -Dtest=JpaPlanItemStoreTest -q
```
Expected: both tests pass

- [ ] **Step 7.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  work-adapter/src/main/java/io/casehub/workadapter/WorkAdapterPlanItemEntity.java \
  work-adapter/src/main/java/io/casehub/workadapter/JpaPlanItemStore.java \
  work-adapter/src/test/java/io/casehub/workadapter/JpaPlanItemStoreTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "feat(work-adapter): add JpaPlanItemStore and WorkAdapterPlanItemEntity

Blocking JPA PlanItemStore using the same persistence unit as
casehub-work so planItemStore.updateStatus() participates in the
same JTA transaction as workItemService.create().

Refs #273"
```

---

### Task 8: Update `HumanTaskScheduleHandler` — atomicity fix

**Files:**
- Modify: `work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java`
- Modify: `work-adapter/src/test/resources/application.properties`
- Modify: `work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java`

- [ ] **Step 8.1: Write new failing test — atomicity on WorkItem creation failure**

Add to `HumanTaskScheduleHandlerTest.java` a new inner store class and test. Add the inner class at the bottom of the test file, before the last `}`:

```java
  @Test
  void inlineMode_workItemCreationFails_planItemStaysPending_storeNotUpdated() {
    HumanTaskTarget target =
        HumanTaskTarget.inline().title("Review").build();

    // Force WorkItemStore to throw BEFORE this event fires
    FailingWorkItemStore.shouldFail.set(true);
    try {
      eventBus.publish(
          EventBusAddresses.HUMAN_TASK_SCHEDULE,
          new HumanTaskScheduleEvent(caseId, "irb-binding", target, Map.of()));

      try { Thread.sleep(500); } catch (InterruptedException ignored) {}

      assertThat(planItem.getStatus()).isEqualTo(PlanItem.PlanItemStatus.PENDING);
      // No WorkItem should exist
      assertThat(workItemStore.scanAll()).isEmpty();
      // Store should not show RUNNING for this planItemId
      List<io.casehub.engine.internal.model.PlanItemRecord> records =
          planItemStore.findByCaseId(caseId);
      assertThat(records).noneMatch(r ->
          r.planItemId().equals(planItem.getPlanItemId()) &&
          r.status() == io.casehub.engine.internal.model.PlanItemStatus.RUNNING);
    } finally {
      FailingWorkItemStore.shouldFail.set(false);
    }
  }
```

Also inject `PlanItemStore` in the test class:

```java
  @Inject io.casehub.engine.spi.PlanItemStore planItemStore;
```

Add `FailingWorkItemStore` static inner class inside `HumanTaskScheduleHandlerTest`:

```java
  @ApplicationScoped
  @Alternative
  @Priority(2)
  public static class FailingWorkItemStore implements WorkItemStore {
    static final java.util.concurrent.atomic.AtomicBoolean shouldFail =
        new java.util.concurrent.atomic.AtomicBoolean(false);
    private final java.util.Map<UUID, io.casehub.work.runtime.model.WorkItem> store =
        new java.util.concurrent.ConcurrentHashMap<>();

    public void clear() { store.clear(); }

    @Override
    public io.casehub.work.runtime.model.WorkItem put(io.casehub.work.runtime.model.WorkItem w) {
      if (shouldFail.get()) throw new RuntimeException("Simulated WorkItemStore failure");
      store.put(w.id, w);
      return w;
    }

    @Override
    public java.util.Optional<io.casehub.work.runtime.model.WorkItem> get(UUID id) {
      return java.util.Optional.ofNullable(store.get(id));
    }

    @Override
    public java.util.List<io.casehub.work.runtime.model.WorkItem> scan(
        io.casehub.work.runtime.repository.WorkItemQuery query) {
      return new java.util.ArrayList<>(store.values());
    }
  }
```

- [ ] **Step 8.2: Add `FailingWorkItemStore` and `MemoryPlanItemStore` to `selected-alternatives`**

In `work-adapter/src/test/resources/application.properties`, add to the `quarkus.arc.selected-alternatives` list:

```properties
quarkus.arc.selected-alternatives=\
  io.casehub.persistence.memory.InMemoryCaseInstanceRepository,\
  io.casehub.persistence.memory.InMemoryCaseMetaModelRepository,\
  io.casehub.persistence.memory.InMemoryEventLogRepository,\
  io.casehub.persistence.memory.MemorySubCaseGroupRepository,\
  io.casehub.persistence.memory.MemoryPlanItemStore,\
  io.casehub.workadapter.HumanTaskScheduleHandlerTest$FailingWorkItemStore
```

Also add the `index-dependency` entry so Quarkus discovers the inner class:

```properties
quarkus.index-dependency.work-adapter-test.group-id=io.casehub
quarkus.index-dependency.work-adapter-test.artifact-id=casehub-engine-work-adapter
```

- [ ] **Step 8.3: Run to confirm new test fails**

```bash
mvn --batch-mode install -DskipTests -q -pl common,blackboard,persistence-memory && \
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter \
  -Dtest=HumanTaskScheduleHandlerTest#inlineMode_workItemCreationFails_planItemStaysPending_storeNotUpdated \
  -q 2>&1 | tail -10
```
Expected: test fails — planItem.getStatus() is RUNNING (current broken behaviour)

- [ ] **Step 8.4: Update `HumanTaskScheduleHandler`**

Replace the full content of `HumanTaskScheduleHandler.java`:

```java
package io.casehub.workadapter;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.api.model.HumanTaskTarget;
import io.casehub.blackboard.plan.CasePlanModel;
import io.casehub.blackboard.plan.PlanItem;
import io.casehub.blackboard.registry.BlackboardRegistry;
import io.casehub.engine.internal.event.EventBusAddresses;
import io.casehub.engine.internal.event.HumanTaskScheduleEvent;
import io.casehub.engine.internal.model.PlanItemStatus;
import io.casehub.engine.spi.PlanItemStore;
import io.casehub.work.runtime.model.WorkItemCreateRequest;
import io.casehub.work.runtime.model.WorkItemTemplate;
import io.casehub.work.runtime.service.WorkItemService;
import io.casehub.work.runtime.service.WorkItemTemplateService;
import io.quarkus.vertx.ConsumeEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.Map;
import org.jboss.logging.Logger;

/**
 * Handles outbound human task creation when a {@link HumanTaskTarget} binding is selected.
 *
 * <p>Execution order (both modes) — all inside one {@code @Transactional} boundary:
 * <ol>
 *   <li>Guard: PlanItem must be PENDING (fast-fail before DB work)
 *   <li>WorkItem creation via WorkItemService / WorkItemTemplateService
 *   <li>{@code planItemStore.updateStatus(RUNNING)} — durable record in same transaction
 *   <li>{@code item.markRunning()} — in-memory sync, only reached on clean commit
 * </ol>
 * If steps 2 or 3 throw, the transaction rolls back atomically.
 * {@code item.markRunning()} is never called, leaving the PlanItem PENDING.
 *
 * <p>Refs engine#273, protocol PP-20260517-cbf836.
 */
@ApplicationScoped
public class HumanTaskScheduleHandler {

  private static final Logger LOG = Logger.getLogger(HumanTaskScheduleHandler.class);
  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Inject BlackboardRegistry registry;
  @Inject WorkItemService workItemService;
  @Inject WorkItemTemplateService workItemTemplateService;
  @Inject PlanItemStore planItemStore;

  @Transactional
  @ConsumeEvent(value = EventBusAddresses.HUMAN_TASK_SCHEDULE, blocking = true)
  public void onHumanTaskSchedule(HumanTaskScheduleEvent event) {
    CasePlanModel plan = registry.get(event.caseId()).orElse(null);
    if (plan == null) {
      LOG.warnf("No CasePlanModel for caseId=%s — case may not use blackboard or has completed",
          event.caseId());
      return;
    }

    PlanItem item = plan.getPlanItemByBindingName(event.bindingName()).orElse(null);
    if (item == null) {
      LOG.warnf("PlanItem for binding '%s' not found in case %s",
          event.bindingName(), event.caseId());
      return;
    }

    if (item.getStatus() != PlanItem.PlanItemStatus.PENDING) {
      LOG.warnf("PlanItem for binding '%s' case %s is not PENDING (status=%s) — skipping",
          event.bindingName(), event.caseId(), item.getStatus());
      return;
    }

    if (event.target().isTemplateMode()) {
      handleTemplateMode(item, event);
    } else {
      handleInlineMode(item, event);
    }
  }

  private void handleTemplateMode(PlanItem item, HumanTaskScheduleEvent event) {
    HumanTaskTarget target = event.target();

    WorkItemTemplate template;
    try {
      template = workItemTemplateService.findByRef(target.templateRef()).orElse(null);
    } catch (IllegalStateException e) {
      LOG.warnf(
          "Ambiguous templateRef '%s' for binding '%s' case %s: %s — PlanItem left PENDING",
          target.templateRef(), event.bindingName(), event.caseId(), e.getMessage());
      return;
    }

    if (template == null) {
      LOG.warnf(
          "No template found for ref '%s' binding '%s' case %s — PlanItem left PENDING",
          target.templateRef(), event.bindingName(), event.caseId());
      return;
    }

    String callerRef = CallerRef.encode(event.caseId(), item.getPlanItemId());
    workItemTemplateService.instantiate(
        template, target.title(), null, "casehub-engine", callerRef,
        serializePayload(event.inputData()));

    planItemStore.updateStatus(item.getPlanItemId(), PlanItemStatus.RUNNING);
    item.markRunning();
    LOG.infof("WorkItem created (template=%s) for binding callerRef=%s", template.id, callerRef);
  }

  private void handleInlineMode(PlanItem item, HumanTaskScheduleEvent event) {
    String callerRef = CallerRef.encode(event.caseId(), item.getPlanItemId());
    createInline(event.target(), event.inputData(), callerRef);

    planItemStore.updateStatus(item.getPlanItemId(), PlanItemStatus.RUNNING);
    item.markRunning();
    LOG.infof("WorkItem created (inline) for binding callerRef=%s title='%s'",
        callerRef, event.target().title());
  }

  private void createInline(HumanTaskTarget target, Map<String, Object> inputData, String callerRef) {
    String payload = serializePayload(inputData);
    WorkItemCreateRequest request =
        new WorkItemCreateRequest(
            target.title(), null, null, null, null, null,
            candidateGroupsCsv(target), candidateUsersCsv(target),
            null, "casehub-engine", payload, null,
            target.expiresIn() != null ? Instant.now().plus(target.expiresIn()) : null,
            null, null, null, callerRef, null, null);
    workItemService.create(request);
  }

  private String serializePayload(Map<String, Object> inputData) {
    if (inputData == null || inputData.isEmpty()) return null;
    try {
      return MAPPER.writeValueAsString(inputData);
    } catch (JsonProcessingException e) {
      LOG.warnf(e, "Failed to serialize inputData to JSON payload — using null");
      return null;
    }
  }

  private static String candidateGroupsCsv(HumanTaskTarget target) {
    if (target.candidateGroups() == null || target.candidateGroups().isEmpty()) return null;
    return String.join(",", target.candidateGroups());
  }

  private static String candidateUsersCsv(HumanTaskTarget target) {
    if (target.candidateUsers() == null || target.candidateUsers().isEmpty()) return null;
    return String.join(",", target.candidateUsers());
  }
}
```

- [ ] **Step 8.5: Update existing handler tests to also verify store state**

In `HumanTaskScheduleHandlerTest`, in `setUp()`, add `planItemStore` clearing. If `planItemStore` is a `MemoryPlanItemStore`, cast and clear:

```java
@BeforeEach
@Transactional
void setUp() {
  if (workItemStore instanceof InMemoryWorkItemStore mem) mem.clear();
  if (planItemStore instanceof io.casehub.persistence.memory.MemoryPlanItemStore mem) mem.clear();
  WorkItemTemplate.deleteAll();
  caseId = UUID.randomUUID();
  planItem = PlanItem.create("irb-binding", "unused-worker", 5);
  registry.getOrCreate(caseId).addPlanItem(planItem);
}
```

In the success test `inlineMode_createsWorkItem_withCallerRef_andMarksPlanItemRunning`, add a store assertion:

```java
    // also verify store was updated
    await().atMost(5, TimeUnit.SECONDS).untilAsserted(() ->
        assertThat(planItemStore.findByCaseId(caseId))
            .anyMatch(r -> r.planItemId().equals(planItem.getPlanItemId())
                       && r.status() == io.casehub.engine.internal.model.PlanItemStatus.RUNNING));
```

- [ ] **Step 8.6: Run all work-adapter tests**

```bash
mvn --batch-mode install -DskipTests -q \
  -pl common,blackboard,persistence-memory,work-adapter/../persistence-hibernate && \
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test -pl work-adapter -q
```
Expected: all tests pass including the new atomicity test

- [ ] **Step 8.7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  work-adapter/src/main/java/io/casehub/workadapter/HumanTaskScheduleHandler.java \
  work-adapter/src/test/resources/application.properties \
  work-adapter/src/test/java/io/casehub/workadapter/HumanTaskScheduleHandlerTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix(work-adapter): make markRunning + WorkItem creation atomic

Closes #273.

Execution order: WorkItem creation → planItemStore.updateStatus(RUNNING)
→ item.markRunning(). All three in one @Transactional boundary.
If WorkItem creation fails, the transaction rolls back and markRunning()
is never called — PlanItem stays PENDING.

Refs protocol PP-20260517-cbf836"
```

---

### Task 9: Full build verification

- [ ] **Step 9.1: Install all modules**

```bash
mvn --batch-mode install -DskipTests -q
```
Expected: BUILD SUCCESS

- [ ] **Step 9.2: Run all affected modules' tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn --batch-mode test \
  -pl common,blackboard,persistence-memory,work-adapter -q
```
Expected: all tests pass

- [ ] **Step 9.3: Done — proceed to code review**

Invoke `superpowers:requesting-code-review` before committing.
