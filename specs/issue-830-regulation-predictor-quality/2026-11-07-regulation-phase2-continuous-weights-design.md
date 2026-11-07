# Phase 2 Design: Continuous Weights + Shame Dimensions

**Issue:** casehubio/neocortex#830
**Date:** 2026-11-07
**Phase:** 2 of 2
**Branch:** `issue-830-regulation-predictor-quality`
**Predecessor:** Phase 1 design (`2026-11-06-regulation-predictor-quality-design.md`)
**Literature:** `neocortex/docs/research/2026-11-07-regulation-continuous-weights-literature-review.md`

## Summary

Replace the boolean `isSuppressor()`/`isReappraiser()` classification in `RegulationStrategyPredictor` with continuous capacity weights derived from `CognitiveDerivationEngine`. Add a `ShameContext` that assembles memory compounding, relational closeness, and personality-gated depth for self-conscious emotions. Render both through the existing `buildUserMessage()` interpretive pattern.

## Architecture

Three layers, each doing what it's good at:

```
CDE (startup, static)          Tick Participant (runtime, per-tick)       Generation Phase (LLM)
─────────────────────          ─────────────────────────────────────      ───────────────────────
deriveRegulationDefaults()  →  predictor uses continuous capacities   →  receives pinned strategy
  RegulationDefaults            OR fall-through with context               OR axes + context
  - reappraisalCapacity                                                 
  - suppressionCapacity         assembleShameContext() when               renders interpretive
  - redirectTendency            SELF_CONSCIOUS detected                   narrative sections
                                  ShameContext                            (not raw data)
                                  - episodeCount, recency
                                  - relationalCloseness, role
                                  - grandiosity, callousness
```

## Layer 1: CDE RegulationDefaults

### New record: `RegulationDefaults`

```java
package io.casehub.neocortex.cognitive.index;

public record RegulationDefaults(
    double reappraisalCapacity,   // [0, 1] — ability to execute cognitive change
    double suppressionCapacity,   // [0, 1] — ability to sustain expression control
    double redirectTendency       // [0, 1] — tendency toward attentional deployment / avoidance
) {}
```

### New CDE pathway: `deriveRegulationDefaults(DispositionAxes)`

Follows the existing derivation pattern (`deriveMoodBaseline`, `deriveSocialCognition`). Static `Map<String, Double>` lookup tables per axis, summed across all 5 axes, clamped to [0, 1].

#### Weight Tables (literature-ordered initial values)

Derived from Barańczuk 2019 meta-analytic r values through the disposition axis → Big Five mapping documented in the literature review.

**Reappraisal Capacity** — who can execute cognitive change

| Axis | Value | Weight | Big Five basis |
|------|-------|--------|---------------|
| conflictMode | analytical | +0.30 | O (+0.20) + C (+0.19) |
| conflictMode | cooperative | +0.15 | A (+0.18) |
| conflictMode | avoidant | -0.15 | N (-0.19) |
| conflictMode | competitive | +0.05 | E (+0.22) offset by low-A |
| autonomy | high | +0.25 | O (+0.20) + E (+0.22) |
| autonomy | moderate | +0.05 | — |
| autonomy | low | -0.10 | low-O, low-E |
| ruleFollowing | strict | +0.15 | C (+0.19) |
| ruleFollowing | flexible | +0.10 | O (+0.20) offset by low-C |
| ruleFollowing | moderate | +0.05 | — |
| riskAppetite | bold | +0.20 | low-N (+0.19 inverse) + O |
| riskAppetite | calculated | +0.05 | — |
| riskAppetite | conservative | -0.10 | N (-0.19) |
| socialOrient | cooperative | +0.10 | A (+0.18) |
| socialOrient | independent | +0.05 | moderate O |
| socialOrient | competitive | +0.05 | E offset by low-A |

Baseline: 0.5 + sum of applicable weights, clamped to [0, 1].

Example: analytical/cooperative/calculated/high/strict → 0.5 + 0.30 + 0.10 + 0.05 + 0.25 + 0.15 = 1.0 (clamped) → strong reappraiser.
Example: avoidant/cooperative/conservative/low/moderate → 0.5 - 0.15 + 0.10 - 0.10 - 0.10 + 0.05 = 0.30 → weak reappraiser.

