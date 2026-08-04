# neural-text deep-dive sync — Matryoshka + quantization

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update `docs/repos/casehub-neural-text.md` to reflect Matryoshka embeddings, dense quantization, search-time oversampling, and CDI wiring shipped in neural-text#31, plus fix stale class names from the #17 refactor.

**Architecture:** Seven edits to one file, applied in document order. Edit 7 (stale name fixes) is applied first since it touches lines that later edits reference or sit adjacent to.

**Tech Stack:** Markdown only — no code, no tests.

## Global Constraints

- Target file: `docs/repos/casehub-neural-text.md` in the parent repo
- Spec: `docs/superpowers/specs/2026-06-27-neural-text-deep-dive-matryoshka-quantization-design.md`
- All class names, config keys, and behavioral descriptions verified against neural-text source code
- ARC42STORIES references use full GitHub paths with section anchors
- Issue: casehubio/parent#315

---

### Task 1: Fix stale class names from #17 refactor (Edit 7)

Applied first because later tasks reference or sit adjacent to these lines.

**Files:**
- Modify: `docs/repos/casehub-neural-text.md` — lines 28, 30, 53, 55, 144

**Interfaces:**
- Consumes: nothing
- Produces: corrected class names that Tasks 2–5 reference

- [ ] **Step 1: Fix rag-api module row (line 28)**

In the Module Structure table, `rag-api/` row, change:

`CorpusStore SPI, CaseRetriever SPI (blocking); ReactiveCorpusStore, ReactiveCaseRetriever` → `EmbeddingIngestor SPI, CaseRetriever SPI (blocking); ReactiveEmbeddingIngestor, ReactiveCaseRetriever`

- [ ] **Step 2: Fix rag-testing module row (line 30)**

In the Module Structure table, `rag-testing/` row, change:

`In-memory CorpusStore + CaseRetriever` → `In-memory EmbeddingIngestor + CaseRetriever`

- [ ] **Step 3: Fix Key Abstractions header (line 53)**

Change: `### CorpusStore / CaseRetriever (rag-api)` → `### EmbeddingIngestor / CaseRetriever (rag-api)`

- [ ] **Step 4: Fix Key Abstractions body (line 55)**

Change: `CorpusStore — ingest, delete, and list documents per tenant corpus. Tenancy-scoped; CorpusRef carries tenant ID + corpus name.`

To: `EmbeddingIngestor — ingest pre-chunked text into vector store (embedding + storage), delete and list by source document. Tenancy-scoped; CorpusRef carries tenant ID + corpus name.`

- [ ] **Step 5: Fix C7 row (line 144)**

Change: `CorpusStore SPI, CaseRetriever SPI, QdrantCorpusStore, QdrantCaseRetriever in rag` → `EmbeddingIngestor SPI, CaseRetriever SPI, QdrantEmbeddingIngestor, HybridCaseRetriever in rag`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-neural-text.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: fix stale class names from #17 refactor in neural-text deep-dive (#315)"
```

---

### Task 2: Add Key Abstractions subsections (Edits 1 + 2)

**Files:**
- Modify: `docs/repos/casehub-neural-text.md` — after `### EmbeddingIngestor / CaseRetriever (rag-api)` subsection, before `### Corpus Ingestion Bridge`

**Interfaces:**
- Consumes: corrected header name from Task 1
- Produces: two new ### subsections + updated CaseRetriever paragraph

- [ ] **Step 1: Add ### MatryoshkaEmbeddingModel (rag) subsection**

Insert after the `### EmbeddingIngestor / CaseRetriever (rag-api)` subsection (after the CaseRetriever paragraph), before `### Corpus Ingestion Bridge`:

```markdown
### MatryoshkaEmbeddingModel (rag)

`MatryoshkaEmbeddingModel` — truncating `EmbeddingModel` decorator in `rag/`. Takes a delegate model and `targetDimension`, truncates the output vector to the first N dimensions and L2-renormalizes. Config-driven: active when `casehub.rag.matryoshka.dimension` is set. Reports `modelName()` as `delegate/matryoshka-N`. Validates that target dimension is positive and does not exceed delegate dimension.

The decorator pattern is architecturally significant: `dimension()` returns the truncated size, which flows transparently to `ensureCollection()` — collection vector dimensions are automatically correct without separate dimension tracking. See [`casehub-neural-text/ARC42STORIES.MD` §4](https://github.com/casehubio/neural-text/blob/main/ARC42STORIES.MD#4-solution-strategy) for the dual-vector tiered search alternative that was evaluated and rejected.
```

- [ ] **Step 2: Add ### DenseQuantization (rag) subsection**

Insert immediately after the MatryoshkaEmbeddingModel subsection:

