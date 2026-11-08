# Social Emotion Regulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #805 — feat: social emotion regulation — interpersonal strategies (Zaki & Williams 2013)
**Issue group:** #805 (within epic #851)

**Goal:** Add interpersonal regulation strategies so characters can use social means to regulate emotions (seeking comfort, social sharing, withdrawal, co-regulation, deliberate provocation).

**Architecture:** Extend the existing `EmotionRegulationStrategy` enum with 5 social strategies, add `targetAgentId` to `EmotionRegulationResult`, gate social strategies in the LLM prompt on `RegulationContext.interactingAgents()`, and extend the goal bridge for social goal routing via `appendGoals()`.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Mockito, Quarkus config

## Global Constraints

- All new types in cognition-api must be records or enums (no mutable state)
- Tests use AssertJ assertions, not JUnit assertEquals
- Use `ide_insert_member` for new methods, `ide_create_file` for new files
- Commit after each task with `feat(#805):` or `test(#805):` prefix
- Social strategies are only presented to the LLM when `RegulationContext.interactingAgents()` is non-empty

---

## Batch 1: API Types + Derivation

### Task 1: Extend EmotionRegulationStrategy with social values

**Files:**
- Modify: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationStrategy.java:4-11`
- Test: `cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationStrategyTest.java`

**Interfaces:**
- Produces: `SOCIAL_SUPPORT_SEEKING`, `SOCIAL_SHARING`, `SOCIAL_WITHDRAWAL`, `CO_REGULATION`, `DELIBERATE_PROVOCATION` enum values — consumed by Tasks 2, 4, 5, 6

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.neocortex.cognition.regulation;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class EmotionRegulationStrategyTest {

    @Test
    void socialStrategiesExist() {
        assertThat(EmotionRegulationStrategy.valueOf("SOCIAL_SUPPORT_SEEKING")).isNotNull();
        assertThat(EmotionRegulationStrategy.valueOf("SOCIAL_SHARING")).isNotNull();
        assertThat(EmotionRegulationStrategy.valueOf("SOCIAL_WITHDRAWAL")).isNotNull();
        assertThat(EmotionRegulationStrategy.valueOf("CO_REGULATION")).isNotNull();
        assertThat(EmotionRegulationStrategy.valueOf("DELIBERATE_PROVOCATION")).isNotNull();
    }

    @Test
    void tenStrategiesTotal() {
        assertThat(EmotionRegulationStrategy.values()).hasSize(10);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=EmotionRegulationStrategyTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — enum values not found

- [ ] **Step 3: Add social strategy values to enum**

Use `ide_edit_member` to replace the `EmotionRegulationStrategy` enum:

```java
// Gross 1998 five-strategy process model — situation → attention → appraisal → response
public enum EmotionRegulationStrategy {
    SITUATION_SELECTION,       // Gross strategy 1 — avoid triggering contexts
    SITUATION_MODIFICATION,    // Gross strategy 2 — change the situation
    ATTENTIONAL_DEPLOYMENT,    // Gross strategy 3 — redirect focus (distraction)
    COGNITIVE_CHANGE,          // Gross strategy 4 — reappraise meaning
    RESPONSE_MODULATION,       // Gross strategy 5 — suppress/amplify expression
    // Zaki & Williams 2013 interpersonal strategies
    SOCIAL_SUPPORT_SEEKING,    // intrinsic — seek comfort from a specific agent
    SOCIAL_SHARING,            // intrinsic — share emotional experience with another
    SOCIAL_WITHDRAWAL,         // intrinsic — withdraw from social interaction
    CO_REGULATION,             // extrinsic — soothe/comfort a distressed agent
    DELIBERATE_PROVOCATION     // extrinsic — intentionally provoke emotional response
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=EmotionRegulationStrategyTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationStrategy.java cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationStrategyTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#805): add 5 social regulation strategies to EmotionRegulationStrategy"
```

### Task 2: Add targetAgentId and social classification to EmotionRegulationResult

**Files:**
- Modify: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationResult.java:7-24`
- Modify: `cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationResultTest.java`

**Interfaces:**
- Consumes: Social enum values from Task 1
- Produces: `isSocialStrategy()`, `isExtrinsicStrategy()`, `targetAgentId()` — consumed by Tasks 5, 6

- [ ] **Step 1: Write the failing tests**

Add to existing `EmotionRegulationResultTest.java` (or create if it doesn't exist):

```java
@Test
void isSocialStrategy_trueForSocialStrategies() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.SOCIAL_SUPPORT_SEEKING,
            List.of("ANXIETY"), List.of(), 0.8, 0.4, 0.6,
            "You seek comfort", null, null, "agent-b");
    assertThat(result.isSocialStrategy()).isTrue();
    assertThat(result.isExtrinsicStrategy()).isFalse();
}

@Test
void isExtrinsicStrategy_trueForCoRegulation() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.CO_REGULATION,
            List.of("EMPATHY"), List.of(), 0.7, 0.3, 0.7,
            "You comfort them", null, null, "agent-c");
    assertThat(result.isExtrinsicStrategy()).isTrue();
    assertThat(result.isSocialStrategy()).isTrue();
}

@Test
void intrapersonalStrategy_notSocial() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.COGNITIVE_CHANGE,
            List.of("ANGER"), List.of(), 0.8, 0.4, 0.6,
            "You reframe", null, null, null);
    assertThat(result.isSocialStrategy()).isFalse();
    assertThat(result.isExtrinsicStrategy()).isFalse();
}

@Test
void targetAgentIdAccessible() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.SOCIAL_SUPPORT_SEEKING,
            List.of("ANXIETY"), List.of(), 0.8, 0.4, 0.6,
            "You seek comfort", null, null, "penelope-pitstop");
    assertThat(result.targetAgentId()).isEqualTo("penelope-pitstop");
}

@Test
void existingConstructorPassesNullTargetAgentId() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.COGNITIVE_CHANGE,
            List.of("ANGER"), List.of(), 0.8, 0.4, 0.6,
            "You reframe", null, null);
    assertThat(result.targetAgentId()).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=EmotionRegulationResultTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — no 10-arg constructor, no isSocialStrategy(), no isExtrinsicStrategy()

- [ ] **Step 3: Extend EmotionRegulationResult**

Use `ide_edit_member` to replace the record declaration:

```java
public record EmotionRegulationResult(
    EmotionRegulationStrategy strategy,
    List<String> targetEmotions,
    List<String> targetDrives,
    double peakOriginalIntensity,
    double regulatedIntensity,
    double effectiveness,
    String narrative,
    @Nullable DriveAxis regulationAxis,
    @Nullable String environmentalDirective,
    @Nullable String targetAgentId
) {
    public EmotionRegulationResult(EmotionRegulationStrategy strategy,
                                    List<String> targetEmotions,
                                    List<String> targetDrives,
                                    double peakOriginalIntensity,
                                    double regulatedIntensity,
                                    double effectiveness,
                                    String narrative,
                                    @Nullable DriveAxis regulationAxis,
                                    @Nullable String environmentalDirective) {
        this(strategy, targetEmotions, targetDrives, peakOriginalIntensity,
             regulatedIntensity, effectiveness, narrative, regulationAxis,
             environmentalDirective, null);
    }

    public boolean isEnvironmentalStrategy() {
        return strategy == EmotionRegulationStrategy.SITUATION_SELECTION
            || strategy == EmotionRegulationStrategy.SITUATION_MODIFICATION;
    }

    public boolean isSocialStrategy() {
        return strategy == EmotionRegulationStrategy.SOCIAL_SUPPORT_SEEKING
            || strategy == EmotionRegulationStrategy.SOCIAL_SHARING
            || strategy == EmotionRegulationStrategy.SOCIAL_WITHDRAWAL
            || strategy == EmotionRegulationStrategy.CO_REGULATION
            || strategy == EmotionRegulationStrategy.DELIBERATE_PROVOCATION;
    }

    public boolean isExtrinsicStrategy() {
        return strategy == EmotionRegulationStrategy.CO_REGULATION
            || strategy == EmotionRegulationStrategy.DELIBERATE_PROVOCATION;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=EmotionRegulationResultTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS

- [ ] **Step 5: Verify no downstream compilation breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml compile -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: BUILD SUCCESS (existing callers use 9-arg constructor which now chains to 10-arg with null)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationResult.java cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationResultTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#805): add targetAgentId, isSocialStrategy(), isExtrinsicStrategy() to EmotionRegulationResult"
```

### Task 3: Extend RegulationDefaults with socialRegulationTendency + derivation

**Files:**
- Modify: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/RegulationDefaults.java:18-22`
- Modify: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java:502-525`
- Modify: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java`

**Interfaces:**
- Produces: `RegulationDefaults.socialRegulationTendency()` — consumed by Task 4 (prompt context)
- Produces: `CognitiveDerivationEngine.deriveRegulationDefaults()` returns 4-arg `RegulationDefaults` — consumed by existing callers

- [ ] **Step 1: Write the failing tests**

Add to `CognitiveDerivationEngineTest.java`:

```java
@Test
void deriveRegulationDefaults_gregariousSocialTendency() {
    var axes = new DispositionAxes("harmonizing", "gregarious", "cautious", "collaborative", "flexible");
    var rd = CognitiveDerivationEngine.deriveRegulationDefaults(axes);
    assertThat(rd.socialRegulationTendency()).isGreaterThan(0.6);
}

@Test
void deriveRegulationDefaults_reservedSocialTendency() {
    var axes = new DispositionAxes("confrontational", "reserved", "bold", "independent", "strict");
    var rd = CognitiveDerivationEngine.deriveRegulationDefaults(axes);
    assertThat(rd.socialRegulationTendency()).isLessThan(0.4);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-index/pom.xml test -Dtest="CognitiveDerivationEngineTest#deriveRegulationDefaults_gregariousSocialTendency+deriveRegulationDefaults_reservedSocialTendency" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — no socialRegulationTendency() method

- [ ] **Step 3: Add socialRegulationTendency to RegulationDefaults**

Use `ide_edit_member` to replace the record:

```java
public record RegulationDefaults(
    double reappraisalCapacity,
    double suppressionCapacity,
    double redirectTendency,
    double socialRegulationTendency
) {
    public RegulationDefaults(double reappraisalCapacity, double suppressionCapacity,
                               double redirectTendency) {
        this(reappraisalCapacity, suppressionCapacity, redirectTendency, 0.5);
    }
}
```

- [ ] **Step 4: Add social derivation weights to CognitiveDerivationEngine**

Add static weight maps using `ide_insert_member` (before `deriveRegulationDefaults`):

```java
private static final Map<String, Double> SOCIAL_REG_SOCIAL = Map.of(
        "gregarious", 0.20, "reserved", -0.15, "selective", 0.05);
private static final Map<String, Double> SOCIAL_REG_CONFLICT = Map.of(
        "harmonizing", 0.15, "confrontational", -0.10, "analytical", 0.0, "avoidant", -0.05);
private static final Map<String, Double> SOCIAL_REG_AUTONOMY = Map.of(
        "independent", -0.10, "collaborative", 0.10, "dependent", 0.15);
```

Then update `deriveRegulationDefaults()` to compute `social` and return a 4-arg `RegulationDefaults`:

```java
double social = 0.5
    + SOCIAL_REG_SOCIAL.getOrDefault(lower(axes.socialOrient()), 0.0)
    + SOCIAL_REG_CONFLICT.getOrDefault(lower(axes.conflictMode()), 0.0)
    + SOCIAL_REG_AUTONOMY.getOrDefault(lower(axes.autonomy()), 0.0);
return new RegulationDefaults(
        Math.max(0.0, Math.min(1.0, reappraisal)),
        Math.max(0.0, Math.min(1.0, suppression)),
        Math.max(0.0, Math.min(1.0, redirect)),
        Math.max(0.0, Math.min(1.0, social)));
```

- [ ] **Step 5: Run all RegulationDefaults/derivation tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-index/pom.xml test -Dtest="CognitiveDerivationEngineTest#deriveRegulationDefaults*" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (existing tests + 2 new)

- [ ] **Step 6: Install cognitive-index and verify downstream compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-index/pom.xml install -Dmaven.test.skip=true -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml compile -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/RegulationDefaults.java cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#805): add socialRegulationTendency to RegulationDefaults with disposition-axis derivation"
```

## Batch 2: LLM Generation — Prompt, Parsing, Validation

### Task 4: Extend system prompt and user message with social strategies

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java:27-74` (SYSTEM_PROMPT)
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java:325-513` (buildUserMessage)
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java`

**Interfaces:**
- Consumes: `RegulationContext.interactingAgents()` from #819, `RegulationDefaults.socialRegulationTendency()` from Task 3
- Produces: Extended prompt that enables LLM to select social strategies — consumed by Task 5 (parsing)

- [ ] **Step 1: Write the failing tests**

```java
@Test
void systemPromptContainsSocialStrategies() {
    assertThat(EmotionRegulationGenerationPhase.SYSTEM_PROMPT)
            .contains("SOCIAL_SUPPORT_SEEKING")
            .contains("CO_REGULATION")
            .contains("DELIBERATE_PROVOCATION");
}

@Test
void buildUserMessage_includesSocialContextWhenAgentsPresent() {
    var regCtx = new RegulationContext("library", List.of("penelope-pitstop", "dick-dastardly"), "treasure");
    var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
            null, null, null, regCtx);
    var appraisal = /* minimal AppraisalResult with one emotion */;
    var rd = new RegulationDefaults(0.5, 0.5, 0.5, 0.8);

    var msg = EmotionRegulationGenerationPhase.buildUserMessage(
            ctx, appraisal, List.of(), 0.5, null, 0.0, null, null, null, null, rd, null, null);

    assertThat(msg).contains("Available agents:");
    assertThat(msg).contains("penelope-pitstop");
    assertThat(msg).contains("social regulation tendency");
}

@Test
void buildUserMessage_omitsSocialContextWhenNoAgents() {
    var ctx = new CognitionTickContext("agent1", "tenant1", null, null);
    var appraisal = /* minimal AppraisalResult */;
    var rd = new RegulationDefaults(0.5, 0.5, 0.5, 0.8);

    var msg = EmotionRegulationGenerationPhase.buildUserMessage(
            ctx, appraisal, List.of(), 0.5, null, 0.0, null, null, null, null, rd, null, null);

    assertThat(msg).doesNotContain("Available agents:");
}
```

Note: The test setup for `AppraisalResult` will need to match the existing test patterns in `EmotionRegulationGenerationPhaseTest.java`. Read the existing test `setUp()` for the proper construction.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest#systemPromptContainsSocialStrategies+buildUserMessage_includesSocialContextWhenAgentsPresent+buildUserMessage_omitsSocialContextWhenNoAgents" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL

- [ ] **Step 3: Extend SYSTEM_PROMPT with social strategy descriptions**

Use `ide_edit_member` to replace the `SYSTEM_PROMPT` field. Append after the existing 5 strategy descriptions and before the JSON schema:

```
SOCIAL REGULATION STRATEGIES (Zaki & Williams 2013):

6. SOCIAL_SUPPORT_SEEKING — Seek comfort, reassurance, or advice from \
a specific person present. Intrinsic: regulates YOUR emotions through \
social connection. Produces a goal to approach the target agent.
7. SOCIAL_SHARING — Share your emotional experience with another person \
for relief through disclosure. Intrinsic. Not advice-seeking.
8. SOCIAL_WITHDRAWAL — Remove yourself from social interaction to \
self-regulate alone. Intrinsic. Produces an avoidance goal.
9. CO_REGULATION — Actively comfort or soothe a person you observe \
to be in distress. Extrinsic: regulates THEIR emotions.
10. DELIBERATE_PROVOCATION — Intentionally provoke an emotional response \
in another person for strategic purposes. Extrinsic. Rare.

For social strategies (6-10), set targetAgentId to one of the available \
agents listed in the context. For non-social strategies (1-5), set \
targetAgentId to null.
```

Update the JSON schema in the prompt to include `"targetAgentId": "agent-id or null"`.

- [ ] **Step 4: Extend buildUserMessage with social context section**

Use `ide_insert_member` to add a helper method, then modify `buildUserMessage` to call it. Insert the social context block after the regulation capacities section (after line ~400) and before the temporal staging section:

```java
// Social regulation context (Zaki & Williams 2013)
if (pinnedStrategy == null && context.regulationContext() != null
        && context.regulationContext().interactingAgents() != null
        && !context.regulationContext().interactingAgents().isEmpty()) {
    sb.append("Available agents: ")
      .append(String.join(", ", context.regulationContext().interactingAgents()))
      .append("\n");
    if (regulationDefaults != null) {
        sb.append("Social regulation tendency: ")
          .append(capacityLabel(regulationDefaults.socialRegulationTendency()))
          .append(" (").append(String.format("%.2f", regulationDefaults.socialRegulationTendency()))
          .append(")\n");
    }
    sb.append("\n");
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=EmotionRegulationGenerationPhaseTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (all tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#805): extend regulation prompt with social strategies and agent context"
```

### Task 5: Update parseResponse for targetAgentId + validation in tick participant

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java:523-563` (parseResponse)
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java:88-182` (tick)
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java`
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipantTest.java`

**Interfaces:**
- Consumes: `EmotionRegulationResult` with `targetAgentId` from Task 2, `RegulationContext.interactingAgents()` from #819
- Produces: Validated `EmotionRegulationResult` with valid targetAgentId — consumed by Task 6 (goal routing)

- [ ] **Step 1: Write the failing tests**

For parseResponse:
```java
@Test
void parseResponse_extractsTargetAgentId() {
    String json = """
            {"strategy":"SOCIAL_SUPPORT_SEEKING","targetEmotions":["ANXIETY"],
             "targetDrives":[],"peakOriginalIntensity":0.8,"regulatedIntensity":0.4,
             "effectiveness":0.6,"narrative":"You seek comfort","regulationAxis":null,
             "environmentalDirective":null,"targetAgentId":"penelope-pitstop"}""";
    var result = EmotionRegulationGenerationPhase.parseResponse(json);
    assertThat(result).isNotNull();
    assertThat(result.targetAgentId()).isEqualTo("penelope-pitstop");
    assertThat(result.isSocialStrategy()).isTrue();
}

@Test
void parseResponse_nullTargetAgentIdForIntrapersonal() {
    String json = """
            {"strategy":"COGNITIVE_CHANGE","targetEmotions":["ANGER"],
             "targetDrives":[],"peakOriginalIntensity":0.8,"regulatedIntensity":0.4,
             "effectiveness":0.6,"narrative":"You reframe","regulationAxis":null,
             "environmentalDirective":null,"targetAgentId":null}""";
    var result = EmotionRegulationGenerationPhase.parseResponse(json);
    assertThat(result).isNotNull();
    assertThat(result.targetAgentId()).isNull();
}
```

Note: `parseResponse` is private — these tests may need to go through the static `buildUserMessage` + `parseResponse` path, or the method may need to be made package-private for testing. Check existing test patterns.

For tick validation:
```java
@Test
void tick_discardsResultWithInvalidTargetAgentId() {
    // Setup: regulation fires, LLM returns SOCIAL_SUPPORT_SEEKING targeting "unknown-agent"
    // RegulationContext has interactingAgents=["penelope-pitstop"]
    // Expected: result is discarded because "unknown-agent" is not in interactingAgents
}
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAIL — parseResponse doesn't extract targetAgentId, tick doesn't validate

- [ ] **Step 3: Update parseResponse to extract targetAgentId**

In `parseResponse()`, before the final return statement, add:

```java
String targetAgentId = root.path("targetAgentId").asText(null);
if ("null".equals(targetAgentId)) targetAgentId = null;
```

Update the return to use the 10-arg constructor:

```java
return new EmotionRegulationResult(
        strategy, targetEmotions, targetDrives,
        peakOriginalIntensity, regulatedIntensity, effectiveness,
        narrative, regulationAxis, environmentalDirective, targetAgentId);
```

- [ ] **Step 4: Add targetAgentId validation to EmotionRegulationTickParticipant.tick()**

After the `generate()` call returns a result, add validation before storing it:

```java
if (result != null && result.isSocialStrategy() && result.targetAgentId() != null) {
    var regCtx = context.regulationContext();
    if (regCtx == null || regCtx.interactingAgents() == null
            || !regCtx.interactingAgents().contains(result.targetAgentId())) {
        LOG.fine(() -> "Discarding social regulation result — targetAgentId '"
                + result.targetAgentId() + "' not in interactingAgents");
        result = null;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest,EmotionRegulationTickParticipantTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#805): parse targetAgentId from LLM response and validate against RegulationContext"
```

## Batch 3: Goal Routing

### Task 6: Extend EmotionRegulationGoalParticipant for social strategies

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipant.java:22-38`
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipantTest.java`

**Interfaces:**
- Consumes: `EmotionRegulationResult.isSocialStrategy()`, `targetAgentId()` from Task 2, `GoalProposalOrchestrator.appendGoals()` from #819
- Produces: Social regulation goals registered via `appendGoals()` — consumed by downstream goal pipeline

- [ ] **Step 1: Write the failing tests**

Add to `EmotionRegulationGoalParticipantTest.java`:

```java
@Test
void socialStrategy_producesGoalWithAppendSemantics() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.SOCIAL_SUPPORT_SEEKING,
            List.of("ANXIETY"), List.of(),
            0.8, 0.4, 0.6,
            "You seek comfort from Penelope",
            DriveAxis.AFFILIATION, null, "penelope-pitstop");
    when(regulationParticipant.currentResult("a1", "t1"))
            .thenReturn(Optional.of(result));

    participant.tick(new CognitionTickContext("a1", "t1", null, null));

    verify(goalOrchestrator).appendGoals(eq("a1"), eq("t1"), argThat(goals -> {
        assertThat(goals).hasSize(1);
        assertThat(goals.getFirst().goalName()).contains("social-regulation");
        assertThat(goals.getFirst().goalDescription()).contains("penelope-pitstop");
        return true;
    }));
    verify(goalOrchestrator, never()).registerGoals(any(), any(), any());
}

@Test
void coRegulation_producesGoalWithTarget() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.CO_REGULATION,
            List.of("EMPATHY"), List.of(),
            0.7, 0.3, 0.7,
            "You comfort Dick Dastardly",
            DriveAxis.AFFILIATION, null, "dick-dastardly");
    when(regulationParticipant.currentResult("a1", "t1"))
            .thenReturn(Optional.of(result));

    participant.tick(new CognitionTickContext("a1", "t1", null, null));

    verify(goalOrchestrator).appendGoals(eq("a1"), eq("t1"), argThat(goals -> {
        assertThat(goals).hasSize(1);
        assertThat(goals.getFirst().goalName()).contains("co-regulation");
        return true;
    }));
}

@Test
void environmentalStrategy_stillUsesRegisterGoals() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.SITUATION_SELECTION,
            List.of("ANXIETY"), List.of(),
            0.9, 0.5, 0.7,
            "You leave the room",
            DriveAxis.AUTONOMY, "Avoid the library", null);
    when(regulationParticipant.currentResult("a1", "t1"))
            .thenReturn(Optional.of(result));

    participant.tick(new CognitionTickContext("a1", "t1", null, null));

    verify(goalOrchestrator).registerGoals(eq("a1"), eq("t1"), any());
    verify(goalOrchestrator, never()).appendGoals(any(), any(), any());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=EmotionRegulationGoalParticipantTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — no social goal logic

- [ ] **Step 3: Extend tick() with social goal routing**

Use `ide_edit_member` to replace the `tick` method:

```java
@Override
public void tick(CognitionTickContext context) {
    var result = regulationParticipant.currentResult(context.agentId(), context.tenantId());
    if (result.isEmpty()) return;

    var reg = result.get();
    if (reg.isEnvironmentalStrategy()) {
        if (reg.regulationAxis() == null || reg.environmentalDirective() == null) return;
        var proposal = new DriveGoalProposal(
                reg.regulationAxis(),
                "regulation-" + reg.strategy().name().toLowerCase().replace('_', '-'),
                reg.environmentalDirective(),
                "Emotion regulation: " + reg.narrative(),
                reg.regulatedIntensity());
        goalOrchestrator.registerGoals(context.agentId(), context.tenantId(), List.of(proposal));
    } else if (reg.isSocialStrategy()) {
        var axis = reg.regulationAxis() != null ? reg.regulationAxis() : DriveAxis.AFFILIATION;
        String goalName = "social-regulation-" + reg.strategy().name().toLowerCase().replace('_', '-');
        String description = reg.narrative();
        if (reg.targetAgentId() != null) {
            description += " [target: " + reg.targetAgentId() + "]";
        }
        var proposal = new DriveGoalProposal(
                axis, goalName, description,
                "Social regulation: " + reg.strategy().name(),
                reg.regulatedIntensity());
        goalOrchestrator.appendGoals(context.agentId(), context.tenantId(), List.of(proposal));
    }
}
```

- [ ] **Step 4: Run all goal participant tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=EmotionRegulationGoalParticipantTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (existing + 3 new tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipant.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipantTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#805): extend goal bridge with social regulation routing via appendGoals()"
```

## References

- [2026-11-08-social-regulation-design.md] — design spec this plan implements
- [EmotionRegulationStrategy.java:4-11] — existing 5-strategy enum
- [EmotionRegulationResult.java:7-24] — existing result record
- [RegulationDefaults.java:18-22] — existing 3-field record
- [CognitiveDerivationEngine.java:502-525] — deriveRegulationDefaults() with disposition weights
- [EmotionRegulationGenerationPhase.java:27-74] — system prompt
- [EmotionRegulationGenerationPhase.java:325-513] — buildUserMessage()
- [EmotionRegulationGenerationPhase.java:523-563] — parseResponse()
- [EmotionRegulationGoalParticipant.java:22-38] — existing goal bridge tick()
- [EmotionRegulationTickParticipant.java:88-182] — tick() with validation
- [RegulationContext.java] — interactingAgents for social gate (#819)
- [GoalProposalOrchestrator.java:121-128] — appendGoals() (#819)
- decisions-805.md — D1-D6 design decisions
- GitHub casehubio/neocortex#805 — focal issue
- GitHub casehubio/neocortex#851 — parent epic
