---
layout: post
title: "From Authoring to Architecture"
date: 2026-10-13
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, autobiographical-memory, Conway-SMS, Singer-SDM, Klinger, CAPS, formation-memories, testing-pyramid, LLM-bias]
series: issue-115-drive-expression-gap
---

# From Authoring to Architecture

In Part 1, I described how tuning character data — somatic descriptions and formation memories — seemed to improve drive expression until we realised we were doing the pipeline's job for it. The fix isn't better data. It's a testable derivation chain where every step from personality traits to behaviour is independently verifiable.

The question was: what does the literature say about how memories connect to personality, and can we build that connection into the pipeline? We designed it, built it, and ran two full scenario evaluations. Here's what we found.

## What the Research Says

Three bodies of work turned out to be directly relevant.

**Conway's Self-Memory System** (Conway & Pleydell-Pearce, 2000; Conway, 2005). Autobiographical memories aren't replayed — they're reconstructed. The "working self" (current goals, active concerns) selectively activates relevant memories. You don't dump your entire life story into every moment; your current motivational state determines which memories surface. This is dynamic, context-dependent retrieval.

**Singer's Self-Defining Memories** (Singer, Blagov, Berry & Oost, 2013). Some memories have stable, enduring links to core concerns — they "possess a synchronic existence within narrative identity" and "remain relatively static." A DD memory about an unwitnessed triumph is always linked to recognition, regardless of what's happening right now. The link is structural, not dynamic.

**Klinger's Current Concerns** (Klinger, 1971; Klinger & Cox, 2004). Active motivational states increase cognitive salience for concern-relevant stimuli. When recognition is running hot, recognition-relevant memories surface more readily. This is already partially implemented in our cognitive pipeline via `ConcernBasedSalienceStrategy` — but formation memories weren't connected to it.

The critical insight: these are two complementary mechanisms at different levels, not competing theories. Singer's structural association says WHICH memories are linked to WHICH concerns (stable). Conway/Klinger's dynamic activation says which of those linked memories SURFACE in a given moment (context-dependent).

## The `serves` Annotation

This gave us the design: formation memories get inline annotations declaring which drive they serve, with a degree.

```yaml
- age: 14
  episode: >-
    You built an elaborate trap for the cat — a system of strings,
    counterweights, and a bucket. {{serves: scheming|strong: When it
    worked, there was a settling in your chest — the pieces had come
    together exactly as you'd mapped them.}} Nobody else saw it.
    {{serves: recognition|strong: It felt so much better when there
    was someone to show it to — the architecture deserved an audience.}}
```

The annotations capture Singer's structural link — this passage *serves* the recognition concern. The vocabulary is drive type names, which connect to CAPS network nodes via existing literature-sourced mappings (McClelland 1961 for achievement, Batson 1991 for prosocial motivation, Christie & Geis 1970 for Machiavellianism). The degree scale (faint, moderate, strong, defining) maps to Pillemer's directive function taxonomy — "defining" means the memory IS the formation of this concern.

At runtime, the annotations are stripped — the LLM receives clean narrative text. But the metadata feeds the Klinger salience mechanism: when a drive is active, memories annotated as serving that drive get higher activation weight.

The terminology comes from the autobiographical memory function literature. Pillemer (2003), Bluck (2003), and Singer all use "serves" for the memory-concern relationship: memories *serve* directive functions, memories *serve* as archetypes of personality themes. Not "feeds," not "grounds" — "serves" is the established term.

## The Audit

We annotated all 26 formation memories and fixed 23 problems: 11 pre-resolved drive conflicts, 6 action primings, 5 value declarations, and 1 outright contradiction. The DD age-14 contradiction — "that felt wrong" about a successful plan for a character with scheming at 0.85 — became the comparative framing: "it felt so much better when there was someone to show it to." Preserves scheming satisfaction while engaging recognition through comparison rather than negation.

Then we ran two full scenarios and averaged the results.

## What the Numbers Said

PP gallantry improved by 0.48 points — the largest sustained gain. Removing pre-resolutions that forced gallantry into proving-worth's frame let the personality model differentiate them. Pen adventure recovered to within 0.06 of the original baseline — the pre-resolution "not with fear but with something you'd later call 'the good kind of scared'" had been actively suppressing the adventure drive.

But HC gloating moved 0.06. DD gloating moved 0.02. Essentially flat. We'd cleaned the memories, removed the contradictions, annotated every drive link — and gloating didn't care.

This is where the micro tests changed the story.

## The Micro Test Revelation

We built Level 2-4 tests — single-situation drive expression judgments. Present one character with one triggering situation, judge whether the target drive expresses. When we tested HC gloating with a direct trigger (a rival humiliated by a trap HC had set), the score was 5/5. DD gloating: 5/5. Pen social-harmony: 5/5. Pen adventure: 5/5.

