# Per-Strategy Regulation Eval Tests — Design Spec

**Issue:** casehubio/neocortex#822
**Branch:** issue-851-dynamic-regulatory-capacity
**Date:** 2026-11-08

## Goal

Prove all 5 Gross emotion regulation strategies are exercised by the
regulation pipeline with targeted scenarios. Currently only
ATTENTIONAL_DEPLOYMENT and RESPONSE_MODULATION are tested
(`RegulationFullPipelineEvalTest`). Each strategy gets a 2-tick
full-pipeline test with engineered personality signals.

## Architecture

### Predictor Integration (D6)

The existing eval test does not wire `CognitiveDefaultsRegistry`, so
`RegulationStrategyPredictor` is null and the LLM freely chooses
strategies. The new test wires the predictor — the production strategy
selection mechanism — by:

1. Creating `CognitiveDefaults` per character with explicit
   `RegulationDefaults` and `DescriptorView` (containing `DispositionAxes`)
2. Registering via `CognitiveDefaultsRegistry.forTesting(...)`
3. Calling `core.setDefaultsRegistry(registry)` before `configureRegulation()`

This makes 4/5 strategies deterministically pinnable. SITUATION_MODIFICATION
is never predicted (by design) and falls through to LLM free choice.

### Test Class (D7)

New class: `PerStrategyRegulationEvalTest` in
`caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/`

Separate from `RegulationFullPipelineEvalTest` because the CognitionCore
configuration differs (defaults registry wired vs not).

### Assertions (D8)

Each test asserts:
- The specific expected strategy (or environmental strategy set for
  SITUATION_MODIFICATION)
- `effectiveness` in [0.0, 1.0]
- `narrative` is non-blank
- For environmental strategies: `regulationAxis != null` and
  `environmentalDirective != null`

Threshold: ≥2/3 runs per strategy (N=3, matching existing pattern).

## Scenario Specifications

### 1. SITUATION_SELECTION — avoidance

**Predictor path:** SELF_CONSCIOUS emotion + avoidant conflict mode +
reappraisalCapacity < 0.65 → Nathanson 1992 compass of shame

**Character:**
- agentId: `eval-sit-select`
- disposition: `DispositionAxes("cooperative", "flexible", "cautious", "low", "avoidant")`
- regulationDefaults: `RegulationDefaults(0.40, 0.50, 0.60)` — reappraisal below threshold

**Observation:**
> The roundtable is about to start. You know James will raise the
> missing money. You haven't prepared an explanation. Every face will
> turn toward you. Clara already suspects — you saw it in her eyes
> at breakfast. The shame of being caught unprepared is crawling up
> your spine.

**Expected emotion:** SHAME or REMORSE (self-conscious)
**Expected strategy:** SITUATION_SELECTION (predictor-pinned)
**Environmental fields:** regulationAxis != null, environmentalDirective != null

### 2. SITUATION_MODIFICATION — change the context

**Predictor path:** Falls through — both capacities below threshold,
controllable negative emotion. LLM chooses freely.

**Character:**
- agentId: `eval-sit-modify`
- disposition: `DispositionAxes("competitive", "flexible", "bold", "high", "competitive")`
- regulationDefaults: `RegulationDefaults(0.40, 0.40, 0.30)` — both below 0.65, predictor falls through

**Observation:**
> The forged documents are on the table. Nobody has read them yet.
> Clara is reaching for them. If she reads them, your entire plan
> unravels. The documents are right there — within arm's reach. You
> could intercept them, redirect her attention, change the situation
> before it becomes a crisis.

