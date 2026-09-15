# HANDOFF — 2026-09-15

## Last Session

Issue #55 (dialogue knowledge extraction) designed and implemented — 3 tasks across 2 batches. Characters now learn structured knowledge from conversations via MindMapExtractor. Key pivot: decision review found ConversationBridge.process() ignores principalId and hardcodes confidence — switched to MindMapExtractor.extract() directly with manual node creation per listener (same pattern as ManorCognitiveSeeder). 450 tests, 12 new unit tests + 2 integration tests, 1 pre-existing @QuarkusTest boot error (AgentLangchain4jProperties).

**What was built:**
- `determineListeners()` — listener determination for all 4 dialogue types (directed, PULL_ASIDE, room, aside) with `perception` tag gating for overhearing
- `isExtractableDialogue()` — non-verbal character filter (skips "Hehehehe!" etc.)
- `extractDialogueKnowledge()` — calls MindMapExtractor.extract() once per dialogue event, creates per-listener nodes with principalId and ConfidenceOrigin (STATED 0.8 direct / INFERRED 0.7 overheard)
- Tick loop wiring — extraction runs after all dialogue publication, before action resolution. PULL_ASIDE exchange text collected separately.

## Immediate Next Step

Issue #56: Move SocialConfig to Eidos extensionData. Queue at position 1/6 — advance with `work next`.

## Decisions This Session

- D1: Learner scope — target + perception-gated overhearing (reuses existing `perception` tag from issue-38)
- D2: Confidence model — ConfidenceOrigin (STATED 0.8 / INFERRED 0.7), not trustFormationRate (conflates trust speed with belief confidence)
- D3: Wiring in ScenarioOrchestrator — private method, not dispatcher (blocking LLM call)
- D4: PULL_ASIDE — extract once, scope to both participants (no doubled LLM cost)
- D5: MindMapExtractor.extract() instead of ConversationBridge.process() — principalId/confidence control
- D6: Three-tier deviation — dialogue knowledge writes directly to Tier 3 (immediate recall, not deferred to consolidation)

## Cross-Module

**Upstream issues to file:**
- neocortex: MindMapExtractor.parseOnly() — extract without persisting (eliminates orphaned global nodes)
- neocortex: ConversationBridge.process() — add principalId and custom confidence parameters

**CDI gap (unchanged):** MindMapExtractor, CognitiveProfile, and MindMapStore are not CDI-discoverable in wacky-manor's Quarkus context. All wired via `Instance<>` with graceful degradation.

**Pre-existing test failure:** PersonalityCompositionVerificationTest fails with AgentLangchain4jProperties boot error — all @QuarkusTest tests affected. Not related to this branch's work.

## References

- Plan: `plans/2026-09-15-dialogue-knowledge-extraction.md`
- Spec: `specs/issue-55-conversation-bridge/2026-09-15-conversation-bridge-design.md`
- Decisions: `specs/issue-55-conversation-bridge/decisions.md` (D1-D6)
- Decision review: `/Users/mdproctor/reviews/casehub-examples/issue-55-conversation-bridge-decision-20260915-010958/`
- Spec review: `/Users/mdproctor/reviews/casehub-examples/issue-55-dialogue-extraction-20260915-021007/`
