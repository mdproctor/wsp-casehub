# Per-Leg Embedding API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — feat: per-leg embedding separation — dense uses searchText(), sparse/BM25 use text()
**Issue group:** #117

**Goal:** Move per-leg embedding routing from a retriever-level workaround into the `MultiModalEmbedder` contract, preventing silent regression and enabling per-implementation optimization.

**Architecture:** Two new default methods on `MultiModalEmbedder` (`embed(Map<EmbeddingMode, String>)` and `embedSeparate(String, String)`). `SeparateModelEmbedder` overrides `embed(Map)` for optimized per-model routing (2 calls not 4). Retrievers collapse from a 7-line branching workaround to a single `embedSeparate` call.

**Tech Stack:** Java 21, Quarkus 3.32.2, LangChain4j 1.14.1, ONNX Runtime JVM, Qdrant

## Global Constraints

- `inference-api` is zero-dependency (no Quarkus, no LangChain4j) — default methods on `MultiModalEmbedder` must use only JDK types
- `EmbeddingMode` has exactly three values: `DENSE`, `SPARSE`, `COLBERT`
- `MultiModalEmbedding.dense()` is non-null — `DENSE` must always be present in any `embed(Map)` call
- Parameter naming uses embedding-mode vocabulary (`denseText` / `nonDenseText`), not retrieval vocabulary
- `embedBatch(List<String>)` is unchanged — ingestion callers keep using it

---

### Task 1: MultiModalEmbedder API + implementation overrides

**Files:**
- Modify: `inference-api/src/main/java/io/casehub/neocortex/inference/MultiModalEmbedder.java` (add 2 default methods)
- Modify: `rag/src/main/java/io/casehub/neocortex/rag/runtime/SeparateModelEmbedder.java` (add `embed(Map)` override)
- Modify: `inference-api/src/main/java/io/casehub/neocortex/inference/MatryoshkaMultiModalEmbedder.java` (add `embed(Map)` override)
- Test: `rag/src/test/java/io/casehub/neocortex/rag/runtime/SeparateModelEmbedderTest.java` (add 1 test)
- Test: `inference-api/src/test/java/io/casehub/neocortex/inference/MatryoshkaMultiModalEmbedderTest.java` (add 1 test)

**Interfaces:**
- Produces: `MultiModalEmbedder.embed(Map<EmbeddingMode, String>)` — default method, composes result from per-mode texts
- Produces: `MultiModalEmbedder.embedSeparate(String denseText, String nonDenseText)` — convenience, delegates to `embed(Map)`

- [ ] **Step 1: Add default methods to `MultiModalEmbedder`**

Use `ide_insert_member` to add after the existing `embedBatch` method (line 28).

```java
/**
 * Embed with per-mode text targeting. Each mode can use a different input text.
 * <p>
 * The map must contain {@link EmbeddingMode#DENSE} (dense is always required).
 * Modes not in the map produce null in the result. Modes in the map that the
 * embedder does not support are silently ignored.
 *
 * @param textsByMode mode → input text mapping
 * @return composed embedding with each signal from its designated text
 */
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

/**
 * Embed with dense/non-dense text separation. Dense uses {@code denseText};
 * all other supported modes use {@code nonDenseText}.
 * <p>
 * Short-circuits to {@link #embed(String)} when both texts are equal.
 *
 * @param denseText    text for the DENSE embedding
 * @param nonDenseText text for all non-DENSE modes (SPARSE, COLBERT)
 * @return composed embedding
 */
default MultiModalEmbedding embedSeparate(String denseText, String nonDenseText) {
    Objects.requireNonNull(denseText, "denseText must not be null");
    Objects.requireNonNull(nonDenseText, "nonDenseText must not be null");
    if (denseText.equals(nonDenseText)) return embed(denseText);
    var map = new EnumMap<>(EmbeddingMode.class);
    map.put(EmbeddingMode.DENSE, denseText);
    for (EmbeddingMode mode : supportedModes()) {
        if (mode != EmbeddingMode.DENSE) {
            map.put(mode, nonDenseText);
        }
    }
    return embed(map);
}
```

