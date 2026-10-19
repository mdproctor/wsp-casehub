# Briefing Voice Card Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #89 — Redefine briefing as voice card — cognitive systems carry behavior
**Issue group:** #89

**Goal:** Strip character briefings to voice-only cards, remove observation-layer duplication, add persona switching, and verify cognitive systems drive emergent behavior.

**Architecture:** Two-layer split (system prompt = identity/voice, observation = cognitive state). `CognitiveSystemPromptRenderer` owns the system prompt, `CognitionCore.promptSections()` owns observation sections. `AgentVoiceProfile` record on `AgentDescriptor` replaces prose briefings. Already implemented: eidos-api record, AgentDescriptor field, renderer voice/personality rendering (blocks commit fb508093). Remaining: blocks cleanup, descriptor rewrites, persona activation, emergence eval.

**Tech Stack:** Java 26, Quarkus, eidos-api, blocks-core, neocortex-memory

## Global Constraints

- Java 26, Quarkus 3.x
- Build blocks: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl blocks-core -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`
- Build wacky-manor: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml`
- Use `mvn` not `./mvnw`; never run without `-pl <module>`
- IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and refactoring — never bash grep on `.java` files
- Cross-repo slot: blocks at `/Users/mdproctor/claude/casehub/slots/196/blocks`, examples at `/Users/mdproctor/claude/casehub/slots/196/examples`
- blocks#304 (CognitiveAttentionMediator) in slot 203 touches CognitionCore — see HANDOFF.md §Cross-Slot Dependencies

---

## Batch 1: blocks-core — observation layer cleanup

Safe wrap point: blocks builds clean, PersonalityPromptSection removed, GoalPromptSection renamed. wacky-manor unaffected (consumes via jar).

### Task 1: Remove PersonalityPromptSection and rename GoalPromptSection

**Prerequisite check:** Verify blocks-core builds before changes. If it fails on `CbrRecordStore` mismatch (see .plan deferred note), fix that first — rebuild neocortex to .m2 with the name blocks expects.

**Files:**
- Delete: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/PersonalityPromptSection.java` (use `ide_refactor_safe_delete`)
- Delete: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/PersonalityPromptSectionTest.java` (use `ide_refactor_safe_delete`)
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java:377-413` — remove PersonalityPromptSection from promptSections()
- Rename: `EmergentGoalPromptSection` → `EmergentGoalPromptSection` (use `ide_refactor_rename`)
- Test: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/CognitionCoreTest.java`

**Interfaces:**
- Consumes: `CognitionCore.promptSections()` (modifying), `PersonalityPromptSection` (removing), `EmergentGoalPromptSection` (renaming)
- Produces: `EmergentGoalPromptSection` (renamed, same API), cleaned `promptSections()` without personality duplication

- [ ] **Step 1: Verify blocks-core builds clean**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl blocks-core -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml
```

Expected: BUILD SUCCESS. If CbrRecordStore fails, rebuild neocortex first.

- [ ] **Step 2: Write test asserting PersonalityPromptSection absent**

Add to `CognitionCoreTest.java`:

```java
@Test
void promptSectionsDoesNotContainPersonalitySection() {
    var sections = core.promptSections();
    assertThat(sections).noneMatch(s -> s.getClass().getSimpleName().equals("PersonalityPromptSection"));
}
```

Run test — should FAIL (PersonalityPromptSection still at line 381).

- [ ] **Step 3: Remove PersonalityPromptSection from promptSections()**

In `CognitionCore.java` line 380-382, delete:

```java
if (desc != null && desc.disposition() != null) {
    sections.add(new PersonalityPromptSection(desc.disposition().dispositionProfile()));
}
```

Personality now lives in system prompt via `CognitiveSystemPromptRenderer.renderPersonality()`.

- [ ] **Step 4: Run test — should PASS**

- [ ] **Step 5: Safe-delete PersonalityPromptSection class + test**

Use `ide_refactor_safe_delete` on `PersonalityPromptSection.java`. Then delete `PersonalityPromptSectionTest.java`. Verify no remaining references via `ide_find_references`.

- [ ] **Step 6: Rename GoalPromptSection → EmergentGoalPromptSection**

Use `ide_refactor_rename` — updates all references (CognitionCore line 399, tests, imports).

- [ ] **Step 7: Run full blocks-core test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml
```

