# Personality-Modulated CPTs for TheoryOfMindEngine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/neocortex#1097 — Personality-modulated CPTs for TheoryOfMindEngine
**Issue group:** #1097

**Goal:** Make the Bayesian Theory-of-Mind engine produce personality-conditioned posteriors — anxious agents assume higher baseline threat, callous agents discount others' distress signals.

**Architecture:** `TomPersonalityModulation` record carries two facet signals (anxiousness, callousness) into `TheoryOfMindEngine`, which interpolates its fixed CPTs at construction time. `TheoryOfMindBridge` gains an overload accepting modulation; `CognitionCore.tick()` extracts facets from `EffectiveCognitiveDefaults` and passes them through. Network topology and SEC→evidence mappings are unchanged.

**Tech Stack:** Java 26, Quarkus, quarkmind inference-bayes (BayesNetwork, BayesVariable, EvidenceFusionEngine)

## Global Constraints

- All personality facet scores are in [0, 1] per `PersonalityFacet` validation
- CPT rows must sum to 1.0 (BayesVariable enforces this)
- Default/no-facet behavior must be identical to current fixed CPTs (backward compatibility)
- Facet lookup follows `CognitiveDerivationEngine` pattern: `facetScore(facets, name, defaultValue)`
- Build with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s slot-settings.xml`
- Use `mvn -f neocortex/mindmap-intelligence/pom.xml test -s slot-settings.xml` for engine-level tests

---

## Batch 1: Engine-level personality modulation

### Task 1: TomPersonalityModulation record and parameterized TheoryOfMindEngine

**Files:**
- Create: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/tom/TomPersonalityModulation.java`
- Modify: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/tom/TheoryOfMindEngine.java`
- Test: `mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/tom/TheoryOfMindEngineTest.java`

**Interfaces:**
- Produces: `TomPersonalityModulation(double anxiousness, double callousness)` record with `NEUTRAL` constant; `TheoryOfMindEngine(TomPersonalityModulation)` constructor

- [ ] **Step 1: Write failing tests for personality-modulated engine**

Add four new tests to `TheoryOfMindEngineTest`:

```java
@Test
void highAnxietyRaisesBaselineDistress() {
    var anxious = new TheoryOfMindEngine(new TomPersonalityModulation(0.9, 0.0));
    var calm = new TheoryOfMindEngine(TomPersonalityModulation.NEUTRAL);

    double[] anxiousBaseline = anxious.inferEmotionalState();
    double[] calmBaseline = calm.inferEmotionalState();

    assertTrue(anxiousBaseline[EmotionalValence.DISTRESSED.ordinal()] >
               calmBaseline[EmotionalValence.DISTRESSED.ordinal()],
        "high anxiety should raise baseline distress probability");
}

@Test
void lowAnxietyLowersBaselineDistress() {
    var relaxed = new TheoryOfMindEngine(new TomPersonalityModulation(0.1, 0.0));
    var neutral = new TheoryOfMindEngine(TomPersonalityModulation.NEUTRAL);

    double[] relaxedBaseline = relaxed.inferEmotionalState();
    double[] neutralBaseline = neutral.inferEmotionalState();

    assertTrue(relaxedBaseline[EmotionalValence.DISTRESSED.ordinal()] <
               neutralBaseline[EmotionalValence.DISTRESSED.ordinal()],
        "low anxiety should lower baseline distress probability");
}

@Test
void callousAgentDiscountsAvoidanceEvidence() {
    var callous = new TheoryOfMindEngine(new TomPersonalityModulation(0.5, 0.9));
    var empathic = new TheoryOfMindEngine(new TomPersonalityModulation(0.5, 0.0));

    callous.observeBehavior(BehaviorType.AVOIDANT);
    callous.observeThreat(ThreatLevel.MILD);
    empathic.observeBehavior(BehaviorType.AVOIDANT);
    empathic.observeThreat(ThreatLevel.MILD);

    double[] callousState = callous.inferEmotionalState();
    double[] empathicState = empathic.inferEmotionalState();

    assertTrue(callousState[EmotionalValence.DISTRESSED.ordinal()] <
               empathicState[EmotionalValence.DISTRESSED.ordinal()],
        "callous agent should infer less distress from avoidant behavior: callous=" +
        callousState[EmotionalValence.DISTRESSED.ordinal()] + " empathic=" +
        empathicState[EmotionalValence.DISTRESSED.ordinal()]);
}

