# CbrCollectionManager Async Methods — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #165 — refactor: CbrCollectionManager async methods — complete event-loop safety for reactive Qdrant store
**Issue group:** #165

**Goal:** Eliminate the async → blocking → async double conversion in the Qdrant CBR backend by making `CbrCollectionManager` methods natively async, then updating `ReactiveQdrantCbrCaseMemoryStore` to use them directly.

**Architecture:** Async methods returning `Uni<T>` become the canonical implementation in `CbrCollectionManager`, using a `toUni(ListenableFuture)` adapter to chain Qdrant gRPC calls natively. Blocking methods become one-liner wrappers calling `.await().indefinitely()`. The reactive store drops all `runSubscriptionOn(workerPool)` calls for collection management operations.

**Tech Stack:** Java 21, Mutiny (Uni/Multi), Qdrant Java Client (ListenableFuture/gRPC), Guava Futures, Quarkus, JUnit 5, Testcontainers

## Global Constraints

- All source file edits use IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_insert_member`, `ide_create_file`). No bash Edit/Write on `.java` files.
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl memory-qdrant -am`
- Test single module: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-qdrant`
- All Multi chains use `.collect().asList()` as terminal operator — never `.toUni()` (risks incomplete processing).
- `project_path` for all IntelliJ MCP calls: `/Users/mdproctor/claude/casehub/neocortex`

---

### Task 1: QdrantFutures utility class

**Files:**
- Create: `memory-qdrant/src/main/java/io/casehub/neocortex/memory/cbr/qdrant/QdrantFutures.java`
- Create: `memory-qdrant/src/test/java/io/casehub/neocortex/memory/cbr/qdrant/QdrantFuturesTest.java`

**Interfaces:**
- Consumes: `com.google.common.util.concurrent.ListenableFuture<T>` (Qdrant client returns)
- Produces: `static <T> Uni<T> toUni(ListenableFuture<T> future)` — used by Task 2 (CbrCollectionManager) and Task 3 (ReactiveQdrantCbrCaseMemoryStore)

- [ ] **Step 1: Write the test class**

Create `QdrantFuturesTest.java` mirroring the rag module's existing test at `rag/src/test/java/io/casehub/neocortex/rag/runtime/QdrantFuturesTest.java`. Four tests:

```java
package io.casehub.neocortex.memory.cbr.qdrant;

import com.google.common.util.concurrent.SettableFuture;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class QdrantFuturesTest {

    @Test
    void successPropagatesItemToUni() {
        SettableFuture<String> future = SettableFuture.create();
        var uni = QdrantFutures.<String>toUni(future);
        future.set("hello");
        String result = uni.await().indefinitely();
        assertThat(result).isEqualTo("hello");
    }

    @Test
    void failurePropagatesExceptionToUni() {
        SettableFuture<String> future = SettableFuture.create();
        var uni = QdrantFutures.<String>toUni(future);
        var cause = new RuntimeException("boom");
        future.setException(cause);
        assertThatThrownBy(() -> uni.await().indefinitely())
            .isInstanceOf(RuntimeException.class)
            .hasMessage("boom");
    }

    @Test
    void cancellingUniCancelsListenableFuture() {
        SettableFuture<String> future = SettableFuture.create();
        var uni = QdrantFutures.<String>toUni(future);
        var cancellable = uni.subscribe().with(item -> {}, failure -> {});
        cancellable.cancel();
        assertThat(future.isCancelled()).isTrue();
    }

    @Test
    void alreadyCompletedFutureDeliversImmediately() {
        SettableFuture<Integer> future = SettableFuture.create();
        future.set(42);
        Integer result = QdrantFutures.<Integer>toUni(future).await().indefinitely();
        assertThat(result).isEqualTo(42);
    }
}
```

Use `ide_create_file` to create the test file.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-qdrant -Dtest=QdrantFuturesTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `QdrantFutures` class does not exist yet.

- [ ] **Step 3: Create QdrantFutures.java**

```java
package io.casehub.neocortex.memory.cbr.qdrant;

import com.google.common.util.concurrent.FutureCallback;
import com.google.common.util.concurrent.Futures;
import com.google.common.util.concurrent.ListenableFuture;
import com.google.common.util.concurrent.MoreExecutors;
import io.smallrye.mutiny.Uni;

final class QdrantFutures {

    private QdrantFutures() {}

