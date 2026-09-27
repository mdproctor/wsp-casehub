# Briefing Voice Card Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/examples#89 — Redefine briefing as voice card
**Issue group:** #89

**Goal:** Separate voice (how agents sound) from behavior (what agents do) with a structured `AgentVoiceProfile` record, clean the system prompt / observation layer boundary, and verify cognitive systems produce emergent behavior without behavioral scripting.

**Architecture:** Three-concern separation — identity and voice in the system prompt (rendered by `CognitiveSystemPromptRenderer`), dynamic cognitive state in the observation layer (rendered by `CognitionCore.promptSections()` + `CharacterCognition`). New `AgentVoiceProfile` record on `AgentDescriptor` with vocabulary-resolved register/accent. Personas map for multi-voice characters with cognitive switching.

**Tech Stack:** Java 26, Quarkus, Maven, eidos-api, blocks-core, wacky-manor

## Global Constraints

- Pre-release platform — breaking API changes are acceptable
- eidos-api changes must be built and installed to `~/.m2/repository` before blocks-core can consume them
- blocks-core changes must be built and installed before wacky-manor can consume them
- `mvn` not `./mvnw`, always with `-s slot-settings.xml` in the examples repo
- `JAVA_HOME=$(/usr/libexec/java_home -v 26)` prefix for all Maven commands
- eidos repo: `/Users/mdproctor/claude/casehub/eidos` (NOT in this slot)
- blocks repo: `/Users/mdproctor/claude/casehub/slots/196/blocks`
- examples repo: `/Users/mdproctor/claude/casehub/slots/196/examples`
- Use IntelliJ MCP (`ide_*`) for all code navigation and editing — never bash grep/sed on source files
- Use `ide_insert_member` for new methods/fields, `ide_replace_member` for body rewrites

---

## Batch 1: eidos-api — Voice Profile Record

**Repo:** `/Users/mdproctor/claude/casehub/eidos`
**After this batch:** `AgentVoiceProfile` exists as a first-class type in eidos-api. `AgentDescriptor` has a `voice` field. Existing consumers are unaffected (voice defaults to null).

### Task 1: Create AgentVoiceProfile record

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/AgentVoiceProfile.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentVoiceProfileTest.java`

**Interfaces:**
- Consumes: nothing (new type)
- Produces: `AgentVoiceProfile(String register, String accent, List<String> catchphrases, List<String> speechPatterns, List<String> vocabularyUses, List<String> vocabularyAvoids, List<String> quirks, Map<String, AgentVoiceProfile> personas)` — consumed by Task 2 (AgentDescriptor), Task 3 (renderer)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class AgentVoiceProfileTest {

    @Test void basicVoiceProfile() {
        var voice = new AgentVoiceProfile(
            "southern-belle", "southern-drawl",
            List.of("Why, how delightful!", "Bless your heart!"),
            List.of("warm and effusive"),
            List.of("simply", "delightful"),
            List.of(),
            List.of("exclaims when discovering something new"),
            null);
        assertThat(voice.register()).isEqualTo("southern-belle");
        assertThat(voice.accent()).isEqualTo("southern-drawl");
        assertThat(voice.catchphrases()).hasSize(2);
        assertThat(voice.personas()).isNull();
    }

    @Test void personasWithBaseVoice() {
        var sneekly = new AgentVoiceProfile(
            "obsequious", null,
            List.of("Oh, my DEAR Miss Pitstop!"), List.of("overly helpful"),
            null, null, null, null);
        var claw = new AgentVoiceProfile(
            "grandiose", null,
            List.of("Nyah-ha-ha-HA!"), List.of("dramatic monologues"),
            null, null, null, null);
        var voice = new AgentVoiceProfile(
            null, null, null, null, null, null,
            List.of("explains schemes even when alone"),
            Map.of("sneekly", sneekly, "claw", claw));
        assertThat(voice.personas()).hasSize(2);
        assertThat(voice.personas().get("sneekly").register()).isEqualTo("obsequious");
        assertThat(voice.quirks()).containsExactly("explains schemes even when alone");
    }

    @Test void resolvePersonaInheritsFromBase() {
        var base = new AgentVoiceProfile(
            "formal", "received-pronunciation",
            null, List.of("measured and precise"),
            List.of("indeed"), null,
            List.of("pauses before speaking"),
            Map.of("casual", new AgentVoiceProfile(
                "informal", null, null, null, null, null, null, null)));
        var resolved = base.resolvePersona("casual");
        assertThat(resolved.register()).isEqualTo("informal");
        assertThat(resolved.accent()).isEqualTo("received-pronunciation");
        assertThat(resolved.speechPatterns()).containsExactly("measured and precise");
        assertThat(resolved.vocabularyUses()).containsExactly("indeed");
        assertThat(resolved.quirks()).containsExactly("pauses before speaking");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=AgentVoiceProfileTest`
