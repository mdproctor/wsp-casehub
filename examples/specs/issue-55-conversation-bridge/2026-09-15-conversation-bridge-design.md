# Dialogue Knowledge Extraction — Design Spec

**Issue:** casehubio/examples#55
**Epic:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Date:** 2026-09-15
**Status:** Draft

---

## Problem

Characters in Wacky Manor can speak to each other but learn nothing from conversations. When HC tells Penelope "the poison is in the Ballroom," Penelope's mindmap is unchanged — she can't recall this fact on later ticks. The dialogue text is published as events and recorded as experience memories, but no structured knowledge (entities, relationships, beliefs) is extracted into the mindmap where CognitiveProfile queries can surface it.

---

## Architecture

### MindMapExtractor as the Extraction Engine

ConversationBridge.process() was the original candidate, but decision review found it cannot support per-character scoping: principalId is ignored, confidence is hardcoded to 1.0, and extraction is async with no completion signal. Instead, we use MindMapExtractor.extract() directly — it's the LLM extraction engine that ConversationBridge delegates to, and it's synchronous.

**Important caveat:** MindMapExtractor.extract() is NOT a pure function. It calls `applyExtraction()` internally, which creates global nodes (no principalId, provenance "llm-extraction") and edges in the MindMapStore before returning ExtractionResult. Our code then creates additional per-listener nodes with proper principalId. This produces orphaned global nodes that don't affect CognitiveProfile queries (filtered by principalId) but are waste. File upstream neocortex issue for a `parseOnly()` method that extracts without persisting.

```
ScenarioOrchestrator (per tick)
  ├── dialogue events published (PULL_ASIDE, directed, room)
  └── extractDialogueKnowledge() — after all dialogue for tick
        ├── determine listeners per dialogue type (D1)
        ├── filter non-verbal dialogue (skip if no extractable content)
        ├── MindMapExtractor.extract(text, tenantId, nearbyNames)
        │     → ExtractionResult(entities, relationships, contradictions)
        │     ⚠️ also creates orphaned global nodes (no principalId)
        └── per listener: create nodes via MindMapStore
              ├── principalId = PrincipalId.agent(listenerId)
              ├── confidence = Confidence.stated(0.8) for STATED
              │                Confidence.inferred(0.7) for INFERRED
              └── provenance = "dialogue-extraction"
```

### Three-Tier Write Discipline Deviation (D6)

The Phase A spec established that "mindmap is never written to during the tick loop." This issue deviates from that rule. Dialogue knowledge is immediate recall — when someone tells you something, you know it now. This is fundamentally different from experiential memory consolidation (Tier 2 → Tier 3 graduation, which happens during "sleep" cycles).

Precedent: ManorCognitiveSeeder also writes directly to Tier 3 at scenario start.

Consolidation phases (MergeDetectionPhase, CommunitySummaryPhase) are designed to handle incremental graph changes. Dialogue-extracted nodes have distinct provenance (`dialogue-extraction`) so consolidation can identify and process them appropriately.

### Listener Determination (D1)

| Dialogue type | Direct listeners | Perception-gated |
|---|---|---|
| **PULL_ASIDE** | Both participants | Nobody (private exchange) |
| **Directed dialogue** | Target character | Characters in room with `perception` tag |
| **Room dialogue** | All characters in room | N/A (already all present) |
| **Aside** | Nobody | Nobody |

The `perception` tag is dynamically read from each character's Eidos descriptor capabilities. Characters with perception in current descriptors include Hooded Claw, Ant Hill Mob, Muttley (composite only), and Private Meekly (composite only). The implementation uses the dynamic tag lookup, not a hardcoded list.

### Distinguishing PULL_ASIDE from Directed Dialogue (D4)

ManorEvent records from PULL_ASIDE exchanges are structurally identical to regular directed dialogue — both have type `"dialogue"`, null actionType, and `concealed=false`. The extraction method cannot distinguish them from event metadata alone.

Fix: Track exchange events separately in the tick loop. The PULL_ASIDE block already maintains a `suppressed` set of character IDs. Collect the exchange events from `ExchangeRunner.run()` into a separate list, pass them to `extractDialogueKnowledge()` with an `isExchange` flag. Exchange events skip the perception-gated overhearing logic — both participants get STATED confidence, nobody overhears.

### Confidence Model (D2)

| Listener type | ConfidenceOrigin | Confidence value |
|---|---|---|
| Direct target | STATED | 0.8 |
| PULL_ASIDE participant | STATED | 0.8 |
| Perception-gated overhearer | INFERRED | 0.7 |

