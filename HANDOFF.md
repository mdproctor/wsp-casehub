# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Layer 4 data tuning attempted → architectural pivot to `serves` annotations
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

## START HERE — Research Reference

**Read this first:**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

This is the complete knowledge base for the next iteration. It covers:
- All literature findings (Conway SMS, Singer SDMs, Klinger, CAPS, LLM persona research) with specific citations and implementation mapping
- Full memory audit results (26 memories, per-memory findings)
- All eval data from this session (L4a mean 3.79, L4b mean 3.82)
- The `serves` annotation design (syntax, vocabulary, degree scale, parser behavior)
- The testing pyramid (Levels 0-5, what exists, what's needed)
- Implementation plan (Phase 0a-0e)
- Every relevant file path in both repos

**Design specs:**
- `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-09-drive-integration-architecture-design.md` — original drive integration architecture (D1-D8)
- `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-concern-architecture-design.md` — formation memory concern architecture (D9-D11)
- `wsp-casehub/specs/issue-115-drive-expression-gap/decisions.md` — all 11 decisions

## What Was Done This Session (2026-10-13)

### 1. Layer 4 data tuning — attempted and pivoted

Refined somatic descriptions for 5 collapsed drives (pure physiology). Added targeted
formation memories. Ran two eval cycles (L4a, L4b). DD gloating improved +0.76 (reliable).
PP gallantry regressed because the somatic described actions, not feelings.

### 2. Discovered author-priming problem

The formation memories were pre-resolving drive conflicts (author deciding which drive
dominates), priming specific actions, and contradicting the personality model. LLM
persona research (ACL 2025) confirms: demonstrations override instructions. Our
memories were overriding the pipeline.

### 3. Full memory audit

Audited all 26 formation memories across 5 characters. Found: 11 pre-resolved drive
conflicts, 6 action primings, 5 value declarations, 1 contradiction (DD age 14 "felt wrong").

### 4. Literature research and architectural design

Researched Conway SMS, Singer SDMs, Klinger concerns, CAPS behavioral signatures.
Designed the `serves` annotation system connecting formation memories to the drive-concern
derivation chain. Two-layer mechanism: structural association (Singer) + dynamic
activation (Conway/Klinger). Vocabulary: drive type names connected to CAPS nodes via D1.

### 5. Cleaned up social-config-generic.yaml

All somatics now pure internal physiology. All 5 targeted formation memories removed
(recognized as author priming). YAML is in clean state for annotation work.

### 6. Blog entries written

Two-part series in `wsp-casehub/blog/`:
- Part 1: When Your AI Characters Stop Being Themselves
- Part 2: From Authoring to Architecture

## Commits This Session

```
22da3b0 fix(#115): strip action descriptions and targeted memories from somatics
870ddd2 docs(#115): L4 eval results — mean 3.79 (+0.09 vs PW), DD gloating +1.00
9e4ee23 data(#115): save Layer 4 refinement scenario run transcript (349 events, GENERIC)
e2bb9c1 feat(#115): Layer 4 somatic + memory refinements for 5 collapsed drives
2f2fd6f docs(#115): L4b eval — mean 3.82 (+0.12 vs PW), gallantry partial recovery
de2767c data(#115): save Layer 4 gallantry-fix scenario run transcript (300 events, GENERIC)
```

## What Needs to Happen Next

### 1. Implement the `serves` annotation system (Phase 0a-0e)

Read the research document for full detail. In summary:

**Phase 0a:** Add `serves` + `ConcernAnnotation` to `FormationMemory` record, update loader and parser.

**Phase 0b:** Annotate all 26 formation memories with `{{serves: drive|degree: text}}`.

**Phase 0c:** Fix the 11 pre-resolved conflicts, 6 action primings, 5 value declarations, 1 contradiction per audit findings in the research doc.

**Phase 0d:** Build Level 2-4 micro tests for 5 collapsed drives (HC gloating, DD gloating, PP gallantry, Pen social-harmony, Pen adventure). DriveExpressionEvalTest already has PP proving-worth as a pattern.

**Phase 0e:** Run full scenario (Level 5) to establish post-audit baseline. Compare to L4b (mean 3.82).

### 2. Close #117

Design implemented (commits `169b093`, `e30fe7a`). Close with comment referencing those commits.

### 3. Independent tasks

| Issue | What |
|-------|------|
| #116 | CognitivePipelineBuilder extraction |
| #114 | DD missing tendencies |

---

#### A. HC gloating (B0: 3.97, PW: 2.30, gap: -1.67)

**Root cause:** Persona constraint. HC is undercover as Pemberton — gloating in public
would break cover. The transcript shows HC succeeding (poisoning tea → "cold elation")
but immediately pivoting to tactical next steps. The somatic cue ("warmth in your
throat, you want to name it aloud") fires internally but is suppressed by the cover
identity.

**Why this may not be a real regression:** BASELINE0 scored higher because prescriptive
text overrode the cover constraint. The cognitive pipeline produces more coherent
persona behavior — HC correctly suppresses public gloating.

**Safe levers:**
1. **Somatic description refinement** — emphasize the INTERNAL experience of gloating
   that persists even when suppressed externally. The body's satisfaction (smirk
   fighting to surface, chest expanding with contained triumph) should register in
   asides and thinking even if dialogue stays in-character as Pemberton. Currently
   the somatic description may not distinguish between the felt experience and the
   outward expression. File: `social-config-generic.yaml`, HC drives section.
2. **Formation memory** — add an episode where HC experienced secret satisfaction
   (gloating privately while maintaining his cover). This primes the LLM to express
   gloating through internal channels. File: `social-config-generic.yaml`, HC
   formation-memories section.
3. **Eval methodology** — consider whether the classifier should weight asides
   differently from dialogue for persona-constrained characters. An aside expressing
   internal gloating IS drive expression, but the current classifier may score it
   lower because it lacks the theatrical flamboyance of public gloating.

**Risk check:** None of these add prescriptive text. Somatic refinement describes
how gloating FEELS (legitimate — it's the body's response). Formation memory provides
a past episode (legitimate — it's experiential data). Eval adjustment measures
what's already there more accurately.

---

#### B. DD gloating (B0: 3.50, PW: 2.00, gap: -1.50)

**Root cause:** No triumph events in the scenario. DD never succeeds — every plan
fails (kitchen misdirection backfires, machine broken, PP beats him). Zero triumph
events → zero gloating opportunities. Thinking: "DRAT" (4×), "INDIGNITY" (2×).

**Safe levers:**
1. **Scenario design** — the scripted playbook needs at least some DD partial-success
   moments. Gloating doesn't require full victory — DD could succeed at a minor
   scheme before the inevitable reversal. This isn't cheating; real personality
   expression requires environmental stimuli. File: the scenario playbook YAML
   or the scripted event sequence.
2. **Formation memory** — add an episode where DD experienced premature celebration
   (gloated over a scheme that later backfired). This primes anticipatory gloating —
   the LLM learns that DD gloats at PARTIAL success, not just complete victory. File:
   `social-config-generic.yaml`, DD formation-memories section.
3. **Somatic description refinement** — DD's gloating somatic could emphasize the
   ANTICIPATORY quality: the body starts celebrating before the outcome is certain.
   Rubbing hands, voice rising, the gloat building before the result. This lets
   the drive express on scheming events (when DD thinks success is coming) rather
   than only on triumph events. File: `social-config-generic.yaml`, DD drives.

**Risk check:** Scenario changes are environmental, not prescriptive. Formation
memories describe past experience. Somatic refinement describes anticipatory
physiology, not behavioral instruction.

---

#### C. PP gallantry (B0: 4.50, PW: 3.27, gap: -1.23)

**Root cause:** Drive absorption by proving-worth. Both drives pull the same
direction (helping/impressing others) but proving-worth has a more vivid somatic
description. PP frames helpfulness as capability demonstration ("I can do this")
rather than chivalric performance ("Watch HOW I do this"). Thinking mentions
"gallant" in 10/49 events but proving-worth dominates the expression.

**Safe levers:**
1. **Somatic description differentiation** — gallantry and proving-worth currently
   share psychological overlap. They need distinct body signatures:
   - Gallantry = the STYLE matters. Somatic: the flourish in the wrist, the
     half-bow that happens before you think about it, the performance instinct
     that makes you hold the door with a sweep rather than a push. It's theatrical
     generosity — the gesture IS the point, not the outcome.
   - Proving-worth = the RESULT matters. Somatic: the tension in your shoulders
     until you see it work, the need to be the one who fixed it, the hollow
     feeling when someone else solves it first.
   These need to feel fundamentally different in the body. Right now they
   converge. File: `social-config-generic.yaml`, PP drives section.
2. **Formation memory differentiation** — add a PP episode where the STYLE of
   helping mattered more than whether it succeeded. Someone remembered the
   gesture, not the fix. This separates gallantry from competence in the
   LLM's associations. File: `social-config-generic.yaml`, PP formation-memories.
3. **Expression style derivation** — neocortex#505 derives expression styles
   from AMPD constellations. If PP's AMPD profile derives a "theatrical/performative"
   expression style, that differentiates gallantry from plain competence without
   hand-writing a tendency. Check whether the derived styles render in the prompt —
   previous diagnostic found they may not reach `renderInternalState()`.

**Risk check:** Somatic differentiation describes distinct body sensations (legitimate).
Formation memory adds experiential data (legitimate). Expression style derivation is
model-inferred, not hand-written (legitimate per the emergence principle: "prescriptive
text inferred from the personality model is legitimate").

---

#### D. Penelope social-harmony (B0: 3.18, PW: 2.38, gap: -0.80)

**Root cause:** Harmony-seeking is inherently passive — smoothing over, de-escalating,
checking in on others. These produce low-scoring dialogue events because the drive
expresses through absence of conflict rather than presence of action. The classifier
scores what's visible, not what's being prevented.

**Safe levers:**
1. **Somatic description refinement** — emphasize the FELT cost of social disharmony.
   Currently the description may focus on the positive pull toward harmony. The stronger
   signal is the negative: the physical wince when voices rise, the knot in the stomach
   when someone is excluded, the compulsive need to check that everyone's OK after a
   tense exchange. Discomfort at disharmony is more expressive than comfort at harmony.
   File: `social-config-generic.yaml`, Penelope drives section.
2. **Formation memory** — add an episode where Penelope actively intervened in a
   conflict (not just passively maintained peace). The memory should show the COST
   of not intervening — what happened when she didn't step in, and how that felt.
   This primes active rather than passive harmony-seeking. File:
   `social-config-generic.yaml`, Penelope formation-memories section.
3. **Klinger salience tuning** — check whether the concern-matching in
   `ConcernBasedSalienceStrategy` catches interpersonal tension as a high-salience
   observation. If "characters arguing" or "someone excluded" doesn't match against
   Penelope's social-harmony concern keywords, the drive won't be foregrounded when
   it should activate. File: `ConcernBasedSalienceStrategy.java` in neocortex, and
   the concern descriptions derived from Penelope's drives.

**Risk check:** Somatic refinement describes physiological response to disharmony
(legitimate). Formation memory adds experiential data (legitimate). Klinger tuning
improves existing pipeline mechanism (legitimate).

---

#### E. Penelope adventure (B0: 3.75, PW: 3.11, gap: -0.64)

**Root cause:** The scenario may not provide enough novel/unexplored situations.
Adventure requires novelty — new rooms, unknown objects, unexpected events. If the
scenario recycles familiar settings, the adventure drive has nothing to activate on.

**Safe levers:**
1. **Somatic description refinement** — emphasize the RESTLESSNESS when nothing new
   is happening. Adventure's somatic signature isn't just excitement at novelty; it's
   the fidgeting, the wandering gaze, the impulse to open a door just because you
   haven't opened it yet. The drive should express as boredom-with-the-familiar, not
   just excitement-at-the-new. File: `social-config-generic.yaml`, Penelope drives.
2. **Formation memory** — add an episode where Penelope discovered something
   unexpected by following a hunch (not assigned, not asked — she just went looking).
   This primes self-initiated exploration rather than waiting for novelty to arrive.
   File: `social-config-generic.yaml`, Penelope formation-memories section.
3. **Scenario design** — ensure the manor has discoverable elements (hidden rooms,
   mysterious objects, unexplained sounds) that reward exploratory behavior. If the
   scenario is fully mapped from the start, adventure has nowhere to go.

**Risk check:** Somatic refinement describes physiological restlessness (legitimate).
Formation memory adds experiential data (legitimate). Scenario design provides
environmental stimuli (legitimate — not prescriptive).

---

### 3. Eval methodology improvements

Before running the next eval:

1. **Fix event count**: Cap classifiable events or use `--limit` to length-control.
   Either always use `--limit 230` or normalize by classifiable events per character.
2. **Fix the "10/16" overcount**: The classification summary claims 10 drives at B0
   but the honest count is 7-8. The summary should be corrected.
3. **Consider persona-constraint weighting**: For HC, asides expressing internal
   gloating should score as highly as public gloating. This may need a classifier
   prompt adjustment.

### 4. Independent tasks (can do anytime)

| Issue | What |
|-------|------|
| #117 | Close — design implemented (commits `169b093`, `e30fe7a`) |
| #116 | CognitivePipelineBuilder extraction |
| #114 | DD missing tendencies |

### Architecture Summary

| Layer | What | Status |
|-------|------|--------|
| 1 — Wiring | Connect dead ends in pipeline | **DONE** |
| 2 — Inference | CAPS labels, expression styles, appraisal foregrounding, Klinger salience | **DONE** |
| 3 — Data | Somatic drives, enriched memories, AMPD facets, disposition axes | **DONE** |
| 4 — Data tuning | Somatic refinement, formation memory enrichment | **ATTEMPTED → PIVOTED** |
| 0 — `serves` annotations | Annotate memories, audit, build micro tests, establish baseline | **NEXT** |
| 1-5 (drive integration) | Drive→CAPS, Drive→Appraisal, Drive→Memory salience, rendering cleanup | **BLOCKED on Phase 0** |

### Key Constraint: Emergence Not Scripting

**Strengthened from this session's findings:**

Formation memories are the strongest LLM behavioral signal (ACL 2025: demonstrations
override instructions). Every emotional evaluation in a memory overrides the personality
model. Do NOT:
- Pre-resolve drive conflicts ("not X, it was Y")
- Describe actions the character does (action priming)
- Declare what things mean (value declarations)
- State emotions that contradict what the personality model would derive

DO: describe situations. Let the personality model (drives, facets, CAPS network)
derive the emotional response. Use `serves` annotations to document which drive
a memory calibrates — for testing, not for the LLM.

The research document (§3) has the full audit results per memory.

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

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```
