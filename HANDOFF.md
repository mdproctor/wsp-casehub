# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** All three layers (A + B + C) complete. Full scenario validation run next.
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

## START HERE

**Operational note (progress, direction, what not to re-derive):**
`wsp-casehub/examples/notes/issue-115-drive-expression-gap.md`

**Research document (deep technical reference, literature, methodology):**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

**How to read it:**
- **§0** — current state, what's done, what's settled (do not re-derive), what's next
- **§10** — the root cause investigation: why drives collapse, the full derivation chain, all 7 breaks
- **§11** — Layer A results: what was implemented, empirical findings, topology audit, literature grounding
- **§12** — implementation plan with status: what's done (1a-1c), what's next (2a-3b)

The rest (§1-§9, §13) is supporting evidence — read on demand when §0/§10-§12 reference it.

**This document is the single source of truth for the #115 effort.** It contains all literature citations (with specific papers), all empirical data (eval scores, test results), all architectural decisions (D1-D11 in `decisions.md`), and all implementation findings. Do not start work without reading §0. Do not re-derive settled decisions. Do not guess at psychological models — the literature is already researched and cited.

**Design specs (supplementary, read if §0 references them):**
- `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-09-drive-integration-architecture-design.md` — drive integration architecture (D1-D8)
- `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-concern-architecture-design.md` — formation memory concern architecture (D9-D11)
- `wsp-casehub/specs/issue-115-drive-expression-gap/decisions.md` — all 11 decisions

## What Was Done This Session (2026-10-13, Layer A session)

### Layer A — Weighted Situation Vocabulary (neocortex)

**Problem (§10.1):** The CAPS network's `RuleBasedSituationClassifier` matched event text against INPUT node keywords, but all keywords had equal weight. "Humiliated" activated `dominance` at the same strength as "suggested." Worse, 17 of 31 INPUT nodes had zero outgoing connections — activations died at the input layer. All 5 priority nodes for collapsed drives (dominance, failure, competence_recognition, exclusion, competition) were disconnected.

**What was implemented:**
1. `CapsNode.keywords` (`List<String>`) → `CapsNode.vocabulary` (`Map<String, Double>`) — weighted terms
2. `RuleBasedSituationClassifier` — confidence = max matched weight + reinforcement (0.05/extra match, capped 0.15)
3. `CapsTopologyLoader` — parses `vocabulary:` maps, backward-compat for `keywords:` lists
4. `caps-topology.yaml` — all 31 INPUT nodes converted to weighted vocabulary derived from construct literature (Bowlby, Gray RST, McClelland, Baumeister & Leary, Deci & Ryan, Bandura, Williams)
5. 30 new INPUT→MEDIATING connections added for all 17 disconnected nodes
6. Topology validation test: all INPUT nodes must have connections (regression guard)
7. 3 scenario-realistic classifier tests (rival failure, helping others, competition + recognition)

**Empirical findings:**
- All 33 caps-engine tests pass, 581 wacky-manor tests pass, 0 regressions
- Inflection sensitivity: vocabulary must use morphological forms from event text ("humiliated" not "humiliate") — exact token matching
- Strong terms (0.7-0.9) produce high confidence on single match; faint terms (0.2-0.3) need corroboration
- Full details in research doc §11

**Neocortex branch:** `issue-115-weighted-situation-vocabulary` (2 commits: `989647c6`, `bb4147dd`)

### Cross-Repo Changes

Neocortex changes are on branch `issue-115-weighted-situation-vocabulary`. Neocortex workspace HANDOFF updated at `wsp-casehub-neocortex/HANDOFF.md`.

### Layer B — Memory Calibration (2026-10-14 session)

**Step 2a — Map formation memories to catalogue entries:**
1. Developed mapping methodology (§12.4): three tests (directional correctness, cumulative trajectory, no contradictions), OUTPUT-vs-MEDIATING rule
2. Mapped all 26 formation memories to catalogue entries (§12.5)
3. Added `catalogue` field to `FormationMemory` record with backward-compatible default
4. Updated `ManorCognitiveSeeder` to use `mem.catalogue()` and enrich `situationTypes` with drive annotations
5. Commit: `f3c9673`

**Step 2b — Render mediating substrate:**
1. Added `substrate` field to `SocialConfig`, parsed from `psychological-substrate` in YAML
2. Added "What Shaped You" section to `CharacterCognition.renderCognitiveSections()` — rendered between formation memories and relational schemas
3. Wrote felt-experience substrate descriptions for all 5 characters (no behavioral prescriptions)
4. Commit: `23eb356`

**Key methodology insight:** Always prefer catalogue entries that calibrate MEDIATING nodes over OUTPUT-only entries. Many Bandura/operant entries (observational-learning, model-imitation, positive-reinforcement) only move OUTPUT nodes and can't fix Break #4.

### Previous Session Work (preserved for context)

Phase 0 (`serves` annotations), Layer 4 data tuning, memory audit, literature research, blog entries — all documented in the research doc §7-§8 and prior HANDOFF entries.

## What Needs to Happen Next

### 1. Layer B — Memory Calibration — DONE

Steps 2a-2b complete. See research doc §12.4 (methodology) and §12.5 (mappings). Commits: `f3c9673` (step 2a), `23eb356` (step 2b).