Imports to add: `java.util.EnumMap`, `java.util.LinkedHashMap`, `java.util.Map`, `java.util.Objects`.

- [ ] **Step 2: Build `inference-api` to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl inference-api compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 3: Write failing test for `SeparateModelEmbedder.embed(Map)` routing**

Use `ide_insert_member` in `SeparateModelEmbedderTest` after `embedBatch` (line 115).

```java
@Test
void embedMapRoutesDenseAndSparseToSeparateModels() {
    List<String> denseTexts = new ArrayList<>();
    EmbeddingModel denseModel = new EmbeddingModel() {
        @Override
        public Response<Embedding> embed(String text) {
            denseTexts.add(text);
            return Response.from(Embedding.from(new float[]{1f, 2f}));
        }
        @Override
        public Response<List<Embedding>> embedAll(List<TextSegment> segments) {
            return Response.from(segments.stream()
                .map(s -> { denseTexts.add(s.text()); return Embedding.from(new float[]{1f, 2f}); })
                .toList());
        }
        @Override
        public int dimension() { return 2; }
    };

    List<String> sparseTexts = new ArrayList<>();
    InferenceModel sparseModel = new InferenceModel() {
        @Override
        public InferenceOutput run(InferenceInput input) {
            sparseTexts.add(input.text());
            return InferenceOutput.of(new float[]{0f, 0.5f, 0f, 2.0f});
        }
        @Override
        public List<InferenceOutput> runBatch(List<InferenceInput> inputs) {
            return inputs.stream().map(this::run).toList();
        }
        @Override
        public void close() {}
    };
    SparseEmbedder sparse = new SparseEmbedder(sparseModel);
    SeparateModelEmbedder embedder = new SeparateModelEmbedder(denseModel, sparse, 512);

    MultiModalEmbedding result = embedder.embed(Map.of(
        EmbeddingMode.DENSE, "dense-text",
        EmbeddingMode.SPARSE, "sparse-text"));

    assertArrayEquals(new float[]{1f, 2f}, result.dense(), 1e-5f);
    assertNotNull(result.sparse());
    assertEquals(List.of("dense-text"), denseTexts);
    assertEquals(List.of("sparse-text"), sparseTexts);
}
```

Imports to add: `java.util.ArrayList`, `java.util.Map`.

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl rag test -Dtest="SeparateModelEmbedderTest#embedMapRoutesDenseAndSparseToSeparateModels" -q`
Expected: FAIL — default `embed(Map)` calls `embed(String)` which runs both models for both texts; `denseTexts` will contain `["dense-text", "sparse-text"]`, not `["dense-text"]`.

- [ ] **Step 5: Implement `SeparateModelEmbedder.embed(Map)` override**

Use `ide_insert_member` after `embed(String)` method (line 49).

```java
@Override
public MultiModalEmbedding embed(Map<EmbeddingMode, String> textsByMode) {
    String denseText = Objects.requireNonNull(
        textsByMode.get(EmbeddingMode.DENSE), "DENSE text is required");
    float[] dense = denseModel.embed(denseText).content().vector();

    Map<Integer, Float> sparse = null;
    if (sparseEmbedder != null && textsByMode.containsKey(EmbeddingMode.SPARSE)) {
        sparse = sparseEmbedder.embed(textsByMode.get(EmbeddingMode.SPARSE));
    }

    return new MultiModalEmbedding(dense, sparse, null);
}
```

Imports to add to `SeparateModelEmbedder.java`: `java.util.Map`, `io.casehub.neocortex.inference.EmbeddingMode`.

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl rag test -Dtest="SeparateModelEmbedderTest#embedMapRoutesDenseAndSparseToSeparateModels" -q`
Expected: PASS

- [ ] **Step 7: Add `MatryoshkaMultiModalEmbedder.embed(Map)` override**

Use `ide_insert_member` after `embedBatch` method (line 44).

```java
@Override
public MultiModalEmbedding embed(Map<EmbeddingMode, String> textsByMode) {
    return truncateAndRenormalize(delegate.embed(textsByMode));
}
```

