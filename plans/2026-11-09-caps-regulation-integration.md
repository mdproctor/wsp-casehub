# CAPS Regulation Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #806 — Emotion regulation CAPS integration: regulation strategies as CAUs
**Issue group:** #806

**Goal:** Encode all 10 emotion regulation strategies as CAPS output nodes with 4 literature-grounded social-context input nodes, producing a three-layer cascade (CAPS > predictor > LLM) for regulation strategy selection.

**Architecture:** Extend the unified CAPS topology with 14 new nodes (10 regulation outputs + 4 social-context inputs) and ~34 new connections. Regulation outputs settle alongside behavioral outputs in a single pass. Social-context inputs route through existing mediating nodes for personality-sensitive processing. A new `RegulationCascadeConfig` governs cascade pin decisions separately from the noise filter. `EmotionRegulationTickParticipant` orchestrates the cascade with a safety hard gate for fatigue/intensity overrides.

**Tech Stack:** Java 21 (Java 26 JVM), Quarkus 3.32.2, YAML topology, SQLite, Maven

## Global Constraints

- All weights in caps-topology.yaml use provenance `consensus` for literature-derived values
- All regulation output nodes are `UNIPOLAR` range `[0, 1]`
- `BehavioralAttractor` is renamed to `CapsAttractor` throughout — use `ide_refactor_rename`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f pom.xml test -Pfast -s .mvn/slot-settings.xml` (from neocortex root, not examples)
- Test: run targeted test classes, not full suite — `mvn test -pl <module> -Dtest=<TestClass> -s .mvn/slot-settings.xml`
- Topology YAML: `caps-engine/src/main/resources/caps-topology.yaml`
- All new connections must be enumerated in the spec's §Connection Architecture tables

---

## Batch 1: Foundation — CAPS API Type Rename + Config

Safe wrap point: CAPS API types updated, all existing tests pass with new names. No behavioral change.

### Task 1: Rename BehavioralAttractor → CapsAttractor

**Files:**
- Rename: `BehavioralAttractor` → `CapsAttractor` (use `ide_refactor_rename`)
  - `caps-api/src/main/java/io/casehub/neocortex/caps/BehavioralAttractor.java:3`
- Rename: `BehavioralAttractorEvalTest` → `CapsAttractorEvalTest` (use `ide_refactor_rename`)
  - `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/BehavioralAttractorEvalTest.java:23`
- Modify: `caps-api/src/main/java/io/casehub/neocortex/caps/SettlingResult.java:6` (return type)
- Modify: all callers (IntelliJ handles via rename refactoring)

**Interfaces:**
- Produces: `CapsAttractor` record with fields: `nodeId`, `category`, `strength`, `highSat`

- [ ] **Step 1: Use IntelliJ rename refactoring on BehavioralAttractor**

```
ide_refactor_rename: BehavioralAttractor → CapsAttractor
```

This updates the record, SettlingResult, CapsSettler.extractAttractors(), all test classes, and all callers across the project.

- [ ] **Step 2: Rename BehavioralAttractorEvalTest**

```
ide_refactor_rename: BehavioralAttractorEvalTest → CapsAttractorEvalTest
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml compile -pl caps-api,caps-engine,caps-testing,caps-llm-classifier -s .mvn/slot-settings.xml -Dmaven.test.skip=true`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run CAPS contract tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-testing -Dtest=CapsEngineContractTest -s .mvn/slot-settings.xml`
Expected: All 9 tests PASS

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "refactor(#806): rename BehavioralAttractor → CapsAttractor

Regulation strategies are CAPS outputs but not 'behavioral' in the narrow
sense. The type name now reflects that both behavioral and regulatory
attractors are output categories distinguished by the category field.

