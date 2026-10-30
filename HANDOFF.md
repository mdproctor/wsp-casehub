# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), main (neocortex)
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Status

Pipeline infrastructure complete (Steps 1-5). P22 generates golden-prose-quality descriptions (verified). Rendering layer wraps them correctly. **The LLM ignores the structural guidance and writes analytical prose.** This is the remaining problem.

Register improving: 3.13 → 3.33 → 3.47 (golden target: 4.20).

## Next Step — Single-Event Iteration

Pick ONE golden prose event. Reproduce its scenario in a smoke test. Iterate on the pipeline until the LLM output matches golden prose.

### The test

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pfast \
  -Dtest="PipelineToRenderingTest#goldenScenario*" -s .mvn/slot-settings.xml
```

This dumps the COMPLETE prompt HC receives — all sections. Compare against golden prose event 2 (hooded-claw aside in `golden-prose-20261030/transcript.json`).

### The gap

**Pipeline v2 output:** "The entrance hall. Dusty, grand, full of people I need to manage..." — analytical planning.

**Golden prose target:** "My chest settles into that architectural stillness... My spine finds its vertical before I've decided to stand straight..." — body-first discovery.

### What to tweak (ordered by likely impact)

| Component | File |
|-----------|------|
| Barrett instruction ("How to Feel") | `CharacterCognition.java:208-242` |
| Section ordering | `CharacterCognition.renderCognitiveSections()` |
| Drive descriptions (seeded in test) | `PipelineToRenderingTest.goldenScenario*` |
| P22 system prompt | `DriveDescriptionGenerationPhase.java:43-66` |
| Body-location table | `CognitionObservationBuilder.DRIVE_BODY_LOCATIONS` |

### Iteration loop

Tweak one thing → run test (5s) → send prompt to LLM (25s) → compare → repeat. No full evals until single-event output matches golden prose patterns.

## Key Files

| What | Where |
|------|-------|
| **Smoke test** | `examples/.../agent/PipelineToRenderingTest.java` → `goldenScenario_hcEntranceHall_fullPromptDump` |
| **Golden prose** | `examples/wacky-manor/docs/eval/golden-prose-20261030/transcript.json` |
| **Barrett instruction** | `examples/.../agent/CharacterCognition.java` lines 208-242 |
| **P22 system prompt** | `neocortex/.../consolidation/DriveDescriptionGenerationPhase.java` lines 43-66 |
| **Body-location table** | `neocortex/.../observation/CognitionObservationBuilder.java` → `DRIVE_BODY_LOCATIONS` |
| **Drive rendering** | `neocortex/.../observation/CognitionObservationBuilder.java` → `renderInternalState()` |
| **Classifier** | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| **Social config** | `examples/.../resources/META-INF/eidos/social-config-generic.yaml` |
| **Era 5 research** | `neocortex/docs/research/era5-relational-cognition-research.md` |

## Eval History

| Run | Events | Q | R | D | Notes |
|-----|--------|:-:|:-:|:-:|-------|
| **Golden v12 (target)** | **51** | **4.25** | **4.20** | **4.20** | All per-drive targets ≥4.0 |
| Era 4 baseline | 300 | 3.92 | 3.74 | 3.74 | — |
| Era 5 full pipeline | 105 | 3.84 | 3.13 | 3.57 | — |
| Era 5 pipeline v1 | 350 | 3.83 | 3.33 | 3.66 | Body-location anchors |
| **Era 5 pipeline v2** | **858** | **3.86** | **3.47** | **3.72** | Contrast, discovery verbs |

## Session 2 Summary

Steps 1-5 implemented. Two eval iterations. Three key findings:

1. **P22 produces golden-quality descriptions** — verified via direct Vertex API call. Body territory, contrast formula, discovery verbs all present. Description quality is NOT the bottleneck.
2. **LLM compliance gap** — the LLM ignores "give each pull its own paragraph starting with bold body-location anchor" and writes its natural analytical style instead. The prompt is right; the LLM doesn't follow it.
3. **Consolidation timing** — P22 skips tick 1 (no SessionPipeline), so early events have no descriptions. Biases eval averages down.

Also: ClassificationResult API migration (4 tests), BulkReExtractionService CDI exclusion, GENERIC_ID_MAP in classifier. 657 tests pass (16s).

## Constraints

- Do NOT remove cognitive sections — they represent abilities
- Isolation principle — one change at a time, measure, keep or revert
- Delete consolidation cache before eval runs
- Install neocortex before running examples
