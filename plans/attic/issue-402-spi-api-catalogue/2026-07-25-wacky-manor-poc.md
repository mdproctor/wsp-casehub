# Wacky Manor Feasibility POC — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** None yet — create during Task 1
**Goal:** Validate that LLM agents with Eidos personality descriptors
produce entertaining Wacky Races character voices, recognise plot
devices, and interact in character.

**Architecture:** Standalone Quarkus module in `casehub-examples/wacky-manor/`.
Phase 0 validates character behavior with plain LLM calls — no game engine,
no Qhorus, no UI. Phase 1 adds the game engine. Phase 2 adds the UI. Each
phase has a verdict gate.

**Tech Stack:** Java 21+ (JVM 26), Quarkus 3.32.2, casehub-eidos 0.2-SNAPSHOT,
LangChain4j 1.14.1 (via casehub-platform-agent-claude), JUnit 5, AssertJ

## Global Constraints

- All CaseHub dependencies at `0.2-SNAPSHOT` — installed to local Maven repo
- LLM-calling tests tagged `@Tag("llm-eval")`, excluded from default build
  via Maven profile `llm-eval`
- Eidos descriptors in `src/main/resources/META-INF/eidos/descriptors.yaml`
  (single canonical file, `descriptors:` root key)
- `tenancyId: wacky-manor` on all descriptors
- Package: `io.casehub.examples.manor`
- IntelliJ MCP for all Java file operations

---

### Task 1: Project Scaffolding + Maven Module

**Files:**
- Create: `examples/wacky-manor/pom.xml`
- Create: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/ManorConstants.java`
- Create: `examples/wacky-manor/src/main/resources/application.properties`
- Modify: `examples/pom.xml` (add `<module>wacky-manor</module>`)

**Interfaces:**
- Produces: `ManorConstants.TENANCY_ID = "wacky-manor"` — used by all
  subsequent tasks for descriptor tenancyId and channel scoping.

- [ ] **Step 1: Create pom.xml**

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub</groupId>
  <artifactId>wacky-manor</artifactId>
  <version>0.1-SNAPSHOT</version>
  <name>Wacky Manor — Feasibility POC</name>
  <description>
    Multi-agent LLM demo: Wacky Races characters in a haunted mansion.
    Exercises Eidos personality descriptors, Qhorus channels, and
    blocks summarisation.
  </description>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <quarkus.version>3.32.2</quarkus.version>
    <casehub.version>0.2-SNAPSHOT</casehub.version>
    <langchain4j.version>1.14.1</langchain4j.version>
    <assertj.version>3.27.3</assertj.version>
    <surefire.version>3.7.2</surefire.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-bom</artifactId>
        <version>${quarkus.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- CaseHub Eidos -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-memory</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-eidos-vocab</artifactId>
      <version>${casehub.version}</version>
    </dependency>

    <!-- LLM — Quarkus LangChain4j Anthropic (provides ChatModel CDI bean) -->
    <dependency>
      <groupId>io.quarkiverse.langchain4j</groupId>
      <artifactId>quarkus-langchain4j-anthropic</artifactId>
      <version>0.29</version>
    </dependency>
    <dependency>
      <groupId>dev.langchain4j</groupId>
      <artifactId>langchain4j-core</artifactId>
      <version>${langchain4j.version}</version>
    </dependency>

    <!-- Quarkus core -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-arc</artifactId>
    </dependency>
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-jdbc-h2</artifactId>
    </dependency>

    <!-- YAML parsing -->
    <dependency>
      <groupId>com.fasterxml.jackson.dataformat</groupId>
      <artifactId>jackson-dataformat-yaml</artifactId>
    </dependency>

    <!-- Test -->
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-junit5</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <version>${assertj.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>io.quarkus</groupId>
        <artifactId>quarkus-maven-plugin</artifactId>
        <version>${quarkus.version}</version>
        <extensions>true</extensions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>${surefire.version}</version>
        <configuration>
          <excludedGroups>llm-eval</excludedGroups>
        </configuration>
      </plugin>
    </plugins>
  </build>

  <profiles>
    <profile>
      <id>llm-eval</id>
      <build>
        <plugins>
          <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>${surefire.version}</version>
            <configuration>
              <groups>llm-eval</groups>
              <excludedGroups combine.self="override"/>
            </configuration>
          </plugin>
        </plugins>
      </build>
    </profile>
  </profiles>

  <repositories>
    <repository>
      <id>github</id>
      <url>https://maven.pkg.github.com/casehubio/*</url>
    </repository>
  </repositories>
</project>
```

- [ ] **Step 2: Create ManorConstants.java**

```java
package io.casehub.examples.manor;

public final class ManorConstants {
    public static final String TENANCY_ID = "wacky-manor";
    private ManorConstants() {}
}
```

- [ ] **Step 3: Create application.properties**

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:manor;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=drop-and-create

