# Emotion Regulation: Post-Appraisal Strategy Selection

**Issue:** casehubio/neocortex#590
**Branch:** issue-590-emotion-regulation
**Date:** 2026-11-04

## Overview

Characters currently feel emotions but cannot regulate them. This design adds a post-appraisal emotion regulation system based on Gross's (1998, 2015) process model, implementing all five regulation strategy families as a unified decision layer with distributed execution.

Regulation is the "I wanted to but held back" moment — the gap between feeling an emotion and expressing it. Without it, characters are purely reactive. With it, characters can suppress gloating, reappraise a threat as a challenge, distract from anxiety, avoid triggering situations, or modify their environment to reduce emotional load.

## Literature Foundation

| Model | Contribution | How used |
|-------|-------------|----------|
| Gross 1998 | Five strategy families: situation selection, situation modification, attentional deployment, cognitive change, response modulation | Strategy taxonomy — the enum of what regulation can do |
| Gross 2015 (EPM) | Identify → select → implement → monitor cycle | Control loop structure — the LLM prompt models this cycle |
| Bosse, Pontier & Treur 2010 | Quantitative dynamics: homeostatic baseline, persistence factor (β), adaptation rate (ζ) | Threshold calibration and adaptation — informs prompt context for cumulative regulation effects |
| Petter et al. 2025 | Strategy choice model: high intensity → distraction, low → reappraisal; cognitive load gates reappraisal | Strategy selection guidance — provided to LLM as structured decision factors |

## Architecture

### Pipeline Placement

A new `RegulationTickParticipant` runs in the **DERIVED** phase of `CognitionCore.tick()`, after `AppraisalTickParticipant`. It reads `AppraisalResult` (emotions, action tendencies) and writes `RegulationResult` to MindMap for downstream consumption.

```
DERIVED phase execution:
  1. drives.tick()                    ← DriveOrchestrator computes DriveAxis intensities
  2. AppraisalTickParticipant         ← Scherer SEC → CognitiveEmotion + ActionTendency
  3. RegulationTickParticipant [NEW]  ← reads appraisal, writes RegulationResult to MindMap
  4. GutFeelingParticipant            ← existing
  5. CapsSettler                      ← CAPS attractor settling
```

**Ordering constraint:** The participant list for DERIVED is insertion-ordered. The factory method that configures regulation must register it after appraisal. This convention must be documented and tested — a unit test should verify that `RegulationTickParticipant` appears after `AppraisalTickParticipant` in the participant list for `CognitionPhase.DERIVED`.

### Activation Gate

Regulation does not fire every tick. A mechanical discrepancy threshold gates when the LLM generation call is made:

- After appraisal produces `CognitiveEmotion` instances, compute the discrepancy: `|emotion.intensity - baseline|`
- If discrepancy exceeds a personality-derived threshold, regulation fires
- Threshold is lower for high-neuroticism characters (they regulate more often) and higher for emotionally stable characters

**Bootstrap values:** Initial thresholds derived from Bosse et al. (2010) parameterization. Tuned empirically via A/B evaluation. Starting point: baseline intensity = 0.3, threshold = 0.4 (regulation fires when emotion intensity > 0.7). These are starting estimates — empirical testing will reveal the right values.

### Regulation as a Decision Layer

The core insight: regulation is a **decision system**, not an execution system. The `RegulationTickParticipant` selects a strategy from all five Gross families. The **implementation** is distributed to existing pipeline components based on strategy type:

| Strategy family | Target | Implementation consumer |
|----------------|--------|------------------------|
| Situation selection | Environment | Goal pipeline — avoidance constraints |
| Situation modification | Environment | Goal pipeline — situation-change goals |
| Attentional deployment | Attention/salience | `renderInternalState()` — attention redirection in prompt |
| Cognitive change | Appraisal meaning | `renderInternalState()` — reframed drive descriptions |
| Response modulation | Expression | `renderInternalState()` — dampened drive rendering |

