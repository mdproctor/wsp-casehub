# MemoryHygiene Pattern — Design Spec

**Issue:** casehubio/blocks#120
**Epic:** casehubio/blocks#126 (Autonomous Agent Patterns)
**Package:** `io.casehub.blocks.memory`

## Overview

Manages memory lifecycle for agents with persistent CBR memory stores: consolidation (merging related memories), forgetting (eviction of low-value memories), importance scoring, temporal versioning (invalidate-not-delete), cross-linking (consolidation provenance), and integrity checking (structural + semantic escalation).

Composes existing infrastructure — no new persistence layers:
- **neocortex**: `CbrCaseMemoryStore`, `CbrRetentionPolicy`, `TemporalDecay`, `ScopeDecay`, `ReflectionOrchestrator`
- **blocks/summarisation**: `ContentSummariser<T>`, `TieredContentSummariser<T>`

## Architecture

Two entry modes:

1. **Tick** — `MemoryHygieneOrchestrator.tick(agentId, tenantId)` → `HygieneTick`. On-demand, bounded cost. Runs: importance scoring → consolidation (pass 1: summarise) → eviction. Called by consuming apps at their chosen cadence.

2. **Maintain** — `MemoryHygieneScheduler.maintain(agentId, tenantId)` → `MaintenanceTick`. Full idle-time pipeline. Externally composes: tick() + reflection (pass 2) + cross-linking + integrity checks. Called during idle periods (MemGPT "Sleeptime" concept).

The orchestrator has a single entry point (`tick()`), consistent with `PersonalityEvolutionOrchestrator.tick()` and `InnerLifeOrchestrator.tick()`. The scheduler externally composes the full pipeline.

### Pipeline Stages

