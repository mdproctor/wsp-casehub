# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-29
**Branch:** wip/era5-relational-eval (examples), main (neocortex)
**Era:** 5 — relational cognition, attention competition finding

## Status

Era 5 investigation complete. #477 relational schemas are merged and operational — all five layers (schemas, appraisal, two-channel prompt, evolution, CAPS biasing) are wired. The CognitiveDerivationEngine generates emerged somatic descriptions (17 drives across 5 characters in 54 seconds).

**Key finding:** Adding relational context as SEPARATE prompt sections degrades somatic register from 3.74 to 3.13. Attention is finite and competitive — cognitive sections steal focus from the Barrett somatic instruction. The fix is compositional: fold relational context INTO drive labels as directional anchors (`→ Name`), not as separate sections.

## Era 5 Eval Results

| Run | Events | Quality | Register | Diff |
|-----|--------|---------|----------|------|
| Era 4 baseline | 300 | **3.92** | **3.74** | **3.74** |
| Era 5 + YAML briefing fallback | 30 | 3.84 | 3.27 | 3.70 |
| Era 5 label+degree only | 30 | 3.88 | 3.34 | 3.61 |
| Era 5 full pipeline (emerged descriptions) | 105 | 3.84 | 3.13 | 3.57 |

Register degraded progressively with more relational content. Quality held but didn't improve. Reactive drives (HC gloating, Peter gallantry) did not recover.

## Bugs Found and Fixed This Session

### 1. SessionPipeline not wired in scripted mode (TDD)
`consolidationScheduler.setSessionPipeline()` was inside `if (mode == AUTONOMOUS)`. In SCRIPTED mode, all LLM-calling consolidation phases (drive-description-generation, other-model-derivation, self-standard-derivation) silently skipped. Characters got YAML briefing prose or bare labels instead of emerged somatic descriptions.

**Test:** `ScriptedConsolidationPipelineTest` — structural assertion, red→green.
**Fix:** moved `setSessionPipeline` before the mode conditional.

### 2. YAML briefing prose fallback
`CognitionObservationBuilder` fell back to hand-authored YAML descriptions when no generated descriptions existed. This was cognitive narrative that contaminated the Barrett instruction.

**Fix:** fallback now renders label+degree only (`GLOATING (0.7)`).

### 3. ConditionMatcher NPE (#174)
Trigger conditions referenced cartoon IDs that don't exist after GENERIC ID rewrite. Null-safed — triggers return false for unknown characters.

### 4. hasSessionPipeline() added to ConsolidationScheduler
Testability API — reports whether LLM-calling phases will function.

## Next Session: Implement Targeted Somatic Drives

### The change

Replace separate cognitive sections with directional annotations on drives:

**Before (current — register 3.13):**
```
== What Pulls at You ==
GLOATING (0.7): [somatic description]

== How You Feel About Those Present ==
**Reginald:** Prey. Trust 0.1. Predictable.

== How You Bond ==
You don't need people. Closeness is vulnerability.
```

**After (targeted — expect register recovery):**
```
== What Pulls at You ==
GLOATING (0.7) → Reginald: [somatic description]
SCHEMING (0.9) → Clara: [somatic description]
```

### Implementation steps

1. **`CognitionObservationBuilder.renderInternalState()`** (neocortex) — for drives with `context: social`, resolve the relational target from nearby agents + relational schemas. Annotate the label: `DRIVE (intensity) → DisplayName`.

2. **`CharacterCognition.renderCognitiveSections()`** (examples) — remove calls to:
   - `renderRelationalSchemas()` (line 259-264)
   - `renderAttachmentStyle()` (line 266-268)
   - Trust sections (`renderTrustSections()`, line 270-273)

3. **Tests** — update any assertions that reference removed sections. Add test for `→ Name` annotation.

4. **Eval** — delete consolidation cache, rebuild, run 30-event eval. Compare register against Era 4 baseline (3.74). If register recovers → 300-tick validation.

### Key files

| What | Where |
|------|-------|
| Drive rendering (neocortex) | `cognition/src/main/java/.../CognitionObservationBuilder.java:139` |
| Section composition (examples) | `wacky-manor/src/.../CharacterCognition.java:259-273` |
| Relational schemas (YAML) | `wacky-manor/src/main/resources/META-INF/eidos/social-config-generic.yaml` |
| SocialConfig.Drive.context | `wacky-manor/src/.../SocialConfig.java:87` — `requiresSocialContext()` |
| Era 5 research doc | `neocortex/docs/research/era5-relational-cognition-research.md` |
| Blog entry | `wsp-casehub/blog/2026-10-29-mdp02-the-body-doesnt-listen.md` |
| Eval data | `wacky-manor/docs/eval/era5-full-pipeline-105-20261029/` |
| Consolidation cache | `~/claude/casehub/consolidation-cache/consolidation-cache-generic.json` |

### Constraints

- **No hand-crafted prose in social-config.** The `→ Name` annotation is computed, not authored.
- **Delete consolidation cache before eval.** Schemas change consolidation output.
- **Commit after every substantial change.** WIP commits, squash later.

## Commits This Session

| Repo | Branch | Key commits |
|------|--------|-------------|
| examples | `wip/era5-relational-eval` | ConditionMatcher fix (#174), SessionPipeline TDD fix, test updates, eval data (3 runs) |
| neocortex | `main` | briefing prose fallback fix, hasSessionPipeline, research doc update |
| wsp-casehub | `main` | blog: "The Body Doesn't Listen When You Talk About People" |
