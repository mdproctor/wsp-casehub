# Emotion Regulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#590 — Emotion regulation: post-appraisal strategy selection (Gross 1998)
**Issue group:** casehubio/neocortex#590

**Goal:** Add a post-appraisal emotion regulation system that selects from all five Gross strategy families and modulates character drive rendering, using LLM-driven regulation with literature-structured prompting.

**Architecture:** A new `EmotionRegulationTickParticipant` in the DERIVED phase reads `AppraisalResult`, gates on a discrepancy threshold, makes an LLM call (per-tick, `appraiseMood` pattern) to select a regulation strategy and produce narrative, then writes `EmotionRegulationResult` for downstream consumption by `renderInternalState()` (internal strategies) and a new `EmotionRegulationGoalParticipant` in TERMINAL phase (environmental strategies).

**Tech Stack:** Java 26, Quarkus, neocortex cognition module, MindMap, SessionPipeline

## Global Constraints

- All regulation constants, thresholds, and strategy mappings must cite published literature in code comments per protocol PP-20261017-d5853d
- All new types use `EmotionRegulation` prefix to avoid collision with existing `RegulationTracker` (interpersonal)
- `CognitionConfig` record is positional — adding `regulationEnabled` as 25th field requires updating ALL static factory methods and `with()` switch
- Per-tick LLM calls must have fallback logic for failure/timeout (cf. `MOOD_FALLBACK_DECAY`)
- Neocortex slot: `/Users/mdproctor/claude/casehub/slots/211/neocortex`; examples slot: `/Users/mdproctor/claude/casehub/slots/196/examples`

---

## Batch 1: Data Model and CognitionConfig

### Task 1: EmotionRegulationStrategy enum and EmotionRegulationResult record

**Files:**
- Create: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationStrategy.java`
- Create: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationResult.java`
- Test: `cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationResultTest.java`