**Suppression Capacity** — who can sustain effective expression control

| Axis | Value | Weight | Big Five basis |
|------|-------|--------|---------------|
| conflictMode | avoidant | +0.25 | Habitual suppression; E (-0.37 for expression) |
| conflictMode | competitive | +0.15 | Low-A, expression control as dominance |
| conflictMode | analytical | -0.10 | O prefers cognitive engagement |
| conflictMode | cooperative | -0.05 | A has modest negative r with suppression |
| socialOrient | competitive | +0.10 | Low-A |
| socialOrient | cooperative | -0.05 | A |
| socialOrient | independent | +0.05 | — |
| riskAppetite | conservative | +0.15 | Risk-averse → suppression as safe default |
| riskAppetite | calculated | +0.05 | — |
| riskAppetite | bold | -0.10 | Low-N, prefers approach |
| autonomy | low | +0.10 | Low-E → suppression as default |
| autonomy | moderate | +0.05 | — |
| autonomy | high | -0.05 | High-E, high-O → prefers expression |
| ruleFollowing | strict | +0.05 | C, controlled expression |
| ruleFollowing | moderate | 0.00 | — |
| ruleFollowing | flexible | -0.05 | Low-C |

Baseline: 0.5 + sum, clamped to [0, 1].

**Redirect Tendency** — tendency toward attentional deployment / situation selection

| Axis | Value | Weight | Big Five basis |
|------|-------|--------|---------------|
| conflictMode | avoidant | +0.25 | N (+0.31 avoidance) |
| conflictMode | cooperative | +0.05 | — |
| conflictMode | analytical | -0.15 | O prefers engagement |
| conflictMode | competitive | -0.10 | Approach orientation |
| riskAppetite | conservative | +0.20 | N (+0.31) |
| riskAppetite | calculated | +0.05 | — |
| riskAppetite | bold | -0.15 | Low-N, approach |
| autonomy | low | +0.10 | Withdrawal tendency |
| autonomy | moderate | +0.05 | — |
| autonomy | high | -0.10 | Self-direction, engagement |
| socialOrient | independent | +0.05 | Some withdrawal |
| socialOrient | cooperative | 0.00 | — |
| socialOrient | competitive | -0.05 | Approach |
| ruleFollowing | strict | 0.00 | — |
| ruleFollowing | moderate | 0.00 | — |
| ruleFollowing | flexible | +0.05 | Less bound to resolve |

Baseline: 0.3 + sum, clamped to [0, 1]. Lower baseline because redirect is situational, not a default.

### Integration with CognitiveDefaults

`RegulationDefaults` becomes a field on `CognitiveDefaults`, derived alongside existing fields. Accessed via `CognitiveDefaultsRegistry.forAgentOrDefaults(agentId).regulationDefaults()`.

## Layer 1b: Predictor Changes

### Replace boolean classification with continuous capacities

Current:
```java
if (isSuppressor(axes))  → pin RESPONSE_MODULATION
if (isReappraiser(axes)) → pin COGNITIVE_CHANGE
else                     → fall through
```

New:
```java
RegulationDefaults rd = defaultsRegistry.forAgentOrDefaults(agentId).regulationDefaults();

// Use capacity thresholds instead of boolean classification
if (rd.reappraisalCapacity() >= REAPPRAISAL_THRESHOLD)  → pin COGNITIVE_CHANGE
if (rd.suppressionCapacity() >= SUPPRESSION_THRESHOLD)   → pin RESPONSE_MODULATION
if (rd.redirectTendency() >= REDIRECT_THRESHOLD)         → pin ATTENTIONAL_DEPLOYMENT
else                                                     → fall through with axes + context
```

**Threshold initial values:** 0.65 for all three. Calibrated via end-to-end eval — lower threshold pins more cases, higher threshold lets more fall through to LLM.

**Priority order:** Reappraisal > Suppression > Redirect. Literature basis: reappraisal is more effective long-term (Webb 2012 d+ values); if a personality has capacity for both, prefer reappraisal.

**Emotion-category routing preserved:** The existing `predictFromEmotionAndPersonality` dispatch by emotion category stays. Continuous capacities replace the boolean checks within each category handler, not the category dispatch itself.