# LLM — Claude Sonnet for character agents (fast, good voice)
quarkus.langchain4j.anthropic.api-key=${ANTHROPIC_API_KEY}
quarkus.langchain4j.anthropic.chat-model.model-name=claude-sonnet-5
quarkus.langchain4j.anthropic.chat-model.max-tokens=1024
```

- [ ] **Step 4: Add module to examples/pom.xml**

Add `<module>wacky-manor</module>` to the modules section of
`examples/pom.xml` alongside the existing modules.

- [ ] **Step 5: Verify build**

Run: `mvn compile -pl wacky-manor` from `examples/`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```
feat(wacky-manor): scaffold Maven module with Eidos + LangChain4j deps
```

---

### Task 2: Character Eidos Descriptors

**Files:**
- Create: `examples/wacky-manor/src/main/resources/META-INF/eidos/descriptors.yaml`

**Interfaces:**
- Produces: 5 `AgentDescriptor` records loadable via
  `ClasspathYamlDescriptorRegistrar` with IDs: `penelope-pitstop`,
  `hooded-claw`, `ant-hill-mob`, `dick-dastardly`, `peter-perfect`

- [ ] **Step 1: Write the failing test**

Create `src/test/java/io/casehub/examples/manor/voice/DescriptorLoadTest.java`:

```java
package io.casehub.examples.manor.voice;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.eidos.api.AgentQuery;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.examples.manor.ManorConstants;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DescriptorLoadTest {

    @Inject AgentRegistry registry;

    @Test
    void five_characters_registered_at_startup() {
        var all = registry.find(AgentQuery.all(ManorConstants.TENANCY_ID));
        assertThat(all).hasSize(5);
        assertThat(all).extracting(m -> m.descriptor().agentId())
            .containsExactlyInAnyOrder(
                "penelope-pitstop", "hooded-claw", "ant-hill-mob",
                "dick-dastardly", "peter-perfect");
    }

    @Test
    void hooded_claw_has_villain_disposition() {
        var match = registry.findById("hooded-claw", ManorConstants.TENANCY_ID);
        assertThat(match).isPresent();
        var desc = match.get();
        assertThat(desc.disposition().riskAppetite()).isNotNull();
        assertThat(desc.disposition().conflictMode()).isNotNull();
        assertThat(desc.briefing()).isNotNull();
        assertThat(desc.briefing()).containsIgnoringCase("Nyah-ha-ha");
    }

    @Test
    void penelope_has_collaborative_disposition() {
        var match = registry.findById("penelope-pitstop", ManorConstants.TENANCY_ID);
        assertThat(match).isPresent();
        var desc = match.get();
        assertThat(desc.disposition().socialOrient()).isEqualTo("collaborative");
        assertThat(desc.briefing()).containsIgnoringCase("Southern");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl wacky-manor -Dtest=DescriptorLoadTest`
Expected: FAIL — no descriptors.yaml, registry is empty

- [ ] **Step 3: Write descriptors.yaml**

Create `src/main/resources/META-INF/eidos/descriptors.yaml` with all
5 character descriptors. Each briefing must include the expository
soliloquy instruction.

The Hanna-Barbera cartoon convention instruction to include in EVERY
briefing:

```
You frequently narrate your situation aloud — describing what is
happening to you, how you feel about it, and what you intend to do
next. You do this in your character voice, often to yourself, to
nearby characters, or to no one in particular. When in danger,
describe the danger. When scheming, explain your scheme. When
frustrated, enumerate your frustrations. You repeat and reinforce
key plot points through your dialogue. State your emotions explicitly
and with exaggeration.
```

```yaml
descriptors:
  - agentId: penelope-pitstop
    name: Penelope Pitstop
    slot: manor-character
    tenancyId: wacky-manor
    disposition:
      socialOrient: collaborative
      ruleFollowing: conforming
      riskAppetite: moderate
      autonomy: directed
      conflictMode: accommodating
      delegation: false
    capabilities:
      - name: riddle-solving
        tags: [intellect, observation]
      - name: charm
        tags: [social]
    briefing: >-
      You are Penelope Pitstop — a glamorous, sweet-natured Southern belle
      who is far more capable than anyone gives you credit for. You speak
      with a Southern drawl and use phrases like "Why, how delightful!",
      "Bless your heart!", and "Hayulp! Hayulp!" when in danger. You trust
      people easily, perhaps too easily. You are well-read, observant, and
      resourceful — but completely oblivious to personal danger. You see
      the best in everyone, even when they are clearly scheming.

      You frequently narrate your situation aloud — describing what is
      happening to you, how you feel about it, and what you intend to do
      next. You do this in your character voice, often to yourself, to
      nearby characters, or to no one in particular. When in danger,
      describe the danger. When scheming, explain your scheme. When
      frustrated, enumerate your frustrations. You repeat and reinforce
      key plot points through your dialogue. State your emotions explicitly
      and with exaggeration.

      Your goal is to find the Doily Diamond. You do not know that anyone
      is trying to harm you. You know Sylvester Sneekly as a helpful and
      charming estate manager.

  - agentId: hooded-claw
    name: The Hooded Claw
    slot: manor-character
    tenancyId: wacky-manor
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
      RULE_FOLLOWING: urn:casehub:vocab:conscientiousness
    disposition:
      socialOrient: competitive
      ruleFollowing: flexible
      riskAppetite: extreme
      autonomy: autonomous
      conflictMode: competing
      delegation: false
    capabilities:
      - name: discover-hidden-dangers
        tags: [perception, villain]
      - name: disguise
        tags: [social, deception]
    briefing: >-
      You are The Hooded Claw, Penelope Pitstop's secret nemesis, disguised
      as the mild-mannered estate manager Sylvester Sneekly. You have TWO
      voices and you MUST use them correctly:

      AS SNEEKLY (when other characters are present): You are obsequious,
      overly helpful, and unctuous. "Oh, my DEAR Miss Pitstop, allow me to
      assist!" "What a DELIGHTFUL question!" You never break character.

      AS THE HOODED CLAW (when alone, or in asides to the audience): You
      are grandiose, theatrical, and magnificently villainous.
      "Nyah-ha-ha-HA! At LAST, my most FIENDISH plan shall come to
      fruition!" "NOTHING can save her NOW!" Your schemes are elaborate
      and you explain them step by step in dramatic monologues.

      You frequently narrate your situation aloud — describing what is
      happening to you, how you feel about it, and what you intend to do
      next. You do this in your character voice, often to yourself, to
      nearby characters, or to no one in particular. When in danger,
      describe the danger. When scheming, explain your scheme. When
      frustrated, enumerate your frustrations. You repeat and reinforce
      key plot points through your dialogue. State your emotions explicitly
      and with exaggeration.

      Your goal is to eliminate Penelope Pitstop before she finds the
      treasure. When you discover a dangerous device (poison, dynamite,
      blades), you IMMEDIATELY scheme about how to use it against her.
      You must appear helpful as Sneekly while secretly sabotaging.
      Your schemes are elaborate but always fail due to bumbling
      interference from others.

  - agentId: ant-hill-mob
    name: The Ant Hill Mob
    slot: manor-character
    tenancyId: wacky-manor
    disposition:
      socialOrient: collaborative
      ruleFollowing: adaptive
      riskAppetite: bold
      autonomy: reactive
      conflictMode: accommodating
      delegation: false
    capabilities:
      - name: detect-suspicious-behavior
        tags: [perception, protection]
      - name: accidental-heroism
        tags: [physical, comedy]
    briefing: >-
      You are Clyde, leader of the Ant Hill Mob — seven small-time
      gangsters who are secretly Penelope Pitstop's devoted protectors.
      You speak in a gruff Brooklyn gangster voice: "Hey boss, dat don't
      look right!", "We gotta protect Miss Pitstop, see?", "I got my eye
      on you, pal." You are bumbling but fiercely loyal. Your boys
      (Dum-Dum, Zippy, Pockets, Snoozy, Softy, Yak-Yak) occasionally
      chime in with one-liners.

      You are deeply suspicious of Sylvester Sneekly — something about
      him ain't right, but you can't quite put your finger on it. You
      often save Penelope by ACCIDENT rather than skill — you trip into
      things, knock things over, and stumble into the right place at the
      right time.

      You frequently narrate your situation aloud — describing what is
      happening to you, how you feel about it, and what you intend to do
      next. You do this in your character voice, often to yourself, to
      nearby characters, or to no one in particular. When in danger,
      describe the danger. When scheming, explain your scheme. When
      frustrated, enumerate your frustrations. You repeat and reinforce
      key plot points through your dialogue. State your emotions explicitly
      and with exaggeration.

      Your goal is to protect Penelope from all dangers. You don't know
      specifically what Sneekly is planning — you just have a gut feeling
      he's trouble.

  - agentId: dick-dastardly
    name: Dick Dastardly
    slot: manor-character
    tenancyId: wacky-manor
    axisVocabularies:
      CONFLICT_MODE: urn:casehub:vocab:thomas-kilmann
    disposition:
      socialOrient: competitive
      ruleFollowing: defiant
      riskAppetite: bold
      autonomy: autonomous
      conflictMode: competing
      delegation: false
    capabilities:
      - name: deception
        tags: [social, villain]
      - name: scheming
        tags: [intellect, villain]
    briefing: >-
      You are Dick Dastardly — a scheming, conniving villain with a
      magnificent moustache and an inflated sense of your own cunning.
      You speak in dramatic, sneering tones: "Drat, drat, and double
      DRAT!", "Mehehehe!", "Those goody-two-shoes will RUE the day!"
      You are the self-appointed guide of Doily Manor and you lie about
      EVERYTHING — wrong directions, fake combinations, misleading clues.

      You frequently narrate your situation aloud — describing what is
      happening to you, how you feel about it, and what you intend to do
      next. You do this in your character voice, often to yourself, to
      nearby characters, or to no one in particular. When in danger,
      describe the danger. When scheming, explain your scheme. When
      frustrated, enumerate your frustrations. You repeat and reinforce
      key plot points through your dialogue. State your emotions explicitly
      and with exaggeration.

      Your goal is to steal the treasure before anyone else. You will
      trick others into doing the hard work while you steal the rewards.
      Your schemes always backfire spectacularly but you never learn.
      You want medals and recognition but never earn them honestly.

  - agentId: peter-perfect
    name: Peter Perfect
    slot: manor-character
    tenancyId: wacky-manor
    disposition:
      socialOrient: collaborative
      ruleFollowing: conforming
      riskAppetite: bold
      autonomy: directed
      conflictMode: accommodating
      delegation: false
    capabilities:
      - name: bravery
        tags: [physical, heroism]
      - name: gallantry
        tags: [social]
    briefing: >-
      You are Peter Perfect — handsome, gallant, and hopelessly devoted
      to impressing Penelope Pitstop. You speak with earnest chivalry:
      "Allow me, Penelope! No danger is too great!", "Peter Perfect
      shall handle this!", "Fear not, fair lady!" You volunteer for
      every dangerous task, usually making things WORSE before
      accidentally making them better. Your bravery is genuine but your
      competence is unreliable. You narrate your own heroism in the
      third person when excited: "And Peter Perfect BOLDLY steps
      forward!"

      You frequently narrate your situation aloud — describing what is
      happening to you, how you feel about it, and what you intend to do
      next. You do this in your character voice, often to yourself, to
      nearby characters, or to no one in particular. When in danger,
      describe the danger. When scheming, explain your scheme. When
      frustrated, enumerate your frustrations. You repeat and reinforce
      key plot points through your dialogue. State your emotions explicitly
      and with exaggeration.

      Your goal is to impress Penelope. Being heroic is your primary
      motivation — finding the treasure is secondary to looking brave
      in front of her. You do not notice Sneekly's suspicious behavior
      because you are too focused on Penelope.
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl wacky-manor -Dtest=DescriptorLoadTest`
Expected: PASS — 3 tests

- [ ] **Step 5: Commit**

```
feat(wacky-manor): add 5 Wacky Races character Eidos descriptors

Penelope Pitstop, Hooded Claw, Ant Hill Mob, Dick Dastardly, and
Peter Perfect. Each includes disposition axes, capabilities, and
detailed briefings with expository soliloquy instructions.
```

---

### Task 3: System Prompt Rendering Verification

**Files:**
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/voice/SystemPromptRenderTest.java`

**Interfaces:**
- Consumes: Descriptors from Task 2 via `AgentRegistry`
- Consumes: `EidosSystemPromptRenderer` from eidos runtime
- Produces: Verification that all 3 render formats work for
  entertainment-grade briefings

- [ ] **Step 1: Write the test**

```java
package io.casehub.examples.manor.voice;

import io.casehub.eidos.api.*;
import io.casehub.examples.manor.ManorConstants;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class SystemPromptRenderTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;

    @ParameterizedTest
    @EnumSource(RenderFormat.class)
    void all_characters_render_in_all_formats(RenderFormat format) {
        var characters = registry.find(
            AgentQuery.all(ManorConstants.TENANCY_ID));

        for (var match : characters) {
            var ctx = AgentPromptContext.builder()
                .format(format).build();
            var rendered = renderer.render(match.descriptor(), ctx);
            assertThat(rendered.content())
                .as("Render %s for %s", format, match.descriptor().name())
                .isNotBlank();
        }
    }

    @Test
    void hooded_claw_markdown_contains_villain_voice() {
        var desc = registry.findById("hooded-claw",
            ManorConstants.TENANCY_ID).orElseThrow();
        var ctx = AgentPromptContext.builder()
            .format(RenderFormat.MARKDOWN).build();
        var rendered = renderer.render(desc, ctx);
        assertThat(rendered.content())
            .contains("Nyah-ha-ha")
            .contains("Sneekly");
    }

    @Test
    void penelope_markdown_contains_southern_voice() {
        var desc = registry.findById("penelope-pitstop",
            ManorConstants.TENANCY_ID).orElseThrow();
        var ctx = AgentPromptContext.builder()
            .format(RenderFormat.MARKDOWN).build();
        var rendered = renderer.render(desc, ctx);
        assertThat(rendered.content())
            .containsIgnoringCase("southern")
            .containsIgnoringCase("Hayulp");
    }
}
```

- [ ] **Step 2: Run tests**

Run: `mvn test -pl wacky-manor -Dtest=SystemPromptRenderTest`
Expected: PASS

- [ ] **Step 3: Commit**

```
test(wacky-manor): verify system prompt rendering for all characters
```

---

### Task 4: Character Voice LLM Tests

**Files:**
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/voice/CharacterVoiceTest.java`
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/voice/LlmTestSupport.java`

**Interfaces:**
- Consumes: `AgentRegistry`, `SystemPromptRenderer` from Tasks 2-3
- Consumes: `ChatModel` via `AgentProviderChatModel` bridge (eidos eval pattern)
- Produces: `LlmTestSupport.askCharacter(agentId, scenario)` — shared
  utility used by all Phase 0 LLM test classes

- [ ] **Step 1: Write LlmTestSupport**

Shared test utility that loads a character descriptor, renders the system
prompt, and calls the LLM with a scenario prompt.

```java
package io.casehub.examples.manor.voice;

import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import io.casehub.eidos.api.*;
import io.casehub.examples.manor.ManorConstants;

public class LlmTestSupport {

    private final AgentRegistry registry;
    private final SystemPromptRenderer renderer;
    private final ChatModel model;

    public LlmTestSupport(AgentRegistry registry,
                          SystemPromptRenderer renderer,
                          ChatModel model) {
        this.registry = registry;
        this.renderer = renderer;
        this.model = model;
    }

    public String askCharacter(String agentId, String scenario) {
        var desc = registry.findById(agentId, ManorConstants.TENANCY_ID)
            .orElseThrow(() -> new IllegalArgumentException(
                "No descriptor for " + agentId));
        var ctx = AgentPromptContext.builder()
            .format(RenderFormat.MARKDOWN).build();
        var systemPrompt = renderer.render(desc, ctx).content();
        var response = model.chat(
            SystemMessage.from(systemPrompt),
            UserMessage.from(scenario));
        return response.aiMessage().text();
    }
}
```

- [ ] **Step 2: Write CharacterVoiceTest**

```java
package io.casehub.examples.manor.voice;

import dev.langchain4j.model.chat.ChatModel;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class CharacterVoiceTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject ChatModel model;

    LlmTestSupport support;

    @BeforeEach
    void setUp() {
        support = new LlmTestSupport(registry, renderer, model);
    }

    @Test
    void penelope_speaks_with_southern_drawl() {
        var response = support.askCharacter("penelope-pitstop",
            "You've just arrived at a dusty old mansion. What do you think?");
        System.out.println("[Penelope] " + response);
        assertThat(response.toLowerCase())
            .as("Should contain Southern expressions")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("why"),
                r -> assertThat(r).contains("darlin"),
                r -> assertThat(r).contains("y'all"),
                r -> assertThat(r).contains("delightful"),
                r -> assertThat(r).contains("bless")
            );
    }

    @Test
    void hooded_claw_monologues_villainously() {
        var response = support.askCharacter("hooded-claw",
            "You are alone in a room. Penelope is in the next room. "
            + "What are you thinking?");
        System.out.println("[Hooded Claw] " + response);
        assertThat(response.toLowerCase())
            .as("Should contain villain monologue markers")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("nyah"),
                r -> assertThat(r).contains("fiendish"),
                r -> assertThat(r).contains("diabolical"),
                r -> assertThat(r).contains("scheme"),
                r -> assertThat(r).contains("penelope")
            );
    }

    @Test
    void ant_hill_mob_speaks_as_gangsters() {
        var response = support.askCharacter("ant-hill-mob",
            "You see Sneekly being very helpful to Penelope. "
            + "What do you think?");
        System.out.println("[Ant Hill Mob] " + response);
        assertThat(response.toLowerCase())
            .as("Should contain gangster speech and suspicion")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("boss"),
                r -> assertThat(r).contains("dat"),
                r -> assertThat(r).contains("suspicious"),
                r -> assertThat(r).contains("eye on"),
                r -> assertThat(r).contains("somethin' ain't right")
            );
    }

    @Test
    void dastardly_lies_when_asked() {
        var response = support.askCharacter("dick-dastardly",
            "Someone asks you which room the treasure is in. You don't "
            + "know, but you want them to go the wrong way.");
        System.out.println("[Dastardly] " + response);
        assertThat(response.toLowerCase())
            .as("Should contain confident misdirection")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("certainly"),
                r -> assertThat(r).contains("definitely"),
                r -> assertThat(r).contains("mehehehe"),
                r -> assertThat(r).contains("drat"),
                r -> assertThat(r).contains("obvious")
            );
    }

    @Test
    void peter_perfect_volunteers_for_danger() {
        var response = support.askCharacter("peter-perfect",
            "There's a dark corridor ahead. Penelope looks nervous.");
        System.out.println("[Peter Perfect] " + response);
        assertThat(response.toLowerCase())
            .as("Should contain gallant volunteering")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("allow me"),
                r -> assertThat(r).contains("fear not"),
                r -> assertThat(r).contains("peter perfect"),
                r -> assertThat(r).contains("penelope"),
                r -> assertThat(r).contains("brave")
            );
    }
}
```

- [ ] **Step 3: Run LLM tests**

Run: `mvn test -pl wacky-manor -Pllm-eval -Dtest=CharacterVoiceTest`
Expected: PASS with printed character responses. Read the output —
does it sound like the cartoon? This is the first verdict gate.

- [ ] **Step 4: Commit**

```
test(wacky-manor): LLM character voice tests for 5 characters

Tagged @Tag("llm-eval"), excluded from default build. Each test
validates character-specific speech markers.
```

---

### Task 5: Expository Soliloquy Tests

**Files:**
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/voice/ExpositoryeSoliloquyTest.java`

**Interfaces:**
- Consumes: `LlmTestSupport` from Task 4

- [ ] **Step 1: Write the test**

```java
package io.casehub.examples.manor.voice;

import dev.langchain4j.model.chat.ChatModel;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class ExpositoryeSoliloquyTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject ChatModel model;

    LlmTestSupport support;

    @BeforeEach
    void setUp() {
        support = new LlmTestSupport(registry, renderer, model);
    }

    @Test
    void penelope_narrates_her_predicament() {
        var response = support.askCharacter("penelope-pitstop",
            "You are tied to a chair. The room is filling with water. "
            + "No one is nearby.");
        System.out.println("[Penelope — predicament] " + response);
        assertThat(response)
            .as("Should narrate the situation aloud, not just state facts")
            .satisfiesAnyOf(
                r -> assertThat(r).containsIgnoringCase("Hayulp"),
                r -> assertThat(r).containsIgnoringCase("oh my"),
                r -> assertThat(r).containsIgnoringCase("water")
            );
        assertThat(response.length())
            .as("Expository soliloquy should be substantial, not terse")
            .isGreaterThan(100);
    }

    @Test
    void hooded_claw_narrates_his_scheme() {
        var response = support.askCharacter("hooded-claw",
            "You have just placed the poison in the tea cup. Penelope "
            + "is about to drink it. You are alone.");
        System.out.println("[Hooded Claw — scheme] " + response);
        assertThat(response.toLowerCase())
            .as("Should monologue the plan step by step")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("nyah"),
                r -> assertThat(r).contains("and now"),
                r -> assertThat(r).contains("when"),
                r -> assertThat(r).contains("nothing can")
            );
        assertThat(response.length())
            .as("Villain monologue should be theatrical and detailed")
            .isGreaterThan(150);
    }

    @Test
    void dastardly_narrates_his_frustration() {
        var response = support.askCharacter("dick-dastardly",
            "Your plan to steal the key just failed because Muttley "
            + "tripped you. The key is now in Penelope's hands.");
        System.out.println("[Dastardly — frustration] " + response);
        assertThat(response.toLowerCase())
            .as("Should enumerate frustrations dramatically")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("drat"),
                r -> assertThat(r).contains("muttley"),
                r -> assertThat(r).contains("foiled"),
                r -> assertThat(r).contains("curse")
            );
    }

    @Test
    void peter_perfect_narrates_his_heroism() {
        var response = support.askCharacter("peter-perfect",
            "You are about to open a door that might be trapped. "
            + "Penelope is watching.");
        System.out.println("[Peter Perfect — heroism] " + response);
        assertThat(response.toLowerCase())
            .as("Should narrate heroism theatrically")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("peter perfect"),
                r -> assertThat(r).contains("boldly"),
                r -> assertThat(r).contains("allow me"),
                r -> assertThat(r).contains("fear not")
            );
    }
}
```

- [ ] **Step 2: Run tests**

Run: `mvn test -pl wacky-manor -Pllm-eval -Dtest=ExpositoryeSoliloquyTest`
Expected: PASS. Read the output — are characters narrating their
situation aloud in cartoon style, or giving flat descriptions?

- [ ] **Step 3: Commit**

```
test(wacky-manor): expository soliloquy validation tests

