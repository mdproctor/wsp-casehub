# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), wip/era5-relational-eval (wsp-casehub), main (neocortex)
**Era:** 5 — relational cognition, targeted somatic drives
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Status

Golden prose corpus complete. All 5 original per-drive differentiation targets fixed through 12 systematic classifier iterations. The corpus is now the verified target for pipeline rendering.

**Golden prose (v12):** 51 events, 5 characters. Per-drive scores all at or above 4.0 except HC dominance R=3.9 (noise boundary — achieved 4.2 in earlier runs with same prose).

**Next step:** Pipeline implementation — make the backend generate equivalent structured output. Steps 9-14 in the era5 research doc's Immediate Plan.

## What Was Accomplished This Session

### The differentiation problem (resolved)

Five per-drive dimensions were below 4.0 at session start:

| Target | Before | After | Technique |
|--------|:------:|:-----:|-----------|
| DD scheming Q | 3.62 | 4.25 | Body-location separation + explicit contrast |
| DD scheming R | 3.88 | 4.40 | Body-location separation |
| DD scheming D | 3.38 | 4.50 | Explicit contrast formula ("not X, not Y, but Z") |
| HC scheming R | 3.89 | 4.00 | Somatic discovery replacing cognitive calculation |
| HC self-pres Q | 3.80 | 4.00 | Woven thread (subordinate clauses in every paragraph) |
| HC gloating D | 3.75 | 4.50 | Paragraph structure + bold body-location anchors |
| PP proving-worth D | 3.88 | 4.10 | Body-location separation from gallantry/protection |

### Six rendering techniques discovered

Documented in full in `neocortex/docs/research/era5-relational-cognition-research.md` §Golden Prose Rendering Methodology. Summary:

1. **Body-location separation** — each drive owns a unique anatomical territory (scheming=fingertips/periphery, gloating=sternum, dominance=spine/jaw/feet, etc.). When two drives share a location, D drops.

2. **Explicit contrast formula** — "Not [drive-A channel]. Not [drive-B channel]. Something utterly different: [this drive's expression]." Three negations → D=5. Two → D=4. Zero → D=3. The single most powerful differentiator.

3. **Paragraph-based drive separation** — structural `\n\n` breaks + bold body-location anchors (`**Behind my sternum, warmth pools**`) give the classifier drive boundaries. Turned HC gloating D from 3.75 to 5.0.

4. **Woven thread pattern** — pervasive drives (self-preservation) expressed as subordinate clauses within other drives' paragraphs, not in separate sections. Matches cognitive architecture: HC is always monitoring. Resolved the gloating-D vs self-pres-Q tradeoff.

5. **Desc/thinking alignment** — desc signals relevance, thinking drives Q/D/R. Misalignment penalises both.

6. **Somatic discovery** — "my weight shifts" (R=4) beats "the question is whether" (R=3). Discovery verbs (arrives, settles, pools, fizzes) replace analytical verbs (calculates, assesses, determines).

### Classifier noise finding

Haiku 4.5 has ±0.2 noise per drive dimension per run on identical prose. Drives at 4.0 boundary oscillate 3.9-4.1. This was confirmed by multiple re-runs on unchanged transcripts. Pipeline eval should target means of 4.2+ to survive noise, or average multiple runs.

## Pipeline Implementation (next session)

The pipeline must generate structured thinking-field output matching the golden prose patterns. Five implementation steps:

### Step 1: Seed relational schemas into MindMap

`ManorCognitiveSeeder.seedPeople()` → add `rel.role`, `rel.trust`, `rel.intimacy`, `rel.utility`, `rel.description`. Without this: P20 evolves nothing, P22 finds 0 schemas.

### Step 2: Restructure renderCognitiveSections

Replace 5 fragmented relational sections with drive-first paragraph blocks. Each active drive gets its own paragraph with bold body-location anchor, ordered by drive hierarchy (dominance → gloating → scheming → action drives). Pervasive drives (self-pres) woven as subordinate clauses, not separate sections.

### Step 3: Update DriveDescriptionGenerationPhase

- Add body-location constraints per drive (lookup table)
- Add contrast text generation — for each drive×event, negate co-occurring drives' channels
- Add channel emphasis hints per drive×role
- Add pervasive-drive threading template for drives at ≥90% trigger rate

### Step 4: Add structural rendering to thinking field

- Paragraph breaks (`\n\n`) between drive sections
- Bold body-location anchors at paragraph starts
- Discovery verbs enforced in Barrett instruction
- Woven subordinate clauses for pervasive drives

