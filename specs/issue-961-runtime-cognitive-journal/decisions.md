## D1: Unified journal architecture — single interface, sealed entry hierarchy

**Choice:** Single `CognitiveJournal` interface with one `record(CognitiveEntry)` method, sealed `CognitiveEntry` hierarchy covering all pipeline stages, and `CognitiveTrace` accumulator with unified query API.

**Alternatives:**
- Focused observer family (Approach B) — separate observer interfaces per domain (derivation, runtime, consolidation) with shared entry type. Non-breaking but creates multiple observer patterns that could drift. Rejected: competing implementations at tension.
- Monolith multi-method observer (Approach A) — single interface with 15+ callback methods. Unified but interface grows with every new stage, violates OCP. Rejected: single-method + sealed types is strictly better.
- Tick-scoped journal (Approach C) — journal flows through tick context. Doesn't cover derivation or consolidation lifecycles. Rejected: not truly unified.

**Rationale:** Seven existing observability mechanisms (DerivationObserver, PipelineTrace, MemoryRecallObserver, CompactionObserver, GraphMutationRecorded, ObservationCacheObserver, PipelineSnapshotStore) each capture partial data in different formats with different correlation models. A single journal interface with typed entries eliminates all tension. Single-method + sealed types means the interface never changes — new stages are additive (new record types). Java exhaustive pattern matching preserves compile-time safety at consumer level. NOOP is a lambda. Predicate filtering at the trace level is more flexible than per-observer filtering.

**Trade-offs:** Refactoring all seven existing mechanisms is a large blast radius. Every producer and consumer of the old APIs changes. Worth it for a clean, unified design with no competing implementations.

**Sources:** DerivationObserver (`cognitive-observability-api`), DerivationTrace, PipelineTrace (`cognition-api/.../trace/`), CognitionCore.tick() pipeline structure, #960 derivation journal pattern
**Exploration:** deep-analysis
**Status:** captured
