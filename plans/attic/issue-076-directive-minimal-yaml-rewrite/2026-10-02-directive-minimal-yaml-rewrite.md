# Directive-Minimal YAML Rewrite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #76 — Directive-minimal YAML rewrite and seeding for wacky-manor
**Issue group:** #76

**Goal:** Restructure wacky-manor character definitions so every piece of content is typed and in one structural home — briefings carry identity only, templates carry voice only, social-config carries all cognitive/behavioral data.

**Architecture:** Add `tendencies` as a new structured list in social-config. Strip briefings to 1-2 sentence identity. Strip role templates to voice-only. Add voice sections to all characters. Render tendencies as the first cognitive section in the observation. All existing tests must continue to pass.

**Tech Stack:** Java 26, Quarkus, Maven, YAML (Jackson dataformat-yaml)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl wacky-manor -s slot-settings.xml`
- Use `mcp__intellij-index__*` tools for all code navigation and editing
- No content in multiple homes — if it's a tendency, it's ONLY in tendencies
- Briefings under 40 words
- Composite profile only (descriptors-composite.yaml)

---

## Batch 1: Java infrastructure for tendencies

### Task 1: Add tendencies field to SocialConfig and wire parsing + rendering

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` (add one tendency to penelope-pitstop for test)

**Interfaces:**
- Produces: `SocialConfig.tendencies()` → `List<String>` accessor
- Produces: `CharacterCognition.renderCognitiveSections()` renders "Your Behavioral Tendencies" as first section

- [ ] **Step 1: Write failing test for SocialConfig tendencies parsing**

In `ManorSocialConfigLoaderTest.java`, add:

```java
@Test
void parsesNewTendenciesField() {
    var configs = ManorSocialConfigLoader.load();
    var penelope = configs.get("penelope-pitstop");
    assertThat(penelope.tendencies()).isNotEmpty();
    assertThat(penelope.tendencies().get(0)).contains("wellbeing");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorSocialConfigLoaderTest#parsesNewTendenciesField -s slot-settings.xml`
Expected: compilation error — `tendencies()` method not found on SocialConfig

- [ ] **Step 3: Add tendencies field to SocialConfig record**

Use `ide_replace_member` to update the SocialConfig record. Add `List<String> tendencies` as the last field. Update the compact constructor to default `tendencies` to `List.of()` if null. Update `SocialConfig.empty()` to pass `List.of()` for tendencies. Update the secondary constructor (7-arg) to pass `List.of()` for tendencies.

```java
public record SocialConfig(
        List<GoalConfig> goals,
        List<Drive> drives,
        List<NormEntry> norms,
        List<InitialBelief> initialBeliefs,
        List<Relationship> relationships,
        Map<String, List<ReinforcementMapping>> reinforcement,
        RelationshipStageConfig stageConfig,
        @Nullable PersonaConstraintMapping personaConstraint,
        List<String> tendencies
) {
    public SocialConfig {
        goals          = goals != null ? List.copyOf(goals) : List.of();
        drives         = drives != null ? List.copyOf(drives) : List.of();
        norms          = norms != null ? List.copyOf(norms) : List.of();
        initialBeliefs = initialBeliefs != null ? List.copyOf(initialBeliefs) : List.of();
        relationships  = relationships != null ? List.copyOf(relationships) : List.of();
        reinforcement  = reinforcement != null ? Map.copyOf(reinforcement) : Map.of();
        if (stageConfig == null) {stageConfig = RelationshipStageConfig.defaults();}
        tendencies     = tendencies != null ? List.copyOf(tendencies) : List.of();
    }

    public SocialConfig(List<GoalConfig> goals, List<Drive> drives, List<NormEntry> norms,
                         List<InitialBelief> initialBeliefs, List<Relationship> relationships,
                         Map<String, List<ReinforcementMapping>> reinforcement,
                         RelationshipStageConfig stageConfig) {
        this(goals, drives, norms, initialBeliefs, relationships, reinforcement, stageConfig, null, List.of());
    }

    public static SocialConfig empty() {
        return new SocialConfig(List.of(), List.of(), List.of(), List.of(), List.of(), Map.of(), null, null, List.of());
    }
    // ... rest unchanged
}
```

- [ ] **Step 4: Add tendencies parsing to ManorSocialConfigLoader**

