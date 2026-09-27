# HANDOFF — casehub-examples

## Last Session (2026-09-27 — session 2)

Updated implementation plan to account for prior session work (Batches 1-2 done). Filed soredium#383 (context budget guidance fix — total_tokens unreliable after compression). Commented on blocks#304 with cross-slot coordination guidance. Execution blocked by IntelliJ MCP — JVM heap critically low (8% free).

### Prior session (session 1)

Designed and began implementing #89. Full brainstorming → 8 decisions (6 revised after review) → design spec → implementation plan → Batches 1-2 executed. AgentVoiceProfile record created in eidos-api, CognitiveSystemPromptRenderer updated in blocks-core to render voice profiles and personality. The .m2 was nuked to fix cross-repo API mismatches and rebuilt from canonical sources. All 4 VoiceProfileRendererTest pass in wacky-manor.

## Immediate Next Step

**Prerequisite:** Ensure IntelliJ MCP is available (restart IntelliJ if needed, then `/mcp` to reconnect).

**Then:** Execute the updated plan at `plans/2026-09-27-briefing-voice-card.md` starting from Batch 1 Task 1 (blocks-core cleanup — remove PersonalityPromptSection, rename GoalPromptSection). The prior session's .plan marks Batches 1-2 as done but the PersonalityPromptSection removal was deferred due to a CbrRecordStore mismatch — verify that's resolved before proceeding.

**Before starting Batch 2 (descriptors):** Add eidos to the slot (voice profile changes on canonical eidos `issue-89-voice-profile` branch, should be at `slots/196/eidos`).

## Cross-Slot Dependencies

| Issue | Slot | Repo | Interaction with #89 | Status |
|-------|------|------|---------------------|--------|
| blocks#304 (CognitiveAttentionMediator) | 203 | blocks | Adds attention prompt section to `CognitionCore.promptSections()`. #89 redesigns observation layer — must absorb #304's section. No conflict if #304 uses standard `promptSections()` pattern. | In progress — commented on #304 with guidance |

## Cross-Module

| Repo | Branch | What | Status |
|------|--------|------|--------|
| canonical eidos | `issue-89-voice-profile` | AgentVoiceProfile + AgentDescriptor.voice field | Needs to move into slot |
| canonical blocks | `issue-89-voice-profile` | Renderer update (duplicate of slot blocks main commit) | Can delete after slot work lands |
| slot blocks | main | CognitiveSystemPromptRenderer with voice rendering | Active |
| slot neocortex | main | Behind canonical — has rebase conflicts (known issue) | Uses canonical via .m2 |
| slot 203 blocks | main | #304 CognitiveAttentionMediator landed | Absorb into #89 pipeline when ready |

## References

| Artifact | Path |
|----------|------|
| Design spec | specs/issue-89-briefing-voice-card/2026-09-27-briefing-voice-card-design.md |
| Decisions | specs/issue-89-briefing-voice-card/decisions.md |
| Implementation plan | plans/2026-09-27-briefing-voice-card.md |
| Decision review | ~/reviews/casehub-slots/issue-89-briefing-voice-card-decision-20260927-023048/ |
| Spec review | ~/reviews/casehub-slots/issue-89-briefing-voice-card-spec-20260927-025003/ |
| Voice profile test | wacky-manor/src/test/.../agent/VoiceProfileRendererTest.java |
