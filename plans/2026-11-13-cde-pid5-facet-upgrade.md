# CDE PID-5 Facet Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #845 — Upgrade CDE derivations to use PID-5 facets instead of disposition axes
**Issue group:** #845

**Goal:** Unify the personality pipeline (Engram → Eidos → Bridge → CDE) and upgrade all CDE derivation methods to use continuous PID-5/AMPD facet scores, producing characters whose cognitive defaults reflect richer personality-trait interactions.

**Architecture:** Extend Eidos `AgentDisposition` with AMPD facet data, fix `EidosCognitiveProfileBridge` to read facets into `DescriptorView`, then upgrade 6 Category A methods (DispositionAxes → facetScore()) and add 8 Category B Jungian×PID-5 modulations. Two-tier evidence model: Tier 1 (peer-reviewed) and Tier 2 (first-principles via FFM bridge).

**Tech Stack:** Java 26, Quarkus, Maven multi-module (agency + neocortex repos)

## Global Constraints

- Java 26, use records for new data types
- All facet scores: 0-100 integer in Eidos/engram, 0.0-1.0 double in PersonalityFacet
- `facetScore()` default is 0.5 (population mean) — zero modulation at baseline
- Modulation weight constants start at 0.2 (modest effect), tuned via eval
- Tier 2 modulations must include reasoning chain in code comment
- Use `ide_insert_member` / `ide_replace_member` for all Java edits
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <pom> test -s .mvn/slot-settings.xml`
- Agency repo: `/Users/mdproctor/claude/casehub/slots/196/agency`
- Neocortex repo: `/Users/mdproctor/claude/casehub/slots/196/neocortex`

---

## Batch 1: Pipeline foundation — AMPD facets flow from Eidos to CDE

### Task 1: Extend AgentDisposition with ampdFacets (agency repo)

**Files:**
- Modify: `agency/eidos-api/src/main/java/io/casehub/eidos/api/AgentDisposition.java`
- Create: `agency/eidos-api/src/main/java/io/casehub/eidos/api/AmpdFacet.java`
- Test: `agency/eidos-api/src/test/java/io/casehub/eidos/api/AgentDispositionTest.java`

**Interfaces:**
- Produces: `AmpdFacet` record with `String facet()` and `int score()` (0-100). `AgentDisposition.ampdFacets()` returning `List<AmpdFacet>` defaulting to `List.of()`.

- [ ] **Step 1: Write the AmpdFacet record**

```java
public record AmpdFacet(String facet, int score) {
    public AmpdFacet {
        if (score < 0 || score > 100) throw new IllegalArgumentException("AMPD score must be 0-100: " + score);
    }
}
```

- [ ] **Step 2: Add ampdFacets field to AgentDisposition**

Add `List<AmpdFacet> ampdFacets` field to the `AgentDisposition` record. Default to `List.of()` in the builder. Update any existing constructors/factory methods to pass `List.of()` for backward compatibility.

- [ ] **Step 3: Write test for YAML deserialization**

Create a test YAML descriptor with `ampdFacets` entries. Verify deserialization produces correct `AmpdFacet` list. Also test that a descriptor WITHOUT ampdFacets deserializes with empty list (backward compat).

```java
@Test
void ampdFacetsDeserializeFromYaml() {
    // YAML with ampdFacets: [{facet: callousness, score: 20}, {facet: anxiousness, score: 70}]
    var disposition = loadDisposition("test-ampd-descriptor.yaml");
    assertEquals(2, disposition.ampdFacets().size());
    assertEquals("callousness", disposition.ampdFacets().get(0).facet());
    assertEquals(20, disposition.ampdFacets().get(0).score());
}

@Test
void missingAmpdFacetsDefaultsToEmptyList() {
    var disposition = loadDisposition("test-no-ampd-descriptor.yaml");
    assertTrue(disposition.ampdFacets().isEmpty());
}
```

- [ ] **Step 4: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f agency/eidos-api/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS — new tests green, existing tests unaffected

- [ ] **Step 5: Build and install agency module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f agency/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml`

- [ ] **Step 6: Commit**

```
feat(#845): add ampdFacets to AgentDisposition — Eidos carries PID-5 data
```

### Task 2: Fix EidosCognitiveProfileBridge to read AMPD facets (neocortex)

