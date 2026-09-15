# HANDOFF — 2026-09-15

## Last Session

Two issues completed this session:

**Issue #55 (dialogue knowledge extraction)** — 3 tasks across 2 batches. Characters now learn structured knowledge from conversations via MindMapExtractor. Key pivot: decision review found ConversationBridge.process() ignores principalId and hardcodes confidence — switched to MindMapExtractor.extract() directly with manual node creation per listener.

What was built for #55:
- `determineListeners()` — listener determination for all 4 dialogue types with `perception` tag gating for overhearing
- `isExtractableDialogue()` — non-verbal character filter
- `extractDialogueKnowledge()` — calls MindMapExtractor.extract() once per dialogue event, creates per-listener nodes with principalId and ConfidenceOrigin (STATED 0.8 / INFERRED 0.7)
- Tick loop wiring — extraction runs after dialogue publication, before action resolution

**Issue #56 (SocialConfig to YAML)** — 2 tasks in 1 batch. SocialConfig moved from hardcoded Java map to `META-INF/eidos/social-config.yaml`. ManorSocialConfigLoader reads YAML at startup. Hardcoded CONFIGS map, forCharacter(), hasConfig() removed. SocialConfig is now a pure data record. Note: Eidos AgentDescriptor has no extensionData field — used local YAML file instead.

457 tests total, 21 new this session (12 unit + 2 integration for #55, 7 unit for #56). 1 pre-existing @QuarkusTest boot error (AgentLangchain4jProperties).

## Immediate Next Step

Issue #57: Context-based ManorNormFilter. Queue at position 3/6 — #57 is active in `.plan`.

## Decisions This Session

**#55 decisions (D1-D6):**
- D1: Learner scope — target + perception-gated overhearing
- D2: Confidence model — ConfidenceOrigin (STATED 0.8 / INFERRED 0.7)
- D3: Wiring in ScenarioOrchestrator — private method, not dispatcher (blocking LLM call)
- D4: PULL_ASIDE — extract once, scope to both participants
- D5: MindMapExtractor.extract() instead of ConversationBridge.process()
- D6: Three-tier deviation — dialogue knowledge writes directly to Tier 3

**#56 decisions (D1-D2):**
- D1: Config source — local YAML file instead of Eidos extensionData (no extensionData field in AgentDescriptor)
- D2: Loading mechanism — static loader alongside descriptors

## Cross-Module

**Upstream issues to file:**
- neocortex: MindMapExtractor.parseOnly() — extract without persisting (eliminates orphaned global nodes)
- neocortex: ConversationBridge.process() — add principalId and custom confidence parameters
- eidos: Add extensionData field to AgentDescriptor — enables embedding social config in descriptors

**CDI gap (unchanged):** MindMapExtractor, CognitiveProfile, and MindMapStore are not CDI-discoverable in wacky-manor's Quarkus context. All wired via `Instance<>` with graceful degradation.

**Pre-existing test failure:** PersonalityCompositionVerificationTest fails with AgentLangchain4jProperties boot error — all @QuarkusTest tests affected.

## References

- Plan #55: `plans/2026-09-15-dialogue-knowledge-extraction.md`
- Spec #55: `specs/issue-55-conversation-bridge/2026-09-15-conversation-bridge-design.md`
- Decisions #55: `specs/issue-55-conversation-bridge/decisions.md` (D1-D6)
- Plan #56: `plans/2026-09-15-socialconfig-yaml.md`
- Spec #56: `specs/issue-56-socialconfig-yaml/2026-09-15-socialconfig-yaml-design.md`
- Decisions #56: `specs/issue-56-socialconfig-yaml/decisions.md` (D1-D2)