**Intensity and fatigue overrides preserved:** High intensity (> 0.75) → AD and high fatigue (> 0.7) → AD still override personality-based prediction. These are validated by Petter 2025.

### What changes in the predictor signature

`predict()` currently takes `DispositionAxes`. With continuous weights, it takes `RegulationDefaults` instead (or in addition). The tick participant already has `CognitiveDefaultsRegistry` — it looks up `RegulationDefaults` and passes them to the predictor.

## Layer 2: ShameContext

### New record: `ShameContext`

```java
package io.casehub.neocortex.cognition.regulation;

public record ShameContext(
    int episodeCount,              // shame episodes in recent window
    @Nullable Instant mostRecentEpisode, // recency of last shame event
    double relationalCloseness,    // [0, 1] — intimacy with shame subject
    @Nullable String relationalRole,     // stranger/ally/mentor/protectee/etc
    double grandiosity,            // [0, 1] — from PersonalityFacet
    double callousness             // [0, 1] — from PersonalityFacet
) {
    public boolean isCompounding() {
        return episodeCount >= 3;
    }

    public boolean isRelationallyCharged() {
        return relationalCloseness >= 0.5;
    }

    public boolean isSurfaceLevel() {
        return grandiosity >= 0.7 && callousness >= 0.5;
    }
}
```

### Assembly in tick participant

When `EmotionRegulationTickParticipant.tick()` detects a SELF_CONSCIOUS dominant emotion and the predictor falls through:

1. **Episode count:** Query `EmotionRegulationMindMapAdapter.loadHistory()` (already loaded for regulation history), filter by `targetEmotions` containing SHAME/REMORSE, count recent entries.

2. **Relational closeness:** Get the shame emotion's `subjectId` from the `CognitiveEmotion`. Look up the person overlay node in `MindMapStore` for that subject. Extract `RelationalSchema` (trust, intimacy, role). Use `intimacy` as relational closeness.

3. **Grandiosity × callousness:** Look up `PersonalityProfile.facets()` from `CognitiveDefaultsRegistry`. Extract GRANDIOSITY and CALLOUSNESS scores.

### Data source accessibility

| Field | Source | Already injected? |
|-------|--------|-------------------|
| episodeCount | `MindMapStore` → regulation history filter | Yes |
| mostRecentEpisode | Same as above | Yes |
| relationalCloseness | `MindMapStore` → person overlay → `RelationalSchema.intimacy()` | Yes |
| relationalRole | Same → `RelationalSchema.role()` | Yes |
| grandiosity | `CognitiveDefaultsRegistry` → `PersonalityProfile.facets()` | Yes |
| callousness | Same | Yes |

No new CDI producers required.

## Layer 3: Generation Phase — Interpretive Rendering

### RegulationDefaults rendering (always, when available)

When the predictor falls through, the generation phase already receives disposition axes (Phase 1). Phase 2 adds: if `RegulationDefaults` are available, render a capacity summary section:

```
Your personality profile suggests:
- Reappraisal capacity: [low/moderate/strong] — [interpretive sentence]
- Suppression capacity: [low/moderate/strong] — [interpretive sentence]
- Redirect tendency: [low/moderate/strong] — [interpretive sentence]

The predictor did not pin a specific strategy because no single capacity
is dominant. Choose the strategy this personality would naturally reach for
given these capacities.
```

The interpretive sentences are not the numbers — they're what the numbers mean. Example for low reappraisal (0.30): "This personality's avoidant conflict style and conservative risk appetite limit cognitive flexibility — genuine reframing requires a willingness to engage with uncomfortable perspectives that this personality resists."

### ShameContext rendering (SELF_CONSCIOUS fall-through only)

Three conditional sections, following the Phase 1 pattern:

**Memory compounding** (when `episodeCount >= 3`):
```
This is part of a recurring pattern — the character has experienced shame
[episodeCount] times recently. Each cycle undermines the self-narrative
further. The shame is no longer about this specific act but about the
pattern the acts reveal. Regulation should acknowledge the accumulated
weight, not treat this as an isolated episode.
```

**Relational charging** (when `relationalCloseness >= 0.5`):
```
The subject of this shame is someone the character feels [role-based
relationship description] toward (intimacy: [level]). The failure isn't
"I made a mistake" — it's "I failed [them/someone I'm responsible for]."
This changes the quality of the self-evaluation from performance failure
to relational betrayal of trust.
```

