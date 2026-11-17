# Situational Personality Modulation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#712 — Shadow archetypes: situation-dependent personality degradation profiles
**Issue group:** #712 (parent), #1055 (Part 0a prerequisite), casehubio/eidos#234 (Part 0b prerequisite), casehubio/agency#484 (Part 0d prerequisite)

**Goal:** Characters' PID-5 personality facets shift in response to detected situational states (stress, flow, social context), producing personality-consistent behavioral changes through the existing CDE pipeline.

**Architecture:** One personality state per agent (elastic band model) with per-facet displacement, home offset, and resilience. A TERMINAL tick participant detects situational modes via weighted threshold fusion of CAPS mediating activations, situation inputs, mood PAD, and SEC dimensions; pulls the personality toward archetype-specific attractor profiles; and feeds modulated facets through the CDE via EffectiveCognitiveDefaults. Lasting deformation accrues on a wall-clock-aligned hourly grid.

**Tech Stack:** Java 26, Quarkus, Maven, SQLite (MindMap), YAML topology files, JUnit 5 + AssertJ

## Global Constraints

- All facet arithmetic on [0, 1] scale (Eidos 0–100 converted once at boundary)
- Closed `AmpdFacetName` enum from Part 0b — no string-based facet names in runtime code
- `SituationalMode` enum: `STRESS`, `FLOW`, `SOCIAL_COLLABORATIVE`, `SOCIAL_COMPETITIVE`, `SOCIAL_HIERARCHICAL`
- Group exclusivity: at most one AFFECTIVE and one SOCIAL mode active
- CDE is never modified — personality changes flow through modified DescriptorView
- MindMap `personality-state` subgraph — not COGNITIVE (keeps state out of retrieval/rendering paths)
- All parameters in `situational-personality.yaml`, not in code
- Property-based invariants: `I ∈ [0,1]`, `E ∈ [0,1]`, `X=0 ⇒ E=0`, `|Δ| ≤ S_max`, `|o| ≤ D_max`, `r ∈ [r_min, 1]`, `c ∈ [0,1]`
- Result independent of tick rate and advance cadence (wall-clock grid, exact integrals)

## Prerequisite Gates

| Gate | Issue | What it provides | Blocks |
|---|---|---|---|
| **0b** | casehubio/eidos#234 | `AmpdFacetName` enum, `SituationalProfile` record, `SituationalProfileResolver`, `archetype-situational-profiles.yaml` | Batch 3 (Task 7-9) |
| **0a** | #1055 | `EffectiveCognitiveDefaults` (resting/effective split, `LearningProfile`, `DescriptorViewSource`, `PersonalityStateSource` SPI) | Batch 4 (Task 10-11) |

Batches 1-2 have NO prerequisites and can start immediately.

---

## Batch 1: Pure-function foundations

These are standalone computational components with no CDI, no MindMap, no external dependencies. Fully testable in isolation.

### Task 1: Configuration loader and validator

**Files:**
- Create: `neocortex/cognition/src/main/resources/situational-personality.yaml`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalPersonalityConfig.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalMode.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/ModeDetectionConfig.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SignalConfig.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SignalTransform.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalPersonalityConfigTest.java`

**Interfaces:**
- Produces: `SituationalMode` enum (used by all later tasks), `SituationalPersonalityConfig` record (used by detector, integrator, participant), `SignalTransform` enum + `apply(double)` (used by detector)

- [ ] **Step 1: Write SituationalMode enum**

```java
public enum SituationalMode {
    STRESS(Group.AFFECTIVE), FLOW(Group.AFFECTIVE),
    SOCIAL_COLLABORATIVE(Group.SOCIAL), SOCIAL_COMPETITIVE(Group.SOCIAL), SOCIAL_HIERARCHICAL(Group.SOCIAL);
    public enum Group { AFFECTIVE, SOCIAL }
    private final Group group;
    SituationalMode(Group group) { this.group = group; }
    public Group group() { return group; }
}
```

- [ ] **Step 2: Write SignalTransform enum**

Each transform is a pure function `double → double`:

```java
public enum SignalTransform {
    IDENTITY, POSITIVE_PART, NEGATIVE_PART, COMPLEMENT, ABOVE_FLOOR, RISE_FROM_REST, DROP_FROM_REST;