Expected: BUILD SUCCESS

- [ ] **Step 8: Install blocks to .m2**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl blocks-core -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml -DskipTests
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add -A
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(casehubio/examples#89): remove PersonalityPromptSection, rename GoalPromptSection

Personality rendered in system prompt by CognitiveSystemPromptRenderer.
Raw DispositionValue codes in observation are redundant.
GoalPromptSection → EmergentGoalPromptSection to distinguish
emergent (drive-formed) goals from authored (descriptor) goals.

Refs casehubio/examples#89"
```

---

## Batch 2: wacky-manor — voice card descriptors

Safe wrap point: 4 characters have structured voice profiles, briefings thinned to role context, shared voice conventions defined. Build passes.

### Task 2: Shared Hanna-Barbera voice anchor + Penelope Pitstop voice card

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml`
- Test: new assertions in existing test class or new `VoiceProfileLoadTest.java`

**Interfaces:**
- Consumes: `AgentVoiceProfile` record (eidos-api jar), `CognitiveSystemPromptRenderer` voice rendering (blocks jar)
- Produces: Penelope descriptor with structured `voice:` field; shared `hanna-barbera-style` YAML anchor

- [ ] **Step 1: Write test asserting Penelope has voice profile**

```java
@Test
void penelopeHasVoiceProfile() {
    var desc = registry.findById("penelope-pitstop", TENANCY_ID).orElseThrow();
    assertThat(desc.voice()).isNotNull();
    assertThat(desc.voice().register()).isEqualTo("southern-belle");
    assertThat(desc.voice().catchphrases()).contains("Why, how delightful!");
}
```

Run — should FAIL.

- [ ] **Step 2: Add shared voice anchor + rewrite Penelope**

Add YAML anchor at the top of `descriptors-composite.yaml`. Rewrite Penelope's descriptor: thin `briefing:` to role-only line, add `voice:` with register, accent, catchphrases, speech-patterns (including shared conventions), vocabulary, quirks.

Key fields for Penelope:
- register: `southern-belle`
- accent: `southern-drawl`
- catchphrases: `"Why, how delightful!"`, `"Bless your heart!"`, `"Hayulp! Hayulp!"`
- speech-patterns: shared HB conventions + `warm and effusive`, `phrases observations as discoveries`
- vocabulary-uses: `"simply"`, `"dreadful"`, `"delightful"`
- quirks: `exclaims when discovering something new`

Note: test whether YAML `<<:` merge key works with the eidos deserializer. If not, inline shared patterns directly.

- [ ] **Step 3: Run test — should PASS**

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): Penelope Pitstop voice card + shared Hanna-Barbera anchor

Refs #89"
```

### Task 3: Hooded Claw dual-persona voice card

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml`
- Test: new persona-specific assertions

**Interfaces:**
- Consumes: `AgentVoiceProfile.personas()` map, renderer persona rendering
- Produces: Hooded Claw descriptor with `personas:` map (`sneekly` + `claw`)

- [ ] **Step 1: Write test**

```java
@Test
void hoodedClawHasDualPersonaVoice() {
    var desc = registry.findById("hooded-claw", TENANCY_ID).orElseThrow();
    assertThat(desc.voice()).isNotNull();
    assertThat(desc.voice().personas()).containsKeys("sneekly", "claw");
    assertThat(desc.voice().personas().get("sneekly").register()).isEqualTo("obsequious");
    assertThat(desc.voice().personas().get("claw").register()).isEqualTo("grandiose");
}
```

Run — should FAIL.

- [ ] **Step 2: Rewrite Hooded Claw descriptor**

Thin `briefing:` to role-only. Add `voice:` with `personas:` map containing `sneekly` (obsequious, unctuous-formal) and `claw` (grandiose, theatrical-villain). Base voice `quirks:` shared across both personas.

- [ ] **Step 3: Run test — should PASS**

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): Hooded Claw dual-persona voice card (sneekly + claw)