**Files:**
- Modify: `neocortex/cognitive-eidos/src/main/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridge.java` (line ~95-110)
- Test: `neocortex/cognitive-eidos/src/test/java/io/casehub/neocortex/cognitive/eidos/EidosCognitiveProfileBridgeTest.java`

**Interfaces:**
- Consumes: `AgentDisposition.ampdFacets()` → `List<AmpdFacet>` from Task 1
- Produces: `DescriptorView.personalityFacets()` populated with `List<PersonalityFacet>` (0.0-1.0 scale)

- [ ] **Step 1: Write failing test — bridge maps ampdFacets to PersonalityFacet list**

```java
@Test
void bridgeMapsAmpdFacetsToPersonalityFacets() {
    var descriptor = buildDescriptor(List.of(
        new AmpdFacet("callousness", 20),
        new AmpdFacet("anxiousness", 70)
    ));
    var view = bridge.toDescriptorView(descriptor);
    assertEquals(2, view.personalityFacets().size());
    assertEquals("Callousness", view.personalityFacets().get(0).facet());
    assertEquals(0.20, view.personalityFacets().get(0).score(), 0.001);
    assertEquals("Anxiousness", view.personalityFacets().get(1).facet());
    assertEquals(0.70, view.personalityFacets().get(1).score(), 0.001);
}

@Test
void bridgeHandlesEmptyAmpdFacets() {
    var descriptor = buildDescriptor(List.of());
    var view = bridge.toDescriptorView(descriptor);
    assertTrue(view.personalityFacets().isEmpty());
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — bridge still passes `List.of()`

- [ ] **Step 3: Implement bridge AMPD mapping in toDescriptorView()**

In `EidosCognitiveProfileBridge.toDescriptorView()`, after extracting disposition and dispositionProfile, add:

```java
List<PersonalityFacet> facets = descriptor.disposition().ampdFacets().stream()
    .map(af -> PersonalityFacet.of(af.facet(), af.score() / 100.0))
    .toList();
```

Pass `facets` to the `DescriptorView` constructor instead of `List.of()`.

Note: `PersonalityFacet.of()` may need the facet name capitalised to match the constant names (e.g., "callousness" → "Callousness"). Check how `PersonalityFacet` constants define their `facet` string and match accordingly.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-eidos/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#845): bridge reads ampdFacets from Eidos — PersonalityFacets now flow to CDE
```

---

## Batch 2: Category A — DispositionAxes methods → facetScore()

### Task 3: Upgrade deriveMoodBaseline, deriveCuriosity, deriveCbrStrategy

**Files:**
- Modify: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` (lines 340-402)
- Test: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java`

**Interfaces:**
- Consumes: `facetScore()`, `isHigh()`, `isLow()` helper methods (already exist at lines 884-906)
- Produces: Same output records (MoodBaseline, CuriosityConfig, CbrStrategyDefaults) but now modulated by continuous facet scores

- [ ] **Step 1: Write failing tests — facets modulate mood baseline**

Test that a character with Risk Taking = 0.8 (bold) produces higher pleasure baseline than Risk Taking = 0.2 (conservative). Same for arousal via Emotional Lability, dominance via Withdrawal (inverse).

```java
@Test
void deriveMoodBaselineUsesRiskTakingFacet() {
    var bold = deriveMoodBaseline(facetsOf(RISK_TAKING, 0.8), defaultAxes());
    var conservative = deriveMoodBaseline(facetsOf(RISK_TAKING, 0.2), defaultAxes());
    assertTrue(bold.pleasure() > conservative.pleasure());
}
```

Write similar tests for deriveCuriosity and deriveCbrStrategy.

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAIL — methods don't accept facets yet

- [ ] **Step 3: Update method signatures to accept List<PersonalityFacet>**

For each of the 3 methods, add `List<PersonalityFacet> facets` parameter. Follow the `deriveSearchBudgetDefaults` reference pattern:
1. Extract facet scores via `facetScore(facets, FACET_NAME, 0.5)`
2. Replace `RISK_PLEASURE.getOrDefault(lower(axes.riskAppetite()), 0.0)` with continuous arithmetic from `riskTaking` facet score
3. When facets list is empty, fall back to the existing DispositionAxes lookup

- [ ] **Step 4: Update CDE.derive() to pass facets to these methods**

