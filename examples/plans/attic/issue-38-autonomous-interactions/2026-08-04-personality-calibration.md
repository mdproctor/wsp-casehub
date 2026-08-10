# Personality Calibration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #12 — Briefing-framework coherence validation
**Issue group:** #12, #13, #14 (epic #11)

**Goal:** Build diagnostic and calibration tools for the Eidos personality framework — a coherence judge, a briefing-richness experiment matrix, and three disposition-strengthening mechanisms.

**Architecture:** Three sequential deliverables. #12 adds `BriefingCoherenceJudge` to `casehub-eidos-eval` (cross-repo). #13 adds briefing-mode experiment infrastructure to wacky-manor's `PromptQualityTest`. #14 adds three integration mechanisms measured via the same experiment harness. All share a `DominantFunction` utility and `FunctionFormatConstraint` text bank.

**Tech Stack:** Java 26, Quarkus, LangChain4j, Eidos API/eval, JUnit 5

## Global Constraints

- All LLM-calling tests tagged `@Tag("llm-eval")`, excluded from default build
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -pl wacky-manor`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
- LLM eval: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval`
- Cross-repo eidos work builds from `/Users/mdproctor/claude/casehub/eidos`
- Use `mvn` not `./mvnw`

---

### Task 1: DominantFunction + FunctionFormatConstraint utilities

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/experiment/DominantFunction.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/experiment/FunctionFormatConstraint.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/DominantFunctionTest.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/FunctionFormatConstraintTest.java`

**Interfaces:**
- Consumes: `AgentDisposition` from `casehub-eidos-api`, `DispositionValue` (term + weight)
- Produces:
  - `DominantFunction.of(AgentDisposition) → Optional<String>` — returns dominant function term
  - `FunctionFormatConstraint.forDominant(String) → String` — format constraint text
  - `FunctionFormatConstraint.cognitiveApproach(AgentDisposition) → String` — observation directive text
  - `FunctionFormatConstraint.reasoningInstruction(AgentDisposition) → String` — schema reinforcement text
  - `FunctionFormatConstraint.thinkingDescription(AgentDisposition) → String` — parameterized thinking field description

- [ ] **Step 1: Write DominantFunction failing tests**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionValue;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Optional;
import static org.assertj.core.api.Assertions.assertThat;

class DominantFunctionTest {

    @Test
    void returns_dominant_from_profile() {
        var disposition = AgentDisposition.builder()
            .dispositionProfile(List.of(
                DispositionValue.of("te", 0.40),
                DispositionValue.of("ni", 0.30),
                DispositionValue.of("se", 0.15),
                DispositionValue.of("fi", 0.10)))
            .build();
        assertThat(DominantFunction.of(disposition)).isEqualTo(Optional.of("te"));
    }

    @Test
    void returns_empty_for_no_profile() {
        var disposition = AgentDisposition.builder()
            .socialOrient("collaborative")
            .build();
        assertThat(DominantFunction.of(disposition)).isEmpty();
    }

    @Test
    void returns_empty_for_null_disposition() {
        assertThat(DominantFunction.of(null)).isEmpty();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=DominantFunctionTest`
Expected: compilation error — `DominantFunction` does not exist

- [ ] **Step 3: Implement DominantFunction**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentDisposition;
import java.util.Optional;

public final class DominantFunction {
    private DominantFunction() {}