Use `ide_edit_member` to update the `parseCharacterConfig` method. Add tendencies parsing before the return statement:

```java
@SuppressWarnings("unchecked")
var tendencies = raw.containsKey("tendencies")
    ? ((List<String>) raw.get("tendencies"))
    : List.<String>of();
```

Update the return statement to pass `tendencies`:

```java
return new SocialConfig(goals, drives, norms, beliefs, relationships, reinforcement, stageConfig, personaConstraint, tendencies);
```

- [ ] **Step 5: Add one tendency to penelope-pitstop in social-config.yaml**

Add to the `penelope-pitstop` section in `social-config.yaml`:

```yaml
penelope-pitstop:
  tendencies:
    - "You keep track of everyone's wellbeing and notice when someone is upset or in trouble"
  goals:
    ...
```

- [ ] **Step 6: Run parsing test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorSocialConfigLoaderTest#parsesNewTendenciesField -s slot-settings.xml`
Expected: PASS

- [ ] **Step 7: Write failing test for tendencies rendering**

In `CharacterCognitionTest.java`, add:

```java
@Test
void rendersTendenciesAsFirstCognitiveSection() {
    var socialConfig = ManorSocialConfigLoader.load().get("penelope-pitstop");
    var cognition = new CharacterCognition("penelope-pitstop", null, null,
            socialConfig, java.util.List.of());
    var sections = cognition.renderCognitiveSections(
            new io.casehub.examples.manor.model.CharacterState(
                    "penelope-pitstop", "Penelope", "Room", 0.0, java.util.List.of()),
            java.util.List.of(), java.util.Map.of());
    assertThat(sections).isNotEmpty();
    assertThat(sections.get(0).header()).isEqualTo("Your Behavioral Tendencies");
}
```

- [ ] **Step 8: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CharacterCognitionTest#rendersTendenciesAsFirstCognitiveSection -s slot-settings.xml`
Expected: FAIL — first section is not "Your Behavioral Tendencies"

- [ ] **Step 9: Add tendencies rendering to CharacterCognition**

Use `ide_edit_member` to update `renderCognitiveSections`. Add tendencies rendering at the TOP of the method, before the existing belief rendering:

```java
if (!socialConfig.tendencies().isEmpty()) {
    sections.add(ObservationSection.items(
            "Your Behavioral Tendencies", null, socialConfig.tendencies()));
}
```

- [ ] **Step 10: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CharacterCognitionTest#rendersTendenciesAsFirstCognitiveSection -s slot-settings.xml`
Expected: PASS

- [ ] **Step 11: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests pass

- [ ] **Step 12: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml
git commit -m "feat(#76): add tendencies field to SocialConfig with parsing and rendering"
```

---

## Batch 2: Content rewrite — templates, descriptors, social-config

### Task 2: Strip role templates to voice-only

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/templates.yaml`

**Interfaces:**
- Consumes: nothing
- Produces: templates with reduced parameter lists (cartoon-villain: [catchphrase, scheme_style], cartoon-hero: [heroic_trait], cartoon-protector: [protection_style])

- [ ] **Step 1: Edit cartoon-villain template**

Replace the content of the `cartoon-villain` template entry:

```yaml
- id: cartoon-villain
  name: Cartoon Villain Voice
  parameters: [catchphrase, scheme_style]
  content: |
    Your signature catchphrase is "${catchphrase}".
    You monologue your plans in a ${scheme_style} manner.
```

- [ ] **Step 2: Edit cartoon-hero template**

```yaml
- id: cartoon-hero
  name: Cartoon Hero Voice
  parameters: [heroic_trait]
  content: |
    Your defining heroic trait is ${heroic_trait}.
```

- [ ] **Step 3: Edit cartoon-protector template**

```yaml
- id: cartoon-protector
  name: Cartoon Protector Voice
  parameters: [protection_style]
  content: |
    Your protection style is ${protection_style}.
```

- [ ] **Step 4: Commit**

```bash
git add wacky-manor/src/main/resources/META-INF/eidos/templates.yaml
git commit -m "feat(#76): strip role templates to voice-only content"
```

