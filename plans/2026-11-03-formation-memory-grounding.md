# Formation Memory Grounding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#735 — DriveDescriptionGenerationPhase — formation memory grounding in dialogue-form descriptions
**Issue group:** casehubio/neocortex#735

**Goal:** Restructure the P22 system prompt and user prompt in `DriveDescriptionGenerationPhase` so formation memories surface as body sensations rather than narrative citations, achieving formation echo >= 3 in dialogue-form scenes.

**Architecture:** Single-file change in `DriveDescriptionGenerationPhase.java`. The system prompt's emphasis hierarchy is inverted: formation memory body echoes become the organizing principle, body territory stays required but demoted, contrast becomes brief texture. The user prompt gains per-memory somatic rendering instructions.

**Tech Stack:** Java 26, Quarkus, neocortex mindmap-intelligence module

## Global Constraints

- Changes confined to `DriveDescriptionGenerationPhase.java` only
- No downstream consumer changes (`CognitionObservationBuilder`, `CharacterAgentLoop`)
- Keep 3-5 sentence budget for generated descriptions
- When no formation memories exist for a drive, existing fallback behaviour must be preserved
- All existing tests must continue to pass (prompt assertion tests will need updating to match new prompt text)

---

## Batch 1: Restructure P22 prompts and update tests

### Task 1: Update system prompt assertion test, then restructure SYSTEM_PROMPT

