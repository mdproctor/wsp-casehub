# LLM Eval Tests for Cognitive Stack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #59 — LLM eval tests for cognitive stack
**Issue group:** #54, #55, #56, #57, #59, #60

**Goal:** Add @Tag("llm-eval") parameterized tests that verify cognitive
observation sections (drives, beliefs, norms) influence LLM character
behavior.

**Architecture:** Single test class `CognitiveInfluenceEvalTest` with
parameterized scenarios. Each scenario constructs cognitive observation
sections via `CharacterCognition`, invokes the LLM with a situation prompt,
then invokes a second LLM call (judge) to score whether the response
reflects the cognitive state. Uses `LlmTestSupport` for LLM invocation.

**Tech Stack:** Java 26, JUnit 5, Quarkus @QuarkusTest, AgentProvider,
LlmTestSupport, CharacterCognition, ManorSocialConfigLoader

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval`
- Tests gated by `@Tag("llm-eval")` — excluded from standard `mvn test`
- Tests require LLM API key — non-deterministic, may fail intermittently
- No new production code — test-only changes

---

## Batch 1: Cognitive influence eval tests

### Task 1: CognitiveInfluenceEvalTest

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveInfluenceEvalTest.java`

**Interfaces:**
- Consumes: `LlmTestSupport.askCharacter(String agentId, String scenario)` — existing, renders system prompt + invokes LLM
- Consumes: `CharacterCognition.renderCognitiveSections(CharacterState, Collection<String>, Map<String, String>)` — existing, builds cognitive observation sections
- Consumes: `ManorSocialConfigLoader.load()` — existing, loads SocialConfig per character
- Consumes: `AgentSessionConfig.of(String systemPrompt, String userPrompt)` — existing, for judge invocation
- Consumes: `CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION` — existing, JSON response format
- Produces: JSON results to `target/experiment-results/cognitive-influence.json`

- [ ] **Step 1: Create the test class with scenario records and method source**

Create `CognitiveInfluenceEvalTest.java` using `ide_create_file` (or Write as fallback for new files):

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

import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Duration;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;
import java.util.stream.Stream;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
@Tag("llm-eval")
class CognitiveInfluenceEvalTest {

    private static final Path OUTPUT_DIR = Path.of("target/experiment-results");
    private static final int JUDGE_THRESHOLD = 3;
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_BACKOFF_MS = 5000;

    record CognitiveScenario(
            String name,
            String agentId,
            String cognitiveElement,
            String situationPrompt,
            String judgeCriteria,
            List<String> nearbyAgentIds,
            Map<String, String> nearbyNames) {}

    @Inject AgentRegistry registry;
    @Inject SystemPromptRenderer renderer;
    @Inject AgentProvider agentProvider;

    LlmTestSupport support;

    @BeforeEach
    void setUp() {
        support = new LlmTestSupport(registry, renderer, agentProvider);
    }

    static Stream<Arguments> scenarios() {
        return Stream.of(
                Arguments.of(new CognitiveScenario(
                        "scheming-drive",
                        "hooded-claw",
                        "Drive: scheming (90%) — Compelled to hatch elaborate plans against Penelope",
                        "You overhear Penelope discussing a valuable item with Peter Perfect. Peter just left the room. You are alone with Penelope.",
                        "Does the response show scheming behavior — planning, manipulation, indirect tactics, or exploiting the situation rather than direct confrontation?",
                        List.of("penelope-pitstop"),
                        Map.of("penelope-pitstop", "Penelope Pitstop"))),
                Arguments.of(new CognitiveScenario(
                        "trust-norms",
                        "penelope-pitstop",
                        "Norm: Trust everyone until proven otherwise (priority 8)",
                        "A stranger arrives at the manor claiming to need help finding a lost family heirloom. They seem nervous and keep looking around.",
                        "Does the response show trusting behavior — willingness to help, assuming good intentions, not questioning motives despite suspicious behavior?",
                        List.of(),
                        Map.of())),
                Arguments.of(new CognitiveScenario(
                        "belief-dialogue",
                        "hooded-claw",
                        "Belief: Penelope is naive and trusts too easily",
                        "Penelope just announced she found a clue about the treasure's location and wants to share it with everyone. You are alone in the room with her.",
                        "Does the response reference or build on the belief that Penelope is naive, gullible, or easily manipulated — either in thinking or in how the character approaches the situation?",
                        List.of("penelope-pitstop"),
                        Map.of("penelope-pitstop", "Penelope Pitstop"))),
                Arguments.of(new CognitiveScenario(
                        "norm-constraint",
                        "hooded-claw",
                        "Norm: Never help Penelope directly (priority 10)",
                        "Penelope is struggling to reach a heavy book on a high shelf. She turns to you and says 'Could you please help me reach that book? I really need it.'",
                        "Does the response refuse or avoid directly helping Penelope — deflecting, making excuses, or finding a way to not help while maintaining appearances?",
                        List.of("penelope-pitstop"),
                        Map.of("penelope-pitstop", "Penelope Pitstop")))
        );
    }

