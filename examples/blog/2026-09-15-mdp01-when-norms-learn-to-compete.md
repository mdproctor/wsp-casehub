---
title: "When Norms Learn to Compete"
date: 2026-09-15
author: mdp
entry_type: note
subtype: diary
series: wacky-manor
tags: [cognitive-architecture, norm-filtering, social-comparison, llm-eval, casehub]
status: draft
refs: ["#57", "#59", "#60", "#53"]
---

# When Norms Learn to Compete

The original issue said "match norms to nearby characters." Parse the
rule text for character names, filter by who's in the room. It's the
obvious approach, and it's wrong for anything beyond a toy.

"Never help Penelope directly" — the original design would scan a
character roster for the substring "Penelope." That scales to five
characters. It doesn't scale to a thousand. And more fundamentally, it's
hardcoded relevance. The author decides at config time which norms
activate around which characters. Nothing emerges.

I wanted norms to compete for attention the same way beliefs do — scored
by relevance to what the character actually knows, gated by a budget
that adapts to the situation. A character in a calm, empty room sees
fewer norms. A character under pressure with a schemer nearby sees more,
and the contextually relevant ones float to the top.

The mechanism: each norm gets a base score from its priority. When words
from the character's active cognitive subjects (the entities they have
mindmap knowledge about) appear in the norm text, it gets a context
boost of +10 — enough to outrank any non-relevant norm. Then
`CognitiveBudget.maxNorms()` cuts the list. High arousal widens the
budget. Crowded rooms widen it further.

The matching substrate is simple substring comparison against entity
display names — and that's fine, because the architecture doesn't
depend on it. The substrate can swap to embeddings or LLM judgment
without changing how norms integrate into the cognitive pipeline. What
matters is that relevance comes from the character's cognitive context,
not from a static roster.

## The Perception Problem

SocialComparison integration raised a harder question: when the system
computes how differently two characters perceive the same entity, what
goes into the prompt?

Three options surfaced. Raw PAD metrics — "divergence 0.7, pleasure
delta -0.4." Generic prose — "you sense tension with Peter." Or
perception-level translations — "Peter seems to enjoy your interactions
more than you do."

I'd already applied the garden's profile-data principle (data over
directives) from earlier work. "Pleasure: 0.94" works because it's
self-contained — the LLM interprets it without domain knowledge. But
"divergence 0.7" is not self-contained. It leaks the measurement
framework. The LLM would need to understand PAD theory to interpret
the number, and even then, 0.7 on what scale? With what threshold for
significance?

The insight was that "data over directives" has a hidden assumption:
the data must be interpretable without domain knowledge. Absolute values
pass that test. Comparative values don't. Any metric that came from a
formula — distance, delta, cosine similarity, ratio — needs translation
to a form the reader can act on without understanding the formula.

So perception-level statements: the dominant PAD dimension difference
maps to a character-perspective observation. If my pleasure is lower
than theirs, "Peter seems to view this more positively than you do." If
my arousal is higher, "You're more alert around Penelope than she seems
to be." Trajectory alignment appends direction: "and this gap is
widening" or "though you're converging."

These are data, not directives. The character briefing determines what a
schemer does with "Peter enjoys your interactions more than you do" — a
schemer exploits the asymmetry, a peacemaker addresses it. The
translation step doesn't prescribe behaviour; it makes the comparative
signal interpretable.

## What's Left

The SocialComparison pipeline is wired but inert. No entity nodes with
PAD values and perspectival overlays exist in the mindmap yet. The
rendering gracefully returns nothing — which is the correct early-game
state. As interactions accumulate through ConversationBridge and
consolidation, the mindmap fills with entity perceptions, and the Social
Awareness section starts producing output. No switch to flip, no
"enable social awareness" flag. It emerges from data density.

The remaining Phase B work needs a cross-repo slot. The scoring
interfaces in blocks and neocortex operate on different types —
`ConfidenceScorer` takes CBR cases, `GraduationScorer` takes Memory
objects. The heuristics (arousal keywords, feature diversity) are locked
to input types they don't fundamentally need. A `ContentScorer`
interface operating on extracted signals — text, metadata, timestamp —
would let the same scoring logic compose into both the CBR retrieval
pipeline and the consolidation graduation pipeline. The repos are
siblings; the dependency direction is clean. It's a refactoring worth
doing right.
