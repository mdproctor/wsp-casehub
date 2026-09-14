# HANDOFF — 2026-09-14

## Last Session

Phase A (#52) closed and merged to main — consolidation sleep mechanic, all 7 tasks across 4 batches. Phase B epic (#53) filed with 7 child issues (#54-#60) plus upstream neocortex#336 (ExperienceConsolidationPhase) and blocks#282 (CognitionContextStrategy SPI).

Issue #54 (CognitiveProfile queries) designed and implemented — 5 tasks across 3 batches. CharacterCognition now composes CognitionCore + CognitiveProfile + TemporalFocus. CognitiveBudget provides adaptive attention heuristics. ManorCognitiveSeeder writes initial beliefs to mindmap. ScenarioOrchestrator injects CognitiveProfile and MindMapStore via `Instance<>` (optional — CDI beans not discoverable in current Quarkus context). 436 tests passing.

## Immediate Next Step

Issue #55: Wire ConversationBridge for dialogue → knowledge extraction. Queue advanced — #55 is active in `.plan`.

## Decisions This Session

- D1: CognitionCore adoption — hybrid with local ManorContextStrategy adapter (blocks#282 SPI contract)
- D2: Mapping layer — Thing traits → blocks types inline in CharacterCognition
- D3: Seed beliefs at scenario start, mindmap authoritative after
- D4: Adaptive attention via TemporalFocus (not custom heuristics) — wired from the start

## Cross-Module

**Upstream issues filed:**
- neocortex#336 — ExperienceConsolidationPhase (Tier 2 → Tier 3 graduation)
- blocks#282 — CognitionContextStrategy SPI (situational context filtering for CognitionCore)

**Upstream landed (verified this session):**
- neocortex#322 — cognitive node types + traits (CLOSED)
- blocks#260 — cognitive observation renderers (CLOSED)
- blocks#261 — CognitionCore (CLOSED)

**CDI gap:** CognitiveProfile and MindMapStore are not CDI-discoverable in wacky-manor's Quarkus context. Wired via `Instance<>` with graceful degradation. Dynamic queries activate when these beans become available (likely requires neocortex Quarkus extension or explicit producers).

## References

- Plan: `plans/2026-09-14-cognitive-profile-queries.md`
- Spec: `specs/issue-54-cognitive-profile-queries/2026-09-14-cognitive-profile-queries-design.md`
- Decisions: `specs/issue-54-cognitive-profile-queries/decisions.md` (D1-D4)
- Blog: `blog/2026-09-13-mdp01-the-mansion-learns-to-sleep.md`