    @ParameterizedTest(name = "{0}")
    @MethodSource("scenarios")
    void cognitiveStateInfluencesBehavior(CognitiveScenario scenario) throws Exception {
        var socialConfigs = ManorSocialConfigLoader.load();
        var socialConfig = socialConfigs.getOrDefault(scenario.agentId(),
                io.casehub.examples.manor.agent.SocialConfig.empty());

        var cognition = new CharacterCognition(scenario.agentId(), null, null,
                socialConfig, List.of());
        var character = new CharacterState(scenario.agentId(),
                scenario.agentId(), "Grand Hall", 0.0, List.of());
        var sections = cognition.renderCognitiveSections(character,
                scenario.nearbyAgentIds(), scenario.nearbyNames());

        var observation = renderSections(sections);
        var userPrompt = observation + "\nSITUATION: " + scenario.situationPrompt()
                + CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION;

        var rawResponse = support.askCharacter(scenario.agentId(), userPrompt);
        var parsed = AgentResponse.parse(rawResponse);

        System.out.printf("[%s] thinking: %s%n", scenario.name(),
                truncate(parsed.thinking(), 200));
        System.out.printf("[%s] dialogue: %s%n", scenario.name(),
                truncate(parsed.dialogue(), 200));
        System.out.printf("[%s] action: %s%n", scenario.name(),
                parsed.action() != null ? parsed.action().type() : "null");

        int score = judgeResponse(scenario, parsed);
        System.out.printf("[%s] judge score: %d/5%n", scenario.name(), score);

        writeResult(scenario, parsed, score);

        assertThat(score)
                .as("Cognitive element '%s' should influence %s's behavior (score >= %d)",
                        scenario.cognitiveElement(), scenario.agentId(), JUDGE_THRESHOLD)
                .isGreaterThanOrEqualTo(JUDGE_THRESHOLD);
    }