### Task 3: Rewrite descriptors — briefings, voice sections, template refs

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml`

**Interfaces:**
- Consumes: Task 2 template parameter changes (dropped nemesis, motivation, protected_character)
- Produces: all 18 characters with 1-2 sentence briefings, voice sections, and updated template refs

- [ ] **Step 1: Rewrite all briefings to identity-only**

For each character, replace the `briefing` field with the identity-only text from the spec (Section 1). Example for Peter Perfect:

```yaml
briefing: >-
  You are Peter Perfect — handsome, gallant, and devoted to
  protecting Penelope Pitstop.
```

Full list of 18 briefing rewrites is in the spec Section 1 table.

- [ ] **Step 2: Add voice sections to all characters lacking them**

Add `voice` sections for: Peter Perfect, Muttley, Lazy Luke, Blubber Bear, Pat Pending, Sergeant Blast, Private Meekly, Rock Slag, Gravel Slag, Rufus Ruffcut, Sawtooth, Big Gruesome, Little Gruesome, Red Max.

Example for Peter Perfect:
```yaml
voice:
  description: "Earnest, decisive chivalry with third-person heroic narration"
  register: gallant-planner
  accent: earnest-heroic
  quirks:
    - narrates own heroism in third person
    - references which step of his plan he is executing
    - "Allow me, Penelope! I have PREPARED for precisely this situation!"
    - "Peter Perfect has ANTICIPATED this threat!"
    - "Fear not — phase THREE is already in motion!"
```

Example for Muttley:
```yaml
voice:
  description: "Canine sounds only — snickering, grumbling, whimpering, sniffing"
  register: non-verbal-canine
  quirks:
    - snickering (Hehehehehehe!)
    - grumbling (Rassafrassa...)
    - enthusiastic sniffing when danger or objects are near
```

Example for Lazy Luke:
```yaml
voice:
  description: "Slow, drowsy Southern drawl — speaks rarely and falls asleep mid-sentence"
  register: sleepy-drawl
  accent: southern-drawl
  quirks:
    - "Well now... I reckon... *yawns*... that can wait till tomorrow..."
    - falls asleep mid-sentence
```

Example for Sergeant Blast:
```yaml
voice:
  description: "Military bark — orders, demands, rule citations"
  register: military-commander
  accent: barking-authority
  quirks:
    - "HALT! Nobody passes without the PASSWORD!"
    - "That is a DIRECT violation of Section 47, Paragraph 3!"
    - "MEEKLY! Did I give you permission to BREATHE?!"
```

Example for Private Meekly:
```yaml
voice:
  description: "Timid stammer — apologetic, hesitant, self-deprecating"
  register: timid-subordinate
  accent: stammering
  quirks:
    - "I-I didn't want to make a fuss, sir..."
    - "S-sorry to bother you, but the exit is actually that way..."
    - "I-I just happened to find this dynamite and thought it best to... move it."
```

Example for Pat Pending:
```yaml
voice:
  description: "Technical jargon that confuses everyone — enthusiastic, oblivious"
  register: technical-inventor
  quirks:
    - "Ah yes, the reciprocating oscillation of the main flywheel suggests a phase-coupling deficiency!"
    - speaks in technical terms nobody understands
```

Example for Rock Slag:
```yaml
voice:
  description: "Extremely limited vocabulary — single words and grunts"
  register: caveman-minimal
  quirks:
    - "Slag!"
    - "Ugh!"
    - "Hmm?"
    - "SMASH!"
    - occasionally a longer grunt of confusion
```

Example for Gravel Slag:
```yaml
voice:
  description: "Short confused grunts — slightly more articulate than Rock"
  register: caveman-confused
  quirks:
    - "Huh?"
    - "Rock... why?"
    - "Gravel fix! ...oops."
    - "Slag? Why smash?"
```

Example for Rufus Ruffcut:
```yaml
voice:
  description: "Plain-spoken lumberjack — direct, no-nonsense, frustrated by city ways"
  register: plain-spoken
  accent: rural-practical
  quirks:
    - "That ain't gonna hold without a proper wrench."
    - "Back in the woods, we'd just use a good honest wrench."
    - "City folk and their complicated gadgets..."
```

Example for Sawtooth:
```yaml
voice:
  description: "Non-verbal beaver — teeth-chattering, tail-slapping, gnawing"
  register: non-verbal-beaver
  quirks:
    - teeth-chattering
    - tail-slapping on the floor
    - enthusiastic nodding
    - gnawing