The LLM CAN express every drive at full intensity. The personality model produces rich, emergent, psychologically grounded responses. HC's gloating response included "somatic warmth pooling behind the ribs, swelling pressure, the urge to narrate the trick." DD's included "heat blooming in the chest, honey-thick satisfaction, compulsive laughter." These aren't mechanical outputs — they're the personality model doing exactly what it's supposed to do.

The problem isn't the LLM. The problem isn't the memories. The problem is that in a full scenario, the cognitive pipeline doesn't route gloating-relevant situations through gloating-relevant appraisals. When a rival fails, the pipeline processes the situation generically. The character's gloating drive is sitting at 0.7 intensity with CAPS nodes mapped to `dominance` and `BAS_activation` — but nothing in the pipeline says "this situation, for this character, should foreground these nodes."

Gloating collapse is a situation-to-drive routing gap.

## PP Gallantry and the LLM Bias Question

PP gallantry scored 3/5 on the direct trigger — not 5/5. The judge noted that gallantry was "instrumentalized as a coping mechanism for proving-worth anxiety: 'without something to do for her, what am I?'" The character expressed courtesy, but the internal experience was about proving worth through the courtesy.

This points to something we need to think carefully about: LLM training bias. Achievement narratives dominate training data. Stories about proving yourself, demonstrating competence, earning recognition — the LLM has seen millions of these. Stories about courtesy for its own sake, the pleasure of a well-shaped gesture with no instrumental purpose — it's seen far fewer.

The pipeline needs to account for this, but not through hard-wired rules. The character's personality profile (gallantry at 0.9, proving-worth at 0.7, callousness at 5) should determine which drive dominates — not a universal adjustment. The CAPS topology connects drives to nodes; the nodes compete through the network. If gallantry's nodes are insufficiently independent from proving-worth's nodes, the topology needs revision. That's a literature-informed architectural decision, not a data-tuning hack.

The same applies to gloating. LLMs may suppress schadenfreude because training data treats it as negative behaviour. A character with callousness at 90 and grandiosity at 80 should express gloating more readily than one with callousness at 5. The pipeline conveys that personality signal; the CAPS network routes it through the right nodes; the LLM receives a context where gloating is psychologically inevitable for this specific character. We're not overriding the LLM — we're giving it the information it needs to produce what the personality model predicts.

## The Testing Pyramid

The real win from this work is testability. Full scenario runs take 30 minutes and produce noisy results (±0.3 variance between runs — we saw adventure swing from 3.69 to 3.05 across two runs of the same configuration). What the annotation system gave us is a testing pyramid:

| Level | What | Time |
|---|---|---|
| 0 | Static consistency — does each annotation's text align with the named drive? | milliseconds |
| 1 | Prompt rendering — what does each character actually see? | seconds |
| 2 | Single-situation drive expression — one trigger, one drive, one judgment | ~10 seconds |
| 3 | Drive differentiation — trigger drive A, verify drive B doesn't absorb it | ~10 seconds |
| 4 | If-then behavioural signature — Mischel's CAPS prediction | ~1 minute |
| 5 | Full scenario — confirms micro improvements compose | ~30 minutes |

Level 2 is where the pipeline proves itself. If PP's gallantry doesn't express independently of proving-worth when someone needs help in a theatrical way, the problem is in the pipeline — not in the somatic description. Fix the pipeline, not the data.

Level 5 is for baseline confirmation. Run it after the micro tests pass. Not for iteration.

## What We're Building Next

The derivation chain from personality traits to behaviour now has testable links at every step:

```
Personality Model (facets, drives, attachment)
  → CAPS Network (literature-sourced mappings)
    → Enduring Concerns (drives as Klinger concerns)
      → Formation Memories (serves annotations — structural link)
        → Activated Memories (PAD-proximity — dynamic retrieval)
          → LLM Context → Emergent Behaviour
```

The `serves` annotations and the memory audit were Phase 0 — getting the data clean and testable. Phases 1-5 wire the pipeline: drive intensities prime CAPS nodes, CAPS nodes foreground appraisals, appraisals activate drive-relevant memories, and the rendering layer presents all of this as clean psychological context.

The constraint we're holding to: every mechanism must be rooted in published computational models (CAPS, Conway SMS, Klinger concerns, Mischel's if-then profiles). No hard-wired "rival fails → activate gloating" rules. The personality profile determines which drives activate; the pipeline provides the routing. The LLM produces the behaviour. And the testing pyramid verifies each step independently.

We started this session trying to tune data. We ended it understanding that the data was never the problem. The problem is that personality information doesn't flow through the system the way computational psychology says it should. The literature gives us the architecture. Now we wire it up.