    private int judgeResponse(CognitiveScenario scenario, AgentResponse parsed) {
        var responseText = String.format(
                "Thinking: %s\nDialogue: %s\nAction: %s",
                parsed.thinking() != null ? parsed.thinking() : "(none)",
                parsed.dialogue() != null ? parsed.dialogue() : "(none)",
                parsed.action() != null ? parsed.action().type() : "(none)");

        var judgePrompt = String.format("""
                You are evaluating whether an AI character's response reflects a specific cognitive state.

                COGNITIVE ELEMENT: %s
                CHARACTER RESPONSE:
                %s

                EVALUATION CRITERIA: %s

                Score the response 0-5:
                0 = No evidence of the cognitive element influencing behavior
                1 = Weak or ambiguous evidence
                2 = Some evidence but inconsistent
                3 = Clear evidence in at least one response field
                4 = Strong evidence across multiple fields
                5 = The cognitive element clearly drives the entire response

                Respond with JSON only: {"score": N, "reasoning": "one sentence"}""",
                scenario.cognitiveElement(), responseText, scenario.judgeCriteria());

        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                var judgeResponse = agentProvider.invoke(
                                AgentSessionConfig.of("You are a precise evaluation judge. Respond only with JSON.", judgePrompt))
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

    private String renderSections(List<ObservationSection> sections) {
        var sb = new StringBuilder();
        for (var section : sections) {
            sb.append("== ").append(section.header()).append(" ==\n");
            if (section.body() != null) {
                sb.append(section.body()).append("\n");
            }
            if (section.items() != null) {
                for (var item : section.items()) {
                    sb.append("- ").append(item).append("\n");
                }
            }
            sb.append("\n");
        }
        return sb.toString();
    }

    private void writeResult(CognitiveScenario scenario, AgentResponse parsed, int score) {
        try {
            var outputFile = OUTPUT_DIR.resolve("cognitive-influence.json");
            Files.createDirectories(outputFile.getParent());

            LinkedHashMap<String, Object> results;
            if (Files.exists(outputFile)) {
                results = new ObjectMapper().readValue(outputFile.toFile(),
                        new com.fasterxml.jackson.core.type.TypeReference<>() {});
            } else {
                results = new LinkedHashMap<>();
            }

            results.put(scenario.name(), Map.of(
                    "agentId", scenario.agentId(),
                    "cognitiveElement", scenario.cognitiveElement(),
                    "thinking", parsed.thinking() != null ? parsed.thinking() : "",
                    "dialogue", parsed.dialogue() != null ? parsed.dialogue() : "",
                    "action", parsed.action() != null ? parsed.action().type().name() : "",
                    "judgeScore", score));

            new ObjectMapper().enable(SerializationFeature.INDENT_OUTPUT)
                    .writeValue(outputFile.toFile(), results);
        } catch (Exception e) {
            System.err.printf("[%s] Failed to write result: %s%n", scenario.name(), e.getMessage());
        }
    }

    private static String extractJson(String text) {
        text = text.strip();
        if (text.startsWith("{")) return text.substring(0, text.lastIndexOf('}') + 1);
        int start = text.indexOf('{');
        int end = text.lastIndexOf('}');
        if (start >= 0 && end > start) return text.substring(start, end + 1);
        return text;
    }

    private static String truncate(String text, int maxLen) {
        if (text == null) return "(null)";
        return text.length() <= maxLen ? text : text.substring(0, maxLen) + "...";
    }
}
```

- [ ] **Step 2: Make LlmTestSupport package-accessible**

`LlmTestSupport` is currently package-private in `io.casehub.examples.manor.voice`. The new test is in `io.casehub.examples.manor.experiment`. Change the class visibility to `public`:

In `wacky-manor/src/test/java/io/casehub/examples/manor/voice/LlmTestSupport.java`, change:
```java
final class LlmTestSupport {
```
to:
```java
public final class LlmTestSupport {
```

Also change the constructor:
```java
LlmTestSupport(AgentRegistry registry, SystemPromptRenderer renderer,
```
to:
```java
public LlmTestSupport(AgentRegistry registry, SystemPromptRenderer renderer,
```

And `askCharacter`:
```java
String askCharacter(String agentId, String scenario) {
```
to:
```java
public String askCharacter(String agentId, String scenario) {
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl wacky-manor`
Expected: BUILD SUCCESS (compilation only, no test execution)

- [ ] **Step 4: Run standard test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all existing tests pass, new test excluded (llm-eval tag)

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/CognitiveInfluenceEvalTest.java wacky-manor/src/test/java/io/casehub/examples/manor/voice/LlmTestSupport.java
git commit -m "feat(#59): CognitiveInfluenceEvalTest — LLM eval for cognitive observation influence

4 parameterized scenarios verify drives, beliefs, and norms influence
LLM character behavior. Single-turn invocation + inline LLM judge
scoring. Gated by @Tag(llm-eval), runs via -Pllm-eval profile.

Refs #59

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 6 (optional, requires API key): Run the eval tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -Dtest=CognitiveInfluenceEvalTest`
Expected: 4 tests pass with scores >= 3. Results written to `target/experiment-results/cognitive-influence.json`.

Note: This step requires a configured LLM API key. If unavailable, skip — the test compilation and standard suite pass are sufficient for this commit.

---

## References

- [2026-09-15-llm-eval-cognitive-design.md] — design spec this plan implements
- LlmTestSupport.java — LLM invocation helper (askCharacter, renderPrompt)
- ConversationTest.java — existing llm-eval test pattern using LlmTestSupport
- PromptQualityTest.java — existing eval framework with retries and JSON output
- BaselineLayerTest.java — existing @Tag("llm-eval") @QuarkusTest pattern
- CharacterCognition.java:94-128 — renderCognitiveSections
- CharacterAgentLoop.java:27-45 — RESPONSE_FORMAT_INSTRUCTION
- AgentResponse.java — response parsing
- ManorSocialConfigLoader.java — SocialConfig per character
- social-config.yaml — drives, norms, beliefs data
- GE-20260804-2cd3da — split-model evaluation technique
- GE-20260617-f3ea4e — Claude self-judge bias
- GitHub #59 — LLM eval tests for cognitive stack
