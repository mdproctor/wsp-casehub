# Directive-Minimal Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/blocks#283 — Directive-minimal architecture
**Issue group:** casehubio/blocks#283, casehubio/examples#TBD (file before Batch 3)

**Goal:** Shift cognitive data from the briefing directive (system prompt) to neurocortex observation sections (user prompt), so the LLM treats dynamic cognitive state as active rather than supplementary.

**Architecture:** Custom `CognitiveSystemPromptRenderer` overrides eidos renderer via `@Alternative @Priority(1)`. Renders minimal directive: identity + voice + Prime Directives (HARD constraints) + auto-generated cognitive preamble. All dynamic cognitive data (goals, drives, beliefs, strategies, mood, narrative) rendered exclusively through `CognitionCore.promptSections()` observation sections.

**Tech Stack:** Java 26, Quarkus, CDI, eidos-api SPI, blocks-core

## Global Constraints

- Eidos `AgentDescriptor` model unchanged — no new fields
- Eidos `EidosSystemPromptRenderer` untouched — override only
- `CognitionConfig` record fields unchanged — only CDI registration added
- All template content stays in system prompt (all 4 templates are voice/style)
- `ManorCognitiveSeeder` enhanced, not replaced
- Goal seeding requires `DriveAxis` annotation in social-config.yaml

---

## Batch 1: Cognitive Renderer Foundation (blocks)

### Task 1: CognitivePreambleGenerator

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CognitivePreambleGenerator.java`
- Test: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/CognitivePreambleGeneratorTest.java`

