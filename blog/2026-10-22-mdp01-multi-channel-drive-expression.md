---
layout: post
title: "Why Somatic Markers Aren't Enough"
date: 2026-10-22
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [drive-expression, multi-channel, affective-neuroscience, somatic-markers, cognitive-architecture]
series: issue-122-drive-expression-pipeline
---

# Why Somatic Markers Aren't Enough

The thinking prompt that guides every character's inner voice said: "What do you feel in your body right now? A tightness, a warmth, a prickle, a numbness? Start there — in the body, not the head."

That prompt made somatic sensation the only channel for drive expression. Scheming felt like tension. Loyalty felt like warmth. So did care. So did lust. When I looked at the discrimination eval scores — 3.97 with multi-channel generation, 2.16 with somatic-only — the reason was obvious. Body sensations don't carry enough information to distinguish seventeen drives.

## What the literature actually says

I'd been leaning on Damasio's somatic marker hypothesis as the theoretical anchor for this prompt. Somatic markers matter — they're the body's signal that biases decision-making toward learned outcomes. But Dunn, Dalgleish, and Lawrence (2006) ran a critical evaluation and found something inconvenient: patients with impaired decision-making showed no somatic marker deficits, and patients with somatic marker deficits sometimes decided just fine. Somatic markers are neither necessary nor sufficient on their own.

This isn't a rejection of Damasio — it's a scope correction. The body IS a signal. It's just not the only one.

Scherer's Component Process Model (2001) provides the organising framework. He defines five synchronised subsystems during an emotion episode: cognitive appraisal, neurophysiological support (the body), motivational executive (action tendencies), motor expression (face, voice, gesture), and subjective feeling. Our characters don't have faces or voices in the motor-expression sense — they express through dialogue and action — so four of Scherer's five subsystems map cleanly.

But the specific channels come from more targeted sources:

**Attentional narrowing** — Easterbrook (1959) showed that emotional arousal narrows the attentional field to the most relevant cues. Different drives narrow attention differently. Scheming scans for leverage. Protection scans for threats. Curiosity scans for novelty. This is inherently discriminating in a way that "tightness in the chest" is not.

**Temporal distortion** — Barrett's theory of constructed emotion (2017) treats time perception as part of the brain's predictive model during emotion. Fear compresses time into urgency. Care stretches it into patience. Play suspends it. When I think about what makes a scheming character feel different from a protective one, temporal experience is the channel that carries it. Both might feel tension in the body. But scheming stretches time — everything slows while the plan assembles — and protection compresses it into right now.

**Action tendency** — Frijda (1986) made the strongest claim: emotions ARE action tendencies. Not a consequence. Constitutive. The pull toward shielding IS protection. The pull toward manoeuvring IS scheming. This turned out to be the most discriminating channel of all, because action tendencies are the one dimension where every drive produces a unique signature.

## What changed

The neocortex generation pipeline had already moved to multi-channel expression — somatic, attentional, temporal, and action tendency. But the thinking prompt in the runtime agent loop still funnelled everything through the body. The generation side was producing four-channel descriptions, and then the thinking prompt was telling the character to ignore three of them.

The fix was a single prompt change. Instead of "Start there — in the body, not the head," the prompt now walks the character through all four channels: notice the body, notice what's grabbing your attention, notice how time feels, notice what you're pulled toward doing.

I verified that all seventeen drive entries across the five characters already have serving formation memories — every drive has at least one experiential episode anchoring it. The pipeline is complete end to end.

## The principle this establishes

Somatic markers are necessary but not sufficient for drive discrimination. Our own eval data confirms the same result Dunn et al. found in the clinical literature: a single channel cannot carry the information needed to distinguish between drives that share similar body signatures. The four-channel model — somatic, attentional, temporal, action tendency — provides the discriminating dimensions.

The next question is whether expression should be uniform across channels or situation-dependent. Easterbrook's work suggests it should vary — attentional narrowing is stronger under high arousal, weaker under calm. A vocabulary classifier that weights channels by situation context would be more faithful to the literature than a fixed four-channel prompt. That's the neocortex side of this epic.