**Files:**
- Modify: `mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhaseTest.java`
- Modify: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhase.java`

**Interfaces:**
- Consumes: `capturedInits` list from test harness (system prompt captured via `AgentSessionInit`)
- Produces: Updated `SYSTEM_PROMPT` constant used by `callLlm()` and `ParallelLlmExecutor`

- [ ] **Step 1: Write the failing test — system prompt requires formation memory as body echo**

Add a new test method `systemPromptRequiresFormationMemoryAsBodyEcho` that asserts the system prompt contains formation-memory-first instructions and the anti-pattern guidance:

```java
@Test
void systemPromptRequiresFormationMemoryAsBodyEcho() {
    seedDriveNode("gallantry", 0.9);
    seedFormationMemory("Your father knelt to help the stranger", "gallantry");

    phase.run(TENANT, List.of());

    var systemPrompt = capturedInits.stream()
                                    .filter(i -> !i.systemPrompt().contains("discrimination"))
                                    .findFirst().orElseThrow()
                                    .systemPrompt();
    assertThat(systemPrompt).as("must require formation memory as body sensation")
                            .contains("body learned");
    assertThat(systemPrompt).as("must include anti-pattern guidance")
                            .contains("NOT");
    assertThat(systemPrompt).as("contrast must not be called single most important")
                            .doesNotContain("single most important");
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest#systemPromptRequiresFormationMemoryAsBodyEcho" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: FAIL — current system prompt doesn't contain "body learned" and does contain "single most important"

- [ ] **Step 3: Restructure the SYSTEM_PROMPT constant**

Use `ide_replace_member` to replace the `SYSTEM_PROMPT` field in `DriveDescriptionGenerationPhase` with:

```java
// Barrett (2017) constructionist 4-channel framework: somatic (Damasio 1994), attentional
// (Easterbrook 1959), temporal (Barrett prediction), action tendency (Frijda 1986)
private static final String SYSTEM_PROMPT = """
                                            You derive character-specific drive profiles from personality and formation memories.
                                            For each drive, describe how it TYPICALLY manifests in THIS person's body — shaped \
                                            by their unique history, personality scores, and attachment style.

                                            FORMATION MEMORY: When formation memories are listed, START with what the body \
                                            LEARNED from them — not what happened, but the somatic imprint left behind. \
                                            Each description must include at least one formation memory rendered as a body \
                                            sensation. NOT "like when you were ten" — instead describe what the body does \
                                            NOW because of that memory ("your chest opens the way it first opened at ten, \
                                            crouched over buried treasure").

                                            Personality scores are body calibration: high Callousness means the sympathy region \
                                            is quiet; high Suspiciousness means shoulders square before worry registers. \
                                            Attachment is the somatic ground state.

                                            BODY TERRITORY: Each pull has an assigned body location. Anchor the description \
                                            in that territory — the formation memory echo lives HERE, in this part of the body. \
                                            Never place a pull in another pull's territory. Use discovery verbs (arrives, \
                                            settles, pools, fizzes, shifts, tightens, drops, spreads) not analytical verbs \
                                            (calculates, assesses, determines, considers, evaluates).

                                            Weave the two primary channels as the backbone. The four channels:
                                            - Somatic: where exactly, what sensation (tension, warmth, numbness, weight)
                                            - Attentional: what seized focus, what stopped being noticed
                                            - Temporal: compressed, stretched, frozen, urgent
                                            - Action tendency: what the body prepares to do before deciding
                                            Add the other two channels as texture, not sections.

                                            CONTRAST: When other pulls' body locations are listed, note the contrast \
                                            briefly: "Not the [other-location] — this is [body-territory]."

                                            Write in second person. Do not name the drive. 3-5 sentences.
                                            Respond with JSON only: {"description": "..."}""";
```

Key changes from current:
- `FORMATION MEMORY` block added at top with mandatory requirement and anti-pattern guidance
- `BODY TERRITORY` keeps location anchoring but loses "START the description with" — formation memory now leads
- Discovery verbs moved into BODY TERRITORY section (was standalone)
- `CONTRAST` demoted from bold emphasis with "single most important differentiator" to brief inline instruction
- "Ground in specific formation memories where possible" removed (replaced by FORMATION MEMORY block)

- [ ] **Step 4: Run the new test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest#systemPromptRequiresFormationMemoryAsBodyEcho" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: PASS

- [ ] **Step 5: Run all existing tests to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: all tests PASS. The test `descriptionPromptRequestsMultiChannelExperience` should still pass since the four channels are unchanged. If any prompt-assertion test fails due to removed text (e.g. checking for exact wording that changed), update that test to match the new prompt text.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhase.java mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhaseTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#735): restructure SYSTEM_PROMPT — formation memory body echo as organizing principle"
```

### Task 2: Update user prompt to include somatic rendering instructions per memory

**Files:**
- Modify: `mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhaseTest.java`
- Modify: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhase.java`

**Interfaces:**
- Consumes: `capturedUserPrompts` list from test harness (user prompt captured via `AgentSession.query()`)
- Produces: Updated user prompt structure in `callLlm()` method

- [ ] **Step 1: Write the failing test — user prompt includes somatic rendering instruction**

Add a new test method `promptIncludesSomaticRenderingInstructionPerMemory`:

```java
@Test
void promptIncludesSomaticRenderingInstructionPerMemory() {
    seedDriveNode("gallantry", 0.9);
    seedFormationMemory("Your father knelt to help the stranger", "gallantry");

    phase.run(TENANT, List.of());

    assertThat(capturedUserPrompts).hasSize(1);
    var userPrompt = capturedUserPrompts.getFirst();
    assertThat(userPrompt).as("must instruct somatic rendering")
                          .contains("body LEARNED");
    assertThat(userPrompt).as("must include per-memory rendering cue")
                          .contains("what does your body do NOW");
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest#promptIncludesSomaticRenderingInstructionPerMemory" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: FAIL — current user prompt has "Formation memories that shaped this pull:" without somatic rendering instruction

- [ ] **Step 3: Update the user prompt in callLlm()**

Use `ide_replace_member` to update the `callLlm` method. Change lines 398-403 (the formation memories section) from:

```java
if (!servingMemories.isEmpty()) {
    promptBuilder.append("Formation memories that shaped this pull:\n");
    for (int i = 0; i < servingMemories.size(); i++) {
        promptBuilder.append(i + 1).append(". ").append(servingMemories.get(i).name()).append("\n");
    }
    promptBuilder.append("\n");
}
```

To:

```java
if (!servingMemories.isEmpty()) {
    promptBuilder.append("Formation memories — describe what the body LEARNED from each, not what happened:\n");
    for (int i = 0; i < servingMemories.size(); i++) {
        promptBuilder.append(i + 1).append(". ").append(servingMemories.get(i).name())
                     .append("\n   → What does your body do NOW when this pull arrives, because of this memory?\n");
    }
    promptBuilder.append("\n");
}
```

- [ ] **Step 4: Run the new test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest#promptIncludesSomaticRenderingInstructionPerMemory" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: PASS

- [ ] **Step 5: Run all DriveDescriptionGenerationPhaseTest tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: all PASS. The existing test `promptIncludesFormationMemoriesAndDriveContext` checks for "gallantry", "0.9", and "knelt to help the stranger" — all still present in the new prompt, so it should pass without changes.

- [ ] **Step 6: Run full fast test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`

Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhase.java mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhaseTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#735): add somatic rendering instructions to user prompt per formation memory"
```

## References

- [2026-11-03-formation-memory-grounding-design.md] — design spec this plan implements
- [DriveDescriptionGenerationPhase.java:55-85] — current SYSTEM_PROMPT being restructured
- [DriveDescriptionGenerationPhase.java:398-403] — current user prompt formation memory section
- [DriveDescriptionGenerationPhaseTest.java:340-359] — existing prompt assertion test pattern
- [CognitionObservationBuilder.java:626-644] — downstream consumer (no changes needed)
- [casehubio/neocortex#735] — focal issue with acceptance criteria
