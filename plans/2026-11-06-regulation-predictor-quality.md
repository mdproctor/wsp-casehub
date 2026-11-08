# Regulation Predictor Quality — Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #830 — fix: regulation predictor quality — address shame fall-through and 4→5 ceiling
**Issue group:** #830

**Goal:** Improve LLM regulation quality when the predictor falls through by injecting personality disposition axes and self-conscious emotion guidance into the user message.

**Architecture:** Two conditional sections added to `buildUserMessage()` in `EmotionRegulationGenerationPhase`, following the existing pattern for fatigue/load/failure signal injection. The tick participant passes `DispositionAxes` and dominant `EmotionType` through to the generation phase when the predictor falls through.

**Tech Stack:** Java 26, Quarkus, JUnit 5, AssertJ, Mockito

## Global Constraints

- All model-derived guidance must cite published literature (protocol: cite-literature-for-model-constructs)
- Build with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <pom> test -s .mvn/slot-settings.xml`
- Use `ide_insert_member`/`ide_replace_member` for structural code edits
- Commit after each task

---

## Batch 1: Personality-Aware Fall-Through Guidance

### Task 1: Add disposition axes section to buildUserMessage

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java:164-232`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java`

**Interfaces:**
- Consumes: `DispositionAxes` (existing record from cognitive-index module)
- Produces: Extended `buildUserMessage()` with two new parameters: `@Nullable DispositionAxes axes`, `@Nullable EmotionType dominantEmotionType`

- [ ] **Step 1: Write failing test — axes appear in user message when provided**

Add to `EmotionRegulationGenerationPhaseTest.java`:

```java
@Test
void userMessageIncludesDispositionAxesWhenProvided() {
    var emotion = new CognitiveEmotion(EmotionType.SHAME, 0.6, "self",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);
    var axes = new DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant");

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null, null, axes, EmotionType.SHAME);

    assertThat(message).contains("Personality disposition");
    assertThat(message).contains("predictor did not pin a strategy");
    assertThat(message).contains("conflict-mode: avoidant");
    assertThat(message).contains("social-orient: cooperative");
    assertThat(message).contains("risk-appetite: conservative");
    assertThat(message).contains("autonomy: low");
    assertThat(message).contains("rule-following: moderate");
}
```

- [ ] **Step 2: Write failing test — axes omitted when pinnedStrategy is present**

```java
@Test
void userMessageOmitsAxesWhenStrategyIsPinned() {
    var emotion = new CognitiveEmotion(EmotionType.ANGER, 0.5, "rival",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);
    var axes = new DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant");

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null,
            EmotionRegulationStrategy.RESPONSE_MODULATION, axes, EmotionType.ANGER);

    assertThat(message).doesNotContain("Personality disposition");
    assertThat(message).contains("Strategy: RESPONSE_MODULATION");
}
```

- [ ] **Step 3: Write failing test — axes omitted when null**

```java
@Test
void userMessageOmitsAxesWhenNull() {
    var emotion = new CognitiveEmotion(EmotionType.ANGER, 0.5, "rival",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null, null, null, EmotionType.ANGER);

    assertThat(message).doesNotContain("Personality disposition");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest" -s .mvn/slot-settings.xml`

Expected: compilation error — `buildUserMessage` does not accept axes/emotionType parameters yet.

- [ ] **Step 5: Add new buildUserMessage overload with axes and dominantEmotionType**

Add a new overload to `EmotionRegulationGenerationPhase` that extends the existing 8-parameter version with two new parameters. The existing 8-parameter overload delegates to this new one with `null, null`.

The new overload adds after the existing conditional sections (pinned strategy, effort, fatigue, load, failure signal, emotions, history):

```java
static String buildUserMessage(CognitionTickContext context, AppraisalResult appraisal,
                                 List<EmotionRegulationResult> history, double regulationEffort,
                                 @Nullable CognitiveLoadEstimate loadEstimate,
                                 double regulationFatigue,
                                 @Nullable StrategyFailureSignal failureSignal,
                                 @Nullable EmotionRegulationStrategy pinnedStrategy,
                                 @Nullable DispositionAxes axes,
                                 @Nullable EmotionType dominantEmotionType) {
    // ... existing logic from 8-param overload ...

    // NEW: disposition axes on fall-through (after pinned strategy section, before emotions)
    if (pinnedStrategy == null && axes != null) {
        sb.append("Personality disposition (predictor did not pin a strategy — use these to ")
                .append("select a personality-congruent regulation):\n")
                .append("  conflict-mode: ").append(axes.conflictMode()).append("\n")
                .append("  social-orient: ").append(axes.socialOrient()).append("\n")
                .append("  risk-appetite: ").append(axes.riskAppetite()).append("\n")
                .append("  autonomy: ").append(axes.autonomy()).append("\n")
                .append("  rule-following: ").append(axes.ruleFollowing()).append("\n\n");
    }

    // ... rest of existing logic (emotions, history) ...
}
```

Update the existing 8-parameter overload to delegate:

```java
static String buildUserMessage(CognitionTickContext context, AppraisalResult appraisal,
                                 List<EmotionRegulationResult> history, double regulationEffort,
                                 @Nullable CognitiveLoadEstimate loadEstimate,
                                 double regulationFatigue,
                                 @Nullable StrategyFailureSignal failureSignal,
                                 @Nullable EmotionRegulationStrategy pinnedStrategy) {
    return buildUserMessage(context, appraisal, history, regulationEffort, loadEstimate,
            regulationFatigue, failureSignal, pinnedStrategy, null, null);
}
```

Also update the 7-parameter overload (without pinnedStrategy) to delegate through:

```java
static String buildUserMessage(CognitionTickContext context, AppraisalResult appraisal,
                                 List<EmotionRegulationResult> history, double regulationEffort,
                                 @Nullable CognitiveLoadEstimate loadEstimate,
                                 double regulationFatigue,
                                 @Nullable StrategyFailureSignal failureSignal) {
    return buildUserMessage(context, appraisal, history, regulationEffort, loadEstimate,
            regulationFatigue, failureSignal, null, null, null);
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest" -s .mvn/slot-settings.xml`

Expected: all tests PASS (new tests + existing tests via delegation)

- [ ] **Step 7: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java
git commit -m "feat(#830): add disposition axes to regulation user message on predictor fall-through

Adds a new buildUserMessage overload accepting DispositionAxes and
dominant EmotionType. When the predictor falls through (pinnedStrategy
is null), the LLM receives the character's personality axes to make a
personality-congruent strategy selection.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Add self-conscious emotion guidance section

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/RegulationStrategyPredictor.java` (expose `emotionCategory`)
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java`

**Interfaces:**
- Consumes: `RegulationStrategyPredictor.emotionCategory(EmotionType)` (already static package-private — no change needed, just use it)
- Produces: Self-conscious emotion guidance injected into user message when SHAME/REMORSE is dominant and no strategy is pinned

- [ ] **Step 1: Write failing test — shame guidance appears for unpinned shame**

```java
@Test
void userMessageIncludesSelfConsciousGuidanceForShame() {
    var emotion = new CognitiveEmotion(EmotionType.SHAME, 0.6, "self",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);
    var axes = new DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant");

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null, null, axes, EmotionType.SHAME);

    assertThat(message).contains("Self-conscious emotion guidance");
    assertThat(message).contains("self-evaluative");
    assertThat(message).contains("temporal containment");
    // Webb et al. 2012 citation
    assertThat(message).contains("Webb et al. 2012");
}
```

- [ ] **Step 2: Write failing test — shame guidance appears for unpinned remorse**

```java
@Test
void userMessageIncludesSelfConsciousGuidanceForRemorse() {
    var emotion = new CognitiveEmotion(EmotionType.REMORSE, 0.6, "self",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null, null, null, EmotionType.REMORSE);

    assertThat(message).contains("Self-conscious emotion guidance");
}
```

- [ ] **Step 3: Write failing test — no guidance for anger**

```java
@Test
void userMessageOmitsSelfConsciousGuidanceForAnger() {
    var emotion = new CognitiveEmotion(EmotionType.ANGER, 0.6, "rival",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null, null, null, EmotionType.ANGER);

    assertThat(message).doesNotContain("Self-conscious emotion guidance");
}
```

- [ ] **Step 4: Write failing test — no guidance when strategy is pinned (even for shame)**

```java
@Test
void userMessageOmitsSelfConsciousGuidanceWhenStrategyPinned() {
    var emotion = new CognitiveEmotion(EmotionType.SHAME, 0.6, "self",
                                       Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);

    var message = EmotionRegulationGenerationPhase.buildUserMessage(
            context, appraisal, List.of(), 0.5, null, 0.0, null,
            EmotionRegulationStrategy.COGNITIVE_CHANGE, null, EmotionType.SHAME);

    assertThat(message).doesNotContain("Self-conscious emotion guidance");
}
```

- [ ] **Step 5: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest" -s .mvn/slot-settings.xml`

Expected: FAIL — no self-conscious guidance section exists yet.

- [ ] **Step 6: Implement self-conscious emotion guidance section**

In the 10-parameter `buildUserMessage()`, after the disposition axes section and before the emotions section, add:

```java
// Webb et al. 2012, Nathanson 1992, Yang et al. 2025 — self-conscious emotion regulation guidance
if (pinnedStrategy == null && dominantEmotionType != null
        && RegulationStrategyPredictor.emotionCategory(dominantEmotionType)
                == RegulationStrategyPredictor.EmotionCategory.SELF_CONSCIOUS) {
    sb.append("Self-conscious emotion guidance: Shame and remorse are self-evaluative — ")
            .append("the emotion IS about the self's failure against its own standards. ")
            .append("Pure suppression leaves the self-evaluation unaddressed (Webb et al. 2012). ")
            .append("For avoidant personalities, temporal containment is adaptive: contain the expression ")
            .append("now, with awareness that the self-evaluation will be processed when cognitive ")
            .append("resources are available (Nathanson 1992 compass of shame; Yang et al. 2025). ")
            .append("This is not suppression-as-avoidance — it is staged regulation where ")
            .append("containment enables later reappraisal.\n\n");
}
```

Note: `emotionCategory()` is already `static` and package-private in `RegulationStrategyPredictor`. Since `EmotionRegulationGenerationPhase` is in the same package, it can call it directly. No visibility change needed.

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest" -s .mvn/slot-settings.xml`

Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java
git commit -m "feat(#830): add self-conscious emotion guidance for predictor fall-through

Webb et al. 2012: shame is self-evaluative — pure suppression leaves the
self-evaluation unaddressed. Nathanson 1992 + Yang et al. 2025: temporal
containment (contain now, reappraise when safe) is adaptive for avoidant
personalities facing shame/remorse.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 2: Wire Through Tick Participant

### Task 3: Pass axes and dominant emotion from tick participant to generation phase

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java:116-133` (add generate overload accepting axes and dominantEmotionType)
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java:112-131` (pass axes and peakEmotion type on fall-through)
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipantTest.java`

**Interfaces:**
- Consumes: `DispositionAxes` from `defaultsRegistry.forAgent()` (already accessed in `tryPredictStrategy`), `EmotionType` from peak emotion selection
- Produces: `generate()` overload that accepts `@Nullable DispositionAxes axes, @Nullable EmotionType dominantEmotionType` and passes them through to `invokeLlm()` → `buildUserMessage()`

- [ ] **Step 1: Write failing test — tick participant passes axes when predictor falls through**

Add to `EmotionRegulationTickParticipantTest.java`. Uses the existing mock pattern from `mechanicalPrediction_fallsToLlmWhenAmbiguous` (line 128) but adds a mocked generation phase to capture the call:

```java
@Test
void passesDispositionAxesOnFallThrough() {
    // Incoherent personality → predictor falls through
    var axes = new DispositionAxes("independent", "flexible", "conservative", "moderate", "avoidant");
    var descriptor = DescriptorView.of("a1", axes, List.of(), List.of());
    var defaults = CognitiveDefaults.empty("a1").withDescriptor(descriptor);

    var registry = mock(CognitiveDefaultsRegistry.class);
    when(registry.forAgent("a1")).thenReturn(Optional.of(defaults));

    var emotion = new CognitiveEmotion(EmotionType.ANGER, 0.65, "subject1",
            Instant.now(), EmotionSource.INTRINSIC, PadProjection.NEUTRAL);
    var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);
    when(appraisalParticipant.currentResult("a1", "t1")).thenReturn(Optional.of(appraisal));

    var generationPhase = mock(EmotionRegulationGenerationPhase.class);
    var config = CognitionConfig.none().with("regulation", true).with("appraisal", true);
    var participant = new EmotionRegulationTickParticipant(
            appraisalParticipant, config, generationPhase, null, null, registry, null);
    participant.tick(context);

    // Verify generate was called with axes and dominant emotion type
    verify(generationPhase).generate(
            eq(context), any(), any(), anyDouble(), anyDouble(), any(), isNull(),
            eq(axes), eq(EmotionType.ANGER));
}
```

Import additions: `import static org.mockito.Mockito.verify;` and `import static org.mockito.ArgumentMatchers.*;`

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationTickParticipantTest" -s .mvn/slot-settings.xml`

