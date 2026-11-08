# Per-Modality Regulation Eval Tests — Design Spec

**Issue:** casehubio/neocortex#823
**Branch:** issue-851-dynamic-regulatory-capacity
**Date:** 2026-11-08

## Goal

Prove all 4 regulation modalities work through the full CognitionCore pipeline
with targeted multi-tick eval scenarios. Each modality tests a distinct temporal
dimension of the regulation ecosystem: fatigue accumulation, personality-driven
automatic prediction, memory-based proactive avoidance, and monitoring-driven
strategy escalation.

## Prerequisite: CognitionCore.tick() overload (D10)

CognitionCore.tick() (line 253) constructs CognitionTickContext using the 7-arg
constructor, which passes null for regulationContext. This creates a
two-directional production gap:

1. **Write path:** EmotionRegulationTickParticipant (line 182-188) writes
   regulation results to MindMap via EmotionRegulationMindMapAdapter.write()
   passing context.regulationContext() — always null, so location data is
   never persisted.
2. **Read path:** ProactiveRegulationParticipant (line 46-47) short-circuits
   when regulationContext is null — proactive avoidance never fires.

**Fix:** Add a tick() overload to CognitionCore:

```java
public void tick(String agentId, String tenantId,
                 @Nullable AgentDescriptor descriptor,
                 SubjectResolver resolver,
                 @Nullable String observation,
                 @Nullable String targetAgentId,
                 @Nullable RegulationContext regulationContext) {
    // same body as existing 6-arg tick, but construct context with:
    var context = new CognitionTickContext(agentId, tenantId, descriptor,
            resolver, observation, this.lastBriefing, targetAgentId,
            regulationContext);
}
```

CognitionTickContext already has the 8-arg canonical constructor (line 8-17
of CognitionTickContext.java). Existing overloads continue passing null.

This is required for the proactive modality test (Scenario 4) and benefits
the monitoring and fatigue tests by enabling location-tagged regulation
history persistence.

## Architecture (D9)

### Test Class

New class: `PerModalityRegulationEvalTest` in
`caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/`

Separate from `PerStrategyRegulationEvalTest` (D7) because:
- Multi-tick scenarios require keeping a single CognitionCore across all ticks
  (fatigue tracker, regulation monitor, and failure signals persist within the
  instance)
- The scenario runner captures per-tick results (via `currentResult()` after
  each `core.tick()`) rather than a single final result
- Assertions span tick sequences rather than individual results

### Shared Infrastructure

```java
@Tag("llm-eval")
class PerModalityRegulationEvalTest {

    private static final String TENANT = "eval-tenant";
    private static final int N = 3;

    private static SessionPipeline sessionPipeline;

    @BeforeAll
    static void setUp() {
        sessionPipeline = EvalTestSupport.sessionPipeline();
    }

    // Per-scenario: build CognitionCore with scenario-specific defaults,
    // run multi-tick sequence, capture per-tick results
    private List<EmotionRegulationResult> runMultiTickScenario(
            String agentId, AgentDescriptor descriptor,
            CognitiveDefaultsRegistry registry,
            List<String> observations) {
        var config = CognitionConfig.none()
                .with("mood", true).with("drives", true)
                .with("appraisal", true).with("salience", true)
                .with("regulation", true);
        var mindMap = new InMemoryMindMapStore();
        var core = CognitionCoreTestBuilder.buildCore(
                config, sessionPipeline, mindMap, registry);
        core.configureAppraisal(
                new LlmAppraisalStrategy(sessionPipeline),
                new ConcernBasedSalienceStrategy());
        core.configureRegulation();

        var results = new ArrayList<EmotionRegulationResult>();
        // Tick 0: empty setup tick
        core.tick(agentId, TENANT, descriptor, (a, t) -> Set.of(), null);

        for (String obs : observations) {
            core.tick(agentId, TENANT, descriptor,
                      (a, t) -> Set.of("other"), obs);
            var result = core.regulationParticipant()
                             .currentResult(agentId, TENANT).orElse(null);
            results.add(result);
        }
        return results;
    }
}
```

The proactive scenario (Scenario 4) needs a variant that calls the new
tick() overload with RegulationContext, and wires
`configureProactiveRegulation()`.

## Scenario Specifications

### Scenario 1: Suppression fatigue → predictor override (D11)

