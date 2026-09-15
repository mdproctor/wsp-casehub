# Dynamic Norm Relevance Scoring — Design Spec

**Issue:** casehubio/examples#57
**Epic:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Date:** 2026-09-15
**Status:** Draft

---

## Problem

ManorNormFilter.filter() ignores its `nearbyCharacterNames` and `inventory` parameters — it returns all norms sorted by priority. This means every character sees every norm every tick, regardless of context. "Never help Penelope" appears even when Penelope isn't present and the character has no cognitive context about her.

The original issue proposed static text matching against a character name roster. This doesn't scale and isn't emergent — it hardcodes norm relevance at author time rather than deriving it from the character's cognitive state.

---

## Design Principle

Norms compete for attention, same as beliefs. Relevance is derived from the character's active cognitive context — what they know about nearby entities — not from static text parsing against a fixed roster. The attention budget gates how many norms surface, adapting to the situation (arousal, crowding, goal pressure).

---

## Architecture

### Integration into the Tick Loop

The norm scoring integrates into the existing per-tick cognitive pipeline from #54:

```
ScenarioOrchestrator tick loop (per character):
  1. CognitionCore.tick() — mood, drives
  2. ManorContextStrategy resolves situation — nearby agents, arousal, budget
  3. CognitiveProfile.resolve() per nearby entity → EntityKnowledge
  4. EntityKnowledge → TemporalEntry conversion
  5. TemporalFocus.focus() — ranks by salience
  6. Budget selection
  7. Blocks rendering
     └── renderCognitiveSections()
         └── ManorNormFilter.score() ← NEW: scores norms by cognitive context
             → budget-gated selection via CognitiveBudget.maxNorms()
```

The norm filter receives its cognitive context from the same EntityKnowledge results already computed at step 3. No additional CognitiveProfile queries.

### Data Flow

```
ScenarioOrchestrator
  ├── builds nearbyIds (agent IDs from world state)
  ├── builds agentNameMap (id → display name)
  └── calls cognition.renderCognitiveSections(character, nearbyIds, agentNameMap)

CharacterCognition.renderCognitiveSections()
  ├── extracts activeEntityNames from agentNameMap values
  └── calls contextStrategy.selectNorms(
  │       socialConfig.norms(),
  │       activeEntityNames,    ← display names from cognitive context
  │       character.inventory(),
  │       budget)
  └── renders filtered norms as "Social Rules" section

ManorContextStrategy.selectNorms()
  └── delegates to ManorNormFilter.score()
      └── returns budget-gated, scored norms
```

### Key Change: Agent IDs → Display Names

The current call site passes `nearbyAgentIds` (e.g., "penelope-pitstop") to the norm filter. Norm text references display names (e.g., "Penelope"). The fix: pass display names from `agentNameMap.values()` instead of agent IDs.

The `agentNameMap` is already built in ScenarioOrchestrator (line 247-251) and passed to `renderCognitiveSections`. The display names are the cognitive subjects — what the character perceives, not internal identifiers.

---

## Scoring Model

### ManorNormFilter.score()

Replaces the current `filter()` method. For each norm:

1. **Base score** = `norm.priority()` (integer, typically 1-10)
2. **Context relevance boost** = binary. If any word (≥3 chars) from any active entity display name appears in the norm's rule text (case-insensitive), the norm gets a fixed boost of `+10`
3. **Inventory relevance boost** = same logic applied to inventory item names, same `+10` boost
4. **Final score** = `priority + contextBoost + inventoryBoost`
5. Sort by score descending, return top `budget.maxNorms()` entries

The `+10` boost value means a context-relevant norm with priority 1 outscores a non-relevant norm with priority 10. This ensures contextual relevance dominates over static priority. When multiple norms are context-relevant, priority breaks ties.

### Word Matching

Entity display names are tokenized into words. Each word of length ≥ 3 is checked against the norm rule text via case-insensitive containment.

Example: Entity "Penelope Pitstop" → words ["Penelope", "Pitstop"]. Norm "Never help Penelope directly" contains "Penelope" → boost applied.

The length floor (≥3) prevents false positives from articles, prepositions, and very short name fragments.

### What Happens to Non-Relevant Norms

Non-relevant norms (no entity match, no inventory match) are NOT excluded — they stay in the candidate pool at their base priority. If the budget is large enough (high arousal, crowded room), general norms like "Maintain charm" still surface. They're just ranked below context-relevant norms.

This is the key architectural difference from static filtering: no norm is ever hard-excluded. The budget determines the cut line.

---

## Modified Types

### ManorNormFilter

```java
public final class ManorNormFilter {

    static final int CONTEXT_BOOST = 10;
    static final int MIN_WORD_LENGTH = 3;

    private ManorNormFilter() {}

    public record ScoredNorm(SocialConfig.NormEntry norm, double score) {}

    public static List<SocialConfig.NormEntry> score(
            List<SocialConfig.NormEntry> allNorms,
            Collection<String> activeCognitiveSubjects,
            Collection<String> activeInventory,
            int maxNorms) {
        // 1. Extract match words from cognitive subjects and inventory
        // 2. Score each norm: priority + contextBoost + inventoryBoost
        // 3. Sort by score desc
        // 4. Take top maxNorms
        // 5. Return List<NormEntry> (callers don't need the score)
    }
}
```

The method returns `List<NormEntry>` not `List<ScoredNorm>` — callers render norm text, they don't need scores. `ScoredNorm` is an internal implementation detail for sorting.

The old `filter()` method is removed — all callers go through `score()`.

