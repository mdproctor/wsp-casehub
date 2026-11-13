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

## D2: Entry type inventory — 16 types across three domains

**Choice:** 16 sealed record types grouped by lifecycle domain:
- **Derivation (4):** DerivationStarted, StageComputed, DataFlowed, DerivationCompleted — replaces DerivationObserver's 4 callbacks
- **Runtime tick (7):** AttentionBriefed, MoodShifted, DriveActivated, AppraisalRecorded, RegulationDecided, GoalDecided, CapsSettled — replaces PipelineTrace end-of-tick snapshot with root-to-tip entries at point of computation
- **Memory & consolidation (5):** MemoryRecalled, MemoryFormed, GraphMutated, Consolidated, ObservationCacheRefreshed — replaces MemoryRecallObserver, CompactionObserver, GraphMutationRecorded CDI event, ObservationCacheObserver

**Alternatives:**
- Fewer entry types (coarser) — e.g., single TickRecorded with nested data. Rejected: loses the root-to-tip causal chain between stages.
- More entry types (finer) — including narrative, strategy learning, reflection, per-subject mental model. Rejected for first cut: these don't represent discrete cognitive state changes. Additive — can be added to the sealed hierarchy later without changing the interface.

**Rationale:** Maps 1:1 to the stages in CognitionCore.tick() that produce meaningful cognitive state changes. Each entry captures inputs, outputs, and decision rationale at the point of computation. The 16-type inventory covers everything PipelineTrace captured (end-of-tick) plus 10 intermediate states it missed (mood deltas, drive modulation, attention briefing, goal decisions, etc.).

**Trade-offs:** Narrative/strategy/reflection stages excluded — these produce prompt text, not structured state changes. If explainability requires tracing prompt assembly, add entry types later.

**Depends on:** D1 (sealed entry hierarchy)
**Sources:** CognitionCore.tick() execution order (lines 237-351), PipelineTrace fields, existing leaf observers
**Exploration:** quick
**Status:** captured

## D3: Journal injection — CognitionCore field, passed to participants

**Choice:** CognitiveJournal as a field on CognitionCore (replacing pipelineSnapshotStore conceptually). CognitionCore passes the journal to participants during configuration. Each participant records entries at point of computation. For consolidation (async), ConsolidationScheduler receives the journal via CDI injection.

**Alternatives:**
- CDI injection into each participant independently — more decoupled, but harder to scope journal lifecycle (per-tick vs per-session). Rejected: CognitionCore already orchestrates the pipeline.
- Journal on tick context object — clean if such a context existed, but no first-class tick context flows through participants today. Creating one would be a larger refactor for no benefit.

**Rationale:** CognitionCore already holds pipelineSnapshotStore and passes participants during configuration. Same pattern, same wiring. Zero new mechanism needed.

**Depends on:** D1 (CognitiveJournal interface)
**Sources:** CognitionCore configuration methods, pipelineSnapshotStore field
**Exploration:** quick
**Status:** captured

## D4: Module placement — cognitive-observability-api + cognitive-observability-core

**Choice:** CognitiveEntry (sealed interface + all 16 record types) and CognitiveJournal (interface) in `cognitive-observability-api`. CognitiveTrace (in-memory accumulator) in `cognitive-observability-api`. Persistent journal store (replacing PipelineSnapshotStore) in `cognitive-observability-core`.

**Alternatives:**
- New module `cognitive-journal-api` — unnecessary module proliferation. Rejected: the observability-api module is exactly the right home.
- CognitiveTrace in core instead of api — possible, but DerivationTrace is already in api. Consistency wins.

**Rationale:** Follows existing module structure. DerivationObserver and DerivationTrace are in `cognitive-observability-api`. PipelineSnapshotStore is in `cognitive-observability-core`. Same split: interfaces + in-memory impl in api, persistence in core.

**Depends on:** D1 (CognitiveJournal interface), D2 (entry types)
**Sources:** cognitive-observability-api module structure, cognitive-observability-core module structure
**Exploration:** quick
**Status:** captured

## D5: eventRef format — standardized from #960 conventions

**Choice:** Standardize eventRef values established by #960:
- `"init"` — startup derivation
- `"tick:<N>"` — runtime tick N (e.g., `"tick:47"`)
- `"event:<id>"` — specific event trigger
- `"consolidation:tick:<N>"` — consolidation triggered by tick N context

All 16 entry types carry agentId + eventRef + timestamp as first-class fields via the sealed interface contract.

**Alternatives:**
- Structured eventRef object instead of string — type-safe but adds a class for minimal benefit. Pattern is simple enough that string conventions work. Rejected: YAGNI.

**Rationale:** #960 already uses these conventions. Standardizing them across all entry types is the consistency goal. String-based keeps serialization trivial (JSON, logs).

**Depends on:** D1 (correlation model)
**Sources:** DerivationObserver eventRef usage in #960
**Exploration:** quick
**Status:** captured