@Test
void neutralModulationMatchesDefault() {
    var modulated = new TheoryOfMindEngine(TomPersonalityModulation.NEUTRAL);
    var unmodulated = new TheoryOfMindEngine();

    modulated.observeBehavior(BehaviorType.AVOIDANT);
    modulated.observeThreat(ThreatLevel.MILD);
    unmodulated.observeBehavior(BehaviorType.AVOIDANT);
    unmodulated.observeThreat(ThreatLevel.MILD);

    double[] m = modulated.inferEmotionalState();
    double[] u = unmodulated.inferEmotionalState();

    for (int i = 0; i < 3; i++) {
        assertEquals(u[i], m[i], 0.001,
            "NEUTRAL modulation should match default for " + EmotionalValence.values()[i]);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest=TheoryOfMindEngineTest -s slot-settings.xml`
Expected: compilation failure — `TomPersonalityModulation` does not exist

- [ ] **Step 3: Create TomPersonalityModulation record**

Create `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/tom/TomPersonalityModulation.java`:

```java
package io.casehub.neocortex.mindmap.intelligence.tom;

public record TomPersonalityModulation(
        double anxiousness,
        double callousness
) {
    public static final TomPersonalityModulation NEUTRAL = new TomPersonalityModulation(0.5, 0.0);

    public TomPersonalityModulation {
        if (anxiousness < 0.0 || anxiousness > 1.0)
            throw new IllegalArgumentException("anxiousness must be in [0,1], got " + anxiousness);
        if (callousness < 0.0 || callousness > 1.0)
            throw new IllegalArgumentException("callousness must be in [0,1], got " + callousness);
    }
}
```

- [ ] **Step 4: Parameterize TheoryOfMindEngine constructor and buildNetwork**

Modify `TheoryOfMindEngine`:

1. Add a new constructor `TheoryOfMindEngine(TomPersonalityModulation modulation)` that passes modulation to `buildNetwork`.
2. Change the no-arg constructor to delegate: `this(TomPersonalityModulation.NEUTRAL)`.
3. Change `buildNetwork()` signature to `buildNetwork(TomPersonalityModulation mod)`.
4. Modulate the threat prior based on `mod.anxiousness()`:
   - Baseline (anxiousness=0.5): `{0.60, 0.30, 0.10}` (current values)
   - High anxiousness (1.0): `{0.30, 0.40, 0.30}` — elevated threat
   - Low anxiousness (0.0): `{0.80, 0.15, 0.05}` — relaxed
   - Interpolate linearly: `prior[i] = baseline[i] + (anxiousness - 0.5) * 2.0 * (anxious[i] - baseline[i])`
5. Modulate the emotion→behavior CPT distressed row based on `mod.callousness()`:
   - Baseline (callousness=0.0): `{0.05, 0.20, 0.75}` (current values)
   - High callousness (1.0): `{0.15, 0.50, 0.35}` — callous observer reads distress weakly
   - Interpolate: `distressedRow[i] = baseline[i] + callousness * (callous[i] - baseline[i])`

The resulting `buildNetwork(TomPersonalityModulation mod)` method:

```java
private static BayesNetwork buildNetwork(TomPersonalityModulation mod) {
    var graph = new BayesNetwork("TheoryOfMind");

    double ax = mod.anxiousness();
    double ca = mod.callousness();

    // Threat prior — anxiety shifts toward elevated threat
    double[] baselinePrior = {0.60, 0.30, 0.10};
    double[] anxiousPrior  = {0.30, 0.40, 0.30};
    double[] relaxedPrior  = {0.80, 0.15, 0.05};
    double[] threatPrior = new double[3];
    for (int i = 0; i < 3; i++) {
        if (ax >= 0.5) {
            threatPrior[i] = baselinePrior[i] + (ax - 0.5) * 2.0 * (anxiousPrior[i] - baselinePrior[i]);
        } else {
            threatPrior[i] = baselinePrior[i] + (0.5 - ax) * 2.0 * (relaxedPrior[i] - baselinePrior[i]);
        }
    }

    GraphNode<BayesVariable> threatNode = graph.addNode();
    threatNode.setContent(new BayesVariable<>(VAR_THREAT,
        threatNode.getId(),
        new String[]{"NONE", "MILD", "SEVERE"},
        new double[][]{threatPrior}));

    // Emotion | Threat CPT — unchanged
    GraphNode<BayesVariable> emotionNode = graph.addNode();
    emotionNode.setContent(new BayesVariable<>(VAR_EMOTION,
        emotionNode.getId(),
        new String[]{"POSITIVE", "NEUTRAL", "DISTRESSED"},
        new double[][]{
            {0.50, 0.40, 0.10},
            {0.15, 0.40, 0.45},
            {0.05, 0.15, 0.80}
        },
        new String[]{VAR_THREAT}));

    // Behavior | Emotion CPT — callousness weakens distress→avoidant coupling
    double[] baseDistressed = {0.05, 0.20, 0.75};
    double[] callousDistressed = {0.15, 0.50, 0.35};
    double[] distressedRow = new double[3];
    for (int i = 0; i < 3; i++) {
        distressedRow[i] = baseDistressed[i] + ca * (callousDistressed[i] - baseDistressed[i]);
    }

    GraphNode<BayesVariable> behaviorNode = graph.addNode();
    behaviorNode.setContent(new BayesVariable<>(VAR_BEHAVIOR,
        behaviorNode.getId(),
        new String[]{"APPROACH", "NEUTRAL", "AVOIDANT"},
        new double[][]{
            {0.70, 0.25, 0.05},  // POSITIVE — unchanged
            {0.25, 0.60, 0.15},  // NEUTRAL — unchanged
            distressedRow         // DISTRESSED — modulated by callousness
        },
        new String[]{VAR_EMOTION}));

    graph.addEdge(threatNode, emotionNode);
    graph.addEdge(emotionNode, behaviorNode);

    return graph;
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest=TheoryOfMindEngineTest -s slot-settings.xml`
Expected: all 11 tests PASS (7 existing + 4 new)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add \
  mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/tom/TomPersonalityModulation.java \
  mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/tom/TheoryOfMindEngine.java \
  mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/tom/TheoryOfMindEngineTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#1097): personality-modulated CPTs for TheoryOfMindEngine

Add TomPersonalityModulation record (anxiousness, callousness).
Parameterize TheoryOfMindEngine constructor — anxiety shifts threat
prior, callousness weakens distress→avoidant coupling in behavior CPT.
Backward-compatible: NEUTRAL modulation produces identical posteriors."
```

---

## Batch 2: Bridge and CognitionCore wiring

### Task 2: Extend TheoryOfMindBridge with personality-aware overload

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/tom/TheoryOfMindBridge.java:41-47`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/tom/TheoryOfMindBridgeTest.java`

**Interfaces:**
- Consumes: `TomPersonalityModulation` from Task 1; `TheoryOfMindEngine(TomPersonalityModulation)` constructor
- Produces: `TheoryOfMindBridge.infer(AppraisalResult, TomPersonalityModulation)` overload

- [ ] **Step 1: Write failing tests for personality-modulated bridge inference**

Add three tests to `TheoryOfMindBridgeTest`:

```java
@Test
void anxiousAgentInfersMoreDistressFromSameAppraisal() {
    var anxiousMod = new TomPersonalityModulation(0.9, 0.0);
    var calmMod = new TomPersonalityModulation(0.1, 0.0);

    var appraisal = appraisalWith(
            Map.of("conduciveness", -0.3, "controllability", 0.5),
            List.of(new ActionTendency(ActionReadiness.ATTENDING, 0.5, "")));

    var anxiousPosterior = bridge.infer(appraisal, anxiousMod);
    var calmPosterior = bridge.infer(appraisal, calmMod);

    assertTrue(anxiousPosterior.distressed() > calmPosterior.distressed(),
        "anxious agent should infer more distress: anxious=" +
        anxiousPosterior.distressed() + " calm=" + calmPosterior.distressed());
}

@Test
void callousAgentInfersLessDistressFromAvoidantBehavior() {
    var callousMod = new TomPersonalityModulation(0.5, 0.9);
    var empathicMod = new TomPersonalityModulation(0.5, 0.0);

    var appraisal = appraisalWith(
            Map.of("conduciveness", -0.5, "controllability", 0.3),
            List.of(new ActionTendency(ActionReadiness.AVOIDANCE, 0.8, "")));

    var callousPosterior = bridge.infer(appraisal, callousMod);
    var empathicPosterior = bridge.infer(appraisal, empathicMod);

    assertTrue(callousPosterior.distressed() < empathicPosterior.distressed(),
        "callous agent should infer less distress: callous=" +
        callousPosterior.distressed() + " empathic=" + empathicPosterior.distressed());
}

@Test
void nullModulationMatchesUnmodulatedInference() {
    var appraisal = appraisalWith(
            Map.of("conduciveness", -0.5, "controllability", 0.3),
            List.of(new ActionTendency(ActionReadiness.AVOIDANCE, 0.7, "")));

    var withDefault = bridge.infer(appraisal, TomPersonalityModulation.NEUTRAL);
    var withoutMod = bridge.infer(appraisal);

    assertEquals(withoutMod.positive(), withDefault.positive(), 0.001);
    assertEquals(withoutMod.neutral(), withDefault.neutral(), 0.001);
    assertEquals(withoutMod.distressed(), withDefault.distressed(), 0.001);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest=TheoryOfMindBridgeTest -s slot-settings.xml`
Expected: compilation failure — no `infer(AppraisalResult, TomPersonalityModulation)` method

- [ ] **Step 3: Add overloaded infer method to TheoryOfMindBridge**

Add import for `TomPersonalityModulation`. Add new overload and delegate existing method:

```java
public TheoryOfMindPosterior infer(AppraisalResult appraisal, TomPersonalityModulation modulation) {
    var engine = new TheoryOfMindEngine(modulation);
    engine.observeThreat(threatDistribution(appraisal.secDimensions()));
    engine.observeBehavior(behaviorDistribution(appraisal.actionTendencies()));
    double[] posterior = engine.inferEmotionalState();
    return new TheoryOfMindPosterior(posterior[0], posterior[1], posterior[2]);
}

public TheoryOfMindPosterior infer(AppraisalResult appraisal) {
    return infer(appraisal, TomPersonalityModulation.NEUTRAL);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -Dtest=TheoryOfMindBridgeTest -s slot-settings.xml`
Expected: all 13 tests PASS (10 existing + 3 new)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add \
  cognition/src/main/java/io/casehub/neocortex/cognition/tom/TheoryOfMindBridge.java \
  cognition/src/test/java/io/casehub/neocortex/cognition/tom/TheoryOfMindBridgeTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#1097): personality-aware TheoryOfMindBridge.infer() overload

Bridge accepts TomPersonalityModulation to construct a personality-
conditioned engine. Existing no-arg infer() delegates with NEUTRAL."
```

### Task 3: Wire personality modulation into CognitionCore.tick()

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:354-363`

**Interfaces:**
- Consumes: `TheoryOfMindBridge.infer(AppraisalResult, TomPersonalityModulation)` from Task 2; `EffectiveCognitiveDefaults.forAgentOrDefaults()` (existing); `PersonalityFacet.ANXIOUSNESS`, `PersonalityFacet.CALLOUSNESS` (existing)
- Produces: personality-conditioned ToM posteriors in the tick pipeline

- [ ] **Step 1: Modify CognitionCore.tick() ToM block to extract facets and pass modulation**

At `CognitionCore.java:354-363`, the current code is:

```java
if (theoryOfMindBridge != null && mentalModel != null && appraisalParticipant != null) {
    appraisalParticipant.currentResult(agentId, tenantId).ifPresent(appraisalResult ->
            safeRun(() -> {
                var posterior = theoryOfMindBridge.infer(appraisalResult);
                mentalModel.recordTomPosterior(agentId, subjectId, tenantId, posterior);
                // journal record...
            }));
}
```

Replace the inner `safeRun` lambda to build modulation from facets:

```java
if (theoryOfMindBridge != null && mentalModel != null && appraisalParticipant != null) {
    appraisalParticipant.currentResult(agentId, tenantId).ifPresent(appraisalResult ->
            safeRun(() -> {
                var mod = tomModulation(agentId, tenantId);
                var posterior = theoryOfMindBridge.infer(appraisalResult, mod);
                mentalModel.recordTomPosterior(agentId, subjectId, tenantId, posterior);
                cognitiveJournal.record(new io.casehub.neocortex.cognition.journal.CognitiveEntry.TheoryOfMindInferred(
                        tenantId, agentId, tickEventRef, java.time.Instant.now(), subjectId,
                        posterior.dominantValence(), posterior.dominantProbability()));
            }));
}
```

- [ ] **Step 2: Add tomModulation helper method to CognitionCore**

Add a private method that extracts facets and builds the modulation:

```java
private io.casehub.neocortex.mindmap.intelligence.tom.TomPersonalityModulation tomModulation(String agentId, String tenantId) {
    if (effectiveDefaults == null) {
        return io.casehub.neocortex.mindmap.intelligence.tom.TomPersonalityModulation.NEUTRAL;
    }
    var defaults = effectiveDefaults.forAgentOrDefaults(tenantId, agentId);
    var profile = defaults.personalityProfile();
    if (profile == null || profile.facets().isEmpty()) {
        return io.casehub.neocortex.mindmap.intelligence.tom.TomPersonalityModulation.NEUTRAL;
    }
    double anxiousness = facetScore(profile.facets(), io.casehub.neocortex.cognitive.index.PersonalityFacet.ANXIOUSNESS);
    double callousness = facetScore(profile.facets(), io.casehub.neocortex.cognitive.index.PersonalityFacet.CALLOUSNESS);
    return new io.casehub.neocortex.mindmap.intelligence.tom.TomPersonalityModulation(anxiousness, callousness);
}

private static double facetScore(java.util.List<io.casehub.neocortex.cognitive.index.PersonalityFacet> facets, String name) {
    return facets.stream()
        .filter(f -> name.equals(f.facet()))
        .findFirst()
        .map(io.casehub.neocortex.cognitive.index.PersonalityFacet::score)
        .orElse(0.5);
}
```

Note: default is 0.5 (not 0.0) so that absent facets produce NEUTRAL modulation behavior.

- [ ] **Step 3: Verify cognition module compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml compile -s slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run full cognition test suite to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add \
  cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#1097): wire personality-modulated ToM into CognitionCore.tick()

Extract ANXIOUSNESS and CALLOUSNESS facets from EffectiveCognitiveDefaults
and build TomPersonalityModulation for each ToM inference call. Absent
facets default to 0.5 (neutral modulation)."
```

---

## References

- [TheoryOfMindEngine.java] — existing fixed-CPT Bayesian ToM engine
- [TheoryOfMindBridge.java] — SEC→ToM mapping bridge
- [CognitionCore.java:354-363] — tick() ToM inference site
- [EffectiveCognitiveDefaults.java] — personality-derived cognitive defaults
- [PersonalityFacet.java] — PID-5 facet constants (ANXIOUSNESS, CALLOUSNESS)
- [AttachmentProfile.java] — attachment anxiety/avoidance dimensions
- [CognitiveDerivationEngine.java:1139-1141] — `facetScore()` lookup pattern
- [CognitiveDerivationEngine.java:688-691] — anxiety → fear onset (Bar-Haim 2007)
- [CognitiveDerivationEngine.java:598-601] — callousness → relationship bias (Blair 2005/2008/2013)
- [GitHub #1097] — Personality-modulated CPTs for TheoryOfMindEngine
- [GitHub #1094] — Bayesian ToM wired into tick (prerequisite, done)