    static <T> Uni<T> toUni(ListenableFuture<T> future) {
        return Uni.createFrom().emitter(em -> {
            em.onTermination(() -> future.cancel(false));
            Futures.addCallback(future, new FutureCallback<>() {
                @Override
                public void onSuccess(T result) {
                    em.complete(result);
                }

                @Override
                public void onFailure(Throwable t) {
                    em.fail(t);
                }
            }, MoreExecutors.directExecutor());
        });
    }
}
```

Use `ide_create_file` to create the source file.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-qdrant -Dtest=QdrantFuturesTest`
Expected: 4 tests PASS.

- [ ] **Step 5: Verify with ide_diagnostics**

Run `ide_diagnostics` on `QdrantFutures.java` and `QdrantFuturesTest.java` to confirm no errors or warnings.

- [ ] **Step 6: Commit**

```bash
git add memory-qdrant/src/main/java/io/casehub/neocortex/memory/cbr/qdrant/QdrantFutures.java \
      memory-qdrant/src/test/java/io/casehub/neocortex/memory/cbr/qdrant/QdrantFuturesTest.java
git commit -m "feat(#165): add QdrantFutures utility — toUni with cancellation propagation"
```

---

### Task 2: CbrCollectionManager — async canonical methods

**Files:**
- Modify: `memory-qdrant/src/main/java/io/casehub/neocortex/memory/cbr/qdrant/CbrCollectionManager.java`

**Interfaces:**
- Consumes: `QdrantFutures.toUni(ListenableFuture<T>)` from Task 1
- Produces:
  - `Uni<Void> ensureCollectionAsync(String caseType, int vectorDimension)`
  - `Uni<Void> registerSchemaIndexesAsync(CbrFeatureSchema schema, int vectorDimension)`
  - `Uni<Integer> deleteByFilterAsync(String collection, Filter filter)`
  - Blocking wrappers unchanged: `void ensureCollection(...)`, `void registerSchemaIndexes(...)`, `int deleteByFilter(...)`

- [ ] **Step 1: Add Mutiny imports to CbrCollectionManager**

Add required imports using `ide_edit_member` on the class declaration or `ide_insert_member`. The file needs:
```java
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;
import static io.casehub.neocortex.memory.cbr.qdrant.QdrantFutures.toUni;
```

- [ ] **Step 2: Add `createBasePayloadIndexesAsync` private method**

Convert the existing `createBasePayloadIndexes` to async. Use `ide_insert_member` to add after the existing `createBasePayloadIndexes` method:

```java
private Uni<Void> createBasePayloadIndexesAsync(String collection) {
    return Multi.createFrom().items(BASE_KEYWORD_FIELDS)
        .onItem().transformToUniAndConcatenate(field ->
            toUni(client.createPayloadIndexAsync(collection, field,
                PayloadSchemaType.Keyword, null, true, null, null)).replaceWithVoid())
        .collect().asList()
        .chain(() -> toUni(client.createPayloadIndexAsync(collection, "_stored_at",
            PayloadSchemaType.Float, null, true, null, null)).replaceWithVoid());
}
```

- [ ] **Step 3: Add `ensureCollectionAsync` method**

Use `ide_insert_member` to add after the existing `ensureCollection` method. This is the largest method — translates the blocking try/catch chain into Uni composition:

```java
Uni<Void> ensureCollectionAsync(String caseType, int vectorDimension) {
    String collection = collectionName(caseType);
    if (knownCollections.contains(collection)) {
        return Uni.createFrom().voidItem();
    }

    int effectiveDim = vectorDimension > 0 ? vectorDimension : 1;

    return toUni(client.collectionExistsAsync(collection))
        .chain(exists -> {
            if (exists) {
                return toUni(client.getCollectionInfoAsync(collection))
                    .chain(info -> handleExistingCollectionAsync(collection, caseType, info, effectiveDim));
            }
            return createNewCollectionAsync(collection, effectiveDim);
        })
        .invoke(() -> knownCollections.add(collection));
}
```

- [ ] **Step 4: Add `handleExistingCollectionAsync` private helper**

Use `ide_insert_member`:

