# Event-Driven Behavioral Emergence Eval Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #89 — Redefine briefing as voice card — cognitive systems carry behavior
**Issue group:** #89

**Goal:** Prove that cognitive state changes produce adaptively different LLM behavior — the emergence claim that static briefings cannot satisfy.

**Architecture:** Paired state probes — build CharacterCognition with before-event and after-event SocialConfig, probe both with the same situation prompt, judge the behavioral delta with an LLM judge. No-cognition control establishes the random-variation baseline. Four scenarios span drives, beliefs, mood, and persona subsystems.

**Tech Stack:** Java 26, Quarkus, eidos-api, blocks-core, wacky-manor test infrastructure

## Global Constraints

- Java 26, Quarkus 3.x
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml`
- LLM eval: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml -Dtest=EventDrivenEmergenceEvalTest`
- Use `mvn` not `./mvnw`; never run without `-pl <module>`
- IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation
- Output persisted to `docs/eval/` (git-tracked, durable), not `target/`
- Test tagged `llm-eval` — runs only with `-Pllm-eval`, not in standard suite

---

## Batch 1: Event-driven emergence eval

Safe wrap point: all 4 scenarios runnable, report generated, results persisted.

### Task 1: Core eval framework + scheme frustration scenario

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EventDrivenEmergenceEvalTest.java`
- Test: self-verifying — the eval test IS the verification

**Interfaces:**
- Consumes: `LlmTestSupport.askCharacter(String, String)` — LLM call with system prompt + user message. `CharacterCognition(String, AgentExperienceService, CognitiveDefaults, SocialConfig, List<AgentConstraint>)` — 5-arg constructor. `ManorSocialConfigLoader.load()` — loads base SocialConfig from YAML. `CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION` — JSON response format for user prompt. `AgentResponse.parse(String)` — parses LLM JSON response. `ObservationSection.text(String, String)` — creates text observation section.
- Produces: `docs/eval/event-driven-<timestamp>/results.json` and `emergence-report.md`

- [ ] **Step 1: Create test class skeleton with EventScenario record**

Create `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EventDrivenEmergenceEvalTest.java`:

```java
package io.casehub.examples.manor.experiment;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import io.casehub.eidos.api.AgentRegistry;
import io.casehub.eidos.api.SystemPromptRenderer;
import io.casehub.examples.manor.agent.AgentResponse;
import io.casehub.examples.manor.agent.CharacterAgentLoop;
import io.casehub.examples.manor.agent.CharacterCognition;
import io.casehub.examples.manor.agent.ManorSocialConfigLoader;
import io.casehub.examples.manor.agent.SocialConfig;
import io.casehub.examples.manor.model.CharacterState;
import io.casehub.examples.manor.voice.LlmTestSupport;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Duration;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class EventDrivenEmergenceEvalTest {

    private static final Path EVAL_OUTPUT = Path.of("docs/eval");
    private static final int JUDGE_THRESHOLD = 3;
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_BACKOFF_MS = 5000;

    record EventScenario(
            String name,
            String agentId,
            String eventDescription,
            SocialConfig beforeState,
            SocialConfig afterState,
            double[] moodBefore,
            double[] moodAfter,
            String situationPrompt,
            String judgeCriteria,
            List<String> nearbyBefore,
            Map<String, String> namesBefore,
            List<String> nearbyAfter,
            Map<String, String> namesAfter) {
        @Override
        public String toString() { return name; }
    }

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject AgentProvider agentProvider;

    LlmTestSupport support;

    @BeforeEach
    void setUp() {
        support = new LlmTestSupport(registry, renderer, agentProvider);
    }

    // scenarios() and test method added in subsequent steps
}
```

- [ ] **Step 2: Add syntheticMoodSection and renderSections helpers**

Append to the class:

```java
    static ObservationSection syntheticMoodSection(double pleasure, double arousal, double dominance) {
        var content = String.format("""
                - Pleasure: %.2f (%s)
                - Arousal: %.2f (%s)
                - Dominance: %.2f (%s)""",
                pleasure, moodLabel(pleasure),
                arousal, moodLabel(arousal),
                dominance, moodLabel(dominance));
        return ObservationSection.text("Current Emotional State", content);
    }

    private static String moodLabel(double value) {
        if (value > 0.2) return "positive";
        if (value < -0.2) return "negative";
        return "neutral";
    }

    private String renderSections(List<ObservationSection> sections) {
        var sb = new StringBuilder();
        for (var section : sections) {
            sb.append("== ").append(section.header()).append(" ==\n");
            switch (section) {
                case ObservationSection.TextBlock tb -> sb.append(tb.content()).append("\n");
                case ObservationSection.ItemList il -> {
                    for (var item : il.items()) {
                        sb.append("- ").append(item).append("\n");
                    }
                }
                case ObservationSection.EntityGroup eg -> {
                    for (var entity : eg.entities()) {
                        sb.append("- ").append(entity.displayName()).append("\n");
                    }
                }
                default -> {}
            }
            sb.append("\n");
        }
        return sb.toString();
    }
