# Runtime Cognitive Pipeline Journal — Tick-Correlated Observability

**Issue:** casehubio/examples#961
**Branch:** issue-961-runtime-cognitive-journal
**Date:** 2026-11-13

## Overview

The derivation journal (#960) captures the static personality-to-cognition pipeline at character registration time. But during a scenario run, the runtime cognitive pipeline — CAPS appraisals, drive activations, regulation strategy selections, mood shifts, memory consolidation — produces changes that aren't journaled in the same structured, correlatable format.

Given a 300-tick scenario transcript, there's no way to answer: "which prompt at tick 47 caused this character's fear appraisal to fire, and how did that trace back to their PID-5 facet scores?"

This design introduces a unified cognitive journal that replaces five competing observability mechanisms with a single typed entry model, enabling root-to-tip causal tracing from stimulus through every pipeline stage to response.

## Problem Analysis

### Current State — Five Competing Mechanisms

| Mechanism | What it captures | Correlation | Pattern |
|-----------|-----------------|-------------|---------|
| DerivationObserver | Static derivation steps | agentId + eventRef | Multi-method observer |
| PipelineTrace | End-of-tick final state | tick number (implicit) | Monolithic snapshot |
| MemoryRecallObserver | Recall events | agentId only | Single-method observer |
| GraphMutationRecorded | Graph mutations | tenantId only | CDI event |
| PipelineSnapshotStore | Persisted snapshots | tenantId + agentId + tick | Store service |

Two dead-code observers also exist in cognitive-observability-api (CompactionObserver, ObservationCacheObserver) — zero references anywhere. These are deleted, not unified.

### The Root-to-Tip Gap

PipelineTrace is an end-of-tick photograph. It captures what the final state IS, but not how it got there. Within each tick, CognitionCore runs ~15 stages in sequence. PipelineTrace captures the output of 5 of them (appraisal, regulation, CAPS settling, drives, channel weights). It misses:

- Mood PAD deltas (ticked twice — pre and post appraisal)
- Drive modulation deltas from appraisal (only final categories survive)
- Attention briefing drained from attentionMediator
- Appraisal-to-mood bridge signals
- Goal formation decisions
- All intermediate state between stages

The journal captures events as they flow through the pipeline — you see every step, every decision, every intermediate state change. Root to tip.

## Architecture

### Three Components

**1. CognitiveEntry** — sealed interface in `cognition-api`

Every cognitive state change in the system is a CognitiveEntry. Every entry carries `agentId()`, `eventRef()`, `timestamp()`.

```java
public sealed interface CognitiveEntry {

    String agentId();
    EventRef eventRef();
    Instant timestamp();

    // Derivation domain (replaces DerivationObserver callbacks)
    record DerivationStarted(String agentId, EventRef eventRef, Instant timestamp,
                             String axes, List<String> profileTerms,
                             List<String> facetNames, List<String> goals) implements CognitiveEntry {}

    record StageComputed(String agentId, EventRef eventRef, Instant timestamp,
                         String method, Map<String, Double> baseValues,
                         List<FacetEffect> facetEffects,
                         Map<String, Double> finalValues) implements CognitiveEntry {}

    record DataFlowed(String agentId, EventRef eventRef, Instant timestamp,
                      String sourceMethod, String sourceField,
                      String targetMethod, String targetField,
                      double value) implements CognitiveEntry {}

    record DerivationCompleted(String agentId, EventRef eventRef,
                               Instant timestamp) implements CognitiveEntry {}

    // Runtime tick domain (replaces PipelineTrace snapshot)
    record AttentionBriefed(String agentId, EventRef eventRef, Instant timestamp,
                            String briefingContent) implements CognitiveEntry {}

    record MoodShifted(String agentId, EventRef eventRef, Instant timestamp,
                       MoodBaseline before, MoodBaseline after,
                       String trigger) implements CognitiveEntry {}

    record DriveActivated(String agentId, EventRef eventRef, Instant timestamp,
                          List<String> driveCategories,
                          Map<String, Double> intensities) implements CognitiveEntry {}

    record AppraisalRecorded(String agentId, EventRef eventRef, Instant timestamp,
                             AppraisalResult result,
                             Map<String, Double> concernSalience) implements CognitiveEntry {}

    record RegulationDecided(String agentId, EventRef eventRef, Instant timestamp,
                             EmotionRegulationResult result) implements CognitiveEntry {}

    record GoalDecided(String agentId, EventRef eventRef, Instant timestamp,
                       List<String> goalsFormed,
                       List<String> goalsRevised) implements CognitiveEntry {}

    record CapsSettled(String agentId, EventRef eventRef, Instant timestamp,
                       Map<String, Double> inputActivations,
                       CapsSettlingSnapshot settling) implements CognitiveEntry {}

    // Memory and consolidation domain (replaces leaf observers)
    record MemoryRecalled(String agentId, EventRef eventRef, Instant timestamp,
                          String domain, int resultCount) implements CognitiveEntry {}

    record MemoryFormed(String agentId, EventRef eventRef, Instant timestamp,
                        String domain, double salienceScore,
                        String contentSummary) implements CognitiveEntry {}

    record GraphMutated(String agentId, EventRef eventRef, Instant timestamp,
                        GraphMutation mutation) implements CognitiveEntry {}

    record Consolidated(String agentId, EventRef eventRef, Instant timestamp,
                        int inputNodes, int outputNodes,
                        long elapsedMs, String phase) implements CognitiveEntry {}

    record ObservationCacheRefreshed(String agentId, EventRef eventRef, Instant timestamp,
                                     int condensedNodes, int recentNodes,
                                     int tokenBudget,
                                     int actualTokens) implements CognitiveEntry {}
}
```

FacetEffect is preserved as a standalone record (already exists in DerivationObserver).

**2. CognitiveJournal** — single-method interface in `cognition-api`

```java
public interface CognitiveJournal {

    CognitiveJournal NOOP = entry -> {};

    void record(CognitiveEntry entry);
}
```

The interface never changes. Adding a new pipeline stage means adding a new record type to the CognitiveEntry sealed hierarchy — the `permits` clause grows, triggering compile errors in consumers using exhaustive `switch` expressions. This is intentional: compile-time safety at the cost of source stability in the hierarchy declaration.

**3. CognitiveTrace** — accumulating implementation in `cognition-api`

```java
public class CognitiveTrace implements CognitiveJournal {

    private final List<CognitiveEntry> entries = new ArrayList<>();
    private final Predicate<CognitiveEntry> filter;

    public CognitiveTrace() { this(e -> true); }

    public CognitiveTrace(Predicate<CognitiveEntry> filter) {
        this.filter = filter;
    }

    @Override
    public void record(CognitiveEntry entry) {
        if (filter.test(entry)) entries.add(entry);
    }

    public List<CognitiveEntry> entries() {
        return Collections.unmodifiableList(entries);
    }

    public List<CognitiveEntry> entriesForEvent(EventRef eventRef) {
        return entries.stream()
                      .filter(e -> eventRef.equals(e.eventRef()))
                      .toList();
    }

    public <T extends CognitiveEntry> List<T> entriesOfType(Class<T> type) {
        return entries.stream()
                      .filter(type::isInstance)
                      .map(type::cast)
                      .toList();
    }

    public List<CognitiveEntry> entriesForAgent(String agentId) {
        return entries.stream()
                      .filter(e -> agentId.equals(e.agentId()))
                      .toList();
    }

    public void clear() { entries.clear(); }
    public boolean isEmpty() { return entries.isEmpty(); }
    public int size() { return entries.size(); }
}
```

### EventRef — Value Type for Correlation

EventRef is a sealed interface with factory methods, standardizing the formats from #960:

```java
public sealed interface EventRef {

    record Init() implements EventRef {
        @Override public String toString() { return "init"; }
    }

    record Tick(int number) implements EventRef {
        @Override public String toString() { return "tick:" + number; }
    }

    record Event(String id) implements EventRef {
        @Override public String toString() { return "event:" + id; }
    }

    record Consolidation(EventRef trigger) implements EventRef {
        @Override public String toString() { return "consolidation:" + trigger; }
    }

    static EventRef init() { return new Init(); }
    static EventRef tick(int number) { return new Tick(number); }
    static EventRef event(String id) { return new Event(id); }
    static EventRef consolidation(EventRef trigger) { return new Consolidation(trigger); }

    static EventRef parse(String s) {
        if ("init".equals(s)) return init();
        if (s.startsWith("tick:")) return tick(Integer.parseInt(s.substring(5)));
        if (s.startsWith("event:")) return event(s.substring(6));
        if (s.startsWith("consolidation:")) return consolidation(parse(s.substring(14)));
        throw new IllegalArgumentException("Unknown eventRef format: " + s);
    }
}
```

Pattern matching at the consumer level:
```java
switch (eventRef) {
    case EventRef.Tick t   -> queryByTick(t.number());
    case EventRef.Init i   -> queryDerivation();
    case EventRef.Consolidation c -> queryConsolidation(c.trigger());
    case EventRef.Event e  -> queryByEvent(e.id());
}
```

### Journal Lifecycle — Per-EventRef Scope

CognitiveTrace is scoped per-eventRef:

| Scope | Lifecycle | Flush point |
|-------|-----------|-------------|
| `tick:N` | CognitionCore.tick() start → end | End of tick, entries persisted to journal store |
| `init` / `event:id` | Derivation start → complete | Derivation completion |
| `consolidation:tick:N` | Consolidation run start → end | Consolidation completion |

At scope completion, accumulated entries are persisted to the journal store and the in-memory trace is cleared. At most ~16 entries in memory at any time (one per pipeline stage in the current tick).

### Journal Injection

CognitiveJournal as a field on CognitionCore, passed to orchestrators during configuration:

```
CognitionCoreFactory
  └── creates CognitiveJournal instance
  └── passes to CognitionCore via configure*() methods
  └── passes same instance to ConsolidationScheduler via CDI

CognitionCore.tick()
  ├── journal.record(AttentionBriefed(...))       // FOUNDATION
  ├── journal.record(MoodShifted(...))             // FOUNDATION (pre-appraisal)
  ├── journal.record(DriveActivated(...))          // DERIVED (first pass)
  ├── journal.record(AppraisalRecorded(...))       // DERIVED
  ├── journal.record(MoodShifted(...))             // post-DERIVED (re-tick)
  ├── journal.record(DriveActivated(...))          // post-DERIVED (re-tick)
  ├── journal.record(RegulationDecided(...))       // DERIVED
  ├── journal.record(GoalDecided(...))             // TERMINAL
  ├── journal.record(CapsSettled(...))             // when CAPS runs
  └── flush to journal store                       // end of tick

ConsolidationScheduler (async, CDI-injected same instance)
  ├── journal.record(MemoryFormed(...))
  ├── journal.record(Consolidated(...))
  └── flush to journal store                       // end of consolidation run
```

For derivation recording, CognitiveDerivationEngine (L21, cognitive-index) continues to use DerivationObserver (L16). A bridge adapter at L25 (cognition module) converts DerivationObserver callbacks into CognitiveEntry records:

```java
public class DerivationJournalBridge implements DerivationObserver {
    private final CognitiveJournal journal;

    @Override
    public void onPipelineStart(String agentId, String eventRef, PipelineInput input) {
        journal.record(new CognitiveEntry.DerivationStarted(
            agentId, EventRef.parse(eventRef), Instant.now(),
            input.axes(), input.profileTerms(), input.facetNames(), input.goals()));
    }
    // ... same pattern for other callbacks
}
```

### Module Placement

| Component | Module | Layer | Rationale |
|-----------|--------|-------|-----------|
| CognitiveEntry (sealed + 16 records) | cognition-api | L25 | Entry types reference AppraisalResult, EmotionRegulationResult, CapsSettlingSnapshot — all L25 types |
| CognitiveJournal (interface) | cognition-api | L25 | Cognition SPI |
| CognitiveTrace (in-memory) | cognition-api | L25 | Same as DerivationTrace pattern |
| EventRef (sealed value type) | cognition-api | L25 | Correlation type used by entries |
| DerivationJournalBridge | cognition | L25 | Adapts L16 DerivationObserver → L25 CognitiveEntry |
| CognitiveJournalStore (persistence) | cognitive-observability-core | L23 | Same split as PipelineSnapshotStore |

### Relationship to PipelineTrace

PipelineTrace and PipelineSnapshotStore coexist alongside the journal. They serve different purposes:

| Concern | Mechanism | Query pattern |
|---------|-----------|---------------|
| Pipeline OUTPUT (rendered prompts, final state) | PipelineTrace + PipelineSnapshotStore | "What was the state at tick N?" |
| Pipeline PROCESS (causal chain of state changes) | CognitiveJournal + CognitiveTrace | "What happened during tick N and why?" |

PipelineSnapshotStore consumers (CognitionService.pipelineDump(), CognitivePipelineDump, PipelineCoherenceEvalTest, GraphQL resolvers) are unaffected.

GraphMutationRecorded CDI events continue to be fired by MutationTrackingDecorator for domain-logic consumers (EntityErasureTokenHandler, EntityMergeTokenHandler) that need reactive notification. The journal subsumes the observability role — a CDI listener also writes a GraphMutated entry for correlation and querying.

### Mechanisms Eliminated

| Old mechanism | Replaced by | Migration |
|---------------|-------------|-----------|
| DerivationObserver (L16) | CognitiveEntry derivation types + DerivationJournalBridge | Bridge adapter at L25 converts callbacks to entries |
| DerivationTrace (L16) | CognitiveTrace | Unified trace with same query patterns |
| MemoryRecallObserver | CognitiveEntry.MemoryRecalled | Direct journal.record() calls |
| CompactionObserver | Deleted (dead code) | No consumers exist |
| ObservationCacheObserver | Deleted (dead code) | No consumers exist |

### Three-Layer Correlation Model

The journal enables joining three layers by `agentId` + `eventRef`:

1. **Derivation journal** (static) — personality → cognitive defaults. Entry types: DerivationStarted, StageComputed, DataFlowed, DerivationCompleted.
2. **Runtime cognitive journal** (per-tick) — stimulus → appraisal → regulation → response. Entry types: AttentionBriefed, MoodShifted, DriveActivated, AppraisalRecorded, RegulationDecided, GoalDecided, CapsSettled.
3. **Session transcript** (output) — the character's actual words/actions. Correlated via the same eventRef on prompt generation.

A viewer or LLM can trace any line of dialogue backward through all three layers to the facet score that shaped it.

## Testing Strategy

### Unit Tests
- CognitiveTrace: accumulation, filtering by predicate, query methods (entriesForEvent, entriesOfType, entriesForAgent), clear/lifecycle
- EventRef: factory methods, parse/toString round-trip, pattern matching
- DerivationJournalBridge: verify all 4 DerivationObserver callbacks produce correct CognitiveEntry types
- NOOP journal: verify no-op behavior

### Integration Tests
- CognitionCore.tick() produces the expected sequence of journal entries for a single tick
- Entry ordering matches pipeline execution order
- Per-eventRef scoping: entries accumulate during tick, flush at end
- ConsolidationScheduler journal entries carry correct consolidation eventRef

### Evaluation
- Run a GENERIC profile scenario, dump journal entries to JSON
- Verify backward tracing: pick a line of dialogue, trace through runtime entries to derivation entries to facet scores
- Compare coverage: which intermediate states does the journal capture that PipelineTrace misses?

## Scope

### In scope
- [ ] CognitiveEntry sealed interface with 16 record types
- [ ] CognitiveJournal single-method interface with NOOP sentinel
- [ ] CognitiveTrace accumulating implementation with query API
- [ ] EventRef sealed value type with factory methods and parse/toString
- [ ] DerivationJournalBridge adapter (DerivationObserver → CognitiveEntry)
- [ ] CognitiveJournalStore persistent store (replaces journal-related storage)
- [ ] Instrument CognitionCore.tick() — record entries at point of computation for all 7 runtime stages
- [ ] Instrument ConsolidationScheduler — record entries for memory formation, graph mutation, compaction
- [ ] GraphMutated CDI listener — bridge GraphMutationRecorded events to journal entries
- [ ] Delete dead code: CompactionObserver, ObservationCacheObserver
- [ ] Remove MemoryRecallObserver — replace with direct journal.record() calls

### Out of scope
- PipelineTrace/PipelineSnapshotStore changes (coexists as-is)
- UI/visualization (journal data model enables this, but no frontend work)
- Narrative, strategy learning, reflection instrumentation (additive — add entry types later if needed)
- Per-subject mental model instrumentation (additive)
- Session transcript entry type (the transcript already exists; correlation is via eventRef matching, not a new entry type)

## References

- DerivationObserver: `cognitive-observability-api/.../DerivationObserver.java`
- DerivationTrace: `cognitive-observability-api/.../DerivationTrace.java`
- PipelineTrace: `cognition-api/.../trace/PipelineTrace.java`
- PipelineSnapshotStore: `cognitive-observability-core/.../PipelineSnapshotStore.java`
- CognitionCore.tick(): `cognition/.../core/CognitionCore.java:237-351`
- GraphMutationRecorded: `cognitive-observability-api/.../GraphMutationRecorded.java`
- GraphMutation sealed hierarchy: `cognitive-observability-api/.../GraphMutation.java`
- MemoryRecallObserver: `cognitive-observability-api/.../MemoryRecallObserver.java`
- Emotion regulation spec: `wsp-casehub/specs/issue-590-emotion-regulation/2026-11-04-emotion-regulation-design.md`
- Issue #960: Derivation pipeline observability
- Issue #961: Runtime cognitive pipeline journal
