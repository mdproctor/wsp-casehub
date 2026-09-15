# Decisions — Phase B: CognitiveProfile Queries (#54)

## D1: CognitionCore adoption — Hybrid with local strategy adapter

**Choice:** Adopt CognitionCore from blocks#261 inside CharacterCognition. CognitionCore owns tick orchestration, mood, drives, and prompt section rendering. CharacterCognition becomes a thin adapter providing game-world context (room-aware norm filtering, proximity-scoped trust queries, action-type interaction mapping). The adapter implements the contract of the proposed CognitionContextStrategy SPI (blocks#282) locally — when the SPI lands upstream, the adapter becomes the real implementation with no throwaway work.
**Alternatives:**
- Keep CharacterCognition as standalone — contradicts D1 from Phase A ("wire existing platform, don't build"). Misses mood, narrative, and standard orchestration.
- Full CognitionCore replacement (no CharacterCognition) — CognitionCore doesn't know about rooms, nearby characters, or inventory. Callers would need to do context filtering externally, scattering game logic.
- Defer CognitionCore adoption — lower risk but two integration steps instead of one, and doesn't showcase platform composition.
**Rationale:** Wacky-manor is the first real consumer of the cognitive stack. Using the platform component validates it. The gap (situational context filtering) is universal, not game-specific — filed as blocks#282. The adapter pattern means zero throwaway work.
**Trade-offs:** CharacterCognition gains a dependency on CognitionCore. If CognitionCore's API changes, the adapter needs updating. Mitigated by the SPI proposal — once blocks#282 lands, the contract is stable.
**Sources:** blocks#261 CognitionCore, blocks#282 CognitionContextStrategy SPI proposal, Phase A D1 (wire existing platform)
**Exploration:** deep-analysis
**Status:** captured

## D2: Mapping layer — adapter in CharacterCognition

**Choice:** CharacterCognition queries CognitiveProfile, projects nodes via Thing traits (Belieflike, Evaluative, etc.), then maps to blocks renderer types (Belief, TrustSummary, Principle, SocialNorm) inline. ~4 type conversions, local to the adapter.
**Alternatives:**
- Shared mapper in blocks — reusable but creates a blocks dependency on neocortex Thing API. Cross-module coupling.
- Renderers accept Thing nodes directly — tightest coupling, zero mapping code, but modifies the blocks#260 API that just landed.
**Rationale:** The mapping is small and app-local. If multiple consumers need it later, extract to blocks then. YAGNI now.
**Trade-offs:** Each consumer duplicates the mapping. Acceptable at one consumer.
**Sources:** blocks#260 renderer signatures, neocortex Thing trait interfaces
**Exploration:** quick
**Status:** captured

## D3: Static-to-dynamic transition — seed on first tick

**Choice:** Write SocialConfig initial beliefs/norms to the mindmap at scenario start. After seeding, CognitiveProfile queries are the sole source of truth for beliefs, trust, and norms. Drives stay static (no mindmap representation yet — drives are personality, not accumulated knowledge).
**Alternatives:**
- Overlay (static + dynamic merged) — risk of duplication if initial beliefs get re-created during consolidation. Requires dedup.
- Static until first consolidation, then switch — clean cut but no dynamic cognition until first sleep cycle.
**Rationale:** One source of truth after setup. Seeding makes initial beliefs queryable via the same CognitiveProfile API as accumulated knowledge. No special-case rendering paths.
**Trade-offs:** Requires a seeding step at scenario start. If seeding fails, character starts with no beliefs. Mitigated by fail-fast at startup.
**Depends on:** D1 (CognitionCore manages the query path after seeding)
**Sources:** Phase A D5 (three-tier model), SocialConfig.forCharacter()
**Exploration:** quick
**Status:** captured

## D4: Query scope — adaptive attention via TemporalFocus

**Choice:** Wire neocortex TemporalFocus from the start for attention-based cognitive item selection. Per tick: query all cognitive nodes for the character, wrap as TemporalEntry, score via TemporalFocus.focus() using recency + affect trajectory, take top items within a situational budget per category (beliefs, trust, norms). Budget adapts to situation density (nearby count), arousal, and goal pressure via heuristics in the CognitionContextStrategy adapter.
**Alternatives:**
- Nearby-scoped (static filter) — too narrow: HC should think about Penelope even when she's not in the room if his scheme is active.
- Full cognitive state (no filtering) — too broad: 50 accumulated beliefs dilute what matters. Wastes prompt tokens.
- Top-K by salience with fixed K — rigid: doesn't adapt to situation density or arousal level.
- Simple heuristics first, TemporalFocus later — pragmatic but defers the correct solution. TemporalFocus is stateless, synchronous, and callable per tick — no reason to defer.
**Rationale:** This is fundamentally an attention problem, not a filtering problem. Human cognition doesn't load all beliefs into working memory — what surfaces depends on recency, affect, goals, and situation. TemporalFocus already implements this scoring model. Wiring it from the start means correct behavior from day one and validates the neocortex attention infrastructure.
**Trade-offs:** Requires AffectTrajectory data per entity. If no trajectory data exists (early game, no consolidation yet), TemporalFocus falls back to recency-only scoring — graceful degradation.
**Depends on:** D1 (CognitionContextStrategy provides budget sizing)
**Sources:** neocortex TemporalFocus, AttentionItem, TemporalFocusConfig
**Exploration:** deep-analysis
**Status:** captured
