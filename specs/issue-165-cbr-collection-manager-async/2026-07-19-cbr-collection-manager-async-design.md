# CbrCollectionManager Async Methods — Design Spec

**Issue:** casehubio/neocortex#165
**Date:** 2026-07-19
**Status:** Approved

## Problem

`CbrCollectionManager` wraps natively-async Qdrant gRPC calls (`ListenableFuture`) with blocking `.get()`. `ReactiveQdrantCbrCaseMemoryStore` then wraps these blocking calls with `runSubscriptionOn(workerPool)` to avoid blocking the Vert.x event loop. This creates a double conversion: async → blocking → async, with unnecessary worker pool dispatch overhead.

Pre-existing bug: `purgeCollection()` calls `deleteByFilter` via `Uni.createFrom().item(() -> ...)` without `runSubscriptionOn` — blocks the event loop if called from a reactive context.

## Approach

**Async canonical, blocking convenience wrappers.** Async methods become the real implementation using `toUni()` to chain `ListenableFuture` calls natively. Blocking methods become trivial one-liners (`asyncVariant().await().indefinitely()`) for callers that run on worker threads (e.g. `CbrReconciliationService`).

## Changes

### 1. QdrantFutures utility (new file)

Package-private utility class in `io.casehub.neocortex.memory.cbr.qdrant`, matching the `rag` module's existing `QdrantFutures` pattern. Single static method:

```java
static <T> Uni<T> toUni(ListenableFuture<T> future)
```

Includes cancellation propagation (`em.onTermination(() -> future.cancel(false))`). Replaces the private static copy in `ReactiveQdrantCbrCaseMemoryStore`.

### 2. CbrCollectionManager — async canonical

Three public async methods, each returning `Uni`:

**`ensureCollectionAsync(String caseType, int vectorDimension)` → `Uni<Void>`**

Chains:
1. Synchronous fast-path: `knownCollections.contains(collection)` → `Uni.createFrom().voidItem()`
2. `toUni(collectionExistsAsync)` → branch:
   - Exists: `toUni(getCollectionInfoAsync)` → validate dimensions/sparse vectors → possibly `toUni(deleteCollectionAsync)` + recreate
   - Not exists: build `CreateCollection`, `toUni(createCollectionAsync)` → `createBasePayloadIndexesAsync`
3. `.invoke(() -> knownCollections.add(collection))`

Domain exceptions (`CbrDimensionMismatchException`, `CbrSparseVectorMigrationException`) propagate as Uni failures. No `InterruptedException`/`ExecutionException` handling needed — `toUni()` handles this.

**`registerSchemaIndexesAsync(CbrFeatureSchema schema, int vectorDimension)` → `Uni<Void>`**

Chains `ensureCollectionAsync()` then iterates schema fields, creating payload indexes via `toUni(createPayloadIndexAsync(...))` sequentially (concatenated Multi or chained Unis).

**`deleteByFilterAsync(String collection, Filter filter)` → `Uni<Integer>`**

Chains `toUni(scrollAsync)` → count result → if > 0 `toUni(deleteAsync)` → return count.

**Private helper:** `createBasePayloadIndexesAsync(String collection)` → `Uni<Void>` — iterates `BASE_KEYWORD_FIELDS` creating keyword indexes, then `_stored_at` float index.

**Blocking wrappers** (one-liners, for `CbrReconciliationService`):
```java
void ensureCollection(String caseType, int dim) {
    ensureCollectionAsync(caseType, dim).await().indefinitely();
}
int deleteByFilter(String collection, Filter filter) {
    return deleteByFilterAsync(collection, filter).await().indefinitely();
}
void registerSchemaIndexes(CbrFeatureSchema schema, int dim) {
    registerSchemaIndexesAsync(schema, dim).await().indefinitely();
}
```

### 3. ReactiveQdrantCbrCaseMemoryStore — remove worker pool dispatch

**`registerSchema()`** — direct async call, no `runSubscriptionOn`:
```java
schemas.put(schema.caseType(), schema);
return collectionManager.registerSchemaIndexesAsync(schema, vectorDimension());
```

**`store()`** — restructured into a pipeline separating blocking from async:
1. Worker pool: `delegate.store()`, `embeddingModel.embed()`, `sparseEmbedder.embed()`, `CamelCaseExpander.expand()` → intermediate record
2. Event loop: `collectionManager.ensureCollectionAsync()` (async, no pool needed)
3. Event loop: `CbrPointBuilder.buildPoint()` (pure CPU)
4. Event loop: `upsertWithRetry()` (async)

**`eraseFromAllCollections()`** — replace `Uni.createFrom().item(() -> deleteByFilter(...)).runSubscriptionOn(workerPool)` with `collectionManager.deleteByFilterAsync(collection, filter)`.

**`purgeCollection()`** — same replacement. Fixes the pre-existing bug where `deleteByFilter` ran on the calling thread without `runSubscriptionOn`.

### 4. Files unchanged

- `CbrReconciliationService.java` — blocking batch service, uses blocking convenience wrappers
- `QdrantCbrCaseMemoryStore.java` — thin delegate to reactive store, no direct CbrCollectionManager usage
- `QdrantCbrBeanProducer.java` — produces CbrCollectionManager, no API change

## Scope

| File | Change type |
|------|-------------|
| `QdrantFutures.java` | New — package-private `toUni()` utility |
| `CbrCollectionManager.java` | Modified — async methods canonical, blocking wrappers |
| `ReactiveQdrantCbrCaseMemoryStore.java` | Modified — use async methods, restructure `store()`, remove private `toUni()` |

## Testing

Existing `QdrantCbrCaseMemoryStoreTest` and `CbrReconciliationServiceTest` (Testcontainers) validate behavior end-to-end. The refactor is internal — no SPI or API changes. Tests should pass without modification. Any new async-specific edge cases (cancellation, error propagation) are covered by the existing contract test suite.

## Garden entries referenced

- **GE-20260629-0a321f:** `createCollectionAsync` is NOT idempotent — race condition in `ensureCollection` pre-exists and is preserved (not addressed by this issue)
- **GE-20260714-85bd9a:** Qdrant `scrollAsync` returns unmodifiable protobuf lists — `purgeCollection` already handles this with `new ArrayList<>(scrollResult.getResultList())`

## Follow-up

- **#166:** Reactive JPA backend for CbrCaseMemoryStore — eliminates the same double-conversion pattern in `memory-cbr-jpa`
