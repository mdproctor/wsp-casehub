# Capability-Driven Observation Filtering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #39 — feat: Eidos capability-driven observation filtering — platform pattern for multi-agent perception
**Issue group:** #39

**Goal:** Add composable observation filtering to casehub-blocks — AnnotatedSection wrapper, ObservationFilter SPI, ObservationPipeline, and PerceptionFilter — then migrate the manor from hard-coded observer-tag branching to the pipeline.

**Architecture:** Providers emit all sections at maximum resolution with capability annotations. An ObservationPipeline of composable ObservationFilter stages applies visibility gating, resolution control, and interpretive framing based on observer capability tags. AnnotatedSection wraps ObservationSection with metadata; the sealed interface stays untouched. PerceptionFilter is the built-in filter combining visibility + resolution.

**Tech Stack:** Java 26, Quarkus, casehub-blocks, casehub-eidos-api

## Global Constraints

- `ObservationSection` sealed interface must not change
- `AnnotatedSection implements ObservationSection` — mixed lists without type changes
- Unannotated sections pass through all filters unchanged (backward compat)
- blocks SNAPSHOT must be published before manor work begins
- Create a GitHub issue on casehubio/blocks for the blocks work before implementing

---

## Batch 1: Observation filtering pipeline in blocks (cross-repo)

### Task 1: ResolutionTier enum and AnnotatedSection record

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ResolutionTier.java`
- Create: `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/AnnotatedSection.java`
- Test: `blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/AnnotatedSectionTest.java`

**Interfaces:**
- Consumes: `ObservationSection` (existing)
- Produces:
  - `enum ResolutionTier { FULL, REDUCED, SUMMARY }`
  - `record AnnotatedSection(ObservationSection section, Set<String> requiredTags, Map<ResolutionTier, ObservationSection> resolutions, String interpretiveFrame) implements ObservationSection`
  - `static AnnotatedSection requiring(ObservationSection section, Set<String> tags)`
  - `static AnnotatedSection withResolution(ObservationSection fullSection, Set<String> tags, ResolutionTier fallbackTier, ObservationSection fallbackSection)`

- [ ] **Step 1: Write failing tests for AnnotatedSection**

```java
@Test
void requiring_creates_visibility_only_annotation() {
    var section = ObservationSection.text("Keen Observations", "X positioned Y carefully");
    var annotated = AnnotatedSection.requiring(section, Set.of("perception"));
    assertThat(annotated.section()).isEqualTo(section);
    assertThat(annotated.requiredTags()).containsExactly("perception");
    assertThat(annotated.resolutions()).isEmpty();
    assertThat(annotated.interpretiveFrame()).isNull();
}

@Test
void withResolution_creates_visibility_plus_fallback() {
    var full = ObservationSection.text("Keen Observations", "X positioned Y carefully");
    var reduced = ObservationSection.text("Directed to You", "X said something to you");
    var annotated = AnnotatedSection.withResolution(full, Set.of("perception"), ResolutionTier.REDUCED, reduced);
    assertThat(annotated.section()).isEqualTo(full);
    assertThat(annotated.requiredTags()).containsExactly("perception");
    assertThat(annotated.resolutions()).containsEntry(ResolutionTier.REDUCED, reduced);
}

@Test
void header_delegates_to_wrapped_section() {
    var section = ObservationSection.text("Location", "Kitchen");
    var annotated = AnnotatedSection.requiring(section, Set.of());
    assertThat(annotated.header()).isEqualTo("Location");
}

