# CAPS Regulation Integration — Design Spec

**Issue:** casehubio/neocortex#806
**Date:** 2026-11-09
**Decisions:** D20–D27

## Overview

Encode emotion regulation strategies as CAPS output nodes so that the same connectionist settling dynamics that produce behavioral tendencies (approach, withdraw, fight, flight) also produce regulation strategy selections (cognitive change, attentional deployment, suppression, etc.). The LLM regulation call becomes a fallback for novel situations where CAPS doesn't settle confidently.

This is Phase 2 of the regulation architecture. Phase 1 (#590) built LLM-driven regulation outside CAPS. Phase 2 mechanises the validated patterns as CAPS nodes while preserving the LLM as a fallback.

### CASPER Alignment

Camacho et al. (2025) CASPER (Cognitive-Affective Social Processing and Emotion Regulation) decomposes social-emotional processing into interacting subsystems: cognitive control, affective reactivity, social processing, and regulatory processes. Our design maps to CASPER's architecture:

- **Cognitive control** → existing mediating nodes (self_efficacy, self_worth) that gate cognitive_change
- **Affective reactivity** → existing affect_systems nodes (BIS/BAS/FFFS, arousal_level) that drive attentional_deployment and response_modulation
- **Social processing** → new social-context input nodes routed through mediating nodes, providing personality-sensitive social signal processing
- **Regulatory processes** → new regulation output nodes that settle alongside behavioral outputs

CASPER models these subsystems as interacting rather than sequential — consistent with our single-pass settling design (D26). Where we diverge: CASPER is a neurobiological decomposition (prefrontal, amygdala, insula); our CAPS implementation is a functional decomposition (personality-weighted cognitive-affective units). The functional level is appropriate for behavioral simulation — we model the *what* of regulation strategy selection, not the *how* of neural implementation. CASPER validates the theoretical architecture of integrating social processing with emotion regulation in one system.

## Architecture

### Topology Extension

Add 14 nodes to the existing `caps-topology.yaml` (D20, D23, D26):

**10 regulation output nodes** (new `regulation` output category):

| Node ID | Strategy | Source |
|---------|----------|--------|
| `situation_selection` | SITUATION_SELECTION | Gross 1998 |
| `situation_modification` | SITUATION_MODIFICATION | Gross 1998 |
| `attentional_deployment` | ATTENTIONAL_DEPLOYMENT | Gross 1998 |
| `cognitive_change` | COGNITIVE_CHANGE | Gross 1998 |
| `response_modulation` | RESPONSE_MODULATION | Gross 1998 |
| `social_support_seeking` | SOCIAL_SUPPORT_SEEKING | Zaki & Williams 2013 |
| `social_sharing` | SOCIAL_SHARING | Zaki & Williams 2013 |
| `social_withdrawal` | SOCIAL_WITHDRAWAL | Zaki & Williams 2013 |
| `co_regulation` | CO_REGULATION | Zaki & Williams 2013 |
| `deliberate_provocation` | DELIBERATE_PROVOCATION | Zaki & Williams 2013 |

All regulation output nodes use `UNIPOLAR` range `[0, 1]`.

**4 social-context input nodes** (new `social_context` input category):

| Node ID | What it represents | Literature |
|---------|-------------------|-----------|
| `perceived_responsiveness` | Felt understanding, validation, caring from present agent | Reis et al. 2004, Ruan et al. 2019, Main et al. 2025 |
| `target_distress` | Observable distress in another agent | Reeck et al. 2016 (social regulatory cycle) |
| `social_isolation` | Absence of social contact / ostracism | Rimé 2009, Williams 2007 |
| `power_asymmetry` | Relational power advantage (perceived impunity) | Schilke et al. 2021, Tedeschi & Felson 1994, Anderson & Bushman 2002 |

### Connection Architecture

Social-context input nodes connect to existing mediating nodes, following the same INPUT → MEDIATING → OUTPUT layering as all other CAPS inputs (D22). The mediating layer is where personality-weighted processing happens — routing social-context signals through it makes social regulation personality-sensitive through the same mechanism as behavioral outputs.

**Social-context → mediating node connections** (new):

| Social-context input | Target mediating node | Weight | Rationale |
|---------------------|----------------------|--------|-----------|
| `perceived_responsiveness` | `other_safety` | +0.35 | Responsive interaction partner → feel safe (per-tick modulation, not permanent IWM change) |
| `perceived_responsiveness` | `other_reliability` | +0.25 | Responsive interaction partner → current reliability assessment elevated |
| `target_distress` | `arousal_level` | +0.30 | Observing distress → vicarious arousal (Preston & de Waal 2002) |
| `social_isolation` | `BIS_activation` | +0.30 | Ostracism activates behavioral inhibition (Williams 2007) |
| `social_isolation` | `arousal_level` | +0.25 | Social isolation elevates arousal |
| `power_asymmetry` | `social_comparison` | +0.35 | Power differential activates social comparison processing (Festinger 1954) |
| `power_asymmetry` | `BAS_activation` | +0.25 | Power advantage increases approach motivation |

**Intrapersonal strategy connections** (via existing mediating nodes):

| Regulation output | Source node | Weight | Rationale |
|-------------------|------------|--------|-----------|
| `attentional_deployment` | `arousal_level` | +0.45 | High arousal overwhelms cognitive resources → redirect focus. Primary driver (Baranczuk 2019: N→AD r≈.30-.40) |
| `attentional_deployment` | `threat_sensitivity` | +0.35 | Threat sensitivity drives attention redirection |
| `attentional_deployment` | `BIS_activation` | +0.30 | Behavioral inhibition promotes disengagement |
| `cognitive_change` | `self_efficacy` | +0.45 | Reappraisal requires cognitive resources (Gross & John 2003: O→reappraisal r=.36) |
| `cognitive_change` | `self_worth` | +0.30 | Positive self-model supports cognitive flexibility |
| `cognitive_change` | `BIS_activation` | -0.25 | Anxiety impairs reappraisal (Baranczuk 2019: N→reappraisal r=-.25) |
| `response_modulation` | `BIS_activation` | +0.40 | Suppression is the BIS-driven strategy (Gross & John 2003: N→suppression r=.30) |
| `response_modulation` | `self_worth` | -0.25 | Low self-worth drives suppression ("I shouldn't feel this way") |
| `response_modulation` | `self_efficacy` | -0.25 | Low efficacy → can't reappraise → suppress |
| `situation_selection` | `threat_sensitivity` | +0.40 | High threat anticipation drives avoidance |
| `situation_selection` | `BIS_activation` | +0.35 | Behavioral inhibition → avoidance |
| `situation_selection` | `self_worth` | -0.25 | Low self-worth → avoidance |
| `situation_modification` | `self_efficacy` | +0.40 | Changing the situation requires agency (Bandura 1977) |
| `situation_modification` | `BAS_activation` | +0.35 | Approach motivation → active coping |

**Social strategy connections** (via mediating nodes, with justified direct connections noted):

| Regulation output | Source node | Weight | Rationale |
|-------------------|------------|--------|-----------|
| `social_support_seeking` | `other_safety` | +0.35 | Safety with other enables support seeking. `perceived_responsiveness` reaches via mediating layer. |
| `social_support_seeking` | `other_reliability` | +0.30 | Trust in other enables seeking |
| `social_support_seeking` | `BIS_activation` | +0.30 | Anxiety drives comfort-seeking |
| `social_sharing` | `other_safety` | +0.35 | Safety enables disclosure. `perceived_responsiveness` reaches via `other_safety`. |
| `social_sharing` | `BAS_activation` | +0.30 | Approach motivation for sharing |
| `social_withdrawal` | `other_safety` | -0.30 | Unsafe others → withdraw. `social_isolation` reaches via `BIS_activation`. |
| `social_withdrawal` | `BIS_activation` | +0.35 | Inhibition drives withdrawal |
| `co_regulation` | `target_distress` | +0.40 | Observing distress directly drives caregiving (direct — George & Solomon 1999) |
| `co_regulation` | `other_reliability` | +0.30 | Value of the other enables co-regulation |
| `co_regulation` | `self_efficacy` | +0.30 | Capability to help |
| `deliberate_provocation` | `power_asymmetry` | +0.40 | Power advantage enables provocation (direct — Tedeschi & Felson 1994). Also routes through `social_comparison`/`BAS_activation` for personality modulation. |
| `deliberate_provocation` | `BAS_activation` | +0.35 | Approach motivation for provocation |
| `deliberate_provocation` | `dominance` (input) | +0.35 | Dominance drives provocation |

All weights are initial estimates derived from Baranczuk 2019 meta-analysis effect sizes, Gross & John 2003 personality correlations, and consistency with existing topology weight range (±0.25 to ±0.55). Subject to empirical calibration (D25).

### Connection Weight Initialization (D25)

Initial weights derived from two sources, cross-validated:

1. **Literature-derived weights**: Big Five → strategy correlations from Baranczuk 2019 meta-analysis, Gross & John 2003 reappraisal/suppression × personality, mapped to CAPS mediating node dimensions.

2. **Predictor cross-validation**: Run the CAPS network with literature weights against existing eval scenarios. Compare CAPS regulation attractor output with the predictor's known-good strategy selections. Resolve divergence via three-way comparison (D25 protocol):
   - CAPS ≠ predictor, LLM agrees with CAPS → predictor mapping is the outlier
   - CAPS ≠ predictor, LLM agrees with predictor → CAPS weights need adjustment
   - LLM disagrees with both → flag for manual review (shared-source bias)

3. **Ground truth validation**: Curate a set of ground truth scenarios with expert-rated strategy selections (independent of both predictor and LLM). Use these as the PRIMARY validation reference for weight calibration. The three-way protocol (above) serves as the SECONDARY resolution mechanism for scenarios where ground truth is ambiguous or absent.

Weight provenance: `CONSENSUS` for literature-derived weights.

**Rescorla-Wagner learning for regulation connections**: Post-initialization, regulation connection weights update via the existing `CapsEngine.updateWeights()` mechanism. The regulation effectiveness signal maps as follows:

| Parameter | Regulation mapping | Source |
|-----------|-------------------|--------|
| `outcomeIntensity` | `abs(peakOriginalIntensity - regulatedIntensity)` — magnitude of intensity change achieved by regulation | `EmotionRegulationResult.peakOriginalIntensity()`, `.regulatedIntensity()` |
| `outcomeValence` | `effectiveness > 0.5 ? +1.0 : -1.0` — binary: regulation succeeded or failed | `EmotionRegulationResult.effectiveness()` |
| `salienceMultiplier` | `peakOriginalIntensity` — more intense emotions produce stronger learning signals | `EmotionRegulationResult.peakOriginalIntensity()` |
| `reinforcementSchedule` | `"variable_ratio"` — regulation success is probabilistic | Constant |

Learning applies to ALL connections (behavioral + regulation). The outcome signal from regulation episodes updates the same weight matrix that behavioral outcomes update. This is theoretically correct — a regulation episode IS a behavioral outcome that updates the agent's associative structure.

The `EmotionRegulationMindMapAdapter.write()` already persists `effectiveness`, `peakOriginalIntensity`, and `regulatedIntensity` per regulation episode. The weight update is performed within `EmotionRegulationTickParticipant.tick()` itself, at the end of the DERIVED phase — immediately after the regulation result is recorded and persisted to MindMap. The `inputActivations` map (assembled earlier in the same tick for `CapsEngine.settle()`) is still in scope as a local variable, so no cross-phase handoff is needed. This eliminates the need for a separate `RegulationWeightUpdateParticipant` and avoids the problem of reconstituting or storing inputActivations across phases.

Sequence within `EmotionRegulationTickParticipant.tick()`:
1. Assemble `inputActivations` (social-context + situation classification)
2. `CapsEngine.settle(state, inputActivations)` → `SettlingResult`
3. Cascade resolution → pinned or free strategy
4. `EmotionRegulationGenerationPhase.generate()` → `EmotionRegulationResult`
5. Persist result to MindMap
6. `CapsEngine.updateWeights(state, inputActivations, ...)` with regulation effectiveness mapped per the table above

### Disposition Modulation

Regulation output nodes receive disposition-based resting activations via `DispositionWeightMapper`, following the same pattern as existing behavioral outputs:

| DispositionAxes value | Regulation resting activations |
|-----------------------|-------------------------------|
| `conflictMode: avoidant` | situation_selection +0.2, social_withdrawal +0.15 |
| `conflictMode: analytical` | cognitive_change +0.2 |
| `conflictMode: cooperative` | co_regulation +0.15, social_sharing +0.1 |
| `conflictMode: competitive` | response_modulation +0.15, deliberate_provocation +0.1 | **Note:** existing `DispositionWeightMapper.setRestingActivations()` uses `"confrontational"` for this axis value (line 161), inconsistent with the topology YAML and `RegulationStrategyPredictor` which both use `"competitive"`. This spec corrects the existing code to `"competitive"` as part of the mapper extension. |
| `riskAppetite: conservative` | situation_selection +0.15, attentional_deployment +0.1 |
| `riskAppetite: bold` | situation_modification +0.2, cognitive_change +0.1 |
| `socialOrient: cooperative` | social_support_seeking +0.15, co_regulation +0.1 |
| `socialOrient: independent` | response_modulation +0.15, attentional_deployment +0.1 |

These are initial estimates — subject to literature calibration alongside connection weights.

### Existing Agent Migration

Existing agents have persisted `AgentCapsState` in SQLite without regulation node states. For **connection weights**, migration is unnecessary: `CapsSettler` falls back to `conn.defaultWeight()` for missing weights (line 78). For **resting activations**, migration IS necessary: `CapsSettler` falls back to `NodeState.DEFAULT` (restingActivation = 0.0) for missing nodes — existing agents with `conflictMode: avoidant` would NOT get `situation_selection +0.2` without patching.

**On-demand patching**: Add `CapsEngine.ensureTopologyCoverage(AgentCapsState state, DispositionAxes disposition)` that checks for missing regulation node states and applies regulation resting activations from `DispositionWeightMapper`. Called by `EmotionRegulationTickParticipant` after `loadState()`, before settling. Idempotent — if regulation nodes already have state, the call is a no-op.

This handles agents created before and after the topology extension uniformly. No batch migration script needed.

## Strategy Selection Cascade (D21)

Three-layer cascade with safety hard gate and empirical promotion criteria:

```
Safety hard gate (fatigue > 0.7 OR peakIntensity > 0.75)?
  ├─ YES → force ATTENTIONAL_DEPLOYMENT, skip entire cascade
  └─ NO
      └─ CAPS settling → regulation attractors?
          ├─ Single dominant attractor (strength ≥ regulationConfidenceThreshold
          │   AND dominance margin ≥ regulationDominanceMargin over runner-up)
          │   → pin strategy, send to LLM for narrative
          ├─ Multiple attractors above threshold but no clear dominant
          │   → CAPS is ambivalent, fall through
          └─ No attractor above threshold → fall through
              └─ RegulationStrategyPredictor → prediction?
                  ├─ YES → pin strategy, send to LLM for narrative
                  └─ NO → LLM free choice
```

**Safety hard gate**: The predictor's fatigue and intensity overrides (RegulationStrategyPredictor lines 35-42) are resource-based hard limits, not personality predictions. They must fire unconditionally — before CAPS, before the predictor, before the LLM. These are extracted from the predictor and applied as the first step of the cascade. An agent with fatigue 0.9 gets ATTENTIONAL_DEPLOYMENT regardless of what CAPS settling produces — cognitive resources for reappraisal are exhausted (Petter et al. 2025). The predictor retains the safety overrides as well (defense in depth) for the case where it runs as layer 2.

**Multi-attractor resolution**: When multiple regulation attractors exceed `regulationConfidenceThreshold`, the highest-strength attractor wins ONLY if it exceeds the runner-up by `regulationDominanceMargin`. If the margin is not met, CAPS is ambivalent — the cascade falls through to the predictor/LLM. This preserves the "CAPS confident vs. not confident" semantics: close competition between regulation strategies indicates genuine ambivalence, not false confidence.

**Threshold configuration** (new `RegulationCascadeConfig` record):

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `regulationConfidenceThreshold` | 0.3 | Minimum attractor strength for cascade pin decision. Architecturally separate from `CapsSettlingConfig.attractorStrengthMinFilter()` (default 0.1), which is a noise filter for `SettlingResult.attractors()`. A regulation attractor at 0.12 passes the noise filter but does NOT override the predictor/LLM. Production value derived empirically from eval data (D24). |
| `regulationDominanceMargin` | 0.15 | Minimum strength gap between top two regulation attractors. Below this margin, CAPS is ambivalent and falls through. |

The existing `CapsSettlingConfig.attractorStrengthMinFilter()` (default 0.1) remains unchanged — it governs what appears in `SettlingResult.attractors()` for ALL output categories. `RegulationCascadeConfig` governs the cascade PIN decision for regulation attractors only.

**Predictor retirement criteria** (D21):
- PRIMARY: CAPS concordance with ground truth eval scenarios ≥ 80%
- SECONDARY: CAPS-LLM free-choice concordance ≥ 70%
- Degradation monitoring: automatic fallback if concordance drops > 10% below thresholds

The predictor retains safety overrides internally (fatigue > 0.7 → force AD, intensity > 0.75 → force AD) as defense in depth, but the safety hard gate at cascade entry makes them redundant during normal operation.

## Social-Context Input Injection

The 4 social-context input nodes are injected per-tick from `RegulationContext` + MindMap relationship data. Computed as continuous [0,1] activations:

| Node | Formula | Inputs |
|------|---------|--------|
| `perceived_responsiveness` | `clamp01(trustWeight × trust + trajectoryWeight × trajectorySignal)` where `trust` = mean trust edge value for interacting agents from MindMap (normalized to [0,1]), `trajectorySignal` = 1.0 if affect trajectory is IMPROVING, 0.5 if STABLE, 0.0 if WORSENING. Default weights: `trustWeight = 0.6`, `trajectoryWeight = 0.4`. Multiple interacting agents: use max trust (most responsive agent dominates). | MindMap trust edges, `RegulationTracker` affect trajectories |
| `target_distress` | `clamp01((-pleasure + arousal) / 2.0)` where `pleasure` and `arousal` are PAD dimensions from MindMap overlay for interacting agents, normalized to [-1,1] and [0,1] respectively. Multiple agents: use max distress. Zero if no interacting agents. | MindMap PAD overlays (from `RelationalSchema`) |
| `social_isolation` | `1.0 - clamp01(agentCount / saturatingCount)` where `agentCount` = `RegulationContext.interactingAgents().size()`, `saturatingCount` = 3 (configurable). Gradient: 0 agents → 1.0, 1 agent → 0.67, 2 agents → 0.33, 3+ agents → 0.0. | `RegulationContext.interactingAgents()` |
| `power_asymmetry` | `clamp01((dominanceScore - submissionScore + socialComparisonBias) / 2.0)` where `dominanceScore` and `submissionScore` are from MindMap relationship edges for the most dominant interacting agent, `socialComparisonBias` is from `CognitiveProfile.socialComparisonBias()` (default 0.0). | MindMap relationship edges, `CognitiveProfile` |

`clamp01(x) = max(0, min(1, x))`

Injection happens in `EmotionRegulationTickParticipant` before calling `CapsEngine.settle()`. Social-context activations are merged with situation classification activations into the `inputActivations` map.

## Integration with Existing Pipeline

The regulation CAPS integration touches `EmotionRegulationTickParticipant`, which orchestrates the cascade:

1. **AppraisalTickParticipant** produces `AppraisalResult` with emotions (DERIVED phase)
2. **EmotionRegulationTickParticipant** (DERIVED phase):
   a. Computes regulation effort (Bosse et al. 2010)
   b. If effort above noise floor:
      - **Safety hard gate**: if fatigue > 0.7 OR peakIntensity > 0.75 → force ATTENTIONAL_DEPLOYMENT, skip CAPS and predictor
      - Assembles social-context input activations from RegulationContext + MindMap
      - Merges with existing situation classification activations
      - Calls `CapsEngine.settle()` with merged input activations
      - Extracts regulation attractors from `SettlingResult` (category = `regulation`)
      - Applies multi-attractor resolution: dominant attractor with sufficient margin → pin strategy
      - Else → fall through to `RegulationStrategyPredictor`
      - Else → LLM free choice
   c. Calls `EmotionRegulationGenerationPhase.generate()` with pinned or free strategy
   d. Records fatigue, monitors for failure, persists to MindMap
   e. Calls `CapsEngine.updateWeights()` with the same `inputActivations` and regulation effectiveness

### Goal Namespace Isolation (D3, D27)

**Code change required:** `GoalProposalOrchestrator` currently has no namespace isolation — `registerGoals()` performs a full replace of ALL registered goals (verified: `state.registeredGoals = List.copyOf(goals)` at line 120). If `EmotionRegulationGoalParticipant` calls `registerGoals()` after `ProactiveRegulationParticipant` calls `appendGoals()`, proactive avoidance goals are silently wiped.

This spec proposes adding a new method to `GoalProposalOrchestrator`:

```java
public void registerParticipantGoals(String participantId, String agentId,
                                      String tenantId, List<DriveGoalProposal> goals)
```

Internal implementation: `GoalProposalState` gains a `Map<String, List<DriveGoalProposal>> namespacedGoals` field. `registerParticipantGoals()` replaces only the specified participant's namespace. `currentProposals()` merges all namespaces with `cachedProposals`. Both `EmotionRegulationGoalParticipant` and `ProactiveRegulationParticipant` migrate from `registerGoals()`/`appendGoals()` to `registerParticipantGoals()` with participant-specific IDs.

FOUNDATION phase ordering is undefined; participants are namespace-independent.

## Settling Dynamics

Regulation and behavioral outputs settle together in one pass (D26). This is faithful to CAPS theory — Mischel & Shoda (1995) explicitly include self-regulatory processes as interacting CAUs.

**Type rename:** `BehavioralAttractor` → `CapsAttractor`. Regulation strategies are CAPS outputs but not "behavioral" in the narrow sense — the type name should reflect that both behavioral and regulatory attractors are CAPS output categories distinguished by the `category` field. This is a breaking rename; the migration is mechanical (record rename, all callers updated). `SettlingResult.attractors()` returns `List<CapsAttractor>`.

**No interference between output categories**: verified from `CapsSettler.java` — output nodes are terminal in the graph (incoming edges only, no outgoing). Each output's activation is computed independently from its incoming weights and source activations. No normalization, no softmax, no cross-output competition. Adding regulation output nodes cannot change behavioral attractor strengths or convergence behavior.

**Distortions and regulation nodes**: Existing distortion definitions target `self_model`, `other_model`, `world_model`, `response_outcome`, `threat_processing`, and `affect_systems` categories. Regulation output nodes (category `regulation`) are NOT targeted by distortions. This is architecturally correct: distortions operate on the mediating layer, which feeds regulation outputs. Regulation outputs already consume distortion-affected mediating activations — applying distortions to regulation outputs would double-count. For example, `catastrophizing` amplifies `threat_processing` activations, which increases `attentional_deployment` through the `threat_sensitivity → attentional_deployment` connection. The distortion effect is already present in the regulation output without targeting it directly.

**Projected topology size**: 91 nodes (77 baseline + 10 regulation outputs + 4 social inputs), 110 connections (76 baseline + 7 social-context→mediating + 14 intrapersonal regulation + 13 social regulation). All connections enumerated in §Connection Architecture tables above. Per-iteration cost increases proportionally (~18% more nodes) but iteration count is unchanged — output nodes don't affect convergence dynamics.

## Avoidance Temporal Decay (D5, revised)

**Breaking change to `ProactiveRegulationParticipant`**: The constructor parameter `decayHorizonTicks` (linear semantics: weight reaches zero at this tick count) is replaced by `decayTimeConstant` (exponential semantics: weight reaches 37% at this tick count). All callers constructing `ProactiveRegulationParticipant` must update.

New constructor signature:
```java
public ProactiveRegulationParticipant(MindMapStore mindMapStore,
                                      GoalProposalOrchestrator goalOrchestrator,
                                      CognitiveDefaultsRegistry defaultsRegistry,
                                      String cognitiveSubgraphId,
                                      ProactiveAvoidanceMode mode,
                                      int decayTimeConstant,      // was: decayHorizonTicks
                                      double tickIntervalSeconds)
```

Default `decayTimeConstant`: 30 ticks.

Proactive avoidance weight uses exponential decay (Ebbinghaus 1885, Wixted & Ebbesen 1991):

```
avoidanceWeight = initialWeight * Math.exp(-elapsedTicks / decayTimeConstant)
```

Decay profile: 37% at 30 ticks, 14% at 60 ticks, 5% at 90 ticks. Smooth asymptotic approach — no cliff, unlike the prior linear decay model where weight drops abruptly to zero at `decayHorizonTicks`.

## Eval Strategy

Run the existing eval suite with CAPS regulation settling alongside the predictor:

1. **Paired comparison**: for each eval scenario, record CAPS regulation attractor (strategy + strength), predictor strategy (+ confidence), and LLM free-choice strategy
2. **Threshold calibration**: derive minimum CAPS strength at which CAPS matches predictor/LLM at acceptable rates
3. **Weight calibration**: adjust connection weights where CAPS diverges from ground truth, using the D25 protocol (three-way resolution as secondary mechanism for ambiguous cases)
4. **Regression monitoring**: verify behavioral attractors are unchanged by the topology extension

## References

- Mischel, W. & Shoda, Y. (1995). A cognitive-affective system theory of personality. *Psychological Review*, 102(2), 246-268.
- Gross, J. J. (1998). The emerging field of emotion regulation. *Review of General Psychology*, 2(3), 271-299.
- Gross, J. J. & John, O. P. (2003). Individual differences in two emotion regulation processes. *Journal of Personality and Social Psychology*, 85(2), 348-362.
- Zaki, J. & Williams, W. C. (2013). Interpersonal emotion regulation. *Emotion*, 13(5), 803-810.
- Baranczuk, U. (2019). The five factor model of personality and emotion regulation: A meta-analysis. *Personality and Individual Differences*, 139, 217-227.
- Petter, J. et al. (2025). Personality × emotion-type regulation strategy predictions.
- Reis, H. T., Clark, M. S. & Holmes, J. G. (2004). Perceived partner responsiveness as an organizing construct. In *Handbook of Closeness and Intimacy* (pp. 201-225).
- Ruan, Y. et al. (2019). PPR and emotional expression/support-seeking.
- Main, A., Yung, S. T., Chen, Y. & Zawadzki, M. J. (2025). Predicting emotion regulation strategies from aspects of the social context. *J. Social and Personal Relationships*, 42(2), 568-590.
- Reeck, C., Ames, D. R. & Ochsner, K. N. (2016). The social regulation of emotion. *Trends in Cognitive Sciences*, 20(1), 47-63.
- Niven, K., Totterdell, P. & Holman, D. (2009). A classification of controlled interpersonal affect regulation strategies. *Emotion*, 9(4), 498-509.
- Rimé, B. (2009). Emotion elicits the social sharing of emotion. *Emotion Review*, 1(1), 60-85.
- Williams, K. D. (2007). Ostracism. *Annual Review of Psychology*, 58, 425-452.
- Schilke, O., Reimann, M. & Cook, K. S. (2021). Power asymmetry and trust.
- Tedeschi, J. T. & Felson, R. B. (1994). *Violence, Aggression, and Coercive Actions*. American Psychological Association.
- Anderson, C. A. & Bushman, B. J. (2002). Human aggression. *Annual Review of Psychology*, 53, 27-51.
- Camacho, T., Deshpande, G. & Perino, M. T. (2025). CASPER: Cognitive-Affective Social Processing and Emotion Regulation. *Neuropsychopharmacology*, 51, 136-152.
- Preston, S. D. & de Waal, F. B. M. (2002). Empathy: Its ultimate and proximate bases. *Behavioral and Brain Sciences*, 25(1), 1-20.
- George, C. & Solomon, J. (1999). Attachment and caregiving: The caregiving behavioral system. In *Handbook of Attachment* (pp. 649-670).
- Bosse, T., Pontier, M. & Treur, J. (2010). A computational model based on Gross' emotion regulation theory. *Cognitive Systems Research*, 11, 211-230.
- Ebbinghaus, H. (1885). *Memory: A Contribution to Experimental Psychology*.
- Wixted, J. T. & Ebbesen, E. B. (1991). On the form of forgetting. *Psychological Science*, 2(6), 409-415.
- Bonanno, G. A. & Burton, C. L. (2013). Regulatory flexibility. *Clinical Psychology Review*, 33(5), 591-597.
- CapsSettler.java (settling algorithm, convergence, attractor extraction)
- CapsTopologyLoader.java (YAML topology loading)
- DispositionWeightMapper.java (personality → weight modulation)
- RegulationStrategyPredictor.java (existing personality × strategy prediction)
- EmotionRegulationTickParticipant.java (regulation orchestration)
- caps-topology.yaml (existing 77-node, 76-connection topology)