```
┌─────────────────────────────────────────────────────────────┐
│ tick() — on-demand                                          │
│                                                             │
│  1. Score importance    ImportanceScorer × each memory       │
│  2. Consolidate (P1)   ContentSummariser<ScoredCbrCase>     │
│     └─ Supersede sources, store consolidated case           │
│  3. Evict              Composite score < threshold → erase  │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│ maintain() — idle-time (scheduler composes externally)      │
│                                                             │
│  4. Reflect (P2)       ReflectionOrchestrator.reflect()     │
│     └─ Store as ReflectionEntry records                     │
│  5. Cross-link          Annotate consolidated cases with    │
│     └─ source caseIds as StringListVal features             │
│  6. Integrity check    IntegrityChecker → structural        │
│     └─ Escalate flagged anomalies to SemanticIntegrity      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Relationship with Existing Retention

MemoryHygiene **replaces** `CbrRetentionScheduler` for agents that use it. The orchestrator implements its own scan-and-evict loop via `retrieveSimilar` + `erase` — it cannot use `CbrRetentionPolicy.purge()` because purge uses simple filter criteria (maxAgeDays, maxCasesPerType, minTrustScore), not weighted composite scores.

Retrieval-time decay (`TemporalDecayCbrCaseMemoryStore` decorator) and eviction-time decay serve different purposes: retrieval decay modulates ranking during queries; eviction decay determines whether a memory is worth keeping at all. These are independent and intentionally separate.

Agents using MemoryHygiene should disable `CbrRetentionScheduler` for the same domain to avoid conflicting retention decisions.

## Type Catalogue

### Importance Scoring

| Type | Kind | Description |
|------|------|-------------|
| `ImportanceScorer` | `@FunctionalInterface` | `double score(CbrCaseSummary summary, Instant now)` → [0,1]. Pluggable importance scoring. |
| `ArousalScorer` | class, `@DefaultBean` | Heuristic approximation: word-list sentiment intensity from case problem/solution text. Zero LLM cost. Production override with LLM-backed implementation for psychological fidelity. |
| `SurpriseScorer` | class, `@DefaultBean` | Heuristic approximation: information entropy of case features relative to agent's typical feature distribution. Zero LLM cost. |
| `CompositeImportanceScorer` | class | Weighted combination: `score = Σ(scorer_i.score() × weight_i) / Σ(weight_i)`. Constructor takes `List<WeightedScorer>`. |
| `WeightedScorer` | record | `(ImportanceScorer scorer, double weight)` — entry in the composite. Weight validated > 0. |

### Retention & Eviction

| Type | Kind | Description |
|------|------|-------------|
| `RetentionScore` | record | `(String caseId, double importance, double recencyFactor, double scopeFactor, double trustFactor, double composite)` — full audit trail per memory. `composite = (importance × iw + recency × rw + scope × sw + trust × tw) / (iw + rw + sw + tw)` (weighted arithmetic mean, all inputs [0,1]). |
| `RetentionConfig` | record | `retentionThreshold` [0,1] (default 0.1), `importanceWeight` (default 1.0), `recencyWeight` (default 1.0), `scopeWeight` (default 0.5), `trustWeight` (default 0.5). Validates: all weights ≥ 0, at least one > 0. |
| `MemoryHygieneConfig` | interface (`@ConfigMapping`) | `retentionConfig()`, `consolidationBatchSize()` (default 100), `maxReflectionSources()` (default 50), `crossLinkSimilarityThreshold()` (default 0.7), `memoryDomain()` (required), `caseTypes()` (required). |

### Consolidation

Consolidation uses `ContentSummariser<ScoredCbrCase<? extends CbrCase>>` — the existing blocks summarisation SPI. The orchestrator groups memories by similarity (via `retrieveSimilar` with high topK), then feeds each group to the summariser. The summariser produces a merged case; sources are superseded via `CbrCaseMemoryStore.supersede(sourceId, mergedId, "hygiene-consolidation")`.

The tick uses `TieredContentSummariser` dispatch: small groups (≤5) get heuristic merging (feature union, text concatenation), larger groups get LLM-backed synthesis.

### Reflection Storage

Reflections from `ReflectionOrchestrator.reflect()` are `List<String>` — abstract insights that don't fit the CbrCase contract (problem/solution/outcome). Stored as lightweight records:

| Type | Kind | Description |
|------|------|-------------|
| `ReflectionEntry` | record | `(String agentId, String tenantId, String insight, Instant generatedAt, List<String> sourceCaseIds)` — a single reflection with provenance. |
| `ReflectionStore` | `@FunctionalInterface` | `void store(ReflectionEntry entry)`. SPI for reflection persistence. |
| `NoOpReflectionStore` | class, `@DefaultBean` | No-op implementation. Consumers override with real persistence. |

### Integrity Checking

| Type | Kind | Description |
|------|------|-------------|
| `IntegrityChecker` | `@FunctionalInterface` | `List<IntegrityViolation> check(String agentId, String tenantId, MemoryDomain domain)` |
| `IntegrityViolation` | record | `(String caseId, ViolationType type, String detail, boolean escalateToSemantic)` |
| `ViolationType` | enum | `ORPHANED_SUPERSESSION` (superseded case points to non-existent superseding case), `DUPLICATE_CASE` (same content stored twice), `MISSING_FEATURES` (required feature keys absent), `UNPROCESSED_STALE` (old memory never reviewed by hygiene pipeline), `SEMANTIC_CONFLICT` (contradictory memories — semantic checker only) |
| `SemanticIntegrityChecker` | `@FunctionalInterface` | `List<IntegrityViolation> checkSemantic(List<IntegrityViolation> flagged, String agentId, String tenantId)` |
| `NoOpSemanticIntegrityChecker` | class, `@DefaultBean` | Returns empty list. Consumers override with LLM-backed implementation. |
| `DefaultIntegrityChecker` | class, `@ApplicationScoped` | Structural checks via `CbrCaseMemoryStore.scan()` + `findSupersededCases()`. Sets `escalateToSemantic=true` on anomalies with high feature overlap but contradictory outcomes. Delegates flagged items to injected `SemanticIntegrityChecker`. |

### Orchestrator & Scheduler

| Type | Kind | Description |
|------|------|-------------|
| `MemoryHygieneOrchestrator` | `@ApplicationScoped` | Single entry: `HygieneTick tick(String agentId, String tenantId)`. Per-agent `ReentrantLock` for tick serialisation. Injects: `CbrCaseMemoryStore`, `ImportanceScorer`, `TemporalDecay`, `ScopeDecay`, `ContentSummariser`, `MemoryHygieneConfig`. |
| `HygieneTick` | sealed interface | `Idle(String reason)` — nothing to do (no memories, or below batch threshold). `Completed(int consolidated, int evicted, int totalScored, List<RetentionScore> scores)` — full audit. `Failed(String reason)` — pipeline error. |
| `MaintenanceTick` | sealed interface | `Completed(HygieneTick hygiene, int reflectionsGenerated, int crossLinksCreated, List<IntegrityViolation> violations)`. `Failed(String stage, String reason)` — which stage failed. |
| `MemoryHygieneScheduler` | class (not CDI) | Constructor: `(MemoryHygieneOrchestrator, ReflectionOrchestrator, ReflectionStore, IntegrityChecker, CbrCaseMemoryStore, MemoryHygieneConfig)`. `MaintenanceTick maintain(String agentId, String tenantId)` — composes: tick() + reflect + cross-link + integrity. Consumer constructs and wires into their scheduler. |

## Tick Pipeline Detail

### Step 1: Importance Scoring

```java
// Retrieve all active memories for this agent in the configured domain
var query = CbrQuery.of(tenantId, domain, scope, caseType, Map.of(), config.consolidationBatchSize())
    .withMinSimilarity(0.0);
var memories = store.retrieveSimilar(query, CbrCase.class);

