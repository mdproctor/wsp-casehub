---
layout: post
title: "When Your AI Characters Stop Being Themselves"
date: 2026-10-13
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, personality-emergence, CAPS, formation-memories, evaluation]
series: issue-115-drive-expression-gap
---

# When Your AI Characters Stop Being Themselves

We've been building a cognitive architecture where LLM-driven characters express personality through emergent behaviour — not through prescriptive instructions that tell them how to act. The personality model feeds drives, memories, and facet scores into a computational pipeline; the pipeline enriches the LLM's context; the LLM produces behaviour that feels psychologically authentic. When it works, a character's scheming or gallantry or loyalty emerges from who they are, not from what we told them to do.

When it stops working, you get a character who is technically correct but emotionally flat. And the gap between "technically correct" and "psychologically alive" is where this story starts.

## The Gap

Our evaluation framework classifies every event in a scenario run — each line of dialogue, each aside — scoring how strongly each character drive is expressed on a 1-5 scale. When we replaced prescriptive text (somatic descriptions that told the LLM exactly how each drive feels) with a cognitive pipeline that derives behaviour from personality traits, overall drive expression dropped. Not catastrophically — the mean went from 3.93 to 3.57 — but five specific drives collapsed:

| Character | Drive | Prescriptive | Pipeline | Gap |
|---|---|---|---|---|
| Hooded Claw | gloating | 3.97 | 2.30 | -1.67 |
| Dick Dastardly | gloating | 3.50 | 2.00 | -1.50 |
| Peter Perfect | gallantry | 4.50 | 3.27 | -1.23 |
| Penelope | social-harmony | 3.18 | 2.38 | -0.80 |
| Penelope | adventure | 3.75 | 3.11 | -0.64 |

The obvious response: tune the data. Refine the somatic descriptions, enrich the formation memories, give the pipeline more to work with.

## The Data-Tuning Trap

I refined somatic descriptions for all five drives. Made them more vivid, more body-felt. Added formation memories — a premature-celebration episode for Dick Dastardly, a bow-in-the-rain memory for Peter Perfect, a hidden-cove discovery for Penelope.

The results were mixed. DD's gloating improved by a full point (2.00 → 3.00) — the biggest single-drive improvement we'd seen. But PP's gallantry actually regressed (3.27 → 2.93). The somatic I'd written described actions ("your hand sweeps the door open, a half-bow happens before you think about it") rather than feelings. And the formation memory I'd added — PP carrying shopping bags with a theatrical bow — existed specifically to make the LLM produce theatrical gestures. It was targeted priming, not emergence.

Claude caught this mid-session. The DD gloating somatic said "hands rubbing together, voice rising" — actions the LLM should derive from feeling the drive, not actions we should put in the prompt. If the character feels a fizzing in their fingers from rising anticipation, the LLM figures out that means hand-rubbing. Writing "hands rubbing together" is us doing the LLM's job.

## "That Felt Wrong"

The deeper problem surfaced when I looked at an existing formation memory for DD at age 14. He builds an elaborate trap for a cat. It works perfectly. Nobody sees it. The memory says: "That felt wrong — like a firework going off in an empty field."

But DD has scheming at 0.85 intensity. A successful plan should feel good. The "felt wrong" is the author pre-resolving a conflict between two drives: scheming satisfaction (the plan worked) and recognition frustration (nobody witnessed it). The author decided recognition wins. The personality model — with its drive intensities, facet scores, and CAPS network — should make that determination, not us.

Worse: LLM persona research (ACL 2025) shows that when few-shot demonstrations and personality instructions conflict, demonstrations win. Formation memories ARE few-shot demonstrations. They're the strongest behavioural signal in the prompt. Every emotional evaluation we bake into a memory overrides what the personality model would derive.

We'd been fighting the wrong battle. The problem wasn't insufficient data richness. The problem was that our data was doing the pipeline's job — and doing it badly, because hand-authored emotional evaluations can't match the situational sensitivity of a computational personality model.

## The Pivot

Claude and I audited all 24 formation memories across five characters. We found pre-resolved drive conflicts in 11 of them. Action priming in 6. Value declarations in 5. A DD memory that contradicted what his own personality model would produce.

The fix wasn't more tuning. The fix was architectural: connect formation memories to the cognitive pipeline through a testable derivation chain, so we could verify at every step that the pipeline — not the author — was producing the emotional interpretation.

That's Part 2.