Import to add: `java.util.Map`, `io.casehub.neocortex.inference.EmbeddingMode` (EmbeddingMode may already be imported — check first).

- [ ] **Step 8: Write test for Matryoshka delegation of `embed(Map)`**

Use `ide_insert_member` in `MatryoshkaMultiModalEmbedderTest` after `batchTruncatesAll` (line 86).

```java
@Test
void embedMapDelegatesToWrappedThenTruncates() {
    MultiModalEmbedder delegate = stubEmbedder(
        new float[]{1f, 2f, 3f, 4f}, null, null, 4);
    var matryoshka = new MatryoshkaMultiModalEmbedder(delegate, 2);

    MultiModalEmbedding result = matryoshka.embed(
        Map.of(EmbeddingMode.DENSE, "test"));

    assertEquals(2, result.dense().length);
}
```

Import to add: `java.util.Map` (if not already present).

- [ ] **Step 9: Run all unit tests in both modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl inference-api,rag test -q`
Expected: All tests PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/31/neocortex add \
  inference-api/src/main/java/io/casehub/neocortex/inference/MultiModalEmbedder.java \
  inference-api/src/main/java/io/casehub/neocortex/inference/MatryoshkaMultiModalEmbedder.java \
  inference-api/src/test/java/io/casehub/neocortex/inference/MatryoshkaMultiModalEmbedderTest.java \
  rag/src/main/java/io/casehub/neocortex/rag/runtime/SeparateModelEmbedder.java \
  rag/src/test/java/io/casehub/neocortex/rag/runtime/SeparateModelEmbedderTest.java
```
Message: `feat(#117): add embed(Map) and embedSeparate to MultiModalEmbedder`

---

### Task 2: Retriever simplification + test updates

**Files:**
- Modify: `rag/src/test/java/io/casehub/neocortex/rag/runtime/RagTestFixtures.java:128-200` (stub call tracking)
- Modify: `rag/src/test/java/io/casehub/neocortex/rag/runtime/HybridCaseRetrieverTest.java:304-355` (update 2 tests)
- Modify: `rag/src/test/java/io/casehub/neocortex/rag/runtime/ReactiveHybridCaseRetrieverTest.java:220-273` (update 2 tests)
- Modify: `rag/src/main/java/io/casehub/neocortex/rag/runtime/HybridCaseRetriever.java:70-209,259-333` (simplify retrieve + CC fusion)
- Modify: `rag/src/main/java/io/casehub/neocortex/rag/runtime/ReactiveHybridCaseRetriever.java:72-107,109-215,263-370` (simplify retrieve + executeRetrieve + CC fusion)

**Interfaces:**
- Consumes: `MultiModalEmbedder.embedSeparate(String denseText, String nonDenseText)` from Task 1

- [ ] **Step 1: Add `embedSeparate` call tracking to `StubMultiModalEmbedder`**

In `RagTestFixtures.StubMultiModalEmbedder`:

Use `ide_insert_member` to add a field after `batchCalls` (line 133):
```java
private final List<List<String>> separateCalls = new ArrayList<>();
```

Use `ide_insert_member` to add method after `embedBatch` (line 154):
```java
@Override
public MultiModalEmbedding embedSeparate(String denseText, String nonDenseText) {
    separateCalls.add(List.of(denseText, nonDenseText));
    return MultiModalEmbedder.super.embedSeparate(denseText, nonDenseText);
}
```

Use `ide_insert_member` to add accessor after `batchCalls()` (line 162):
```java
List<List<String>> separateCalls() {
    return List.copyOf(separateCalls);
}
```

Use `ide_replace_member` to update `clearCalls` (line 164):
```java
void clearCalls() {
    embedCalls.clear();
    batchCalls.clear();
    separateCalls.clear();
}
```

- [ ] **Step 2: Update `HybridCaseRetrieverTest` — rewrite the two expansion tests**

Use `ide_edit_member` to replace `usesEmbedBatchWhenExpansionActive` (line 304):