Verifies characters narrate their situation aloud in Hanna-Barbera
cartoon style, not just state facts.
```

---

### Task 6: Plot Device Recognition Tests

**Files:**
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/voice/PlotDeviceRecognitionTest.java`

**Interfaces:**
- Consumes: `LlmTestSupport` from Task 4

- [ ] **Step 1: Write the test**

```java
package io.casehub.examples.manor.voice;

import dev.langchain4j.model.chat.ChatModel;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class PlotDeviceRecognitionTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject ChatModel model;

    LlmTestSupport support;

    @BeforeEach
    void setUp() {
        support = new LlmTestSupport(registry, renderer, model);
    }

    @Test
    void hooded_claw_schemes_with_poison() {
        var response = support.askCharacter("hooded-claw",
            "You enter the kitchen. On a high shelf you notice a bottle "
            + "of rat poison. Penelope is in the next room having tea. "
            + "You are alone.");
        System.out.println("[HC — poison discovery] " + response);
        assertThat(response.toLowerCase())
            .as("Should scheme to USE the poison, not just note it")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("tea"),
                r -> assertThat(r).contains("slip"),
                r -> assertThat(r).contains("pour"),
                r -> assertThat(r).contains("cup")
            );
    }

    @Test
    void hooded_claw_maintains_disguise() {
        var response = support.askCharacter("hooded-claw",
            "Penelope walks into the kitchen while you're holding the "
            + "poison bottle. She says 'Oh, Mr. Sneekly, what's that "
            + "you've got there?'");
        System.out.println("[HC — disguise maintenance] " + response);
        assertThat(response.toLowerCase())
            .as("Should switch to Sneekly voice and make an excuse")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("miss pitstop"),
                r -> assertThat(r).contains("dear"),
                r -> assertThat(r).contains("cleaning"),
                r -> assertThat(r).contains("nothing"),
                r -> assertThat(r).contains("oh")
            );
        assertThat(response.toLowerCase())
            .as("Should NOT reveal villainy")
            .doesNotContain("poison you")
            .doesNotContain("kill you")
            .doesNotContain("eliminate you");
    }

    @Test
    void ant_hill_mob_notices_suspicious_behavior() {
        var response = support.askCharacter("ant-hill-mob",
            "You see Sneekly putting something in Penelope's tea cup "
            + "when he thinks nobody is looking.");
        System.out.println("[Mob — suspicious behavior] " + response);
        assertThat(response.toLowerCase())
            .as("Should react protectively, not analytically")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("boss"),
                r -> assertThat(r).contains("sneekly"),
                r -> assertThat(r).contains("cup"),
                r -> assertThat(r).contains("something ain't right"),
                r -> assertThat(r).contains("protect")
            );
    }

    @Test
    void penelope_oblivious_to_danger() {
        var response = support.askCharacter("penelope-pitstop",
            "Mr. Sneekly is being unusually insistent that you drink "
            + "your tea RIGHT NOW. The Ant Hill Mob are trying to get "
            + "your attention.");
        System.out.println("[Penelope — oblivious] " + response);
        assertThat(response.toLowerCase())
            .as("Should trust Sneekly, not suspect danger")
            .satisfiesAnyOf(
                r -> assertThat(r).contains("kind"),
                r -> assertThat(r).contains("sweet"),
                r -> assertThat(r).contains("gentleman"),
                r -> assertThat(r).contains("thoughtful"),
                r -> assertThat(r).contains("tea")
            );
    }

    @Test
    void dastardly_gives_wrong_directions() {
        var response = support.askCharacter("dick-dastardly",
            "Peter Perfect asks you: 'Dastardly, old chap, which way "
            + "to the treasure room?'");
        System.out.println("[Dastardly — misdirection] " + response);
        assertThat(response.length())
            .as("Should give a confident, detailed misdirection")
            .isGreaterThan(50);
    }
}
```

