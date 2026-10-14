---
layout: post
title: "When Drives Compete"
date: 2026-10-14
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, CAPS, drive-competition, metacognition, emotion-differentiation, formation-memories, experience-behaviour-catalogue]
series: issue-115-drive-expression-gap
---

*Drive Expression Gap — Part 3 of 3 ([Part 1](2026-10-13-mdp01-when-your-characters-stop.md) · [Part 2](2026-10-13-mdp02-from-authoring-to-architecture.md))*

# When Drives Compete

In Part 2, we designed the three-layer fix for the drive expression gap: weighted situation vocabulary (Layer A), memory-to-catalogue calibration (Layer B), and drive→CAPS activation injection (Layer C). The derivation chain had seven breaks. The design addressed all of them. This entry covers what happened when we built it — including a discovery that none of the design work anticipated.

## The Three Layers

Layer A was already done: 31 CAPS INPUT nodes got literature-derived weighted vocabulary, 17 disconnected nodes got wired to MEDIATING nodes. The CAPS network could now classify situations by content, not just by personality defaults.

Layer B connected formation memories to the experience-behaviour catalogue. Each memory got a `catalogue` reference — not a generic "formation-memory" placeholder, but a specific psychological model: `attachment-dismissive` for a memory of transactional care, `cbt-negative-self` for achievement that nobody noticed, `bandura-mastery-efficacy` for a first successful deception.

The mapping required a methodology. The key rule: always prefer catalogue entries that calibrate MEDIATING nodes (`self_worth`, `self_efficacy`, `other_reliability`) over entries that only move OUTPUT nodes (`approach`, `persist`, `withdraw`). Many Bandura and operant entries sound right for a memory but only adjust behavioural outputs — they can't fix a break in the mediating layer, which is exactly what Layer B exists to do.

Twenty-six memories across five characters. Each mapping traced against three tests: does the catalogue entry push the right mediating nodes in the right direction? Does the cumulative trajectory produce the extreme substrate the character needs? Does any outcome contradict the character's profile in ways the CAPS network can't resolve?

Layer C replaced text-based drive classification with direct injection. Instead of hoping a drive's somatic description contains vocabulary terms the situation classifier can match, each drive type maps to specific CAPS nodes at the drive's intensity. Fourteen mappings, all literature-sourced.

The derivation chain is now complete. Formation memories → catalogue entries calibrate mediating nodes → situations fire INPUT nodes via vocabulary → drives inject persistent CAPS activations → the network settles → behavioural attractors emerge → the LLM receives a felt-experience prompt. Seven breaks, all fixed.

## The Substrate Section

Layer B also added a "What Shaped You" section to the character's rendered prompt — a paragraph of felt-experience text describing the psychological core that the formation memories produced. Not behavioural prescription ("you gloat when rivals fail") but mediating substrate ("Your achievements exist but nobody sees them. The gap between what you can do and what anyone notices creates a restless hunger").

This matters because the LLM receives formation memories as narrative ("Age 8: You won the spelling bee. Nobody asked about it.") and drives as somatic descriptions ("There's a warmth that rises when courtesy matters"). The substrate bridges them — it's the "so what" that connects developmental history to current motivational state.

The micro test results held or improved. Cross-character differentiation was strong: the same event (a rival falls publicly) produced gloating from a character with low `self_worth` and high `competence_recognition` sensitivity, concern from a character with high `other_safety`, and cold satisfaction from a character with deep `distrust`. Same situation, different substrate, different emergent behaviour. That's Mischel's if-then profile operating through the CAPS network.

## Where It Broke

PP gallantry.

PP has three drives: gallantry (0.9), protection (0.8), proving-worth (0.7). Gallantry is about the shape of the gesture — the theatrical courtesy, the moment landing with grace. Proving-worth is about competence — fixing the thing, demonstrating capability. When a stuck door presents both a problem to fix AND a person to help with style, the LLM defaulted to the concrete drive every time. The proving-worth score was always high. The gallantry score sat at 2-3 out of 5.