```java
@Test
void usesEmbedSeparateWhenExpansionActive() {
    RagTestFixtures.StubMultiModalEmbedder stub = RagTestFixtures.stubEmbedder(DENSE_DIM, true);

    CorpusRef corpus = uniqueCorpus();
    var ingestor = new QdrantEmbeddingIngestor(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());
    ingestor.ingest(corpus, List.of(
        new ChunkInput("test content", "doc-1", Map.of())));

    var retriever = new HybridCaseRetriever(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());

    stub.clearCalls();

    var expandedQuery = RetrievalQuery.of("original").withExpansion("hypothetical");
    retriever.retrieve(expandedQuery, corpus, 10, null);

    assertThat(stub.separateCalls()).hasSize(1);
    assertThat(stub.separateCalls().get(0)).containsExactly("hypothetical", "original");
    assertThat(stub.batchCalls()).isEmpty();
    assertThat(stub.embedCalls()).isEmpty();
}
```

Use `ide_edit_member` to replace `usesSingleEmbedWhenNoExpansion` (line 332):

```java
@Test
void usesEmbedSeparateWithSameTextWhenNoExpansion() {
    RagTestFixtures.StubMultiModalEmbedder stub = RagTestFixtures.stubEmbedder(DENSE_DIM, true);

    CorpusRef corpus = uniqueCorpus();
    var ingestor = new QdrantEmbeddingIngestor(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());
    ingestor.ingest(corpus, List.of(
        new ChunkInput("test content", "doc-1", Map.of())));

    var retriever = new HybridCaseRetriever(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());

    stub.clearCalls();

    retriever.retrieve(RetrievalQuery.of("original"), corpus, 10, null);

    assertThat(stub.separateCalls()).hasSize(1);
    assertThat(stub.separateCalls().get(0)).containsExactly("original", "original");
    assertThat(stub.batchCalls()).isEmpty();
}
```

- [ ] **Step 3: Run the updated tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl rag test -Dtest="HybridCaseRetrieverTest#usesEmbedSeparateWhenExpansionActive+usesEmbedSeparateWithSameTextWhenNoExpansion" -q`
Expected: FAIL — retrievers still call `embedBatch`/`embed`, not `embedSeparate`.

- [ ] **Step 4: Simplify `HybridCaseRetriever.retrieve()`**

Use `ide_replace_member` on `retrieve` method. The key changes:

1. Replace the 7-line embedBatch/embed branch (lines 88-99) with:
```java
MultiModalEmbedding embedding = embedder.embedSeparate(query.searchText(), query.text());
```

2. Replace all `searchTextEmbedding` references with `embedding` (for `.dense()`)
3. Replace all `originalTextEmbedding` references with `embedding` (for `.sparse()`, `.colbert()`)
4. Update `executeConvexCombinationFusion` call to pass one embedding instead of two

Use `ide_edit_member` on `executeConvexCombinationFusion` to change its signature from:
```java
private List<RetrievedChunk> executeConvexCombinationFusion(
        String collection, RetrievalQuery query, MultiModalEmbedding searchTextEmbedding,
        MultiModalEmbedding originalTextEmbedding, Optional<Filter> mergedFilter, int maxResults)
```
to:
```java
private List<RetrievedChunk> executeConvexCombinationFusion(
        String collection, RetrievalQuery query, MultiModalEmbedding embedding,
        Optional<Filter> mergedFilter, int maxResults)
```

Inside, replace `searchTextEmbedding.dense()` → `embedding.dense()` and `originalTextEmbedding.sparse()` → `embedding.sparse()`.

- [ ] **Step 5: Run `HybridCaseRetrieverTest` to verify updated tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl rag test -Dtest="HybridCaseRetrieverTest" -q`
Expected: All 12 tests PASS (including the renamed tests and all existing integration tests)

- [ ] **Step 6: Update `ReactiveHybridCaseRetrieverTest` — rewrite the two expansion tests**

Use `ide_edit_member` to replace `usesEmbedBatchWhenExpansionActive` (line 220):