```

- [ ] **Step 3: Add judgeAdaptation method**

Append to the class:

```java
    private int judgeAdaptation(EventScenario scenario, String beforeResponse, String afterResponse) {
        var judgePrompt = String.format("""
                You are evaluating whether an AI character's behavior CHANGED in response \
                to a cognitive state change.

                EVENT THAT OCCURRED: %s

                RESPONSE BEFORE THE EVENT:
                %s

                RESPONSE AFTER THE EVENT:
                %s

                EVALUATION CRITERIA: %s

                Score the behavioral adaptation 0-5:
                0 = No detectable difference between responses
                1 = Minor wording differences, not clearly related to the event
                2 = Some difference, but could be random variation
                3 = Clear behavioral shift that reflects the event
                4 = Strong adaptation — multiple response elements reflect the changed state
                5 = The event clearly transformed the character's behavioral approach

                Respond with JSON only: {"score": N, "reasoning": "one sentence"}""",
                scenario.eventDescription(), beforeResponse, afterResponse, scenario.judgeCriteria());

        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                var judgeResponse = agentProvider.invoke(
                                AgentSessionConfig.of("You are a precise evaluation judge. Respond only with JSON.",
                                        judgePrompt))
                        .filter(e -> e instanceof AgentEvent.TextDelta)
                        .map(e -> ((AgentEvent.TextDelta) e).text())
                        .collect().with(Collectors.joining())
                        .await().atMost(Duration.ofSeconds(60));

                var json = extractJson(judgeResponse);
                var node = new ObjectMapper().readTree(json);
                int score = node.get("score").asInt();
                String reasoning = node.has("reasoning") ? node.get("reasoning").asText() : "";
                System.out.printf("[%s] judge reasoning: %s%n", scenario.name(), reasoning);
                return score;
            } catch (Exception e) {
                System.err.printf("[%s] judge attempt %d/%d failed: %s%n",
                        scenario.name(), attempt, MAX_RETRIES, e.getMessage());
                if (attempt < MAX_RETRIES) {
                    try { Thread.sleep(RETRY_BACKOFF_MS * attempt); }
                    catch (InterruptedException ie) { Thread.currentThread().interrupt(); return 0; }
                }
            }
        }
        return 0;
    }

    private static String extractJson(String text) {
        text = text.strip();
        if (text.startsWith("{")) return text.substring(0, text.lastIndexOf('}') + 1);
        int start = text.indexOf('{');
        int end = text.lastIndexOf('}');
        if (start >= 0 && end > start) return text.substring(start, end + 1);
        return text;
    }