```java
private Uni<Void> handleExistingCollectionAsync(String collection, String caseType,
                                                 io.qdrant.client.grpc.Collections.CollectionInfo info,
                                                 int effectiveDim) {
    var vectorsConfig = info.getConfig().getParams().getVectorsConfig();
    if (vectorsConfig.hasParamsMap()) {
        var params = vectorsConfig.getParamsMap().getMapMap().get(config.denseVectorName());
        if (params != null && params.getSize() != effectiveDim) {
            if (!config.allowDimensionMigration()) {
                return Uni.createFrom().failure(
                    new CbrDimensionMismatchException(collection, (int) params.getSize(), effectiveDim));
            }
            LOG.warning("Collection " + collection + " dimension mismatch ("
                + params.getSize() + " → " + effectiveDim
                + ") — recreating. ALL tenants sharing caseType=" + caseType
                + " are affected. Run reconciliation per tenant to recover data.");
            return toUni(client.deleteCollectionAsync(collection))
                .chain(v -> createNewCollectionAsync(collection, effectiveDim));
        } else if (hasMissingSparseVectors(info)) {
            if (!config.allowSparseVectorMigration()) {
                return Uni.createFrom().failure(new CbrSparseVectorMigrationException(collection));
            }
            LOG.warning("Collection " + collection
                + " missing required sparse vectors — recreating."
                + " Run reconciliation per tenant to recover data.");
            return toUni(client.deleteCollectionAsync(collection))
                .chain(v -> createNewCollectionAsync(collection, effectiveDim));
        } else {
            return Uni.createFrom().voidItem();
        }
    } else if (hasMissingSparseVectors(info)) {
        if (!config.allowSparseVectorMigration()) {
            return Uni.createFrom().failure(new CbrSparseVectorMigrationException(collection));
        }
        LOG.warning("Collection " + collection
            + " missing required sparse vectors — recreating."
            + " Run reconciliation per tenant to recover data.");
        return toUni(client.deleteCollectionAsync(collection))
            .chain(v -> createNewCollectionAsync(collection, effectiveDim));
    } else {
        return Uni.createFrom().voidItem();
    }
}
```

- [ ] **Step 5: Add `createNewCollectionAsync` private helper**

Use `ide_insert_member`:

```java
private Uni<Void> createNewCollectionAsync(String collection, int effectiveDim) {
    VectorParams denseParams = VectorParams.newBuilder()
        .setSize(effectiveDim)
        .setDistance(Distance.Cosine)
        .build();

    CreateCollection.Builder createBuilder = CreateCollection.newBuilder()
        .setCollectionName(collection)
        .setVectorsConfig(VectorsConfig.newBuilder()
            .setParamsMap(VectorParamsMap.newBuilder()
                .putMap(config.denseVectorName(), denseParams)
                .build())
            .build());

    if (config.spladeEnabled() || config.bm25Enabled()) {
        SparseVectorConfig.Builder sparseBuilder = SparseVectorConfig.newBuilder();
        if (config.spladeEnabled()) {
            sparseBuilder.putMap(config.spladeVectorName(), SparseVectorParams.getDefaultInstance());
        }
        if (config.bm25Enabled()) {
            sparseBuilder.putMap(config.bm25VectorName(),
                SparseVectorParams.newBuilder().setModifier(Modifier.Idf).build());
        }
        createBuilder.setSparseVectorsConfig(sparseBuilder.build());
    }

    return toUni(client.createCollectionAsync(createBuilder.build()))
        .chain(v -> createBasePayloadIndexesAsync(collection));
}
```

- [ ] **Step 6: Add `indexesForField` private helper**

Use `ide_insert_member`:

```java
private Uni<Void> indexesForField(String collection, String payloadKey, FeatureField field) {
    return switch (field) {
        case FeatureField.Categorical c -> toUni(client.createPayloadIndexAsync(collection, payloadKey,
            PayloadSchemaType.Keyword, null, true, null, null)).replaceWithVoid();
        case FeatureField.Numeric n -> toUni(client.createPayloadIndexAsync(collection, payloadKey,
            PayloadSchemaType.Float, null, true, null, null)).replaceWithVoid();
        case FeatureField.Text t -> toUni(client.createPayloadIndexAsync(collection, payloadKey,
            PayloadSchemaType.Keyword, null, true, null, null)).replaceWithVoid();
        case FeatureField.CategoricalList cl -> toUni(client.createPayloadIndexAsync(collection, payloadKey,
            PayloadSchemaType.Keyword, null, true, null, null)).replaceWithVoid();
        case FeatureField.NumericList nl -> toUni(client.createPayloadIndexAsync(collection, payloadKey,
            PayloadSchemaType.Float, null, true, null, null)).replaceWithVoid();
        case FeatureField.NestedObject no -> Multi.createFrom().iterable(no.innerFields())
            .onItem().transformToUniAndConcatenate(inner ->
                toUni(client.createPayloadIndexAsync(collection, payloadKey + "." + inner.name(),
                    innerPayloadType(inner), null, true, null, null)).replaceWithVoid())
            .collect().asList().replaceWithVoid();
        case FeatureField.ObjectList ol -> Multi.createFrom().iterable(ol.innerFields())
            .onItem().transformToUniAndConcatenate(inner ->
                toUni(client.createPayloadIndexAsync(collection, payloadKey + "[]." + inner.name(),
                    innerPayloadType(inner), null, true, null, null)).replaceWithVoid())
            .collect().asList().replaceWithVoid();
        case FeatureField.TimeSeries ts -> Uni.createFrom().voidItem();
        case FeatureField.DiscreteSequence ds -> Uni.createFrom().voidItem();
    };
}
```