```java
@Test
void usesEmbedSeparateWhenExpansionActive() {
    RagTestFixtures.StubMultiModalEmbedder stub = RagTestFixtures.stubEmbedder(DENSE_DIM, true);

    CorpusRef corpus = uniqueCorpus();
    var ingestor = new QdrantEmbeddingIngestor(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());
    ingestor.ingest(corpus, List.of(
        new ChunkInput("test content", "doc-1", Map.of())));

    var retriever = new ReactiveHybridCaseRetriever(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());

    stub.clearCalls();

    var expandedQuery = RetrievalQuery.of("original").withExpansion("hypothetical");
    retriever.retrieve(expandedQuery, corpus, 10, null)
        .await().indefinitely();

    assertThat(stub.separateCalls()).hasSize(1);
    assertThat(stub.separateCalls().get(0)).containsExactly("hypothetical", "original");
    assertThat(stub.batchCalls()).isEmpty();
    assertThat(stub.embedCalls()).isEmpty();
}
```

Use `ide_edit_member` to replace `usesSingleEmbedWhenNoExpansion` (line 249):

```java
@Test
void usesEmbedSeparateWithSameTextWhenNoExpansion() {
    RagTestFixtures.StubMultiModalEmbedder stub = RagTestFixtures.stubEmbedder(DENSE_DIM, true);

    CorpusRef corpus = uniqueCorpus();
    var ingestor = new QdrantEmbeddingIngestor(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());
    ingestor.ingest(corpus, List.of(
        new ChunkInput("test content", "doc-1", Map.of())));

    var retriever = new ReactiveHybridCaseRetriever(client, stub,
        TenantGuard.of(RagTestFixtures.stubPrincipal(TENANT)),
        RagTestFixtures.stubConfig());

    stub.clearCalls();

    retriever.retrieve(RetrievalQuery.of("original"), corpus, 10, null)
        .await().indefinitely();

    assertThat(stub.separateCalls()).hasSize(1);
    assertThat(stub.separateCalls().get(0)).containsExactly("original", "original");
    assertThat(stub.batchCalls()).isEmpty();
}
```

- [ ] **Step 7: Simplify `ReactiveHybridCaseRetriever`**

Same pattern as Step 4 but for the reactive variant.

In `retrieve()` (lines 72-107): replace the `if/else` embedBatch/embed branch with:
```java
return Uni.createFrom().item(() -> embedder.embedSeparate(query.searchText(), query.text()))
    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
    .chain(embedding -> executeRetrieve(query, collection, mergedFilter, embedding, maxResults));
```

Update `executeRetrieve` signature: drop one `MultiModalEmbedding` parameter (from two to one). Replace `searchTextEmbedding`/`originalTextEmbedding` references with `embedding`.

Update `executeConvexCombinationFusion` signature: same as blocking — one embedding instead of two. Update all `.dense()` / `.sparse()` references.

- [ ] **Step 8: Run `ReactiveHybridCaseRetrieverTest` to verify all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl rag test -Dtest="ReactiveHybridCaseRetrieverTest" -q`
Expected: All 9 tests PASS

- [ ] **Step 9: Run full module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -pl inference-api,rag test -q`
Expected: All tests PASS across both modules

- [ ] **Step 10: Run `ide_diagnostics` on all modified files**

Check each modified file for compilation errors/warnings:
- `inference-api/.../MultiModalEmbedder.java`
- `inference-api/.../MatryoshkaMultiModalEmbedder.java`
- `rag/.../SeparateModelEmbedder.java`
- `rag/.../HybridCaseRetriever.java`
- `rag/.../ReactiveHybridCaseRetriever.java`

Expected: No errors

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/31/neocortex add \
  rag/src/main/java/io/casehub/neocortex/rag/runtime/HybridCaseRetriever.java \
  rag/src/main/java/io/casehub/neocortex/rag/runtime/ReactiveHybridCaseRetriever.java \
  rag/src/test/java/io/casehub/neocortex/rag/runtime/RagTestFixtures.java \
  rag/src/test/java/io/casehub/neocortex/rag/runtime/HybridCaseRetrieverTest.java \
  rag/src/test/java/io/casehub/neocortex/rag/runtime/ReactiveHybridCaseRetrieverTest.java
```
Message: `feat(#117): replace embedBatch workaround with embedSeparate in retrievers`