Expected: FAIL — `AgentVoiceProfile` class not found

- [ ] **Step 3: Create AgentVoiceProfile record**

```java
package io.casehub.eidos.api;

import java.util.List;
import java.util.Map;
import org.jspecify.annotations.Nullable;

public record AgentVoiceProfile(
    @Nullable String register,
    @Nullable String accent,
    @Nullable List<String> catchphrases,
    @Nullable List<String> speechPatterns,
    @Nullable List<String> vocabularyUses,
    @Nullable List<String> vocabularyAvoids,
    @Nullable List<String> quirks,
    @Nullable Map<String, AgentVoiceProfile> personas
) {
    public AgentVoiceProfile {
        catchphrases    = catchphrases != null ? List.copyOf(catchphrases) : null;
        speechPatterns  = speechPatterns != null ? List.copyOf(speechPatterns) : null;
        vocabularyUses  = vocabularyUses != null ? List.copyOf(vocabularyUses) : null;
        vocabularyAvoids = vocabularyAvoids != null ? List.copyOf(vocabularyAvoids) : null;
        quirks          = quirks != null ? List.copyOf(quirks) : null;
        personas        = personas != null ? Map.copyOf(personas) : null;
    }

    public AgentVoiceProfile resolvePersona(String personaName) {
        if (personas == null || !personas.containsKey(personaName)) {
            return this;
        }
        var persona = personas.get(personaName);
        return new AgentVoiceProfile(
            persona.register() != null ? persona.register() : this.register,
            persona.accent() != null ? persona.accent() : this.accent,
            persona.catchphrases() != null ? persona.catchphrases() : this.catchphrases,
            persona.speechPatterns() != null ? persona.speechPatterns() : this.speechPatterns,
            persona.vocabularyUses() != null ? persona.vocabularyUses() : this.vocabularyUses,
            persona.vocabularyAvoids() != null ? persona.vocabularyAvoids() : this.vocabularyAvoids,
            persona.quirks() != null ? persona.quirks() : this.quirks,
            null
        );
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=AgentVoiceProfileTest`
Expected: PASS (3/3)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentVoiceProfile.java api/src/test/java/io/casehub/eidos/api/AgentVoiceProfileTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#89): add AgentVoiceProfile record with persona inheritance Refs casehubio/examples#89"
```

### Task 2: Add voice field to AgentDescriptor

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` — add `voice` field
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` (Builder) — add `voice` builder method
- Modify: `core/src/main/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializer.java` — deserialize `voice` from YAML
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java` — existing tests + new voice test
- Test: `core/src/test/java/io/casehub/eidos/core/yaml/AgentDescriptorDeserializerTest.java` — YAML voice parsing

**Interfaces:**
- Consumes: `AgentVoiceProfile` from Task 1
- Produces: `AgentDescriptor.voice()` returning `@Nullable AgentVoiceProfile` — consumed by Task 3 (renderer)

- [ ] **Step 1: Write failing test for AgentDescriptor.voice()**

