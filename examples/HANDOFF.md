# HANDOFF — casehub-examples — 2026-10-13

## START HERE

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Phase 0 complete — `serves` annotation system built, all memories audited, post-audit baseline established
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

**Research document (still authoritative for theory + pipeline design):**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

**Phase 0 eval results:**
`wacky-manor/docs/eval/phase0-audit-baseline-2026-10-13/RESULTS.md`

## What Was Done This Session

### Phase 0 — Formation Memory Audit + `serves` Annotation System

| Phase | Commit | What |
|---|---|---|
| 0a | `85e6d3a` | `ConcernAnnotation` record, `FormationMemoryParser`, loader + renderer updates |
| 0b | `dcfdf2c` | Annotated all 26 formation memories with `serves` annotations |
| 0c | `9bc9dec` | Fixed 11 pre-resolutions, 6 action primings, 5 value declarations, 1 contradiction |
| 0d | `a83cfba` | 6 micro tests for 5 collapsed drives (HC/DD gloating, PP gallantry, Pen harmony/adventure) |
| 0e | `bab91f7` | Full scenario run (294 events), post-audit baseline established |

### Phase 0 Eval Results (mean 3.76, vs L4b 3.82, vs B0 3.96)

**Winners:** Pen adventure +0.45 (near B0), PP gallantry +0.34, PP protection +0.59
**Flat:** HC gloating +0.09, DD gloating +0.09 — confirms gloating is a pipeline problem, not memory authoring
**Regressed:** Pen social-harmony -0.29 — age-7 memory may have been over-trimmed

### Key Finding

The audit confirmed that **gloating collapse is not a memory problem** — it's a pipeline gap. Clean memories with proper annotations show essentially no gloating improvement (+0.09). This validates the need for Phases 1-5 (drive→CAPS, drive→appraisal, drive→memory salience integration).

The audit DID fix **adventure** (nearly to B0) and **gallantry** (significant recovery), showing that pre-resolutions were actively suppressing these drives.

## What Needs to Happen Next

### 1. Investigate Pen social-harmony regression (-0.29)

The age-7 memory was shortened significantly (duck story + value declaration removed). The felt experience of physical sickness from conflict may need to be restored without the action priming and value declaration. Consider:
- Restoring bodily felt experience (tightness, nausea) while removing the duck story
- Adding more situational detail without specifying what Penelope DOES

### 2. Consider restoring DD age-6 felt experience

"The fullness wasn't just in your stomach" was removed as a pre-resolution, but it carried useful felt experience. Consider restoring the physical sensation without the "not X, it was Y" framing.

### 3. Proceed to Phases 1-5 (Drive Integration Pipeline)

Per existing design spec: `2026-10-09-drive-integration-architecture-design.md`

The gloating collapse requires pipeline-level support:
- Phase 1: Drive→CAPS activation (connect drive intensities to CAPS node priming)
- Phase 2: Drive→Appraisal (foreground drive-relevant appraisals)
- Phase 3: Drive→Memory salience (use `serves` annotations for concern-based retrieval)
- Phase 4: Enrichment (catalogue entry connection, situationTypes from annotations)
- Phase 5: Somatic removal (replace prescriptive text with derived prompts)

### 4. Close #117

Design implemented (commits `169b093`, `e30fe7a`). Close with comment referencing those commits.

## Architecture Summary

| Layer | What | Status |
|-------|------|--------|
| 0 — `serves` annotations | Annotate memories, audit, build micro tests, establish baseline | **DONE** |
| 1-5 (drive integration) | Drive→CAPS, Drive→Appraisal, Drive→Memory salience, rendering cleanup | **NEXT** |

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
| `phase0-audit-baseline-2026-10-13/transcript.json` | 294 | Phase 0 post-audit baseline | GENERIC |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

## Blog Entries

- `wsp-casehub/blog/2026-10-13-mdp01-when-your-characters-stop.md` — Part 1: data tuning trap
- `wsp-casehub/blog/2026-10-13-mdp02-from-authoring-to-architecture.md` — Part 2: literature → architecture