// Score each memory
var scored = memories.stream()
    .map(m -> new RetentionScore(
        m.caseId(),
        importanceScorer.score(toSummary(m), now),
        temporalDecay.factor(m.storedAt(), now),
        scopeDecay.factor(depthDistance(m, scope)),
        trustFactor(m),
        composite(importance, recency, scope, trust, config.retentionConfig())))
    .toList();
```

### Step 2: Consolidation (Pass 1)

Group high-similarity memories and merge via `ContentSummariser`:

1. Retrieve memory pairs with similarity above `crossLinkSimilarityThreshold`
2. Group into merge candidates (connected components)
3. For each group: summarise → store merged case → supersede sources
4. Supersession provides invalidate-not-delete: `store.supersede(sourceId, mergedId, "hygiene-consolidation")`
5. `SupersessionStatus.supersededAt` provides temporal context for when the original memory was valid

### Step 3: Eviction

```java
var toEvict = scored.stream()
    .filter(s -> s.composite() < config.retentionConfig().retentionThreshold())
    .toList();
for (var eviction : toEvict) {
    store.erase(new EraseRequest(eviction.caseId(), tenantId));
}
```

## Maintain Pipeline Detail (Scheduler)

### Step 4: Reflection (Pass 2)

```java
var reflections = reflectionOrchestrator.reflect(agentId, tenantId, since, config.maxReflectionSources());
for (var insight : reflections) {
    reflectionStore.store(new ReflectionEntry(agentId, tenantId, insight, now, sourceCaseIds));
}
```

### Step 5: Cross-Linking

During consolidation (step 2), the merged case stores its source caseIds as a `StringListVal` feature:

```java
var features = new HashMap<>(mergedCase.features());
features.put("source_cases", FeatureValue.stringList(sourceCaseIds));
mergedCase = mergedCase.withFeatures(features);
```

This is a write-only annotation — consolidation provenance, not a navigable graph.

### Step 6: Integrity Checking

```java
var violations = integrityChecker.check(agentId, tenantId, domain);
// DefaultIntegrityChecker runs structural checks:
// - Orphaned supersessions: supersededCaseId points to non-existent case
// - Duplicate cases: same features+outcome within similarity threshold
// - Missing features: required feature keys absent
// - Unprocessed stale: cases older than config threshold never reviewed by hygiene
// Then escalates flagged items to SemanticIntegrityChecker
```

## Dependencies

**Compile:** `casehub-neocortex-memory-api` (CbrCaseMemoryStore, TemporalDecay, ScopeDecay, CbrRetentionPolicy, ReflectionOrchestrator, MemoryDomain, CbrCase, CbrQuery, FeatureValue, SupersessionStatus, EraseRequest)
**Compile:** `casehub-blocks` (ContentSummariser, TieredContentSummariser, SummarisationRunner — same module, different package)
**Provided:** `casehub-platform-agent-api` (AgentProvider — for LLM-backed implementations)
**Test:** JUnit 5, Mockito, AssertJ

## Testing Strategy

Plain JUnit 5 with Mockito — no Quarkus runtime (consistent with blocks test conventions).

| Test class | What it covers |
|-----------|----------------|
| `ImportanceScorerTest` | ArousalScorer, SurpriseScorer, CompositeImportanceScorer — boundary values, weight normalisation |
| `RetentionScoreTest` | Composite score computation, weight validation, threshold comparison |
| `MemoryHygieneOrchestratorTest` | Full tick pipeline: mock store → score → consolidate → evict. Per-agent locking. Idle/Completed/Failed outcomes. |
| `MemoryHygieneSchedulerTest` | maintain() composition: tick + reflect + cross-link + integrity. Stage failure isolation. |
| `DefaultIntegrityCheckerTest` | Each ViolationType detected. Semantic escalation for flagged items. |
| `ConsolidationTest` | Supersession of sources, feature merge, StringListVal provenance annotation |

## References

- Research: `docs/research/2026-08-16-autonomous-agent-patterns-landscape.md` §2.2, §2.8
- LUFY: "Should RAG Chatbots Forget Unimportant Conversations?" (2024, arXiv:2409.12524)
- MemGPT / Letta: "Sleeptime Agents" (ICLR 2024; Letta 2025)
- ACT-R memory activation model — power-law decay (recency × frequency)
- Memory OS of AI Agent (EMNLP 2025)
- GE-20260804-eb75e0 — scan() returns CbrCaseSummary without features
- GE-20260716-f292d3 — score-replacing decorators discard pre-applied decay
- GE-20260720-b7a8b9 — eraseEntity() is not domain-scoped
- CbrCaseMemoryStore (neocortex-memory-api)
- TemporalDecay, ScopeDecay (neocortex-memory-api)
- ReflectionOrchestrator (neocortex-memory-api)
- ContentSummariser, TieredContentSummariser (blocks/summarisation)
- PersonalityEvolutionOrchestrator, InnerLifeOrchestrator (blocks/agentic/personality)
- SupersessionStatus (neocortex-memory-api)