Add test to existing `AgentDescriptorTest.java` (or create if absent) verifying `AgentDescriptor.builder().voice(voiceProfile).build().voice()` returns the profile.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=AgentDescriptorTest`
Expected: FAIL — no `voice` field on AgentDescriptor

- [ ] **Step 3: Add voice field to AgentDescriptor record**

Use `ide_edit_member` to add `@Nullable AgentVoiceProfile voice` field after `briefing` (field 22) in the record component list. Update the Builder to include `voice(AgentVoiceProfile)`. The canonical constructor already handles null fields — voice is nullable.

- [ ] **Step 4: Update AgentDescriptorDeserializer to parse voice from YAML**

Use `ide_find_class` to locate `AgentDescriptorDeserializer` in eidos-core. Add voice parsing: when `"voice"` key is present in the YAML node, deserialize it into an `AgentVoiceProfile`. Handle nested `personas` map. Write a test with YAML containing a voice profile and verify it deserializes correctly.

- [ ] **Step 5: Run all eidos tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/eidos/pom.xml`
Expected: All tests pass, including new voice tests

- [ ] **Step 6: Install eidos to .m2**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/eidos/pom.xml -DskipTests`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add -A
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(#89): add voice field to AgentDescriptor with YAML deserialization Refs casehubio/examples#89"
```

---

## Batch 2: blocks-core — Renderer + Duplication Cleanup

**Repo:** `/Users/mdproctor/claude/casehub/slots/196/blocks`
**Depends on:** Batch 1 (eidos-api installed to .m2)
**After this batch:** `CognitiveSystemPromptRenderer` renders voice profiles and vocabulary-resolved personality. `PersonalityPromptSection` removed. `GoalPromptSection` renamed. Tests pass.

### Task 3: Update CognitiveSystemPromptRenderer

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CognitiveSystemPromptRenderer.java`
- Modify: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/CognitiveSystemPromptRendererTest.java`
- Test: same test file — add voice profile rendering tests

**Interfaces:**
- Consumes: `AgentDescriptor.voice()` (from Task 2), `AgentDescriptor.disposition()`, `VocabularyRegistry` (from eidos-api)
- Produces: `RenderedPrompt` with voice profile content in system prompt — consumed by ScenarioOrchestrator (existing wiring)

- [ ] **Step 1: Write failing tests for voice rendering**

