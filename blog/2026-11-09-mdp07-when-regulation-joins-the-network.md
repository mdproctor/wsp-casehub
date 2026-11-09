---
layout: post
title: "When Regulation Joins the Network"
date: 2026-11-09
entry_type: article
subtype: essay
projects: [casehubio/neocortex]
tags: [emotion-regulation, cognitive-architecture, CAPS, personality, connectionism, social-regulation, cognitive-agents]
series: issue-851-dynamic-regulatory-capacity
---

# When Regulation Joins the Network

The last post described the problem: a regulation predictor that maps personality to strategy works for single episodes but gets stuck under compounding pressure. The fix was experience-modulated capacity — feedback that degrades effective capacity when strategies fail and recovers it when they're not needed.

That fix solves the stuck-agent problem. But it's still a scalar overlay on a deterministic if/else cascade. The predictor checks personality axes, checks the emotion category, and picks the strategy with the dominant capacity score. The modulation adjusts the score. The architecture is the same: a lookup table with softer edges.

CAPS — Mischel and Shoda's cognitive-affective processing system — is not a lookup table. It is a connectionist network where the same internal state that drives behavioural tendencies also drives regulatory responses. The network settles. What it settles toward depends on how the weights interact across dozens of connections, modulated by the personality of the specific agent running on it. What I am building now is the integration of regulation into that network.

## 1. The Same Internal State Drives Both Responses

The existing CAPS topology has 77 nodes across three layers. Input nodes represent situations — rejection, physical threat, mastery, dominance, reward. Mediating nodes represent internal models — self-worth, self-efficacy, BIS and BAS activation, threat sensitivity. Output nodes represent behavioural tendencies — approach, withdraw, fight, flight, freeze, persist, rebel.

When an agent encounters a situation, the situation classifier activates the relevant input nodes. Those activations flow through the mediating layer, modulated by personality-derived connection weights. The network iterates until it converges on a stable pattern of output activations — the behavioural attractors.

The observation behind the CAPS regulation design is that the same mediating nodes that drive behavioural outputs also contain the information needed for regulation strategy selection. High BIS activation plus high arousal pushes both the "withdraw" behavioural output AND the decision to deploy attention away from the threat. High self-efficacy pushes both "persist" AND the decision to reappraise the situation. The personality state does not select behaviour and regulation separately. It does both at once.

![CAPS unified topology with regulation and social-context nodes](svg/2026-11-09-caps-regulation-topology.svg)

Ten regulation strategies become output nodes in the same network: five from Gross's 1998 process model (situation selection, situation modification, attentional deployment, cognitive change, response modulation) and five from Zaki and Williams's 2013 interpersonal taxonomy (social support seeking, social sharing, social withdrawal, co-regulation, deliberate provocation). They settle alongside the behavioural outputs in a single pass. No separate regulation system. No second network. One settling step produces a personality-coherent pattern of what to do AND how to regulate.

## 2. Social Context Needs Its Own Inputs

Intrapersonal strategies — reappraisal, suppression, attention redirection — can be driven by existing mediating nodes. BIS activation pushes suppression. Self-efficacy pushes cognitive change. The connections map naturally from the meta-analysis data.

Social strategies require something the topology does not have: knowledge of who is present and what state they are in.

An agent deciding between social support seeking and social withdrawal needs to know whether a trusted partner is available. One deciding whether to co-regulate needs to know whether another agent is in distress. The CAPS network cannot differentiate between five social strategy outputs if every social strategy node receives the same undifferentiated signal.

I spent time searching the literature on this. Reis, Clark, and Holmes (2004) established perceived partner responsiveness as the organising construct for intimacy processes. Ruan and colleagues (2019) showed that perceived responsiveness directly predicts emotional expression and support-seeking behaviour. Main, Yung, Chen, and Zawadzki (2025) confirmed in a 3,158-assessment experience sampling study that closeness to the interaction partner predicts strategy choice — less closeness produces more suppression. Reeck, Ames, and Ochsner (2016) formalised the social regulatory cycle, where the target's emotional state is the primary trigger for extrinsic regulation. Niven, Totterdell, and Holman (2009) used hierarchical cluster analysis on 378 interpersonal regulation strategies and found the primary dimension was affect-improving versus affect-worsening intent.

No published computational model has encoded these social-context dimensions as CAPS-like input activations. CASPER (Camacho, Deshpande, and Perino 2025) provides the closest theoretical decomposition — cognitive control, affective reactivity, social processing, and regulatory processes as interacting subsystems — but no network implementation exists. This is novel territory.

Four social-context input nodes, each grounded in the empirical evidence:

**Perceived responsiveness** — how responsive the present interaction partner is (Reis et al. 2004, Main et al. 2025). Discriminates social support seeking from social withdrawal.

**Target distress** — observable distress in another agent (Reeck et al. 2016). Discriminates co-regulation from other social strategies.

**Social isolation** — absence of social contact (Rimé 2009, Williams 2007). Drives social withdrawal. Computed as a gradient — zero agents produces full activation, three or more agents produces zero.

**Power asymmetry** — relational power advantage (Schilke, Reimann, and Cook 2021; Tedeschi and Felson 1994). Discriminates deliberate provocation via perceived impunity. An agent with a power advantage over the target can provoke with reduced retaliation risk — the power differential IS the enabling condition.

