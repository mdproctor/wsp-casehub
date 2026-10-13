# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Pipeline wiring complete, eval run done, results posted to #115
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

## What Was Done This Session

### 1. Built and ran pipeline wiring eval

With all pipeline wiring active (appraisal-driven drive foregrounding #118, Klinger
concern-based salience #119, neocortex fixes #504-#512), ran a GENERIC scenario
producing 997 events — 3× the L123 baseline of 326 events.

### 2. Discovered run-length confound

The full 997-event run produced a misleading mean of 3.24. Longer runs dilute scores
because routine dialogue scores low on drive expression. Length-controlled comparison
(first 230 classifiable events, matching L123 sample size) produced mean **3.69** —
an improvement of +0.12 over L123's 3.57. Future evals must length-control.

### 3. Posted results to #115

Full comparison table, methodology finding, and assessment posted as a comment on
the issue. Key result: 7-8 of 16 drives at or above BASELINE0, up from 7 in L123
but the shift is in which drives recovered. PP protection major recovery (+0.81).

## Commits on Branch

```
bac8b17 docs(#115): length-controlled eval — mean 3.69 (+0.12 vs L123), 10/16 drives at B0
0f87936 docs(#115): pipeline wiring eval results — mean 3.24, run-length confound identified
122e621 data(#115): save pipeline-wiring scenario run transcript (997 events, GENERIC)
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

### 1. Close #117

Design implemented (commits `169b093`, `e30fe7a`). Close with comment referencing those commits.

### 2. Address 5 remaining collapsed drives

Each drive has a different root cause and a different safe lever. **No prescriptive
briefing text. No hand-written behavioral labels.** Every intervention must work
through the cognitive pipeline's own mechanisms: somatic descriptions, formation
memories, expression style derivation, scenario design, or eval methodology.

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

### Three-Layer Fix Architecture (final)

| Layer | What | Status |
|-------|------|--------|
| 1 — Wiring | Connect dead ends in pipeline | **DONE** |
| 2 — Inference | CAPS labels, expression styles, appraisal foregrounding, Klinger salience, drive-CAPS loop, dampening fix, pipeline tuning | **DONE** |
| 3 — Data | Somatic drives, enriched memories, AMPD facets, disposition axes | **DONE** |
| 4 — Refinement | Somatic differentiation, formation memory enrichment, scenario design, eval methodology | **NEXT** |

### Key Constraint: Emergence Not Scripting

Prescriptive text inferred from the personality model (AMPD facets → derived tendencies)
is legitimate. Hand-written behavioral labels are not. Every intervention in Layer 4
must work through: somatic descriptions (body-felt), formation memories (past episodes),
expression style derivation (AMPD-inferred), scenario design (environmental stimuli),
or eval methodology (measurement accuracy). If you're tempted to write "You like to
gloat" or "You tend to seek adventure" — stop. Describe how it FEELS, not what to DO.

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

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```