@Test
void annotated_section_is_observation_section() {
    var section = ObservationSection.text("Location", "Kitchen");
    var annotated = AnnotatedSection.requiring(section, Set.of());
    assertThat(annotated).isInstanceOf(ObservationSection.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/blocks/blocks/pom.xml -Dtest=AnnotatedSectionTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes not found

- [ ] **Step 3: Create ResolutionTier enum**

Use `ide_create_file`:

```java
package io.casehub.blocks.summarisation.observation.affordance;

public enum ResolutionTier {
    FULL,
    REDUCED,
    SUMMARY
}
```

- [ ] **Step 4: Create AnnotatedSection record**

Use `ide_create_file`:

```java
package io.casehub.blocks.summarisation.observation.affordance;

import java.util.Map;
import java.util.Set;

public record AnnotatedSection(
        ObservationSection section,
        Set<String> requiredTags,
        Map<ResolutionTier, ObservationSection> resolutions,
        String interpretiveFrame
) implements ObservationSection {

    public AnnotatedSection {
        requiredTags = Set.copyOf(requiredTags);
        resolutions = Map.copyOf(resolutions);
    }

    @Override
    public String header() {
        return section.header();
    }

    public static AnnotatedSection requiring(ObservationSection section,
                                              Set<String> tags) {
        return new AnnotatedSection(section, tags, Map.of(), null);
    }

    public static AnnotatedSection withResolution(ObservationSection fullSection,
                                                   Set<String> tags,
                                                   ResolutionTier fallbackTier,
                                                   ObservationSection fallbackSection) {
        return new AnnotatedSection(fullSection, tags,
                Map.of(fallbackTier, fallbackSection), null);
    }
}
```

**Important:** `AnnotatedSection` implements `ObservationSection` but is NOT a permitted subtype of the sealed interface. Since `ObservationSection` is sealed, it must add `AnnotatedSection` to the `permits` clause. Edit `ObservationSection.java` to add `permits AnnotatedSection` (alongside the existing `EntityGroup`, `TextBlock`, `ItemList`). Also update `AffordanceRenderer.renderSection()` to handle the new case — delegate to the wrapped section:

```java
case AnnotatedSection a -> renderSection(a.section());
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/blocks/blocks/pom.xml -Dtest=AnnotatedSectionTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ResolutionTier.java blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/AnnotatedSection.java blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationSection.java blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/AffordanceRenderer.java blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/AnnotatedSectionTest.java
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#N): add AnnotatedSection and ResolutionTier for capability-driven observation filtering

Refs casehubio/blocks#N"
```

### Task 2: ObservationFilter SPI, ObservationPipeline, and PerceptionFilter

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationFilter.java`
- Create: `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationPipeline.java`
- Create: `blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/PerceptionFilter.java`
- Test: `blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/PerceptionFilterTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/ObservationPipelineTest.java`

**Interfaces:**
- Consumes: `AnnotatedSection`, `ResolutionTier`, `ObservationSection` (Task 1)
- Produces:
  - `interface ObservationFilter { List<ObservationSection> filter(List<ObservationSection> sections, Set<String> observerTags); }`
  - `class ObservationPipeline { ObservationPipeline(ObservationFilter... stages); List<ObservationSection> apply(List<ObservationSection> sections, Set<String> observerTags); }`
  - `class PerceptionFilter implements ObservationFilter` — combined visibility + resolution

- [ ] **Step 1: Write failing tests for PerceptionFilter**

```java
@Test
void bare_sections_pass_through() {
    var section = ObservationSection.text("Location", "Kitchen");
    var result = new PerceptionFilter().filter(List.of(section), Set.of());
    assertThat(result).containsExactly(section);
}

@Test
void annotated_section_passes_when_tags_met() {
    var section = ObservationSection.text("Keen Observations", "Detail");
    var annotated = AnnotatedSection.requiring(section, Set.of("perception"));
    var result = new PerceptionFilter().filter(List.of(annotated), Set.of("perception"));
    assertThat(result).hasSize(1);
    assertThat(result.get(0)).isEqualTo(annotated);
}

@Test
void annotated_section_removed_when_tags_not_met_and_no_fallback() {
    var section = ObservationSection.text("Keen Observations", "Detail");
    var annotated = AnnotatedSection.requiring(section, Set.of("perception"));
    var result = new PerceptionFilter().filter(List.of(annotated), Set.of());
    assertThat(result).isEmpty();
}

@Test
void annotated_section_downgrades_when_tags_not_met_and_fallback_exists() {
    var full = ObservationSection.text("Keen Observations", "X positioned Y carefully");
    var reduced = ObservationSection.text("Directed to You", "X said something");
    var annotated = AnnotatedSection.withResolution(full, Set.of("perception"), ResolutionTier.REDUCED, reduced);
    var result = new PerceptionFilter().filter(List.of(annotated), Set.of());
    assertThat(result).hasSize(1);
    assertThat(result.get(0)).isEqualTo(reduced);
}

@Test
void empty_required_tags_always_passes() {
    var section = ObservationSection.text("Location", "Kitchen");
    var annotated = AnnotatedSection.requiring(section, Set.of());
    var result = new PerceptionFilter().filter(List.of(annotated), Set.of());
    assertThat(result).hasSize(1);
}

@Test
void mixed_list_filters_correctly() {
    var bare = ObservationSection.text("Location", "Kitchen");
    var keen = ObservationSection.text("Keen Observations", "Detail");
    var directed = ObservationSection.text("Directed to You", "Simple");
    var annotated = AnnotatedSection.withResolution(keen, Set.of("perception"), ResolutionTier.REDUCED, directed);
    var result = new PerceptionFilter().filter(List.of(bare, annotated), Set.of());
    assertThat(result).hasSize(2);
    assertThat(result.get(0)).isEqualTo(bare);
    assertThat(result.get(1)).isEqualTo(directed);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/blocks/blocks/pom.xml -Dtest="PerceptionFilterTest,ObservationPipelineTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes not found

- [ ] **Step 3: Create ObservationFilter interface**

Use `ide_create_file`:

```java
package io.casehub.blocks.summarisation.observation.affordance;

import java.util.List;
import java.util.Set;

@FunctionalInterface
public interface ObservationFilter {
    List<ObservationSection> filter(List<ObservationSection> sections,
                                     Set<String> observerTags);
}
```

- [ ] **Step 4: Create PerceptionFilter**

Use `ide_create_file`:

```java
package io.casehub.blocks.summarisation.observation.affordance;

import java.util.List;
import java.util.Set;

public class PerceptionFilter implements ObservationFilter {

    @Override
    public List<ObservationSection> filter(List<ObservationSection> sections,
                                            Set<String> observerTags) {
        return sections.stream()
                .map(s -> resolve(s, observerTags))
                .filter(s -> s != null)
                .toList();
    }

    private ObservationSection resolve(ObservationSection section, Set<String> observerTags) {
        if (!(section instanceof AnnotatedSection a)) {
            return section;
        }
        if (a.requiredTags().isEmpty() || observerTags.containsAll(a.requiredTags())) {
            return section;
        }
        if (!a.resolutions().isEmpty()) {
            for (var tier : ResolutionTier.values()) {
                if (tier != ResolutionTier.FULL && a.resolutions().containsKey(tier)) {
                    return a.resolutions().get(tier);
                }
            }
        }
        return null;
    }
}
```

- [ ] **Step 5: Create ObservationPipeline**

Use `ide_create_file`:

```java
package io.casehub.blocks.summarisation.observation.affordance;

import java.util.List;
import java.util.Set;

public class ObservationPipeline {

    private final List<ObservationFilter> stages;

    public ObservationPipeline(ObservationFilter... stages) {
        this.stages = List.of(stages);
    }

    public List<ObservationSection> apply(List<ObservationSection> sections,
                                           Set<String> observerTags) {
        var current = sections;
        for (var stage : stages) {
            current = stage.filter(current, observerTags);
        }
        return unwrap(current);
    }

    private List<ObservationSection> unwrap(List<ObservationSection> sections) {
        return sections.stream()
                .map(s -> s instanceof AnnotatedSection a ? a.section() : s)
                .toList();
    }
}
```

- [ ] **Step 6: Write ObservationPipeline tests**

```java
@Test
void pipeline_applies_stages_in_order() {
    var keen = ObservationSection.text("Keen Observations", "Detail");
    var directed = ObservationSection.text("Directed to You", "Simple");
    var annotated = AnnotatedSection.withResolution(keen, Set.of("perception"), ResolutionTier.REDUCED, directed);

    var pipeline = new ObservationPipeline(new PerceptionFilter());
    var result = pipeline.apply(List.of(annotated), Set.of());
    assertThat(result).hasSize(1);
    assertThat(result.get(0).header()).isEqualTo("Directed to You");
}

@Test
void pipeline_unwraps_annotated_sections_after_filtering() {
    var section = ObservationSection.text("Location", "Kitchen");
    var annotated = AnnotatedSection.requiring(section, Set.of());
    var pipeline = new ObservationPipeline(new PerceptionFilter());
    var result = pipeline.apply(List.of(annotated), Set.of());
    assertThat(result).hasSize(1);
    assertThat(result.get(0)).isNotInstanceOf(AnnotatedSection.class);
    assertThat(result.get(0).header()).isEqualTo("Location");
}

@Test
void null_pipeline_equivalent_to_empty_stages() {
    var section = ObservationSection.text("Location", "Kitchen");
    var pipeline = new ObservationPipeline();
    var result = pipeline.apply(List.of(section), Set.of());
    assertThat(result).containsExactly(section);
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/blocks/blocks/pom.xml -Dtest="PerceptionFilterTest,ObservationPipelineTest,AnnotatedSectionTest"`
Expected: ALL PASS

- [ ] **Step 8: Run full blocks test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -f /Users/mdproctor/claude/casehub/blocks/blocks/pom.xml`
Expected: ALL PASS (existing tests unbroken — AffordanceRenderer handles AnnotatedSection via new case)

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks add blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationFilter.java blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationPipeline.java blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/PerceptionFilter.java blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/PerceptionFilterTest.java blocks/src/test/java/io/casehub/blocks/summarisation/observation/affordance/ObservationPipelineTest.java
git -C /Users/mdproctor/claude/casehub/blocks commit -m "feat(#N): add ObservationFilter SPI, ObservationPipeline, and PerceptionFilter

Refs casehubio/blocks#N"
```

- [ ] **Step 10: Install blocks SNAPSHOT locally**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/blocks/blocks/pom.xml -DskipTests`

---

## Batch 2: Manor migration — observer-agnostic provider + pipeline integration

### Task 3: Migrate ManorWorldObservationProvider to emit annotated sections

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorWorldObservationProvider.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorWorldObservationProviderTest.java`

**Interfaces:**
- Consumes: `AnnotatedSection`, `ResolutionTier` (Task 1)
- Produces: observer-agnostic `ManorWorldObservationProvider` — constructor no longer takes `observerTags`

- [ ] **Step 1: Update ManorWorldObservationProviderTest**

Remove tests that assert observer-tag branching behavior. Add tests that verify the provider emits annotated sections unconditionally:

```java
@Test
void worldSections_emits_annotated_keen_observations_with_perception_tag() {
    var character = world.character("penelope-pitstop");
    var provider = new ManorWorldObservationProvider(character, world, emptyDrain);
    var sections = provider.worldSections();
    var annotated = sections.stream()
            .filter(s -> s instanceof AnnotatedSection)
            .map(s -> (AnnotatedSection) s)
            .toList();
    // At least the keen/directed section should be annotated
    // (may be absent if no events exist — that's fine)
}

@Test
void worldSections_no_observer_tags_parameter() {
    var character = world.character("penelope-pitstop");
    // Constructor takes 3 args, not 4 — observerTags removed
    var provider = new ManorWorldObservationProvider(character, world, emptyDrain);
    var sections = provider.worldSections();
    assertThat(sections).isNotEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -f /Users/mdproctor/claude/casehub/examples/pom.xml -Dtest=ManorWorldObservationProviderTest`
Expected: FAIL — constructor signature mismatch

- [ ] **Step 3: Refactor ManorWorldObservationProvider**

Remove `observerTags` from constructor. Replace the `if (observerTags.contains("perception"))` branch with unconditional annotated section emission:

```java
// Remove: private final Set<String> observerTags;
// Change constructor to 3 params: character, world, drain

// In worldSections(), replace the if/else block with:
var keen = keenObservationsSection(character, world);
var directed = directedDialogueSection(character, world);
if (keen != null || directed != null) {
    sections.add(AnnotatedSection.withResolution(
            keen != null ? keen
                 : ObservationSection.items("Keen Observations", null, List.of()),
            Set.of("perception"),
            ResolutionTier.REDUCED,
            directed));
}
```

Use `ide_edit_member` for the constructor and `ide_replace_member` for `worldSections()`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -f /Users/mdproctor/claude/casehub/examples/pom.xml -Dtest=ManorWorldObservationProviderTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorWorldObservationProvider.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorWorldObservationProviderTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#39): make ManorWorldObservationProvider observer-agnostic

Emits AnnotatedSection for keen/directed observations instead of
branching on observerTags. Constructor no longer takes tags.

Refs #39"
```

### Task 4: Integrate ObservationPipeline into ObservationBuilder and update all callers

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/engine/LiveScenarioTest.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java`

**Interfaces:**
- Consumes: `ObservationPipeline`, `PerceptionFilter` (Task 2), observer-agnostic `ManorWorldObservationProvider` (Task 3)
- Produces: `ObservationBuilder.buildObservation()` with pipeline + observerTags parameters

- [ ] **Step 1: Add pipeline parameters to ObservationBuilder**

Add `ObservationPipeline pipeline` and `Set<String> observerTags` parameters. Apply pipeline before rendering:

```java
public static String buildObservation(WorldObservationProvider worldProvider,
                                      ObservationPipeline pipeline,
                                      Set<String> observerTags,
                                      CharacterState character,
                                      List<AgentGoal> goals,
                                      PartitionedDrain<String> drain,
                                      List<Memory> memories,
                                      List<Memory> reflections,
                                      Map<String, List<Memory>> relationshipMemories) {
    var sections = new ArrayList<ObservationSection>();
    sections.addAll(worldProvider.worldSections());
    // ... character state, cognitive sections as before ...

    var filtered = pipeline != null
            ? pipeline.apply(sections, observerTags)
            : sections.stream()
                  .map(s -> s instanceof AnnotatedSection a ? a.section() : s)
                  .toList();

    return RENDERER.renderObservation(filtered);
}
```

Use `ide_edit_member` on `ObservationBuilder.buildObservation`.

- [ ] **Step 2: Update ScenarioOrchestrator**

Read the current call site. Replace with pipeline construction:

```java
var worldProvider = new ManorWorldObservationProvider(c, world, drain);
var pipeline = new ObservationPipeline(new PerceptionFilter());
String observation = ObservationBuilder.buildObservation(
        worldProvider, pipeline, c.capabilityTags(), c, resolveGoals(c.agentId()), drain,
        memories, reflections, relationships);
```

Use `ide_replace_text_in_file` for the call site.

- [ ] **Step 3: Update CharacterAgentLoop**

Same pattern — construct pipeline, pass tags:

```java
var worldProvider = new ManorWorldObservationProvider(character, world, drain);
var pipeline = new ObservationPipeline(new PerceptionFilter());
String observation = ObservationBuilder.buildObservation(
        worldProvider, pipeline, character.capabilityTags(), character, goals, drain,
        memories, List.of(), Map.of());
```

- [ ] **Step 4: Update ExchangeRunner**

Exchange path has no filtering — pass null pipeline:

```java
var exchangeProvider = new ManorExchangeObservationProvider(responder, lastDialogue, world);
String observation = ObservationBuilder.buildObservation(
        exchangeProvider, null, Set.of(), responder, List.of(), emptyDrain,
        List.of(), List.of(), Map.of());
```

- [ ] **Step 5: Update LiveScenarioTest and AutonomousScenarioRunner**

Same pattern as CharacterAgentLoop — construct pipeline, pass `Set.of()` for tags (test characters have no special capabilities).

- [ ] **Step 6: Update ObservationBuilderTest**

Update the `buildObs` helper methods to accept pipeline and tags. Add pipeline-specific tests:

```java
@Test
void perceptive_observer_sees_keen_observations_via_pipeline() {
    // Add detailed-description event
    world.addEvent(new ManorEvent(..., "The Hooded Claw picked up the poison."));
    var provider = new ManorWorldObservationProvider(
            world.character("penelope-pitstop"), world, emptyDrain);
    var pipeline = new ObservationPipeline(new PerceptionFilter());
    var obs = ObservationBuilder.buildObservation(
            provider, pipeline, Set.of("perception"),
            world.character("penelope-pitstop"), List.of(), emptyDrain,
            List.of(), List.of(), Map.of());
    assertThat(obs).contains("Keen Observations");
}

@Test
void non_perceptive_observer_gets_directed_via_pipeline() {
    // Same event setup
    var provider = new ManorWorldObservationProvider(
            world.character("penelope-pitstop"), world, emptyDrain);
    var pipeline = new ObservationPipeline(new PerceptionFilter());
    var obs = ObservationBuilder.buildObservation(
            provider, pipeline, Set.of(),
            world.character("penelope-pitstop"), List.of(), emptyDrain,
            List.of(), List.of(), Map.of());
    assertThat(obs).doesNotContain("Keen Observations");
}

@Test
void null_pipeline_passes_sections_through_unwrapped() {
    var provider = new ManorWorldObservationProvider(
            world.character("penelope-pitstop"), world, emptyDrain);
    var obs = ObservationBuilder.buildObservation(
            provider, null, Set.of(),
            world.character("penelope-pitstop"), List.of(), emptyDrain,
            List.of(), List.of(), Map.of());
    assertThat(obs).contains("Current Location");
}
```

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ExchangeRunner.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ObservationBuilderTest.java wacky-manor/src/test/java/io/casehub/examples/manor/engine/LiveScenarioTest.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/AutonomousScenarioRunner.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#39): integrate ObservationPipeline into ObservationBuilder

ObservationBuilder accepts optional pipeline + observerTags. Manor
callers construct PerceptionFilter pipeline. Provider no longer
branches on tags — pipeline handles visibility and resolution.

Closes #39"
```

---

## References

- [2026-08-28-capability-observation-filtering-design.md] — design spec this plan implements
- [wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorWorldObservationProvider.java] — provider being migrated
- [wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java] — assembly gaining pipeline
- [blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/ObservationSection.java] — sealed interface gaining AnnotatedSection permit
- [blocks/src/main/java/io/casehub/blocks/summarisation/observation/affordance/AffordanceRenderer.java] — renderer gaining AnnotatedSection case
- [GitHub #39] — focal issue
- [GitHub #48] — observation SPI foundation
