# Phase 2: Continuous Weights + Shame Dimensions — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #830 — fix: regulation predictor quality — address shame fall-through and 4→5 ceiling
**Issue group:** #830, #820

**Goal:** Replace boolean isSuppressor/isReappraiser classification with literature-grounded continuous weights and add shame-specific context (memory compounding, relational closeness, personality-gated depth) to the regulation pipeline.

**Architecture:** Three layers: (1) CDE derives RegulationDefaults with continuous capacities from personality, (2) tick participant assembles ShameContext for self-conscious emotions, (3) generation phase renders both as interpretive narrative sections. Per-category predictor thresholds with dominance margin replace boolean classification.

**Tech Stack:** Java 26, Quarkus, JUnit 5, AssertJ

## Global Constraints

- Weight table values are literature-ordered initial values (Barańczuk 2019), calibrated via end-to-end eval
- Thresholds: reappraisal=0.65, suppression=0.65, dominance margin=0.10
- Redirect tendency is LLM-context-only — never pinned by predictor
- ShameContext assembled for ALL SELF_CONSCIOUS emotions, not just fall-throughs
- Surface-level gating (grandiosity≥0.7 && callousness≥0.7) supersedes Phase 1 temporal containment
- Intensity (>0.75→AD) and fatigue (>0.7→AD) overrides preserved

---

## Batch 1: CDE Foundation — RegulationDefaults + Derivation

### Task 1: Create RegulationDefaults record and CDE derivation pathway

**Files:**
- Create: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/RegulationDefaults.java`
- Modify: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java`
- Modify: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDefaults.java`
- Test: `cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java`

**Interfaces:**
- Produces: `RegulationDefaults(reappraisalCapacity, suppressionCapacity, redirectTendency)` record
- Produces: `CognitiveDerivationEngine.deriveRegulationDefaults(DispositionAxes)` → `RegulationDefaults`
- Produces: `CognitiveDefaults.regulationDefaults()` accessor + `withRegulationDefaults()` builder

- [ ] **Step 1: Write the failing test — weight table derivation for known axes**

Add test method to `CognitiveDerivationEngineTest`:

```java
@Test
void deriveRegulationDefaults_reappraiserProfile() {
    var axes = new DispositionAxes("cooperative", "strict", "calculated", "high", "analytical");
    var rd = CognitiveDerivationEngine.deriveRegulationDefaults(axes);
    assertThat(rd).isNotNull();
    assertThat(rd.reappraisalCapacity()).isGreaterThanOrEqualTo(0.85);
    assertThat(rd.suppressionCapacity()).isLessThan(0.5);
    assertThat(rd.redirectTendency()).isLessThan(0.3);
}

@Test
void deriveRegulationDefaults_suppressorProfile() {
    var axes = new DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant");
    var rd = CognitiveDerivationEngine.deriveRegulationDefaults(axes);
    assertThat(rd).isNotNull();
    assertThat(rd.suppressionCapacity()).isGreaterThanOrEqualTo(0.7);
    assertThat(rd.reappraisalCapacity()).isLessThan(0.5);
    assertThat(rd.redirectTendency()).isGreaterThanOrEqualTo(0.5);
}

@Test
void deriveRegulationDefaults_mixedProfile() {
    var axes = new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "cooperative");
    var rd = CognitiveDerivationEngine.deriveRegulationDefaults(axes);
    assertThat(rd).isNotNull();
    // Mixed: both capacities near threshold, neither dominates
    assertThat(Math.abs(rd.reappraisalCapacity() - rd.suppressionCapacity())).isLessThan(0.20);
}

