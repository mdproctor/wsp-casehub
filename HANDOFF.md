# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** All layers complete, neocortex pipeline fully wired — awaiting eval run
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

## What Was Done This Session

### 1. Verified neocortex epic fully closed

All six neocortex issues (casehubio/neocortex#513 epic) confirmed CLOSED:

| Issue | What |
|-------|------|
| neocortex#506 | Appraisal-to-selection feedback |
| neocortex#508 | Klinger Current Concerns — motivational salience |
| neocortex#509 | Wire drive adaptation into CAPS settling and appraisal |
| neocortex#510 | CAPS settling dampens too aggressively |
| neocortex#511 | Pipeline tuning — graduated placement, differentiation |
| neocortex#512 | Personality-driven trust interpretation |

Neocortex 0.2-SNAPSHOT jars installed locally (Oct 12 03:32). CDI-wired paths (drive→CAPS settling, expression styles, dampening fix) take effect automatically.

### 2. Wired appraisal-driven drive foregrounding (#118)

`renderInternalState()` now accepts activated drive categories from `cognitionCore.activatedDriveCategories()`. Drives activated by the current situation render first (salience ordering), ambient drives follow by intensity. Changed from `private` to package-private for testability.

Two new tests: `renderInternalStateForegoundsActivatedDrives` (verifies activated drive reorders above higher-intensity ambient) and `renderInternalStateNoActivationPreservesIntensityOrder` (verifies fallback to intensity ordering).

568/568 tests pass. Commit `4eae1bf`.

### 3. Wired Klinger concern-based salience (#119)

Replaced passThrough lambda in `PlaybookOrchestrator.configureAppraisal()` with `ConcernBasedSalienceStrategy`. Enabled `salienceEnabled` in `CognitionConfig` (was `false` in `CognitionConfig.all()`).

The strategy matches observation text against active concerns (drive descriptions + active goals) via keyword overlap. High-salience concerns feed into `activatedDriveCategories()`, composing with the drive foregrounding from #118.

568/568 tests pass. Commit `c45d7a1`.

### 4. Confirmed #117 design implemented — closeable

The unified disposition renderer design (#117) was fully implemented in commits `169b093` and `e30fe7a` — `renderInternalState()`, boundary tests, BehavioralPromptSection disabled. Issue is still OPEN on GitHub, ready to close.

## Commits on Branch

```
c45d7a1 feat(#119): wire Klinger concern-based salience into cognitive pipeline
4eae1bf feat(#118): wire appraisal-driven drive foregrounding into renderInternalState
c6dfa54 feat(#118): wire CognitiveDefaultsRegistry into RelationshipStagePhase
509b7a3 docs(#115): Layer 1+2+3 eval results — mean 3.57 vs BASELINE0 3.93
6617d92 data(#115): save Layer 1+2+3 scenario run transcript (326 events, GENERIC)
3f4f3af fix(#115): update ManorSocialConfigLoaderTest for somatic drive descriptions
130747f feat(#115): enrich formation memories with drive-expression cues + fill data gaps
62ea75f feat(#115): wire appraisal-driven foregrounding into CharacterCognition
12e6db9 feat(#115): prefer personality-label in unified attractor renderer
435945b wip(#115): rewrite all character drives to somatic felt-state format
d02a0d4 docs(#118): add literature-first design principle to CLAUDE.md
e30fe7a test(#115): update section count and add boundary tests for unified renderer
169b093 wip(#115): unified renderInternalState — somatic drives + CAPS attractors
f9a683f fix(#115): wire dead ends 1/3/4 in PlaybookOrchestrator
fbe0262 test(#115): update CharacterCognitionTest for formation memories section
42f8113 fix(#115): wire renderFormationMemories into renderCognitiveSections
65044d3 test(#115): add propagation boundary tests for 5 dead ends
```

## What Needs to Happen Next

### 1. Run eval to measure impact of pipeline wiring

The last eval (mean 3.57 vs BASELINE0 3.93) was BEFORE:
- Appraisal-driven drive foregrounding (#118)
- Klinger concern-based salience (#119)
- CAPS dampening fix (neocortex#510 — attractors can now reach established/deep-rooted)
- Personality-specific attractor labels (neocortex#504)
- Expression style derivation from AMPD (neocortex#505)
- Drive adaptation → CAPS settling loop (neocortex#509)
- Pipeline tuning (neocortex#511)

A new GENERIC scenario run + `classify_emotions.py` is the critical next step. Success criteria: mean ≥ 3.93, no drive > 0.3 below BASELINE0.

### 2. Close #117

Design implemented. Close with comment referencing commits `169b093` and `e30fe7a`.

### 3. Assess #115/#118/#119 closure after eval

If eval meets success criteria:
- Close #115 (epic complete)
- Close #118 (drive adaptation wired)
- Close #119 (Klinger salience wired)

If eval shows remaining gaps, formation memory salience modulation is the next lever — ordering memories by drive relevance in `renderFormationMemories()`. This was deferred because it requires adding drive associations to formation memories (data change) and lacks empirical justification yet.

### 4. Independent tasks (can do anytime)

| Issue | What |
|-------|------|
| #116 | CognitivePipelineBuilder extraction |
| #114 | DD missing tendencies |

### Three-Layer Fix Architecture (updated)

| Layer | What | Status |
|-------|------|--------|
| 1 — Wiring | Connect dead ends in pipeline | **DONE** |
| 2 — Inference | CAPS labels, expression styles, appraisal foregrounding, Klinger salience, drive-CAPS loop, dampening fix, pipeline tuning | **DONE** |
| 3 — Data | Somatic drives, enriched memories, AMPD facets, disposition axes | **DONE** |

### Key Constraint: Emergence Not Scripting

Prescriptive text inferred from the personality model (AMPD facets → derived tendencies) is legitimate. Hand-written behavioral labels are not.

## Baseline Transcripts — DO NOT DELETE

| Transcript | Events | What it tests | Profile |
|---|---|---|---|
| `old-briefings-20261002/transcript.json` | 327 | Verbose briefings (pre-rewrite) | BASELINE |
| `new-briefings-20261002/transcript.json` | 333 | Directive-minimal (post-rewrite) | BASELINE |
| `generic-briefings-20261002/transcript.json` | 306 | Generic + prescribed catchphrases | GENERIC |
| `generic-colloquial-20261002/transcript.json` | 310 | Generic + style-directive only | GENERIC |
| `full-run-2026-10-08.json` | — | Full run baseline | — |
| `pipeline-hardening-run-2026-10-08/transcript.json` | — | Pipeline hardening run | — |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```
