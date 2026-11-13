## D1: Unified journal architecture — single interface, sealed entry hierarchy

**Choice:** Single `CognitiveJournal` interface with one `record(CognitiveEntry)` method, sealed `CognitiveEntry` hierarchy covering all pipeline stages, and `CognitiveTrace` accumulator with unified query API.

**Alternatives:**
- Focused observer family (Approach B) — separate observer interfaces per domain (derivation, runtime, consolidation) with shared entry type. Non-breaking but creates multiple observer patterns that could drift. Rejected: competing implementations at tension.
- Monolith multi-method observer (Approach A) — single interface with 15+ callback methods. Unified but interface grows with every new stage, violates OCP. Rejected: single-method + sealed types is strictly better.
- Tick-scoped journal (Approach C) — journal flows through tick context. Doesn't cover derivation or consolidation lifecycles. Rejected: not truly unified.

**Rationale:** Five existing observability mechanisms (DerivationObserver, PipelineTrace/MemoryRecallObserver, GraphMutationRecorded, PipelineSnapshotStore) each capture partial cognitive data in different formats with different correlation models. (CompactionObserver and ObservationCacheObserver exist in cognitive-observability-api but have zero references anywhere — dead code to be deleted, not unified.) GraphMutationRecorded is a CDI event (fire-and-forget via `@Observes`) rather than an observer interface; the journal subsumes its observability role (the `GraphMutated` entry type captures the same mutation data for journal queries and correlation). The CDI event mechanism continues unchanged — `MutationTrackingDecorator` keeps firing `GraphMutationRecorded` events for domain-logic consumers (`EntityErasureTokenHandler`, `EntityMergeTokenHandler`) that react to mutations for entity reference index maintenance. These handlers need immediate reactive notification, not historical journal access. A single journal interface with typed entries eliminates all tension between competing observation mechanisms. The CognitiveJournal recording interface (`record(CognitiveEntry)`) never changes when new entry types are added. The sealed entry hierarchy grows by adding types to the `permits` clause — a mechanical source modification that triggers compile errors in consumers using exhaustive `switch` expressions, forcing explicit handling of new types. This is the intended design: compile-time safety at the cost of source stability in the hierarchy declaration. NOOP is a lambda. Predicate filtering at the trace level is more flexible than per-observer filtering.

**Trade-offs:** Refactoring five existing mechanisms is a large blast radius. Every producer and consumer of the old APIs changes. Worth it for a clean, unified design with no competing implementations.

**Sources:** DerivationObserver (`cognitive-observability-api`), DerivationTrace, PipelineTrace (`cognition-api/.../trace/`), CognitionCore.tick() pipeline structure, #960 derivation journal pattern
**Exploration:** deep-analysis
**Status:** revised (R1-02: corrected mechanism count from seven to five — two are dead code. R1-04: clarified that CognitiveJournal interface never changes; sealed hierarchy permits clause grows intentionally. R2-01: clarified GraphMutationRecorded CDI events continue for domain-logic consumers; journal subsumes observability role only.)

## D2: Entry type inventory — 16 types across three domains

**Choice:** 16 sealed record types grouped by lifecycle domain:
- **Derivation (4):** DerivationStarted, StageComputed, DataFlowed, DerivationCompleted — replaces DerivationObserver's 4 callbacks
- **Runtime tick (7):** AttentionBriefed, MoodShifted, DriveActivated, AppraisalRecorded, RegulationDecided, GoalDecided, CapsSettled — replaces PipelineTrace end-of-tick snapshot with root-to-tip entries at point of computation
- **Memory & consolidation (5):** MemoryRecalled, MemoryFormed, GraphMutated, Consolidated, ObservationCacheRefreshed — replaces MemoryRecallObserver, GraphMutationRecorded CDI event

**Alternatives:**
- Fewer entry types (coarser) — e.g., single TickRecorded with nested data. Rejected: loses the root-to-tip causal chain between stages.
- More entry types (finer) — including narrative, strategy learning, reflection, temporal focus, memory hygiene, per-subject mental model. Rejected for first cut: these stages produce prompt text or perform maintenance, not discrete structured cognitive state changes. Additive — can be added to the sealed hierarchy later.