- [ ] **Step 7: Add `registerSchemaIndexesAsync` method**

Use `ide_insert_member` to add after the existing `registerSchemaIndexes`:

```java
Uni<Void> registerSchemaIndexesAsync(CbrFeatureSchema schema, int vectorDimension) {
    return ensureCollectionAsync(schema.caseType(), vectorDimension)
        .chain(() -> {
            String collection = collectionName(schema.caseType());
            return Multi.createFrom().iterable(schema.fields())
                .onItem().transformToUniAndConcatenate(field ->
                    indexesForField(collection, "f_" + field.name(), field))
                .collect().asList()
                .replaceWithVoid();
        });
}
```

- [ ] **Step 8: Add `deleteByFilterAsync` method**

Use `ide_insert_member` to add after the existing `deleteByFilter`:

```java
Uni<Integer> deleteByFilterAsync(String collection, io.qdrant.client.grpc.Common.Filter filter) {
    var scrollBuilder = io.qdrant.client.grpc.Points.ScrollPoints.newBuilder()
        .setCollectionName(collection)
        .setFilter(filter)
        .setLimit(10000)
        .setWithPayload(io.qdrant.client.WithPayloadSelectorFactory.enable(false));

    return toUni(client.scrollAsync(scrollBuilder.build()))
        .chain(response -> {
            int count = response.getResultCount();
            if (count > 0) {
                return toUni(client.deleteAsync(collection, filter))
                    .replaceWith(count);
            }
            return Uni.createFrom().item(0);
        });
}
```

- [ ] **Step 9: Convert blocking methods to wrappers**

Use `ide_replace_member` on each of the three blocking methods to replace their bodies:

**`ensureCollection`:**
```java
void ensureCollection(String caseType, int vectorDimension) {
    ensureCollectionAsync(caseType, vectorDimension).await().indefinitely();
}
```

**`registerSchemaIndexes`:**
```java
void registerSchemaIndexes(CbrFeatureSchema schema, int vectorDimension) {
    registerSchemaIndexesAsync(schema, vectorDimension).await().indefinitely();
}
```

**`deleteByFilter`:**
```java
int deleteByFilter(String collection, io.qdrant.client.grpc.Common.Filter filter) {
    return deleteByFilterAsync(collection, filter).await().indefinitely();
}
```

The old `createBasePayloadIndexes` private method can be removed (replaced by `createBasePayloadIndexesAsync`). Use `ide_refactor_safe_delete` or `ide_edit_member` to remove it.

- [ ] **Step 10: Verify with ide_diagnostics**

Run `ide_diagnostics` on `CbrCollectionManager.java` to confirm no errors.

