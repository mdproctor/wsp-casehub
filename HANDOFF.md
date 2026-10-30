# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), main (neocortex)
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Status

Pipeline infrastructure complete (Steps 1-5). **Barrett v3 breaks the LLM compliance gap.** Single-event smoke test hits golden standard: Q=4.25, R=4.50, D=4.0 — register jumped +1.03 from pipeline v2 (3.47 → 4.50).

Root cause was the Barrett instruction asking for a drive-catalog format the golden prose never used. Fix: reframe "STRUCTURE IN YOUR ASIDE" as scene-discovery instead of drive-catalog. Three key shifts: "body discovering" framing, scene anchors replacing body-location anchors, relaxed paragraph structure.

## Orientation — Start Here

Read these three things first:

1. **Golden prose target** — `examples/wacky-manor/docs/eval/golden-prose-20261030/transcript.json`. This is what the LLM output should look like. Event 0 is the next target (HC dialogue, entrance hall). Read the `thinking` field — that's the golden standard for body-first discovery prose.

2. **The smoke test** — `examples/wacky-manor/src/test/java/io/casehub/examples/manor/agent/PipelineToRenderingTest.java`, method `goldenScenario_hcEntranceHall_fullPromptDump` (line 583). This seeds exact drive intensities and P22 descriptions, renders through `CharacterCognition`, and dumps the FULL prompt the LLM receives. Event 1 (aside) is already covered. You need to add a new test method for event 0 (dialogue).

3. **Barrett v3 instruction** — `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java` lines 237-248. This is the "STRUCTURE IN YOUR ASIDE" section that was rewritten this session. It now says "write as the body discovering this scene" instead of "give each pull its own paragraph." This change is what broke the compliance gap.

**The classifier** is at `examples/wacky-manor/docs/eval/classify_emotions.py`. To score a single event: create a `transcript.json` with one event, run `python3 classify_emotions.py <dir> --character hooded-claw --mode full --verbose`.

## Next Step — Iterate Next Golden Prose Event

**Do not skip ahead.** One event at a time. Get event 0 to golden standard before touching event 5. Get event 5 before trying a different room. The methodology is: reproduce → iterate → classify → confirm → next.

Pick event 0 (HC dialogue, entrance hall). This tests whether the scene-discovery instruction works for DIALOGUE (not just aside). Create a smoke test, iterate, classify, confirm Q/R/D ≥ 4.0.

### Event queue

1. ~~HC aside, entrance hall (event 1)~~ — Q=4.25 R=4.50 D=4.0
2. **HC dialogue, entrance hall (event 0)** ← next
3. HC confrontation, entrance hall (event 5) — self-preservation under threat
4. HC in different room — instruction generalisation
5. Different character (PP or AHM) — different personality/attachment

After 5+ events at golden standard → full 300-tick eval.

### The iteration loop

1. Create smoke test reproducing the golden event scenario
2. Run test (5s) → dumps full prompt
3. Send prompt to LLM → get output
4. Compare against golden prose target
5. If gap exists: tweak ONE pipeline component → repeat from 2
6. When output matches: classify → confirm Q/R/D ≥ 4.0 → move to next event

### The test

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pfast \
  -Dtest="PipelineToRenderingTest#goldenScenario*" -s .mvn/slot-settings.xml
```

### What to tweak (ordered by likely impact)

| Component | File |
|-----------|------|
| Barrett instruction ("How to Feel") | `CharacterCognition.java:208-248` |
| Section ordering | `CharacterCognition.renderCognitiveSections()` |
| Drive descriptions (seeded in test) | `PipelineToRenderingTest.goldenScenario*` |
| P22 system prompt | `DriveDescriptionGenerationPhase.java:43-66` |
| Body-location table | `CognitionObservationBuilder.DRIVE_BODY_LOCATIONS` |

## What Barrett v3 Changed

Previous instruction: "give each active pull its own paragraph starting with its bold body-location anchor" — asked for drive-catalog format.

New instruction: "Write as the body discovering this scene — not as a mind analysing it. Bold the scene anchor — **a person acting**, **a sensation arriving**, **a body territory waking** — and let the drives surface as reactions to what is happening, not as a catalog."

Three shifts:
1. "Body discovering" vs "give each pull its own paragraph" — experiencing vs cataloging
2. "Scene anchor" vs "body-location anchor" — anchors to what's happening, not drive labels
3. "Multiple drives can share a paragraph" — removes rigid one-drive-per-paragraph

## Known Gaps

- **Gloating D=3** on event 1 — paragraph blends into dominance without contrast clause. May need P22 contrast fix if it recurs.
- **N=1** — single event, single character, single model (Opus 4.6). Production uses Gemini.
- **Section ordering** — prompt still front-loads analytical content before embodied instruction. Not yet tested whether reordering matters.

## Key Files

| What | Where |
|------|-------|
| **Smoke test** | `examples/.../agent/PipelineToRenderingTest.java` → `goldenScenario*` |
| **Golden prose** | `examples/wacky-manor/docs/eval/golden-prose-20261030/transcript.json` |
| **Barrett v3 eval** | `examples/wacky-manor/docs/eval/barrett-v3-single/transcript.json` |
| **Barrett instruction** | `examples/.../agent/CharacterCognition.java` lines 208-248 |
| **Era 5 research** | `neocortex/docs/research/era5-relational-cognition-research.md` (see §Barrett v3) |
| **Classifier** | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| **P22 system prompt** | `neocortex/.../consolidation/DriveDescriptionGenerationPhase.java` lines 43-66 |
| **Body-location table** | `neocortex/.../observation/CognitionObservationBuilder.java` → `DRIVE_BODY_LOCATIONS` |
| **Social config** | `examples/.../resources/META-INF/eidos/social-config-generic.yaml` |

## Eval History

| Run | Events | Q | R | D | Notes |
|-----|--------|:-:|:-:|:-:|-------|
| **Golden v12 (target)** | **51** | **4.25** | **4.20** | **4.20** | All per-drive targets ≥4.0 |
| Era 4 baseline | 300 | 3.92 | 3.74 | 3.74 | — |
| Era 5 full pipeline | 105 | 3.84 | 3.13 | 3.57 | — |
| Era 5 pipeline v1 | 350 | 3.83 | 3.33 | 3.66 | Body-location anchors |
| Era 5 pipeline v2 | 858 | 3.86 | 3.47 | 3.72 | Contrast, discovery verbs |
| **Barrett v3 (1 event)** | **1** | **4.25** | **4.50** | **4.00** | Scene-discovery instruction |

## Constraints

- Do NOT remove cognitive sections — they represent abilities
- Isolation principle — one change at a time, measure, keep or revert
- Delete consolidation cache before eval runs
- Install neocortex before running examples