**Rationale:** Curated subset of CognitionCore.tick() stages, selected by the criterion: stages that produce structured cognitive state changes with discrete inputs and outputs. The 7 runtime tick types cover attention briefing, mood shifts, drive activation, appraisal, emotion regulation, goal decisions, and CAPS settling. Stages excluded from the first cut: temporalFocus.tick() (context setup), memoryHygiene.tick() (maintenance), narrative.tick() / strategy.tick() / reflectionOrchestrator.tick() (prompt text generation, not structured state), per-subject userModel/mentalModel (per-subject iteration, not top-level state change), custom participant phases (extensible by definition — covered if participants record their own entries). The 16-type inventory covers everything PipelineTrace captured (end-of-tick) plus 10 intermediate states it missed (mood deltas, drive modulation, attention briefing, goal decisions, etc.).

**Trade-offs:** Excluded stages may need journal entries later if explainability requires tracing prompt assembly or maintenance operations. Adding them is a mechanical `permits` clause extension — consumers get compile errors guiding them to handle the new types.

**Depends on:** D1 (sealed entry hierarchy)
**Sources:** CognitionCore.tick() execution order (lines 237-351), PipelineTrace fields, existing leaf observers
**Exploration:** quick
**Status:** revised (R1-03: replaced false "1:1 mapping" claim with honest "curated subset" with explicit selection criteria. R1-04: clarified that adding entry types extends the permits clause, not the CognitiveJournal interface.)

## D3: Journal injection — CognitionCore field, passed to participants

**Choice:** CognitiveJournal as a field on CognitionCore (alongside pipelineSnapshotStore). CognitionCore passes the journal to orchestrators during configuration. Each orchestrator records entries at point of computation. For consolidation (async), ConsolidationScheduler receives the same journal instance via CDI injection.

**Alternatives:**
- CDI injection into each participant independently — more decoupled, but harder to scope journal lifecycle (per-tick vs per-session). Rejected: CognitionCore already orchestrates the pipeline.
- Journal on tick context object — clean if such a context existed, but no first-class tick context flows through participants today. Creating one would be a larger refactor for no benefit.
- CognitionTickParticipant as journal recorder — participants observe phase boundaries (FOUNDATION, SOURCE, DERIVED, TERMINAL), not individual stage computations within a phase. The journal needs to capture inputs, outputs, and decision rationale at the point of computation (e.g., mood delta from tickMood inside FOUNDATION, drive activation from drives.tick inside DERIVED). Participants receive CognitionTickContext which lacks computation results. Rejected: wrong abstraction level for per-stage granularity.

**Rationale:** CognitionCore already holds pipelineSnapshotStore and passes participants during configuration. Same pattern, same wiring. Zero new mechanism needed. The dual injection mechanism (field-passing for CognitionCore, CDI for ConsolidationScheduler) reflects the architecture: CognitionCore is imperatively configured via configure*() methods, ConsolidationScheduler is CDI-managed (@ApplicationScoped). CognitionCoreFactory creates the journal instance and wires it to both. Thread safety is handled by per-eventRef scoping (D6) — tick and consolidation write to independent scopes of the same journal, with no shared mutable state.

**Depends on:** D1 (CognitiveJournal interface), D6 (per-eventRef scoping)
**Sources:** CognitionCore configuration methods, pipelineSnapshotStore field, ConsolidationScheduler CDI wiring
**Exploration:** quick
**Status:** revised (R1-09: added CognitionTickParticipant as explicitly rejected alternative. R1-10: clarified dual injection rationale and same-instance guarantee.)

## D4: Module placement — cognition-api + cognitive-observability-core

**Choice:** CognitiveEntry (sealed interface + all 16 record types), CognitiveJournal (interface), and CognitiveTrace (in-memory accumulator) in `cognition-api` (L25). Persistent journal store in `cognitive-observability-core` (L23).