We tuned the substrate. Four iterations. Version 1 (proving-worth only): gallantry scored 2. Version 2 (gallantry added): recovered to 3. Version 3 (gallantry emphasised as "deeper"): regressed to 2. The text wasn't the problem. The signal imbalance was.

PP has five formation memories. Four are proving-worth. One is gallantry — shared with proving-worth. The substrate, no matter how carefully worded, is one paragraph competing against a wall of competence-demonstration memories.

## The Discovery

The question that unstuck it: "Is there something meta we can do — not more data, but guidance on HOW to process competing signals?"

Two drives compete when they activate overlapping CAPS nodes. Gallantry maps to `other_safety`, `acceptance`, `approach`. Protection maps to `other_safety`, `physical_threat`, `approach`. They share `other_safety` and `approach` — shared activation pathways that produce similar felt experience. The LLM can't distinguish which drive is active because the neural substrate is the same.

This is detectable. The D1 drive→CAPS mappings tell you which nodes each drive activates. If any pair shares nodes, they potentially confuse the LLM. A universal meta-instruction — "Each of these pulls is a distinct felt state. When more than one activates, attend to which quality of feeling is at the front" — teaches the LLM to differentiate between competing drives without prescribing which one to pick.

PP gallantry went from 3/5 to 5/5. The LLM explicitly caught the proving-worth impulse and separated it: "Not the problem-solving tightness — that comes second. First there's just... her." In the differentiation test (a dinner scene with nothing to fix): "Not because she needs her chair pulled out. Because the moment deserves it."

The meta-instruction is two sentences. It names no drives, no characters, no scenarios. It works for any combination of competing drives in any world. It lives in the cognitive preamble — the system-prompt-level meta-reasoning that tells the LLM how to process its own internal states.

## What This Maps To

This is metacognitive monitoring. The literature on it is specific and relevant.

Dimaggio and Lysaker (2010) describe metacognition as the ability to distinguish between different internal states. Deficits in this ability cause state confusion — you can't tell anxiety from excitement, you can't tell gallantry from proving-worth. The meta-instruction is training exactly this: attend to the qualitative difference between felt states.

Semerari et al. (2003) built a Metacognition Assessment Scale with a "differentiation" subscale — measuring precisely the ability to distinguish between mental states. Barrett (2017) and Kashdan et al. (2015) extended this as "emotion differentiation" or "emotional granularity" — the capacity to make fine-grained distinctions between similar-seeming emotional states. People with higher granularity have better self-regulation. Characters with higher granularity should have more distinctive drive expression.

In CAPS terms: the network settles competing activations through attractor dynamics. But the LLM simulating the character doesn't have the settler — it has a text prompt. The meta-instruction does at the prompt level what the CAPS settler does at the network level: resolve ambiguity between competing activations by attending to qualitative differences.

## What's Not Right Yet

The detection currently uses a static mapping table — fourteen drive types hard-coded to CAPS nodes. A new world with different drives has no mappings and no detection. The meta-instruction is generic ("warmth, tension, or urgency") rather than grounded in the character's actual developmental history.

The architecture that would fix both: during consolidation (the cognitive sleep cycle), classify each drive's description through the same vocabulary classifier that processes events. Discover which CAPS nodes each drive activates — by inference, not declaration. If overlap is detected, trace the CAPS activation pathway through the mediating nodes, find which formation memories calibrated those nodes, and derive what each drive "feels like" for THIS specific character — grounded in their own developmental trajectory, not in generic adjectives.

The vocabulary infrastructure exists. The catalogue-to-mediating-node wiring exists. The formation memories are annotated. The consolidation pipeline already processes memories and builds attractors. The missing piece is the analysis step that connects drive descriptions → CAPS overlap → memory-grounded differentiation text. That's the next session's work.
