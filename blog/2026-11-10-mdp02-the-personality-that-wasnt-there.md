---
layout: post
title: "The Personality That Wasn't There"
date: 2026-11-10
entry_type: article
subtype: diary
projects: [casehubio/neocortex]
tags: [emotion-regulation, cognitive-architecture, CAPS, personality, disposition-mapping, calibration, connectionism, eval]
series: issue-851-dynamic-regulatory-capacity
---

# The Personality That Wasn't There

The previous entry described auditing the regulation topology — connection weights that were individually correct but collectively wrong. That was a network-level problem. This one operates a layer below: the personality initialization that sets each agent's starting position in the attractor landscape before the network even begins to settle.

Every connection weight in the CAPS regulation topology has a citation. But a connection weight only fires when its source node is active. If the personality initialization doesn't set the right mediating nodes to the right resting activations, the network settles on a landscape that doesn't reflect the agent's personality at all.

## The Missing Personality

The regulation comparison eval has five scenarios, each designed for a different regulation strategy. An accommodating agent facing daily unfairness should reappraise — cognitive change. A competitive agent whose chore rota was rigged should modify the situation. An avoidant agent about to relive embarrassment should avoid the room entirely.

The accommodating agent wasn't reappraising. CAPS settled to social support seeking every time.

The `DispositionWeightMapper` maps personality axes to CAPS initialization — connection weight scaling and resting activations. It had mappings for avoidant, analytical, cooperative, and competitive conflict modes. It had nothing for accommodating. The if-else chain fell through silently. No warning. No log. The agent entered the CAPS network with zero personality bias toward any regulation strategy. Its regulation output was driven entirely by the bold riskAppetite axis (situation modification +0.30) and the cooperative social orientation (social support seeking +0.25). The accommodating disposition — the very thing that defines this agent's regulatory preference — contributed nothing.

The fix was straightforward: add resting activations for cognitive change (+0.25) and social sharing (+0.15) under the accommodating case. But the first run after the fix showed no change. The eval output was identical. The accommodating agent still settled to social support seeking with no cognitive change in sight.

## The Jar That Wasn't Rebuilt

The fix was in `caps-engine`. The eval test lives in `caps-llm-classifier`. Maven resolves `caps-engine` as a dependency from the local repository. The jar in `~/.m2` was timestamped 02:03. The commit adding the accommodating case was timestamped 02:16. The test compiled and ran against the old jar — the one without the fix.

This is the kind of failure that wastes hours if you don't check. After rebuilding `caps-engine` and reinstalling, the accommodating agent's cognitive change appeared as the runner-up at 0.711 — below social support seeking at 0.806, but present.

## Why Resting Activations Don't Do What You Think

Resting activation in CAPS is not a persistent bias during settling. It is two things: an initial value that gets overwritten on the first iteration, and a post-settling bonus added to the final attractor strength.

During settling, every non-input node's activation is recalculated each iteration from its incoming connections. A mediating node with resting activation 0.3 starts at 0.3, is used as a source value for one iteration, then drops to whatever its incoming connections support. If those connections carry zero input — because no relevant situational features are active — the node settles to zero regardless of its resting value.

For output nodes, the resting activation matters only in the attractor extraction step:

```
strength = min(1.0, settled_activation + resting_activation)
```

The resting activation is a constant bonus applied after the network has finished settling. It shifts the attractor ranking without changing the network dynamics.

This means resting activations for output nodes operate as a personality-level thumb on the scale — not a force within the network, but a bias in how results are read. An accommodating agent's cognitive change resting of 0.25 doesn't make the network *want* to settle toward reappraisal. It makes reappraisal rank 0.25 points higher than the network dynamics alone would produce.

## The Dominance Gate

The CAPS cascade has three layers. The safety gate fires first for extreme intensity. CAPS settling fires second. The predictor and LLM fire third. The critical design element is the dominance gate between layers two and three.

![Regulation cascade — three-layer resolution with dominance gate](svg/2026-11-10-cascade-dominance-gate.svg)

CAPS doesn't need to select the right strategy. It needs to either select the right strategy *with confidence* or admit it doesn't know. The dominance gate enforces this: the top attractor must exceed the runner-up by at least 0.15 (the dominance margin) to pin a strategy. If the margin is thinner, CAPS does not pin, and the cascade falls through to the predictor.

For the accommodating agent, social support seeking settles at 0.806 and cognitive change at 0.711. The margin is 0.095 — below 0.15. CAPS doesn't pin social support seeking. The predictor checks the agent's reappraisal capacity (0.80), recognises the mild emotion (anger at 0.18), and selects cognitive change.

The cascade works not because CAPS is right — CAPS thinks this agent should seek support — but because CAPS isn't confident enough to override the predictor. The accommodating agent's personality resolves at the predictor layer, not the CAPS layer. The resting activation's job is to keep the margin narrow enough that CAPS can't dominate.

## The String That Broke Three Agents

