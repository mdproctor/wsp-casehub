# Decisions — Issue #55: Wire ConversationBridge for dialogue knowledge extraction

## D1: Learner scope — target + perception-gated overhearing

**Choice:** Direct target always extracts knowledge from dialogue. Characters with the `perception` capability tag in the same room also extract (overhearing). Both participants extract from PULL_ASIDE exchanges. All present extract from room dialogue. Asides are internal — nobody extracts.
**Alternatives:**
- Listener-only (no overhearing) — simpler but misses the personality differentiation that makes Hooded Claw, Ant Hill Mob, and Muttley interesting observers
- All present — too noisy, high LLM cost, doesn't model the difference between directed and room dialogue
- Trait-scaled graduated overhearing — more realistic but requires a new SocialCognitionDefaults field upstream; `perception` tag is already designed for this (issue-38 spec)
**Rationale:** The `perception` tag already gates concealed-action visibility in ObservationService (issue-38). Reusing it for dialogue overhearing is consistent and requires zero new trait infrastructure. Hooded Claw (villain perception), Ant Hill Mob (suspicious protectors), and Muttley (keen nose, composite only) are the characters who should notice what others are saying.
**Trade-offs:** Binary gate — either you overhear or you don't. No graduated "how much did you catch?" mechanic. Acceptable for now; the open tag vocabulary means a future `acute-hearing` or `social-awareness` tag could add gradation.
**Sources:** ObservationService.java:69-74 (perception gate), issue-38 spec §Capability-Gated Overhearing, descriptors-baseline.yaml (perception tags)
**Exploration:** quick
**Status:** captured

## D2: Confidence model — ConfidenceOrigin-based with overhearing discount

**Choice:** Extracted knowledge confidence uses ConfidenceOrigin enum values: STATED (confidence ~0.8) for directly addressed dialogue targets, INFERRED (confidence ~0.7) for perception-gated overhearers. Per-speaker trust modulation deferred to #336 (ExperienceConsolidation) when real trust edges become available.
**Alternatives:**
- trustFormationRate as confidence proxy — personality-derived from tick 1, but conflates "how fast I form trust" with "how much I believe this statement." A character with low trust formation rate might still believe factual claims.
- Fixed baseline, trust later — all speakers get a fixed default confidence. Simpler but ConfidenceOrigin is already in the platform.
- Relationship-aware from start — seed initial trust edges. Most expressive but couples seeding to trust topology.
**Rationale:** ConfidenceOrigin is an existing platform enum (STATED, INFERRED, SPECULATED, UNKNOWN) that directly maps to the overhearing distinction. STATED = "I was told this directly." INFERRED = "I overheard this." Clean semantic mapping, no conflation of unrelated concepts. STATED confidence at 0.8 matches ManorCognitiveSeeder's seeded belief confidence — dialogue-learned beliefs should not outrank initial convictions.
**Trade-offs:** No personality differentiation at the confidence level from tick 1 — all characters who are directly told something assign the same confidence. Acceptable: personality influences how characters *act on* beliefs (via drives, norms, principles), not how confidently they record them.
**Depends on:** D1 (learner scope determines who gets which confidence level)
**Sources:** ConfidenceOrigin enum (neocortex cognitive-api), MindMapConfidenceDefaults, ManorCognitiveSeeder (0.8 confidence), decision review R1-05, R1-06
**Exploration:** quick
**Status:** revised (review R1-05, R1-06)

## D3: Wiring approach — direct method in ScenarioOrchestrator

**Choice:** Add a private method `extractDialogueKnowledge()` in ScenarioOrchestrator. Inject MindMapExtractor as a CDI `Instance<>` (consistent with existing MindMapStore/ConsolidationScheduler injection pattern). The method determines listeners per dialogue type, applies the perception gate, calls MindMapExtractor.extract() once per dialogue event, then creates per-listener nodes via MindMapStore with appropriate principalId and ConfidenceOrigin. Called after all dialogue events for a tick are published, before action resolution.
**Alternatives:**
- ManorEventDispatcher hook — dispatcher is the single choke point for all dialogue. But MindMapExtractor.extract() is a synchronous blocking LLM call — putting it in the dispatcher would block event publishing for all listeners.
- ManorDialogueExtractor — dedicated class. Clean separation but adds a type for wiring logic.
- CharacterCognition.processDialogue() — mixes read/write paths, grows constructor.
**Rationale:** ScenarioOrchestrator is the orchestration hub. MindMapExtractor.extract() is blocking (LLM call) — the orchestrator controls timing, running extraction after dialogue publication when it won't block event delivery. The extraction runs once per dialogue event (not per listener), then nodes are created per listener — addressing the review's doubled-LLM-cost concern.
**Trade-offs:** ScenarioOrchestrator grows by ~60-80 lines. Extraction in three code paths (PULL_ASIDE, directed, room) but a helper method handles the common extract-then-scope pattern.
**Depends on:** D5 (MindMapExtractor as the extraction API)
**Sources:** ScenarioOrchestrator.java, MindMapExtractor CDI API, decision review R1-08
**Exploration:** quick
**Status:** revised (review R1-05, R1-08, R1-09)