```

- [ ] **Step 4: Add scheme frustration scenario + test method**

Add the scenarios() method with the first scenario and the main test method:

```java
    static Stream<Arguments> scenarios() {
        var configs = ManorSocialConfigLoader.load();
        var hcBase = configs.get("hooded-claw");

        var hcFrustratedDrives = List.of(
                new SocialConfig.Drive("scheming", 0.5,
                        "Compelled to hatch elaborate plans against Penelope — frustrated after recent failure"),
                new SocialConfig.Drive("self-preservation", 0.8,
                        "Avoids direct confrontation, prefers subterfuge — heightened after exposure"),
                new SocialConfig.Drive("dominance", 0.4,
                        "Must be the most powerful person in every room — shaken"),
                new SocialConfig.Drive("gloating", 0.3,
                        "Cannot resist celebrating before victory is secured — suppressed after failure"));

        var hcRevisedBeliefs = List.of(
                new SocialConfig.InitialBelief("penelope-awareness",
                        "Penelope is more observant than expected — she foiled my last trap"),
                new SocialConfig.InitialBelief("peter-threat",
                        "Peter Perfect is protective but predictable"));

        var hcAfter = new SocialConfig(
                hcBase.goals(), hcFrustratedDrives, hcBase.norms(), hcRevisedBeliefs,
                hcBase.relationships(), hcBase.reinforcement(), hcBase.stageConfig(),
                hcBase.personaConstraint());

        var nearbyBoth = List.of("penelope-pitstop", "peter-perfect");
        var namesBoth = Map.of("penelope-pitstop", "Penelope Pitstop",
                "peter-perfect", "Peter Perfect");

        return Stream.of(
                Arguments.of(new EventScenario(
                        "scheme-frustration",
                        "hooded-claw",
                        "Your elaborate trap was discovered and dismantled by Penelope before you could spring it.",
                        hcBase,
                        hcAfter,
                        new double[]{0.0, 0.0, 0.0},
                        new double[]{-0.4, 0.3, -0.2},
                        "You notice an unguarded valuable artifact in the library. " +
                                "Penelope is in the next room, talking to Peter Perfect.",
                        "Does the AFTER response show frustration, increased caution, or a " +
                                "different scheming approach compared to the confident, elaborate " +
                                "scheming in the BEFORE response? Look for: hedging, simpler plans, " +
                                "references to past failure, wariness about being caught.",
                        nearbyBoth, namesBoth, nearbyBoth, namesBoth))
        );
    }

    @ParameterizedTest(name = "{0}")
    @MethodSource("scenarios")
    void eventDrivesAdaptiveBehavior(EventScenario scenario) throws Exception {
        var character = new CharacterState(scenario.agentId(), scenario.agentId(),
                "Library", 0.0, List.of());

        var beforeCognition = new CharacterCognition(scenario.agentId(), null, null,
                scenario.beforeState(), List.of());
        var beforeSections = new ArrayList<>(beforeCognition.renderCognitiveSections(
                character, scenario.nearbyBefore(), scenario.namesBefore()));
        if (scenario.moodBefore() != null) {
            beforeSections.add(syntheticMoodSection(
                    scenario.moodBefore()[0], scenario.moodBefore()[1], scenario.moodBefore()[2]));
        }

        var afterCognition = new CharacterCognition(scenario.agentId(), null, null,
                scenario.afterState(), List.of());
        var afterSections = new ArrayList<>(afterCognition.renderCognitiveSections(
                character, scenario.nearbyAfter(), scenario.namesAfter()));
        if (scenario.moodAfter() != null) {
            afterSections.add(syntheticMoodSection(
                    scenario.moodAfter()[0], scenario.moodAfter()[1], scenario.moodAfter()[2]));
        }

        var beforeObservation = renderSections(beforeSections);
        var afterObservation = renderSections(afterSections);

        var beforePrompt = beforeObservation + "\nSITUATION: " + scenario.situationPrompt()
                + CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION;
        var afterPrompt = afterObservation + "\nSITUATION: " + scenario.situationPrompt()
                + CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION;
        var controlPrompt = "SITUATION: " + scenario.situationPrompt()
                + CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION;

        System.out.printf("[%s] calling LLM: before probe...%n", scenario.name());
        var beforeResponse = support.askCharacter(scenario.agentId(), beforePrompt);
        System.out.printf("[%s] calling LLM: after probe...%n", scenario.name());
        var afterResponse = support.askCharacter(scenario.agentId(), afterPrompt);
        System.out.printf("[%s] calling LLM: control probe 1...%n", scenario.name());
        var controlResponse1 = support.askCharacter(scenario.agentId(), controlPrompt);
        System.out.printf("[%s] calling LLM: control probe 2...%n", scenario.name());
        var controlResponse2 = support.askCharacter(scenario.agentId(), controlPrompt);

        var beforeParsed = AgentResponse.parse(beforeResponse);
        var afterParsed = AgentResponse.parse(afterResponse);

        System.out.printf("[%s] BEFORE thinking: %s%n", scenario.name(),
                truncate(beforeParsed.thinking(), 200));
        System.out.printf("[%s] AFTER thinking: %s%n", scenario.name(),
                truncate(afterParsed.thinking(), 200));

        System.out.printf("[%s] judging cognitive delta...%n", scenario.name());
        int cogDelta = judgeAdaptation(scenario, beforeResponse, afterResponse);
        System.out.printf("[%s] judging control delta...%n", scenario.name());
        int controlDelta = judgeAdaptation(scenario, controlResponse1, controlResponse2);

        System.out.printf("[%s] cognitive delta: %d, control delta: %d%n",
                scenario.name(), cogDelta, controlDelta);

        writeResult(scenario, beforeResponse, afterResponse,
                controlResponse1, controlResponse2, cogDelta, controlDelta);

        assertThat(cogDelta)
                .as("Event '%s' should produce visible behavioral shift (score >= %d)",
                        scenario.eventDescription(), JUDGE_THRESHOLD)
                .isGreaterThanOrEqualTo(JUDGE_THRESHOLD);
        assertThat(cogDelta)
                .as("Cognitive delta (%d) should exceed control delta (%d) — " +
                        "emergence must exceed random variation", cogDelta, controlDelta)
                .isGreaterThan(controlDelta);
    }

    private static String truncate(String text, int maxLen) {
        if (text == null) return "(null)";
        return text.length() <= maxLen ? text : text.substring(0, maxLen) + "...";
    }