In the `derive()` method (~line 189), pass `descriptor.personalityFacets()` to each upgraded method.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS — new tests green, existing tests still green (empty facets = same behavior)

- [ ] **Step 6: Commit**

```
feat(#845): upgrade deriveMoodBaseline, deriveCuriosity, deriveCbrStrategy to PID-5 facets
```

### Task 4: Upgrade deriveSocialCognition, deriveHabituationConfig, deriveRegulationDefaults

**Files:**
- Modify: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` (lines 404-592)
- Test: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java`

**Interfaces:**
- Consumes: `facetScore()` helpers, same as Task 3
- Produces: SocialCognitionDefaults, HabituationConfig, RegulationDefaults — now facet-modulated

- [ ] **Step 1: Write failing tests**

Key test cases:
- `deriveSocialCognition`: high Suspiciousness → lower trust rate; high Hostility → competitive conflict interpretation
- `deriveHabituationConfig`: high Impulsivity → faster habituation (lower novelty threshold)
- `deriveRegulationDefaults`: high Emotional Lability → higher reappraisal capacity needed; high Callousness → lower social regulation tendency

```java
@Test
void deriveRegulationDefaultsUsesEmotionalLabilityFacet() {
    var labile = deriveRegulationDefaults(facetsOf(EMOTIONAL_LABILITY, 0.85), defaultAxes());
    var stable = deriveRegulationDefaults(facetsOf(EMOTIONAL_LABILITY, 0.15), defaultAxes());
    assertTrue(labile.reappraisalCapacity() < stable.reappraisalCapacity(),
        "High emotional lability should reduce effective reappraisal capacity");
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement facet-based derivation for all 3 methods**

Same pattern as Task 3. `deriveRegulationDefaults` is the most complex (30 lines, uses multiple axes). Map each axis lookup to its PID-5 facet equivalent:
- axes.socialOrient → Withdrawal (inverse), Intimacy Avoidance (inverse)
- axes.conflictMode → Hostility, Suspiciousness
- axes.riskAppetite → Risk Taking
- axes.ruleFollowing → Impulsivity (inverse)
- axes.autonomy → Withdrawal

- [ ] **Step 4: Update CDE.derive() calls**

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#845): upgrade deriveSocialCognition, deriveHabituationConfig, deriveRegulationDefaults to PID-5 facets
```

---

## Batch 3: Category B Tier 1 — peer-reviewed Jungian×PID-5 modulations

### Task 5: Impulsivity→WM + Callousness→relationship bias + Anxiousness→fear weight

**Files:**
- Modify: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` (deriveCapacity ~271, deriveExtractionBias ~444, deriveAppraisalWeights ~481)
- Test: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java`

**Interfaces:**
- Consumes: Existing Jungian-derived base values + `facetScore()` for modulation
- Produces: Same output records with facet-modulated values

- [ ] **Step 1: Write failing tests for all 3 Tier 1 modulations**

```java
@Test
void deriveCapacityImpulsivityDegradesWorkingMemory() {
    // Tier 1: Heydarloo 2024, Kofler 2024
    var impulsive = deriveCapacity(jungianProfile(), facetsOf(IMPULSIVITY, 0.9));
    var controlled = deriveCapacity(jungianProfile(), facetsOf(IMPULSIVITY, 0.1));
    assertTrue(impulsive.workingMemorySpan() < controlled.workingMemorySpan());
}

@Test
void deriveExtractionBiasCallousnessReducesRelationshipBias() {
    // Tier 1: Blair 2005, 2008, 2013
    var callous = deriveExtractionBias(jungianProfile(), facetsOf(CALLOUSNESS, 0.9));
    var empathic = deriveExtractionBias(jungianProfile(), facetsOf(CALLOUSNESS, 0.1));
    assertTrue(callous.relationshipBias() < empathic.relationshipBias());
}

@Test
void deriveAppraisalWeightsAnxiousnessAmplifiesFear() {
    // Tier 1: Bar-Haim et al. 2007
    var anxious = deriveAppraisalWeights(jungianProfile(), facetsOf(ANXIOUSNESS, 0.9), defaultAxes());
    var calm = deriveAppraisalWeights(jungianProfile(), facetsOf(ANXIOUSNESS, 0.1), defaultAxes());
    assertTrue(anxious.fearWeight() > calm.fearWeight());
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Add facet parameter and implement modulations**

For each method, add `List<PersonalityFacet> facets` parameter. Apply modulation after the Jungian base computation:

```java
// deriveCapacity — Impulsivity degrades WM span
// Tier 1: Heydarloo et al. (2024) Memory; Kofler et al. (2024) Frontiers in Psychiatry
double impulsivity = facetScore(facets, PersonalityFacet.IMPULSIVITY, 0.5);
double wmSpan = compressCapacity(0.5 * (te + si) - (impulsivity - 0.5) * 0.2);