```

Example for Big Gruesome:
```yaml
voice:
  description: "Deep, gentle, delighted — everything is LOVELY"
  register: gentle-monster
  quirks:
    - "Oh, isn't this LOVELY!"
    - "I think the cobwebs look MUCH better over here!"
    - "What a PRETTY thing!"
```

Example for Little Gruesome:
```yaml
voice:
  description: "Squeaks only — excited, urgent, frustrated, happy squeaks"
  register: squeak-only
  quirks:
    - "SQUEAK squeak squeak SQUEAK!"
    - squeaks urgently and flaps toward passages
    - tugs at clothing to lead others
```

Example for Red Max:
```yaml
voice:
  description: "Dramatic military flair — every situation is an aerial engagement"
  register: flying-ace
  accent: dramatic-military
  quirks:
    - refers to himself in the third person when boasting
    - "The Red Max makes his approach!"
    - "Tally-ho! Enemy territory ahead!"
    - "No obstacle can ground the Red Max!"
```

- [ ] **Step 3: Update template references to drop removed parameters**

Four characters need template ref updates:

Hooded Claw — remove `nemesis` arg:
```yaml
- ref: cartoon-villain
    args:
      catchphrase: "Nyah-ha-ha-HA!"
      scheme_style: grandiose and theatrical
```

Dick Dastardly — remove `nemesis` arg:
```yaml
- ref: cartoon-villain
    args:
      catchphrase: "Drat, drat, and double DRAT!"
      scheme_style: sneering and self-aggrandising
```

Peter Perfect — remove `motivation` arg:
```yaml
- ref: cartoon-hero
    args:
      heroic_trait: gallant chivalry
```

Ant Hill Mob — remove `protected_character` arg:
```yaml
- ref: cartoon-protector
    args:
      protection_style: bumbling and accidental
```

- [ ] **Step 4: Run full test suite to verify descriptors still load**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests pass (descriptor loading, voice tests, etc.)

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml
git commit -m "feat(#76): rewrite descriptors — identity briefings, voice sections, template ref updates"
```

### Task 4: Expand social-config with tendencies and beliefs for all characters

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`

**Interfaces:**
- Consumes: Task 1 tendencies field in SocialConfig
- Produces: all 18 characters with tendencies and expanded beliefs

- [ ] **Step 1: Add tendencies for Hooded Claw**

```yaml
hooded-claw:
  tendencies:
    - "You explain your schemes step by step, especially when you believe you are about to succeed"
    - "You gloat prematurely"
    - "Your plans are always elaborate when simple would work"
```

- [ ] **Step 2: Add tendencies for Dick Dastardly**

```yaml
dick-dastardly:
  tendencies:
    - "You explain your schemes step by step when you believe you are about to succeed"
    - "You gloat prematurely"
    - "You treat everyone as a rival and adversary"
```

- [ ] **Step 3: Add tendencies for Peter Perfect**

```yaml
peter-perfect:
  tendencies:
    - "You plan obsessively before acting"
    - "You never improvise — you prepare, commit, and see every plan through"
    - "You volunteer for danger without hesitation"
    - "You maintain optimistic determination when things go wrong"
    - "You never give up, even when your competence does not match your confidence"
```

- [ ] **Step 4: Add tendencies for Ant Hill Mob**

```yaml
ant-hill-mob:
  tendencies:
    - "You are suspicious of anyone too helpful toward Penelope"
```

- [ ] **Step 5: Add tendencies and beliefs for Muttley**

```yaml
muttley:
  tendencies:
    - "You have an extraordinary nose — you can smell danger, hidden objects, and suspicious substances"
    - "If someone offers you a medal — any medal, real or fake — you will happily trade anything for it"
  initial-beliefs:
    - key: brass-key-attitude
      value: "You have a brass key but do not care about it at all"
    - key: medal-value
      value: "Medals are the most valuable things in the world"
```

- [ ] **Step 6: Add tendencies and beliefs for remaining characters**

Add for each character per the spec Section 5:
- Lazy Luke: 2 tendencies
- Blubber Bear: 1 tendency, 1 belief
- Pat Pending: 2 tendencies
- Sergeant Blast: 2 tendencies
- Private Meekly: 2 tendencies
- Rock Slag: 1 tendency, 1 belief
- Gravel Slag: 2 tendencies
- Rufus Ruffcut: 2 tendencies, 1 belief
- Sawtooth: 2 tendencies, 1 belief
- Big Gruesome: 2 tendencies, 1 belief
- Little Gruesome: 2 tendencies, 1 belief
- Red Max: 2 tendencies, 1 belief

Full content for each character is in the spec Section 5.

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests pass

- [ ] **Step 8: Commit**

```bash
git add wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml
git commit -m "feat(#76): expand social-config with tendencies and beliefs for all characters"
```

---

## Batch 3: Integration validation

### Task 5: Expand DirectiveMinimalIntegrationTest

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/DirectiveMinimalIntegrationTest.java`