```markdown
### DenseQuantization (rag)

`DenseQuantization` — enum in `rag/` with values `NONE`, `BINARY`, `SCALAR`. Configures Qdrant quantization on the **dense vector params** at collection creation time — applied to `denseParamsBuilder` specifically, not to the entire collection (sparse vectors are not quantized). `BINARY` applies `BinaryQuantization`; `SCALAR` applies `ScalarQuantization` with `Int8` type. Both respect `casehub.rag.quantization.always-ram` (default `true`). Config: `casehub.rag.quantization.type` (default `NONE`).

Named `DenseQuantization` rather than `QuantizationType` because the Qdrant client already defines `io.qdrant.client.grpc.Collections.QuantizationType` — both enums appear in `ensureCollection()` / `buildCreateRequest()` and sharing the name would create ambiguous unqualified usage (see [`casehub-neural-text/ARC42STORIES.MD` §8](https://github.com/casehubio/neural-text/blob/main/ARC42STORIES.MD#8-crosscutting-concepts)).
```

- [ ] **Step 3: Update CaseRetriever paragraph (Edit 2)**

In the `### EmbeddingIngestor / CaseRetriever (rag-api)` subsection, append to the existing `CaseRetriever` paragraph (after the reactive variant sentence):

```markdown
`HybridCaseRetriever` (and `ReactiveHybridCaseRetriever`) accept `DenseQuantization` type and optional oversampling. When quantization is active (`DenseQuantization != NONE`) and `casehub.rag.quantization.oversampling` is set, the dense prefetch leg applies `QuantizationSearchParams` with the configured oversampling factor + `rescore=true`. Compensates for quantization precision loss by fetching more candidates from the quantized index before rescoring against full-precision vectors. Sparse prefetch is unaffected — sparse vectors are not quantized. See [`casehub-neural-text/ARC42STORIES.MD` §6](https://github.com/casehubio/neural-text/blob/main/ARC42STORIES.MD#6-runtime-view) for the oversampling design rationale.
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-neural-text.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: add Matryoshka, DenseQuantization, and oversampling to neural-text deep-dive (#315)"
```

---

### Task 3: Update Module Structure table, Relationship to LangChain4j table, Current State, and Design Documents (Edits 3–6)

**Files:**
- Modify: `docs/repos/casehub-neural-text.md` — rag/ row (line 29), Relationship to LangChain4j table, C7 row (line 144), Design Documents section

**Interfaces:**
- Consumes: corrected C7 row from Task 1
- Produces: complete deep-dive update

- [ ] **Step 1: Update rag/ module row (Edit 3, line 29)**

Append to the existing rag/ module description text (before the closing `|`):

```
; `MatryoshkaEmbeddingModel` — truncating `EmbeddingModel` decorator (config-driven via `casehub.rag.matryoshka.dimension`); `DenseQuantization` enum — binary/scalar quantization config for Qdrant dense vector params; `RagBeanProducer` — CDI producer that conditionally wraps `EmbeddingModel` in `MatryoshkaEmbeddingModel` and passes quantization config to both `QdrantEmbeddingIngestor` (collection creation: type + alwaysRam) and `HybridCaseRetriever` (search-time: type + oversampling); `ReactiveRagBeanProducer` — same conditional Matryoshka wrapping and quantization wiring for reactive implementations (`ReactiveQdrantEmbeddingIngestor`, `ReactiveHybridCaseRetriever`), gated by `casehub.rag.reactive.enabled=true`.
```

- [ ] **Step 2: Add two rows to Relationship to LangChain4j table (Edit 4)**

Add after the existing `casehub-specific RAG wiring + tenancy` row:

```markdown
| Matryoshka dimension reduction + L2 renorm | `rag` (this module) — decorator above LangChain4j `EmbeddingModel` |
| Dense vector quantization (binary/scalar) + search-time oversampling | `rag` (this module) — Qdrant collection config + search params |
```

- [ ] **Step 3: Update C7 row in Current State (Edit 5)**

The C7 row (after Task 1's fix) reads: `EmbeddingIngestor SPI, CaseRetriever SPI, QdrantEmbeddingIngestor, HybridCaseRetriever in rag; rag-testing in-memory stubs`

Append: `; storage/search optimization (neural-text#31): MatryoshkaEmbeddingModel (truncating decorator), DenseQuantization (binary/scalar dense vector params config), search-time oversampling on quantized dense prefetch, RagBeanProducer / ReactiveRagBeanProducer CDI wiring — both blocking and reactive implementations carry all features`

- [ ] **Step 4: Add ARC42STORIES.MD to Design Documents (Edit 6)**

Add after the existing design spec entries (after line 152):

```markdown
- [casehubio/neural-text ARC42STORIES.MD](https://github.com/casehubio/neural-text/blob/main/ARC42STORIES.MD) — authoritative architecture record (Matryoshka §4, oversampling §6, dimension consistency §7, naming §8)
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-neural-text.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs: update module table, LangChain4j table, C7 row, and design docs in neural-text deep-dive (#315)"
```
