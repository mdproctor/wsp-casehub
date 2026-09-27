# HANDOFF — casehub-examples

## Last Session (2026-09-27 — session 3)

Executed Batches 1-2 of the implementation plan. Rebased slot blocks onto canonical main (absorbed #304 CognitiveAttentionMediator). All 57 targeted tests green.

### What was done

- **Batch 1 (blocks-core cleanup):** Removed PersonalityPromptSection from CognitionCore.promptSections() and deleted the class. Renamed GoalPromptSection → EmergentGoalPromptSection via ide_refactor_rename (93 references updated). Installed to .m2.
- **Rebase:** Slot blocks rebased onto canonical main. Absorbed blocks#304 (CognitiveAttentionMediator). Resolved conflicts in CognitionCore.java (combined #304's isEnabled() pattern with our EmergentGoalPromptSection rename), CLAUDE.md, and consumer-guide.md.
- **Batch 2 (voice card descriptors):** Wrote voice profiles for 4 characters — Penelope (southern-belle), Hooded Claw (dual-persona sneekly/claw), Ant Hill Mob (brooklyn-gangster), Dick Dastardly (dramatic-villain). Briefings thinned to role-only. Adapted CognitionCore constructor calls for #304's new CognitiveAttentionMediator parameter (null). Fixed DirectiveMinimalIntegrationTest assertion for #296 goal rendering format change.
- **Cross-slot:** Filed comment on blocks#304 with integration guidance. Filed soredium#383 (context budget fix).

### Prior sessions

- **Session 1:** Design phase — brainstorming, 8 decisions, design spec, decision review, spec review, implementation plan. eidos-api AgentVoiceProfile + CognitiveSystemPromptRenderer updates committed.
- **Session 2:** Updated plan, filed soredium#383, blocked on IntelliJ heap.

## Immediate Next Step

Execute plan Batch 3 (persona activation + behavioral extraction) then Batch 4 (emergence eval).

**Batch 3 tasks:**
- Task 5: Create PersonaActivationSection — cognitive persona switching for Hooded Claw (sneekly/claw based on nearby agents + never-break-cover constraint). Add persona-constraint mapping to social-config.yaml. Wire into CharacterCognition.renderCognitiveSections().
- Task 6: Extract behavioral template content to SocialConfig — gloating drives, theatrical-outrage norms, suspicion norms. Templates remain for non-cognitive apps.

**Batch 4 tasks:**
- Task 7: Three-run emergence eval — voice-only no cognition vs voice-only full cognition, delta comparison report.

## Cross-Slot Dependencies

| Issue | Slot | Repo | Interaction with #89 | Status |
|-------|------|------|---------------------|--------|
| blocks#304 (CognitiveAttentionMediator) | 203 | blocks | Absorbed — slot blocks rebased onto canonical main with #304. CognitionCore constructor adapted (null mediator). | Done — absorbed via rebase |

## Cross-Module

| Repo | Branch | What | Status |
|------|--------|------|--------|
| canonical eidos | `issue-89-voice-profile` | AgentVoiceProfile + AgentDescriptor.voice field | Needs to move into slot |
| canonical blocks | `issue-89-voice-profile` | Renderer update (duplicate of slot blocks main commit) | Can delete after slot work lands |
| slot blocks | main | Rebased on canonical main — includes #304, #296, #89 renderer + cleanup | Active, green |
| slot neocortex | main | Behind canonical — has rebase conflicts (known issue) | Uses canonical via .m2 |
| slot examples | `issue-89-briefing-voice-card` | Voice cards for 4 characters, constructor adaptation for #304 | Active, 57 tests green |

## References

| Artifact | Path |
|----------|------|
| Design spec | specs/issue-89-briefing-voice-card/2026-09-27-briefing-voice-card-design.md |
| Decisions | specs/issue-89-briefing-voice-card/decisions.md |
| Implementation plan | plans/2026-09-27-briefing-voice-card.md |
| Decision review | ~/reviews/casehub-slots/issue-89-briefing-voice-card-decision-20260927-023048/ |
| Spec review | ~/reviews/casehub-slots/issue-89-briefing-voice-card-spec-20260927-025003/ |
| Voice profile test | wacky-manor/src/test/.../agent/VoiceProfileRendererTest.java |