**Surface-level gating** (when `grandiosity >= 0.7 && callousness >= 0.5`):
```
This character's grandiosity and emotional callousness mean shame doesn't
penetrate to identity. They will reframe the failure externally rather
than internalize it — the self-concept is defended by attribution patterns
that redirect blame. Regulation should match this shallow penetration.
Heavy containment or deep temporal staging would be disproportionate.
```

### Interaction with Phase 1 guidance

Phase 1's self-conscious emotion guidance (temporal containment for suppressors) is preserved. ShameContext sections are rendered AFTER the Phase 1 guidance, providing additional context the LLM can integrate. They don't replace the temporal containment guidance — they refine it with runtime specifics.

## Eval Extension

### End-to-end eval changes

1. **Existing 8 cases:** Re-run with continuous weights. Expect: some fall-throughs become predictions (incoherent-anger, mixed-anger may now get pinned). Strategy mean should stay >= 4.0.

2. **New shame-specific cases:** Add test cases that exercise the three shame dimensions:
   - Compounding shame (high episode count) — verify the LLM acknowledges pattern, not isolated event
   - Relational shame (high intimacy subject) — verify the LLM differentiates relational vs performance failure
   - Surface shame (high grandiosity + callousness) — verify the LLM doesn't over-regulate

3. **Temporal containment preserved:** suppressor-shame must still score >= 4 on temporal containment. Phase 2 additions must not break Phase 1's guidance.

### New judge dimension for shame cases

Add a "shame depth appropriateness" judge that scores whether the regulation response matches the shame context:
- Compounding + relational → deep regulation expected (4-5 for depth-appropriate response)
- Surface-level personality → light regulation expected (4-5 for NOT over-regulating)

## Files Changed

### New files
- `cognitive-index/.../RegulationDefaults.java` — record
- `cognition/.../regulation/ShameContext.java` — record

### Modified files
- `cognitive-index/.../CognitiveDerivationEngine.java` — add `deriveRegulationDefaults(DispositionAxes)`
- `cognitive-index/.../CognitiveDefaults.java` — add `regulationDefaults` field
- `cognition/.../regulation/RegulationStrategyPredictor.java` — replace boolean classification with continuous capacity thresholds
- `cognition/.../regulation/EmotionRegulationTickParticipant.java` — assemble `ShameContext`, pass `RegulationDefaults` to predictor
- `cognition/.../regulation/EmotionRegulationGenerationPhase.java` — render `RegulationDefaults` and `ShameContext` sections in `buildUserMessage()`
- `caps-llm-classifier/.../eval/RegulationEndToEndEvalTest.java` — add shame-specific test cases, new judge dimension

### Test files
- `cognitive-index/...CognitiveDerivationEngineTest.java` — test weight table derivation for known axis combinations
- `cognition/.../regulation/RegulationStrategyPredictorTest.java` — test continuous capacity thresholds
- Unit tests for `ShameContext` assembly

## Calibration Plan

1. Implement weight tables with literature-ordered initial values
2. Run end-to-end eval — measure strategy mean and per-case scores
3. Compare pinned vs fall-through distribution to Phase 1 baseline
4. Adjust thresholds if cases that should fall through are being pinned (or vice versa)
5. Adjust weights if specific axis combinations produce wrong capacity ordering
6. Target: strategy mean >= 4.4 (match or improve Phase 1 baseline)

## References

- Barańczuk 2019 — meta-analytic r values (Big Five × 8 regulation strategies, 132 studies)
- Hughes et al. 2020 — personality → EPM stages (identification, selection, implementation capacity)
- Petter et al. 2025 — computational model of intensity → strategy choice
- Webb et al. 2012 — strategy effectiveness d+ values
- Nathanson 1992 — compass of shame, temporal containment
- Yang et al. 2025 — ERP evidence for temporal suppression in avoidant attachment
- Phase 1 findings: `neocortex/docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md`
- Literature review: `neocortex/docs/research/2026-11-07-regulation-continuous-weights-literature-review.md`
- End-to-end eval: `RegulationEndToEndEvalTest.java` (baseline: strategy mean 4.4, temporal containment 5.0)
- Decisions: D5 (layered architecture), D6 (literature-ordered weights)