## D4: PULL_ASIDE handling — extract once, scope to both participants

**Choice:** Concatenate all exchange turns from ExchangeRunner into a single text block. Call MindMapExtractor.extract() once on the concatenated text. Create nodes for both participants via MindMapStore, each with their own principalId and ConfidenceOrigin.STATED. Track exchange events separately from regular dialogue using the suppressed set already maintained in the tick loop.
**Alternatives:**
- Extract per participant — calls extract() twice with the same text. Doubled LLM cost for identical results.
- Per-turn extraction — call extract() per dialogue turn. Higher LLM cost (N calls) and fragments context the extractor needs.
- Summary extraction — summarise first, then extract. Token-efficient but adds a step and loses detail.
**Rationale:** The LLM extractor works best with full context — entity references span turns. Extracting once and scoping to both participants via separate node creation is the correct cost/quality trade-off.
**Trade-offs:** Both participants get the same extraction results. Subjective focus differentiation happens via TemporalFocus attention scoring at query time.
**Depends on:** D5 (MindMapExtractor for synchronous extraction), D2 (ConfidenceOrigin for PULL_ASIDE = STATED)
**Sources:** ExchangeRunner.run() returns List<ManorEvent>, MindMapExtractor.extract() API, decision review R1-12
**Exploration:** quick
**Status:** revised (review R1-04, R1-12)

## D5: Extraction API — MindMapExtractor.extract() instead of ConversationBridge.process()

**Choice:** Use MindMapExtractor.extract(text, tenantId, recentEntityNames) directly instead of ConversationBridge.process(). MindMapExtractor is the LLM extraction engine that ConversationBridge delegates to. Calling it directly gives synchronous extraction results (ExtractionResult with entities, relationships, contradictions) and full control over node creation with proper principalId, ConfidenceOrigin, and per-listener scoping.
**Alternatives:**
- ConversationBridge.process() — the original plan. But review found: (a) confidence hardcoded to 1.0, (b) principalId parameter ignored (nodes get null), (c) async extraction with no completion signal. Can't support per-character scoping or custom confidence.
- Patch ConversationBridge upstream — file a neocortex issue to add principalId/confidence support. Correct long-term but blocks this issue on upstream work.
- Post-process ConversationBridge nodes — call process(), then retroactively adjust nodes. Races against async extraction — fragile and complex.
**Rationale:** MindMapExtractor.extract() is synchronous, returns structured ExtractionResult, and is CDI-injectable. The seeder pattern (ManorCognitiveSeeder) already demonstrates manual node creation via MindMapStore with proper principalId and confidence.
**Trade-offs:** MindMapExtractor.extract() is NOT a pure function — it creates global nodes internally (no principalId) before returning ExtractionResult. This produces orphaned global nodes alongside our per-listener nodes. These orphans don't affect CognitiveProfile queries (filtered by principalId) but are waste. File upstream for a `parseOnly()` method that extracts without persisting.
**Sources:** MindMapExtractor.extract() API (javap + decompilation), ConversationBridge.process() decompilation (decision review R1-05), ManorCognitiveSeeder.java (seeder pattern), spec review R1-02
**Exploration:** quick (surfaced by decision review)
**Status:** revised (spec review R1-02)

## D6: Three-tier write discipline deviation — immediate dialogue knowledge

**Choice:** Write dialogue-extracted knowledge directly to the mindmap (Tier 3) during the tick loop. This deviates from the Phase A three-tier model which said "mindmap is never written to during the tick loop."
**Alternatives:**
- Buffer in Tier 2, graduate during consolidation — preserves Phase A discipline. But characters wouldn't know what they were told until after the next sleep cycle. "HC tells Penelope the poison is in the Ballroom" → Penelope forgets until she sleeps on it.
- Async write (ConversationBridge pattern) — fire-and-forget, eventually consistent. But principalId/confidence can't be controlled (D5).
**Rationale:** Dialogue knowledge is immediate recall — when someone tells you something, you know it now. This is fundamentally different from experiential memory consolidation (Tier 2 → Tier 3 graduation). ManorCognitiveSeeder also writes directly to Tier 3 at scenario start. The three-tier discipline was designed for processing accumulated experiences, not direct communication.
**Trade-offs:** Consolidation phases now contend with nodes that were written during the tick loop (between sleep cycles). Acceptable: consolidation phases (MergeDetectionPhase, CommunitySummaryPhase) are designed to handle incremental graph changes. The dialogue nodes have distinct provenance (`dialogue-extraction`) so consolidation can identify them.
**Depends on:** D5 (synchronous extraction implies synchronous writes)
**Sources:** Phase A spec §Three-tier memory model, ManorCognitiveSeeder (direct Tier 3 write precedent), spec review R1-03
**Exploration:** quick (surfaced by spec review)
**Status:** captured