Refs #89"
```

### Task 4: Ant Hill Mob + Dick Dastardly voice cards

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml`
- Test: voice profile assertions for both characters

**Interfaces:**
- Consumes: `AgentVoiceProfile` record, shared anchor
- Produces: Two more characters with structured voice profiles

- [ ] **Step 1: Write tests**

```java
@Test
void antHillMobHasVoiceProfile() {
    var desc = registry.findById("ant-hill-mob", TENANCY_ID).orElseThrow();
    assertThat(desc.voice()).isNotNull();
    assertThat(desc.voice().register()).isEqualTo("brooklyn-gangster");
}

@Test
void dickDastardlyHasVoiceProfile() {
    var desc = registry.findById("dick-dastardly", TENANCY_ID).orElseThrow();
    assertThat(desc.voice()).isNotNull();
    assertThat(desc.voice().register()).isEqualTo("dramatic-villain");
}
```

Run — should FAIL.

- [ ] **Step 2: Rewrite Ant Hill Mob descriptor**

Thin `briefing:` to role-only. Add `voice:` with register `brooklyn-gangster`, accent `gruff-brooklyn`, catchphrases (`"Hey boss, dat don't look right!"`, etc.), quirks (boys chiming in).

- [ ] **Step 3: Rewrite Dick Dastardly descriptor**

Thin `briefing:` to role-only. Add `voice:` with register `dramatic-villain`, accent `sneering-theatrical`, catchphrases (`"Drat, drat, and double DRAT!"`, etc.).

- [ ] **Step 4: Run full wacky-manor tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml
```

Expected: BUILD SUCCESS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): Ant Hill Mob + Dick Dastardly voice cards

Refs #89"
```

---

## Batch 3: wacky-manor — persona activation + behavioral extraction

Safe wrap point: multi-persona characters get active persona signal in observations, behavioral template content moved to SocialConfig. Build passes.

### Task 5: PersonaActivationSection for multi-persona characters

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/PersonaActivationSection.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:100-167`
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` — add persona-constraint mapping
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/PersonaActivationSectionTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor.voice().personas()`, `AgentDescriptor.constraints()`, nearby agent IDs
- Produces: `ObservationSection` with "Active Voice: **Sneekly**" for multi-persona characters; null for single-persona

- [ ] **Step 1: Write failing tests**

Test three cases: constraint satisfied (nearby agents → sneekly), alone (→ claw), single-persona character (→ null).

```java
@Test
void emitsActivePersonaWhenConstraintSatisfied() {
    var mapping = new PersonaActivationSection.PersonaConstraintMapping(
        "never-break-cover", "sneekly", "claw");
    var section = new PersonaActivationSection(mapping);
    var result = section.resolve(Set.of("penelope-pitstop"));
    assertThat(result).contains("Sneekly");
}

@Test
void emitsAlternatePersonaWhenAlone() {
    var mapping = new PersonaActivationSection.PersonaConstraintMapping(
        "never-break-cover", "sneekly", "claw");
    var section = new PersonaActivationSection(mapping);
    var result = section.resolve(Set.of());
    assertThat(result).contains("Claw");
}

@Test
void returnsNullForSinglePersona() {
    var section = new PersonaActivationSection(null);
    assertThat(section.resolve(Set.of("someone"))).isNull();
}
```

- [ ] **Step 2: Implement PersonaActivationSection**

Inner record `PersonaConstraintMapping(String constraintName, String whenActive, String whenInactive)`. Constructor takes nullable mapping. `resolve(Set<String> nearbyAgentIds)` returns null if no mapping; otherwise returns `"## Active Voice\nYou are currently presenting as **{persona}**. Maintain this voice."` — `whenActive` if nearby agents exist, `whenInactive` if alone.

- [ ] **Step 3: Run tests — should PASS**

- [ ] **Step 4: Add persona-constraint mapping to social-config.yaml**

Under `hooded-claw:`:

```yaml
  persona-constraint:
    constraint: never-break-cover
    when-active: sneekly
    when-inactive: claw