// deriveExtractionBias — Callousness reduces relationship bias
// Tier 1: Blair (2005, 2008, 2013) Nature Reviews Neuroscience
double callousness = facetScore(facets, PersonalityFacet.CALLOUSNESS, 0.5);
double relationshipBias = baseBias - (callousness - 0.5) * 0.2;

// deriveAppraisalWeights — Anxiousness amplifies fear weight
// Tier 1: Bar-Haim et al. (2007) Psychological Bulletin, 172-study meta-analysis, d=0.45
double anxiousness = facetScore(facets, PersonalityFacet.ANXIOUSNESS, 0.5);
double fearWeight = baseFear + (anxiousness - 0.5) * 0.2;
```

- [ ] **Step 4: Update CDE.derive() calls**

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#845): add Tier 1 PID-5 modulations — Impulsivity/WM, Callousness/empathy, Anxiousness/fear

Peer-reviewed: Heydarloo 2024, Kofler 2024, Blair 2005/2008/2013, Bar-Haim 2007
```

---

## Batch 4: Category B Tier 2 + integration test + eval

### Task 6: Tier 2 modulations — 5 FFM bridge extensions

**Files:**
- Modify: `neocortex/cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngine.java` (deriveExtractionBias, deriveAppraisalWeights, deriveCapacity, deriveGraphStructure)
- Test: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationEngineTest.java`

**Interfaces:**
- Consumes: facetScore() on methods already accepting facets from Task 5
- Produces: Additional modulations on existing outputs

- [ ] **Step 1: Write failing tests for all 5 Tier 2 modulations**

```java
@Test
void deriveExtractionBiasEmotionalLabilityAmplifiesAffect() {
    // Tier 2: high lability = high Neuroticism → amplified emotional reactivity
    // Bridge: McCrae & Costa 1989 + Gore & Widiger 2013
    var labile = deriveExtractionBias(jungianProfile(), facetsOf(EMOTIONAL_LABILITY, 0.9));
    var stable = deriveExtractionBias(jungianProfile(), facetsOf(EMOTIONAL_LABILITY, 0.1));
    assertTrue(labile.affectSensitivity() > stable.affectSensitivity());
}

@Test
void deriveAppraisalWeightsRiskTakingReducesFear() {
    // Tier 2: high risk = Disinhibition → reduced threat sensitivity
    var bold = deriveAppraisalWeights(jungianProfile(), facetsOf(RISK_TAKING, 0.9), defaultAxes());
    var cautious = deriveAppraisalWeights(jungianProfile(), facetsOf(RISK_TAKING, 0.1), defaultAxes());
    assertTrue(bold.fearWeight() < cautious.fearWeight());
}

@Test
void deriveAppraisalWeightsImpulsivityAmplifiesUrgency() {
    // Tier 2: Disinhibition → immediate threat/opportunity weighting
    var impulsive = deriveAppraisalWeights(jungianProfile(), facetsOf(IMPULSIVITY, 0.9), defaultAxes());
    var deliberate = deriveAppraisalWeights(jungianProfile(), facetsOf(IMPULSIVITY, 0.1), defaultAxes());
    assertTrue(impulsive.urgencyWeight() > deliberate.urgencyWeight());
}

@Test
void deriveCapacityAnxiousnessNarrowsAssociativeReach() {
    // Tier 2: Negative Affectivity → attentional narrowing
    var anxious = deriveCapacity(jungianProfile(), facetsOf(ANXIOUSNESS, 0.9));
    var calm = deriveCapacity(jungianProfile(), facetsOf(ANXIOUSNESS, 0.1));
    assertTrue(anxious.associativeReach() < calm.associativeReach());
}