- [ ] **Step 11: Run existing tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-qdrant`
Expected: All existing tests pass — blocking wrappers preserve identical behavior.

- [ ] **Step 12: Commit**

```bash
git add memory-qdrant/src/main/java/io/casehub/neocortex/memory/cbr/qdrant/CbrCollectionManager.java
git commit -m "feat(#165): CbrCollectionManager async canonical — ensureCollectionAsync, registerSchemaIndexesAsync, deleteByFilterAsync"
```

---

### Task 3: ReactiveQdrantCbrCaseMemoryStore — use async methods, remove worker pool dispatch

**Files:**
- Modify: `memory-qdrant/src/main/java/io/casehub/neocortex/memory/cbr/qdrant/ReactiveQdrantCbrCaseMemoryStore.java`

**Interfaces:**
- Consumes: `CbrCollectionManager.ensureCollectionAsync()`, `.registerSchemaIndexesAsync()`, `.deleteByFilterAsync()` from Task 2; `QdrantFutures.toUni()` from Task 1
- Produces: No API changes — `ReactiveCbrCaseMemoryStore` SPI unchanged

- [ ] **Step 1: Remove private `toUni` method and update imports**

Remove the private static `toUni` method at lines 111-117. Add static import:
```java
import static io.casehub.neocortex.memory.cbr.qdrant.QdrantFutures.toUni;
```

Remove the now-unused imports for `FutureCallback`, `Futures`, `MoreExecutors` (these were only used by the private `toUni`). Use `ide_edit_member` to remove the method, then `ide_optimize_imports` to clean up.

All existing `toUni()` call sites throughout the file continue to compile — they now resolve to the package-level `QdrantFutures.toUni()` via the static import.

- [ ] **Step 2: Verify compilation after toUni migration**

Run `ide_diagnostics` on `ReactiveQdrantCbrCaseMemoryStore.java`. All `toUni()` calls should resolve. This is a safety checkpoint before making behavioral changes.

- [ ] **Step 3: Update `registerSchema()`**

Use `ide_replace_member` on `registerSchema`. Replace:

```java
@Override
public Uni<Void> registerSchema(CbrFeatureSchema schema) {
    schemas.put(schema.caseType(), schema);
    return collectionManager.registerSchemaIndexesAsync(schema, vectorDimension());
}
```

This removes the `Uni.createFrom().voidItem().invoke(...).runSubscriptionOn(workerPool)` pattern.

- [ ] **Step 4: Restructure `store()`**

Use `ide_replace_member` on `store`. The new implementation separates blocking prep (worker pool) from async collection ops (event loop):

```java
@Override
public Uni<String> store(CbrCase cbrCase, String caseType, String entityId,
                         MemoryDomain domain, String tenantId, String caseId,
                         io.casehub.platform.api.path.Path scope) {
    CbrFeatureSchema schema = schemas.get(caseType);
    if (schema != null) {
        CbrFeatureValidator.validateStoreFeatures(cbrCase.features(), schema);
    }

    return Uni.createFrom().item(() -> {
                  String mid = caseId;
                  if (delegate != null) {
                      MemoryInput memoryInput = CbrMemorySerializer.serialize(
                              cbrCase, entityId, domain, tenantId, caseId, caseType);
                      mid = delegate.store(memoryInput);
                  }

                  Embedding embedding = null;
                  if (embeddingModel != null) {
                      embedding = embeddingModel.embed(TextSegment.from(cbrCase.problem())).content();
                  }

                  Map<Integer, Float> sparseEmbedding = null;
                  if (sparseEmbedder != null && config.spladeEnabled() && cbrCase.problem() != null) {
                      sparseEmbedding = sparseEmbedder.embed(cbrCase.problem());
                  }

                  String bm25Text = null;
                  if (config.bm25Enabled() && cbrCase.problem() != null) {
                      bm25Text = CamelCaseExpander.expand(cbrCase.problem());
                  }

                  PointStruct point = CbrPointBuilder.buildPoint(
                          cbrCase, caseType, entityId, domain.name(), tenantId, caseId,
                          embedding, config.denseVectorName(),
                          sparseEmbedding, config.spladeVectorName(),
                          bm25Text, config.bm25VectorName(), config.bm25Model(), scope.value());

                  String collection = collectionManager.collectionName(caseType);
                  return new StoreContext(mid, collection, point);
              })
              .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
              .chain(ctx -> collectionManager.ensureCollectionAsync(caseType, vectorDimension())
                                             .replaceWith(ctx))
              .chain(ctx -> upsertWithRetry(ctx.collection(), List.of(ctx.point()), config.maxRetries())
                                    .replaceWith(ctx.memoryId()));
}
```

Key change: `ensureCollection` moved out of the `runSubscriptionOn` block and into a chained async step.

- [ ] **Step 5: Update `eraseFromAllCollections()`**

Use `ide_replace_member` on `eraseFromAllCollections`:

```java
private Uni<Integer> eraseFromAllCollections(Filter filter) {
    if (schemas.isEmpty()) {return Uni.createFrom().item(0);}
    return Multi.createFrom().iterable(schemas.keySet())
                .onItem().transformToUniAndConcatenate(caseType -> {
                String collection = collectionManager.collectionName(caseType);
                return toUni(collectionManager.client().collectionExistsAsync(collection))
                               .chain(exists -> {
                                   if (!exists) {return Uni.createFrom().item(0);}
                                   return collectionManager.deleteByFilterAsync(collection, filter);
                               });
            })
                .collect().asList()
                .map(counts -> counts.stream().mapToInt(Integer::intValue).sum());
}
```

Change: replaced `Uni.createFrom().item(() -> collectionManager.deleteByFilter(...)).runSubscriptionOn(workerPool)` with `collectionManager.deleteByFilterAsync(collection, filter)`.

- [ ] **Step 6: Update `purgeCollection()` — fixes pre-existing bug**

Use `ide_replace_member` on `purgeCollection`:

```java
private Uni<Integer> purgeCollection(String collection, CbrRetentionPolicy policy) {
    final Uni<Integer> ageUni;
    if (policy.maxAgeDays() != null) {
        long cutoffMillis = Instant.now().minus(java.time.Duration.ofDays(policy.maxAgeDays())).toEpochMilli();
        Filter ageFilter = Filter.newBuilder()
                                 .addMust(ConditionFactory.matchKeyword("tenantId", policy.tenantId()))
                                 .addMust(ConditionFactory.matchKeyword("domain", policy.domain().name()))
                                 .addMust(ConditionFactory.range("_stored_at",
                                                                 io.qdrant.client.grpc.Common.Range.newBuilder().setLt(cutoffMillis).build()))
                                 .build();
        ageUni = collectionManager.deleteByFilterAsync(collection, ageFilter);
    } else {
        ageUni = Uni.createFrom().item(0);
    }

    final Uni<Integer> countUni;
    if (policy.maxCasesPerType() != null) {
        Filter scopeFilter = Filter.newBuilder()
                                   .addMust(ConditionFactory.matchKeyword("tenantId", policy.tenantId()))
                                   .addMust(ConditionFactory.matchKeyword("domain", policy.domain().name()))
                                   .build();
        countUni = toUni(collectionManager.client().scrollAsync(
                io.qdrant.client.grpc.Points.ScrollPoints.newBuilder()
                                                         .setCollectionName(collection)
                                                         .setFilter(scopeFilter)
                                                         .setLimit(100000)
                                                         .setWithPayload(WithPayloadSelectorFactory.include(List.of("_stored_at")))
                                                         .build()))
                           .chain(scrollResult -> {
                               var points = new ArrayList<>(scrollResult.getResultList());
                               if (points.size() <= policy.maxCasesPerType()) {return Uni.createFrom().item(0);}
                               points.sort((a, b) -> {
                                   long aTime = a.getPayloadOrDefault("_stored_at", ValueFactory.value(0L)).getIntegerValue();
                                   long bTime = b.getPayloadOrDefault("_stored_at", ValueFactory.value(0L)).getIntegerValue();
                                   return Long.compare(bTime, aTime);
                               });
                               List<PointId> toDelete = new ArrayList<>();
                               for (int i = policy.maxCasesPerType(); i < points.size(); i++) {
                                   toDelete.add(points.get(i).getId());
                               }
                               return toUni(collectionManager.client().deleteAsync(collection, toDelete))
                                              .replaceWith(toDelete.size());
                           });
    } else {
        countUni = Uni.createFrom().item(0);
    }

    return ageUni.chain(ageCount -> countUni.map(countCount -> ageCount + countCount));
}
```

Change: `Uni.createFrom().item(() -> collectionManager.deleteByFilter(...))` (blocking, no `runSubscriptionOn` — the bug) → `collectionManager.deleteByFilterAsync(...)` (async, event-loop safe).

- [ ] **Step 7: Remove unused Infrastructure import if no longer needed**

Check if `Infrastructure.getDefaultWorkerPool()` is still used anywhere in the file (it is — `store()` still uses it for blocking embeddings). If still used, keep it. Use `ide_optimize_imports` to clean up any unused imports.

- [ ] **Step 8: Verify with ide_diagnostics**

Run `ide_diagnostics` on `ReactiveQdrantCbrCaseMemoryStore.java` to confirm no errors.

- [ ] **Step 9: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl memory-qdrant`
Expected: All tests pass — `QdrantCbrCaseMemoryStoreTest` (contract tests via Testcontainers), `CbrReconciliationServiceTest`, `QdrantFuturesTest`, `QdrantCbrDenseSearchTest`.

- [ ] **Step 10: Commit**

```bash
git add memory-qdrant/src/main/java/io/casehub/neocortex/memory/cbr/qdrant/ReactiveQdrantCbrCaseMemoryStore.java
git commit -m "feat(#165): ReactiveQdrantCbrCaseMemoryStore — use async CbrCollectionManager, fix purge event-loop bug"
```
