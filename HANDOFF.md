# HANDOFF — casehub-examples

## Last Session

Fixed the drive discrimination pipeline end-to-end and discovered the
reactive-drive problem is about appraisal-based filtering, not
description quality or rendering order.

### What was done

**Pipeline fixes (neocortex #584, #585, #586):**
- DriveProfileDerivationPhase graceful degradation without CAPS deps
- Cross-phase pipeline verification (Phase 1 → Phase 2)
- Multi-channel prompts (Barrett 4-channel constructed emotion)
- Ambient+acute generation prompt (to be superseded by appraisal filtering)
- Pipeline observability logging (per-phase timing, per-agent progress)

**Rendering fixes (examples #122):**
- Drives render by intensity regardless of activation status
- Formation memories render by salience (|pleasure| + arousal), not age
- Norms already rendered by priority (verified with test)
- All ordering rules unit tested

**Key finding — the reactive-drive problem:**
- Reactive drives (gallantry, gloating, adventure) score 2.3-2.8 in full scenarios
- Proactive drives (scheming, curiosity, proving-worth) score 4.0-5.0
- Micro tests score 4/5 — descriptions are good enough
- Root cause: drive descriptions rendered in EVERY event regardless of situational relevance. PP alone in the library reads gallantry text about "the person exposed" — irrelevant, LLM ignores it, scores 1
- Ambient coloring was attempted (prompt asks for background+acute) but is the wrong fix
- **Correct fix: appraisal-based drive filtering** — only render drives whose profile inputs overlap with the current situation's classifier activations

### Next Session: Appraisal-Based Drive Filtering

**The architecture already exists.** `selectSituationPrimedMemories` in
CharacterCognition does exactly this for memories:
1. SituationClassifier classifies current situation → CAPS input activations
2. DriveProfileInferred links drives to CAPS inputs
3. Overlap determines relevance

Apply the same pattern to drive rendering:
1. Classify the situation text
2. Check which drive profiles overlap with situation activations
3. Only render overlapping drives in "What Pulls at You"

**Before implementing:** revert the ambient+acute prompt change (commit
`cf3b0fa3` on neocortex). Appraisal filtering makes ambient coloring
unnecessary — when a drive IS rendered, the acute multi-channel
description is correct. When it's NOT rendered, no description is better
than an irrelevant one.

**TDD approach:**
1. RED: test that gallantry is NOT rendered when PP is alone (no social inputs in situation)
2. RED: test that gallantry IS rendered when Clara is present (acceptance/sharing inputs match)
3. GREEN: implement appraisal filtering in `renderInternalState`
4. Verify: run PP gallantry micro tests, then fresh 300-event scenario

**Meta-rule (established this session):** within any multi-item
observation section, items render in strength/priority/salience order.
Unit tested for drives, norms, formation memories.

## Branch State

- `issue-115-drive-expression-gap` (examples) — active, 26 commits ahead of main
- `issue-115-weighted-situation-vocabulary` (neocortex) — active, 6 commits this session
- Paused branch stack: `issue-115-drive-expression-gap`

## Neocortex Commits This Session

All on branch `issue-115-weighted-situation-vocabulary`:
- `33235928` fix(#584): DriveProfileDerivationPhase works without SituationClassifier/CapsTopology
- `9c7ddea6` fix(#585,#586): multi-channel drive discrimination + cross-phase pipeline verification
- `dd65ac70` wip: pipeline observability logging — phase timing
- `8b519bbb` wip: pipeline observability — per-agent profile/competition/description logging
- `cf3b0fa3` fix(#585): ambient+acute prompt — **revert before implementing appraisal filtering**

## Examples Commits This Session

On branch `issue-115-drive-expression-gap`:
- `51f784f` data(#122): multichannel pipeline run — 900 events, cognitive dumps
- `d55c3ae` fix(#122): render drives in strength order, not activation order
- `4049c2c` docs(#122): multichannel pipeline run results
- `0fd3be1` fix(#122): formation memories render by salience, not age

## Eval Data

- `wacky-manor/docs/eval/multichannel-pipeline-2026-10-14/` — 900-event run, cognitive dumps, RESULTS.md
- Classification @300 events: mean 3.68 (vs B0 3.96)
- PP gallantry: 2.50 @300 events, 4/5 in micro test