    public static Optional<String> of(AgentDisposition disposition) {
        if (disposition == null || disposition.dispositionProfile().isEmpty()) {
            return Optional.empty();
        }
        return Optional.of(disposition.dispositionProfile().getFirst().term());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=DominantFunctionTest`
Expected: 3 tests PASS

- [ ] **Step 5: Write FunctionFormatConstraint failing tests**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionValue;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class FunctionFormatConstraintTest {

    private final AgentDisposition teDisposition = AgentDisposition.builder()
        .dispositionProfile(List.of(
            DispositionValue.of("te", 0.40),
            DispositionValue.of("ni", 0.30)))
        .build();

    private final AgentDisposition noProfile = AgentDisposition.builder()
        .socialOrient("collaborative")
        .build();

    @Test
    void forDominant_returns_te_constraint() {
        assertThat(FunctionFormatConstraint.forDominant("te"))
            .contains("numbered action plans");
    }

    @Test
    void forDominant_returns_null_for_unknown() {
        assertThat(FunctionFormatConstraint.forDominant("xx")).isNull();
    }

    @Test
    void cognitiveApproach_returns_directive_for_te() {
        String result = FunctionFormatConstraint.cognitiveApproach(teDisposition);
        assertThat(result).contains("systematic");
    }

    @Test
    void cognitiveApproach_returns_null_for_no_profile() {
        assertThat(FunctionFormatConstraint.cognitiveApproach(noProfile)).isNull();
    }

    @Test
    void reasoningInstruction_returns_text_for_te() {
        String result = FunctionFormatConstraint.reasoningInstruction(teDisposition);
        assertThat(result).contains("systematic analysis");
    }

    @Test
    void thinkingDescription_returns_parameterized_for_te() {
        String result = FunctionFormatConstraint.thinkingDescription(teDisposition);
        assertThat(result).contains("systematic analysis");
    }

    @Test
    void all_eight_functions_have_format_constraints() {
        for (String fn : List.of("te", "ti", "fe", "fi", "se", "si", "ni", "ne")) {
            assertThat(FunctionFormatConstraint.forDominant(fn))
                .as("Missing format constraint for %s", fn)
                .isNotNull();
        }
    }
}
```

- [ ] **Step 6: Implement FunctionFormatConstraint**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentDisposition;
import java.util.Map;

public final class FunctionFormatConstraint {
    private FunctionFormatConstraint() {}

    private static final Map<String, String> FORMAT_CONSTRAINTS = Map.of(
        "te", "Structure your responses as numbered action plans with explicit criteria for success",
        "ti", "Present your reasoning as a logical chain: premise → analysis → conclusion",
        "fe", "Frame your responses around how actions affect the group — who benefits, who is harmed, what's the relational impact",
        "fi", "Ground your responses in your core values — state what matters to you and why before deciding",
        "se", "Focus on what's immediately actionable — concrete objects, present dangers, physical options",
        "si", "Reference what you've seen before, established procedures, and proven approaches",
        "ni", "Converge to a single strategic insight — one prediction, one pattern, one conclusion",
        "ne", "Explore multiple possibilities before converging — what-ifs, alternatives, connections"
    );

    private static final Map<String, String> COGNITIVE_APPROACHES = Map.of(
        "te", "As a systematic strategic thinker, prioritise structured execution over creative exploration. Evaluate each option against your objectives. Organise your approach into clear steps.",
        "ti", "As an analytical thinker, build your reasoning from first principles. Seek internal consistency and precision. Question assumptions before acting.",
        "fe", "As a group-aware communicator, consider how your actions affect everyone present. Seek consensus where possible. Frame decisions in terms of relational impact.",
        "fi", "As a value-driven individual, check your actions against your deeply held principles. Choose what feels authentically right over what others expect.",
        "se", "As a hands-on pragmatist, focus on what is immediately in front of you. Act on concrete details and present realities. Deliver practical solutions.",
        "si", "As a methodical practitioner, draw on established procedures and past experience. Follow proven approaches step by step.",
        "ni", "As a pattern-focused strategist, look for the deeper meaning beneath surface events. Converge on a single insight or prediction.",
        "ne", "As an idea-generating explorer, brainstorm multiple possibilities and connections. Open up alternatives before narrowing down."
    );

    private static final Map<String, String> REASONING_INSTRUCTIONS = Map.of(
        "te", "Think through your response using systematic analysis — what options exist, which is optimal, why.",
        "ti", "Think through your response using logical analysis from first principles — what is the underlying structure, what follows logically.",
        "fe", "Think through your response by considering group dynamics — who is affected, what maintains harmony, what serves the collective.",
        "fi", "Think through your response by checking against your core values — what feels right, what aligns with who you are.",
        "se", "Think through your response by scanning the immediate environment — what is actionable right now, what concrete details matter.",
        "si", "Think through your response by recalling precedent — what has worked before, what established procedure applies.",
        "ni", "Think through your response by converging on one deep insight — what is the single pattern or prediction that explains this situation.",
        "ne", "Think through your response by exploring possibilities — what connections, alternatives, and what-ifs emerge from this situation."
    );

    private static final Map<String, String> THINKING_DESCRIPTIONS = Map.of(
        "te", "(describe your systematic analysis — what options exist, which is optimal, why; not shown to others)",
        "ti", "(reason from first principles — what is the logical structure, what follows; not shown to others)",
        "fe", "(consider group impact — who is affected, what maintains harmony; not shown to others)",
        "fi", "(check against your core values — what feels right, what aligns with who you are; not shown to others)",
        "se", "(scan the immediate environment — what is actionable, what concrete details matter; not shown to others)",
        "si", "(recall precedent — what has worked before, what established procedure applies; not shown to others)",
        "ni", "(converge on one insight — what is the single pattern or prediction here; not shown to others)",
        "ne", "(explore possibilities — what connections and alternatives emerge; not shown to others)"
    );

    public static String forDominant(String function) {
        return FORMAT_CONSTRAINTS.get(function.toLowerCase());
    }

    public static String cognitiveApproach(AgentDisposition disposition) {
        return DominantFunction.of(disposition)
            .map(fn -> COGNITIVE_APPROACHES.get(fn.toLowerCase()))
            .orElse(null);
    }

    public static String reasoningInstruction(AgentDisposition disposition) {
        return DominantFunction.of(disposition)
            .map(fn -> REASONING_INSTRUCTIONS.get(fn.toLowerCase()))
            .orElse(null);
    }

    public static String thinkingDescription(AgentDisposition disposition) {
        return DominantFunction.of(disposition)
            .map(fn -> THINKING_DESCRIPTIONS.get(fn.toLowerCase()))
            .orElse(null);
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest="DominantFunctionTest,FunctionFormatConstraintTest"`
Expected: all PASS

- [ ] **Step 8: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/experiment/DominantFunction.java wacky-manor/src/main/java/io/casehub/examples/manor/experiment/FunctionFormatConstraint.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/DominantFunctionTest.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/FunctionFormatConstraintTest.java
git commit -m "feat(#12): DominantFunction + FunctionFormatConstraint utilities

Foundation utilities for personality calibration. DominantFunction
extracts the dominant Jungian function from an AgentDisposition profile.
FunctionFormatConstraint provides function-aligned text for format
constraints, cognitive approach directives, reasoning instructions,
and thinking field descriptions.

Refs #12, #13, #14"
```

---

### Task 2: toBuilder() on AgentDescriptor (cross-repo: eidos-api)

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/eidos/api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java`
- Test: `/Users/mdproctor/claude/casehub/eidos/api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java` (add test)

**Interfaces:**
- Consumes: existing `AgentDescriptor` record fields + `Builder` class
- Produces: `AgentDescriptor.toBuilder() → Builder` — copies all 21 fields into a new Builder

- [ ] **Step 1: Write failing test**

Add to existing `AgentDescriptorTest` (or create if absent):

```java
@Test
void toBuilder_roundtrip_preserves_all_fields() {
    var original = AgentDescriptor.builder()
        .agentId("test-agent")
        .name("Test Agent")
        .slot("test-slot")
        .tenancyId("test-tenant")
        .briefing("Original briefing text that is long enough")
        .build();

    var copy = original.toBuilder().build();
    assertThat(copy).isEqualTo(original);
}

@Test
void toBuilder_allows_field_override() {
    var original = AgentDescriptor.builder()
        .agentId("test-agent")
        .name("Test Agent")
        .slot("test-slot")
        .tenancyId("test-tenant")
        .briefing("Original briefing text that is long enough")
        .build();

    var modified = original.toBuilder()
        .briefing("Modified briefing text that is also long enough")
        .build();

    assertThat(modified.agentId()).isEqualTo("test-agent");
    assertThat(modified.briefing()).isEqualTo("Modified briefing text that is also long enough");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=AgentDescriptorTest`
Expected: compilation error — `toBuilder()` does not exist

- [ ] **Step 3: Implement toBuilder()**

Add to `AgentDescriptor.java`, after the `builder()` static method:

```java
public Builder toBuilder() {
    return new Builder()
        .agentId(this.agentId).name(this.name).version(this.version)
        .provider(this.provider).modelFamily(this.modelFamily)
        .modelVersion(this.modelVersion)
        .weightsFingerprint(this.weightsFingerprint)
        .domainVocabulary(this.domainVocabulary)
        .slotVocabulary(this.slotVocabulary)
        .dispositionVocabulary(this.dispositionVocabulary)
        .axisVocabularies(this.axisVocabularies)
        .slot(this.slot).capabilities(this.capabilities)
        .disposition(this.disposition)
        .jurisdiction(this.jurisdiction)
        .dataHandlingPolicy(this.dataHandlingPolicy)
        .tenancyId(this.tenancyId).briefing(this.briefing)
        .templates(this.templates).goals(this.goals)
        .constraints(this.constraints);
}
```

- [ ] **Step 4: Run tests and verify pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=AgentDescriptorTest`
Expected: PASS

- [ ] **Step 5: Install to local Maven**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -f /Users/mdproctor/claude/casehub/eidos/pom.xml -DskipTests`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java api/src/test/java/io/casehub/eidos/api/AgentDescriptorTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(api): add toBuilder() to AgentDescriptor

Enables creating modified copies of AgentDescriptor records.
Required for experiment utilities that vary briefing text while
preserving all other descriptor fields.

Refs casehubio/examples#13"
```

---

### Task 3: BriefingCoherenceJudge (cross-repo: eidos-eval)

**Files:**
- Create: `/Users/mdproctor/claude/casehub/eidos/eval/src/main/java/io/casehub/eidos/eval/BriefingCoherenceJudge.java`
- Test: `/Users/mdproctor/claude/casehub/eidos/eval/src/test/java/io/casehub/eidos/eval/BriefingCoherenceJudgeTest.java`

**Interfaces:**
- Consumes: `AgentDisposition`, `DispositionValue`, `ChatModel`, `ObjectMapper`, `PromptJudge.extractJson()`
- Produces:
  - `BriefingCoherenceJudge.evaluate(String briefingText, AgentDisposition disposition, String dispositionVocabulary, String agentId) → CoherenceResult`
  - `CoherenceResult(String agentId, List<FunctionCoherence> functions, List<Tension> tensions, double overallCoherence)`
  - `FunctionCoherence(String function, double coherence, String briefingSignal, String dispositionExpectation)`
  - `Tension(String function, String briefingPhrase, String dispositionConflict, Severity severity)`
  - `Severity { LOW, MEDIUM, HIGH }`

- [ ] **Step 1: Write failing test for self-guard (non-Jungian vocabulary)**

```java
package io.casehub.eidos.eval;

import io.casehub.eidos.api.AgentDisposition;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BriefingCoherenceJudgeTest {

    @Test
    void returns_sentinel_for_non_jungian_vocabulary() {
        var judge = new BriefingCoherenceJudge(null, null);
        var disposition = AgentDisposition.builder()
            .socialOrient("collaborative")
            .build();

        var result = judge.evaluate("Some briefing", disposition, null, "test-agent");

        assertThat(result.overallCoherence()).isEqualTo(-1.0);
        assertThat(result.functions()).isEmpty();
        assertThat(result.tensions()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=BriefingCoherenceJudgeTest`
Expected: compilation error

- [ ] **Step 3: Implement BriefingCoherenceJudge with records and self-guard**

Create `BriefingCoherenceJudge.java` with:
- Inner records: `FunctionCoherence`, `Tension`, `Severity`, `CoherenceResult`
- Self-guard: if vocabulary is not `urn:casehub:vocab:jungian`, return sentinel
- Judge prompt: single LLM call that reads disposition profile, reads briefing, scores 4 stack functions, lists tensions
- JSON parsing via `PromptJudge.extractJson()` with `MalformedJudgeResponseException` on failure
- `overallCoherence` = weighted mean (dominant×4, auxiliary×3, tertiary×2, inferior×1)

```java
package io.casehub.eidos.eval;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.*;
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.request.ChatRequest;
import io.casehub.eidos.api.AgentDisposition;
import io.casehub.eidos.api.DispositionValue;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import java.util.*;
import java.util.stream.Collectors;

@ApplicationScoped
public class BriefingCoherenceJudge {

    private static final String JUNGIAN_VOCAB = "urn:casehub:vocab:jungian";
    private static final int[] STACK_WEIGHTS = {4, 3, 2, 1};

    static final String SYSTEM_PROMPT = """
        You are a personality coherence analyst. You will be given:
        1. A character's Jungian cognitive function stack (dominant to inferior)
        2. A character briefing text

        For each function in the stack, evaluate whether the briefing text
        reinforces or contradicts what that function expects.

        Return ONLY raw JSON — no markdown, no code blocks:
        {
          "functions": [
            {"function": "te", "coherence": 0.8,
             "briefingSignal": "what the briefing implies for this function",
             "dispositionExpectation": "what the function expects"},
            ...
          ],
          "tensions": [
            {"function": "te", "briefingPhrase": "exact quote from briefing",
             "dispositionConflict": "what it contradicts",
             "severity": "LOW|MEDIUM|HIGH"},
            ...
          ]
        }

        Coherence scoring:
        - 1.0 = briefing strongly reinforces the function's expected behavior
        - 0.7 = briefing is neutral or mildly supportive
        - 0.4 = briefing sends mixed signals
        - 0.1 = briefing actively contradicts the function

        Severity:
        - LOW = minor stylistic mismatch
        - MEDIUM = mixed signals that could confuse the LLM
        - HIGH = direct contradiction (e.g., J-type briefing reads as P)
        """;

    private final ChatModel judgeModel;
    private final ObjectMapper mapper;

    @Inject
    public BriefingCoherenceJudge(@Any Instance<ChatModel> models,
                                   ObjectMapper mapper) {
        this.judgeModel = models != null && models.isResolvable()
                          ? models.get() : null;
        this.mapper = mapper;
    }

    BriefingCoherenceJudge(ChatModel model, ObjectMapper mapper) {
        this.judgeModel = model;
        this.mapper = mapper;
    }

    public CoherenceResult evaluate(String briefingText,
                                     AgentDisposition disposition,
                                     String dispositionVocabulary,
                                     String agentId) {
        if (!JUNGIAN_VOCAB.equals(dispositionVocabulary)
                || disposition == null
                || disposition.dispositionProfile().isEmpty()) {
            return new CoherenceResult(agentId, List.of(), List.of(), -1.0);
        }

        String stackDescription = formatStack(disposition);
        String userPrompt = "Function stack:\n" + stackDescription
                + "\n\nBriefing text:\n" + briefingText;

        ChatRequest request = ChatRequest.builder()
            .messages(SystemMessage.from(SYSTEM_PROMPT),
                      UserMessage.from(userPrompt))
            .build();

        String response = judgeModel.chat(request).aiMessage().text();
        return parse(agentId, disposition, response);
    }

    private String formatStack(AgentDisposition disposition) {
        var profile = disposition.dispositionProfile();
        String[] roles = {"Dominant", "Auxiliary", "Tertiary", "Inferior"};
        var sb = new StringBuilder();
        for (int i = 0; i < Math.min(profile.size(), 4); i++) {
            sb.append(roles[i]).append(": ")
              .append(profile.get(i).term().toUpperCase());
            if (profile.get(i).weight() > 0) {
                sb.append(" (weight: ")
                  .append(String.format("%.2f", profile.get(i).weight()))
                  .append(")");
            }
            sb.append("\n");
        }
        return sb.toString();
    }

    private CoherenceResult parse(String agentId,
                                   AgentDisposition disposition,
                                   String json) {
        try {
            JsonNode root = mapper.readTree(PromptJudge.extractJson(json));
            var functions = new ArrayList<FunctionCoherence>();
            if (root.has("functions")) {
                for (JsonNode fn : root.get("functions")) {
                    functions.add(new FunctionCoherence(
                        fn.get("function").asText(),
                        fn.get("coherence").asDouble(),
                        fn.has("briefingSignal")
                            ? fn.get("briefingSignal").asText() : "",
                        fn.has("dispositionExpectation")
                            ? fn.get("dispositionExpectation").asText() : ""));
                }
            }
            var tensions = new ArrayList<Tension>();
            if (root.has("tensions")) {
                for (JsonNode t : root.get("tensions")) {
                    tensions.add(new Tension(
                        t.get("function").asText(),
                        t.has("briefingPhrase")
                            ? t.get("briefingPhrase").asText() : "",
                        t.has("dispositionConflict")
                            ? t.get("dispositionConflict").asText() : "",
                        Severity.valueOf(
                            t.get("severity").asText().toUpperCase())));
                }
            }
            double overall = weightedCoherence(functions, disposition);
            return new CoherenceResult(agentId, functions, tensions, overall);
        } catch (Exception e) {
            throw new MalformedJudgeResponseException(
                "Failed to parse coherence response: " + e.getMessage());
        }
    }

    private double weightedCoherence(List<FunctionCoherence> functions,
                                      AgentDisposition disposition) {
        if (functions.isEmpty()) return 0.0;
        var profile = disposition.dispositionProfile();
        double weightedSum = 0, totalWeight = 0;
        for (int i = 0; i < Math.min(functions.size(), STACK_WEIGHTS.length); i++) {
            weightedSum += functions.get(i).coherence() * STACK_WEIGHTS[i];
            totalWeight += STACK_WEIGHTS[i];
        }
        return totalWeight > 0 ? weightedSum / totalWeight : 0.0;
    }

    public record FunctionCoherence(String function, double coherence,
                                     String briefingSignal,
                                     String dispositionExpectation) {}

    public record Tension(String function, String briefingPhrase,
                          String dispositionConflict, Severity severity) {}

    public enum Severity { LOW, MEDIUM, HIGH }

    public record CoherenceResult(String agentId,
                                   List<FunctionCoherence> functions,
                                   List<Tension> tensions,
                                   double overallCoherence) {}
}
```

- [ ] **Step 4: Run self-guard test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml -Dtest=BriefingCoherenceJudgeTest`
Expected: PASS

- [ ] **Step 5: Install to local Maven**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl eval -f /Users/mdproctor/claude/casehub/eidos/pom.xml -DskipTests`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/eidos add eval/src/main/java/io/casehub/eidos/eval/BriefingCoherenceJudge.java eval/src/test/java/io/casehub/eidos/eval/BriefingCoherenceJudgeTest.java
git -C /Users/mdproctor/claude/casehub/eidos commit -m "feat(eval): add BriefingCoherenceJudge

LLM-based pre-flight validation that scores coherence between a
character's briefing text and their Jungian function stack. Self-guards
against non-Jungian vocabularies (returns sentinel -1.0). Weighted
scoring: dominant×4, auxiliary×3, tertiary×2, inferior×1.

Refs casehubio/examples#12"
```

---

### Task 4: BriefingMode + BriefingTransform + EvalFilter extension

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/model/BriefingMode.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/experiment/BriefingTransform.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EvalFilter.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/BriefingTransformTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor`, `AgentDescriptor.toBuilder()` (from Task 2)
- Produces:
  - `BriefingMode { EMPTY, NAME_ONLY, NAME_ROLE, RICH }`
  - `BriefingTransform.withBriefing(AgentDescriptor, BriefingMode) → AgentDescriptor`
  - `EvalFilter.includesBriefing(String)` — new method
  - `EvalFilter.includesMechanism(String)` — new method

- [ ] **Step 1: Write BriefingTransform failing tests**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.examples.manor.model.BriefingMode;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class BriefingTransformTest {

    private final AgentDescriptor base = AgentDescriptor.builder()
        .agentId("hooded-claw")
        .name("The Hooded Claw")
        .slot("shaper")
        .tenancyId("wacky-manor")
        .briefing("You are The Hooded Claw, a villain with elaborate schemes.")
        .build();

    @Test
    void empty_sets_null_briefing() {
        var result = BriefingTransform.withBriefing(base, BriefingMode.EMPTY);
        assertThat(result.briefing()).isNull();
        assertThat(result.agentId()).isEqualTo("hooded-claw");
    }

    @Test
    void name_only_uses_descriptor_name() {
        var result = BriefingTransform.withBriefing(base, BriefingMode.NAME_ONLY);
        assertThat(result.briefing())
            .isEqualTo("You are an agent named The Hooded Claw.");
    }

    @Test
    void name_role_uses_hardcoded_role() {
        var result = BriefingTransform.withBriefing(base, BriefingMode.NAME_ROLE);
        assertThat(result.briefing())
            .isEqualTo("You are The Hooded Claw, a villain and secret nemesis.");
    }

    @Test
    void rich_preserves_original() {
        var result = BriefingTransform.withBriefing(base, BriefingMode.RICH);
        assertThat(result.briefing()).isEqualTo(base.briefing());
    }

    @Test
    void name_role_throws_for_unmapped_agent() {
        var unknown = base.toBuilder().agentId("unknown-agent").build();
        assertThatThrownBy(() ->
            BriefingTransform.withBriefing(unknown, BriefingMode.NAME_ROLE))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=BriefingTransformTest`
Expected: compilation error

- [ ] **Step 3: Create BriefingMode enum**

```java
package io.casehub.examples.manor.model;

public enum BriefingMode { EMPTY, NAME_ONLY, NAME_ROLE, RICH }
```

- [ ] **Step 4: Implement BriefingTransform**

```java
package io.casehub.examples.manor.experiment;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.examples.manor.model.BriefingMode;
import java.util.Map;

public final class BriefingTransform {
    private BriefingTransform() {}

    private static final Map<String, String> ROLE_PHRASES = Map.of(
        "hooded-claw", "a villain and secret nemesis",
        "penelope-pitstop", "a resourceful Southern belle",
        "ant-hill-mob", "a gang of protective bodyguards",
        "dick-dastardly", "a scheming cheat",
        "peter-perfect", "a gallant hero"
    );

    public static AgentDescriptor withBriefing(AgentDescriptor desc,
                                                BriefingMode mode) {
        String briefing = switch (mode) {
            case EMPTY -> null;
            case NAME_ONLY -> "You are an agent named " + desc.name() + ".";
            case NAME_ROLE -> {
                String role = ROLE_PHRASES.get(desc.agentId());
                if (role == null) {
                    throw new IllegalArgumentException(
                        "No role phrase mapped for: " + desc.agentId());
                }
                yield "You are " + desc.name() + ", " + role + ".";
            }
            case RICH -> desc.briefing();
        };
        return desc.toBuilder().briefing(briefing).build();
    }
}
```

- [ ] **Step 5: Extend EvalFilter with briefing and mechanism support**

Add to `EvalFilter.java`:

```java
record EvalFilter(Set<String> characters, Set<String> layers,
                  Set<String> briefings, Set<String> mechanisms) {

    static EvalFilter from(Optional<String> characters,
                           Optional<String> layers,
                           Optional<String> briefings,
                           Optional<String> mechanisms) {
        return new EvalFilter(parse(characters), parse(layers),
                              parse(briefings), parse(mechanisms));
    }

    boolean includesCharacter(String agentId) {
        return characters.isEmpty() || characters.contains(agentId);
    }

    boolean includesLayer(String layerKey) {
        return layers.isEmpty() || layers.contains(layerKey);
    }

    boolean includesBriefing(String briefingKey) {
        return briefings.isEmpty() || briefings.contains(briefingKey);
    }

    boolean includesMechanism(String mechanismKey) {
        return mechanisms.isEmpty() || mechanisms.contains(mechanismKey);
    }

    // parse() unchanged
}
```

Update `PromptQualityTest.evaluate_all_profiles()` to read the new properties:
```java
var filter = EvalFilter.from(
    config.getOptionalValue("eval.characters", String.class),
    config.getOptionalValue("eval.layers", String.class),
    config.getOptionalValue("eval.briefings", String.class),
    config.getOptionalValue("eval.mechanisms", String.class));
```

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=BriefingTransformTest`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/model/BriefingMode.java wacky-manor/src/main/java/io/casehub/examples/manor/experiment/BriefingTransform.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/BriefingTransformTest.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EvalFilter.java
git commit -m "feat(#13): BriefingMode, BriefingTransform, EvalFilter extension

BriefingMode enum (EMPTY/NAME_ONLY/NAME_ROLE/RICH) for experiment
briefing richness levels. BriefingTransform creates modified
AgentDescriptor copies with replaced briefings. EvalFilter gains
briefing and mechanism filtering for incremental experiment runs.

Refs #13"
```

---

### Task 5: PromptQualityTest — coherence + briefing matrix (#12 + #13)

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/PromptQualityTest.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EvalJudgeProducer.java`

**Interfaces:**
- Consumes: `BriefingCoherenceJudge` (Task 3), `BriefingTransform` (Task 4), `EvalFilter` (Task 4), `BriefingMode`, `ProfileMode`
- Produces: extended `prompt-quality.json` with qualified keys and coherence data

- [ ] **Step 1: Add BriefingCoherenceJudge to EvalJudgeProducer**

```java
@Produces @ApplicationScoped
BriefingCoherenceJudge briefingCoherenceJudge(
        @Any Instance<ChatModel> models, ObjectMapper mapper) {
    return new BriefingCoherenceJudge(models, mapper);
}
```

- [ ] **Step 2: Extend PromptQualityTest with briefing matrix and coherence**

Key changes to `evaluate_all_profiles()`:

1. Inject `BriefingCoherenceJudge coherenceJudge`
2. Add outer loop over `BriefingMode` values
3. Add coherence evaluation pass for JUNGIAN/COMPOSITE layers
4. Use qualified keys: `{layer}-{briefingMode}`
5. Key migration: rename unqualified keys to `{layer}-rich` on first run

```java
@Inject BriefingCoherenceJudge coherenceJudge;

@Test
void evaluate_all_profiles() throws Exception {
    var config = ConfigProvider.getConfig();
    var filter = EvalFilter.from(
        config.getOptionalValue("eval.characters", String.class),
        config.getOptionalValue("eval.layers", String.class),
        config.getOptionalValue("eval.briefings", String.class),
        config.getOptionalValue("eval.mechanisms", String.class));

    var outputFile = OUTPUT_DIR.resolve("prompt-quality.json");
    var results = loadOrCreate(outputFile);

    migrateUnqualifiedKeys(results);

    for (ProfileMode profile : ProfileMode.values()) {
        String layerKey = profile.name().toLowerCase();
        if (!filter.includesLayer(layerKey)) continue;

        var descriptors = loadDescriptors(profile);

        for (BriefingMode briefing : BriefingMode.values()) {
            String briefingKey = briefing.name().toLowerCase();
            if (!filter.includesBriefing(briefingKey)) continue;

            String resultKey = layerKey + "-" + briefingKey;
            @SuppressWarnings("unchecked")
            var profileResults = results.containsKey(resultKey)
                ? new LinkedHashMap<>((Map<String, Object>) results.get(resultKey))
                : new LinkedHashMap<String, Object>();

            for (AgentDescriptor desc : descriptors) {
                if (!filter.includesCharacter(desc.agentId())) continue;

                var transformed = BriefingTransform.withBriefing(desc, briefing);
                String rendered = renderer.render(transformed,
                    AgentPromptContext.forFormat(RenderFormat.MARKDOWN)).content();

                var charResult = new LinkedHashMap<String, Object>();

                // Coherence check (from raw descriptor, not rendered)
                if (profile == ProfileMode.JUNGIAN || profile == ProfileMode.COMPOSITE) {
                    var coherenceResult = callWithRetry(
                        () -> coherenceJudge.evaluate(
                            transformed.briefing(),
                            transformed.disposition(),
                            transformed.dispositionVocabulary(),
                            transformed.agentId()),
                        resultKey + "/" + desc.agentId() + "/coherence");
                    if (coherenceResult != null) {
                        charResult.put("coherence", coherenceResult);
                    }
                }

                // MBTI alignment
                if (profile == ProfileMode.JUNGIAN || profile == ProfileMode.COMPOSITE) {
                    String expectedType = EXPECTED_MBTI.get(desc.agentId());
                    if (expectedType != null) {
                        var mbtiResult = callWithRetry(
                            () -> mbtiJudge.evaluate(rendered, expectedType),
                            resultKey + "/" + desc.agentId() + "/mbti");
                        if (mbtiResult != null) {
                            charResult.put("mbtiAlignment", mbtiResult);
                        }
                    }
                }

                // Function activation
                var scenarios = SCENARIOS.getOrDefault(desc.agentId(), List.of());
                if (!scenarios.isEmpty()) {
                    var funcResult = callWithRetry(
                        () -> functionJudge.evaluate(rendered, desc.agentId(), scenarios),
                        resultKey + "/" + desc.agentId() + "/function");
                    if (funcResult != null) {
                        charResult.put("functionActivation", funcResult);
                    }
                }

                profileResults.put(desc.agentId(), charResult);
            }
            results.put(resultKey, profileResults);
            System.out.printf("--- %s-%s complete ---%n", profile, briefing);
        }
    }

    writeResults(outputFile, results);
}

private void migrateUnqualifiedKeys(LinkedHashMap<String, Object> results) {
    for (ProfileMode p : ProfileMode.values()) {
        String old = p.name().toLowerCase();
        String qualified = old + "-rich";
        if (results.containsKey(old) && !results.containsKey(qualified)) {
            results.put(qualified, results.remove(old));
            System.out.printf("Migrated key: %s → %s%n", old, qualified);
        }
    }
}
```

- [ ] **Step 3: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl wacky-manor`
Expected: compilation success

- [ ] **Step 4: Run a targeted eval to verify integration (single character, single layer)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -Dtest=PromptQualityTest -Deval.characters=hooded-claw -Deval.layers=composite -Deval.briefings=name_only`
Expected: produces `target/experiment-results/prompt-quality.json` with key `composite-name_only` containing hooded-claw results

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/PromptQualityTest.java wacky-manor/src/test/java/io/casehub/examples/manor/experiment/EvalJudgeProducer.java
git commit -m "feat(#12,#13): extend PromptQualityTest with coherence + briefing matrix

Adds BriefingCoherenceJudge integration for JUNGIAN/COMPOSITE layers.
Extends experiment matrix with BriefingMode dimension (EMPTY/NAME_ONLY/
NAME_ROLE/RICH). Uses qualified result keys ({layer}-{briefingMode}).
Migrates existing unqualified keys to {layer}-rich on first run.

Refs #12, #13"
```

---

### Task 6: PromptQualityTest — mechanism experiment (#14)

**Files:**
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/experiment/PromptQualityTest.java`

**Interfaces:**
- Consumes: `FunctionFormatConstraint` (Task 1), `DominantFunction` (Task 1), `EvalFilter.includesMechanism()` (Task 4)
- Produces: extended `prompt-quality.json` with mechanism-qualified keys

- [ ] **Step 1: Add mechanism experiment loop to PromptQualityTest**

After the briefing matrix loop (Task 5), add:

```java
// --- #14: Mechanism experiment ---
Set<String> allMechanisms = Set.of(
    "format_constraint", "observation_directive",
    "schema_reinforcement", "all");

for (String mechanism : allMechanisms) {
    if (!filter.includesMechanism(mechanism)) continue;

    String resultKey = "composite-rich-" + mechanism;
    var mechResults = results.containsKey(resultKey)
        ? new LinkedHashMap<>((Map<String, Object>) results.get(resultKey))
        : new LinkedHashMap<String, Object>();

    var descriptors = loadDescriptors(ProfileMode.COMPOSITE);
    Set<String> active = "all".equals(mechanism)
        ? Set.of("format_constraint", "observation_directive",
                 "schema_reinforcement")
        : Set.of(mechanism);

    for (AgentDescriptor desc : descriptors) {
        if (!filter.includesCharacter(desc.agentId())) continue;

        String rendered = renderer.render(desc,
            AgentPromptContext.forFormat(RenderFormat.MARKDOWN)).content();

        // Mechanism 1: append format constraint to system prompt
        if (active.contains("format_constraint")) {
            DominantFunction.of(desc.disposition()).ifPresent(dominant -> {
                String constraint = FunctionFormatConstraint.forDominant(dominant);
                if (constraint != null) {
                    rendered += "\n\n## Response Format\n" + constraint;
                }
            });
        }

        // Mechanisms 2+3: enrich scenarios
        var scenarios = SCENARIOS.getOrDefault(desc.agentId(), List.of());
        var enriched = enrichScenarios(scenarios, desc.disposition(), active);

        if (!enriched.isEmpty()) {
            var funcResult = callWithRetry(
                () -> functionJudge.evaluate(rendered, desc.agentId(), enriched),
                resultKey + "/" + desc.agentId() + "/function");
            if (funcResult != null) {
                mechResults.put(desc.agentId(),
                    Map.of("functionActivation", funcResult));
            }
        }
    }
    results.put(resultKey, mechResults);
    System.out.printf("--- mechanism %s complete ---%n", mechanism);
}
```

- [ ] **Step 2: Add enrichScenarios method**

```java
private List<FunctionActivationJudge.FunctionScenario> enrichScenarios(
        List<FunctionActivationJudge.FunctionScenario> scenarios,
        AgentDisposition disposition, Set<String> activeMechanisms) {
    String prefix = "";
    String suffix = "";
    if (activeMechanisms.contains("observation_directive")) {
        String approach = FunctionFormatConstraint.cognitiveApproach(disposition);
        if (approach != null) {
            prefix = "== Cognitive Approach ==\n" + approach + "\n\n";
        }
    }
    if (activeMechanisms.contains("schema_reinforcement")) {
        String instruction = FunctionFormatConstraint.reasoningInstruction(disposition);
        if (instruction != null) {
            suffix = "\n\n" + instruction;
        }
    }
    String p = prefix, s = suffix;
    return scenarios.stream()
        .map(sc -> new FunctionActivationJudge.FunctionScenario(
            sc.targetFunction(), p + sc.prompt() + s))
        .toList();
}
```

- [ ] **Step 3: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl wacky-manor`
Expected: compilation success

- [ ] **Step 4: Run a targeted mechanism eval**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Pllm-eval -Dtest=PromptQualityTest -Deval.characters=hooded-claw -Deval.mechanisms=format_constraint`
Expected: produces key `composite-rich-format_constraint` with hooded-claw function activation results

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/test/java/io/casehub/examples/manor/experiment/PromptQualityTest.java
git commit -m "feat(#14): mechanism experiment in PromptQualityTest

Adds eval.mechanisms property to run format_constraint,
observation_directive, schema_reinforcement, or all. Enriches
scenarios with cognitive approach and reasoning instructions.
Results keyed as composite-rich-{mechanism}.

Refs #14"
```

---

## Settled Decisions

| Decision | Rationale |
|----------|-----------|
| `toBuilder()` on AgentDescriptor, not manual record copy | Spec direction + reusable for future experiments |
| EMPTY briefing = null (not blank) | `validateOptional` skips null, throws on blank |
| Self-guard returns -1.0, not exception | Callers don't need to know applicability rule |
| Mechanisms off by default (no `eval.mechanisms` = skip) | No effect on production game runs or existing experiments |
| Post-render transform for Mechanism 1, not eidos pipeline | Experiment mechanism, not a platform feature |
| Reuse `thinking` field for Mechanism 3, not new `reasoning` field | Two overlapping introspection prompts dilute rather than reinforce |