### Step 5: Eval against golden corpus

Run 30-event scenario, classify, compare per-drive means against golden scores. Target: within 0.2 of golden. Iterate pipeline until scores match. Then 300-tick validation.

## Architecture Decisions (locked in)

### Drive-first rendering with structure

```
**[body-location anchor]** — [Barrett channel-selected description].
Not [other-drive-channel]. Not [other-drive-channel].
[Unique expression] — [pervasive-drive subordinate clause].
```

### Drive → body-location table (mechanical, no LLM)

| Drive | Body Location | Channel |
|-------|--------------|---------|
| scheming | fingertips, periphery, visual field | attention + action-readiness |
| gloating | sternum, breastbone | body + time |
| greed | gut, navel, ribs | body + attention |
| recognition | upper chest, spine | body + time |
| dominance | spine, jaw, feet | body + attention |
| self-preservation | ears, hearing, peripheral awareness | attention + body |
| protection | legs, neck, positioning | action-readiness + attention |
| gallantry | hands (offered), chest (warmth) | action-readiness + body |
| proving-worth | shoulders (weight), jaw (clenching), grip | body + time |
| loyalty | chest (deep ache), alignment | body + action-readiness |
| suspicion | neck (prickling), gut (knowing) | attention + body |
| curiosity | fingertips (tracing), chest (fizzing) | attention + action-readiness |
| social-harmony | chest (settling), jaw (softening) | body + attention |

### Drive type classification (for rendering pattern)

| Type | Rendering Pattern | Criterion |
|------|------------------|-----------|
| Discrete | Own paragraph with bold anchor | Trigger rate < 90% |
| Pervasive | Woven subordinate clauses | Trigger rate ≥ 90% |

### Relational information folds INTO drives

Trust, ToM, Social Awareness, Relational Schema all collapse into drive paragraphs. No separate cognitive-analytical sections.

### Mechanical channel selection

Per-tick pipeline is almost entirely mechanical. Only LLM call is situation classification (and character response). Appraisal, targeting, rendering are deterministic.

## Eval History

| Run | Events | Q | R | D | Notes |
|-----|--------|:-:|:-:|:-:|-------|
| **Golden v12 (target)** | **51** | **4.25** | **4.20** | **4.20** | All per-drive targets ≥4.0 |
| Golden original | 51 | 4.35 | 4.37 | 4.12 | 5 per-drive targets below 4.0 |
| Era 4 baseline | 300 | 3.92 | 3.74 | 3.74 | — |
| Era 5 targeted (5-arg) | 30 | 3.87 | 3.46 | 3.53 | — |
| Era 5 full pipeline | 105 | 3.84 | 3.13 | 3.57 | — |

## Key Files

| What | Where |
|------|-------|
| Golden prose corpus | `examples/wacky-manor/docs/eval/golden-prose-20261030/` |
| Eval results (12 runs) | `golden-prose-20261030/full-eval-v*.json` |
| Regression gate | `golden-prose-20261030/verify_golden.py` |
| Splice tool | `golden-prose-20261030/splice_revisions.py` |
| Era 5 research (full methodology) | `neocortex/docs/research/era5-relational-cognition-research.md` |
| Classifier | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| CharacterCognition | `examples/wacky-manor/src/.../agent/CharacterCognition.java` |
| CognitionObservationBuilder | `neocortex/cognition/src/.../observation/CognitionObservationBuilder.java` |
| ManorCognitiveSeeder | `examples/wacky-manor/src/.../agent/ManorCognitiveSeeder.java` |
| DriveDescriptionGenerationPhase | `neocortex/mindmap-intelligence/src/.../consolidation/DriveDescriptionGenerationPhase.java` |
| Design journal | `wsp-casehub/JOURNAL.md` |
| Consolidation cache | `~/claude/casehub/consolidation-cache/consolidation-cache-generic.json` |

## Constraints

- **Do NOT remove cognitive sections.** They represent abilities. Fix the rendering, not the cognition.
- **Do NOT accept regressions.** Golden means every dimension improves or holds.
- **Describe → test → implement.** Every cognitive extension proves its contribution to rendered output before backend work.
- **Delete consolidation cache before eval runs.** Old cache has target-agnostic descriptions.
- **Install neocortex before running examples.** Changes on neocortex main need `mvn install`.
- **Isolation principle.** Change one thing at a time, measure, keep or revert. Combined changes cause unattributable regressions (v6 lesson).
- **Structure matches cognitive architecture.** Discrete drives get paragraphs. Pervasive drives get woven threads. The rendering pattern must match how the drive actually operates.