These inputs route through the existing mediating layer. Perceived responsiveness flows through `other_safety` and `other_reliability`. Social isolation flows through `BIS_activation` and `arousal_level`. The mediating layer is where personality-weighted processing happens, so routing social-context signals through it makes social regulation personality-sensitive through the same mechanism as everything else.

## 3. The Three-Layer Cascade

A connectionist network that has just been extended with 14 new nodes and 34 new connections should not be trusted to override a validated predictor on day one. The weights are literature-derived but uncalibrated. The confidence threshold is an initial estimate, not an empirical finding.

The design uses a three-layer cascade: CAPS first, predictor second, LLM third.

![Three-layer regulation strategy cascade](svg/2026-11-09-cascade-selection.svg)

A safety hard gate fires first. If the agent's regulation fatigue exceeds 0.7 or the peak emotional intensity exceeds 0.75, the cascade forces attentional deployment unconditionally — cognitive resources for reappraisal are exhausted, and the network's opinion is irrelevant. This gate was inside the predictor. Now it sits upstream of everything.

If the safety gate passes, CAPS settles. If a single regulation attractor dominates — its strength exceeds the confidence threshold AND it leads the runner-up by a sufficient margin — it pins the strategy. The LLM still generates the regulation narrative, but the strategy is fixed by the network.

If CAPS is ambivalent — multiple regulation attractors above threshold with no clear winner — the cascade falls through to the predictor. The predictor's if/else logic handles the case. If the predictor also falls through (positive emotions, unknown categories), the LLM chooses freely.

**A fair counter:** why keep the predictor at all if CAPS is supposed to subsume its logic? Because CAPS regulation nodes are new and uncalibrated, and the predictor has evaluation data. The predictor's strategy selections agree with an LLM judge 85% of the time across validated personality-emotion combinations. CAPS has to earn that concordance. The retirement criteria are explicit: CAPS concordance with ground truth evaluation scenarios at or above 80%, and CAPS-LLM free-choice concordance at or above 70%. Below those thresholds, the predictor stays.

## 4. The Network Learns from Regulation

The predictor does not learn. Its weights are computed once from personality axes and applied every time. The experience-modulation system I described in the previous post adds a feedback loop, but it operates on scalar capacity values, not on the connection weights that drive the settling dynamics.

CAPS learns through Rescorla-Wagner weight updates — the same associative learning mechanism that drives all connections in the existing topology. When a regulation episode produces a result, the outcome maps to weight update parameters. High-effectiveness regulation (the strategy worked — intensity dropped substantially) produces a positive outcome signal that strengthens the connections to the strategy that was selected. Low-effectiveness regulation weakens them.

The learning is not strategy-specific in isolation. It updates the connections from the mediating nodes to the regulation output nodes. An avoidant agent who tries situation selection for shame and fails does not just lose confidence in situation selection — the connections from `BIS_activation` and `threat_sensitivity` to `situation_selection` weaken. The next time those mediating nodes activate strongly, the settling dynamics shift toward cognitive change or attentional deployment — whichever alternative pathway has stronger connections for this agent.

This is different from the scalar experience modulation. That system degrades a single capacity number. This system changes the shape of the network. An agent that fails at avoidance does not just have less avoidance capacity — it has a network that has been sculpted by experience to settle differently when the same internal state recurs. The personality baseline is preserved in the initial resting activations. What changes is the weight landscape that the settling process navigates.

## 5. Why This Matters for Cognitive Agents

Most AI agents handle emotion regulation — if they handle it at all — as a prompt instruction. "You are neurotic, so suppress your feelings." The strategy is picked by the system prompt and never changes. Under this model, a neurotic agent that fails at suppression for ten episodes will try suppression for the eleventh, because nothing in the architecture learns from outcomes.

The step from prompt-level personality to deterministic prediction was an improvement. The predictor mapped published meta-analysis correlations (Baranczuk 2019) to strategy selections — empirically grounded, personality-coherent, and fast enough that LLM calls became a fallback rather than a default.

The step from prediction to network settling is a different kind of improvement. It is not about accuracy — the predictor may be more accurate than the network for months while calibration continues. It is about the architecture admitting the right kind of complexity. Personality is not a lookup table. It is not five disposition axes mapped to five regulation strategies. It is a network of interacting cognitive-affective units where the same internal states drive multiple outputs simultaneously, where settling dynamics produce patterns that no single if/else branch can express, and where outcomes change the network for next time.

The 10% of variance that personality explains in strategy choice (Baranczuk 2019) is a starting condition. The other 90% — context, situation, mood, relationships, and accumulated experience — enters through input activations, mediating node dynamics, and weight updates. A connectionist architecture gives that 90% somewhere to live. A lookup table does not.

The implementation plan is 9 tasks across 6 batches. Topology extension, disposition modulation, goal namespace isolation, social-context injection, cascade integration, and paired comparison evaluation. The first four batches change no existing behaviour. Batch 5 wires the cascade. Batch 6 collects the data that determines whether CAPS earns its place at the top of the cascade or falls back to the predictor that already works.

I'll write about the calibration results when they exist.