Expected: FAIL — generate() doesn't accept axes parameters yet.

- [ ] **Step 3: Add generate overload accepting axes and dominantEmotionType**

Add to `EmotionRegulationGenerationPhase`:

```java
public @Nullable EmotionRegulationResult generate(CognitionTickContext context,
                                                   AppraisalResult appraisal,
                                                   List<EmotionRegulationResult> history,
                                                   double regulationEffort,
                                                   double regulationFatigue,
                                                   @Nullable StrategyFailureSignal failureSignal,
                                                   @Nullable EmotionRegulationStrategy pinnedStrategy,
                                                   @Nullable DispositionAxes axes,
                                                   @Nullable EmotionType dominantEmotionType) {
    if (sessionPipeline == null) return null;
    try {
        String response = invokeLlm(context, appraisal, history, regulationEffort,
                regulationFatigue, failureSignal, pinnedStrategy, axes, dominantEmotionType);
        if (response == null) return null;
        return parseResponse(response);
    } catch (Exception e) {
        LOG.log(Level.WARNING, "Emotion regulation generation failed", e);
        return null;
    }
}
```

Update `invokeLlm` to accept and pass through axes and dominantEmotionType to the 10-parameter `buildUserMessage`.

- [ ] **Step 4: Update tick participant to pass axes and dominant emotion type**