**Internal strategies (3-5):** Consumed by `CognitionObservationBuilder.renderInternalState()`. The integration point is `resolveAdaptedDrives()` and `appraisalFilterDrives()`, which read drive-intensity nodes from MindMap. When a `RegulationResult` exists for the current tick, drive rendering is modified — suppressed drives get dampened text, reappraised drives get reframed text.

**Environmental strategies (1-2):** Consumed by the goal pipeline. This requires extending `DriveGoalFormationContext` to accept regulation directives:

```java
record DriveGoalFormationContext(
    String agentId, String tenantId,
    DriveAxis axis, double intensity, String trigger,
    List<AgentGoal> existingGoals, int remainingCapacity,
    RegulationDirective regulationDirective  // NEW — nullable, carries avoidance/modification intent
)
```

The preferred approach is a new `RegulationGoalFormationStrategy` that reads `RegulationResult` directly from MindMap — this avoids modifying the existing `DriveGoalFormationContext` record and breaking existing implementations. The `DriveGoalFormationContext` extension is a fallback if the separate strategy proves unwieldy.

### Regulation Target: Emotions Trigger, Drives Are Modulated

Regulation operates across two layers:

- **Trigger:** `CognitiveEmotion` from `AppraisalResult` — the discrepancy detection reads emotion intensity to decide whether regulation is needed
- **Modulation surface:** Character-level drives from MindMap (scheming, gloating, gallantry, dominance, etc.) — regulation modifies how these drives are rendered in the LLM prompt

This aligns with Gross: regulation operates on the emotional **response** (how the feeling manifests behaviorally), not on the appraisal itself. In this architecture, character drives ARE the behavioral response surface — they're how emotions manifest as motivational pulls.

### LLM Generation

The regulation LLM call follows the `CognitionCore.appraiseMood()` pattern:

1. Open a `SessionPipeline` session with a regulation system prompt
2. Provide structured context: current emotions (from appraisal), personality traits, strategy selection factors (intensity, cognitive load), recent regulation history
3. Stream and parse JSON response into `RegulationResult`
4. Fallback: if the LLM call fails or times out, decay toward no-regulation (same as mood fallback pattern `MOOD_FALLBACK_DECAY`)

**System prompt structure:**

```
You are modeling the emotion regulation process for a character.

REGULATION STRATEGIES (Gross 1998):
[five strategy descriptions with when-to-use guidance]

STRATEGY SELECTION FACTORS (Petter 2025):
- High intensity emotions → attentional deployment (distraction)
- Low-moderate intensity → cognitive change (reappraisal)
- Cognitive load high → simpler strategies (distraction, suppression)
- Cognitive load low → complex strategies (reappraisal, situation modification)

CHARACTER PERSONALITY:
[Big Five scores with regulation tendency mappings]
- High neuroticism → more suppression, more frequent regulation
- High openness → more reappraisal, more cognitive change
- High agreeableness → more situation modification in social contexts
- High conscientiousness → more response modulation (self-control)

CURRENT STATE:
[emotions from appraisal, drive intensities, recent regulation history]

REGULATION HISTORY (last 3-5 ticks):
[what strategies were used, cumulative effects]
Note: repeated suppression increases physiological strain (Gross empirical finding).
Adaptation: strategies used repeatedly become less effective (Bosse 2010 ζ parameter).

Respond with JSON: {strategy, targetEmotion, targetDrives[], originalIntensity, 
regulatedIntensity, effectiveness, narrative}
```

### Data Model

```java
record RegulationResult(
    RegulationStrategy strategy,      // Gross 1998 five-family enum
    String targetEmotion,             // CognitiveEmotion that triggered regulation
    List<String> targetDrives,        // character drives being modulated
    double originalIntensity,         // emotion intensity before regulation
    double regulatedIntensity,        // emotion intensity after regulation
    double effectiveness,             // 0.0-1.0 how well the strategy worked
    String narrative                  // character-experience text for prompt rendering
)

enum RegulationStrategy {
    SITUATION_SELECTION,       // Gross strategy 1 — avoid triggering contexts
    SITUATION_MODIFICATION,    // Gross strategy 2 — change the situation
    ATTENTIONAL_DEPLOYMENT,    // Gross strategy 3 — redirect focus (distraction)
    COGNITIVE_CHANGE,          // Gross strategy 4 — reappraise meaning
    RESPONSE_MODULATION        // Gross strategy 5 — suppress/amplify expression
}
```

