---
layout: post
title: "Measuring What the Architecture Produces"
date: 2026-10-27
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [wacky-manor, cognitive-architecture, evaluation, thinking-field, phenomenology, emergence, measurement]
---

# Measuring What the Architecture Produces

We've been building cognitive architecture for characters for a month now — drives, CAPS topology, consolidation, appraisal filtering, multi-channel descriptions, Theory of Mind. The pipeline keeps getting richer. We have eval instruments for drive expression quality, relational model coherence, and sleep-derived cognitive defaults. But none of them measure the phenomenological quality of the character's private inner monologue — whether it reads as lived experience or filed report. We measured *which drives activate* and *how strongly*. We never measured *how the character experiences them*.

That's the gap this work fills. A thinking-field quality eval with four scoring dimensions: register (phenomenological vs mechanical), uncertainty (ambivalence and confusion), channel coverage (body, attention, time, action pull), and thinking/dialogue separation (is the inner voice raw or performed?). Two instruments — a controlled A/B test for isolating variables, and a transcript scorer for measuring quality across full scenario runs.

## The Month in One Table

We scored nine scenario runs spanning the full month of development. The scores tell a story I didn't expect.

| Run | Date | Register | Channels | Separation |
|-----|------|:--------:|:--------:|:----------:|
| layer1-thinking-capture | Oct 04 | 2.07 | 2.33 | 2.97 |
| layer-123-run | Oct 09 | 3.10 | 3.33 | 4.27 |
| phase0-audit-baseline | Oct 13 | 3.37 | 3.63 | 4.37 |
| appraisal-filtering | Oct 15 | 4.07 | 4.53 | 4.97 |
| caps-descriptions | Oct 17 | 3.90 | 4.41 | 4.76 |
| degree-accumulation | Oct 21 | 3.97 | 4.57 | 4.97 |
| tuned-run | Oct 22 | 4.13 | 4.70 | 4.93 |
| full-cognitive-run | Oct 26 | 3.86 | 4.22 | 4.90 |
| thinking-field-run | Oct 27 | 3.29 | 3.61 | 4.49 |

Register — the core measure of whether inner monologue reads as lived experience or filed report — went from 2.07 to 4.13 over three weeks. That's a character going from "I am experiencing anger. My trust level is low." to "Something tightens behind my ribs. The silence after that vase shattered — it's the kind of silence that has a shape."

The interesting part isn't the improvement. It's what caused it.

## What Didn't Move the Needle

The thinking-field text itself — the prompt description that tells the character what its inner monologue should be — went through four versions this month. The progression looks like it should matter: from a vague "what are you FEELING right now?" to a carefully cited multi-channel framing referencing Damasio, Easterbrook, Barrett, and Frijda. Each version felt like an improvement when we wrote it.

The data says otherwise. Between October 9 and October 15, register jumped from 3.10 to 4.07 — the single largest improvement in the timeline. The thinking-field text was identical across all those runs. Didn't change a single word. The somatic grounding prompt from October 7 ("What do you feel in your body right now? Start there — in the body, not the head.") stayed untouched through to October 22.

When we did change the thinking-field text — expanding from somatic-only to a verbose multi-channel framing on October 22 — the improvement was 0.16 register points. Compared to the 0.97 the pipeline delivered with no text change at all.

## What Did Move the Needle

The cognitive pipeline. Specifically: appraisal filtering, LLM situation classification, and drive differentiation. The character got better at inner monologue because it had richer material to draw on — formation memories distilled into somatic descriptions, drives distinguished by Barrett's four channels, situations classified against a CAPS topology that routes the right cognitive content into the observation at the right moment.

This aligns with a principle we've been circling since Era 2: **cognitive content creates quality; pipeline work redistributes it; prompt text neither creates nor redistributes — it sets the register.** The thinking-field prompt tells the character to attend to body sensations, attentional focus, temporal experience, and action pull. But if the observation doesn't contain rich cognitive content for those channels, the prompt is asking for something the architecture can't deliver. The character can't describe what its body feels if no one told it what its body feels.