```

- [ ] **Step 5: Wire into CharacterCognition.renderCognitiveSections()**

Read `persona-constraint` from `SocialConfig`. Construct `PersonaActivationSection` with mapping. Call `resolve(nearbyAgentIds)`. If non-null, add as `ObservationSection` at the end of the cognitive sections list.

- [ ] **Step 6: Run full test suite**

Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): PersonaActivationSection — cognitive persona switching

Hooded Claw gets active voice signal (Sneekly/Claw) based on nearby
agents and never-break-cover constraint.

Refs #89"
```

### Task 6: Extract behavioral template content to SocialConfig

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- Test: verify build + existing tests still pass (behavioral content is additive)

**Interfaces:**
- Consumes: Template behavioral analysis (cartoon-villain, cartoon-protector)
- Produces: SocialConfig drives and norms that replace template behavioral instructions

- [ ] **Step 1: Add extracted behavioral entries to social-config.yaml**

For `hooded-claw:` add gloating drive and theatrical-outrage norm.
For `dick-dastardly:` add gloating drive, theatrical-outrage norm (create section).
For `ant-hill-mob:` add suspicion norm (create section).

See spec §2.7 for the full extraction table.

- [ ] **Step 2: Run full test suite**

Expected: BUILD SUCCESS (behavioral content is additive — existing tests unaffected)

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): extract behavioral template content to SocialConfig

Gloating drives, theatrical-outrage norms, and suspicion norms
now in SocialConfig. Templates remain for non-cognitive apps.

Refs #89"
```

---

## Batch 4: wacky-manor — emergence verification

Safe wrap point: three-run eval validates cognitive system contribution to behavior.

### Task 7: Three-run emergence eval design

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveEvalTest.java`
- Create: `wacky-manor/docs/eval/` directory for durable output
- Test: self-verifying — eval test IS the verification

**Interfaces:**
- Consumes: `CognitionConfig.none()`, `CognitionConfig.all()`, existing eval infrastructure (ConfigLevel, progressiveEvalWithDeltaCapture pattern), voice-card descriptors from Batch 2
- Produces: Run A (voice-only, no cognition), Run B (voice-only, full cognition), delta comparison report in `docs/eval/`

- [ ] **Step 1: Add emergence eval method**

New `@Test @Tag("llm-eval")` method `emergenceVerification()`. Runs the eval scenario twice with different `CognitionConfig` settings (none vs all). Both use voice-card descriptors. Captures delta streams. Generates markdown comparison report. Structural assertion: run B produces strictly more cognitive sections than run A.

- [ ] **Step 2: Implement comparison report generator**

Generates markdown showing per-character, per-tick: sections in B that were absent in A (mood, drives, narrative, strategy, etc.). This is the emergence signal.

- [ ] **Step 3: Run emergence eval (requires API key, non-deterministic)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml -Dtest=CognitiveEvalTest#emergenceVerification
```

Review output qualitatively. Structural assertion (more sections) should pass deterministically.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/test/ wacky-manor/docs/eval/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): three-run emergence verification eval

Run A (voice-only no cognition) vs Run B (voice-only full cognition).
Delta report isolates cognitive system behavioral contributions.

Refs #89"
```

---

## References

- [2026-09-27-briefing-voice-card-design.md] — design spec
- [decisions.md D1-D8] — design decisions with revision history
- [CognitiveSystemPromptRenderer.java:15-154] — renderer (already updated, blocks commit fb508093)
- [CognitionCore.java:377-413] — promptSections() with PersonalityPromptSection at line 381
- [PersonalityPromptSection.java:11-34] — raw codes section to remove
- [GoalPromptSection.java:18-134] — rename to EmergentGoalPromptSection
- [CharacterCognition.java:100-167] — renderCognitiveSections() method
- [descriptors-composite.yaml] — current character briefings (prose, no voice profiles yet)
- [social-config.yaml] — behavioral seed data
- [templates.yaml] — templates with mixed voice/behavioral content
- [CognitiveEvalTest.java:26-162] — eval infrastructure
- [Phase D spec §4] — directive-minimal seeding design
- [blocks#304] — CognitiveAttentionMediator (slot 203, touches CognitionCore)
- [GitHub #89] — focal issue
