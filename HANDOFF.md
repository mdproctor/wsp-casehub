# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-28
**Branch:** main (examples), issue-714-barrett-personality-profiles (neocortex)

## What was done this session

### Barrett 4-Channel Architecture (replacing prose templating)

Discovered that `social-config-generic.yaml` had accumulated ~640 lines of hand-crafted prose briefing injected verbatim into the LLM prompt. This bypasses the cognitive architecture. Replaced with:

1. **Barrett 4-channel instruction** (V2) — universal framework for computed drive expression. Tested at 4.7 mean in isolated harness (somatic 5.0, situational 4.9, bespoke 4.5, discovery 4.5).

2. **Briefing-mode toggle** — `manor.cognitive.briefing-mode=false` in application.properties. Wired through ManorConfigProducer → ManorConfig → PlaybookOrchestrator → CharacterCognition. Default `true` preserves current behavior.

3. **Consolidation phase updated** (neocortex#714) — `DriveDescriptionGenerationPhase` now reads personality facets + attachment from MindMap nodes for personality-aware Barrett profile derivation.

4. **Biography engram YAML** — 25 files across 5 characters (per-character directories under `META-INF/eidos/biographies/`), fully enriched: 13 AMPD facets, Jungian stack, DISC, Belbin, TK, SDI, Locus, archetypes, disposition, life events with sub_thoughts, beliefs, relationships, goals. Audited and corrected.

5. **Biography import** wired into PlaybookOrchestrator — per-character directory loading via `BiographyImportRunner`.

6. **Barrett test harness** — `wacky-manor/docs/eval/test_barrett_generation.py` with 10 test cases, LLM judge (bespoke/somatic/situational/discovery), A/B version support, `--profile` mode for consolidated profiles.

### First Computed Run Results

30-event eval, 5 characters, `briefing-mode=false`:

| Change | Drives improved | Drives regressed |
|---|---|---|
| **Quality** | AHM prot +0.50, AHM susp +0.50, DD gloat +0.60, PP prov-worth +1.10 | HC gloat -1.50, PP prot -1.77, PP gall -0.67 |
| **Register** | AHM prot +0.63, AHM susp +0.70, DD scheme +0.67 | PP gall -2.20, PP prot -1.70, HC gloat -2.70 |

Mixed results. Root causes identified and filed as issues.

## Issues filed this session

| # | Title | Priority | Status |
|---|-------|----------|--------|
| #156 | Improve five weak drive expressions | Active | Branch exists with toggle + Barrett instruction |
| #158 | Epic: migrate characters to biography engrams | Active | Engrams written, import wired |
| #159-#163 | Per-character engram migration | Done | Engram YAML created for all 5 |
| #164 | GENERIC profile agentId leak | **High** | Filed, not started |
| #165 | Ungate formation memories in computed mode | **High** | Filed, not started |
| #166 | Discrimination text personality context | Medium | Filed, not started |

## Immediate Next Steps (for autonomous continuation)

Work in short iterative cycles (~10 min each). Fix the obvious gaps first, eval after each change:

### 1. Fix #165 — Ungate formation memories (HIGHEST IMPACT)
In `CharacterCognition.renderCognitiveSections()`, let formation memories render in BOTH modes. They're biographical data, not prose templates. This is likely causing Peter's gallantry/protection register drops (3.40→1.20, 4.00→2.30).

### 2. Fix #164 — GENERIC profile agentId leak
The LLM sees `penelope-pitstop`, `peter-perfect` as agentIds and hallucinates cartoon references ("Muttley", "Peter Perfect"). Fix by ensuring display names are used in all observation text.

### 3. Rerun with both fixes + GENERIC profile
30-event eval, compare against V2 baseline. This should close the gap on the regressed drives.

### 4. If regressions persist, iterate
- Check consolidated Barrett profiles (`/manor/cognitive-dump/{character}`) — are they bespoke enough?
- Check if the "How to Feel" instruction needs tuning for specific drives
- Run Barrett test harness (`--profile`) to test generation quality in isolation

### 5. When short run looks good, do a full 300-tick run
- All 5 characters, GENERIC profile, `briefing-mode=false`
- Save transcript, run full eval
- Compare against V2 baseline (30-event data in `docs/eval/full-cognitive-run-2026-10-26/`)
- If the computed pipeline matches or exceeds the prose baseline across all drives, the templating can be removed

## Key References

- Barrett instruction V2: `CharacterCognition.java` lines 208-244 (in `!briefingMode` block)
- Barrett test harness: `wacky-manor/docs/eval/test_barrett_generation.py`
- Engrams: `wacky-manor/src/main/resources/META-INF/eidos/biographies/{character}/`
- V2 baseline eval: `wacky-manor/docs/eval/full-cognitive-run-2026-10-26/full-eval-v2.json`
- Computed run eval: `wacky-manor/docs/eval/computed-engram-run-20261028/full-eval.json`
- Era 4 journal: `neocortex/docs/research/era4-measurement-methodology-research.md` (Entry 3)
- Consolidation phase: `neocortex/mindmap-intelligence/.../DriveDescriptionGenerationPhase.java`
- Toggle: `-Dmanor.cognitive.briefing-mode=false`

## Constraint: NO TEMPLATING

Do NOT add hand-crafted prose to:
- Drive descriptions in social-config YAML
- Personality facet origins
- Formation memory episode text (as somatic scripting)
- Psychological substrate paragraphs
- Any text that gets injected verbatim into the LLM prompt

The Barrett instruction + consolidation-derived profiles + structured engram data should produce all drive expression content. If quality is low, improve the instruction or the consolidation prompt — do not fall back to hand-crafted descriptions.

## Repo State

- **examples:** main, needs push. `issue-156-weak-drive-improvements` branch has toggle + Barrett instruction (needs merge to main).
- **neocortex:** `issue-714-barrett-personality-profiles` branch, needs push. Era 4 Entry 3 committed.
- **Server:** wacky-manor may be running on port 8180. Check before starting.