@Test
void deriveRegulationDefaults_nullAxes() {
    assertThat(CognitiveDerivationEngine.deriveRegulationDefaults(null)).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -Dtest="CognitiveDerivationEngineTest#deriveRegulationDefaults*" -s neocortex/.mvn/slot-settings.xml`
Expected: compilation failure — `deriveRegulationDefaults` doesn't exist, `RegulationDefaults` doesn't exist.

- [ ] **Step 3: Create RegulationDefaults record**

Use `ide_create_file` to create:
```java
package io.casehub.neocortex.cognitive.index;

public record RegulationDefaults(
    double reappraisalCapacity,
    double suppressionCapacity,
    double redirectTendency
) {}
```

- [ ] **Step 4: Implement deriveRegulationDefaults in CognitiveDerivationEngine**

Add static weight table fields (use `ide_insert_member` after the existing `CONFLICT_INTERPRETATION` map):

```java
private static final Map<String, Double> REAPPRAISAL_CONFLICT = Map.of(
    "analytical", 0.30, "cooperative", 0.15, "avoidant", -0.15, "competitive", 0.05);
private static final Map<String, Double> REAPPRAISAL_AUTONOMY = Map.of(
    "high", 0.25, "moderate", 0.05, "low", -0.10);
private static final Map<String, Double> REAPPRAISAL_RULES = Map.of(
    "strict", 0.15, "flexible", 0.10, "moderate", 0.05);
private static final Map<String, Double> REAPPRAISAL_RISK = Map.of(
    "bold", 0.20, "calculated", 0.05, "conservative", -0.10);
private static final Map<String, Double> REAPPRAISAL_SOCIAL = Map.of(
    "cooperative", 0.10, "independent", 0.05, "competitive", 0.05);

private static final Map<String, Double> SUPPRESSION_CONFLICT = Map.of(
    "avoidant", 0.25, "competitive", 0.15, "analytical", -0.10, "cooperative", -0.05);
private static final Map<String, Double> SUPPRESSION_SOCIAL = Map.of(
    "competitive", 0.10, "independent", 0.05, "cooperative", -0.05);
private static final Map<String, Double> SUPPRESSION_RISK = Map.of(
    "conservative", 0.15, "calculated", 0.05, "bold", -0.10);
private static final Map<String, Double> SUPPRESSION_AUTONOMY = Map.of(
    "low", 0.10, "moderate", 0.05, "high", -0.05);
private static final Map<String, Double> SUPPRESSION_RULES = Map.of(
    "strict", 0.05, "moderate", 0.0, "flexible", -0.05);

private static final Map<String, Double> REDIRECT_CONFLICT = Map.of(
    "avoidant", 0.25, "cooperative", 0.05, "analytical", -0.15, "competitive", -0.10);
private static final Map<String, Double> REDIRECT_RISK = Map.of(
    "conservative", 0.20, "calculated", 0.05, "bold", -0.15);
private static final Map<String, Double> REDIRECT_AUTONOMY = Map.of(
    "low", 0.10, "moderate", 0.05, "high", -0.10);
private static final Map<String, Double> REDIRECT_SOCIAL = Map.of(
    "independent", 0.05, "cooperative", 0.0, "competitive", -0.05);
private static final Map<String, Double> REDIRECT_RULES = Map.of(
    "strict", 0.0, "moderate", 0.0, "flexible", 0.05);
```

Add the derivation method (use `ide_insert_member` after `deriveHabituationConfig`):

```java
static RegulationDefaults deriveRegulationDefaults(DispositionAxes axes) {
    if (axes == null) return null;
    double reappraisal = 0.5
        + REAPPRAISAL_CONFLICT.getOrDefault(lower(axes.conflictMode()), 0.0)
        + REAPPRAISAL_AUTONOMY.getOrDefault(lower(axes.autonomy()), 0.0)
        + REAPPRAISAL_RULES.getOrDefault(lower(axes.ruleFollowing()), 0.0)
        + REAPPRAISAL_RISK.getOrDefault(lower(axes.riskAppetite()), 0.0)
        + REAPPRAISAL_SOCIAL.getOrDefault(lower(axes.socialOrient()), 0.0);
    double suppression = 0.5
        + SUPPRESSION_CONFLICT.getOrDefault(lower(axes.conflictMode()), 0.0)
        + SUPPRESSION_SOCIAL.getOrDefault(lower(axes.socialOrient()), 0.0)
        + SUPPRESSION_RISK.getOrDefault(lower(axes.riskAppetite()), 0.0)
        + SUPPRESSION_AUTONOMY.getOrDefault(lower(axes.autonomy()), 0.0)
        + SUPPRESSION_RULES.getOrDefault(lower(axes.ruleFollowing()), 0.0);
    double redirect = 0.3
        + REDIRECT_CONFLICT.getOrDefault(lower(axes.conflictMode()), 0.0)
        + REDIRECT_RISK.getOrDefault(lower(axes.riskAppetite()), 0.0)
        + REDIRECT_AUTONOMY.getOrDefault(lower(axes.autonomy()), 0.0)
        + REDIRECT_SOCIAL.getOrDefault(lower(axes.socialOrient()), 0.0)
        + REDIRECT_RULES.getOrDefault(lower(axes.ruleFollowing()), 0.0);
    return new RegulationDefaults(
        Math.max(0.0, Math.min(1.0, reappraisal)),
        Math.max(0.0, Math.min(1.0, suppression)),
        Math.max(0.0, Math.min(1.0, redirect)));
}
```

- [ ] **Step 5: Wire into CognitiveDefaults**

Add `regulationDefaults` field to `CognitiveDefaults` record (use `ide_edit_member` to update the record fields and constructor). Add `withRegulationDefaults(RegulationDefaults)` builder method following the existing pattern.

Update `CognitiveDerivationEngine.derive()` to call `deriveRegulationDefaults(descriptor.disposition())` and chain `.withRegulationDefaults(regulation)`.

Update `CognitiveDerivationEngine.deriveAndMerge()` to add the merge clause:
```java
.withRegulationDefaults(explicit.regulationDefaults() != null
    ? explicit.regulationDefaults() : derived.regulationDefaults())
```

Update `CognitiveDefaults.empty()` to include `null` for the new field.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -Dtest="CognitiveDerivationEngineTest" -s neocortex/.mvn/slot-settings.xml`
Expected: all tests pass, including the new regulation defaults tests.

- [ ] **Step 7: Verify existing tests still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s neocortex/.mvn/slot-settings.xml`
Expected: all module tests pass — CognitiveDefaults field addition may require updating test fixtures.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognitive-index/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#830): add RegulationDefaults with literature-grounded weight tables to CDE"
```

---

## Batch 2: Predictor Refactor — Continuous Capacity Thresholds

### Task 2: Replace boolean classification with per-category continuous logic

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/RegulationStrategyPredictor.java`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/RegulationStrategyPredictorTest.java`

**Interfaces:**
- Consumes: `RegulationDefaults(reappraisalCapacity, suppressionCapacity, redirectTendency)` from Task 1
- Produces: `predict(RegulationDefaults, DispositionAxes, EmotionType, double, double, CognitiveLoadLevel)` → `Optional<RegulationPrediction>`
- Produces: `dominantAboveThreshold(double, EmotionRegulationStrategy, double, EmotionRegulationStrategy, double, double)` → `@Nullable EmotionRegulationStrategy`

- [ ] **Step 1: Write failing tests for continuous predictor**

Add to `RegulationStrategyPredictorTest`:

```java
@Test
void predict_strongReappraiser_controllableNeg_pinsCognitiveChange() {
    var rd = new RegulationDefaults(0.90, 0.40, 0.20);
    var axes = new DispositionAxes("cooperative", "strict", "calculated", "high", "analytical");
    var result = predictor.predict(rd, axes, EmotionType.ANGER, 0.55, 0.0, null);
    assertThat(result).isPresent();
    assertThat(result.get().strategy()).isEqualTo(EmotionRegulationStrategy.COGNITIVE_CHANGE);
}

@Test
void predict_strongSuppressor_controllableNeg_pinsResponseModulation() {
    var rd = new RegulationDefaults(0.30, 0.80, 0.60);
    var axes = new DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant");
    var result = predictor.predict(rd, axes, EmotionType.ANGER, 0.55, 0.0, null);
    assertThat(result).isPresent();
    assertThat(result.get().strategy()).isEqualTo(EmotionRegulationStrategy.RESPONSE_MODULATION);
}

@Test
void predict_mixedProfile_controllableNeg_fallsThrough() {
    var rd = new RegulationDefaults(0.68, 0.70, 0.45);
    var axes = new DispositionAxes("cooperative", "moderate", "calculated", "moderate", "cooperative");
    var result = predictor.predict(rd, axes, EmotionType.ANGER, 0.55, 0.0, null);
    assertThat(result).isEmpty();
}

@Test
void predict_selfConscious_onlyChecksReappraisal() {
    var rd = new RegulationDefaults(0.30, 0.80, 0.60);
    var axes = new DispositionAxes("cooperative", "moderate", "conservative", "low", "avoidant");
    var result = predictor.predict(rd, axes, EmotionType.SHAME, 0.55, 0.0, null);
    assertThat(result).isEmpty();
}

@Test
void predict_selfConscious_strongReappraiser_pinsCognitiveChange() {
    var rd = new RegulationDefaults(0.90, 0.40, 0.20);
    var axes = new DispositionAxes("cooperative", "strict", "bold", "high", "analytical");
    var result = predictor.predict(rd, axes, EmotionType.SHAME, 0.55, 0.0, null);
    assertThat(result).isPresent();
    assertThat(result.get().strategy()).isEqualTo(EmotionRegulationStrategy.COGNITIVE_CHANGE);
}

@Test
void predict_highIntensity_overrides_toAD() {
    var rd = new RegulationDefaults(0.90, 0.40, 0.20);
    var axes = new DispositionAxes("cooperative", "strict", "calculated", "high", "analytical");
    var result = predictor.predict(rd, axes, EmotionType.ANGER, 0.80, 0.0, null);
    assertThat(result).isPresent();
    assertThat(result.get().strategy()).isEqualTo(EmotionRegulationStrategy.ATTENTIONAL_DEPLOYMENT);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="RegulationStrategyPredictorTest" -s neocortex/.mvn/slot-settings.xml`
Expected: compilation failure — `predict` doesn't accept `RegulationDefaults`.

- [ ] **Step 3: Implement the refactored predictor**

Use `ide_replace_member` to replace the `predict` method signature. Add `dominantAboveThreshold` helper. Replace `predictFromEmotionAndPersonality` internals: `predictForControllableNegative` uses `dominantAboveThreshold(rd.reappraisalCapacity(), CC, rd.suppressionCapacity(), RM, ...)`. `predictForSelfConscious` checks only `rd.reappraisalCapacity() >= REAPPRAISAL_THRESHOLD`. `predictForOtherDirected` uses reversed order. Preserve `hasCompetitiveAvoidantConflict(axes)` guard. Remove `isSuppressor`, `isReappraiser`, `suppressorSignals`, `ccEnablingSignals` methods. Add threshold constants: `REAPPRAISAL_THRESHOLD = 0.65`, `SUPPRESSION_THRESHOLD = 0.65`, `DOMINANCE_MARGIN = 0.10`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest="RegulationStrategyPredictorTest" -s neocortex/.mvn/slot-settings.xml`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#830): replace boolean predictor classification with continuous capacity thresholds"
```

### Task 3: Wire tick participant to pass RegulationDefaults to predictor

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java`

**Interfaces:**
- Consumes: `predict(RegulationDefaults, DispositionAxes, ...)` from Task 2
- Consumes: `CognitiveDefaultsRegistry.forAgent(agentId)` → `Optional<CognitiveDefaults>` with `regulationDefaults()` from Task 1

- [ ] **Step 1: Update tryPredictStrategy to use RegulationDefaults**

Use `ide_replace_member` on `tryPredictStrategy`. Current logic retrieves `DispositionAxes` and calls `predictor.predict(axes, emotion, ...)`. New logic: retrieve `CognitiveDefaults` from `defaultsRegistry.forAgent(agentId)`, extract both `regulationDefaults()` and `descriptor().disposition()`, call `predictor.predict(rd, axes, emotion, ...)`. If `regulationDefaults` is null, fall through (return null).

- [ ] **Step 2: Run cognition module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s neocortex/.mvn/slot-settings.xml`
Expected: all pass. Existing tests that construct `EmotionRegulationTickParticipant` may need `CognitiveDefaultsRegistry` stubs updated.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#830): wire tick participant to pass RegulationDefaults to predictor"
```

---

## Batch 3: ShameContext — Runtime Shame Dimensions

### Task 4: Create ShameContext and assemble in tick participant

**Files:**
- Create: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ShameContext.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/AppraisalTickParticipant.java` (extract person overlay lookup)
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/ShameContextTest.java`

**Interfaces:**
- Consumes: `MindMapStore` (regulation history nodes), `CognitiveDefaultsRegistry` (PersonalityFacets)
- Produces: `ShameContext(episodeCount, mostRecentEpisode, relationalCloseness, relationalRole, grandiosity, callousness)`
- Produces: `ShameContext.isCompounding()`, `.isRelationallyCharged()`, `.isSurfaceLevel()`

- [ ] **Step 1: Write failing tests for ShameContext**

Create `ShameContextTest.java`:

```java
@Test
void isCompounding_trueWhenThreeOrMore() {
    var ctx = new ShameContext(3, Instant.now(), 0.0, null, 0.3, 0.2);
    assertThat(ctx.isCompounding()).isTrue();
}

@Test
void isCompounding_falseWhenLessThanThree() {
    var ctx = new ShameContext(2, Instant.now(), 0.0, null, 0.3, 0.2);
    assertThat(ctx.isCompounding()).isFalse();
}

@Test
void isRelationallyCharged_trueWhenHighIntimacy() {
    var ctx = new ShameContext(0, null, 0.6, "mentor", 0.3, 0.2);
    assertThat(ctx.isRelationallyCharged()).isTrue();
}

@Test
void isSurfaceLevel_trueWhenHighGrandiositAndCallousness() {
    var ctx = new ShameContext(0, null, 0.0, null, 0.8, 0.8);
    assertThat(ctx.isSurfaceLevel()).isTrue();
}

@Test
void isSurfaceLevel_falseWhenCallousnessBelowThreshold() {
    var ctx = new ShameContext(0, null, 0.0, null, 0.8, 0.6);
    assertThat(ctx.isSurfaceLevel()).isFalse();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: compilation failure — `ShameContext` doesn't exist.

- [ ] **Step 3: Create ShameContext record**

Use `ide_create_file`:

```java
package io.casehub.neocortex.cognition.regulation;

import org.jspecify.annotations.Nullable;
import java.time.Instant;

public record ShameContext(
    int episodeCount,
    @Nullable Instant mostRecentEpisode,
    double relationalCloseness,
    @Nullable String relationalRole,
    double grandiosity,
    double callousness
) {
    public boolean isCompounding() {
        return episodeCount >= 3;
    }

    public boolean isRelationallyCharged() {
        return relationalCloseness >= 0.5;
    }

    public boolean isSurfaceLevel() {
        return grandiosity >= 0.7 && callousness >= 0.7;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Expected: all pass.

- [ ] **Step 5: Extract person overlay lookup from AppraisalTickParticipant**

Extract the `resolveRelationalSchema()` method from `AppraisalTickParticipant` to a shared static utility — either `RelationalSchema.resolve(MindMapStore, String observerAgentId, String targetAgentId, String tenantId)` or a new `PersonOverlayLookup` class. Use `ide_find_references` to verify no other callers of the private method, then extract. Both `AppraisalTickParticipant` and `EmotionRegulationTickParticipant` call the extracted utility.

- [ ] **Step 6: Add ShameContext assembly to tick participant**

Add `assembleShameContext(String agentId, String tenantId, CognitiveEmotion dominantEmotion)` method to `EmotionRegulationTickParticipant`. Call it when `emotionCategory(dominantEmotion.type()) == SELF_CONSCIOUS`, regardless of whether the predictor pinned a strategy. Steps:
1. Count recent shame regulation entries from MindMap (30-minute window)
2. If subjectId != agentId, look up person overlay for relational closeness
3. Look up grandiosity + callousness from `CognitiveDefaultsRegistry`
4. Return `ShameContext`

Wire the assembled ShameContext into the `generate()` call chain (new parameter).

- [ ] **Step 7: Run cognition module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s neocortex/.mvn/slot-settings.xml`
Expected: all pass.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#830): add ShameContext with memory compounding, relational closeness, personality-gated depth"
```

---

## Batch 4: Generation Phase — Interpretive Rendering

### Task 5: Add RegulationDefaults and ShameContext rendering to buildUserMessage

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java`

**Interfaces:**
- Consumes: `RegulationDefaults` from Task 1
- Consumes: `ShameContext` from Task 4
- Produces: Extended `buildUserMessage()` with RegulationDefaults capacity summary and ShameContext interpretive sections

- [ ] **Step 1: Add RegulationDefaults rendering**

Add a new overload of `generate()` and `invokeLlm()` that accepts `@Nullable RegulationDefaults` and `@Nullable ShameContext`. Add a capacity summary section in `buildUserMessage()` — rendered only on fall-through (pinnedStrategy == null) when `RegulationDefaults` are available. Use the interpretive pattern: map capacity values to low/moderate/strong labels with personality-grounded explanation sentences.

- [ ] **Step 2: Add ShameContext rendering**

Add three conditional sections to `buildUserMessage()` for ShameContext, rendered for ALL self-conscious emotions (pinned or fall-through):
1. Memory compounding section (when `isCompounding()`)
2. Relational charging section (when `isRelationallyCharged()`)
3. Surface-level gating section (when `isSurfaceLevel()`)

**Precedence rule:** When `isSurfaceLevel()` is true, do NOT render Phase 1's temporal containment guidance. Add `if (!shameContext.isSurfaceLevel())` guard around the existing temporal containment block.

- [ ] **Step 3: Wire through tick participant and generate overloads**

Add the new parameters to the `generate()` → `invokeLlm()` → `buildUserMessage()` chain. The tick participant passes `RegulationDefaults` and `ShameContext` (both nullable) through the generation phase.

- [ ] **Step 4: Run cognition module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s neocortex/.mvn/slot-settings.xml`
Expected: all pass — existing tests use older generate() overloads which still work.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#830): add interpretive rendering for RegulationDefaults and ShameContext"
```

---

## Batch 5: Eval Extension + Calibration

### Task 6: Add shame-specific eval cases and run calibration

**Files:**
- Modify: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/RegulationEndToEndEvalTest.java`

**Interfaces:**
- Consumes: Full pipeline from Tasks 1-5
- Produces: Extended eval with 3 shame-specific cases + shame depth judge + calibration results

- [ ] **Step 1: Add shame depth judge system prompt**

Add `SHAME_DEPTH_JUDGE` constant with the prompt from the spec (scores 1-5 for depth appropriateness).

- [ ] **Step 2: Add shame-specific test cases**

Add three new JudgeCase entries with ShameContext configuration:
1. `compounding-shame` — suppressor with 4 prior episodes, stranger subject, normal personality depth
2. `relational-shame` — reappraiser with 1 prior episode, mentor subject (intimacy 0.8), normal depth
3. `surface-shame` — avoidant with 0 prior episodes, stranger, grandiosity 0.8 + callousness 0.8

Each case needs: axes, emotion, ShameContext fields, expected predictor behavior, and judge criteria.

- [ ] **Step 3: Update test method to pass RegulationDefaults and ShameContext**

Update `llmRegulationStrategiesAreAppropriate()` to derive `RegulationDefaults` from `CognitiveDerivationEngine.deriveRegulationDefaults(tc.axes())` and pass them to the predictor and generation phase. For shame-specific cases, construct the `ShameContext` and pass it to the generation phase.

- [ ] **Step 4: Add shame depth scoring**

For shame-specific cases, add a judge call using `SHAME_DEPTH_JUDGE` that scores depth appropriateness. Record scores in the results JSON.

- [ ] **Step 5: Run the extended eval**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/caps-llm-classifier/pom.xml test -Pllm-eval -Dtest="RegulationEndToEndEvalTest" -s neocortex/.mvn/slot-settings.xml`
Expected: all cases produce results. Strategy mean >= 4.0. Shame depth scores >= 3.0.

- [ ] **Step 6: Review results and calibrate if needed**

Read `target/regulation-e2e-judge-results.json`. Check:
1. Do the original 8 cases maintain >= 4.4 mean?
2. Do the 3 shame cases score >= 4 on strategy appropriateness?
3. Do the shame depth scores show context-awareness?
4. Are any previously-pinned cases now falling through (or vice versa)?

Adjust thresholds or weights if scores indicate miscalibration.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add caps-llm-classifier/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "eval(#830): extend end-to-end eval with shame-specific cases and depth judge"
```

---

## References

- [2026-11-07-regulation-phase2-continuous-weights-design.md] — reviewed design spec
- [neocortex/docs/research/2026-11-07-regulation-continuous-weights-literature-review.md] — Barańczuk 2019 weight table derivation
- [cognitive-index/.../CognitiveDerivationEngine.java:141-167] — derive() pattern
- [cognitive-index/.../CognitiveDefaults.java:31-151] — record structure, with* builders
- [cognition/.../regulation/RegulationStrategyPredictor.java] — current boolean classification
- [cognition/.../regulation/EmotionRegulationTickParticipant.java] — tick pipeline wiring
- [cognition/.../regulation/EmotionRegulationGenerationPhase.java:201-312] — buildUserMessage conditional sections
- [caps-llm-classifier/.../eval/RegulationEndToEndEvalTest.java] — end-to-end eval baseline
- [GitHub #830] — regulation predictor quality