Add tests to `CognitiveSystemPromptRendererTest`:
- `rendersVoiceProfileWhenPresent()` — descriptor with voice profile → output contains "## Voice", register, catchphrases
- `rendersPersonasWithBaseInheritance()` — descriptor with personas → output contains "## Voice: sneekly" and "## Voice: claw"
- `rendersPersonalityFromDisposition()` — descriptor with disposition → output contains "## Personality"
- `fallsBackToBriefingWhenNoVoice()` — descriptor without voice → current behavior unchanged

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml -Dtest=CognitiveSystemPromptRendererTest`
Expected: FAIL — voice rendering not implemented

- [ ] **Step 3: Update the renderer**

Add `VocabularyRegistry` as an optional constructor parameter (nullable for backward compat). Update `render()` method:

1. Name heading (unchanged)
2. If `descriptor.voice() != null`: render briefing as role line, then personality section (from disposition + vocabulary resolution), then voice section (register, accent, catchphrases, speech patterns, vocabulary, quirks). If personas present, render base voice + each persona.
3. If `descriptor.voice() == null`: fall back to current behavior (briefing as prose)
4. Prime Directives (unchanged)
5. Cognitive preamble (unchanged)

Voice rendering method:
```java
private void renderVoice(StringBuilder sb, AgentVoiceProfile voice) {
    if (voice.register() != null) sb.append("Register: ").append(resolveVocab(voice.register())).append("\n");
    if (voice.accent() != null) sb.append("Accent: ").append(resolveVocab(voice.accent())).append("\n");
    if (voice.catchphrases() != null && !voice.catchphrases().isEmpty()) {
        sb.append("Catchphrases: ").append(String.join(", ", voice.catchphrases().stream().map(c -> "\"" + c + "\"").toList())).append("\n");
    }
    // ... similar for speechPatterns, vocabularyUses, vocabularyAvoids, quirks
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml -Dtest=CognitiveSystemPromptRendererTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/src
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#89): voice profile and personality rendering in CognitiveSystemPromptRenderer Refs casehubio/examples#89"
```

### Task 4: Remove PersonalityPromptSection, rename GoalPromptSection

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java:377-413` — remove PersonalityPromptSection from promptSections()
- Rename: `GoalPromptSection` → `EmergentGoalPromptSection` (use `ide_refactor_rename`)
- Delete: `PersonalityPromptSection.java` (use `ide_refactor_safe_delete` after removing references)
- Modify: tests that reference PersonalityPromptSection or GoalPromptSection

**Interfaces:**
- Consumes: `CognitionCore.promptSections()` (existing)
- Produces: `CognitionCore.promptSections()` without personality section, with renamed goal section

- [ ] **Step 1: Write test asserting PersonalityPromptSection absent**

Add test to existing CognitionCore tests:
```java
@Test void promptSectionsOmitsPersonality() {
    var core = buildCore(CognitionConfig.all(), agentProvider);
    var sections = core.promptSections();
    assertThat(sections).noneMatch(s -> s instanceof PersonalityPromptSection);
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — PersonalityPromptSection still present in promptSections()

- [ ] **Step 3: Remove PersonalityPromptSection from CognitionCore.promptSections()**

Use `ide_replace_member` on `CognitionCore.promptSections()`. Remove the block at lines 380-382 that adds `PersonalityPromptSection`. Keep ConstraintPromptSection (SOFT constraints stay in observation).

- [ ] **Step 4: Rename GoalPromptSection → EmergentGoalPromptSection**

Use `ide_refactor_rename` on `GoalPromptSection` class → `EmergentGoalPromptSection`. IntelliJ updates all references.

- [ ] **Step 5: Safe-delete PersonalityPromptSection**

Use `ide_refactor_safe_delete` on `PersonalityPromptSection.java`. Verify no remaining references.

- [ ] **Step 6: Run all blocks-core tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`
Expected: All tests pass

- [ ] **Step 7: Install blocks to .m2**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml -DskipTests`

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add -A
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#89): remove PersonalityPromptSection, rename GoalPromptSection to EmergentGoalPromptSection Refs casehubio/examples#89"
```

---

## Batch 3: wacky-manor — Character Migration + Persona Switching

**Repo:** `/Users/mdproctor/claude/casehub/slots/196/examples`
**Depends on:** Batch 2 (blocks-core installed to .m2)
**After this batch:** 4 characters use voice profiles. Hooded Claw has persona switching. Template behavioral content moved to SocialConfig. Tests pass.

### Task 5: Rewrite 4 character descriptors with voice profiles

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml` — 4 characters
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/VoiceProfileDescriptorTest.java` (new)

**Interfaces:**
- Consumes: `AgentVoiceProfile` (from Task 1), `AgentDescriptor.voice()` (from Task 2)
- Produces: Updated descriptor YAML consumed by eidos's `AgentDescriptorBootstrap` at startup

- [ ] **Step 1: Write failing test**

```java
@QuarkusTest
class VoiceProfileDescriptorTest {
    @Inject AgentRegistry agentRegistry;

    @Test void penelopeHasVoiceProfile() {
        var desc = agentRegistry.findById("penelope-pitstop", "wacky-manor").orElseThrow();
        assertThat(desc.voice()).isNotNull();
        assertThat(desc.voice().register()).isEqualTo("southern-belle");
        assertThat(desc.voice().catchphrases()).contains("Why, how delightful!");
    }

    @Test void hoodedClawHasPersonas() {
        var desc = agentRegistry.findById("hooded-claw", "wacky-manor").orElseThrow();
        assertThat(desc.voice()).isNotNull();
        assertThat(desc.voice().personas()).containsKeys("sneekly", "claw");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=VoiceProfileDescriptorTest`
Expected: FAIL — voice is null (descriptors not updated)

- [ ] **Step 3: Define shared YAML anchor for Hanna-Barbera conventions**

At the top of `descriptors-composite.yaml`, add:
```yaml
voice-anchors:
  hanna-barbera: &hanna-barbera-voice
    speech-patterns:
      - "Narrate your situation aloud — describe what is happening, how you feel, what you intend to do"
      - "State emotions explicitly with exaggeration"
      - "Use your signature phrases at emotional peaks"
      - "Describe physical events with sound effects and dramatic flair"
```

- [ ] **Step 4: Rewrite Penelope Pitstop descriptor**

Strip behavioral instructions from briefing. Add voice profile with YAML anchor merge. Briefing retains role-only text.

- [ ] **Step 5: Rewrite Dick Dastardly descriptor**

Similar pattern — voice profile with dramatic register, catchphrases, anchor merge.

- [ ] **Step 6: Rewrite Ant Hill Mob descriptor**

Voice profile with brooklyn-gangster accent, ensemble quirks.

- [ ] **Step 7: Rewrite Hooded Claw descriptor**

Voice profile with personas map (sneekly + claw). Base quirks shared. Briefing retains role-only text.

- [ ] **Step 8: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=VoiceProfileDescriptorTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): rewrite 4 character descriptors with voice profiles Refs #89"
```

### Task 6: Add PersonaActivationSection for Hooded Claw

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/PersonaActivationSection.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:100-167` — add persona section to renderCognitiveSections()
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java` — add PersonaMapping record
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` — add persona-constraint mapping for hooded-claw
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/PersonaActivationSectionTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor.voice().personas()` (from Task 1), `AgentDescriptor.constraints()`, nearby agent IDs (from CharacterCognition)
- Produces: `ObservationSection` with active persona signal — consumed by ObservationBuilder in the observation pipeline

- [ ] **Step 1: Write failing test**

```java
class PersonaActivationSectionTest {
    @Test void activatesSneeklyWhenOthersPresent() {
        var mapping = new SocialConfig.PersonaMapping("never-break-cover", "sneekly", "claw");
        var section = new PersonaActivationSection(mapping);
        var result = section.evaluate(Set.of("penelope-pitstop", "peter-perfect"));
        assertThat(result.personaName()).isEqualTo("sneekly");
    }

