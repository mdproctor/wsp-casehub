# HANDOFF — casehub-examples — 2026-10-13

## START HERE

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Phase 0 complete (all sub-phases). Post-audit baseline established (2-run average).
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

**Key documents:**
- Research: `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`
- Design: `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-09-drive-integration-architecture-design.md`
- Eval results: `wacky-manor/docs/eval/phase0-audit-baseline-2026-10-13/RESULTS.md`

## What Was Done

### Phase 0 — Formation Memory Audit + `serves` Annotation System

| Phase | Commit | What |
|---|---|---|
| 0a | `85e6d3a` | `ConcernAnnotation` record, `FormationMemoryParser`, loader + renderer |
| 0b | `dcfdf2c` | Annotated all 26 formation memories with `serves` annotations |
| 0c | `9bc9dec` | Fixed 11 pre-resolutions, 6 action primings, 5 value declarations, 1 contradiction |
| 0d | `a83cfba` | 6 micro tests for 5 collapsed drives + baseline scores captured |
| 0e-r1 | `bab91f7` | Full scenario run 1 (294 events), classified |
| fix | `a2f5e9a` | Restored Pen age-7 and DD age-6 felt intensity |
| 0e-r2 | `58de5db` | Full scenario run 2 (241 events), classified |

### Phase 0 Eval Summary (2-run average, mean 3.72)

| Drive | B0 | L4b | P0-avg | vs L4b | Verdict |
|---|---|---|---|---|---|
| HC gloating | 3.97 | 2.50 | 2.56 | +0.06 | Collapsed — pipeline problem |
| DD gloating | 3.50 | 2.76 | 2.78 | +0.02 | Collapsed — pipeline problem |
| PP gallantry | 4.50 | 3.08 | 3.56 | +0.48 | Recovering — audit helped |
| Pen social-harmony | 3.18 | 2.71 | 2.66 | -0.05 | Stable — needs pipeline |
| Pen adventure | 3.75 | 3.24 | 3.37 | +0.13 | Modest improvement |

### Micro Test Baselines

HC gloating 5/5, DD gloating 5/5, PP gallantry 3/5, PP gallantry-diff 3/5, Pen social-harmony 5/5, Pen adventure 5/5.

**Critical insight:** LLM CAN express all drives perfectly on direct trigger (5/5). Gloating collapse is a situation-to-drive routing gap in the pipeline, not LLM capability or memory content.

**LLM bias consideration (from user):** Pipeline work must be literature-rooted (CAPS, Mischel if-then profiles), not hard-wired rules. LLM bias against "negative" drives (gloating/schadenfreude) may need balanced counterweighting — informed by the character's personality profile (callousness, grandiosity), not universal adjustments.

## What Needs to Happen Next

### 1. Phases 1-5 — Drive Integration Pipeline

Per design spec: `2026-10-09-drive-integration-architecture-design.md`

The gloating collapse requires pipeline-level support:
- Phase 1: Drive→CAPS activation (connect drive intensities to CAPS node priming)
- Phase 2: Drive→Appraisal (foreground drive-relevant appraisals)
- Phase 3: Drive→Memory salience (use `serves` annotations for concern-based retrieval)
- Phase 4: Enrichment (catalogue entry connection, situationTypes from annotations)
- Phase 5: Somatic removal (replace prescriptive text with derived prompts)

**Approach constraint:** Mechanisms must be rooted in literature (CAPS topology, Mischel's if-then profiles, Conway/Singer memory systems). No hard-wired "situation→drive" rules. The personality profile (drives + facets + CAPS network) determines which drives activate — the pipeline provides the routing, not the answers.

### 2. DD greed regression (-0.56 vs L4b)

Needs investigation. The pre-resolution removal may have weakened the greed signal beyond what the pipeline can compensate for.

### 3. PP gallantry differentiation

3/5 on differentiation test — gallantry instrumentalized as proving-worth container. CAPS topology review needed to ensure gallantry nodes have sufficient independence.

### 4. Close #117

Design implemented (commits `169b093`, `e30fe7a`). Close with comment.

## Baseline Transcripts — DO NOT DELETE

| Transcript | Events | What it tests | Profile |
|---|---|---|---|
| `old-briefings-20261002/transcript.json` | 327 | Verbose briefings (pre-rewrite) | BASELINE |
| `new-briefings-20261002/transcript.json` | 333 | Directive-minimal (post-rewrite) | BASELINE |
| `generic-briefings-20261002/transcript.json` | 306 | Generic + prescribed catchphrases | GENERIC |
| `generic-colloquial-20261002/transcript.json` | 310 | Generic + style-directive only | GENERIC |
| `full-run-2026-10-08.json` | — | Full run baseline | — |
| `pipeline-hardening-run-2026-10-08/transcript.json` | — | Pipeline hardening run | — |
| `layer-123-run-2026-10-09/transcript.json` | 326 | Layer 1+2+3 baseline | GENERIC |
| `pipeline-wiring-run-2026-10-13/transcript.json` | 997 | Full pipeline wiring | GENERIC |
| `layer4-refinement-run-2026-10-13/transcript.json` | 349 | L4a somatic + memory refinements | GENERIC |
| `layer4-gallantry-fix-2026-10-13/transcript.json` | 300 | L4b gallantry somatic fix | GENERIC |
| `phase0-audit-baseline-2026-10-13/transcript.json` | 294 | Phase 0 run 1 (post-audit) | GENERIC |
| `phase0-fix-run-2026-10-13/transcript.json` | 241 | Phase 0 run 2 (felt-intensity fix) | GENERIC |

## Build Notes

```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```