**Interfaces:**
- Consumes: `DriveAxis` from `io.casehub.neocortex.cognition.drive.DriveAxis`
- Produces: `EmotionRegulationStrategy` (enum with 5 values), `EmotionRegulationResult` (record with `strategy`, `targetEmotions`, `targetDrives`, `peakOriginalIntensity`, `regulatedIntensity`, `effectiveness`, `narrative`, `regulationAxis`, `environmentalDirective`, `isEnvironmentalStrategy()`)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.drive.DriveAxis;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class EmotionRegulationResultTest {

    @Test
    void internalStrategyHasNoEnvironmentalFields() {
        var result = new EmotionRegulationResult(
                EmotionRegulationStrategy.RESPONSE_MODULATION,
                List.of("anger"), List.of("gloating"),
                0.85, 0.4, 0.6,
                "You hold back the triumph", null, null);
        assertFalse(result.isEnvironmentalStrategy());
        assertNull(result.regulationAxis());
        assertNull(result.environmentalDirective());
    }

    @Test
    void environmentalStrategyHasAxisAndDirective() {
        var result = new EmotionRegulationResult(
                EmotionRegulationStrategy.SITUATION_SELECTION,
                List.of("anxiety"), List.of(),
                0.9, 0.5, 0.7,
                "You decide to leave", DriveAxis.AFFILIATION,
                "Avoid being alone with Muttley");
        assertTrue(result.isEnvironmentalStrategy());
        assertEquals(DriveAxis.AFFILIATION, result.regulationAxis());
        assertEquals("Avoid being alone with Muttley", result.environmentalDirective());
    }

    @Test
    void allFiveStrategiesExist() {
        assertEquals(5, EmotionRegulationStrategy.values().length);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest="EmotionRegulationResultTest" -s .mvn/slot-settings.xml`
Expected: FAIL — classes do not exist

- [ ] **Step 3: Implement EmotionRegulationStrategy enum**

```java
package io.casehub.neocortex.cognition.regulation;

// Gross 1998 five-strategy process model — situation → attention → appraisal → response
public enum EmotionRegulationStrategy {
    SITUATION_SELECTION,       // Gross strategy 1 (Gross 1998) — avoid triggering contexts
    SITUATION_MODIFICATION,    // Gross strategy 2 (Gross 1998) — change the situation
    ATTENTIONAL_DEPLOYMENT,    // Gross strategy 3 (Gross 1998) — redirect focus (distraction)
    COGNITIVE_CHANGE,          // Gross strategy 4 (Gross 1998) — reappraise meaning
    RESPONSE_MODULATION        // Gross strategy 5 (Gross 1998) — suppress/amplify expression
}
```

- [ ] **Step 4: Implement EmotionRegulationResult record**

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.drive.DriveAxis;
import jakarta.annotation.Nullable;
import java.util.List;

public record EmotionRegulationResult(
    EmotionRegulationStrategy strategy,
    List<String> targetEmotions,
    List<String> targetDrives,
    double peakOriginalIntensity,
    double regulatedIntensity,
    double effectiveness,
    String narrative,
    @Nullable DriveAxis regulationAxis,
    @Nullable String environmentalDirective
) {
    // Gross 1998 — strategies 1-2 target the environment, 3-5 target internal state
    public boolean isEnvironmentalStrategy() {
        return strategy == EmotionRegulationStrategy.SITUATION_SELECTION
            || strategy == EmotionRegulationStrategy.SITUATION_MODIFICATION;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest="EmotionRegulationResultTest" -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/
git add cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/
git commit -m "feat(neocortex#590): add EmotionRegulationStrategy and EmotionRegulationResult data model"
```

### Task 2: Add regulationEnabled to CognitionConfig

**Files:**
- Modify: `cognition-api/src/main/java/io/casehub/neocortex/cognition/core/CognitionConfig.java:40-116`
- Modify: `cognition-api/src/test/java/io/casehub/neocortex/cognition/core/CognitionConfigTest.java`

**Interfaces:**
- Consumes: existing `CognitionConfig` record (24 boolean fields)
- Produces: `CognitionConfig` with 25th field `regulationEnabled`, updated `all()`, `withAllSubsystems()`, `none()`, `with("regulation", bool)`, `without("regulation")`

- [ ] **Step 1: Write the failing test**

```java
@Test
void regulationEnabledInAllConfig() {
    assertTrue(CognitionConfig.all().regulationEnabled());
}

@Test
void regulationDisabledInNoneConfig() {
    assertFalse(CognitionConfig.none().regulationEnabled());
}

@Test
void withRegulationSwitch() {
    var config = CognitionConfig.none().with("regulation", true);
    assertTrue(config.regulationEnabled());
}

@Test
void withoutRegulation() {
    var config = CognitionConfig.all().without("regulation");
    assertFalse(config.regulationEnabled());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest="CognitionConfigTest" -s .mvn/slot-settings.xml`
Expected: FAIL — `regulationEnabled()` method does not exist

- [ ] **Step 3: Add regulationEnabled field to CognitionConfig record**

Add `boolean regulationEnabled` as the 25th field in the record constructor. Update:
- `all()` — pass `true` for `regulationEnabled`
- `withAllSubsystems()` — pass `true`
- `none()` — pass `false`
- `withDirectives()` — pass `false`
- `with()` switch — add `case "regulation" -> new CognitionConfig(... regulationEnabled: enabled ...)`
- `without()` — handle "regulation" case

Use `ide_replace_member` for the record constructor and each static method.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest="CognitionConfigTest" -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Fix compilation across modules**

Adding a field to the positional record breaks ALL callers. Run:
`JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f pom.xml compile -s .mvn/slot-settings.xml`

Use `ide_find_references` on the `CognitionConfig` constructor to find all call sites and update them with the new `regulationEnabled` parameter.

- [ ] **Step 6: Run full cognition-api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add cognition-api/
git commit -m "feat(neocortex#590): add regulationEnabled to CognitionConfig (25th field)"
```

---

## Batch 2: EmotionRegulationTickParticipant and LLM Generation

### Task 3: EmotionRegulationTickParticipant with activation gate

**Files:**
- Create: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipantTest.java`

**Interfaces:**
- Consumes: `AppraisalTickParticipant.currentResult(agentId, tenantId)` → `Optional<AppraisalResult>`, `CognitiveEmotion.intensity()`, `CognitionConfig.regulationEnabled()`
- Produces: `EmotionRegulationTickParticipant` implementing `CognitionTickParticipant`, with `currentResult(agentId, tenantId)` → `Optional<EmotionRegulationResult>`, `tick(CognitionTickContext)`

- [ ] **Step 1: Write the failing test — activation gate**

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.appraisal.AppraisalResult;
import io.casehub.neocortex.cognition.appraisal.AppraisalTickParticipant;
import io.casehub.neocortex.cognitive.CognitiveEmotion;
import io.casehub.neocortex.cognitive.EmotionType;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class EmotionRegulationTickParticipantTest {

    @Test
    void noRegulationWhenBelowThreshold() {
        // emotion intensity 0.3 — below default threshold 0.7 (baseline 0.3 + threshold 0.4)
        var emotion = new CognitiveEmotion(EmotionType.JOY, 0.3, "subject1",
                Instant.now(), null, null);
        var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);
        // ... test that regulation does not fire (currentResult is empty)
    }

    @Test
    void regulationFiresWhenAboveThreshold() {
        // emotion intensity 0.85 — above default threshold
        var emotion = new CognitiveEmotion(EmotionType.ANGER, 0.85, "subject1",
                Instant.now(), null, null);
        var appraisal = new AppraisalResult(List.of(emotion), List.of(), null);
        // ... test that regulation fires (currentResult is present)
    }

    @Test
    void noRegulationWhenAppraisalEmpty() {
        var appraisal = AppraisalResult.empty();
        // ... test that regulation does not fire
    }
}
```

Note: Full test implementation requires mocking `AppraisalTickParticipant.currentResult()` and `SessionPipeline`. Use the existing test patterns from `AppraisalTickParticipantTest` for setup.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationTickParticipantTest" -s .mvn/slot-settings.xml`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement EmotionRegulationTickParticipant**

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.appraisal.AppraisalTickParticipant;
import io.casehub.neocortex.cognition.core.CognitionConfig;
import io.casehub.neocortex.cognition.core.CognitionTickContext;
import io.casehub.neocortex.cognition.core.CognitionTickParticipant;
import io.casehub.neocortex.cognitive.CognitiveEmotion;
import io.casehub.neocortex.mindmap.MindMapStore;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

public class EmotionRegulationTickParticipant implements CognitionTickParticipant {

    private final AppraisalTickParticipant appraisalParticipant;
    private final MindMapStore mindMapStore;
    private final CognitionConfig config;
    // Bosse et al. 2010 — homeostatic baseline intensity
    private static final double BASELINE_INTENSITY = 0.3;
    // Starting threshold for empirical tuning
    private static final double ACTIVATION_THRESHOLD = 0.4;

    private final ConcurrentHashMap<String, EmotionRegulationResult> results = new ConcurrentHashMap<>();

    public EmotionRegulationTickParticipant(AppraisalTickParticipant appraisalParticipant,
                                            MindMapStore mindMapStore,
                                            CognitionConfig config) {
        this.appraisalParticipant = appraisalParticipant;
        this.mindMapStore = mindMapStore;
        this.config = config;
    }

    @Override
    public void tick(CognitionTickContext context) {
        if (!config.regulationEnabled() || !config.appraisalEnabled()) return;

        var appraisal = appraisalParticipant.currentResult(
                context.agentId(), context.tenantId());
        if (appraisal.isEmpty()) return;

        double maxDiscrepancy = appraisal.get().emotions().stream()
                .mapToDouble(e -> Math.abs(e.intensity() - BASELINE_INTENSITY))
                .max().orElse(0.0);

        if (maxDiscrepancy <= ACTIVATION_THRESHOLD) return;

        // LLM generation call — implemented in Task 4
        var result = generateRegulation(context, appraisal.get());
        if (result != null) {
            String key = context.agentId() + ":" + context.tenantId();
            results.put(key, result);
            writeToMindMap(context, result);
        }
    }

    public Optional<EmotionRegulationResult> currentResult(String agentId, String tenantId) {
        return Optional.ofNullable(results.get(agentId + ":" + tenantId));
    }

    // Placeholder for Task 4
    EmotionRegulationResult generateRegulation(CognitionTickContext context,
                                                io.casehub.neocortex.cognition.appraisal.AppraisalResult appraisal) {
        return null;
    }

    private void writeToMindMap(CognitionTickContext context, EmotionRegulationResult result) {
        // MindMap persistence — implemented in Task 5
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationTickParticipantTest" -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/
git commit -m "feat(neocortex#590): add EmotionRegulationTickParticipant with activation gate"
```

### Task 4: LLM generation phase (regulation system prompt + JSON parsing)

**Files:**
- Create: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhase.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java` — wire generation
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGenerationPhaseTest.java`

**Interfaces:**
- Consumes: `AppraisalResult`, `SessionPipeline`, personality data from MindMap (`CognitiveDefaultsRegistry`)
- Produces: `EmotionRegulationGenerationPhase.generate(context, appraisal, regulationHistory)` → `EmotionRegulationResult`

- [ ] **Step 1: Write the failing test — JSON parsing**

Test that the generation phase correctly parses a valid LLM JSON response into `EmotionRegulationResult`. Mock the `SessionPipeline` to return a known JSON string. Test the system prompt contains the five Gross strategy descriptions, Petter strategy selection factors, and personality mappings.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest" -s .mvn/slot-settings.xml`
Expected: FAIL

- [ ] **Step 3: Implement EmotionRegulationGenerationPhase**

Follow the `CognitionCore.appraiseMood()` pattern:
1. Build system prompt with Gross strategy descriptions, Petter selection factors, personality mappings
2. Build user prompt with current emotions, drive intensities, regulation history
3. Call `sessionPipeline.openSession()` with the prompts
4. Parse JSON response into `EmotionRegulationResult`
5. Fallback: return `null` on parse failure or timeout (caller treats null as no-regulation)

The system prompt must include literature citations per protocol PP-20261017-d5853d.

- [ ] **Step 4: Wire generation into EmotionRegulationTickParticipant**

Replace the placeholder `generateRegulation()` method with a call to `EmotionRegulationGenerationPhase.generate()`.

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="EmotionRegulationGenerationPhaseTest" -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/
git commit -m "feat(neocortex#590): add EmotionRegulationGenerationPhase with LLM call and JSON parsing"
```

### Task 5: MindMap persistence and regulation history retrieval

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java` — implement `writeToMindMap()` and add `loadRegulationHistory()`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipantTest.java` — add history tests

**Interfaces:**
- Consumes: `MindMapStore.save()`, `MindMapStore.search(MindMapQuery)`, `SubgraphTypes.COGNITIVE`
- Produces: `writeToMindMap(context, result)` persists regulation result as MindMap node; `loadRegulationHistory(agentId, tenantId, limit)` retrieves recent regulation nodes

- [ ] **Step 1: Write the failing test — MindMap persistence round-trip**

Write `EmotionRegulationResult`, persist via `writeToMindMap()`, retrieve via `loadRegulationHistory()`, verify all fields survive the round-trip. Use the MindMap node property schema from the spec (cognitiveKind, agent-id, timestamp, strategy, etc.).

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `writeToMindMap()` is a no-op placeholder

- [ ] **Step 3: Implement writeToMindMap and loadRegulationHistory**

`writeToMindMap()` creates a MindMap node in `SubgraphTypes.COGNITIVE` with all properties from the spec schema. `loadRegulationHistory()` queries `MindMapStore.search()` with `MindMapQuery.of(tenantId, limit).withType(SubgraphTypes.COGNITIVE)`, filters by `cognitiveKind = "emotion-regulation-result"` and `agent-id`, sorts by `timestamp` descending in-memory, limits to history window.

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/
git commit -m "feat(neocortex#590): add MindMap persistence and history retrieval for regulation results"
```

---

## Batch 3: CognitionCore Wiring

### Task 6: Wire EmotionRegulationTickParticipant into CognitionCore

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:72+` — add `configureRegulation()`, `regulationParticipant()` accessor, private volatile field
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/CognitionDefaultBeans.java:33+` — call `configureRegulation()` after `configureAppraisal()`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationOrderingTest.java`

**Interfaces:**
- Consumes: `EmotionRegulationTickParticipant`, `AppraisalTickParticipant`, `SessionPipeline`, `MindMapStore`
- Produces: `CognitionCore.configureRegulation(appraisalParticipant, sessionPipeline, mindMapStore)`, `CognitionCore.regulationParticipant()` → `@Nullable EmotionRegulationTickParticipant`

- [ ] **Step 1: Write the failing test — participant ordering**

```java
@Test
void regulationParticipantRegisteredAfterAppraisal() {
    // Build a CognitionCore, configure appraisal, configure regulation
    // Get the DERIVED phase participant list
    // Assert that EmotionRegulationTickParticipant appears AFTER AppraisalTickParticipant
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `configureRegulation()` does not exist

- [ ] **Step 3: Implement configureRegulation() and regulationParticipant() accessor**

Add to `CognitionCore`:
- `private volatile EmotionRegulationTickParticipant regulationParticipant;`
- `public void configureRegulation(...)` — creates participant, calls `addParticipant(CognitionPhase.DERIVED, ...)`
- `public @Nullable EmotionRegulationTickParticipant regulationParticipant()` — accessor

Add to `CognitionDefaultBeans`:
- Call `configureRegulation()` after `configureAppraisal()`, guarded by `config.regulationEnabled()`

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Run full cognition module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS (no regressions)

- [ ] **Step 6: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/
git add cognition/src/test/java/io/casehub/neocortex/cognition/
git commit -m "feat(neocortex#590): wire EmotionRegulationTickParticipant into CognitionCore DERIVED phase"
```

---

## Batch 4: Drive Rendering Integration

### Task 7: Regulation-aware drive rendering in CognitionObservationBuilder

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilder.java:27+` — add regulation post-processing in `renderInternalState()`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilderRegulationTest.java`

**Interfaces:**
- Consumes: `CognitionCore.regulationParticipant().currentResult(agentId, tenantId)`, `EmotionRegulationResult.targetDrives()`, `EmotionRegulationResult.narrative()`
- Produces: Modified `ObservationSection` parts list with regulation section and regulation-colored drive text

- [ ] **Step 1: Write the failing test — regulation section and modified drives**

Test that when a regulation result is present:
1. A "Regulation" `ObservationSection` appears in the rendered parts
2. Drive descriptions for drives in `targetDrives` include appended regulation narrative
3. Drive descriptions for drives NOT in `targetDrives` are unchanged

Test that when no regulation result is present, rendering is unchanged from current behavior.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — no regulation logic in `renderInternalState()`

- [ ] **Step 3: Implement regulation post-processing in renderInternalState()**

After `resolveRelationalTargets()` produces the `TargetedDrive` list and before final formatting:
1. Read `EmotionRegulationResult` via `cognitionCore.regulationParticipant().currentResult()` — null-safe check
2. For drives in `targetDrives`, modify the description lambda to append regulation narrative text
3. Add a "Regulation" `ObservationSection` to the `parts` list with strategy, target, effectiveness, narrative

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS

- [ ] **Step 5: Run full observation builder tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="CognitionObservationBuilder*" -s .mvn/slot-settings.xml`
Expected: PASS (no regressions)

- [ ] **Step 6: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/
git add cognition/src/test/java/io/casehub/neocortex/cognition/prompt/observation/
git commit -m "feat(neocortex#590): add regulation-aware drive rendering and Regulation ObservationSection"
```

---

## Batch 5: Environmental Strategy Goal Integration

### Task 8: EmotionRegulationGoalParticipant for situation selection/modification

**Files:**
- Create: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipant.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java` — add `configureRegulationGoals()`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/CognitionDefaultBeans.java` — call `configureRegulationGoals()` before `configureDriveGoalBridge()`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipantTest.java`

**Interfaces:**
- Consumes: `EmotionRegulationTickParticipant.currentResult()`, `GoalProposalOrchestrator.registeredGoals()`, `GoalProposalOrchestrator.registerGoals()`
- Produces: `EmotionRegulationGoalParticipant` implementing `CognitionTickParticipant`, registered in TERMINAL phase before `DriveGoalBridgeParticipant`

- [ ] **Step 1: Write the failing test**

Test that when regulation result is SITUATION_SELECTION:
1. A `DriveGoalProposal` is created with the regulation's `regulationAxis` and `environmentalDirective`
2. The proposal is registered via `GoalProposalOrchestrator.registerGoals()`
3. Existing registered goals are preserved (not replaced)

Test that when regulation result is RESPONSE_MODULATION (internal strategy), no goal is registered.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL

- [ ] **Step 3: Implement EmotionRegulationGoalParticipant**

Follow the spec's code from the Environmental Strategies section. The `toGoalProposal()` method constructs a `DriveGoalProposal` from the regulation result.

- [ ] **Step 4: Wire into CognitionCore and CognitionDefaultBeans**

Add `configureRegulationGoals()` to `CognitionCore` — registers `EmotionRegulationGoalParticipant` in TERMINAL phase. Call from `CognitionDefaultBeans` BEFORE `configureDriveGoalBridge()`.

- [ ] **Step 5: Run test to verify it passes**

Expected: PASS

- [ ] **Step 6: Run full cognition module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/
git add cognition/src/main/java/io/casehub/neocortex/cognition/core/
git add cognition/src/main/java/io/casehub/neocortex/cognition/CognitionDefaultBeans.java
git add cognition/src/test/java/io/casehub/neocortex/cognition/regulation/
git commit -m "feat(neocortex#590): add EmotionRegulationGoalParticipant for environmental strategies"
```

---

## Batch 6: Wacky Manor Integration and Eval

### Task 9: Wacky Manor configuration and integration

**Files:**
- Modify: `wacky-manor/src/main/resources/application.properties` (in examples repo) — add `manor.cognition.regulation.enabled` and `manor.cognition.regulation.threshold` properties
- Modify: wacky-manor `CharacterCognition` or equivalent configuration class — wire regulation config
- Test: manual scenario run (A/B comparison)

**Interfaces:**
- Consumes: `CognitionConfig.regulationEnabled()`, neocortex regulation module
- Produces: regulation toggle config properties, wired into wacky-manor character cognition

- [ ] **Step 1: Add configuration properties to application.properties**

```properties
manor.cognition.regulation.enabled=true
manor.cognition.regulation.threshold=0.4
```

- [ ] **Step 2: Wire configuration into CognitionConfig construction**

Find where `CognitionConfig` is constructed in wacky-manor (likely in a CDI producer or config class). Add `regulationEnabled` from the config property.

- [ ] **Step 3: Verify build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run fast tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pfast -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/
git commit -m "feat(#590): wire emotion regulation configuration in wacky-manor"
```

### Task 10: A/B evaluation baseline run

**Files:**
- Test: manual scenario runs, saved to `docs/eval/`

- [ ] **Step 1: Baseline run (regulation disabled)**

Start wacky-manor with `manor.cognition.regulation.enabled=false` and GENERIC profile. Run a full scenario (300+ events). Save transcript to `docs/eval/regulation-baseline/transcript.json`.

- [ ] **Step 2: Regulation run (regulation enabled)**

Start wacky-manor with `manor.cognition.regulation.enabled=true` and GENERIC profile. Run the same scenario. Save transcript to `docs/eval/regulation-enabled/transcript.json`.

- [ ] **Step 3: Classify both runs**

```bash
python3 wacky-manor/docs/eval/classify_emotions.py docs/eval/regulation-baseline
python3 wacky-manor/docs/eval/classify_emotions.py docs/eval/regulation-enabled
```

- [ ] **Step 4: Compare scores, check regression gate**

Existing drive expression scores must not regress beyond threshold. Document regulation quality observations.

- [ ] **Step 5: Commit eval results**

```bash
git add docs/eval/regulation-baseline/ docs/eval/regulation-enabled/
git commit -m "eval(neocortex#590): A/B comparison baseline vs regulation-enabled runs"
```

---

## References

- [2026-11-04-emotion-regulation-design.md] — design spec this plan implements
- [decisions.md] — 12 design decisions (D1-D12) captured during brainstorming
- `cognition-api/src/main/java/io/casehub/neocortex/cognition/core/CognitionConfig.java:40` — 24-field record, `with()` switch
- `cognition-api/src/main/java/io/casehub/neocortex/cognition/core/CognitionTickParticipant.java:4` — interface with `tick(CognitionTickContext)`
- `cognition-api/src/main/java/io/casehub/neocortex/cognition/appraisal/AppraisalResult.java:7` — record with `emotions()`, `actionTendencies()`, `updatedHabituation()`
- `cognition-api/src/main/java/io/casehub/neocortex/cognition/goal/DriveGoalProposal.java:10` — record with `axis`, `goalName`, `goalDescription`, etc.
- `cognitive-api/src/main/java/io/casehub/neocortex/cognitive/CognitiveEmotion.java:6` — record with `type()`, `intensity()`, `subjectId()`, `pad()`
- `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:72` — tick phase architecture, `configureAppraisal()` pattern
- `cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/AppraisalTickParticipant.java:28` — `currentResult()` in-memory accessor pattern
- `cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilder.java:27` — `renderInternalState()` flow
- `cognition/src/main/java/io/casehub/neocortex/cognition/CognitionDefaultBeans.java:33` — factory wiring
- `cognition/src/main/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestrator.java:32` — goal registration
- Protocol PP-20261017-d5853d — literature citation requirement
- casehubio/neocortex#590 — focal issue
- Decision review: `/Users/mdproctor/reviews/casehub-slots/issue-590-decision-20261104-015751/`
- Spec review: `/Users/mdproctor/reviews/casehub-slots/issue-590-spec-20261104-022141/`