    @Test void activatesClawWhenAlone() {
        var mapping = new SocialConfig.PersonaMapping("never-break-cover", "sneekly", "claw");
        var section = new PersonaActivationSection(mapping);
        var result = section.evaluate(Set.of());
        assertThat(result.personaName()).isEqualTo("claw");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Create PersonaActivationSection and SocialConfig.PersonaMapping**

`PersonaMapping` record: `(String constraintName, String whenActive, String whenInactive)`. When nearby agents are present, constraint is "active" → use `whenActive` persona. When alone, constraint is "inactive" → use `whenInactive` persona.

- [ ] **Step 4: Wire into CharacterCognition.renderCognitiveSections()**

Add persona activation section at the end of `renderCognitiveSections()`. Only added when the descriptor has personas AND the SocialConfig has a persona mapping.

- [ ] **Step 5: Add persona-constraint mapping to social-config.yaml**

```yaml
hooded-claw:
  persona-mapping:
    constraint: never-break-cover
    when-active: sneekly
    when-inactive: claw
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=PersonaActivationSectionTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): add PersonaActivationSection for multi-voice character switching Refs #89"
```

### Task 7: Move template behavioral content to SocialConfig

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` — add behavioral norms, drives, goals from templates
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml` — remove template refs from the 4 migrated characters (voice content now in voice profile)
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/TemplateMigrationTest.java`

**Interfaces:**
- Consumes: `SocialConfig` records (existing)
- Produces: Updated SocialConfig seed data — consumed by `ManorSocialConfigLoader` → `ManorCognitiveSeeder`

- [ ] **Step 1: Write failing test**

```java
@Test void hoodedClawHasGloatingDriveFromTemplate() {
    var configs = ManorSocialConfigLoader.load();
    var hcConfig = configs.get("hooded-claw");
    assertThat(hcConfig.drives()).anyMatch(d -> d.type().equals("gloating"));
}

@Test void antHillMobHasProtectGoalFromTemplate() {
    var configs = ManorSocialConfigLoader.load();
    var mobConfig = configs.get("ant-hill-mob");
    assertThat(mobConfig.goals()).anyMatch(g -> g.name().equals("protect-penelope"));
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add behavioral content from templates to social-config.yaml**

For each of the 4 characters, extract behavioral instructions from their templates and add to their SocialConfig entries as norms, drives, or goals.

- [ ] **Step 4: Remove template references from the 4 migrated characters**

Remove `templates:` section from the 4 migrated characters in `descriptors-composite.yaml`. Voice content is now in the voice profile. Behavioral content is now in social-config.yaml. Template definitions stay in `templates.yaml` (not deleted — still used by non-migrated characters).

- [ ] **Step 5: Run full wacky-manor test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: All tests pass (including existing tests — backward compatibility preserved for non-migrated characters)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): move template behavioral content to SocialConfig for 4 characters Refs #89"
```

---

## Batch 4: Emergence Verification

**Repo:** `/Users/mdproctor/claude/casehub/slots/196/examples`
**Depends on:** Batch 3
**After this batch:** Three-run emergence eval complete. Output persisted to `docs/eval/`. Evidence of whether cognitive systems produce emergent behavior.

### Task 8: Three-run emergence verification eval

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveEvalTest.java` — add emergence comparison runs
- Create: `wacky-manor/docs/eval/` directory for persisted output
- Test: the eval IS the test

**Interfaces:**
- Consumes: voice-only descriptors (from Task 5), `CognitionConfig.none()` and `CognitionConfig.all()`, `CognitiveEvalTest` infrastructure
- Produces: eval output in `docs/eval/emergence-YYYYMMDD/` with run-A, run-B directories and delta summary

- [ ] **Step 1: Add emergence eval method to CognitiveEvalTest**

```java
@Test @Tag("emergence-eval")
void emergenceVerification() throws IOException {
    var outputDir = Path.of("docs/eval/emergence-" + java.time.LocalDate.now());
    Files.createDirectories(outputDir.resolve("run-a-no-cognition"));
    Files.createDirectories(outputDir.resolve("run-b-full-cognition"));

    // Run A: voice-only, no cognitive systems
    var coreA = buildCore(CognitionConfig.none(), null);
    runScenarioAndCapture(coreA, outputDir.resolve("run-a-no-cognition"));

    // Run B: voice-only, full cognitive systems
    var coreB = buildCore(CognitionConfig.all(), agentProvider);
    runScenarioAndCapture(coreB, outputDir.resolve("run-b-full-cognition"));

    // Structural assertion: Run B produces more cognitive section output
    var deltaA = countCognitiveSections(outputDir.resolve("run-a-no-cognition"));
    var deltaB = countCognitiveSections(outputDir.resolve("run-b-full-cognition"));
    assertThat(deltaB).isGreaterThan(deltaA);
}
```

- [ ] **Step 2: Run the emergence eval**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Pllm-eval -Dtest=CognitiveEvalTest#emergenceVerification`
Expected: PASS — structural assertion that run B has more cognitive content than run A

- [ ] **Step 3: Review the output qualitatively**

Read the delta output in `docs/eval/emergence-*/` and assess:
- Do characters in run B exhibit drive-aligned behavior absent in run A?
- Is mood modulation visible in run B?
- Do characters show distinct behavioral signatures beyond voice differences?

- [ ] **Step 4: Commit eval output**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/test wacky-manor/docs/eval
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): three-run emergence verification eval Refs #89"
```

---

## References

- [2026-09-27-briefing-voice-card-design.md] — design spec this plan implements
- [decisions.md] — D1-D8 design decisions
- [CognitiveSystemPromptRenderer.java:15] — existing renderer to update
- [CognitionCore.java:377] — promptSections() method to modify
- [CharacterCognition.java:100] — renderCognitiveSections() to extend
- [AgentDescriptor.java:7] — eidos-api record to extend
- [descriptors-composite.yaml] — character descriptors to rewrite
- [social-config.yaml] — behavioral seed data
- [templates.yaml] — templates with voice/behavior mix
- [CognitiveEvalTest.java] — eval infrastructure to extend
- [PersonalityPromptSection.java] — to be deleted
- [GoalPromptSection.java] — to be renamed
- [directive-minimal-architecture-design.md] — prior spec establishing CognitiveSystemPromptRenderer
- [casehubio/examples#89] — focal issue