The STATED confidence of 0.8 matches ManorCognitiveSeeder's seeded belief confidence. Dialogue-learned beliefs should not outrank initial convictions. Per-speaker trust modulation (e.g., "I trust HC less than Penelope") is deferred to #336 (ExperienceConsolidation), when real trust edges from interaction history become available.

### Non-Verbal Character Filter

Characters with non-verbal constraints (Muttley, Sawtooth, Blubber Bear, Little Gruesome) produce dialogue text that is pure sounds ("Hehehehehehe!", "SQUEAK squeak squeak SQUEAK!"). Running MindMapExtractor on these texts wastes an LLM call for zero knowledge extraction.

Filter: Before calling extract(), check if the dialogue text contains recognisable words. Skip extraction for text that is purely exclamations, sounds, or single-word utterances. The check is lightweight — a simple heuristic (e.g., text length > 10 characters AND contains at least 3 alphabetic words) avoids the LLM call entirely.

### Per-Tick Extraction Flow

For each tick in `runAutonomousTicks()`:

1. **Collect dialogue events** — track which dialogue texts were generated this tick, their speakers, targets, and rooms. Separate PULL_ASIDE exchange events from regular dialogue.
2. **After all dialogue is published** — call `extractDialogueKnowledge()`
3. **For each dialogue event with extractable text:**
   a. Skip non-verbal character dialogue (heuristic filter)
   b. Determine listeners (direct + perception-gated per D1)
   c. If no listeners, skip
   d. Build `recentEntityNames` from all character names in the room (including the speaker)
   e. Call `MindMapExtractor.extract(text, tenantId, recentEntityNames)` — one LLM call
   f. For each listener: create nodes from ExtractionResult via MindMapStore
4. **Continue to action resolution** — extracted knowledge is immediately available for CognitiveProfile queries on subsequent ticks

### Node Creation Pattern

Follows the ManorCognitiveSeeder pattern for per-character node creation:

```java
for (ExtractedEntity entity : result.entities()) {
    var nodeInput = NodeInput.of(entity.name(), subgraphId)
        .withConfidence(Confidence.stated(0.8, Instant.now()))
        .withProvenance("dialogue-extraction")
        .withPrincipalId(PrincipalId.agent(listenerId))
        .withProperties(entity.properties());
    if (entity.subgraphType() != null) {
        nodeInput = nodeInput.withTraits(Set.of(entity.subgraphType()));
    }
    mindMapStore.addNode(nodeInput, tenantId);
}
```

For perception-gated overhearers, use `Confidence.inferred(0.7, Instant.now())` instead.

For relationships (edges between extracted entities):
```java
for (ExtractedRelationship rel : result.relationships()) {
    // resolve source/target node IDs from entity names within this listener's nodes
    mindMapStore.addEdge(edgeInput, tenantId);
}
```

Contradictions from `result.contradictions()` are logged but not acted on — contradiction resolution is future work.

### PULL_ASIDE Exchange Handling (D4)

ExchangeRunner.run() returns `List<ManorEvent>`. For extraction:

1. Collect all event `detailedDescription()` values from the exchange
2. Concatenate into a single text block with speaker attribution: `"Speaker: text\n"`
3. Call `MindMapExtractor.extract()` once on the concatenated text
4. Create nodes for both participants with ConfidenceOrigin.STATED (confidence 0.8)
5. Both get the same ExtractionResult — subjective focus differentiation happens via TemporalFocus attention scoring at query time

### Subgraph Strategy

Use the existing `beliefs-{agentId}` subgraph (created by ManorCognitiveSeeder). Dialogue-extracted knowledge IS beliefs — "the poison is in the Ballroom" is a belief learned from dialogue. The provenance field (`dialogue-extraction` vs `manor-seed`) distinguishes origin when needed.

If a listener has no existing beliefs subgraph (character without SocialConfig initial beliefs, or a character added after seeding), create one on first extraction: `MindMapStore.createSubgraph(new SubgraphInput("beliefs-" + listenerId, "cognitive", null), tenantId)`. The subgraph type MUST be `"cognitive"` to match ManorCognitiveSeeder's convention. Extract subgraph naming to a shared constant or helper to prevent drift.

Cache subgraph IDs per agent to avoid repeated lookups.

### LLM Cost and Latency

MindMapExtractor.extract() is a synchronous blocking LLM call (~1-3 seconds). In a typical tick:
- 3-5 dialogue events (directed + room) → 3-5 LLM calls → 3-15 seconds
- PULL_ASIDE exchanges → 1 additional LLM call per exchange

This adds 3-15 seconds of blocking time per tick. For a game running at ~10-30 second tick intervals (LLM invocation + action resolution), this is a 10-50% increase in tick duration.