@Test
void deriveGraphStructureWithdrawalReducesConnectiveBias() {
    // Tier 2: Detachment → reduced social node connectivity
    var withdrawn = deriveGraphStructure(jungianProfile(), facetsOf(WITHDRAWAL, 0.9));
    var engaged = deriveGraphStructure(jungianProfile(), facetsOf(WITHDRAWAL, 0.1));
    assertTrue(withdrawn.connectiveBias() < engaged.connectiveBias());
}
```

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement all 5 Tier 2 modulations**

Add each modulation after the Jungian base computation, with full reasoning chain in comments:

```java
// Tier 2: Emotional Lability → amplified affect sensitivity
// Bridge: Emotional Lability → Negative Affectivity (PID-5) → high Neuroticism (FFM) → amplified emotional reactivity
// Grounded in McCrae & Costa 1989 + Gore & Widiger 2013
double emotionalLability = facetScore(facets, PersonalityFacet.EMOTIONAL_LABILITY, 0.5);
double affectSensitivity = baseAffect + (emotionalLability - 0.5) * 0.2;
```

- [ ] **Step 4: Add facets parameter to deriveGraphStructure**

This method currently only takes `List<WeightedTerm>`. Add `List<PersonalityFacet> facets` parameter and update `CDE.derive()` call.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#845): add Tier 2 PID-5 modulations — 5 FFM bridge extensions

First-principles via McCrae & Costa 1989 + Gore & Widiger 2013.
Each modulation documents reasoning chain in code comment.
```

### Task 7: Integration test + full test suite verification

**Files:**
- Modify: `neocortex/cognitive-index/src/test/java/io/casehub/neocortex/cognitive/index/CognitiveDerivationIntegrationTest.java`

**Interfaces:**
- Consumes: Full CDE pipeline — DescriptorView with Jungian functions + AMPD facets → CognitiveDefaults

- [ ] **Step 1: Add integration test with full Jungian + AMPD profile**

Create a DescriptorView with both Jungian function weights AND AMPD facets representing a specific character archetype (e.g., high-Ti analytical thinker with high Callousness + high Risk Taking). Verify that CognitiveDefaults reflect both channels:
- Jungian-derived base values are present
- AMPD facets modulate those base values (e.g., relationship bias reduced by Callousness)

```java
@Test
void fullProfileCombinesJungianBaseWithAmpdModulation() {
    var view = DescriptorView.of("test-agent",
        defaultAxes(),
        analyticJungianProfile(), // Ti=0.35, Te=0.25, etc.
        List.of("investigate"),
        null,
        List.of(
            PersonalityFacet.of(PersonalityFacet.CALLOUSNESS, 0.80),
            PersonalityFacet.of(PersonalityFacet.IMPULSIVITY, 0.70),
            PersonalityFacet.of(PersonalityFacet.ANXIOUSNESS, 0.20)
        )
    );
    var defaults = CognitiveDerivationEngine.derive(view);

    // Callousness reduces relationship bias from Fi/Fe base
    assertTrue(defaults.extractionBias().relationshipBias() < 0.5);
    // Impulsivity degrades WM from Te+Si base
    assertNotNull(defaults.capacityConfig());
    // Low anxiousness means fear weight not amplified
    assertTrue(defaults.appraisalWeights().fearWeight() < 0.5);
}
```

- [ ] **Step 2: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognitive-index/pom.xml test -s .mvn/slot-settings.xml`
Expected: ALL PASS

- [ ] **Step 3: Run fast tests across wacky-manor to check for regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pfast -s .mvn/slot-settings.xml`
Expected: 712+ tests PASS, 0 failures

- [ ] **Step 4: Commit**

```
test(#845): integration test for combined Jungian + AMPD cognitive derivation
```

---

## References

- [2026-11-13-cde-pid5-facet-upgrade-design.md] — design spec this plan implements
- [era7-personality-trait-integration-research.md] — evidence-level documentation
- [decisions.md] — D1-D5 design decisions
- [CognitiveDerivationEngine.java:765-844] — reference pattern (deriveSearchBudgetDefaults)
- [EidosCognitiveProfileBridge.java:110] — bridge gap
- [PersonalityFacet.java] — facet record with 13 PID-5 constants
- [AgentDisposition.java] — Eidos disposition schema (agency repo)
- [GitHub #845] — focal issue
- [GitHub #712] — shadow archetypes (seam architecture connection)
- [GitHub #851] — dynamic regulatory capacity (MindMap runtime state)