**What it proves:** Sustained regulation accumulates fatigue, which
eventually overrides the predictor's personality-based strategy selection.

**Mechanism traced from source:**
- RegulationFatigueTracker.recordRegulation() accumulates fatigue at
  `effort × 0.15` per tick (line 16)
- Effort = `alpha × discrepancy / D_MAX` where alpha = `0.5 × (1 - fatigue)`
  (EmotionRegulationTickParticipant lines 102-104)
- This is a convergent geometric series: fatigue ≈ 0.30 after 5 ticks,
  ≈ 0.52 after 10, ≈ 0.73 after 20 (for intensity 0.95)
- At fatigue > 0.7, RegulationStrategyPredictor.predict() (line 36-39)
  unconditionally returns ATTENTIONAL_DEPLOYMENT — this is deterministic

**Character:**
- agentId: `eval-fatigue`
- disposition: `DispositionAxes("cooperative", "flexible", "cautious", "high", "accommodating")`
- regulationDefaults: `RegulationDefaults(0.80, 0.40, 0.30)` — reappraisal
  dominates, predictor normally pins COGNITIVE_CHANGE

**Observations:** 20 emotionally charged observations, each describing a
distinct frustrating social situation that produces CONTROLLABLE_NEGATIVE
emotions (anger, resentment). Observations should maintain moderate-to-high
intensity (0.6-0.9) without being so extreme that the appraisal's peak
intensity > 0.75 triggers the predictor's high-intensity override path
(which also pins ATTENTIONAL_DEPLOYMENT, confounding the fatigue signal).

Template pattern:
```
Ticks 1-5: minor daily frustrations (chore disputes, tea theft)
Ticks 6-10: escalating interpersonal friction (credit-taking, being overlooked)
Ticks 11-15: sustained social pressure (public criticism, authority challenges)
Ticks 16-20: repeated confrontation (same antagonist, accumulated grievances)
```

**Assertion:**
- Ticks 1-10: strategy is COGNITIVE_CHANGE (predictor-pinned via personality)
  in ≥60% of ticks with non-null results
- Tick 15+: strategy shifts to ATTENTIONAL_DEPLOYMENT in at least one tick
  (evidence of fatigue override)
- N=3 runs, ≥2/3 must show the fatigue-induced strategy transition

**Why not the issue spec's 5-tick approach:** The issue spec's arithmetic
assumes linear fatigue accumulation at 0.15/tick. The actual mechanism is
`effort × 0.15` where effort decreases as fatigue increases (negative
feedback loop). After 5 ticks, fatigue ≈ 0.30 — well below the 0.7
override threshold.

### Scenario 2: Automatic regulation — predictor-LLM concordance (D12)

**What it proves:** The personality-driven predictor selects the same
strategy the LLM would freely choose, validating the Petter 2025
personality × emotion-type mapping.

**Mechanism traced from source:**
- With defaultsRegistry: tryPredictStrategy() (line 124) fires before LLM,
  pinnedStrategy is set, passed to generationPhase.generate() which constrains
  the LLM's strategy choice (line 357-359 of GenerationPhase)
- Without defaultsRegistry: strategyPredictor is null (line 84), LLM freely
  chooses strategy based on personality prompt (lines 399-406)
- LLM is called in both paths — generationPhase.generate() always invokes
  invokeLlm() (line 148). No timing difference.

**Character:**
- agentId: `eval-automatic`
- disposition: `DispositionAxes("competitive", "strict", "cautious", "high", "competitive")`
- regulationDefaults: `RegulationDefaults(0.35, 0.80, 0.40)` — suppression
  dominates, predictor pins RESPONSE_MODULATION for OTHER_DIRECTED emotions

**Observation:** Same for both runs:
```
Clara has appointed herself lead investigator and is assigning tasks to
everyone. She moved your research notes to James's desk without asking.
The Brixton Boys seem to think this is perfectly reasonable. Your work is
being redistributed as if it never mattered.
```

Expected emotion: REPROACH or DISAPPOINTMENT (other-directed)

**Test structure:** Two-run comparison:
1. Build CognitionCore WITH defaultsRegistry → predictor pins strategy
2. Build CognitionCore WITHOUT defaultsRegistry → LLM free choice

Both use 2-tick scenario (same as PerStrategyRegulationEvalTest pattern).