### ManorContextStrategy

```java
public List<SocialConfig.NormEntry> selectNorms(
        List<SocialConfig.NormEntry> norms,
        Collection<String> activeCognitiveSubjects,
        Collection<String> activeInventory,
        CognitiveBudget budget) {
    return ManorNormFilter.score(norms, activeCognitiveSubjects, activeInventory, budget.maxNorms());
}
```

Renamed from `filterNorms()` to `selectNorms()` — reflects the scoring semantics. Passes `budget.maxNorms()` through.

### CharacterCognition.renderCognitiveSections()

Changes to the norm filtering call:

```java
// Before:
var filteredNorms = contextStrategy.filterNorms(socialConfig.norms(), nearbyAgentIds, character.inventory());

// After:
var budget = contextStrategy.budgetFor(nearbyAgentIds.size(), /* arousal */ 0.5, /* activeGoals */ 0);
var activeEntityNames = agentNames.values();
var selectedNorms = contextStrategy.selectNorms(socialConfig.norms(), activeEntityNames, character.inventory(), budget);
```

**Arousal and activeGoals**: The current `renderCognitiveSections` doesn't have access to mood arousal or active goal count. For the initial implementation, use defaults (arousal 0.5, goals 0). When CognitionCore is fully wired (#54 pipeline completion), these values flow from the mood orchestrator and goal evaluator. The budget still adapts to nearby count immediately.

### Signature Change: renderCognitiveSections

The existing method receives `Collection<String> nearbyAgentIds` and `Map<String, String> agentNames`. Both are already available. No signature change needed — `agentNames.values()` provides the display names.

---

## Testing

### Unit Tests (ManorNormFilterTest)

| Test | Asserts |
|------|---------|
| contextRelevantNormsRankedAboveGeneral | Norm mentioning "Penelope" scores higher when "Penelope Pitstop" in cognitive subjects |
| generalNormsIncludedWithinBudget | Norms with no entity reference still appear when budget allows |
| budgetGatesNormCount | Returns at most maxNorms entries |
| inventoryMatchBoostsRelevance | Norm mentioning "treasure" boosted when "treasure" in inventory |
| shortWordsIgnored | Words < 3 chars from entity names don't trigger matching |
| caseInsensitiveMatching | "penelope" in norm matches "Penelope Pitstop" entity |
| emptyContextReturnsTopByPriority | No cognitive subjects → all norms scored by priority only |
| multipleCognitiveSubjectsMatchIndependently | Norms matching different entities both get boosted |
| noDoubleBoost | Norm matching both a cognitive subject and inventory gets max one boost per source |
| priorityBreaksTiesAmongBoosted | Among boosted norms, higher priority wins |

### Unit Tests (ManorContextStrategyTest)

| Test | Asserts |
|------|---------|
| selectNormsDelegatesToFilter | selectNorms passes budget.maxNorms() through |
| budgetForSituationAffectsNormCount | Different situations produce different maxNorms |

### Integration Tests

| Test | Asserts |
|------|---------|
| renderCognitiveSectionsFiltersNormsByContext | Social Rules section only contains relevant norms when budget is tight |
| fullTickNormSelectionUsesEntityNames | End-to-end: ScenarioOrchestrator tick produces context-appropriate norms |

### Existing Test Impact

- `ManorNormFilterTest.sortsByPriorityDescending` — update: now calls `score()` with empty subjects and adequate budget; priority sort preserved
- `ManorNormFilterTest.emptyNormsReturnsEmpty` — update: calls `score()` with empty norms
- `ManorNormFilterTest.preservesAllNorms` — update or remove: budget may now limit count
- `CharacterCognitionTest.beliefsAndNormsRendered` — update: verify norms still render with display names
- `CharacterCognitionTest.allFourSectionsForFullyConfiguredCharacter` — update: same

---

## What's NOT in Scope

- Emergent norm activation from absent entities (cognitive context beyond physically nearby) — requires independent query path, future enhancement
- Norm learning or evolution through experience — norms are static SocialConfig entries, not mindmap nodes
- Embedding-based or LLM-based matching substrate — current word matching is sufficient, substrate is swappable
- Arousal/goal wiring into renderCognitiveSections — depends on full CognitionCore pipeline (#54 completion); defaults used for now
- Norm deactivation or suppression based on trust/relationship changes

---

## Design Decisions

- D1: Cognitive context source — reuse tick queries (zero additional CognitiveProfile calls)
- D2: Relevance boost model — binary presence (entity known → full boost)
- D3: Matching substrate — case-insensitive word match (≥3 chars)
- D4: Integration point — scoring + budget gating in ManorNormFilter

Full decision records: `specs/issue-57-norm-relevance-scoring/decisions.md`

---

## References

- ManorNormFilter.java:6-18 — current implementation (sorts by priority, ignores params)
- ManorContextStrategy.java:12-16 — current filterNorms delegation
- CharacterCognition.java:94-128 — renderCognitiveSections with norm rendering at line 122
- ScenarioOrchestrator.java:243-257 — tick loop with nearbyIds and agentNameMap construction
- CognitiveBudget.java:3-22 — maxNorms with situational adaptation
- SocialConfig.java:31-34 — NormEntry record
- social-config.yaml — actual norm data with character references
- #54 design spec §Per-Tick Query Flow — cognitive pipeline architecture
- GE-20260912-ff141b — neocortex cognitive stack components
- GE-20260912-c4c279 — Thing trait projections for cognitive state
- GE-20260914-e3cb03 — profile data over prose directives
