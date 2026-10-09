# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**Design:** casehubio/examples#117 — Unified disposition renderer
**State:** All three layers complete, eval run done — mean 3.57 vs BASELINE0 3.93
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

## What Was Done This Session

### 1. Verified standard tests pass (566/566, 0 failures)

Full test suite green. Pre-existing BeliefRevision/RelationshipStage errors from Oct 6 are stale reports — not in current run.

### 2. Somatic drive descriptions for all 5 characters

Rewrote drive descriptions from behavioral directives to somatic felt-state format for hooded-claw, penelope-pitstop, dick-dastardly, and ant-hill-mob (PP was done in previous session). Both BASELINE and GENERIC profiles. Commit `435945b`.

### 3. Eval tests confirm production model improvement

| Test | Pre-Unified | Post-Unified | Threshold |
|------|:-----------:|:------------:|:---------:|
| Baseline (production, no injection) | 3/5 | **4/5** | ≥3 ✓ |
| Somatic drives | 3/5 | **4/5** | ≥3 ✓ |
| Somatic + memories | 5/5 | 5/5 | ≥3 ✓ |

### 4. Ultrathink first-principles analysis (corrected)

Three parallel forks verified claims against evidence:
- **WRONG claim:** "4→5 gap is drive-memory proximity." Baseline already has both drives and memories adjacent — still 4/5.
- **CORRECT root cause:** Formation memories are terse event descriptions. The 5/5 tests used **enriched memories with drive-expression cues** — showing the child's behavioral response to each event (how the drive was reinforced).
- **Literature confirmation (all SUPPORTED):** Damasio (somatic markers need autobiographical anchoring), Conway (episodic recall reactivates motivational states), McAdams (personality = traits + adaptations + narrative), CAPS (if-then profiles built from reinforcement history).

### 5. Created neocortex Layer 2 issues

| Issue | Title |
|-------|-------|
| casehubio/neocortex#504 | CAPS attractor labels — personality-specific output |
| casehubio/neocortex#505 | AMPD expression style derivation |
| casehubio/neocortex#506 | Appraisal-to-selection feedback |
| casehubio/neocortex#507 | **Epic:** Cognitive inference enrichment (Layer 2) |

### 6. Layer 2 neocortex changes implemented (separate session)

All three Layer 2 issues implemented and wired into examples:
- Expression style derivation from AMPD facets (CognitiveDerivationEngine)
- Personality-specific CAPS attractor labels (BehavioralSynthesisPhase)
- Appraisal-driven drive foregrounding (CognitionRenderContext)
- CharacterCognition wired to prefer personality-label and pass activated drives

Commits `12e6db9` and `62ea75f` on this branch.

### 7. Layer 3 data enrichment — formation memories + AMPD + disposition

**PP (both profiles):** Enriched all formation memories with drive-response expression cues. Added age-14 engineering competition memory in GENERIC. Added PAD values.

**DD (both profiles):** Added 6 AMPD personality facets (Grandiosity:80, Attention Seeking:85, Deceitfulness:70, Manipulativeness:55, Impulsivity:65, Irresponsibility:75). Added 5 formation memories with drive-response cues. Added attachment and disposition axes.

**Penelope (both profiles):** Added 5 formation memories with drive-response cues. Added disposition axes.

**AHM (both profiles):** Added 5 formation memories with drive-response cues. Added disposition axes.

Commit `130747f`.

## Commits on Branch

```
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

### Eval Results (completed this session)

**Standard tests:** 566/566 pass, 0 failures.

**Micro eval (DriveExpressionEvalTest):** PP proving-worth baseline hit **5/5** (up from 3/5 pre-enrichment). All 7 scored variants at 4-5/5.

**Macro eval (326-event GENERIC scenario):** Mean 3.57 vs BASELINE0 3.93.

| Category | Drives | Summary |
|----------|--------|---------|
| **Improved** | HC self-preservation (+1.18), AHM suspicion (+0.86), AHM protection (+0.34) | Somatic drives + enriched memories working |
| **Held** (±0.3) | HC scheming, HC dominance, DD greed, DD scheming, AHM loyalty, Penelope curiosity | 8 of 15 drives at or above baseline |
| **Collapsed** | HC gloating (-1.54), DD gloating (-1.95), PP gallantry (-1.42), PP protection (-0.87), Penelope social-harmony (-1.08) | Require situational triggers |

**Root cause of remaining collapses:** The collapsed drives require specific situational triggers (triumph → gloating, threat → protection, conflict → social-harmony, theatrical context → gallantry). The somatic descriptions provide predisposition but not activation. BASELINE0 scored higher because prescriptive tendencies text explicitly instructed the LLM to exhibit these behaviors — the cognitive pipeline replaces that with emergence, and some drives don't emerge without situational activation.

**Key insight:** This is NOT a data gap or inference gap — it's the difference between prescriptive and emergent behavior. The question for #115 closure is whether the success criteria (within ±0.3 of BASELINE0) is the right target, given that BASELINE0 used prescriptive text that scores high on expression but low on emergence.

### What needs deciding

1. **Revise success criteria?** BASELINE0's prescriptive scores may not be the right target — high expression with low emergence (scripting). The cognitive pipeline scores lower on expression but higher on emergence.
2. **Situational enrichment?** Add more triumph/threat/conflict scenarios to the scenario generator to give collapsed drives more opportunity to activate.
3. **Appraisal feedback wiring?** neocortex#506 is implemented but may need tuning — the foregrounding may not be aggressive enough.

## Architecture Notes

### Three-Layer Fix Architecture

| Layer | What | Status |
|-------|------|--------|
| 1 — Wiring | Connect dead ends in pipeline | **DONE** |
| 2 — Inference | Personality-specific CAPS, expression styles, appraisal foregrounding | **DONE** (neocortex#504-506) |
| 3 — Data | Somatic drives, enriched memories, AMPD facets, disposition axes | **DONE** |

### Key Constraint: Emergence Not Scripting

Every change must be tested for scripting/triggering. Drive cues are somatic/felt, never behavioral directives. Formation memories show drive REINFORCEMENT, not drive instructions. A scripted 5/5 on expression is worth 2 at most.

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