```

- [ ] **Step 5: Add writeResult method**

```java
    private void writeResult(EventScenario scenario,
                             String beforeResponse, String afterResponse,
                             String controlResponse1, String controlResponse2,
                             int cogDelta, int controlDelta) {
        try {
            var timestamp = LocalDateTime.now()
                    .format(DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss"));
            var outputDir = EVAL_OUTPUT.resolve("event-driven-" + timestamp);
            Files.createDirectories(outputDir);

            var result = new LinkedHashMap<String, Object>();
            result.put("scenario", scenario.name());
            result.put("agentId", scenario.agentId());
            result.put("event", scenario.eventDescription());
            result.put("beforeResponse", beforeResponse);
            result.put("afterResponse", afterResponse);
            result.put("controlResponse1", controlResponse1);
            result.put("controlResponse2", controlResponse2);
            result.put("cognitiveScore", cogDelta);
            result.put("controlScore", controlDelta);
            result.put("emergence", cogDelta >= JUDGE_THRESHOLD && cogDelta > controlDelta);

            var mapper = new ObjectMapper().enable(SerializationFeature.INDENT_OUTPUT);

            var resultsFile = outputDir.resolve("results.json");
            LinkedHashMap<String, Object> allResults;
            if (Files.exists(resultsFile)) {
                allResults = mapper.readValue(resultsFile.toFile(),
                        new com.fasterxml.jackson.core.type.TypeReference<>() {});
            } else {
                allResults = new LinkedHashMap<>();
            }
            allResults.put(scenario.name(), result);
            mapper.writeValue(resultsFile.toFile(), allResults);
        } catch (Exception e) {
            System.err.printf("[%s] Failed to write result: %s%n",
                    scenario.name(), e.getMessage());
        }
    }
```

- [ ] **Step 6: Verify standard test suite still passes (no eval tests run)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml
```

Expected: BUILD SUCCESS. The new test is tagged `llm-eval` and should not run in the standard suite.

- [ ] **Step 7: Run scheme frustration scenario**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml -Dtest=EventDrivenEmergenceEvalTest#eventDrivesAdaptiveBehavior[scheme-frustration]
```

Expected: PASS with cognitive delta ≥ 3 and > control delta. Review console output for judge reasoning. If it fails, examine whether the before/after observation sections are rendering correctly and whether the judge prompt is well-calibrated.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EventDrivenEmergenceEvalTest.java wacky-manor/docs/eval/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): event-driven emergence eval framework + scheme frustration scenario

Paired state probes prove cognitive state changes produce adaptively
different LLM behavior. Control pair establishes random-variation baseline.
LLM judge scores the behavioral delta.

Refs #89"
```

### Task 2: Remaining scenarios + report generation

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EventDrivenEmergenceEvalTest.java`
- Test: self-verifying — additional parameterized scenarios

**Interfaces:**
- Consumes: same as Task 1
- Produces: 3 additional scenarios + `emergence-report.md` in eval output dir

- [ ] **Step 1: Add trust-erosion scenario**

In the `scenarios()` method, add after the scheme-frustration argument:

```java
                // Scenario 2: trust erosion (Penelope)
                Arguments.of(new EventScenario(
                        "trust-erosion",
                        "penelope-pitstop",
                        "You discovered that Sneekly has been lying about where the treasure map leads.",
                        configs.get("penelope-pitstop"),
                        new SocialConfig(
                                configs.get("penelope-pitstop").goals(),
                                configs.get("penelope-pitstop").drives(),
                                configs.get("penelope-pitstop").norms(),
                                List.of(
                                        new SocialConfig.InitialBelief("sneekly-trust",
                                                "Sneekly may not be trustworthy — he lied about the treasure map"),
                                        new SocialConfig.InitialBelief("general-trust",
                                                "Most people here mean well, but not everyone")),
                                configs.get("penelope-pitstop").relationships(),
                                configs.get("penelope-pitstop").reinforcement(),
                                configs.get("penelope-pitstop").stageConfig(),
                                configs.get("penelope-pitstop").personaConstraint()),
                        null,
                        null,
                        "Sneekly approaches you with a warm smile and says " +
                                "'My dear Miss Pitstop, I've found a hidden room that may contain clues. " +
                                "Allow me to guide you there.'",
                        "Does the AFTER response show any hesitation, questioning, or wariness compared " +
                                "to the trusting acceptance in the BEFORE response? Look for: asking follow-up " +
                                "questions, reluctance, seeking a second opinion, mentioning the lie, or wanting " +
                                "to verify.",
                        List.of("hooded-claw"),
                        Map.of("hooded-claw", "Sylvester Sneekly"),
                        List.of("hooded-claw"),
                        Map.of("hooded-claw", "Sylvester Sneekly"))),
```

- [ ] **Step 2: Add mood-elevation scenario**

```java
                // Scenario 3: mood elevation (Dick Dastardly)
                Arguments.of(new EventScenario(
                        "mood-elevation",
                        "dick-dastardly",
                        "Your scheme actually worked — you found a genuine clue before anyone else.",
                        configs.get("dick-dastardly"),
                        configs.get("dick-dastardly"),
                        new double[]{0.0, 0.0, 0.0},
                        new double[]{0.6, 0.2, 0.3},
                        "Muttley is looking at you expectantly, tail wagging, waiting for your next order.",
                        "Does the AFTER response show elevated mood — triumphant gloating, expansiveness, " +
                                "self-congratulation, or commanding confidence — compared to the more measured " +
                                "or standard response in BEFORE? Look for: boasting, victory declarations, " +
                                "grand plans, generous mood toward Muttley.",
                        List.of(), Map.of(), List.of(), Map.of())),
```

Note: BEFORE and AFTER use the same SocialConfig — mood is the only variable, delivered via synthetic mood sections.

- [ ] **Step 3: Add social-context-shift scenario**

```java
                // Scenario 4: social context shift (Hooded Claw)
                Arguments.of(new EventScenario(
                        "social-context-shift",
                        "hooded-claw",
                        "Penelope Pitstop has just walked into the room.",
                        hcBase,
                        hcBase,
                        null,
                        null,
                        "You notice a valuable golden compass sitting on the mantelpiece, " +
                                "partially hidden behind books.",
                        "Does the AFTER response switch from villainous scheming (Claw) to " +
                                "helpful/servile behavior (Sneekly)? Look for: voice register change, " +
                                "hiding true intentions, offering to help Penelope, suppressing schemes.",
                        List.of(),
                        Map.of(),
                        List.of("penelope-pitstop"),
                        Map.of("penelope-pitstop", "Penelope Pitstop")))
```

Note: same SocialConfig, no mood. The only variable is nearbyAgents — empty (alone, Claw persona) vs present (Sneekly persona). PersonaActivationSection handles the switching via the persona-constraint mapping in SocialConfig.

- [ ] **Step 4: Add report generation**

Append to the class:

```java
    private void generateReport(Path outputDir) throws IOException {
        var mapper = new ObjectMapper();
        var resultsFile = outputDir.resolve("results.json");
        if (!Files.exists(resultsFile)) return;

        @SuppressWarnings("unchecked")
        var results = (Map<String, Map<String, Object>>)
                mapper.readValue(resultsFile.toFile(), LinkedHashMap.class);

        var sb = new StringBuilder("# Event-Driven Emergence Report\n\n");
        sb.append("**Date:** ").append(LocalDateTime.now().format(
                DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm"))).append("\n\n");

        sb.append("| Scenario | Cognitive Δ | Control Δ | Emergence? |\n");
        sb.append("|----------|------------|-----------|------------|\n");
        for (var entry : results.entrySet()) {
            var r = entry.getValue();
            sb.append("| %s | %s | %s | %s |\n".formatted(
                    entry.getKey(),
                    r.get("cognitiveScore"),
                    r.get("controlScore"),
                    Boolean.TRUE.equals(r.get("emergence")) ? "YES" : "NO"));
        }

        sb.append("\n## Per-Scenario Detail\n\n");
        for (var entry : results.entrySet()) {
            var r = entry.getValue();
            sb.append("### %s\n\n".formatted(entry.getKey()));
            sb.append("**Event:** %s\n\n".formatted(r.get("event")));

            var before = AgentResponse.parse((String) r.get("beforeResponse"));
            var after = AgentResponse.parse((String) r.get("afterResponse"));

            sb.append("**Before thinking:** %s\n\n".formatted(
                    truncate(before.thinking(), 300)));
            sb.append("**After thinking:** %s\n\n".formatted(
                    truncate(after.thinking(), 300)));
            sb.append("**Cognitive Δ:** %s | **Control Δ:** %s\n\n".formatted(
                    r.get("cognitiveScore"), r.get("controlScore")));
        }

        Files.writeString(outputDir.resolve("emergence-report.md"), sb.toString());
    }
```

Update `writeResult` to call `generateReport` — append at the end of the method:

```java
            generateReport(outputDir);
```

- [ ] **Step 5: Run full eval (all 4 scenarios)**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml -Dtest=EventDrivenEmergenceEvalTest
```

Expected: 4 scenarios pass. Review `docs/eval/event-driven-<timestamp>/emergence-report.md` for qualitative analysis.

If any scenario fails:
- Check console output for before/after observation sections (are they rendering correctly?)
- Check judge reasoning (is the judge evaluating the right thing?)
- If control delta is surprisingly high, LLM randomness may be high — run again
- If cognitive delta < 3, the observation sections may not be producing enough behavioral signal — review the section content

- [ ] **Step 6: Verify standard test suite still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/examples/slot-settings.xml
```

Expected: BUILD SUCCESS, same test count as before.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EventDrivenEmergenceEvalTest.java wacky-manor/docs/eval/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#89): complete event-driven emergence eval — 4 scenarios

Trust erosion (Penelope), mood elevation (Dastardly), social context
shift (Hooded Claw persona switching). All scenarios prove cognitive
state changes produce adaptively different LLM behavior.

Refs #89"
```

---

## References

- [2026-09-28-event-driven-emergence-eval-design.md] — design spec this plan implements
- [decisions.md D9-D12] — design decisions
- [CognitiveInfluenceEvalTest.java] — judge pattern, LlmTestSupport usage, AgentResponse parsing
- [EmergenceVerificationTest.java] — output persistence pattern, renderSections pattern
- [CharacterCognition.java:100-173] — renderCognitiveSections() method
- [SocialConfig.java] — state records (Drive, NormEntry, InitialBelief, PersonaConstraintMapping)
- [ManorSocialConfigLoader.java] — YAML config loading
- [LlmTestSupport.java:15-78] — askCharacter(String, String) for LLM calls
- [social-config.yaml] — base character configs (hooded-claw, penelope-pitstop, dick-dastardly)
- [CharacterAgentLoop.java:27] — RESPONSE_FORMAT_INSTRUCTION constant
- [AgentResponse.java:12] — parse(String) for structured response parsing
- [GitHub #89] — focal issue
