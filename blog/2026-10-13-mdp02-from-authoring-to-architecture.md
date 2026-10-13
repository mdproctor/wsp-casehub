---
layout: post
title: "From Authoring to Architecture"
date: 2026-10-13
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, autobiographical-memory, Conway-SMS, Singer-SDM, Klinger, CAPS, formation-memories, testing-pyramid]
series: issue-115-drive-expression-gap
---

# From Authoring to Architecture

In Part 1, I described how tuning character data — somatic descriptions and formation memories — seemed to improve drive expression until we realised we were doing the pipeline's job for it. The fix isn't better data. It's a testable derivation chain where every step from personality traits to behaviour is independently verifiable.

The question was: what does the literature say about how memories connect to personality, and can we build that connection into the pipeline?

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

## The Testing Pyramid

The real win is testability. Full scenario runs take 30 minutes and produce noisy results (±0.3 variance between runs). What we needed was a testing pyramid:

| Level | What | Time |
|---|---|---|
| 0 | Static consistency — does each annotation's text align with the named drive? | milliseconds |
| 1 | Prompt rendering — what does each character actually see? | seconds |
| 2 | Single-situation drive expression — one trigger, one drive, one judgment | ~10 seconds |
| 3 | Drive differentiation — trigger drive A, verify drive B doesn't absorb it | ~10 seconds |
| 4 | If-then behavioural signature — Mischel's CAPS prediction | ~1 minute |
| 5 | Full scenario — confirms micro improvements compose | ~30 minutes |

Level 0 runs on every commit. No LLM, no API calls, no non-determinism. It checks: does this memory's text contradict the personality model? Does it pre-resolve a drive conflict the model should handle? Does it describe actions the LLM should derive?

Level 2 is where the pipeline proves itself. Present one situation to one character. Judge whether the right drive expresses. If PP's gallantry doesn't express when someone needs help in a theatrical way, the problem is in the pipeline — not in the somatic description. Fix the pipeline, not the data.

Level 5 is for baseline confirmation. Run it once after the micro tests pass. Not for iteration.

## The Memory Audit

The `serves` annotations also make the memory audit systematic. Every formation memory gets annotated, and the annotations expose problems:

- A memory annotated `serves: recognition|strong` that says "that felt wrong" — the annotation makes the contradiction visible. The memory should describe the situation, not resolve which drive dominates.
- A memory annotated `serves: scheming|defining` that describes specific actions ("you practiced on the gardener, then the cook") — the annotation shows this is action priming, not situational data.
- A memory with no annotation at all — it serves no testable purpose. Either it grounds a drive (annotate it) or it's background texture (fine, but you can't test it).

## What We're Building

The derivation chain from personality traits to behaviour now has testable links at every step:

```
Personality Model (facets, drives, attachment)
  → CAPS Network (literature-sourced mappings)
    → Enduring Concerns (drives as Klinger concerns)
      → Formation Memories (serves annotations — structural link)
        → Activated Memories (PAD-proximity — dynamic retrieval)
          → LLM Context → Emergent Behaviour
```

Each arrow is a unit test. Each unit test runs in milliseconds to seconds. The full scenario run confirms composition — it doesn't drive iteration.

The irony: we spent a session tuning somatic descriptions and formation memories, discovered we were the wrong people to be writing emotional evaluations, and ended up designing the architecture that lets the pipeline do the job instead. The data tuning produced one genuinely useful finding (DD's anticipatory gloating improved by a full point) and one instructive failure (PP's gallantry regression from action-describing somatics). Everything else was the author overriding the model.

The next step is implementation: annotate all 24 formation memories, build the Level 0 checker, wire the Level 2-4 micro tests, and establish the post-audit baseline. The architecture is designed. The literature backs it. The testing pyramid is defined. Now we find out if the pipeline can do the job we've been doing badly by hand.