**Alternatives:**
- All types in `cognitive-observability-api` (L16) — **layer violation.** Three runtime tick entry types (AppraisalRecorded, RegulationDecided, CapsSettled) reference types from `cognition-api` (L25): AppraisalResult, EmotionRegulationResult, CapsSettlingSnapshot. Placing them in L16 creates an L16 → L25 upward dependency. Rejected: violates architectural layering.
- New module `cognitive-journal-api` at L25 — unnecessary. cognition-api is semantically appropriate ("Cognition SPIs and value types — tick types, state records") and the dependency paths already exist.
- "View" records with primitive/simple fields in cognitive-observability-api — preserves layering but loses type safety and creates duplication of cognitive types. Rejected: unnecessary complexity that weakens the type system.

**Rationale:** cognition-api (L25) is the right home: journal entry types ARE cognitive state records, and CognitiveJournal is a cognition SPI. The persistent store in cognitive-observability-core follows the existing pattern exactly — cognitive-observability-core already depends on cognition-api (verified from pom.xml), just as it already uses PipelineTrace (cognition-api) with PipelineSnapshotStore (cognitive-observability-core). For derivation recording, CognitiveDerivationEngine at L21 (cognitive-index) continues to use DerivationObserver (L16). A bridge adapter at L25 (cognition module) converts DerivationObserver callbacks into CognitiveEntry records — the same pattern DerivationTrace already uses to convert callbacks into sealed Entry records.

**Depends on:** D1 (CognitiveJournal interface), D2 (entry types)
**Sources:** cognitive-observability-core pom.xml (existing cognition-api dependency), PipelineTrace/PipelineSnapshotStore module split pattern, CognitiveDerivationEngine location (cognitive-index, L21)
**Exploration:** quick
**Status:** revised (R1-05: changed from cognitive-observability-api to cognition-api to resolve layer violation. Bridge adapter pattern for derivation recording.)

## D5: eventRef format — EventRef value type with factory methods

**Choice:** EventRef sealed interface with factory methods standardizing the formats established by #960:
- `EventRef.init()` — startup derivation
- `EventRef.tick(47)` — runtime tick N
- `EventRef.event("abc123")` — specific event trigger
- `EventRef.consolidation(EventRef.tick(47))` — consolidation triggered by tick context

All 16 entry types carry agentId + EventRef + timestamp as first-class fields via the sealed interface contract. EventRef serializes to string format (`"init"`, `"tick:47"`, `"event:abc123"`, `"consolidation:tick:47"`) for JSON/SQL compatibility via `toString()`. Static `parse(String)` factory for deserialization.

**Alternatives:**
- Raw string eventRef — no compile-time validation of the multi-format discriminated union. A typo like `"tick47"` or `"ticks:47"` silently corrupts correlation across three observability layers. Rejected: unacceptable for a primary correlation key used in 16+ entry types and SQL predicates.
- Full structured object with no string serialization — adds complexity to JSON/SQL storage layer. Rejected: string serialization format is already established and works well for storage.

**Rationale:** eventRef is a primary correlation key joining derivation journal, runtime cognitive journal, and session transcript. Type safety at this join point is not YAGNI — it's the minimum viable design. Factory methods provide compile-time safety. The sealed interface enables pattern matching for consumers (`switch (eventRef) { case Tick t -> ...; case Init i -> ...; }`). String serialization preserves compatibility with existing PipelineSnapshotStore and JSON formats. A `type()` discriminator method enables efficient SQL predicates (`WHERE eventRef_type = 'tick'`) instead of LIKE patterns.

**Depends on:** D1 (correlation model)
**Sources:** DerivationObserver eventRef usage in #960, PipelineSnapshotStore.findByTick query model
**Exploration:** quick
**Status:** revised (R1-06: changed from raw string to EventRef value type with factory methods and string serialization.)

## D6: Journal lifecycle — per-eventRef scope

