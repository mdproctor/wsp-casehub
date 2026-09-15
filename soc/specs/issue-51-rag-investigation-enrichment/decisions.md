## D1: ATT&CK ingestion module location

**Choice:** In SOC (app/) — SOC-specific ingestion service
**Alternatives:**
- In neocortex as a reusable module — but ATT&CK is SOC-specific; other apps have different threat models
- Separate repo (casehub-threatintel) — maximum isolation but deployment overhead for one pipeline
**Rationale:** ATT&CK is a SOC domain concern. SOC depends on mindmap-api + rag-api (already available). Keeps neocortex generic.
**Trade-offs:** Other CaseHub apps wanting STIX ingestion would need to duplicate or extract later.
**Sources:** neocortex exploration (MindMap API, CaseRetriever API), SOC worker architecture
**Exploration:** quick
**Status:** captured

## D2: ATT&CK data load strategy

**Choice:** Startup ingestion via @Startup CDI bean — check if ATT&CK subgraph exists, ingest from classpath STIX bundle if not, version-stamped for re-ingestion on updates
**Alternatives:**
- CLI/admin endpoint — more control but requires operator action
- Background async — app usable before ingestion but partial-readiness handling adds complexity
**Rationale:** One-time cost (~30s on first boot), automatic, no operator action needed. Version stamp ensures re-ingestion when ATT&CK data updates.
**Trade-offs:** Adds ~30s to first cold start. Subsequent starts skip if version matches.
**Sources:** neocortex MindMapStore.createSubgraph(), EmbeddingIngestor.ingest()
**Exploration:** quick
**Status:** captured

## D3: AttckLookupTable migration strategy

**Choice:** Keep both — AttckLookupTable stays for fast deterministic mapping, RuleAttckMappingWorker ALSO queries MindMap for related techniques, groups, and mitigations to enrich its output
**Alternatives:**
- Replace with MindMap — more consistent but adds latency and hard dependency on ingestion
- Keep separate, don't touch — simplest but misses enrichment opportunity
**Rationale:** No breaking change to existing pipeline. The static table handles the fast path; MindMap provides the context that makes the output richer (related groups, mitigations, sub-techniques).
**Trade-offs:** Two sources of ATT&CK truth. Static table could drift from MindMap if not kept in sync.
**Depends on:** D2 (startup ingestion must populate MindMap before workers fire)
**Sources:** AttckLookupTable.java, RuleAttckMappingWorker.java, MindMapStore.neighbors()
**Exploration:** quick
**Status:** captured
