---
layout: post
title: "Your Body Knows It's a Threat Before You Do"
date: 2026-10-23
entry_type: article
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [drive-expression, channel-weighting, easterbrook, embodied-cognition, cognitive-architecture, affective-neuroscience]
series: issue-122-drive-expression-pipeline
---

# Your Body Knows It's a Threat Before You Do

Continuing from [Why Somatic Markers Aren't Enough](2026-10-22-mdp01-multi-channel-drive-expression.md) — where we established that four expression channels (somatic, attentional, temporal, action tendency) discriminate between drives that body sensations alone cannot.

That entry ended with a question: should the channels be weighted equally, or should the weighting shift with the situation? The literature says shift. We built it, ran it, and the numbers confirmed what Easterbrook showed in 1959.

## The equal-weighting assumption

The multi-channel thinking prompt tells a character: notice your body, notice what grabs your attention, notice time, notice what you're pulled toward. Equal emphasis, every situation. But emotions don't work that way. A character facing a collapsing ceiling beam and a character being welcomed warmly to dinner are not processing through the same channels at the same intensity.

Easterbrook's cue utilisation hypothesis (1959) demonstrated that emotional arousal narrows the perceptual field. Under high arousal — threat, danger, urgency — the organism reduces the range of cues it processes. Attention narrows. Body signals amplify. The pull to act becomes overwhelming. Under low arousal — social warmth, contemplation — attention broadens. More cues are processed simultaneously. The body quiets.

This is not a minor modulation. It is a qualitative shift in how experience is constructed.

## What we built

A `TopologyChannelWeighter` that maps the CAPS situation classification to a four-channel weight profile. Seven category profiles, each grounded in specific literature:

**Threat** — somatic 0.35, attentional 0.10, temporal 0.15, action tendency 0.40. Easterbrook's narrowing in action: attention suppressed, body and behavioral urge foregrounded. Frijda (1986) provides the action tendency basis — fight/flight IS the emotion, not a consequence of it.

**Social** — somatic 0.20, attentional 0.35, temporal 0.15, action tendency 0.30. Baumeister and Leary (1995) on belongingness: social situations demand attention to interpersonal cues. Who is included, who is excluded, what the group dynamics are — this requires broad attentional sampling, the opposite of threat narrowing.

**Autonomy** — somatic 0.25, attentional 0.15, temporal 0.20, action tendency 0.40. Deci and Ryan (2000) self-determination theory: autonomy is about behavioral agency. Powerlessness is strongly embodied — Seligman's (1975) learned helplessness manifests as physical freezing, heaviness, constriction. The action tendency channel carries the signal whether that tendency is toward self-direction or toward collapse.

When multiple categories activate simultaneously — a fight at a dinner party is both threat and social — the weights blend by confidence. The classifier returns activations with confidence scores, and the weighter produces a coherent profile from the mix.

The prompt section that consumes these weights renders differential guidance: "Foreground your somatic: physical sensations — gut reactions, muscle tension, breathing." Or: "Background your attentional: your broad, unfocused awareness." Only channels that deviate significantly from the default appear — a balanced situation produces no emphasis guidance at all.

## What the LLM classifier actually produced

Six scenarios, real LLM classification through the full pipeline. The classifier identifies CAPS input nodes (Bowlby attachment, Gray threat, McClelland achievement, etc.) with confidence scores. The weighter maps those to channel profiles.

The threat scenario — a ceiling beam cracking overhead, someone frozen beneath it — produced `physical_threat=1.00`, `unpredictable_danger=0.70`, `powerlessness=0.80`. The resulting channel profile: somatic 0.32, attentional **0.12**, action tendency **0.40**.

Attentional at 0.12. That is 52% below the default equal weight. The character's perceptual field narrows to the body and the pull to act. Easterbrook's 1959 finding, reproduced computationally from a natural language situation description, sixty-seven years later.

The social scenario — a warm welcome, saved seats, group laughter — produced `acceptance=0.90`, `secure_attachment=0.60`. Channel profile: attentional **0.33**, dominant channel. The attentional field broadens to scan social cues. The inverse of threat narrowing.

The neutral scenario — afternoon sun, ticking clock, cup of tea — produced zero activations. Channel weights: 0.25 across all four. Perfect default. The classifier correctly identified a scene with no psychological salience and the system did nothing. That matters. Injecting channel emphasis into a calm scene would be noise, and the pipeline avoids it.

## Why this matters for cognitive agents

Most LLM agent architectures treat the system prompt as static. The character's personality, goals, memories — all rendered once and held constant regardless of what's happening in the scene. The prompt doesn't know if the character is under attack or at tea.

Situation-dependent channel weighting is one mechanism for dynamic prompt modulation — the cognitive state rendered to the LLM shifts with the classified situation. A character in danger gets prompted toward somatic awareness and action urgency. A character at a gathering gets prompted toward social attention. The same character, the same drives, but the expression channels shift.

This is not a gimmick. It is what embodied cognition theories predict. Barrett's theory of constructed emotion (2017) argues that the brain does not have fixed emotion circuits — it constructs emotional experience from available signals, weighted by context. Scherer's Component Process Model (2001) treats emotion as synchronised subsystems that activate in situation-dependent patterns. The four channels we use map to four of Scherer's five components (we lack motor expression — our characters express through dialogue, not faces).

The practical consequence: a character who has both a protection drive and a scheming drive will express them through different channel profiles when activated. Protection in a threat scenario foregrounds somatic urgency and action readiness. Scheming in a resource scenario foregrounds temporal planning and attentional scanning. The drives already have different descriptions — channel weighting ensures they are experienced differently too.

## What the data says about Frijda

Action tendency was the dominant channel in four of five non-neutral scenarios. Threat (0.40), powerlessness (0.31), competition (0.32), mastery (0.33). Frijda's strongest claim — that emotions ARE action tendencies, constitutive rather than consequential — is reflected in the empirical weighting. The pull to act is consistently the most discriminating channel signal across situation types.

The one exception is social acceptance, where attentional (0.33) edges out action tendency (0.27). This is also consistent — Baumeister and Leary's belongingness need is fundamentally about monitoring the social field, not about acting on it. The action tendency in acceptance is approach, but it is quieter than the attentional scanning that dominates the experience.

## Open questions

The category-to-channel weight profiles are literature-informed initial values. They need empirical tuning against scenario runs — does threat attentional at 0.10 produce qualitatively better drive expression than 0.15 or 0.05? The architecture supports this iteration without code changes; the profiles are data.

The bigger question is whether channel weighting measurably improves drive expression scores in the full 300-event eval. The per-drive discrimination eval (classify_emotions.py) is the instrument. If threat-activated drives score higher on somatic and action tendency language with channel weighting than without, the mechanism is validated at the behavioural level, not just the architectural level. That eval is next.