**Choice:** CognitiveTrace is scoped per-eventRef. Each eventRef defines a lifecycle boundary:
- `tick:N` — entries accumulate during CognitionCore.tick(), trace flushed at tick end
- `init` / `event:id` — entries accumulate during derivation, trace flushed at derivation completion
- `consolidation:tick:N` — entries accumulate during consolidation run, trace flushed at run completion

At scope completion, accumulated entries are persisted to the journal store and the in-memory trace is cleared.

**Alternatives:**
- Per-session scope — trace accumulates across all ticks in a session (like DerivationTrace across a full derive() call). Creates unbounded growth; 300-tick scenarios produce 4,800+ entries in memory. Rejected: no natural flush point.
- Per-tick only — doesn't cover derivation or consolidation lifecycles which have their own temporal boundaries. Rejected: not truly unified.
- Unbounded with consumer filtering — grows without bound, shifts complexity to every consumer. Rejected: violates separation of concerns.

**Rationale:** eventRef already defines the correlation boundary — scoping the trace to the same boundary is natural. Per-eventRef means at most ~16 entries in memory at any time (one per stage in the current tick). Historical queries go to the persistent store. This matches DerivationTrace's existing per-derivation-call lifecycle and PipelineSnapshotStore's per-tick persistence model.

**Depends on:** D1 (CognitiveJournal), D5 (EventRef)
**Sources:** DerivationTrace per-call lifecycle, PipelineSnapshotStore per-tick persistence, CognitionCore.tick() structure
**Exploration:** surfaced by review (R1-07)
**Status:** captured

## D7: PipelineTrace coexistence — journal complements, does not replace

**Choice:** PipelineTrace and PipelineSnapshotStore continue to exist alongside the journal. The journal subsumes the observability role of the leaf observation mechanisms (DerivationObserver, MemoryRecallObserver, GraphMutationRecorded). GraphMutationRecorded CDI events continue to be fired for domain-logic consumers (entity token handlers). PipelineSnapshotStore remains for end-of-tick state snapshots.

**Alternatives:**
- Replace PipelineSnapshotStore entirely with journal reconstruction — reconstruct end-of-tick state from journal entries. Possible but expensive: requires replaying entries to build the snapshot, and `promptSections` (Map<String, String> of rendered prompt text) is a rendering side effect not captured by any cognitive state entry. Rejected for initial implementation: adds complexity with no benefit.
- Add PromptSectionsRendered entry type and deprecate PipelineSnapshotStore — long-term viable but premature. Prompt sections are rendered output, not a cognitive state change. Rejected: mixing concerns.

**Rationale:** PipelineTrace captures the cognitive pipeline's OUTPUT (what prompt sections were rendered, what the final CAPS state was). The journal captures the pipeline's PROCESS (what state changes happened and why). These are architecturally distinct:
- CognitionService.pipelineDump() needs efficient point-in-time state queries → PipelineSnapshotStore with tick-indexed lookups
- Explainability and debugging need the causal chain of state changes → CognitiveJournal with eventRef-scoped entries
- PipelineCoherenceEvalTest needs trajectory serialization from snapshot sequences → PipelineSnapshotStore

The four existing PipelineSnapshotStore consumers (CognitionService, CognitivePipelineDump, PipelineCoherenceEvalTest, GraphQL resolvers) are unaffected. D1's "unified" claim refers to unifying the five observation mechanisms, not replacing all state persistence. GraphMutationRecorded CDI events continue to be fired by MutationTrackingDecorator for domain-logic consumers (EntityErasureTokenHandler, EntityMergeTokenHandler) — the journal subsumes the observability role, not the reactive notification role.

**Depends on:** D1 (journal scope), D4 (module placement)
**Sources:** CognitionService.pipelineDump(), CognitivePipelineDump, PipelineCoherenceEvalTest, PipelineTrace.promptSections field, EntityErasureTokenHandler, EntityMergeTokenHandler (entity-token-cdi)
**Exploration:** surfaced by review (R1-08)
**Status:** revised (R2-01: clarified GraphMutationRecorded CDI events continue for domain-logic consumers. Changed "replaces" to "subsumes the observability role of.")
