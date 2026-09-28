# HANDOFF — casehub-examples

## Last Session (2026-09-28 — session 4)

Executed Batch 3 (Tasks 6-8), evolved the voice profile design to layered authoring, fixed Quarkus test deadlock, and got the full suite green (577 tests, 0 failures, 0 errors).

### What was done

- **Task 6 (PersonaActivationSection):** Cognitive persona switching for Hooded Claw. `PersonaConstraintMapping` on `SocialConfig`, parsed from `social-config.yaml`. `PersonaActivationSection.resolve()` emits "Active Voice: **Sneekly/Claw**" based on nearby agents. Wired into `CharacterCognition.renderCognitiveSections()`. 8 new tests.
- **Task 7 (Behavioral extraction):** Gloating drives for Hooded Claw and Dick Dastardly. Theatrical-outrage and elaborate-schemes norms from `cartoon-villain` template into `SocialConfig`. Templates remain for non-cognitive apps.
- **Task 8 (Emergence eval):** `EmergenceVerificationTest` — standalone (no `@QuarkusTest`) comparison of `CognitionConfig.none()` vs `CognitionConfig.all()`. Run A produces 0 observation sections, Run B produces +1 (MoodPromptSection). Results persisted to `docs/eval/`.
- **Voice description (D2 revision):** Added `description` field to `AgentVoiceProfile` across eidos (eidos#183), blocks (blocks#309), examples (examples#92). Layered voice authoring: description is Layer 1 (LLM training data), structured fields are optional Layer 2 fine-tuning. Slimmed 4 character descriptors from 83 to 19 lines of voice YAML. Renderer updated to render description first, structured fields as supplements.
- **Quarkus deadlock fix (examples#93):** Root cause: Aether parallel metadata resolver + JDK 26 class-init deadlock in `SSLConnectionSocketFactory.<clinit>` vs `FacadeClassLoader`. Fix: `-Daether.metadataResolver.threads=1` in surefire argLine. Also added `TestCdiBeans` producing `SystemPromptRenderer` and `VocabularyRegistry` for `@QuarkusTest` augmentation.
- **Pre-existing fixes:** `DescriptorForEachAdapter.getCondition()` renamed from `getWhen` (eidos-core). `DescriptorLoadTest` updated for cognitive renderer. Slot settings `updatePolicy` changed from `always` to `interval:60`.
- **Issues filed:** eidos#183, blocks#309, examples#92, examples#93.

### Design evolution

D2 revised from "pure structured, no freeform" to "layered voice authoring." Key insight: for well-known characters, the LLM's training data IS the voice definition — enumerated catchphrases/vocabulary are redundant. For original characters (clinical, AML), the enumerated fields ADD information the LLM doesn't have. The `description` field is the merge point.

Discussion identified that the cognitive system must not recreate a static brief by different means. The emergence eval proves structural emergence (sections exist) but not behavioral emergence (LLM responds differently). Event-driven scenarios needed to prove the cognitive system produces contextually adaptive behavior.

### Prior sessions

- **Session 1:** Design phase — brainstorming, 8 decisions, design spec, decision review, spec review, implementation plan. eidos-api AgentVoiceProfile + CognitiveSystemPromptRenderer updates committed.
- **Session 2:** Updated plan, filed soredium#383, blocked on IntelliJ heap.
- **Session 3:** Batches 1-2 — PersonalityPromptSection removed, GoalPromptSection renamed in blocks-core. Voice card descriptors for 4 characters. Slot blocks rebased onto canonical main (absorbed #304 CognitiveAttentionMediator).

## Immediate Next Step

Design an event-driven emergence eval that proves the cognitive system produces contextually adaptive behavior — not just observation sections, but measurably different LLM responses when cognitive state changes over time.

**Brainstorm first.** Key questions:
1. What events trigger cognitive state changes? (scheme failure → drive drops, suspicion → belief revision, persona switch)
2. How to measure behavioral difference? (response comparison at tick 1 vs tick 50)
3. What's the null hypothesis? (character with same voice + no cognitive state produces same behavior every tick)

**Then:**
- Agent-config CDI integration — `LlmCredentialStore` ambiguity needs resolving before `agent-config` can be added as a compile dependency. Currently only works in dev mode.
- Remaining 13 character voice profiles — mechanical follow-up (separate issue).

## Cross-Slot Dependencies

| Issue | Slot | Repo | Interaction with #89 | Status |
|-------|------|------|---------------------|--------|
| blocks#304 (CognitiveAttentionMediator) | 203 | blocks | Absorbed — slot blocks rebased onto canonical main with #304. CognitionCore constructor adapted (null mediator). | Done — absorbed via rebase |
| blocks#309 (voice description renderer) | 196 | blocks | Renders description as primary voice signal | Done — committed |
| eidos#183 (voice description field) | canonical | eidos | description field on AgentVoiceProfile, persona inheritance, deserializer | Done — committed on issue-89-voice-profile branch |

## Cross-Module

| Repo | Branch | What | Status |
|------|--------|------|--------|
| canonical eidos | `issue-89-voice-profile` | AgentVoiceProfile + description field + AgentDescriptor.voice field + DescriptorForEachAdapter fix | Active, 2 commits ahead |
| slot blocks | main | Renderer update for voice description | Active, green |
| slot examples | `issue-89-briefing-voice-card` | Voice cards, persona activation, behavioral extraction, emergence eval, deadlock fix | Active, 577 tests green |

## References

| Artifact | Path |
|----------|------|
| Design spec | specs/issue-89-briefing-voice-card/2026-09-27-briefing-voice-card-design.md |
| Decisions (D2 revised) | specs/issue-89-briefing-voice-card/decisions.md |
| Implementation plan | plans/2026-09-27-briefing-voice-card.md |
| Emergence eval results | docs/eval/emergence-20260927-232108/ |
| Voice profile test | wacky-manor/src/test/.../agent/VoiceProfileRendererTest.java |
| PersonaActivation test | wacky-manor/src/test/.../agent/PersonaActivationSectionTest.java |
| Emergence test | wacky-manor/src/test/.../experiment/EmergenceVerificationTest.java |
| TestCdiBeans | wacky-manor/src/test/.../agent/TestCdiBeans.java |
| Agent config manifest | wacky-manor/agent-config.yaml |