In `EmotionRegulationTickParticipant.tick()`, change the generate call (lines 129-130):

```java
// Extract peak emotion for dominant type (reuse from tryPredictStrategy pattern)
var peakEmotion = appraisal.get().emotions().stream()
        .max(java.util.Comparator.comparingDouble(e -> e.intensity()))
        .orElse(null);

// Get axes when predictor fell through (pinnedStrategy == null)
DispositionAxes axes = null;
EmotionType dominantType = peakEmotion != null ? peakEmotion.type() : null;
if (pinnedStrategy == null && defaultsRegistry != null) {
    var defaults = defaultsRegistry.forAgent(context.agentId());
    if (defaults.isPresent() && defaults.get().descriptor() != null) {
        axes = defaults.get().descriptor().disposition();
    }
}

var result = generationPhase.generate(context, appraisal.get(), history,
        regulationEffort, currentFatigue, pendingSignal, pinnedStrategy,
        axes, dominantType);
```

Note: The peak emotion logic currently lives in `tryPredictStrategy()`. To avoid duplication, extract the peak emotion selection to happen earlier in `tick()` and pass it to both `tryPredictStrategy` and the generate call.

- [ ] **Step 5: Run all regulation tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest,EmotionRegulationTickParticipantTest,RegulationStrategyPredictorTest" -s .mvn/slot-settings.xml`

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipantTest.java
git commit -m "feat(#830): wire disposition axes through tick participant to generation phase

