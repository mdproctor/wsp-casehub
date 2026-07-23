# Per-Leg Embedding Separation — API Contract

**Issue:** casehubio/neural-text#117
**Epic:** casehubio/neural-text#115 (regression-free query expansion)
**Date:** 2026-07-23
**Status:** Design approved

## Problem

`HybridCaseRetriever` routes different texts to different embedding modalities at query time: dense uses `searchText()` (expanded/hypothetical), sparse/ColBERT use `text()` (original query). This per-leg separation was implemented under #113 as a retriever-level workaround using `embedBatch([searchText, text])` with cherry-picked results.

The workaround has two issues:

1. **Compute waste** — `SeparateModelEmbedder.embedBatch([searchText, text])` runs both texts through both models (dense + sparse), producing 4 model calls when 2 suffice. The retriever discards `dense(text)` and `sparse(searchText)`.

2. **Layering violation** — the retriever knows which text to use for which embedding modality. That's embedder-internal knowledge that should be encapsulated in the `MultiModalEmbedder` contract.

This issue makes the separation contractual at the embedder API level, preventing silent regression and enabling per-implementation optimization.

## Design

### API change: `MultiModalEmbedder` (inference-api)

Two new default methods. Zero breaking changes — existing implementations inherit working defaults.

```java
default MultiModalEmbedding embed(Map<EmbeddingMode, String> textsByMode) {
    Objects.requireNonNull(textsByMode.get(EmbeddingMode.DENSE),
        "DENSE text is required");
    Map<String, MultiModalEmbedding> cache = new LinkedHashMap<>();
    for (String text : textsByMode.values()) {
        cache.computeIfAbsent(text, this::embed);
    }
    float[] dense = cache.get(textsByMode.get(EmbeddingMode.DENSE)).dense();
    var sparseText = textsByMode.get(EmbeddingMode.SPARSE);
    var colbertText = textsByMode.get(EmbeddingMode.COLBERT);
    return new MultiModalEmbedding(
        dense,
        sparseText != null ? cache.get(sparseText).sparse() : null,
        colbertText != null ? cache.get(colbertText).colbert() : null);
}

default MultiModalEmbedding embedSeparate(String denseText, String sparseText) {
    if (denseText.equals(sparseText)) return embed(denseText);
    var map = new EnumMap<>(EmbeddingMode.class);
    map.put(EmbeddingMode.DENSE, denseText);
    if (supportedModes().contains(EmbeddingMode.SPARSE))
        map.put(EmbeddingMode.SPARSE, sparseText);
    if (supportedModes().contains(EmbeddingMode.COLBERT))
        map.put(EmbeddingMode.COLBERT, sparseText);
    return embed(map);
}
```

**Method hierarchy:** `embedSeparate` → `embed(Map)` → `embed(String)`. The leaf `embed(String)` stays the implementation point all existing embedders override.

**Parameter naming:** `denseText` / `sparseText` use embedding-mode vocabulary, not retrieval vocabulary. The interface lives in `inference-api` — no retrieval concepts leak.

**Unsupported modes:** Modes in the map that the embedder doesn't support are silently ignored — the composed result has `null` for those signals. Consistent with `embed(String)` where e.g. `SeparateModelEmbedder` returns `null` for ColBERT.

### Implementation overrides

**`SeparateModelEmbedder`** — overrides `embed(Map)` to route each mode's text to only the model that produces it:

```java
@Override
public MultiModalEmbedding embed(Map<EmbeddingMode, String> textsByMode) {
    String denseText = textsByMode.get(EmbeddingMode.DENSE);
    float[] dense = denseModel.embed(denseText).content().vector();

    Map<Integer, Float> sparse = null;
    if (sparseEmbedder != null && textsByMode.containsKey(EmbeddingMode.SPARSE)) {
        sparse = sparseEmbedder.embed(textsByMode.get(EmbeddingMode.SPARSE));
    }

    return new MultiModalEmbedding(dense, sparse, null);
}
```

2 model calls instead of 4 when texts differ.

**`MatryoshkaMultiModalEmbedder`** — delegates to wrapped embedder then truncates:

```java
@Override
public MultiModalEmbedding embed(Map<EmbeddingMode, String> textsByMode) {
    return truncateAndRenormalize(delegate.embed(textsByMode));
}
```

Without this, the default would call `this.embed(String)` — bypassing `SeparateModelEmbedder`'s optimized override when Matryoshka wraps it.