### 2. Layer C — Drive Amplification — DONE

D1 implemented in neocortex `BehavioralSynthesisPhase.mergeDriveIntensities()`. 14 drive types → explicit CAPS node mappings. Commit: `11e64632`.

### 3. Full Scenario Validation — NEXT

All three layers complete. All 7 derivation chain breaks fixed. Next:
1. Install neocortex with Layers A+C (`mvn install -Dmaven.test.skip=true`)
2. Rebuild wacky-manor
3. Run full GENERIC scenario (~300 events)
4. Classify with `classify_emotions.py`, compare to P0 (3.72) and B0 (3.96)

### 3. Layer A validation (§11.7, §12.1 step 1d)

Run a full scenario with the vocabulary active and compare CAPS attractors before/after. This validates that real scenario events produce content-specific activations through the new weighted vocabulary.

### 4. Independent tasks

| Issue | What |
|-------|------|
| #117 | Close — design implemented (commits `169b093`, `e30fe7a`) |
| #116 | CognitivePipelineBuilder extraction |
| #114 | DD missing tendencies |

## Architecture Summary

| Layer | What | Status |
|-------|------|--------|
| 1 — Wiring | Connect dead ends in pipeline | **DONE** |
| 2 — Inference | CAPS labels, expression styles, appraisal foregrounding, Klinger salience | **DONE** |
| 3 — Data | Somatic drives, enriched memories, AMPD facets, disposition axes | **DONE** |
| 4 — Data tuning | Somatic refinement, formation memory enrichment | **ATTEMPTED → PIVOTED** |
| 0 — `serves` annotations | Annotate memories, audit, build micro tests, establish baseline | **DONE** |
| A — Situation vocabulary | Weighted vocabulary for CAPS INPUT nodes, missing connections | **DONE** (neocortex `issue-115-weighted-situation-vocabulary`) |
| B — Memory calibration | Connect memories to catalogue entries, calibrate mediating nodes | **DONE** (examples `issue-115-drive-expression-gap`) |
| C — Drive amplification | D1 drive→CAPS activation injection | **DONE** (neocortex `11e64632`) |

### The Derivation Chain (§10.4, §11.8)

The full chain from memory to emergent behaviour, with break status:

```
Formation Memories (serves: recognition|strong)
  → [catalogue entry] calibrates MEDIATING nodes            ← Break #4: FIXED (Layer B)
  → [situation occurs] INPUT nodes fire via vocabulary       ← Break #1: FIXED (Layer A)
  → [propagation] INPUT→MEDIATING connections exist          ← Break #3: FIXED (Layer A)
  → [drive activation] persistent drive bias co-activates   ← Break #7: FIXED (Layer C)
  → [settling] OUTPUT attractors emerge
  → [rendering] LLM receives attractor + memory salience → behaviour
```

### Key Constraint: Emergence Not Scripting

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

## Per-Drive Root Cause Analysis

These analyses remain valid. Layer A addresses the pipeline gaps identified here;
Layers B and C address the remaining substrate and amplification gaps.

### A. HC gloating (B0: 3.97, P0-avg: 2.56, gap: -1.41)

**Root cause:** Persona constraint (HC as Pemberton suppresses public gloating) AND
situation-routing gap (now partially fixed by Layer A — dominance/failure nodes
are connected). Remaining gap: mediating substrate not calibrated (Layer B) and
drive amplification not implemented (Layer C).

### B. DD gloating (B0: 3.50, P0-avg: 2.78, gap: -0.72)

**Root cause:** Situation-routing gap (fixed by Layer A) AND missing mediating substrate
(DD's depressed self_worth not calibrated from formation memories). The gloating
pathway (§11.8) is now connected but not calibrated.

### C. PP gallantry (B0: 4.50, P0-avg: 3.56, gap: -0.94)

**Root cause:** Drive absorption by proving-worth. Layer A connects competence_recognition
to self_worth and BAS, which helps differentiate the reward signal. Layer B
(mediating substrate) should further differentiate by calibrating PP's self_efficacy
as externally-validated (worth = what you do for others).

### D. Pen social-harmony (B0: 3.18, P0-avg: 2.66, gap: -0.52)

**Root cause:** Passive drive expression AND situation-routing gap. Layer A connects
exclusion to other_safety and BIS — conflict/exclusion events now propagate.
Layer B should calibrate Pen's other_safety sensitivity.

### E. Pen adventure (B0: 3.75, P0-avg: 3.37, gap: -0.38)

**Root cause:** Insufficient novelty in scenario AND weak situation routing.
Smallest gap — may close with Layer A vocabulary alone (reward/competition
nodes now connected). Needs scenario validation (step 1d).

## Eval Baselines

| Metric | Value | Source |
|---|---|---|
| B0 (BASELINE0 prescriptive) mean | 3.96 | `classify_emotions.py` on old-briefings run |
| P0 (Phase 0 post-audit) mean | 3.72 | Two-run average (§8.2) |
| L4b (Layer 4 somatic fix) mean | 3.82 | Single run |

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

**Neocortex must be on `issue-115-weighted-situation-vocabulary` branch** for Layer A vocabulary changes. Install with `mvn install -Dmaven.test.skip=true` before building wacky-manor.