Mitigation: Run extractions on virtual threads (`Thread.ofVirtual()`) in parallel. The extractions are independent — each processes a different dialogue text. MindMapStore is thread-safe (`@ApplicationScoped`). This reduces the blocking time to the duration of the longest single extraction.

---

## What Needs Building

### Modified types

| Type | Change |
|---|---|
| `ScenarioOrchestrator` | Inject `Instance<MindMapExtractor>`. Add `extractDialogueKnowledge()` private method. Track exchange events separately. Call extraction after dialogue publication in `runAutonomousTicks()`. |

### No new types

The extraction logic is a private method in ScenarioOrchestrator (~60-80 lines). No new classes needed.

### Upstream follow-ups

- File neocortex issue: `MindMapExtractor.parseOnly()` — extract without persisting, returns ParsedExtraction instead of ExtractionResult. Eliminates orphaned global nodes.
- File neocortex issue: `ConversationBridge.process()` — support principalId and custom confidence parameters. Enables simpler consumer API for per-character scoping.

---

## Testing

**Unit tests (no CDI):**
- Listener determination logic: directed → target only; directed + perception → target + overhearers; room → all present; PULL_ASIDE → both participants; aside → nobody
- Non-verbal dialogue filter: skip "Hehehehe!", accept "The poison is in the Ballroom"
- Exchange text concatenation from ManorEvent list
- Confidence selection: STATED for direct, INFERRED for overheard

**Integration tests (`@QuarkusTest`):**
- MindMapExtractor injectable via `Instance<>`
- Extract from dialogue text → ExtractionResult contains expected entities
- Node creation with principalId and ConfidenceOrigin — verify via MindMapStore query
- Perception-gated overhearing: character without `perception` tag doesn't get nodes; character with tag does
- Subgraph creation for unseeded listener

**Existing test preservation:** All 436 tests pass unchanged.

---

## Upstream Dependencies

| Repo | Issue | Status | Impact |
|---|---|---|---|
| neocortex | #322 | CLOSED | Typed nodes — improves what MindMapExtractor can extract |
| blocks | #261 | CLOSED | CognitionCore — already wired in #54 |

No blockers.

---

## What's NOT in Scope

- Contradiction resolution — contradictions are logged, not resolved
- Per-speaker trust modulation — requires trust edges from ExperienceConsolidation (#336)
- MindMapExtractor.parseOnly() — upstream neocortex issue (follow-up)
- ConversationBridge API improvement — upstream neocortex issue (follow-up)
- Moving SocialConfig to Eidos extensionData (#56)
- Context-based ManorNormFilter (#57)
- LLM eval tests (#59)
- SocialComparison integration (#60)

---

## Design Decisions

- D1: Learner scope — target + perception-gated overhearing
- D2: Confidence model — ConfidenceOrigin (STATED 0.8 for direct, INFERRED 0.7 for overheard)
- D3: Wiring approach — direct method in ScenarioOrchestrator
- D4: PULL_ASIDE — extract once, scope to both participants, track separately from regular dialogue
- D5: Extraction API — MindMapExtractor.extract() instead of ConversationBridge.process()
- D6: Three-tier deviation — immediate dialogue knowledge write to Tier 3

Full decision records: `specs/issue-55-conversation-bridge/decisions.md`

---

## References

- MindMapExtractor.extract() API — synchronous extraction, creates global nodes internally (spec review R1-02)
- MindMapExtractor.applyExtraction() — decompiled, creates nodes before returning ExtractionResult
- ConversationBridge.process() API — decompiled, principalId/confidence gaps (decision review R1-05)
- ManorCognitiveSeeder.java — node creation pattern with principalId and Confidence.stated(0.8)
- ScenarioOrchestrator.java — existing Instance<MindMapStore>, Instance<ConsolidationScheduler> injection patterns
- ObservationService.java:69-74 — `perception` tag gate for concealed actions
- Issue-38 spec §Capability-Gated Overhearing — perception tag designed for dialogue overhearing
- descriptors-baseline.yaml — perception tag assignments (Hooded Claw, Ant Hill Mob)
- descriptors-composite.yaml — Muttley, Private Meekly perception tags (composite only)
- GE-20260912-ff141b — Neocortex cognitive stack documentation
- GE-20260912-c4c279 — Thing trait projections as typed facades
- Phase A design spec: `specs/issue-52-social-cognition-layer/2026-09-11-social-cognition-design.md`
- Phase A decisions: `specs/issue-52-social-cognition-layer/decisions.md` (D1-D7)
- Issue #54 design spec: `specs/issue-54-cognitive-profile-queries/2026-09-14-cognitive-profile-queries-design.md`