**`BgeM3Embedder`** — no override. Default groups by unique text, calls `embed(String)` per group. ONNX computes all output tensors in a single pass — 2 model runs is already optimal.

### Retriever simplification

**`HybridCaseRetriever.retrieve()`** — the 7-line branching workaround collapses:

```java
// Before: two variables, conditional embedBatch vs embed
MultiModalEmbedding searchTextEmbedding;
MultiModalEmbedding originalTextEmbedding;
if (query.expandedText() != null) {
    List<MultiModalEmbedding> embeddings = embedder.embedBatch(
        List.of(query.searchText(), query.text()));
    searchTextEmbedding = embeddings.get(0);
    originalTextEmbedding = embeddings.get(1);
} else {
    searchTextEmbedding = embedder.embed(query.searchText());
    originalTextEmbedding = searchTextEmbedding;
}

// After: one call, one variable
MultiModalEmbedding embedding = embedder.embedSeparate(query.searchText(), query.text());
```

Two variables (`searchTextEmbedding`, `originalTextEmbedding`) become one (`embedding`). All downstream references simplify — `searchTextEmbedding.dense()` → `embedding.dense()`, `originalTextEmbedding.sparse()` → `embedding.sparse()`.

**`executeConvexCombinationFusion`** — drops a parameter. Currently takes two `MultiModalEmbedding` args, becomes one.

**`ReactiveHybridCaseRetriever`** — same collapse. The `if/else` branch in `retrieve()` becomes a single `embedSeparate` call. `executeRetrieve` drops from two `MultiModalEmbedding` parameters to one.

**BM25 path** — unchanged. Uses `query.text()` directly via `CamelCaseExpander`.

### Test changes

**`StubMultiModalEmbedder`** — adds `embedSeparate` call tracking:

```java
private final List<List<String>> separateCalls = new ArrayList<>();

@Override
public MultiModalEmbedding embedSeparate(String denseText, String sparseText) {
    separateCalls.add(List.of(denseText, sparseText));
    return MultiModalEmbedder.super.embedSeparate(denseText, sparseText);
}
```

**`HybridCaseRetrieverTest`** — two test renames:
- `usesEmbedSeparateWhenExpansionActive` — asserts `separateCalls` = `[["hypothetical", "original"]]`, no embedBatch calls
- `usesEmbedSeparateWithSameTextWhenNoExpansion` — asserts `separateCalls` = `[["original", "original"]]`

**`SeparateModelEmbedderTest`** — new test: verifies `embed(Map.of(DENSE, "textA", SPARSE, "textB"))` routes each text to only its model.

**`ReactiveHybridCaseRetrieverTest`** — mirrors blocking test changes.

**Unchanged:** all existing integration tests (ingest+retrieve roundtrips, payload filters, tenancy, quantization, BM25, three-way RRF).

## Scope

### In scope
- `MultiModalEmbedder` interface: 2 new default methods
- `SeparateModelEmbedder`: override `embed(Map)` for per-model routing
- `MatryoshkaMultiModalEmbedder`: override `embed(Map)` for delegation
- `HybridCaseRetriever` + `ReactiveHybridCaseRetriever`: replace embedBatch workaround with `embedSeparate`
- `StubMultiModalEmbedder`: call tracking
- Test updates in both retriever test classes + new `SeparateModelEmbedderTest` case

### Out of scope
- Ingestion path (`QdrantEmbeddingIngestor`) — all modalities from same text, no separation needed
- `BgeM3Embedder` — default works optimally
- BM25 path — uses `query.text()` directly, not through embedder
- Query expansion logic (`rag-expansion`) — unchanged, still sets `expandedText` on `RetrievalQuery`

## Files changed

| Module | File | Change |
|--------|------|--------|
| inference-api | `MultiModalEmbedder.java` | +2 default methods |
| rag | `SeparateModelEmbedder.java` | +override `embed(Map)` |
| inference-api | `MatryoshkaMultiModalEmbedder.java` | +override `embed(Map)` |
| rag | `HybridCaseRetriever.java` | Simplify embedding calls |
| rag | `ReactiveHybridCaseRetriever.java` | Simplify embedding calls |
| rag (test) | `RagTestFixtures.java` | Stub call tracking |
| rag (test) | `HybridCaseRetrieverTest.java` | Update 2 tests |
| rag (test) | `ReactiveHybridCaseRetrieverTest.java` | Update 2 tests |
| rag (test) | `SeparateModelEmbedderTest.java` | +1 new test |
