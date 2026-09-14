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