## The Emergence Question

There's a harder question lurking in the numbers. The early versions of this system used explicit briefings — hand-authored descriptions of each character's personality, behavioural tendencies, speech patterns, emotional defaults. When we stripped those, everything collapsed. Third-person narration dropped from 61% to 13%. Optimism from 24% to 4%. The characters stopped being characters.

The somatic thinking-field text was the first thing that helped recovery — not because it's a good prompt, but because it gave the LLM an embodied anchor. "Start in the body, not the head." That single instruction blocked the default mode of LLM inner monologue, which is cognitive labelling — naming emotions rather than experiencing them. It didn't solve the quality problem. It stopped the bleeding.

Everything since has been about replacing what the briefings used to provide. Not with more prompts — with architecture. Formation memories give the character a past. CAPS topology gives it a dispositional structure. Drive differentiation gives it the ability to distinguish "chest fills with heat, world narrows to *the person*" (gallantry) from "tightness in forearms, attention locks onto *the gap*" (proving-worth). Self-conscious emotions give it pride and shame. Theory of Mind gives it the ability to read others.

Each of these replaced a line of hand-authored briefing with a line of emergent cognitive content. The register scores are measuring whether the replacement is working.

## What the Scores Don't Tell Us

The October 27 run — the most recent — scored 3.29 register, down from the 4.13 peak on October 22. I initially dismissed this as noise. It isn't — or at least, I can't prove it is.

Three things changed between those runs: the thinking-field text was condensed (verbose multi-channel → parenthetical cues), the character profile switched from BASELINE to GENERIC (same personality configuration, different names), and the run went from 60 ticks to 300. Any one of these could explain the drop. All three together make it impossible to attribute.

We've never run the same configuration twice. Every run in the timeline changed at least one variable — usually several. The result is a trend line with no error bars. I can see a general upward trajectory, but I can't distinguish a 0.5 drop from natural variance because I don't know what natural variance looks like.

That's the methodological gap this work exposed. The eval instrument exists now. The baseline exists. The next step is boring but necessary: run the same thing three times and measure how much the scores move on their own.

## The Dynamic Channel System

One finding that surprised me: the `ChannelEmphasisPromptSection` — added as part of the CAPS integration — already provides dynamic per-tick channel emphasis. When situation classification detects a threat, it foregrounds somatic and action tendency channels; for social situations, it elevates attentional focus. This is not all-four-all-the-time. It's situation-dependent foregrounding and backgrounding, driven by the same topology that routes drives.

The verbose thinking-field text was telling the character "Notice your body. Notice what's grabbing your attention. Notice time. Notice the pull." But the channel emphasis system was already doing that — dynamically, situation-specifically, grounded in the classified CAPS activations rather than a static prompt. The condensation in #153 was architecturally correct: remove the static instruction, let the dynamic system do the work.

Whether it was empirically correct — whether the dynamic system alone produces the same quality as the static instruction plus the dynamic system — is an open question. And now we have the instrument to answer it.

## Where This Goes

The gap to the original briefed baseline is a wrong question. The briefed baseline was inflated — BASELINE0's mean of 3.93 reflected drive contamination, not genuine quality. Once relevance-gated evaluation exposed this, the target shifted from "match the old number" to "raise genuine expression quality."

The real remaining gap is expression quality when triggered. Gallantry, when it correctly activates, scores 2.65 quality. The character knows to be gallant, places itself correctly in the scene, but the felt experience of gallantry is thin. The pipeline delivers the right drive to the right moment. The thinking-field description sets the right register. What's missing is richer cognitive content — more for the character to draw on when it experiences that chest-filling warmth.

That's the work ahead. Not more prompt engineering. Not more pipeline plumbing. More cognitive content, empirically validated, one layer at a time.