With the accommodating fix working, I traced the full CAPS execution to understand why the network dynamics pushed support seeking so far ahead of cognitive change. The trace revealed something else: BIS activation was driving the entire regulation landscape, but without any personality amplification.

The CAPS topology defines disposition modifiers — connection weight multipliers that scale groups of connections based on personality. Conservative risk appetite multiplies all BIS-tagged connections by 1.3 and all BAS-tagged connections by 0.7. This makes cautious agents more anxiety-driven and less approach-motivated. It's a significant personality effect: 14 connections are tagged "bis" and 11 are tagged "bas".

Three of the five eval agents had risk appetite set to "cautious". The topology modifier was keyed on "conservative". The string comparison failed silently. No BIS amplification. No BAS reduction. No warning. Three agents were running with default connection weights where they should have had personality-scaled ones.

This is the same class of bug as the missing accommodating case, but worse. The accommodating case was a missing branch in an if-else chain — obviously absent if you looked. The cautious/conservative mismatch was invisible. The agents initialised. The network settled. The attractors extracted. Everything produced numbers. The numbers were wrong because the personality modifiers never fired, but nothing in the output flagged this.

## Calibrating on the Wrong Foundation

The regulation eval had been calibrated to 5/5 before I found the terminology mismatch. The resting activations I'd added — competitive situation modification at 0.10, strict response modulation at 0.10 — were tuned against the wrong BIS/BAS dynamics. They worked, but only because the amplification they should have been competing against wasn't present.

Fixing "cautious" to "conservative" in the eval tests activated the correct personality modifiers. BIS connections went to 1.3x across the board for three agents. The regulation landscape shifted. Situation selection and attentional deployment — both BIS-driven — surged. Response modulation gained activation from BIS but lost its dominance margin. The eval dropped from 5/5 to fragile results where two scenarios depended on non-deterministic LLM fallback.

The rebalancing required understanding what the personality modifiers actually do to the attractor landscape. Competitive agents don't avoid situations — they confront. Adding negative situation selection resting (-0.15) for competitive conflict mode counteracted the conservative risk appetite's avoidance boost. Strict rule followers suppress impulses to maintain compliance — increasing the strict response modulation resting from 0.10 to 0.15 gave the suppression strategy enough margin to dominate under amplified BIS dynamics.

![Regulation resting activations by disposition axis](svg/2026-11-10-disposition-resting-map.svg)

After rebalancing on the correct foundation, the eval returned to 5/5 with three CAPS-pinned scenarios (margins 0.166, 0.166, 0.168), one safety gate, and one predictor-resolved. All deterministic.

## The Protective Case

With the disposition mapping complete for the five original conflict modes, one remained: protective. The fear/urgency scenario uses a protective agent — cooperative, bold, and oriented toward caregiving. CAPS settled to situation modification (the bold risk appetite's contribution) with no personality-level expression of protectiveness.

Protective agents orient toward two regulatory behaviours: co-regulation (caregiving — George and Solomon 1999, the attachment caregiving system activates when another agent is in distress) and situation modification (active intervention — the protective instinct to change the dangerous situation rather than manage one's own reaction to it). Adding co-regulation resting at 0.25 and situation modification resting at 0.15 gave the protective agent a CAPS landscape that reads correctly: situation modification at 1.000 (ceiling — bold resting plus protective resting) with co-regulation at 0.904 as runner-up. The safety gate still overrides to attentional deployment for the extreme distress intensity. Personality captured. Resource limitation handled.

## What This Means for Cognitive Agent Design

The CAPS regulation calibration exposed a pattern that applies beyond this specific system.

**Personality initialisation and network dynamics are separate concerns.** You can have a perfectly grounded network — every connection weight sourced from published meta-analyses — and still get the wrong output because the personality-specific starting conditions are missing or misconfigured. The network encodes universal psychological mechanisms. The initialisation encodes individual differences. Both must be right.

**Silent string mismatches are the worst kind of bug in data-driven systems.** The topology is YAML. The disposition axes are Java strings. The connection between them is a string equality check with no compile-time safety and no runtime warning on miss. Three agents lost their personality modifiers because one system used "cautious" and another used "conservative". Typed enums would have caught this at compile time. Failing that, a warning log on unrecognised axis values would have caught it at runtime. Neither existed.

**Calibrating against broken foundations is worse than not calibrating.** The resting activations I tuned before finding the terminology mismatch produced the right eval output for the wrong reasons. They compensated for absent personality modifiers without knowing those modifiers were absent. When I fixed the foundation, the calibration was meaningless — the numbers had to be rederived from first principles against the correct dynamics.

The eval is now 5/5 on the correct foundation. Every disposition axis value that appears in the test scenarios has a regulation resting mapping. Every connection weight modifier fires for the personality profile it was designed for. The cascade resolves each scenario deterministically — no scenario depends on LLM non-determinism. The next step is the dynamic regulatory capacity epic: experience-modulated defaults that degrade when strategies fail, so agents learn from emotional failure over simulated months and years.

 t
