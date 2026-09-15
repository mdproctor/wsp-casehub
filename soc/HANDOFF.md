# HANDOFF — casehub-soc

**Date:** 2026-09-15
**Branch:** `issue-51-rag-investigation-enrichment`
**Epic:** #51 — RAG-powered investigation enrichment

---

## Last Session

Implemented #56 (ATT&CK STIX ingestion) end-to-end:

1. **Design review** — standard depth, 4 dimensions (coherence, structure, robustness, cross-cutting). 52 findings raised, 37 verified, 5 accepted, 0 unresolved. Major improvements: replaced volatile cache with MindMap alias system, explicit root node for version storage, crash recovery algorithm, error boundary on @Startup, RAG failure isolation. $50 total.

2. **Implementation** — 4 tasks across 3 batches, all complete:
   - `AttckStixParser` — pure-Java STIX 2.1 parser, filters deprecated/revoked objects (10 unit tests)
   - `AttckEnrichmentService` — stateless graph queries via MindMap aliases, no custom cache (6 unit tests)
   - `AttckIngestionService` — @Startup service populating MindMap subgraph + RAG corpus with version-stamped crash recovery (8 unit tests)
   - `RuleAttckMappingWorker` enhancement — enriched output with related groups, mitigations, sub-techniques. `AttckLookupTable` moved to `threatintel.attck` package. Wired through `SocInvestigationCaseDescriptor` and `SocCaseHub`.

3. **41 tests total** — all green. Used `InMemoryMindMapStore` + `InMemoryEmbeddingIngestor` (not @QuarkusTest — blocked by qhorus#438).

## Immediate Next Step

Run `work next` to advance the queue. #56 is complete — next child issue in epic #51 is ready.

## Deferred Items (in .plan)

- #60 — @QuarkusTest integration tests for ATT&CK ingestion (S / Med) — blocked by casehubio/qhorus#438
- #61 — Download ATT&CK Enterprise STIX bundle (XS / Low) — needed for deployment, not development

## Cross-Module

- casehubio/qhorus#438 — SNAPSHOT CDI deployment errors (blocks @QuarkusTest integration tests)
- casehubio/parent#470 — CBR-driven case definition selection (blocks playbook selection, not this epic)

## References

- `specs/issue-51-rag-investigation-enrichment/2026-09-15-attck-stix-ingestion-design.md`
- `specs/issue-51-rag-investigation-enrichment/decisions.md`
- `plans/2026-09-15-attck-stix-ingestion.md`
- `app/src/main/java/io/casehub/soc/threatintel/attck/` — all new ATT&CK classes
- `app/src/test/java/io/casehub/soc/threatintel/attck/` — all ATT&CK tests