    public double apply(double x) { ... }
    public double apply(double x, double floor) { ... } // for ABOVE_FLOOR
    public double apply(double x, double rest, double max, double min) { ... } // for *_FROM_REST
}
```

`BAND` and `CHALLENGE_SKILL` are parameterised — model as separate record `ParameterisedTransform(SignalTransform base, double... params)`.

- [ ] **Step 3: Write failing config validation tests**

Test every rejection rule from the spec's load-time validation:
- Unknown mode name → fail
- Exogenous weights not summing to 1 → fail
- Endogenous weights not summing to 1 → fail
- `θ_off ≥ θ_on` → fail
- `β_N < 0` or `β_R < 0` → fail
- `G_m · x_sub_m ≥ θ_off_m` → fail (sub-floor hold margin violated)
- `x_floor` outside [0.3, 0.9] → fail
- Situation node not in CapsTopology input set → fail
- Mediating node not in mediating set → fail
- Situation-input transform not `ABOVE_FLOOR` → fail
- Mediating transform not `*_FROM_REST` → fail

```java
@Test void rejectsExogenousWeightsNotSummingToOne() {
    var yaml = baseConfig().withExogenousWeights("stress", List.of(0.3, 0.3));
    assertThatThrownBy(() -> SituationalPersonalityConfig.load(yaml, topology))
        .isInstanceOf(IllegalArgumentException.class)
        .hasMessageContaining("exogenous weights");
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest=SituationalPersonalityConfigTest -s .mvn/slot-settings.xml`
Expected: compilation error (classes don't exist yet)

- [ ] **Step 5: Write config records and YAML loader**

`SituationalPersonalityConfig` record with nested records for `ModeDetectionConfig`, `SignalConfig`, gains, dynamics parameters. YAML parsed via SnakeYAML (already a dependency). Validation in a `validate(CapsTopology)` method.

Write the full `situational-personality.yaml` from the spec's configuration section.

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest=SituationalPersonalityConfigTest -s .mvn/slot-settings.xml`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/
git add neocortex/cognition/src/main/resources/situational-personality.yaml
git add neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/
git commit -m "feat(cognition): situational personality config loader and validator Refs casehubio/neocortex#712"
```

### Task 2: Integrator — the elastic band state machine

**Files:**
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/PersonalityState.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/PersonalityStateIntegrator.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/PersonalityStateIntegratorTest.java`

**Interfaces:**
- Consumes: `SituationalPersonalityConfig` (dynamics parameters)
- Produces: `PersonalityState` record (per-facet `o`, `d`, `r`, `A`, `L`, `hourCursor`, `exposedUntil`), `PersonalityStateIntegrator.advance(state, to)` and `pull(state, compositeDeltas, intensities)` (used by state service and tick participant)

- [ ] **Step 1: Write PersonalityState record**

Per-facet state: `Map<String, FacetState>` where `FacetState(double displacement, double homeOffset, double resilience, double exposure, double load)`. Plus `Instant hourCursor`, `Instant exposedUntil`, mode intensities, engaged flags, group winners.

- [ ] **Step 2: Write failing integrator tests**

Property-based tests from the spec's invariants:
```java
@Test void briefExposureReturnsNearlyToHome() {
    var state = PersonalityState.initial(FACETS, Instant.now());
    state = integrator.pull(state, stressDelta(0.20), 1.0);
    // 10 turns of exposure
    for (int i = 0; i < 10; i++) {
        state = integrator.advance(state, state.hourCursor().plus(1, MINUTES));
    }
    // 24h idle
    state = integrator.advance(state, state.hourCursor().plus(24, HOURS));
    assertThat(Math.abs(state.facet("Emotional Lability").displacement())).isLessThan(0.005);
    assertThat(state.facet("Emotional Lability").homeOffset()).isLessThan(0.001);
}

@Test void prolongedExposureDeformsPermanently() { ... } // 100 turns daily × 30 days → o ∈ [0.05, D_max]
@Test void cadenceIndependent() { ... } // 5 min vs 7 day steps give same o, r within 1e-6
@Test void idleWeekReturnsDToZero() { ... }
@Test void facetsWithZeroDeltaAccrueNoLoad() { ... }
@Test void doseResponseMonotonic() { ... } // o(10 turns) < o(30 turns) < o(100 turns)
@Test void exposureWindowWeighting() { ... } // load accrues at 1 inside window, w_tail outside
@Test void workedExampleReproduces() { ... } // spec's worked example table
```

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement PersonalityStateIntegrator**

The `advance(state, to)` method implements the spec's integrator:
- While `cursorTime < to`: segment to next hour boundary or `exposedUntil`
- Weight = 1 if inside exposure window, `w_tail` otherwise
- Relaxation: exact exponential `d ← d·e^{−k·Δ}` with `k = r / τ_return`
- Hour close: fatigue `r ← max(r_min, r − L/τ_fatigue)`, healing `r ← 1 − (1−r)·e^{−1h/τ_heal}`, drift step with saturation clamp, set-point decay
- Fast-forward: once all `|d| < 1e-6`, remaining hours advance in closed form

The `pull(state, deltas, intensity)` method:
- `d ← d + α_pull·(Δ − d)`, clamped so `home + d ∈ [0,1]`
- If evidence sustained: extend `exposedUntil`

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```bash
git add neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/PersonalityState*.java
git add neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/PersonalityStateIntegratorTest.java
git commit -m "feat(cognition): elastic band personality state integrator Refs casehubio/neocortex#712"
```

### Task 3: Detector — situational mode classification

**Files:**
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalDetector.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalEvidence.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/DetectionResult.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalDetectorTest.java`

**Interfaces:**
- Consumes: `SituationalPersonalityConfig` (mode detection configs), `SignalTransform`, `SituationalMode`
- Produces: `SituationalDetector.detect(evidence, previousState) → DetectionResult` (used by tick participant), `SituationalEvidence` record (carrier for all signal inputs), `DetectionResult` (mode intensities, engaged flags, group winners)

- [ ] **Step 1: Write SituationalEvidence record**

```java
public record SituationalEvidence(
    Map<String, Double> situationInputActivations,  // node id → [0,1]
    Map<String, Double> capsMediating,               // node id → activation
    Map<String, Double> capsRestReference,            // node id → resting activation
    Map<String, Double> secDimensions,                // dim name → [0,1] or [-1,1]
    double moodPleasure, double moodArousal, double moodDominance, // [-1,1]
    double relationalTrust,                           // [-1,1], NaN if no interlocutor
    boolean hasInterlocutor
) {}
```

- [ ] **Step 2: Write failing detector tests**

```java
@Test void xZeroImpliesEZero() {
    var evidence = SituationalEvidence.withNoSituationInputs()
        .withMood(-1, 1, -1)        // maximally stressed mood
        .withSec(Map.of("relevance", 1.0, "conduciveness", -1.0));
    var result = detector.detect(evidence, DetectionResult.initial());
    assertThat(result.intensity(STRESS)).isZero();
}

@Test void stressEngagesAboveThreshold() { ... }
@Test void fadeDecayOverTime() { ... }
@Test void groupExclusivity() { ... }
@Test void switchMarginPreventsFlapping() { ... }
@Test void socialGateRequiresInterlocutor() { ... }
@Test void staleSecContributesZero() { ... }
```

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement SituationalDetector**

The gain-form detection model from the spec:
```
X_m = Σ w_s · transform(x_s)        exogenous
N_m = Σ v_s · transform(n_s)        endogenous
R_m = relational transform           (0 if no interlocutor)
E_m = min(1, X_m · (1 + β_N · N_m) · (1 + β_R · R_m))
```

Engagement hysteresis: `θ_on` to engage, `θ_off` to disengage. Exponential fade with `τ_fade`. Group winner selection with switch margin.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```bash
git add neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/Situational{Detector,Evidence}.java
git add neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/DetectionResult.java
git add neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalDetectorTest.java
git commit -m "feat(cognition): situational mode detector with gain-form evidence model Refs casehubio/neocortex#712"
```

## Batch 2: Appraisal fixes (Part 0c)

No prerequisites. Fixes production issues the detector depends on.

### Task 4: LlmAppraisalStrategy SEC dimensions + AppraisalTickParticipant accessors

**Files:**
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/LlmAppraisalStrategy.java` — change `parseResponse` to return 5-arg `AppraisalResult` with parsed `secDimensions`
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/AppraisalTickParticipant.java` — add `currentResultAt(agentId, tenantId, EventRef)`, `currentInputActivations(agentId, tenantId)`, tick EventRef stamping
- Modify: `neocortex/cognition-api/src/main/java/io/casehub/neocortex/cognition/CognitionTickContext.java` — add `EventRef tickEventRef`
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/RelationalBiasing.java` — clamp to [0,1]
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/appraisal/AppraisalFixesTest.java`

**Interfaces:**
- Produces: `AppraisalTickParticipant.currentResultAt(agentId, tenantId, EventRef)` → `Optional<AppraisalResult>`, `currentInputActivations(agentId, tenantId)` → `Map<String, Double>` (used by tick participant to build SituationalEvidence)

- [ ] **Step 1: Write failing tests**

```java
@Test void llmAppraisalReturnsSecDimensions() {
    // Parse a response with SEC dimensions
    var result = strategy.parseResponse(responseWithDims);
    assertThat(result.secDimensions()).isNotEmpty();
    assertThat(result.secDimensions().get("relevance")).isBetween(0.0, 1.0);
}

@Test void currentResultAtReturnsEmptyForWrongTick() {
    participant.tick(context1); // tick 1
    assertThat(participant.currentResultAt(agentId, tenantId, EventRef.tick(2))).isEmpty();
}

@Test void relationalBiasingClampsToUnitRange() {
    // trust = -1, confidence 0.9 → previously 1.125, should be clamped to 1.0
    double biased = RelationalBiasing.apply(0.9, -1.0, 0.25);
    assertThat(biased).isLessThanOrEqualTo(1.0);
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Fix LlmAppraisalStrategy.parseResponse**

Change line ~131 to use the 5-arg `AppraisalResult` constructor that includes `dims`.

- [ ] **Step 4: Add currentResultAt and currentInputActivations to AppraisalTickParticipant**

Stamp each result with the tick `EventRef`. Add `currentInputActivations` accessor cleared at tick start. Add `tickEventRef` to `CognitionTickContext`.

- [ ] **Step 5: Clamp RelationalBiasing.apply output**

At `RelationalBiasing.java:43`, after the multiplication: `return Math.min(1.0, Math.max(0.0, biased));`

- [ ] **Step 6: Add appraisal prompt clause for situational facets**

In `LlmAppraisalStrategy` system prompt (~line 44-47): add clause about situational traits. The rendering (`Anxiousness (0.77): situational [usual 0.62]`) is handled by the DescriptorView facet origin, not by this code — just the prompt text.

- [ ] **Step 7: Run tests to verify they pass**

- [ ] **Step 8: Commit**

```bash
git add neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/
git add neocortex/cognition-api/src/main/java/io/casehub/neocortex/cognition/CognitionTickContext.java
git add neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/appraisal/AppraisalFixesTest.java
git commit -m "fix(cognition): appraisal SEC dimensions, tick stamping, relational bias clamp Refs casehubio/neocortex#712"
```

## Batch 3: State infrastructure [GATE: needs Part 0b (eidos#234)]

### Task 5: MindMap personality-state subgraph type + SituationalStateService core

**Files:**
- Modify: `neocortex/mindmap-api/src/main/java/io/casehub/neocortex/mindmap/SubgraphTypes.java` — add `PERSONALITY_STATE`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalStateService.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/PersonalityStateSnapshot.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalStateServiceTest.java`

**Interfaces:**
- Consumes: `PersonalityStateIntegrator`, `MindMapStore`, `SituationalPersonalityConfig`
- Produces: `SituationalStateService.advance(tenantId, agentId, to)`, `publish(tenantId, agentId, snapshot)`, `PersonalityStateSource` SPI implementation (used by EffectiveCognitiveDefaults from Part 0a)

- [ ] **Step 1: Add SubgraphTypes.PERSONALITY_STATE**

```java
public static final String PERSONALITY_STATE = "personality-state";
```

- [ ] **Step 2: Write failing tests for state service**

```java
@Test void initialStateIsAtRest() {
    var state = service.loadOrCreate(tenantId, agentId, now);
    assertThat(state.facets()).allSatisfy((name, fs) -> {
        assertThat(fs.displacement()).isZero();
        assertThat(fs.homeOffset()).isZero();
        assertThat(fs.resilience()).isEqualTo(1.0);
    });
}

@Test void persistenceRoundTrip() { ... }
@Test void deterministicNodeIdentity() { ... } // SHA-256 base32 naming
@Test void tenantIsolation() { ... }
@Test void concurrentFirstTicksCreateOneSubgraph() { ... }
@Test void agentIdCollisionDetected() { ... }
```

- [ ] **Step 3: Run tests to verify they fail**

- [ ] **Step 4: Implement SituationalStateService**

Core: per-tenant subgraph resolution (create-once under tenant lock), per-agent node resolution (deterministic `situational-personality-state:<base32-sha256>` naming), per-key slot cache with locks, MindMap read/write through `resolveNode`/`addNode`/`updateNode`, `PersonalityStateSource` SPI implementation.

Properties encoding per the spec: `facet.<Name>.displacement`, `.homeOffset`, `.resilience`, `.exposure`, `.load`, plus mode intensities and control fields.

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

### Task 6: Profile resolution and facet delta conversion

**Files:**
- Create: `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/SituationalDeltaConverter.java`
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/RelationalSchemaResolver.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalDeltaConverterTest.java`

**Interfaces:**
- Consumes: `SituationalProfileResolver` (from Part 0b), `AmpdFacetName` enum (from Part 0b)
- Produces: `SituationalDeltaConverter.toSituationalDeltas(List<SituationalProfile>) → Map<SituationalMode, Map<String, Double>>` (used by tick participant), `RelationalSchemaResolver` (extracted from appraisal and regulation duplicates)

- [ ] **Step 1: Write failing tests**

```java
@Test void convertsDeltasToUnitScale() {
    var profiles = List.of(new SituationalProfile("stress",
        List.of(new AmpdFacetShift(ANXIOUSNESS, 15)), "estimated"));
    var deltas = converter.toSituationalDeltas(profiles);
    assertThat(deltas.get(STRESS).get("Anxiousness")).isEqualTo(0.15);
}

@Test void unknownModeIgnored() { ... }
@Test void emptyProfilesGiveEmptyDeltas() { ... }
```

- [ ] **Step 2: Implement and test**

- [ ] **Step 3: Commit**

## Batch 4: Wiring [GATE: needs Part 0a (#1055)]

### Task 7: SituationalPersonalityParticipant — the TERMINAL tick participant

**Files:**
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalPersonalityParticipant.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalPersonalityParticipantTest.java`

**Interfaces:**
- Consumes: `SituationalStateService`, `SituationalDetector`, `PersonalityStateIntegrator`, `SituationalDeltaConverter`, `SituationalProfileResolver`, `AppraisalTickParticipant`, `MoodOrchestrator`, `CapsEngine` (optional), `EffectiveCognitiveDefaults` (from Part 0a), `CognitiveJournal`, `Clock`
- Produces: `CognitionTickParticipant` registered in TERMINAL phase

- [ ] **Step 1: Write failing integration test**

Use in-memory MindMap, test CapsEngine, scripted situation classifier:

```java
@Test void stressDetectedAndPersonalityModulated() {
    // Tick with strong threat situation inputs
    participant.tick(contextWithThreat());
    // Verify detection result
    assertThat(service.currentState(tenantId, agentId).intensity(STRESS)).isGreaterThan(0);
    // Verify effective defaults changed
    var effective = effectiveDefaults.forAgent(tenantId, agentId);
    // Anxiousness facet should be elevated
}

@Test void noSituationInputsNoModulation() { ... }
@Test void neutralConversationDisengagesStress() { ... }
@Test void perTickCapsSettlement() { ... }
@Test void dirtyCheckPreventsRedundantCdeRefresh() { ... }
```

- [ ] **Step 2: Implement SituationalPersonalityParticipant**

Per-tick flow from the spec:
1. `advance(slot, now)`
2. Build `SituationalEvidence` from appraisal, mood, CAPS, relational schema
3. `detector.detect(evidence, previousResult) → DetectionResult`
4. Look up attractor profiles for agent's archetype via `SituationalProfileResolver` + `toSituationalDeltas`
5. Compute composite delta: `Σ(profile_delta × winner_intensity)` per group, clamped to `±S_max`
6. `integrator.pull(state, compositeDeltas)` if a winner exists
7. Publish via `EffectiveCognitiveDefaults.updateState()` if dirty
8. Persist via `SituationalStateService`
9. Journal `SituationalModulation` entry

- [ ] **Step 3: Run tests to verify they pass**

- [ ] **Step 4: Commit**

### Task 8: SituationalDeformationPhase — consolidation

**Files:**
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/situational/SituationalDeformationPhase.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalDeformationPhaseTest.java`

**Interfaces:**
- Consumes: `SituationalStateService`, `EffectiveCognitiveDefaults`, `CognitiveJournal`
- Produces: `ConsolidationPhase` implementation — advances idle agents' state on the hourly grid, publishes drift to EffectiveCognitiveDefaults

- [ ] **Step 1: Write failing test**

```java
@Test void advancesIdleAgentsState() {
    // Set up an agent with displacement, don't tick for 2 hours
    deformationPhase.run(tenantId, mindMapContext);
    // Verify state was advanced and published
    var state = service.currentState(tenantId, agentId);
    assertThat(state.facet("Emotional Lability").displacement()).isLessThan(initialDisplacement);
}

@Test void publishesDriftToEffectiveDefaults() { ... }
@Test void journalsConsolidation() { ... }
```

- [ ] **Step 2: Implement SituationalDeformationPhase**

Iterates `nodesIn(personalityStateSubgraphId, tenantId)`, calls `advance(slot, now)` under slot lock, publishes via `updateState` when any hour closed, journals `SituationalDeformationConsolidated`.

- [ ] **Step 3: Run tests, commit**

## Batch 5: Observability and eval

### Task 9: Journal entries and pipeline trace

**Files:**
- Modify: `neocortex/cognitive-observability-api` — add `CognitiveEntry.SituationalModulation` and `CognitiveEntry.SituationalDeformationConsolidated` record types
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/CognitionCore.java` — add `PipelineTrace.SituationalPersonalitySnapshot` to `currentTrace`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/ObservabilityTest.java`

**Interfaces:**
- Consumes: `SituationalPersonalityParticipant` (tick produces journal entries), `SituationalDeformationPhase` (consolidation produces journal entries)
- Produces: journal entries queryable by existing observability infrastructure

- [ ] **Step 1: Define journal entry records**

```java
public record SituationalModulation(
    String tenantId, String agentId, EventRef eventRef, Instant timestamp,
    Map<SituationalMode, Double> intensities,
    Map<SituationalMode.Group, SituationalMode> winners,
    Map<SituationalMode, Double> exogenousEvidence,
    Map<SituationalMode, Double> endogenousEvidence,
    Map<SituationalMode, Double> relationalEvidence,
    Map<SituationalMode, Double> totalEvidence,
    Map<String, Double> compositeDelta,
    Map<String, Double> publishedFacets,
    Set<String> skippedFacets,
    String profileProvenance
) implements CognitiveEntry {}
```

- [ ] **Step 2: Wire journal recording into participant and deformation phase**

- [ ] **Step 3: Test that journal entries are recorded on publish**

- [ ] **Step 4: Commit**

### Task 10: End-to-end integration test

**Files:**
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/situational/SituationalPersonalityIntegrationTest.java`

**Interfaces:**
- Consumes: all components from Tasks 1-9, `CognitivePipelineBuilder` (from Part 0d or test-local builder)

- [ ] **Step 1: Write end-to-end scenario test**

Using `ScriptedSituationClassifier` + `SqliteCapsEngine` + in-memory MindMap:

```java
@Test void fullStressCycleWithRecovery() {
    // 1. Strong threat observation → stress engages
    tickWith(strongThreatObservation);
    assertStressEngaged();
    // 2. At tick t+1, effective defaults reflect modulated facets
    tickWith(neutralObservation);
    assertEffectiveAnxiousnessElevated();
    // 3. 10 neutral ticks → stress disengages, displacement relaxes
    for (int i = 0; i < 10; i++) tickWith(neutralObservation);
    assertStressDisengaged();
    assertDisplacementDecreasing();
    // 4. Journal entries recorded with tick EventRef
    assertJournalContainsSituationalModulation();
}

@Test void noiseFloorDoesNotHoldMode() { ... }
@Test void restartPreservesState() { ... }
@Test void tenancyIsolation() { ... }
```

- [ ] **Step 2: Run test, fix integration issues, commit**

### Task 11: Noise-floor topology integration test

**Files:**
- Create: `neocortex/caps-engine/src/test/java/io/casehub/neocortex/caps/engine/SituationalNoiseFloorTest.java`

- [ ] **Step 1: Write topology test**

Settle the real `caps-topology.yaml` with every situation input at `x_floor` (and at the uncalibrated default 0.6). Assert from-rest mediating responses ≤ `m_floor`.

```java
@Test void subFloorInputsProduceSubFloorMediatingResponses() {
    var topology = CapsTopologyLoader.load();
    var state = capsEngine.initializeAgent(agentId, tenantId, disposition);
    var restResult = capsEngine.settle(state, Map.of()); // resting
    // Set every situation input to x_floor
    var inputs = situationInputNodes.stream().collect(toMap(n -> n, n -> X_FLOOR));
    var stressResult = capsEngine.settle(state, inputs);
    // Check mediating from-rest responses
    for (var node : mediatingNodes) {
        double fromRest = Math.max(0, stressResult.activation(node) - restResult.activation(node));
        assertThat(fromRest).as("from-rest for %s", node).isLessThanOrEqualTo(M_FLOOR);
    }
}
```

- [ ] **Step 2: Run test, commit**

---

## References

- `wsp-casehub/specs/issue-712-shadow-archetypes-personality-degradation/2026-11-17-situational-personality-modulation-design.md` — design spec (rev 4)
- `wsp-casehub/specs/issue-712-shadow-archetypes-personality-degradation/decisions.md` — architectural decisions D1-D5
- `examples/docs/specs/issue-845-cde-pid5-facets/2026-11-13-cde-pid5-facet-upgrade-design.md` — #845 pipeline unification spec
- `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/AppraisalTickParticipant.java` — appraisal accessor source
- `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/LlmAppraisalStrategy.java:131` — SEC dimensions gap
- `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/RelationalBiasing.java:43` — unclamped bias
- `neocortex/caps-engine/src/main/resources/caps-topology.yaml` — CAPS network definition
- `neocortex/caps-api/src/main/java/io/casehub/neocortex/caps/CapsEngine.java` — CAPS engine interface
- `neocortex/mindmap-api/src/main/java/io/casehub/neocortex/mindmap/SubgraphTypes.java` — subgraph constants
- `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` — CDE
- `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridge.java` — bridge
- casehubio/neocortex#712, #1055, #1056-1065
- casehubio/eidos#234
- casehubio/agency#484