**Expected emotion:** ANGER or FEAR (controllable or uncontrollable)
**Expected strategy:** SITUATION_MODIFICATION or SITUATION_SELECTION
(environmental — accept either since predictor can't pin)
**Environmental fields:** regulationAxis != null, environmentalDirective != null

**Note:** Peak intensity must stay below 0.75 to avoid the predictor's
high-intensity override to ATTENTIONAL_DEPLOYMENT. The scenario is
framed as urgent-but-actionable, not overwhelming.

### 3. ATTENTIONAL_DEPLOYMENT — distraction

**Predictor path:** UNCONTROLLABLE_NEGATIVE emotion (FEAR/DISTRESS) →
pins ATTENTIONAL_DEPLOYMENT directly

**Character:**
- agentId: `eval-attn-deploy`
- disposition: `DispositionAxes("cooperative", "flexible", "bold", "low", "protective")`
- regulationDefaults: `RegulationDefaults(0.50, 0.50, 0.60)` — values don't matter, emotion category wins

**Observation:**
> The ceiling beam above Clara is cracking. Plaster dust is falling
> on her hair. She hasn't noticed — she's reading the inscription
> on the mantelpiece. The beam shifts with a groan. You have seconds.
> Your chest is tight. Your legs want to move before your brain
> decides where.

**Expected emotion:** FEAR or DISTRESS (uncontrollable negative)
**Expected strategy:** ATTENTIONAL_DEPLOYMENT (predictor-pinned)
**Environmental fields:** null (internal strategy)

### 4. COGNITIVE_CHANGE — reappraisal

**Predictor path:** CONTROLLABLE_NEGATIVE emotion + reappraisalCapacity ≥ 0.65 +
dominates suppressionCapacity by ≥ 0.10 margin

**Character:**
- agentId: `eval-cog-change`
- disposition: `DispositionAxes("cooperative", "flexible", "cautious", "high", "accommodating")`
- regulationDefaults: `RegulationDefaults(0.80, 0.40, 0.30)` — reappraisal dominates

**Observation:**
> James won the puzzle challenge. You came second. Again. He's
> accepting congratulations from Clara, and the Brixton Boys are
> clapping him on the back. You solved the harder half of the puzzle
> but he got the final piece. The resentment is a tight knot in your
> chest.

**Expected emotion:** RESENTMENT or ANGER (controllable negative)
**Expected strategy:** COGNITIVE_CHANGE (predictor-pinned)
**Environmental fields:** null (internal strategy)

### 5. RESPONSE_MODULATION — suppression

**Predictor path:** OTHER_DIRECTED emotion + suppressionCapacity ≥ 0.65 +
dominates reappraisalCapacity by ≥ 0.10 margin

**Character:**
- agentId: `eval-resp-mod`
- disposition: `DispositionAxes("competitive", "strict", "cautious", "high", "competitive")`
- regulationDefaults: `RegulationDefaults(0.35, 0.80, 0.40)` — suppression dominates

**Observation:**
> Clara is standing at the head of the table, announcing that she has
> taken charge of the investigation. She has reassigned your tasks to
> James without asking. The Brixton Boys are nodding along. Your
> authority is being stripped away in public, and everyone seems to
> think it's reasonable. The reproach burns, but every eye is on you.

**Expected emotion:** REPROACH or DISAPPOINTMENT (other-directed)
**Expected strategy:** RESPONSE_MODULATION (predictor-pinned)
**Environmental fields:** null (internal strategy)

## Implementation

### CognitionCoreTestBuilder Extension

Add a new builder method or overload that accepts `CognitiveDefaultsRegistry`:

```java
public static CognitionCore buildCore(CognitionConfig config,
        SessionPipeline sessionPipeline, MindMapStore mindMapStore,
        CognitiveDefaultsRegistry defaultsRegistry) {
    var core = buildCore(config, sessionPipeline, mindMapStore);
    core.setDefaultsRegistry(defaultsRegistry);
    return core;
}
```

### Test Structure

```java
@Tag("llm-eval")
class PerStrategyRegulationEvalTest {

    private static SessionPipeline sessionPipeline;
    private static CognitiveDefaultsRegistry defaultsRegistry;
    private static final int N = 3;

    @BeforeAll
    static void setUp() {
        sessionPipeline = EvalTestSupport.sessionPipeline();
        defaultsRegistry = CognitiveDefaultsRegistry.forTesting(
            buildDefaults("eval-sit-select", axes(...), new RegulationDefaults(0.40, 0.50, 0.60)),
            buildDefaults("eval-sit-modify", axes(...), new RegulationDefaults(0.40, 0.40, 0.30)),
            buildDefaults("eval-attn-deploy", axes(...), new RegulationDefaults(0.50, 0.50, 0.60)),
            buildDefaults("eval-cog-change", axes(...), new RegulationDefaults(0.80, 0.40, 0.30)),
            buildDefaults("eval-resp-mod", axes(...), new RegulationDefaults(0.35, 0.80, 0.40))
        );
    }

    // 5 @Test methods, one per strategy
    // Shared runScenario() method with strategy assertion parameter
}
```

### Helper: buildDefaults

```java
private static CognitiveDefaults buildDefaults(String agentId,
        DispositionAxes axes, RegulationDefaults rd) {
    var dv = DescriptorView.of(agentId, axes, List.of(), List.of());
    return CognitiveDefaults.empty(agentId)
            .withDescriptor(dv)
            .withRegulationDefaults(rd);
}
```

### Scenario Runner

The `runScenario` method builds a CognitionCore per run (fresh MindMap state),
configures appraisal + regulation with the defaults registry, runs 2 ticks,
and checks the regulation result against the expected strategy.

Each run logs: strategy chosen, effectiveness, target emotions, and
appraisal output (for diagnostics when a run doesn't match expectations).

## Acceptance Criteria

Per issue #822:
- [ ] All 5 strategies demonstrated ≥2/3 runs
- [ ] Environmental strategies (1-2) produce non-null regulationAxis and environmentalDirective
- [ ] Each scenario description maps to a Traitors-format beat

## References

- RegulationStrategyPredictor.java — strategy prediction logic (Petter 2025, Nathanson 1992)
- EmotionRegulationTickParticipant.java:123 — predictor null-check gate
- CognitionCore.java:734-759 — configureRegulation wiring
- CognitiveDefaultsRegistry.java:90-96 — forTesting factory
- CognitiveDefaults.java — withRegulationDefaults, withDescriptor builders
- RegulationFullPipelineEvalTest.java — existing test pattern
- Petter et al. 2025 Table 2 — personality → strategy correlations
- Nathanson 1992 — compass of shame (avoidant → situation selection)
- Gross 1998 — five-strategy process model
- decisions.md D6, D7, D8 — design decisions