Refs #806"
```

### Task 2: Add RegulationCascadeConfig record

**Files:**
- Create: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/RegulationCascadeConfig.java`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/RegulationCascadeConfigTest.java`

**Interfaces:**
- Produces: `RegulationCascadeConfig(double regulationConfidenceThreshold, double regulationDominanceMargin)` with defaults factory

- [ ] **Step 1: Write the test**

```java
package io.casehub.neocortex.cognition.regulation;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class RegulationCascadeConfigTest {

    @Test
    void defaultsHaveExpectedValues() {
        var config = RegulationCascadeConfig.defaults();
        assertEquals(0.3, config.regulationConfidenceThreshold(), 0.001);
        assertEquals(0.15, config.regulationDominanceMargin(), 0.001);
    }

    @Test
    void rejectsNegativeThreshold() {
        assertThrows(IllegalArgumentException.class,
            () -> new RegulationCascadeConfig(-0.1, 0.15));
    }

    @Test
    void rejectsNegativeMargin() {
        assertThrows(IllegalArgumentException.class,
            () -> new RegulationCascadeConfig(0.3, -0.1));
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest=RegulationCascadeConfigTest -s .mvn/slot-settings.xml`
Expected: FAIL — class not found

- [ ] **Step 3: Implement the record**

```java
package io.casehub.neocortex.cognition.regulation;

public record RegulationCascadeConfig(
    double regulationConfidenceThreshold,
    double regulationDominanceMargin
) {
    public RegulationCascadeConfig {
        if (regulationConfidenceThreshold < 0)
            throw new IllegalArgumentException("regulationConfidenceThreshold must be >= 0");
        if (regulationDominanceMargin < 0)
            throw new IllegalArgumentException("regulationDominanceMargin must be >= 0");
    }

    public static RegulationCascadeConfig defaults() {
        return new RegulationCascadeConfig(0.3, 0.15);
    }
}
```

- [ ] **Step 4: Run test — verify it passes**

Run: same command as step 2
Expected: 3 tests PASS

- [ ] **Step 5: Commit**

```bash
git add cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/RegulationCascadeConfig.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/RegulationCascadeConfigTest.java
git commit -m "feat(#806): add RegulationCascadeConfig for cascade pin decisions

Architecturally separate from CapsSettlingConfig.attractorStrengthMinFilter
(noise filter). regulationConfidenceThreshold (0.3) and
regulationDominanceMargin (0.15) govern when CAPS overrides the predictor.

Refs #806"
```

---

## Batch 2: Topology Extension — YAML + Disposition Modulation

Safe wrap point: CAPS topology has all regulation nodes and connections. Existing settling behavior unchanged. DispositionWeightMapper applies regulation resting activations.

### Task 3: Extend caps-topology.yaml with regulation nodes and connections

**Files:**
- Modify: `caps-engine/src/main/resources/caps-topology.yaml`
- Test: `caps-testing/src/test/java/io/casehub/neocortex/caps/testing/CapsEngineContractTest.java` (existing — verify topology loads)
- Test: Create `caps-testing/src/test/java/io/casehub/neocortex/caps/testing/RegulationTopologyTest.java`

**Interfaces:**
- Produces: 14 new nodes (10 regulation outputs + 4 social-context inputs) and 34 new connections in the topology YAML

- [ ] **Step 1: Write the topology validation test**

```java
package io.casehub.neocortex.caps.testing;

import io.casehub.neocortex.caps.CapsNode;
import io.casehub.neocortex.caps.CapsTopology;
import io.casehub.neocortex.caps.engine.CapsTopologyLoader;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class RegulationTopologyTest {

    private final CapsTopology topology = CapsTopologyLoader.load();

    @Test
    void regulationOutputNodesExist() {
        var expected = List.of(
            "situation_selection", "situation_modification",
            "attentional_deployment", "cognitive_change", "response_modulation",
            "social_support_seeking", "social_sharing", "social_withdrawal",
            "co_regulation", "deliberate_provocation");
        for (var id : expected) {
            var node = topology.nodes().get(id);
            assertNotNull(node, "Missing regulation output node: " + id);
            assertEquals(CapsNode.NodeType.OUTPUT, node.type());
            assertEquals(CapsNode.NodeRange.UNIPOLAR, node.range());
            assertEquals("regulation", node.category());
        }
    }

    @Test
    void socialContextInputNodesExist() {
        var expected = List.of(
            "perceived_responsiveness", "target_distress",
            "social_isolation", "power_asymmetry");
        for (var id : expected) {
            var node = topology.nodes().get(id);
            assertNotNull(node, "Missing social-context input node: " + id);
            assertEquals(CapsNode.NodeType.INPUT, node.type());
            assertEquals("social_context", node.category());
        }
    }

    @Test
    void regulationConnectionsExist() {
        var connections = topology.connections();
        assertTrue(connections.stream().anyMatch(
            c -> c.from().equals("arousal_level") && c.to().equals("attentional_deployment")),
            "Missing arousal_level → attentional_deployment connection");
        assertTrue(connections.stream().anyMatch(
            c -> c.from().equals("self_efficacy") && c.to().equals("cognitive_change")),
            "Missing self_efficacy → cognitive_change connection");
        assertTrue(connections.stream().anyMatch(
            c -> c.from().equals("target_distress") && c.to().equals("co_regulation")),
            "Missing target_distress → co_regulation direct connection");
    }

    @Test
    void socialContextToMediatingConnectionsExist() {
        var connections = topology.connections();
        assertTrue(connections.stream().anyMatch(
            c -> c.from().equals("perceived_responsiveness") && c.to().equals("other_safety")),
            "Missing perceived_responsiveness → other_safety connection");
        assertTrue(connections.stream().anyMatch(
            c -> c.from().equals("social_isolation") && c.to().equals("BIS_activation")),
            "Missing social_isolation → BIS_activation connection");
    }

    @Test
    void topologyNodeCountIsCorrect() {
        assertEquals(91, topology.nodes().size(),
            "Expected 91 nodes (77 baseline + 10 regulation + 4 social-context)");
    }

    @Test
    void topologyConnectionCountIsCorrect() {
        assertEquals(110, topology.connections().size(),
            "Expected 110 connections (76 baseline + 34 new)");
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-testing -Dtest=RegulationTopologyTest -s .mvn/slot-settings.xml`
Expected: FAIL — missing nodes

- [ ] **Step 3: Add social-context input nodes to caps-topology.yaml**

Add under `nodes.input:` section:

```yaml
    # Source: Reis et al. 2004 (PPR), Reeck et al. 2016 (SRC),
    # Rimé 2009, Williams 2007, Schilke et al. 2021
    social_context:
      - id: perceived_responsiveness
        description: "Felt understanding, validation, and caring from a present interaction partner"
        vocabulary:
          responsive: 0.8
          understanding: 0.7
          validating: 0.7
          caring: 0.7
          supportive: 0.6
          attentive: 0.5
          empathic: 0.5
      - id: target_distress
        description: "Observable distress in another agent — crying, trembling, visible anguish"
        vocabulary:
          distressed: 0.9
          crying: 0.8
          anguished: 0.8
          suffering: 0.7
          upset: 0.6
          distraught: 0.7
          troubled: 0.5
      - id: social_isolation
        description: "Absence of social contact — alone, no one nearby, ostracized"
        vocabulary:
          alone: 0.8
          isolated: 0.8
          nobody: 0.7
          solitary: 0.6
          empty: 0.4
          deserted: 0.5
      - id: power_asymmetry
        description: "Relational power advantage — authority, dominance, perceived impunity"
        vocabulary:
          authority: 0.7
          powerful: 0.7
          superior: 0.6
          dominant: 0.6
          advantage: 0.5
          impunity: 0.8
          untouchable: 0.7
```

- [ ] **Step 4: Add regulation output nodes to caps-topology.yaml**

Add under `nodes.output:` section:

```yaml
    regulation:
      - situation_selection
      - situation_modification
      - attentional_deployment
      - cognitive_change
      - response_modulation
      - social_support_seeking
      - social_sharing
      - social_withdrawal
      - co_regulation
      - deliberate_provocation
```

- [ ] **Step 5: Add all 34 connections to caps-topology.yaml**

Add under `connections:` section — 7 social-context→mediating, 14 intrapersonal regulation, 13 social regulation connections. All weights and sources as specified in the design spec §Connection Architecture tables. Two direct connections (target_distress → co_regulation, power_asymmetry → deliberate_provocation) are justified exceptions documented in the spec.

- [ ] **Step 6: Run test — verify it passes**

Run: same command as step 2
Expected: All 6 tests PASS

- [ ] **Step 7: Run existing CAPS contract tests for regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl caps-testing -Dtest=CapsEngineContractTest -s .mvn/slot-settings.xml`
Expected: All 9 existing tests still PASS

- [ ] **Step 8: Commit**

```bash
git add caps-engine/src/main/resources/caps-topology.yaml caps-testing/src/test/java/io/casehub/neocortex/caps/testing/RegulationTopologyTest.java
git commit -m "feat(#806): extend CAPS topology with regulation + social-context nodes

10 regulation output nodes (Gross 1998 + Zaki & Williams 2013).
4 social-context input nodes (Reis 2004, Reeck 2016, Rimé 2009,
Schilke 2021). 34 new connections with literature-derived weights.

Refs #806"
```

### Task 4: Extend DispositionWeightMapper with regulation resting activations

**Files:**
- Modify: `caps-engine/src/main/java/io/casehub/neocortex/caps/engine/DispositionWeightMapper.java:21`
- Test: Create `caps-engine/src/test/java/io/casehub/neocortex/caps/engine/RegulationDispositionMappingTest.java`

**Interfaces:**
- Consumes: `DispositionAxes` (from cognitive-index)
- Produces: regulation output nodes with personality-based resting activations in `AgentCapsState`

- [ ] **Step 1: Write test for regulation resting activations**

```java
package io.casehub.neocortex.caps.engine;

import io.casehub.neocortex.cognitive.index.DispositionAxes;
import io.casehub.neocortex.caps.CapsTopology;
import io.casehub.neocortex.caps.engine.CapsTopologyLoader;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class RegulationDispositionMappingTest {

    private final CapsTopology topology = CapsTopologyLoader.load();

    @Test
    void avoidantConflictModeBoostsSituationSelection() {
        var axes = new DispositionAxes("cooperative", "moderate", "calculated",
            "moderate", "avoidant");
        var mapper = new DispositionWeightMapper(topology);
        var state = mapper.initializeAgent("agent1", "tenant1", axes);
        var situationSelection = state.nodeStates().get("situation_selection");
        assertNotNull(situationSelection);
        assertEquals(0.2, situationSelection.restingActivation(), 0.01);
    }

    @Test
    void analyticalConflictModeBoostsCognitiveChange() {
        var axes = new DispositionAxes("cooperative", "moderate", "calculated",
            "moderate", "analytical");
        var mapper = new DispositionWeightMapper(topology);
        var state = mapper.initializeAgent("agent1", "tenant1", axes);
        var cognitiveChange = state.nodeStates().get("cognitive_change");
        assertNotNull(cognitiveChange);
        assertEquals(0.2, cognitiveChange.restingActivation(), 0.01);
    }

    @Test
    void cooperativeSocialOrientBoostsSocialSupportSeeking() {
        var axes = new DispositionAxes("cooperative", "moderate", "calculated",
            "moderate", "cooperative");
        var mapper = new DispositionWeightMapper(topology);
        var state = mapper.initializeAgent("agent1", "tenant1", axes);
        var supportSeeking = state.nodeStates().get("social_support_seeking");
        assertNotNull(supportSeeking);
        assertEquals(0.15, supportSeeking.restingActivation(), 0.01);
    }
}
```

- [ ] **Step 2: Run test — verify it fails**

Expected: FAIL — regulation nodes have no resting activations yet

- [ ] **Step 3: Add regulation resting activations to DispositionWeightMapper**

In `setRestingActivations()` method, add the regulation output resting activation mappings from the spec §Disposition Modulation table. Also fix the `"confrontational"` → `"competitive"` inconsistency noted in the spec.

- [ ] **Step 4: Run test — verify it passes**

- [ ] **Step 5: Commit**

```bash
git add caps-engine/src/main/java/io/casehub/neocortex/caps/engine/DispositionWeightMapper.java caps-engine/src/test/java/io/casehub/neocortex/caps/engine/RegulationDispositionMappingTest.java
git commit -m "feat(#806): add regulation resting activations to DispositionWeightMapper

Personality-based baseline bias for regulation output nodes:
avoidant→situation_selection, analytical→cognitive_change,
cooperative→co_regulation, etc. Fixes confrontational→competitive
inconsistency.

Refs #806"
```

---

## Batch 3: Goal Namespace Isolation

Safe wrap point: GoalProposalOrchestrator supports namespace-scoped goals. No more clobbering between ProactiveRegulation and EmotionRegulationGoal participants.

### Task 5: Add namespace isolation to GoalProposalOrchestrator

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestrator.java:32`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationGoalParticipant.java`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipant.java:18`
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestratorTest.java:28`
- Test: Add namespace isolation tests to `GoalProposalOrchestratorTest`

**Interfaces:**
- Produces: `registerParticipantGoals(String participantId, String agentId, String tenantId, List<DriveGoalProposal> goals)` — replaces only the specified namespace

- [ ] **Step 1: Write failing test for namespace isolation**

```java
@Test
void registerParticipantGoalsDoesNotClobberOtherNamespace() {
    var proactiveGoal = new DriveGoalProposal(
        DriveAxis.AFFILIATION, "proactive-avoidance-library", 0.8, null, null);
    var regulationGoal = new DriveGoalProposal(
        DriveAxis.AUTONOMY, "regulation-reappraise", 0.6, null, null);

    orchestrator.registerParticipantGoals("proactive", agentId, tenantId,
        List.of(proactiveGoal));
    orchestrator.registerParticipantGoals("regulation", agentId, tenantId,
        List.of(regulationGoal));

    var proposals = orchestrator.currentProposals(agentId, tenantId);
    assertEquals(2, proposals.size(), "Both namespaces should contribute goals");
}
```

- [ ] **Step 2: Run test — verify it fails**

- [ ] **Step 3: Implement namespace isolation**

Add `Map<String, List<DriveGoalProposal>> namespacedGoals` to `GoalProposalState`. Implement `registerParticipantGoals()` with per-namespace replace. Update `currentProposals()` to merge all namespaces with `cachedProposals`.

- [ ] **Step 4: Migrate callers**

Replace `EmotionRegulationGoalParticipant.registerGoals()` → `registerParticipantGoals("emotion-regulation", ...)`.
Replace `ProactiveRegulationParticipant.appendGoals()` → `registerParticipantGoals("proactive-regulation", ...)`.

- [ ] **Step 5: Run all regulation tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest="GoalProposalOrchestratorTest,EmotionRegulationGoalParticipantTest,ProactiveRegulationParticipantTest" -s .mvn/slot-settings.xml`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add cognition/
git commit -m "feat(#806): add namespace isolation to GoalProposalOrchestrator

registerParticipantGoals() replaces only the specified namespace.
Eliminates clobbering between proactive avoidance (FOUNDATION) and
reactive regulation (TERMINAL) goal participants.

Refs #806"
```

---

## Batch 4: Social-Context Injection + Avoidance Decay

Safe wrap point: Social-context activations computable. Exponential decay for avoidance.

### Task 6: Create SocialContextInputComputer

**Files:**
- Create: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/SocialContextInputComputer.java`
- Test: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/SocialContextInputComputerTest.java`

**Interfaces:**
- Consumes: `RegulationContext`, `MindMapStore`, `CognitiveProfile` (optional)
- Produces: `Map<String, Double> computeSocialInputs(RegulationContext ctx, String tenantId)` — 4 keyed activations

- [ ] **Step 1: Write tests for each social-context formula**

Test `perceived_responsiveness` (trust × 0.6 + trajectory × 0.4), `target_distress` (PAD formula), `social_isolation` (gradient with saturatingCount=3), `power_asymmetry` (dominance-submission+bias formula). Test edge cases: null RegulationContext → all zeros, empty interactingAgents → social_isolation = 1.0, multiple agents → max trust/distress.

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement SocialContextInputComputer**

Implement all 4 formulas from spec §Social-Context Input Injection with `clamp01()`. Constructor takes `MindMapStore` (required), `CognitiveProfile` (optional via `Instance<CognitiveProfile>`).

- [ ] **Step 4: Run tests — verify they pass**

- [ ] **Step 5: Commit**

```bash
git add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/SocialContextInputComputer.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/SocialContextInputComputerTest.java
git commit -m "feat(#806): add SocialContextInputComputer for CAPS regulation inputs

Computes 4 social-context activations per tick from RegulationContext
and MindMap data: perceived_responsiveness, target_distress,
social_isolation, power_asymmetry. Literature-grounded formulas.

Refs #806"
```

### Task 7: Exponential decay in ProactiveRegulationParticipant

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipant.java:18`
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipantTest.java:23`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java` (constructor call)

**Interfaces:**
- Breaking change: `decayHorizonTicks` → `decayTimeConstant` constructor parameter

- [ ] **Step 1: Write test for exponential decay profile**

```java
@Test
void exponentialDecayAt30TicksIs37Percent() {
    double initialWeight = 1.0;
    int decayTimeConstant = 30;
    double decayed = initialWeight * Math.exp(-30.0 / decayTimeConstant);
    assertEquals(0.368, decayed, 0.01);
}

@Test
void exponentialDecayAt60TicksIs14Percent() {
    double initialWeight = 1.0;
    int decayTimeConstant = 30;
    double decayed = initialWeight * Math.exp(-60.0 / decayTimeConstant);
    assertEquals(0.135, decayed, 0.01);
}
```

- [ ] **Step 2: Replace linear decay with exponential in avoidance score calculation**

Change the decay formula in `ProactiveRegulationParticipant` from linear to `Math.exp(-elapsedTicks / decayTimeConstant)`. Rename constructor parameter.

- [ ] **Step 3: Update CognitionCore constructor call**

Update `configureProactiveRegulation()` to pass `decayTimeConstant` (default 30) instead of `decayHorizonTicks`.

- [ ] **Step 4: Run proactive regulation tests**

Expected: All PASS with updated decay semantics

- [ ] **Step 5: Commit**

```bash
git add cognition/
git commit -m "feat(#806): exponential decay for proactive avoidance

Replaces linear decay (hard cliff) with exponential decay
(Ebbinghaus 1885, Wixted & Ebbesen 1991). decayTimeConstant=30
gives 37% at 30 ticks, 14% at 60 ticks. Smooth asymptotic approach.

Refs #806"
```

---

## Batch 5: Cascade Integration — EmotionRegulationTickParticipant

Safe wrap point: Full CAPS > predictor > LLM cascade operational. The core deliverable.

### Task 8: Integrate CAPS settling into EmotionRegulationTickParticipant

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java:21`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:72` (wiring)
- Create: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/RegulationCascadeTest.java`

**Interfaces:**
- Consumes: `CapsEngine`, `SocialContextInputComputer`, `RegulationCascadeConfig`
- Consumes: `SituationClassifier` (existing — for input activations)
- Produces: Regulation strategy selection via three-layer cascade

- [ ] **Step 1: Write cascade test — CAPS pins strategy**

Test that when CAPS settling produces a dominant regulation attractor above threshold with sufficient margin, it pins the strategy (skipping the predictor).

- [ ] **Step 2: Write cascade test — CAPS ambivalent, predictor fallback**

Test that when CAPS produces multiple regulation attractors without a dominant one, the predictor is consulted.

- [ ] **Step 3: Write cascade test — safety hard gate**

Test that fatigue > 0.7 forces ATTENTIONAL_DEPLOYMENT regardless of CAPS output.

- [ ] **Step 4: Write cascade test — CAPS weight update after regulation**

Test that `CapsEngine.updateWeights()` is called with the regulation effectiveness mapping after a successful regulation tick.

- [ ] **Step 5: Run tests — verify they fail**

- [ ] **Step 6: Implement cascade in EmotionRegulationTickParticipant**

Modify `tick()` method:
1. Extract safety hard gate (fatigue/intensity) as first check
2. After effort check, assemble social-context inputs via `SocialContextInputComputer`
3. Merge with situation classification `inputActivations`
4. Call `CapsEngine.settle()` with merged activations
5. Extract regulation attractors (category = "regulation")
6. Apply multi-attractor resolution (dominance margin check)
7. If dominant → pin strategy; else → fall through to existing predictor logic
8. After regulation result, call `CapsEngine.updateWeights()` with effectiveness mapping

- [ ] **Step 7: Wire CapsEngine and SocialContextInputComputer into CognitionCore**

Update `CognitionCore.configureRegulation()` to pass `CapsEngine`, `SocialContextInputComputer`, and `RegulationCascadeConfig` to `EmotionRegulationTickParticipant`. Add `ensureTopologyCoverage()` call on first tick for agent migration.

- [ ] **Step 8: Run tests — verify they pass**

- [ ] **Step 9: Run full regulation unit test suite for regression**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -pl cognition -Dtest="EmotionRegulationTickParticipantTest,RegulationCascadeTest,RegulationStrategyPredictorTest,RegulationMonitorTest,RegulationFatigueTrackerTest" -s .mvn/slot-settings.xml`
Expected: All PASS

- [ ] **Step 10: Commit**

```bash
git add cognition/
git commit -m "feat(#806): integrate CAPS settling into regulation cascade

Three-layer cascade: safety gate → CAPS settling → predictor → LLM.
CAPS regulation attractors with dominance margin pin the LLM strategy.
Weight update via Rescorla-Wagner with regulation effectiveness.
Social-context inputs merged with situation classification.

Refs #806"
```

---

## Batch 6: Eval Infrastructure

Safe wrap point: Paired comparison eval test captures CAPS vs. predictor vs. LLM data for threshold calibration.

### Task 9: Paired comparison eval test

**Files:**
- Create: `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/CapsRegulationComparisonEvalTest.java`

**Interfaces:**
- Consumes: `CognitionCoreTestBuilder`, `CapsEngine`, `RegulationCascadeConfig`

- [ ] **Step 1: Write eval test structure**

Create test class tagged `@Tag("llm-eval")` that runs existing regulation scenarios from `PerStrategyRegulationEvalTest` with CAPS settling enabled. For each scenario, record:
- CAPS regulation attractor (strategy + strength)
- Predictor strategy (+ confidence)
- LLM free-choice strategy (run without pinning)
- Agreement matrix

- [ ] **Step 2: Implement comparison recording**

Use `CognitionCoreTestBuilder` with `CapsEngine` + `CognitiveDefaultsRegistry` wired. Run each scenario twice: once with cascade (CAPS → predictor → LLM), once with LLM-only (no pin). Compare results.

- [ ] **Step 3: Add threshold sweep analysis**

After running scenarios, compute concordance at different threshold values (0.1, 0.2, 0.3, 0.4, 0.5) — output as structured JSON for empirical threshold selection.

- [ ] **Step 4: Run eval (requires API key)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/pom.xml test -Pllm-eval -Dtest=CapsRegulationComparisonEvalTest -s .mvn/slot-settings.xml`
Expected: Test produces comparison data

- [ ] **Step 5: Commit**

```bash
git add caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/CapsRegulationComparisonEvalTest.java
git commit -m "feat(#806): add CAPS regulation comparison eval test

Paired comparison: CAPS attractor vs predictor vs LLM free-choice.
Threshold sweep for empirical calibration of regulationConfidenceThreshold.

Refs #806"
```

---

## References

- [2026-11-09-caps-regulation-integration-design.md] — design spec this plan implements
- [caps-api/src/main/java/io/casehub/neocortex/caps/BehavioralAttractor.java:3] — renamed to CapsAttractor
- [caps-api/src/main/java/io/casehub/neocortex/caps/CapsEngine.java:6] — CapsEngine SPI
- [caps-api/src/main/java/io/casehub/neocortex/caps/SettlingResult.java:6] — settling result type
- [caps-engine/src/main/java/io/casehub/neocortex/caps/engine/CapsSettler.java:22] — settling algorithm
- [caps-engine/src/main/java/io/casehub/neocortex/caps/engine/DispositionWeightMapper.java:21] — personality modulation
- [caps-engine/src/main/resources/caps-topology.yaml] — network topology YAML
- [cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java:21] — regulation orchestrator
- [cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipant.java:18] — proactive avoidance
- [cognition/src/main/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestrator.java:32] — goal namespace
- [cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:72] — composition root
- [GitHub #806] — focal issue
- [GitHub #590] — Phase 1 regulation (prerequisite)
