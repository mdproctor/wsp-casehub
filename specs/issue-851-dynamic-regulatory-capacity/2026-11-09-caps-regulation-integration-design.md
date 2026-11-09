# CAPS Regulation Integration — Design Spec

**Issue:** casehubio/neocortex#806
**Date:** 2026-11-09
**Decisions:** D20–D27

## Overview

Encode emotion regulation strategies as CAPS output nodes so that the same connectionist settling dynamics that produce behavioral tendencies (approach, withdraw, fight, flight) also produce regulation strategy selections (cognitive change, attentional deployment, suppression, etc.). The LLM regulation call becomes a fallback for novel situations where CAPS doesn't settle confidently.

This is Phase 2 of the regulation architecture. Phase 1 (#590) built LLM-driven regulation outside CAPS. Phase 2 mechanises the validated patterns as CAPS nodes while preserving the LLM as a fallback.

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

Regulation output nodes connect to existing mediating nodes (D22). The same internal state (BIS/BAS, self_worth, threat_sensitivity, arousal_level, etc.) that drives behavioral outputs also drives regulation strategy selection. Regulation is another behavioral response dimension, not a separate system (Mischel & Shoda 1995).

**Intrapersonal strategy connections** (via existing mediating nodes):

| Regulation output | Primary mediating drivers | Rationale |
|-------------------|--------------------------|-----------|
| `attentional_deployment` | arousal_level (+), threat_sensitivity (+), BIS_activation (+) | High arousal/threat overwhelms cognitive resources → redirect focus |
| `cognitive_change` | self_efficacy (+), self_worth (+), BIS_activation (-) | Reappraisal requires cognitive resources and self-belief |
| `response_modulation` | BIS_activation (+), self_worth (-), self_efficacy (-) | Suppression when cognitive resources insufficient for reappraisal |
| `situation_selection` | threat_sensitivity (+), BIS_activation (+), self_worth (-) | Avoidance when threat is anticipated and self-model is negative |
| `situation_modification` | self_efficacy (+), BAS_activation (+) | Changing the situation requires agency and approach motivation |

**Social strategy connections** (via social-context input nodes + existing mediating nodes):

| Regulation output | Primary drivers | Rationale |
|-------------------|-----------------|-----------|
| `social_support_seeking` | perceived_responsiveness (+), other_reliability (+), BIS_activation (+) | Seek comfort from responsive, reliable other when inhibited |
| `social_sharing` | perceived_responsiveness (+), other_safety (+), BAS_activation (+) | Share emotions with safe, responsive other; approach-motivated |
| `social_withdrawal` | social_isolation (+), other_safety (-), BIS_activation (+) | Withdraw when isolated or others feel unsafe |
| `co_regulation` | target_distress (+), other_reliability (+), self_efficacy (+) | Soothe distressed other when capable and other is valued |
| `deliberate_provocation` | power_asymmetry (+), BAS_activation (+), dominance (input, +) | Provoke when power advantage provides perceived impunity |

### Connection Weight Initialization (D25)

Initial weights derived from two sources, cross-validated:

1. **Literature-derived weights**: Big Five → strategy correlations from Baranczuk 2019 meta-analysis, Gross & John 2003 reappraisal/suppression × personality, mapped to CAPS mediating node dimensions.

2. **Predictor cross-validation**: Run the CAPS network with literature weights against existing eval scenarios. Compare CAPS regulation attractor output with the predictor's known-good strategy selections. Resolve divergence via three-way comparison (D25 protocol):
   - CAPS ≠ predictor, LLM agrees with CAPS → predictor mapping is the outlier
   - CAPS ≠ predictor, LLM agrees with predictor → CAPS weights need adjustment
   - LLM disagrees with both → flag for manual review (shared-source bias)

Weight provenance: `CONSENSUS` for literature-derived weights. Rescorla-Wagner learning applies post-initialization with regulation effectiveness as the outcome signal.

### Disposition Modulation

Regulation output nodes receive disposition-based resting activations via `DispositionWeightMapper`, following the same pattern as existing behavioral outputs:

| DispositionAxes value | Regulation resting activations |
|-----------------------|-------------------------------|
| `conflictMode: avoidant` | situation_selection +0.2, social_withdrawal +0.15 |
| `conflictMode: analytical` | cognitive_change +0.2 |
| `conflictMode: cooperative` | co_regulation +0.15, social_sharing +0.1 |
| `conflictMode: competitive` | response_modulation +0.15, deliberate_provocation +0.1 |
| `riskAppetite: conservative` | situation_selection +0.15, attentional_deployment +0.1 |
| `riskAppetite: bold` | situation_modification +0.2, cognitive_change +0.1 |
| `socialOrient: cooperative` | social_support_seeking +0.15, co_regulation +0.1 |
| `socialOrient: independent` | response_modulation +0.15, attentional_deployment +0.1 |

These are initial estimates — subject to literature calibration alongside connection weights.

## Strategy Selection Cascade (D21)

Three-layer cascade with empirical promotion criteria:

```
CAPS settling → regulation attractor?
  ├─ YES (strength ≥ threshold) → pin strategy, send to LLM for narrative
  └─ NO
      └─ RegulationStrategyPredictor → prediction?
          ├─ YES → pin strategy, send to LLM for narrative
          └─ NO → LLM free choice
```

**Threshold**: configurable via `CapsSettlingConfig`, default 0.1 (existing noise floor). Production threshold derived empirically from eval data comparing CAPS attractor strength vs. predictor accuracy vs. LLM agreement (D24).

**Predictor retirement criteria** (D21):
- CAPS-predictor concordance ≥ 80% on eval scenarios
- CAPS-LLM free-choice concordance ≥ 70%
- Degradation monitoring: automatic fallback if concordance drops > 10% below thresholds

The predictor handles safety overrides during calibration: fatigue > 0.7 → force ATTENTIONAL_DEPLOYMENT, intensity > 0.75 → force ATTENTIONAL_DEPLOYMENT.

## Social-Context Input Injection

The 4 social-context input nodes are injected per-tick from `RegulationContext` + MindMap relationship data:

| Node | Injection source |
|------|-----------------|
| `perceived_responsiveness` | MindMap trust edges + affect trajectory for interacting agents. High trust + positive affect trajectory → high activation. |
| `target_distress` | MindMap PAD dimensions for interacting agents. Low pleasure + high arousal → high activation. |
| `social_isolation` | `RegulationContext.interactingAgents()` empty or null → high activation. |
| `power_asymmetry` | MindMap relationship edges (dominance/submission) + social comparison data from `CognitiveProfile`. |

Injection happens in `EmotionRegulationTickParticipant` before calling `CapsEngine.settle()`. Social-context activations are merged with situation classification activations into the `inputActivations` map.

## Integration with Existing Pipeline

The regulation CAPS integration touches `EmotionRegulationTickParticipant`, which orchestrates the cascade:

1. **AppraisalTickParticipant** produces `AppraisalResult` with emotions (DERIVED phase)
2. **EmotionRegulationTickParticipant** (DERIVED phase):
   a. Computes regulation effort (Bosse et al. 2010)
   b. If effort above noise floor:
      - Assembles social-context input activations from RegulationContext + MindMap
      - Merges with existing situation classification activations
      - Calls `CapsEngine.settle()` with merged input activations
      - Extracts regulation attractors from `SettlingResult`
      - If regulation attractor above threshold → use as pinned strategy
      - Else → fall through to `RegulationStrategyPredictor`
      - Else → LLM free choice
   c. Calls `EmotionRegulationGenerationPhase.generate()` with pinned or free strategy
   d. Records fatigue, monitors for failure, persists to MindMap

### Goal Namespace Isolation (D3, D27)

`GoalProposalOrchestrator` uses participant-scoped namespaces with replace semantics per namespace. Each participant (ProactiveRegulationParticipant, EmotionRegulationGoalParticipant) replaces only its own namespace — no cross-participant clobbering. `currentProposals()` merges all namespaces. FOUNDATION phase ordering is undefined; participants are namespace-independent.

## Settling Dynamics

Regulation and behavioral outputs settle together in one pass (D26). This is faithful to CAPS theory — Mischel & Shoda (1995) explicitly include self-regulatory processes as interacting CAUs.

**No interference between output categories**: verified from `CapsSettler.java` — output nodes are terminal in the graph (incoming edges only, no outgoing). Each output's activation is computed independently from its incoming weights and source activations. No normalization, no softmax, no cross-output competition. Adding regulation output nodes cannot change behavioral attractor strengths or convergence behavior.

**Projected topology size**: ~91 nodes (77 baseline + 10 regulation outputs + 4 social inputs), ~140+ connections. Per-iteration cost increases proportionally (~18% more nodes) but iteration count is unchanged — output nodes don't affect convergence dynamics.

## Avoidance Temporal Decay (D5, revised)

Proactive avoidance weight uses exponential decay (Ebbinghaus 1885, Wixted & Ebbesen 1991):

```
avoidanceWeight = initialWeight * Math.exp(-elapsedTicks / decayTimeConstant)
```

`decayTimeConstant` default ~30 ticks. Decay profile: 37% at 30 ticks, 14% at 60 ticks, 5% at 90 ticks. Smooth asymptotic approach — no cliff, unlike the prior linear decay model.

## Eval Strategy

Run the existing eval suite with CAPS regulation settling alongside the predictor:

1. **Paired comparison**: for each eval scenario, record CAPS regulation attractor (strategy + strength), predictor strategy (+ confidence), and LLM free-choice strategy
2. **Threshold calibration**: derive minimum CAPS strength at which CAPS matches predictor/LLM at acceptable rates
3. **Weight calibration**: adjust connection weights where CAPS diverges from predictor, using the D25 three-way resolution protocol
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