### Rendering in Internal State

Two channels reinforce the regulation signal in the LLM prompt:

**1. Regulation ObservationSection** — a new section in `renderInternalState()`:

```
## Regulation
You are actively regulating your emotional response.
Strategy: cognitive change (reappraisal)
Target: the anger you feel toward Muttley's betrayal
You are reframing this as a competitive challenge rather than a personal attack.
Effectiveness: moderate — the reframe holds but the sting persists.
```

**2. Modified drive rendering** — drives affected by regulation get modified text:

```
# Before regulation:
"gloating" (intensity: 0.8) — "A triumphant heat rises in your chest..."

# After regulation (response modulation/suppression):
"gloating" (intensity: 0.8, suppressed to 0.4) — "The triumph is there but you 
hold it back — your jaw tightens with the effort of keeping your expression neutral."
```

The modified drive text should integrate with Barrett's 4-channel framework (somatic, attentional, temporal, action tendency):

- **Suppression:** somatic channel shows physiological cost (increased tension, sympathetic activation)
- **Reappraisal:** temporal channel shifts (reframing changes how the moment feels)
- **Distraction:** attentional channel redirects (focus moves away from the trigger)

### Temporal Context (Cross-Tick Awareness)

Each tick's regulation is independent, but the generation phase prompt includes regulation history from MindMap (last 3-5 ticks). The LLM adapts naturally:

- "You have been suppressing anger for 4 ticks" → strain accumulates
- "Reappraisal worked well last tick" → strategy persistence
- "Distraction is losing effectiveness" → strategy switching

**MindMap storage:** Regulation results stored as nodes in `SubgraphTypes.COGNITIVE` with `cognitiveKind = "regulation-result"` and `agent-id` property. Queried via `MindMapQuery.of(tenantId, limit)` to retrieve recent history by count (not time — MindMap doesn't track tick numbers).

**Pruning:** Regulation nodes older than the history window (configurable, default 5) can be pruned during consolidation to prevent unbounded accumulation.

### Interaction with Existing Systems

**Interpersonal regulation (`RegulationTracker`):** The existing `RegulationTracker` in `CognitionObservationBuilder` tracks OTHER characters' emotion trajectories via Theory of Mind. This new system handles INTRAPERSONAL regulation (a character's own emotions). The two are complementary — a character can be regulating their own anger toward someone whose distress they're also tracking via RegulationTracker. The spec acknowledges both systems must coexist; naming convention distinguishes them (`RegulationResult` for intrapersonal, `RegulationTracker` remains for interpersonal).

**Social regulation strategies:** Gross 2015 and Zaki & Williams 2013 describe social regulation (seeking comfort, co-regulation, social sharing). These are out of scope for this issue — documented as a known extension point for future work. The five-strategy enum is extensible if social strategies are added later.

**Mood bridge:** Regulation output does not modify mood directly. Mood is derived from appraisal via PAD projection. However, sustained successful regulation may influence mood indirectly through the memory system (regulation experiences become memories that inform future mood appraisal).

### Personality-Strategy Connection

Both personality-trait tendencies and current-state factors are provided as structured prompt context. The LLM integrates them rather than a mechanical priority rule:

**Trait tendencies (from Big Five):**
| Trait | High → | Low → |
|-------|--------|-------|
| Neuroticism | More suppression, lower threshold | Less regulation needed |
| Openness | More reappraisal | More response modulation |
| Agreeableness | More situation modification | More avoidance |
| Conscientiousness | More response modulation | Less deliberate regulation |
| Extraversion | More social strategies (future) | More attentional deployment |

**State factors (per-tick):**
- Emotion intensity (Petter 2025): high → distraction, low → reappraisal
- Cognitive load: high → simpler strategies, low → complex strategies
- Regulation history: repeated strategy → declining effectiveness (Bosse 2010)

## Evaluation Strategy

### A/B with Regression Gate