When the predictor falls through, the tick participant now passes the
character's DispositionAxes and dominant EmotionType to the generation
phase, which includes them in the LLM user message for personality-aware
strategy selection.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## Batch 3: Eval and Document

### Task 4: Run LLM judge eval and document findings

**Files:**
- Run: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/RegulationPredictorBackTest.java`
- Create: `docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md`

**Interfaces:**
- Consumes: Everything from Batches 1-2
- Produces: Eval results comparison, research findings document

- [ ] **Step 1: Build cognition module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml`

Expected: BUILD SUCCESS

- [ ] **Step 2: Run the LLM judge eval**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f caps-llm-classifier/pom.xml test -Dtest="RegulationPredictorBackTest" -Pllm-eval -s .mvn/slot-settings.xml`

Expected: test passes (mean ≥ 4.0 gate). Read results from `caps-llm-classifier/target/regulation-predictor-judge-results.json`.

- [ ] **Step 3: Compare results with baseline**

Baseline (from #818):
- scored: 5, skipped: 3, mean: 4.6
- suppressor-anger: 4, reappraiser-anger: 5, reappraiser-reproach: 4, anyone-fear: 5, reappraiser-shame: 5
- suppressor-shame: FELL_THROUGH (-1), incoherent-anger: FELL_THROUGH (-1), mixed-anger: FELL_THROUGH (-1)

Compare: did suppressor-shame get scored? Did existing scores hold? Did the mean change?

Note: The eval test currently skips fell-through cases. The predictor still falls through for suppressor-shame — what changed is that the LLM now gets personality context. But the eval test judges the PREDICTOR's selection, not the LLM's response. This means we may need to also test the LLM's output quality separately, or adjust the eval to judge the end-to-end result.

If the eval test still skips fell-through cases (predictor logic unchanged), the direct quality improvement won't be visible in this eval. In that case, document this observation and describe what a proper end-to-end eval would look like.

- [ ] **Step 4: Write research findings**

Create `docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md`:

Include:
- Baseline judge scores (from #818)
- Phase 1 judge scores (from this run)
- What changed in the LLM input
- What the judge can and cannot measure (predictor vs. end-to-end)
- Literature grounding for the changes (Webb et al. 2012, Nathanson 1992, Yang et al. 2025)
- Observations for Phase 2 design

- [ ] **Step 5: Commit**

```bash
git add docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md
git commit -m "eval(#830): Phase 1 regulation quality findings — personality-aware fall-through guidance

Documents baseline comparison, literature grounding, and observations
for Phase 2 CognitiveDerivationEngine design.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

## References

- [2026-11-06-regulation-predictor-quality-design.md] — design spec this plan implements
- [EmotionRegulationGenerationPhase.java:155-232] — buildUserMessage conditional sections pattern
- [EmotionRegulationTickParticipant.java:112-184] — tick method and tryPredictStrategy
- [RegulationStrategyPredictor.java:92-108] — self-conscious category, emotionCategory method
- [RegulationPredictorBackTest.java] — LLM judge eval test
- [cite-literature-for-model-constructs] — protocol requiring literature citations
- [GitHub #830] — focal issue
- [GitHub #818] — original regulation predictor (closed)
