# Design Journal — issue-126-autonomous-agent-patterns

## §120 MemoryHygiene — 2026-08-20

**Package placement revised:** Originally `agentic.memory`, moved to top-level `blocks.memory` after spec review (R1-09). Memory hygiene is reusable infrastructure like `blocks.summarisation` — not inherently agentic. A non-agentic CBR app could use it.

**Pipeline order revised:** Originally score → consolidate → evict. Spec review (R1-07) identified that consolidation creates new unscored memories while eviction uses stale pre-consolidation scores. Reversed to score → evict → consolidate — evict garbage first, then consolidate survivors.

**Consolidation mapping:** `ContentSummariser` returns `SummaryResult` (text), not `CbrCase`. Two-step mapping: text synthesis via summariser, then `FeatureVectorCbrCase` construction with merged features + `source_cases` `StringListVal` provenance.

**Replaces, not composes:** `MemoryHygieneOrchestrator` replaces `CbrRetentionScheduler` for agents that opt in. `CbrRetentionPolicy.purge()` can't express weighted composite scores. Own scan-and-evict loop via `retrieveSimilar` + `erase`.

**Reflections stored separately:** Decision review (R1-07) caught that reflections don't fit the `CbrCase` contract (problem/solution/outcome). Stored as `ReflectionEntry` records via `ReflectionStore` SPI, not crammed into CBR.