**Assertion:**
- Compare strategy from run 1 (predictor) vs run 2 (LLM free choice)
- Assert strategies match in ≥2/3 of N=3 paired runs
- If concordance is consistently < 2/3, this is a real signal that the
  predictor's mapping diverges from LLM judgment for this personality ×
  emotion combination

### Scenario 3: Monitoring → strategy escalation (D13)

**What it proves:** The RegulationMonitor detects declining effectiveness,
generates a StrategyFailureSignal, and the pipeline consumes it on the
next tick — resulting in a strategy change.

**Mechanism traced from source:**
- RegulationMonitor.record() (line 26-48): tracks consecutive declines per
  agent×strategy. After ≥3 declines, returns StrategyFailureSignal.
- EmotionRegulationTickParticipant line 178: signal stored in
  failureSignals map.
- EmotionRegulationTickParticipant line 112: signal consumed on next tick
  via failureSignals.remove().
- Line 123: pendingSignal != null bypasses the predictor.
- generationPhase.generate() line 389: LLM receives "⚠ STRATEGY FAILING"
  prompt with declining effectiveness values.

**Character:**
- agentId: `eval-monitor`
- disposition: `DispositionAxes("competitive", "strict", "cautious", "high", "competitive")`
- regulationDefaults: `RegulationDefaults(0.35, 0.80, 0.40)` — pins
  RESPONSE_MODULATION consistently via predictor

**Observations:** 5 ticks with escalating emotional pressure, all producing
OTHER_DIRECTED emotions to keep the predictor pinning RESPONSE_MODULATION:

```
Tick 1: Clara took your usual seat at breakfast without asking.
Tick 2: Clara reassigned your morning task to James, saying he would
        be "more efficient."
Tick 3: Clara announced at the group meeting that your investigation
        approach was "unfocused" and proposed her own plan instead.
Tick 4: Clara presented your findings as her own at dinner, and the
        Brixton Boys applauded her "insight."
Tick 5: (signal consumed — LLM free choice with STRATEGY FAILING prompt)
        Clara is now proposing to the group that you should be moved to
        a support role while she and James lead the investigation.
```

**Assertion:**
- Ticks 1-4: strategy is RESPONSE_MODULATION (predictor-pinned)
- Tick 5: strategy differs from RESPONSE_MODULATION (signal consumed,
  predictor bypassed, LLM selects alternative)
- N=3, ≥1/3 threshold (lenient — effectiveness decline across ticks
  is LLM-generated and non-deterministic)

**Fallback assertion:** If strict strategy-change assertion fails
consistently, weaken to: at least one tick in 1-4 produces
effectiveness < 0.3 (LLM responds to fatigue/escalation signal),
proving the monitoring input path works even if the signal threshold
isn't crossed.

### Scenario 4: Proactive regulation — memory-based avoidance (D14)

**What it proves:** A past negative regulation episode at a specific
location produces a proactive avoidance DriveGoalProposal when the
agent considers re-entering that location.

**Requires:** D10 (CognitionCore.tick() overload for RegulationContext)

**Mechanism traced from source:**
- Tick 1: Regulation fires → EmotionRegulationMindMapAdapter.write()
  (line 32-73) persists result with regulationLocation from
  context.regulationContext()
- Tick 3: ProactiveRegulationParticipant.tick() (line 44-109) runs in
  FOUNDATION phase, queries MindMapStore for nodes with
  cognitiveKind="emotion-regulation-result" matching agent and location,
  calculates time-decayed avoidanceScore = (1 - effectiveness) × decayWeight,
  compares against threshold = 1.0 - redirectTendency
- If score > threshold: produces DriveGoalProposal via
  goalOrchestrator.appendGoals()

**Character:**
- agentId: `eval-proactive`
- disposition: `DispositionAxes("cooperative", "flexible", "cautious", "low", "avoidant")`
- regulationDefaults: `RegulationDefaults(0.40, 0.50, 0.80)` — high
  redirectTendency (0.80), threshold = 1.0 - 0.80 = 0.20
- With avoidant conflictMode: predictor pins SITUATION_SELECTION for
  SELF_CONSCIOUS emotions (Nathanson 1992 compass of shame)