**Interfaces:**
- Consumes: `CognitionConfig` record (fields: `moodEnabled`, `drivesEnabled`, `mentalModelEnabled`, `userModelEnabled`, `strategyEnabled`, `narrativeEnabled`, `goalsEnabled`)
- Produces: `String generate(CognitionConfig config)` — returns cognitive preamble paragraph

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.CognitionConfig;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CognitivePreambleGeneratorTest {

    @Test
    void allSubsystemsEnabled_producesCompletePreamble() {
        var config = CognitionConfig.all();
        var preamble = CognitivePreambleGenerator.generate(config);
        assertNotNull(preamble);
        assertFalse(preamble.isBlank());
        assertTrue(preamble.contains("emotional state"));
        assertTrue(preamble.contains("drives"));
        assertTrue(preamble.contains("beliefs"));
        assertTrue(preamble.contains("goals"));
        assertTrue(preamble.contains("strategies"));
    }

    @Test
    void noSubsystemsEnabled_producesMinimalPreamble() {
        var config = CognitionConfig.none();
        var preamble = CognitivePreambleGenerator.generate(config);
        assertNotNull(preamble);
        assertTrue(preamble.contains("inner life"));
        assertFalse(preamble.contains("drives"));
        assertFalse(preamble.contains("strategies"));
    }

    @Test
    void partialSubsystems_includesOnlyActive() {
        var config = CognitionConfig.none().with("mood", true).with("goals", true);
        var preamble = CognitivePreambleGenerator.generate(config);
        assertTrue(preamble.contains("emotional state"));
        assertTrue(preamble.contains("goals"));
        assertFalse(preamble.contains("strategies"));
        assertFalse(preamble.contains("drives"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitivePreambleGeneratorTest`
Expected: FAIL — class not found

- [ ] **Step 3: Write minimal implementation**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.CognitionConfig;

public final class CognitivePreambleGenerator {

    private CognitivePreambleGenerator() {}

    public static String generate(CognitionConfig config) {
        var sb = new StringBuilder("You have an inner life.");
        if (config.moodEnabled()) {
            sb.append(" Your emotional state colours how you respond — let it shape your tone and choices.");
        }
        if (config.drivesEnabled()) {
            sb.append(" You have motivational drives that pull at you; some are stronger than others right now.");
        }
        if (config.mentalModelEnabled()) {
            sb.append(" You hold beliefs about the people around you, formed from observation — act on them, update them when evidence contradicts.");
        }
        if (config.userModelEnabled()) {
            sb.append(" You build a sense of who each person is from how they behave — use that understanding.");
        }
        if (config.narrativeEnabled()) {
            sb.append(" You carry a personal narrative — significant moments and themes that define who you are becoming.");
        }
        if (config.goalsEnabled()) {
            sb.append(" You have goals that emerged from your motivations — pursue them, reprioritise when circumstances change.");
        }
        if (config.strategyEnabled()) {
            sb.append(" You have learned strategies from past interactions — apply what worked, abandon what didn't.");
        }
        return sb.toString();
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitivePreambleGeneratorTest`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CognitivePreambleGenerator.java blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/CognitivePreambleGeneratorTest.java
git commit -m "feat(#283): CognitivePreambleGenerator — auto-generated cognitive instruction paragraph

Refs casehubio/blocks#283"
```

### Task 2: CognitiveSystemPromptRenderer

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CognitiveSystemPromptRenderer.java`
- Test: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/CognitiveSystemPromptRendererTest.java`

**Interfaces:**
- Consumes: `SystemPromptRenderer` SPI (`io.casehub.eidos.api.SystemPromptRenderer`), `CognitivePreambleGenerator.generate(CognitionConfig)`, `AgentDescriptor` (`.name()`, `.briefing()`, `.constraints()`, `.templates()`), `CognitionConfig`, `ConstraintSeverity.HARD`
- Produces: `RenderedPrompt render(AgentDescriptor, AgentPromptContext)` — implements `SystemPromptRenderer`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.CognitionConfig;
import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class CognitiveSystemPromptRendererTest {

    private static AgentDescriptor descriptor(String name, String briefing,
                                               List<AgentConstraint> constraints) {
        return AgentDescriptor.builder()
                .agentId("test-agent")
                .name(name)
                .briefing(briefing)
                .constraints(constraints)
                .build();
    }

    @Test
    void rendersIdentityAndBriefing() {
        var desc = descriptor("Penelope Pitstop",
                "A glamorous Southern belle. You speak with a Southern drawl.",
                List.of());
        var renderer = new CognitiveSystemPromptRenderer(CognitionConfig.all());
        var result = renderer.render(desc, AgentPromptContext.forFormat(
                SystemPromptRenderer.RenderFormat.MARKDOWN));
        assertTrue(result.content().contains("Penelope Pitstop"));
        assertTrue(result.content().contains("Southern belle"));
        assertEquals(SystemPromptRenderer.RenderFormat.MARKDOWN, result.format());
        assertFalse(result.enriched());
    }

    @Test
    void rendersOnlyHardConstraints() {
        var hard = new AgentConstraint("no-break", "Never break cover",
                Visibility.PUBLIC, ConstraintSeverity.HARD);
        var soft = new AgentConstraint("be-polite", "Always be polite",
                Visibility.PUBLIC, ConstraintSeverity.SOFT);
        var desc = descriptor("Agent", "Identity.", List.of(hard, soft));
        var renderer = new CognitiveSystemPromptRenderer(CognitionConfig.all());
        var result = renderer.render(desc, AgentPromptContext.forFormat(
                SystemPromptRenderer.RenderFormat.MARKDOWN));
        assertTrue(result.content().contains("Never break cover"));
        assertFalse(result.content().contains("Always be polite"));
    }

    @Test
    void primeDirectivesHeading() {
        var hard = new AgentConstraint("no-break", "Never break cover",
                Visibility.PUBLIC, ConstraintSeverity.HARD);
        var desc = descriptor("Agent", "Identity.", List.of(hard));
        var renderer = new CognitiveSystemPromptRenderer(CognitionConfig.all());
        var result = renderer.render(desc, AgentPromptContext.forFormat(
                SystemPromptRenderer.RenderFormat.MARKDOWN));
        assertTrue(result.content().contains("Prime Directives"));
    }

    @Test
    void includesCognitivePreamble() {
        var desc = descriptor("Agent", "Identity.", List.of());
        var renderer = new CognitiveSystemPromptRenderer(CognitionConfig.all());
        var result = renderer.render(desc, AgentPromptContext.forFormat(
                SystemPromptRenderer.RenderFormat.MARKDOWN));
        assertTrue(result.content().contains("inner life"));
        assertTrue(result.content().contains("emotional state"));
    }

    @Test
    void noHardConstraints_noPrimeDirectivesSection() {
        var soft = new AgentConstraint("be-polite", "Always be polite",
                Visibility.PUBLIC, ConstraintSeverity.SOFT);
        var desc = descriptor("Agent", "Identity.", List.of(soft));
        var renderer = new CognitiveSystemPromptRenderer(CognitionConfig.all());
        var result = renderer.render(desc, AgentPromptContext.forFormat(
                SystemPromptRenderer.RenderFormat.MARKDOWN));
        assertFalse(result.content().contains("Prime Directives"));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitiveSystemPromptRendererTest`
Expected: FAIL — class not found

- [ ] **Step 3: Write minimal implementation**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.CognitionConfig;
import io.casehub.eidos.api.*;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.List;

public class CognitiveSystemPromptRenderer implements SystemPromptRenderer {

    private final CognitionConfig config;

    public CognitiveSystemPromptRenderer(CognitionConfig config) {
        this.config = config;
    }

    @Override
    public RenderedPrompt render(AgentDescriptor descriptor, AgentPromptContext context) {
        var sb = new StringBuilder();

        // Identity
        sb.append("# ").append(descriptor.name()).append("\n\n");
        if (descriptor.briefing() != null && !descriptor.briefing().isBlank()) {
            sb.append(descriptor.briefing()).append("\n\n");
        }

        // Templates (voice/style — rendered by eidos descriptor, passed through as-is)
        if (descriptor.templates() != null) {
            for (var tpl : descriptor.templates()) {
                if (tpl.resolvedContent() != null && !tpl.resolvedContent().isBlank()) {
                    sb.append(tpl.resolvedContent()).append("\n\n");
                }
            }
        }

        // Prime Directives (HARD constraints only)
        var hardConstraints = descriptor.constraints() != null
                ? descriptor.constraints().stream()
                    .filter(c -> c.severity() == ConstraintSeverity.HARD)
                    .toList()
                : List.<AgentConstraint>of();
        if (!hardConstraints.isEmpty()) {
            sb.append("## Prime Directives\n\n");
            for (var c : hardConstraints) {
                sb.append("- ").append(c.description()).append("\n");
            }
            sb.append("\n");
        }

        // Cognitive preamble
        sb.append("## Your Mind\n\n");
        sb.append(CognitivePreambleGenerator.generate(config)).append("\n");

        var content = sb.toString().stripTrailing();
        var hash = shortHash(content);
        return new RenderedPrompt(content, RenderFormat.MARKDOWN, hash, hash, false);
    }

    private static String shortHash(String input) {
        try {
            var digest = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(digest.digest(input.getBytes())).substring(0, 16);
        } catch (NoSuchAlgorithmException e) {
            return Integer.toHexString(input.hashCode());
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitiveSystemPromptRendererTest`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CognitiveSystemPromptRenderer.java blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/CognitiveSystemPromptRendererTest.java
git commit -m "feat(#283): CognitiveSystemPromptRenderer — minimal directive with Prime Directives

Identity + voice + HARD constraints + cognitive preamble.
Does NOT render goals, soft constraints, disposition, or dynamic cognitive data.

Refs casehubio/blocks#283"
```

### Task 3: Constraint Severity Filtering in CognitionCore

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java:349-350`
- Test: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/CognitionCoreConstraintFilterTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor.constraints()` → `List<AgentConstraint>`, `ConstraintSeverity.HARD`
- Produces: `CognitionCore.promptSections()` now excludes HARD constraints from `ConstraintPromptSection`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social;

import io.casehub.blocks.speech.PromptContext;
import io.casehub.blocks.speech.PromptSection;
import io.casehub.blocks.agentic.social.prompt.ConstraintPromptSection;
import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class CognitionCoreConstraintFilterTest {

    @Test
    void promptSections_excludesHardConstraints() {
        var hard = new AgentConstraint("no-break", "Never break cover",
                Visibility.PUBLIC, ConstraintSeverity.HARD);
        var soft = new AgentConstraint("be-polite", "Always be polite",
                Visibility.PUBLIC, ConstraintSeverity.SOFT);
        var desc = AgentDescriptor.builder()
                .agentId("test").name("Test")
                .constraints(List.of(hard, soft))
                .build();

        var core = new CognitionCore(CognitionConfig.all(), null, null, null, null, null, null);
        core.updateDescriptor(desc);
        var sections = core.promptSections();

        var constraintSection = sections.stream()
                .filter(s -> s instanceof ConstraintPromptSection)
                .findFirst().orElseThrow();
        var rendered = constraintSection.contribute(PromptContext.DEFAULT);
        assertNotNull(rendered);
        assertTrue(rendered.contains("Always be polite"));
        assertFalse(rendered.contains("Never break cover"));
    }

    @Test
    void promptSections_noSoftConstraints_noConstraintSection() {
        var hard = new AgentConstraint("no-break", "Never break cover",
                Visibility.PUBLIC, ConstraintSeverity.HARD);
        var desc = AgentDescriptor.builder()
                .agentId("test").name("Test")
                .constraints(List.of(hard))
                .build();

        var core = new CognitionCore(CognitionConfig.all(), null, null, null, null, null, null);
        core.updateDescriptor(desc);
        var sections = core.promptSections();

        var constraintSections = sections.stream()
                .filter(s -> s instanceof ConstraintPromptSection)
                .toList();
        assertTrue(constraintSections.isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitionCoreConstraintFilterTest`
Expected: FAIL — HARD constraints still included

- [ ] **Step 3: Modify CognitionCore.promptSections()**

In `CognitionCore.java`, change lines 349-350 from:

```java
if (desc != null && desc.constraints() != null && !desc.constraints().isEmpty())
    sections.add(new ConstraintPromptSection(desc.constraints()));
```

To:

```java
if (desc != null && desc.constraints() != null && !desc.constraints().isEmpty()) {
    var softConstraints = desc.constraints().stream()
            .filter(c -> c.severity() != ConstraintSeverity.HARD)
            .toList();
    if (!softConstraints.isEmpty()) {
        sections.add(new ConstraintPromptSection(softConstraints));
    }
}
```

Add import: `import io.casehub.eidos.api.ConstraintSeverity;`

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitionCoreConstraintFilterTest`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java blocks-core/src/test/java/io/casehub/blocks/agentic/social/CognitionCoreConstraintFilterTest.java
git commit -m "feat(#283): filter HARD constraints from CognitionCore.promptSections()

HARD constraints rendered as Prime Directives in system prompt by
CognitiveSystemPromptRenderer. SOFT constraints rendered in observation
sections by ConstraintPromptSection.

Refs casehubio/blocks#283"
```

## Batch 2: Deduplication (blocks)

### Task 4: PersonalityPromptSection Dedup + DirectiveSection Toggle

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/SocialAvatarCognition.java:104-116`
- Test: `blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/SocialAvatarCognitionDedupTest.java`

**Interfaces:**
- Consumes: `SocialAvatarCognition.buildSections(String agentId, String tenantId)`, `CognitionCore.promptSections()`
- Produces: `buildSections()` no longer adds duplicate `PersonalityPromptSection`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.CognitionConfig;
import io.casehub.blocks.agentic.social.CognitionCore;
import io.casehub.blocks.speech.PromptSection;
import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

class SocialAvatarCognitionDedupTest {

    @Test
    void buildSections_noDuplicatePersonalitySection() {
        var desc = AgentDescriptor.builder()
                .agentId("test").name("Test")
                .disposition(new AgentDisposition("INTJ", "architect", false))
                .build();
        var config = CognitionConfig.all();
        var core = new CognitionCore(config, null, null, null, null, null, null);
        core.updateDescriptor(desc);

        var cog = new SocialAvatarCognition(core, Optional.of(
                new SimpleAgentRegistry(List.of(desc))));
        var sections = cog.buildSections("test", "tenant");

        var personalitySections = sections.stream()
                .filter(s -> s instanceof PersonalityPromptSection)
                .toList();
        assertEquals(1, personalitySections.size(),
                "Expected exactly one PersonalityPromptSection, got " + personalitySections.size());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=SocialAvatarCognitionDedupTest`
Expected: FAIL — 2 PersonalityPromptSections found

- [ ] **Step 3: Modify SocialAvatarCognition.buildSections()**

In `SocialAvatarCognition.java`, change `buildSections()` (lines 104-116) from:

```java
List<PromptSection> buildSections(String agentId, String tenantId) {
    var sections = new ArrayList<PromptSection>();
    if (agentRegistry.isPresent()) {
        agentRegistry.get().findById(agentId, tenantId)
                .ifPresent(desc -> {
                    var profile = desc.disposition() != null
                            ? desc.disposition().dispositionProfile() : null;
                    sections.add(new PersonalityPromptSection(profile));
                });
    }
    sections.addAll(core.promptSections());
    return sections;
}
```

To:

```java
List<PromptSection> buildSections(String agentId, String tenantId) {
    return new ArrayList<>(core.promptSections());
}
```

CognitionCore already adds `PersonalityPromptSection` from the descriptor at line 348. The duplicate from `SocialAvatarCognition` is removed.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=SocialAvatarCognitionDedupTest`
Expected: PASS

- [ ] **Step 5: Run full blocks-core test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/SocialAvatarCognition.java blocks-core/src/test/java/io/casehub/blocks/agentic/social/prompt/SocialAvatarCognitionDedupTest.java
git commit -m "fix(#283): remove duplicate PersonalityPromptSection from SocialAvatarCognition

CognitionCore.promptSections() already provides PersonalityPromptSection.
SocialAvatarCognition.buildSections() now delegates directly.

Refs casehubio/blocks#283"
```

## Batch 3: Wacky-Manor YAML + Seeding (examples)

**Prerequisite:** File issue on casehubio/examples for wacky-manor changes before starting this batch.

### Task 5: Descriptor YAML Rewrite

**Files:**
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml`
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`

**Interfaces:**
- Consumes: Current descriptor YAML structure
- Produces: Briefing text stripped to identity + voice. Goals moved to social-config.yaml with DriveAxis annotations. Character knowledge moved to social-config initial-beliefs.

- [ ] **Step 1: Rewrite briefing text for each character**

For each character in `descriptors-composite.yaml`:
1. Strip behavioral instructions from `briefing` — keep only identity and voice
2. Remove `goals:` section entirely (goals move to social-config.yaml)
3. Keep only `severity: HARD` constraints. Move `severity: SOFT` constraints to social-config.yaml norms
4. Keep `templates` and `disposition` as-is (templates are voice/style, disposition stays in YAML but won't be rendered in directive)

Example — Penelope Pitstop before:
```yaml
briefing: >-
    You are Penelope Pitstop — a glamorous, sweet-natured Southern belle...
    You speak with a Southern drawl... You know Sylvester Sneekly as helpful...
goals:
    - {name: find-diamond, description: "...", priority: PRIMARY, visibility: PUBLIC}
```

After:
```yaml
briefing: >-
    A glamorous, sweet-natured Southern belle and heiress.
    You speak with a Southern drawl and use phrases like
    "Why, how delightful!" and "Oh my stars!"
# goals removed — seeded via social-config.yaml
```

- [ ] **Step 2: Add goal entries to social-config.yaml**

For each character, add a `goals:` section to their social-config entry:

```yaml
penelope-pitstop:
  goals:
    - name: find-diamond
      description: "Find the legendary Doily Diamond hidden somewhere in the manor"
      axis: CURIOSITY
      intensity: 0.8
      formation-reason: character-definition
    - name: solve-puzzles
      description: "Solve the mansion's puzzles and mysteries"
      axis: CURIOSITY
      intensity: 0.6
      formation-reason: character-definition
  initial-beliefs:
    # Add character knowledge moved from briefing
    - {key: sneekly-identity, value: "Sylvester Sneekly is a helpful and charming estate manager"}
  # existing drives, norms, relationships unchanged
```

Map each goal to its most appropriate `DriveAxis`:
- Discovery/exploration goals → `CURIOSITY`
- Achievement/mastery goals → `COMPETENCE`
- Social/relationship goals → `AFFILIATION`
- Independence/control goals → `AUTONOMY`

- [ ] **Step 3: Verify YAML is syntactically valid**

Run: `python3 -c "import yaml; yaml.safe_load(open('wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml')); print('OK')"` and same for social-config.yaml.

- [ ] **Step 4: Commit**

```bash
git add wacky-manor/src/main/resources/META-INF/eidos/descriptors-composite.yaml wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml
git commit -m "feat(#N): rewrite descriptor YAML — strip briefing to identity+voice, move goals to social-config

Goals annotated with DriveAxis for GoalProposalOrchestrator seeding.
Character knowledge moved to initial-beliefs.

Refs casehubio/examples#N"
```

### Task 6: Goal Seeding in ManorCognitiveSeeder

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java` (add goal parsing)
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederGoalTest.java`

**Interfaces:**
- Consumes: `SocialConfig` (new `goals()` method returning goal config list), `GoalProposalOrchestrator.registerGoals(String agentId, String tenantId, List<DriveGoalProposal>)`, `DriveGoalProposal(DriveAxis, String goalName, String goalDescription, String formationReason, double driveIntensity)`
- Produces: `seedGoals(String agentId, SocialConfig config, GoalProposalOrchestrator goals, String tenantId)` — seeds static goals into GoalProposalOrchestrator

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.drive.DriveAxis;
import io.casehub.blocks.agentic.social.goal.*;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class ManorCognitiveSeederGoalTest {

    @Test
    void seedGoals_registersWithOrchestrator() {
        // Create a mock GoalProposalOrchestrator or use a real one with test config
        // Verify that registerGoals is called with correctly mapped DriveGoalProposal list
        var goalConfigs = List.of(
            new ManorGoalConfig("find-diamond", "Find the diamond", "CURIOSITY", 0.8, "character-definition"),
            new ManorGoalConfig("solve-puzzles", "Solve puzzles", "CURIOSITY", 0.6, "character-definition")
        );

        var proposals = ManorCognitiveSeeder.mapGoals(goalConfigs);

        assertEquals(2, proposals.size());
        assertEquals(DriveAxis.CURIOSITY, proposals.get(0).axis());
        assertEquals("find-diamond", proposals.get(0).goalName());
        assertEquals(0.8, proposals.get(0).driveIntensity());
        assertEquals("character-definition", proposals.get(0).formationReason());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorCognitiveSeederGoalTest`
Expected: FAIL — ManorGoalConfig and mapGoals() don't exist

- [ ] **Step 3: Add ManorGoalConfig record and mapGoals() method**

Add to `ManorCognitiveSeeder.java`:

```java
public record ManorGoalConfig(String name, String description, String axis,
                                double intensity, String formationReason) {}

public static List<DriveGoalProposal> mapGoals(List<ManorGoalConfig> configs) {
    return configs.stream()
            .map(c -> new DriveGoalProposal(
                    DriveAxis.valueOf(c.axis()),
                    c.name(),
                    c.description(),
                    c.formationReason(),
                    c.intensity()))
            .toList();
}

public void seedGoals(String agentId, List<ManorGoalConfig> goalConfigs,
                       GoalProposalOrchestrator goals, String tenantId) {
    if (goalConfigs == null || goalConfigs.isEmpty()) return;
    var proposals = mapGoals(goalConfigs);
    goals.registerGoals(agentId, tenantId, proposals);
}
```

Update `ManorSocialConfigLoader` to parse the new `goals:` YAML section and return `List<ManorGoalConfig>`.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorCognitiveSeederGoalTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederGoalTest.java
git commit -m "feat(#N): goal seeding in ManorCognitiveSeeder via DriveGoalProposal

Maps social-config.yaml goals to DriveAxis-annotated DriveGoalProposal
records and registers them via GoalProposalOrchestrator.registerGoals().

Refs casehubio/examples#N"
```

## Batch 4: CharacterCognition Dedup + Integration (examples)

### Task 7: CharacterCognition Deduplication

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`

**Interfaces:**
- Consumes: `CognitiveSystemPromptRenderer`, `CognitionCore.promptSections()`, `GoalProposalOrchestrator.registerGoals()`
- Produces: `CharacterCognition.renderCognitiveSections()` no longer renders constraints or goals directly. `ScenarioOrchestrator` wires `CognitiveSystemPromptRenderer` and goal seeding at bootstrap.

- [ ] **Step 1: Remove constraint rendering from CharacterCognition**

In `CharacterCognition.renderCognitiveSections()`, remove the section that renders constraints directly from `SocialConfig` or `AgentConstraint`. Constraints are now handled by:
- HARD → `CognitiveSystemPromptRenderer` (system prompt, Prime Directives)
- SOFT → `ConstraintPromptSection` (CognitionCore observation sections)

Retain in `renderCognitiveSections()`:
- Character motivations (from social-config drives — free-form)
- Initial beliefs (from social-config)
- Trust perceptions
- Social awareness
- Norms

- [ ] **Step 2: Wire CognitiveSystemPromptRenderer in ScenarioOrchestrator**

In `ScenarioOrchestrator`, the `renderer` field is an injected `SystemPromptRenderer`. With `CognitiveSystemPromptRenderer` as an `@Alternative @Priority(1)` bean in blocks-core on the classpath, CDI automatically resolves it. No wiring changes needed in `ScenarioOrchestrator` — the override is transparent.

Verify: `ScenarioOrchestrator.renderPrompt()` calls `renderer.render(desc, ctx)` — when `CognitiveSystemPromptRenderer` is on the classpath, this produces the minimal directive. When it's not (e.g., in tests without blocks-core), eidos default renders as before.

- [ ] **Step 3: Wire goal seeding in ScenarioOrchestrator bootstrap**

In `ScenarioOrchestrator`'s bootstrap method (where `ManorCognitiveSeeder.seed()` and `seedPeople()` are called), add goal seeding:

```java
// After existing ManorCognitiveSeeder calls
var goalConfigs = socialConfigLoader.loadGoals(agentId);
if (goalConfigs != null) {
    seeder.seedGoals(agentId, goalConfigs, cognitionCore.goals(), ManorConstants.TENANCY_ID);
}
```

- [ ] **Step 4: Set config.directivePrompts to false**

In the wacky-manor cognition config setup, ensure `directivePrompts` is `false` so `CognitionCore.promptSections()` returns unwrapped sections (no `DirectiveSection` wrapping). The cognitive preamble in the system prompt replaces per-section directives.

- [ ] **Step 5: Run full wacky-manor test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git commit -m "feat(#N): CharacterCognition dedup + goal seeding wiring

Remove constraint rendering from CharacterCognition (now split:
HARD in system prompt as Prime Directives, SOFT in CognitionCore).
Wire goal seeding at bootstrap. Set directivePrompts=false.

Refs casehubio/examples#N"
```

### Task 8: Integration Verification

**Files:**
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/DirectiveMinimalIntegrationTest.java`

**Interfaces:**
- Consumes: All prior tasks
- Produces: Integration test verifying the full prompt structure

- [ ] **Step 1: Write integration test**

```java
package io.casehub.examples.manor.agent;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.blocks.agentic.social.prompt.CognitiveSystemPromptRenderer;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class DirectiveMinimalIntegrationTest {

    @Inject
    SystemPromptRenderer renderer;

    @Test
    void rendererIsCognitiveOverride() {
        assertInstanceOf(CognitiveSystemPromptRenderer.class, renderer);
    }

    @Test
    void systemPromptIsMinimal() {
        // Load a test descriptor and render
        // Verify: contains identity, voice, Prime Directives for HARD constraints
        // Verify: does NOT contain goals, soft constraints, disposition details
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=DirectiveMinimalIntegrationTest`
Expected: PASS

- [ ] **Step 3: Run full test suite across both repos**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml`
Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: All green

- [ ] **Step 4: Commit**

```bash
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/DirectiveMinimalIntegrationTest.java
git commit -m "test(#N): integration test — verify CognitiveSystemPromptRenderer override

Refs casehubio/examples#N"
```

## References

- [2026-09-16-directive-minimal-architecture-design.md] — design spec this plan implements
- [CognitionCore.java:344-369] — promptSections() method
- [SocialAvatarCognition.java:104-116] — buildSections() with PersonalityPromptSection duplication
- [CognitionConfig.java] — subsystem feature flags
- [SystemPromptRenderer.java] — SPI interface with RenderedPrompt record
- [AgentConstraint.java] — constraint record with ConstraintSeverity enum
- [DriveGoalProposal.java] — goal proposal record with DriveAxis
- [GoalProposalOrchestrator.java:102-107] — registerGoals() method
- [ConstraintPromptSection.java] — current constraint rendering
- [DirectiveSection.java] — per-section directive wrapping
- [ManorCognitiveSeeder.java] — existing belief/relationship seeder
- casehubio/blocks#283 — focal issue
- specs/issue-63-fix-cdi-config-beans/decisions.md — 8 decisions (3 revised)