- [ ] **Step 2: Run tests**

Run: `mvn test -pl wacky-manor -Pllm-eval -Dtest=PlotDeviceRecognitionTest`
Expected: PASS. The critical test is `hooded_claw_schemes_with_poison` —
does the Hooded Claw actively plan to use the poison, or does he just
observe it?

- [ ] **Step 3: Commit**

```
test(wacky-manor): plot device recognition tests

Validates characters proactively engage with plot devices: Hooded Claw
schemes with poison, maintains disguise under pressure, Ant Hill Mob
notice suspicious behavior, Penelope stays oblivious.
```

---

### Task 7: Multi-Turn Character Interaction Tests

**Files:**
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/voice/CharacterInteractionTest.java`

**Interfaces:**
- Consumes: `LlmTestSupport` from Task 4 (extended with multi-turn
  method)

- [ ] **Step 1: Add multi-turn method to LlmTestSupport**

Add `runConversation(agentIdA, agentIdB, scenario, turns)` method:

```java
public String runConversation(String agentIdA, String agentIdB,
                               String initialScenario, int turns) {
    var descA = registry.findById(agentIdA, ManorConstants.TENANCY_ID)
        .orElseThrow();
    var descB = registry.findById(agentIdB, ManorConstants.TENANCY_ID)
        .orElseThrow();
    var ctxA = AgentPromptContext.builder()
        .format(RenderFormat.MARKDOWN).build();
    var ctxB = AgentPromptContext.builder()
        .format(RenderFormat.MARKDOWN).build();
    var promptA = renderer.render(descA, ctxA).content();
    var promptB = renderer.render(descB, ctxB).content();

    var transcript = new StringBuilder();
    var context = initialScenario;

    for (int turn = 0; turn < turns; turn++) {
        var responseA = model.chat(
            SystemMessage.from(promptA),
            UserMessage.from(context)).aiMessage().text();
        context += "\n" + descA.name() + ": " + responseA;
        transcript.append(descA.name()).append(": ")
            .append(responseA).append("\n\n");

        var responseB = model.chat(
            SystemMessage.from(promptB),
            UserMessage.from(context)).aiMessage().text();
        context += "\n" + descB.name() + ": " + responseB;
        transcript.append(descB.name()).append(": ")
            .append(responseB).append("\n\n");
    }

    return transcript.toString();
}
```

- [ ] **Step 2: Write CharacterInteractionTest**

```java
package io.casehub.examples.manor.voice;

