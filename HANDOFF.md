# HANDOFF — casehub-examples (slot 196)

## Last Session

Designed and implemented casehubio/examples#961 — unified cognitive pipeline journal. Full brainstorming → spec → plan → implementation cycle in one session.

### What Was Built

**CognitiveJournal** — a unified observability mechanism replacing five competing mechanisms with one typed entry model. All code on branch `issue-961-runtime-cognitive-journal` in neocortex.

**New types in `cognition-api` (`io.casehub.neocortex.cognition.journal`):**
- `CognitiveEntry` — sealed interface, 16 record types across derivation/runtime/consolidation domains
- `CognitiveJournal` — single-method interface (`record(CognitiveEntry)`) with NOOP sentinel
- `CognitiveTrace` — in-memory accumulator with predicate filtering and query API
- `EventRef` — sealed value type for tick/event/consolidation correlation
- `FacetEffect` — extracted from DerivationObserver inner type
- `CapsSettlingResult` — extracted from PipelineTrace inner type

**New types in `cognition` (`io.casehub.neocortex.cognition.core`):**
- `ScopedCognitiveJournal` — `@ApplicationScoped` routing implementation with ThreadLocal scope isolation
- `DerivationJournalBridge` — adapts DerivationObserver (L16) callbacks to CognitiveEntry records (L25)
- `MemoryRecallJournalBridge` — adapts MemoryRecallObserver to journal entries

**New types in `cognitive-observability-core`:**
- `CognitiveJournalStore` — interface for persistent journal storage
- `InMemoryCognitiveJournalStore` — in-memory implementation

**CognitionCore.tick() instrumented** — records AttentionBriefed, MoodShifted (pre/post-appraisal), DriveActivated (first/post-appraisal), AppraisalRecorded, RegulationDecided at each pipeline stage's point of computation. Tick counter moved to start of tick() for correlation.

**Dead code deleted:** CompactionObserver, ObservationCacheObserver (zero references).

### Code Review Findings — Fixed

- **ThreadLocal leak (H2):** endScope() now in `finally` block — no leak on tick() exception
- **GoalDecided placeholder (H3):** Removed empty-list placeholder — GoalProposalOrchestrator API needed for real data

### Code Review Findings — Deferred as Issues

- **H1:** CognitionCoreFactory doesn't wire CognitiveJournal → neocortex#969
- **M1:** MemoryRecallJournalBridge uses static EventRef("recall") instead of tick correlation
- **M4:** FacetEffect/CapsSettlingResult deprecation annotations not added to original inner types

### Filed Issues

- neocortex#965 — GraphMutated CDI listener (agentId enrichment)
- neocortex#966 — ConsolidationScheduler journal instrumentation
- neocortex#967 — CognitiveJournal layer evaluation for direct MemoryRecallObserver replacement
- neocortex#968 — Remaining CognitionCore.tick() stage instrumentation
- neocortex#969 — Wire CognitiveJournal in CognitionCoreFactory CDI context

### Design Artifacts

- Spec: `wsp-casehub/specs/issue-961-runtime-cognitive-journal/2026-11-13-runtime-cognitive-journal-design.md`
- Decisions: `wsp-casehub/specs/issue-961-runtime-cognitive-journal/decisions.md` (7 decisions, standard review with 3 rounds)
- Plan: `wsp-casehub/plans/2026-11-13-runtime-cognitive-journal.md`
- Blog: `wsp-casehub/blog/2026-11-13-mdp01-the-journal-that-sees-everything.md`

### Test Coverage

- 30 new tests: EventRef (7), CognitiveTrace (8), JournalStore contract (8), ScopedJournal (6), DerivationBridge (4)
- 896 existing cognition tests pass, 0 regressions

### Key Architecture Decisions

- **D1:** Single CognitiveJournal interface, not a family of observers — avoids competing mechanisms
- **D4:** Types in `cognition-api` (L25), not `cognitive-observability-api` (L16) — layer constraint from AppraisalResult/EmotionRegulationResult references
- **D5:** EventRef is a sealed value type, not a raw string — compile-time safety for the primary correlation key
- **D7:** PipelineTrace coexists (OUTPUT vs PROCESS) — journal doesn't replace snapshot queries

## Priorities for Next Session

1. **neocortex#969** — Wire CognitiveJournal in CognitionCoreFactory (activates journal in production, S/Low)
2. **neocortex#965** — GraphMutated CDI listener with agentId enrichment (S/Med)
3. **neocortex#966** — ConsolidationScheduler journal instrumentation (S/Med)
4. **Work-end for #961** — code review, squash, push, close issue

## Paused Work

Pause stack depth: 2
- `issue-935-session-reuse-audit` — LLM session reuse audit (paused to start #961)

## Slot 211 Overlap

Coordinate on neocortex main changes to `CognitionCoreFactory` and `CognitivePipelineBuilder` — both slots now modify these files.

## Neocortex Build Note

Full reactor build may fail at `schema-generator` module. Individual module builds work. Use `-pl <module> -am` for targeted builds.

## References

- Branch: `issue-961-runtime-cognitive-journal` (neocortex + examples + workspaces)
- Design spec: `wsp-casehub/specs/issue-961-runtime-cognitive-journal/2026-11-13-runtime-cognitive-journal-design.md`
- GitHub issue: casehubio/examples#961