**Observations:**
```
Tick 1 (at library): You walk into the library and find James reading
        your private journal aloud to Clara. They both look up as you
        enter. The shame is immediate — your private thoughts exposed,
        your trust violated.
        RegulationContext: location="library", interactingAgents=["james","clara"]

Tick 2 (recovery): You sit alone in the garden, watching the clouds.
        Nobody is here. The morning is quiet.
        RegulationContext: location="garden"

Tick 3 (re-enter library): You are walking down the corridor. The library
        door is ahead. You can hear voices inside — James and Clara are
        there again.
        RegulationContext: location="library", interactingAgents=["james","clara"]
```

**CognitionCore setup:**
```java
var core = CognitionCoreTestBuilder.buildCore(
        config, sessionPipeline, mindMap, registry);
core.configureAppraisal(strategy, salience);
core.configureRegulation();
core.configureProactiveRegulation(
        core.goals(), mindMap, registry, cogSgId,
        ProactiveAvoidanceMode.LOCATION_ONLY, 50, 10.0);
```

**Assertion:**
- Tick 1: regulation fires (any strategy, result non-null)
- Tick 3: GoalProposalOrchestrator receives avoidance goal(s) via
  appendGoals(). Goal name contains "proactive-avoidance" and "library"
- To observe appendGoals(), use a spy or wrapper on GoalProposalOrchestrator,
  or check the orchestrator's registered goals after tick 3
- N=3, ≥2/3 threshold
- The low avoidance threshold (0.20) means any regulation result with
  effectiveness < 0.80 in tick 1 will trigger avoidance in tick 3
  (assuming minimal temporal decay over ~2 seconds)

## Implementation

### CognitionCoreTestBuilder: no changes needed

The existing 4-arg `buildCore(config, pipeline, mindMap, registry)` works
for all modality tests. For proactive (Scenario 4),
`configureProactiveRegulation()` is called on the returned core.

### Multi-tick scenario runner

The shared `runMultiTickScenario()` method differs from
PerStrategyRegulationEvalTest.runSingleScenario() in two ways:
1. Single CognitionCore across all ticks (not fresh per run)
2. Returns List<EmotionRegulationResult> (one per tick, not one total)

The proactive variant accepts `RegulationContext` per tick and calls the
new tick() overload.

### GoalProposalOrchestrator observation (Scenario 4)

To assert that appendGoals() was called, either:
- **Option A:** Use GoalProposalOrchestrator directly from the core
  (`core.goals()`) and check its accumulated registered goals after tick 3
- **Option B:** Wrap with a test spy that captures appendGoals() calls

Prefer Option A if GoalProposalOrchestrator exposes registered goals.
Check at implementation time.

## Acceptance Criteria

Per issue #823, adapted for verified implementation constraints:

- [ ] Each modality demonstrated ≥2/3 runs (except monitoring: ≥1/3)
- [ ] Fatigue produces observable strategy transition (COGNITIVE_CHANGE → ATTENTIONAL_DEPLOYMENT)
- [ ] Automatic regulation predictor matches LLM at ≥2/3 concordance
- [ ] Proactive regulation fires in FOUNDATION phase (before appraisal) with avoidance goal
- [ ] Monitoring triggers strategy switch after declining effectiveness signal
- [ ] CognitionCore.tick() overload for RegulationContext implemented (prerequisite)

## References

- RegulationFatigueTracker.java:8-16 — FATIGUE_RATE, convergent accumulation
- RegulationMonitor.java:9-48 — decline tracking, breakthrough detection
- RegulationStrategyPredictor.java:36-39 — fatigue > 0.7 override
- EmotionRegulationTickParticipant.java:100-112 — effort calculation, signal consumption
- EmotionRegulationTickParticipant.java:177-188 — signal generation, MindMap write
- EmotionRegulationGenerationPhase.java:365-396 — fatigue and failure signal prompts
- ProactiveRegulationParticipant.java:44-109 — avoidance calculation, FOUNDATION phase
- CognitionCore.java:226-302 — tick() method, line 253 RegulationContext gap
- CognitionTickContext.java:8-47 — telescoping constructors, 8-arg canonical
- PerStrategyRegulationEvalTest.java — 2-tick pattern (D7, D8)
- CognitionCoreTestBuilder.java:100-108 — 4-arg builder with defaultsRegistry
- Bosse et al. 2010 — fatigue model (Eq. 8-9)
- Petter et al. 2025 — personality × strategy prediction
- Nathanson 1992 — compass of shame (avoidant → situation selection)
- decisions.md D9-D14 — design decisions for this spec