**Interfaces:**
- Consumes: Task 1 tendencies rendering, Task 3 descriptor rewrites, Task 4 social-config expansion

- [ ] **Step 1: Write briefing content check test**

```java
@Test
void allBriefingsAreIdentityOnly() {
    var registrar = new io.casehub.examples.manor.ProfileAwareDescriptorRegistrar(
            io.casehub.examples.manor.model.ProfileMode.COMPOSITE);
    for (var desc : registrar.descriptors()) {
        var briefing = desc.briefing();
        if (briefing == null || briefing.isBlank()) continue;
        var wordCount = briefing.strip().split("\\s+").length;
        assertThat(wordCount)
                .as("Briefing for %s is %d words (max 40)", desc.agentId(), wordCount)
                .isLessThanOrEqualTo(40);
        assertThat(briefing)
                .as("Briefing for %s should not contain situational instructions", desc.agentId())
                .doesNotContainIgnoringCase("when you")
                .doesNotContainIgnoringCase("before you")
                .doesNotContainIgnoringCase("if someone");
    }
}
```

- [ ] **Step 2: Write voice section coverage test**

```java
@Test
void allCharactersHaveVoiceSections() {
    var registrar = new io.casehub.examples.manor.ProfileAwareDescriptorRegistrar(
            io.casehub.examples.manor.model.ProfileMode.COMPOSITE);
    for (var desc : registrar.descriptors()) {
        assertThat(desc.voice())
                .as("Character %s should have a voice section", desc.agentId())
                .isNotNull();
    }
}
```

- [ ] **Step 3: Write tendencies rendering position test**

```java
@Test
void tendenciesRenderedAsFirstCognitiveSection() {
    var socialConfigs = ManorSocialConfigLoader.load();
    for (var entry : socialConfigs.entrySet()) {
        if (entry.getValue().tendencies().isEmpty()) continue;
        var cognition = new CharacterCognition(entry.getKey(), null, null,
                entry.getValue(), java.util.List.of());
        var sections = cognition.renderCognitiveSections(
                new io.casehub.examples.manor.model.CharacterState(
                        entry.getKey(), entry.getKey(), "Room", 0.0, java.util.List.of()),
                java.util.List.of(), java.util.Map.of());
        assertThat(sections).isNotEmpty();
        assertThat(sections.get(0).header())
                .as("First cognitive section for %s should be tendencies", entry.getKey())
                .isEqualTo("Your Behavioral Tendencies");
    }
}
```

- [ ] **Step 4: Write all-characters-have-tendencies test**

```java
@Test
void allCharactersHaveTendencies() {
    var socialConfigs = ManorSocialConfigLoader.load();
    for (var entry : socialConfigs.entrySet()) {
        assertThat(entry.getValue().tendencies())
                .as("Character %s should have tendencies", entry.getKey())
                .isNotEmpty();
    }
}
```

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests pass including new integration tests

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/DirectiveMinimalIntegrationTest.java
git commit -m "feat(#76): expand DirectiveMinimalIntegrationTest for structured content taxonomy"
```

---

## References

- [2026-10-02-directive-minimal-yaml-rewrite-design.md] — design spec this plan implements
- [SocialConfig.java] — record definition with constructors and factory methods
- [ManorSocialConfigLoader.java] — YAML parsing with manual Jackson mapping
- [CharacterCognition.java:100-172] — renderCognitiveSections method
- [DirectiveMinimalIntegrationTest.java] — existing 4 integration tests
- [descriptors-composite.yaml] — 18 character descriptors
- [social-config.yaml] — existing goals/drives/norms/beliefs
- [templates.yaml] — 4 templates (1 genre + 3 role)
- [ProfileAwareDescriptorRegistrar.java] — profile-aware descriptor loading
- [GitHub #76] — focal issue
- [casehubio/neocortex#394] — directive-minimal architecture