1. **Baseline run:** Same scenario with regulation disabled (toggle via configuration flag)
2. **Regulation run:** Same scenario with regulation enabled
3. **Regression check:** Existing drive expression eval (`classify_emotions.py`) scores must not regress beyond threshold on the regulation run
4. **Regulation scoring:** New eval dimension measures regulation quality:
   - Strategy appropriateness (does the chosen strategy match intensity/personality?)
   - Narrative coherence (does the regulation text make sense in context?)
   - Personality consistency (does a neurotic character regulate differently from a stable one?)
   - Selectivity (does regulation fire at appropriate moments, not every tick?)

### Configuration Toggle

A boolean flag enables/disables the regulation participant for A/B comparison:

```properties
manor.cognition.regulation.enabled=true   # default true
manor.cognition.regulation.threshold=0.4  # activation threshold, tunable
```

### Eval Workflow

```bash
# 1. Baseline run (regulation off)
# Start server with regulation disabled, run scenario, save transcript
# 2. Regulation run (regulation on)  
# Start server with regulation enabled, same scenario, save transcript
# 3. Compare
python3 wacky-manor/docs/eval/classify_emotions.py docs/eval/<baseline-run>
python3 wacky-manor/docs/eval/classify_emotions.py docs/eval/<regulation-run>
# 4. Check regression gate + score regulation quality
```

## Implementation Scope

All five strategies designed and implemented together — no batching. The unified decision layer handles all strategies; distributed execution is the only variation (renderInternalState for internal, goal pipeline for environmental).

### New Components (neocortex)
- `RegulationTickParticipant` — DERIVED phase participant, runs after appraisal
- `RegulationResult` / `RegulationStrategy` — data model
- `RegulationGenerationPhase` — LLM call for strategy selection + narrative (appraiseMood pattern)
- Regulation ObservationSection in `CognitionObservationBuilder`
- Modified drive rendering in `resolveAdaptedDrives()` / `appraisalFilterDrives()`
- `RegulationGoalFormationStrategy` for environmental strategies (preferred over modifying `DriveGoalFormationContext`)
- Configuration properties for enable/disable toggle and threshold

### Modified Components (neocortex)
- `CognitionCore` — register RegulationTickParticipant in DERIVED phase (after appraisal)
- `CognitionObservationBuilder.renderInternalState()` — read RegulationResult, render regulation section, modify drive text
- Goal formation — register RegulationGoalFormationStrategy alongside existing strategies

### Test Components (examples/wacky-manor)
- A/B eval harness (configuration toggle)
- Regulation eval dimension (classify_emotions.py extension or new script)
- Unit tests for activation gate, strategy selection, rendering integration

## References

- Gross, J.J. (1998). The emerging field of emotion regulation. *Review of General Psychology*, 2(3), 271-299.
- Gross, J.J. (2015). Emotion regulation: Current status and future prospects. *Psychological Inquiry*, 26(1), 1-26.
- Bosse, T., Pontier, M. & Treur, J. (2010). A computational model based on Gross' emotion regulation theory. *Cognitive Systems Research*, 11(3), 211-230.
- Petter, J. et al. (2025). Emotion regulation, fast or slow. *Emotion*, 25(5), 1273-1292.
- Zaki, J. & Williams, W.C. (2013). Interpersonal emotion regulation. *Emotion*, 13(5), 803-810.
- `CognitionCore.java` — tick phase architecture, appraiseMood() pattern
- `AppraisalTickParticipant.java` — Scherer SEC appraisal, AppraisalResult
- `CognitionObservationBuilder.java` — renderInternalState(), resolveAdaptedDrives(), RegulationTracker
- `DriveOrchestrator.java` / `DriveComposer.java` — ModulationLayer, DriveAxis
- `DriveDescriptionGenerationPhase.java` — Barrett 4-channel framework
- `DriveGoalFormationContext.java` — GoalFormationStrategy API
- `classify_emotions.py` — drive expression eval framework
- Protocol PP-20261017-d5853d — literature citation requirement for model-derived constructs
- Decision review: `/Users/mdproctor/reviews/casehub-slots/issue-590-decision-20261104-015751/`