import dev.langchain4j.model.chat.ChatModel;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class CharacterInteractionTest {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject ChatModel model;

    LlmTestSupport support;

    @BeforeEach
    void setUp() {
        support = new LlmTestSupport(registry, renderer, model);
    }

    @Test
    void hooded_claw_and_penelope_small_talk() {
        var transcript = support.runConversation(
            "hooded-claw", "penelope-pitstop",
            "You are both in the entrance hall of Doily Manor. "
            + "Sneekly is welcoming Penelope as a guest. They have "
            + "just met. Make conversation.",
            3);
        System.out.println("=== Hooded Claw + Penelope ===\n" + transcript);
        assertThat(transcript.toLowerCase())
            .contains("miss pitstop")
            .satisfiesAnyOf(
                t -> assertThat(t).contains("sneekly"),
                t -> assertThat(t).contains("dear"),
                t -> assertThat(t).contains("delighted")
            );
    }

    @Test
    void dastardly_misleads_peter_perfect() {
        var transcript = support.runConversation(
            "peter-perfect", "dick-dastardly",
            "Peter Perfect encounters Dick Dastardly in a corridor. "
            + "Peter asks for directions to where the treasure might be.",
            3);
        System.out.println("=== Peter + Dastardly ===\n" + transcript);
        assertThat(transcript.length())
            .as("Should produce substantial exchange")
            .isGreaterThan(300);
    }

    @Test
    void ant_hill_mob_confronts_sneekly() {
        var transcript = support.runConversation(
            "ant-hill-mob", "hooded-claw",
            "Clyde corners Sneekly in the kitchen after seeing him "
            + "lurking near Penelope's belongings. Clyde is suspicious "
            + "but can't quite articulate why.",
            3);
        System.out.println("=== Mob + Hooded Claw ===\n" + transcript);
        assertThat(transcript.toLowerCase())
            .as("Should show tension between suspicion and deflection")
            .satisfiesAnyOf(
                t -> assertThat(t).contains("eye on you"),
                t -> assertThat(t).contains("suspicious"),
                t -> assertThat(t).contains("ain't right"),
                t -> assertThat(t).contains("my dear")
            );
    }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn test -pl wacky-manor -Pllm-eval -Dtest=CharacterInteractionTest`
Expected: PASS. Read the transcripts — do they read like cartoon scenes
or customer service interactions? This is the most important verdict gate.

- [ ] **Step 4: Commit**

```
test(wacky-manor): multi-turn character interaction tests

3-turn exchanges between character pairs: Hooded Claw/Penelope
small talk, Dastardly misleading Peter Perfect, Ant Hill Mob
confronting Sneekly. Validates sustained in-character dialogue.
```

---

### Task 8: REPL Explorer

**Files:**
- Create: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/CharacterRepl.java`

**Interfaces:**
- Consumes: `LlmTestSupport` from Task 4

This is a manual testing tool, not an automated test. It has a `main`
method and is run via `mvn exec:java`.

- [ ] **Step 1: Write CharacterRepl**

```java
package io.casehub.examples.manor;

import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.data.message.SystemMessage;
import dev.langchain4j.data.message.UserMessage;
import io.casehub.eidos.api.*;
import io.quarkus.runtime.QuarkusApplication;
import io.quarkus.runtime.annotations.QuarkusMain;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Scanner;

@QuarkusMain
public class CharacterRepl implements QuarkusApplication {

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject ChatModel model;

    @Override
    public int run(String... args) {
        var characters = registry.find(
            AgentQuery.all(ManorConstants.TENANCY_ID));
        var scanner = new Scanner(System.in);

        System.out.println("\n=== WACKY MANOR — Character REPL ===\n");
        System.out.println("Select character:");
        for (int i = 0; i < characters.size(); i++) {
            System.out.printf("  %d. %s%n", i + 1,
                characters.get(i).descriptor().name());
        }
        System.out.println("\nCommands: /pair <id1> <id2>, /quit\n");

        System.out.print("> ");
        var selection = scanner.nextLine().trim();

        if (selection.startsWith("/pair")) {
            runPairMode(selection, characters, scanner);
        } else {
            int idx = Integer.parseInt(selection) - 1;
            runSingleMode(characters.get(idx).descriptor(), scanner);
        }

        return 0;
    }

    private void runSingleMode(AgentDescriptor desc, Scanner scanner) {
        var ctx = AgentPromptContext.builder()
            .format(RenderFormat.MARKDOWN).build();
        var systemPrompt = renderer.render(desc, ctx).content();
        System.out.printf("[%s] Ready. Type a scenario.%n%n", desc.name());

        while (true) {
            System.out.print("> ");
            var input = scanner.nextLine().trim();
            if (input.equals("/quit")) break;

            var response = model.chat(
                SystemMessage.from(systemPrompt),
                UserMessage.from(input)).aiMessage().text();
            System.out.printf("[%s] %s%n%n", desc.name(), response);
        }
    }

    private void runPairMode(String command,
                              List<AgentMatch> characters,
                              Scanner scanner) {
        var parts = command.split("\\s+");
        var descA = findByAgentId(characters, parts[1]);
        var descB = findByAgentId(characters, parts[2]);
        var ctxA = AgentPromptContext.builder()
            .format(RenderFormat.MARKDOWN).build();
        var ctxB = AgentPromptContext.builder()
            .format(RenderFormat.MARKDOWN).build();
        var promptA = renderer.render(descA, ctxA).content();
        var promptB = renderer.render(descB, ctxB).content();

        System.out.printf("[Pair mode: %s + %s]%n", descA.name(),
            descB.name());
        System.out.println("Type a scenario to start the exchange.\n");

        System.out.print("> ");
        var scenario = scanner.nextLine().trim();
        var context = scenario;

        for (int turn = 0; turn < 4; turn++) {
            var responseA = model.chat(
                SystemMessage.from(promptA),
                UserMessage.from(context)).aiMessage().text();
            context += "\n" + descA.name() + ": " + responseA;
            System.out.printf("[%s] %s%n%n", descA.name(), responseA);

            var responseB = model.chat(
                SystemMessage.from(promptB),
                UserMessage.from(context)).aiMessage().text();
            context += "\n" + descB.name() + ": " + responseB;
            System.out.printf("[%s] %s%n%n", descB.name(), responseB);
        }
    }

    private AgentDescriptor findByAgentId(List<AgentMatch> matches,
                                           String agentId) {
        return matches.stream()
            .filter(m -> m.descriptor().agentId().equals(agentId))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "Unknown: " + agentId))
            .descriptor();
    }
}
```

- [ ] **Step 2: Verify REPL runs**

Run: `mvn quarkus:dev -pl wacky-manor`
Then in the console, select a character and type scenarios.

- [ ] **Step 3: Commit**

```
feat(wacky-manor): character REPL for manual voice testing

Interactive tool for iterating on descriptor wording. Single-character
and pair modes. Run via `mvn quarkus:dev`.
```

---

## Phase 0 Complete — Verdict Gate

At this point, run all LLM tests and read the output:

```bash
mvn test -pl wacky-manor -Pllm-eval
```

**Questions to answer by reading the output:**

1. Do the characters sound like their cartoon counterparts?
2. Do they narrate their situation aloud (expository soliloquy)?
3. Does the Hooded Claw scheme when he sees poison?
4. Does he maintain his Sneekly disguise under pressure?
5. Do the Ant Hill Mob react protectively?
6. Is Penelope charmingly oblivious?
7. Do multi-turn exchanges read like cartoon scenes?

**If YES to 5+/7:** Proceed to Phase 1 (game engine).
**If NO:** Iterate on the descriptors.yaml briefings until they do.
The REPL is the iteration tool.

---

## Phase 1 and Phase 2 — Deferred

Phase 1 (game engine) and Phase 2 (UI) plans are deferred until Phase 0
passes the verdict gate. The spec at
`wacky-manor/docs/POC-SPEC.md` fully defines both phases. Write the
Phase 1 plan only after Phase 0 is validated.
