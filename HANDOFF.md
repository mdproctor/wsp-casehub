# HANDOFF — casehub-examples

## Last Session

Fixed the drive discrimination pipeline end-to-end (neocortex #584, #585, #586) and established the strength-order rendering meta-rule (examples #122). Ran a full 900-event GENERIC scenario with pipeline-generated multi-channel descriptions. Results written up in `wacky-manor/docs/eval/multichannel-pipeline-2026-10-14/RESULTS.md`.

### What was done

1. **neocortex #584**: DriveProfileDerivationPhase graceful degradation — works without SituationClassifier/CapsTopology via catalogue-entry overlap
2. **neocortex #586**: Cross-phase pipeline verification — Phase 1 → Phase 2 data flow confirmed end-to-end
3. **neocortex #585**: Multi-channel prompts — Barrett (2017) 4-channel constructed emotion replaces somatic-only
4. **Pipeline observability**: per-agent logging for profile derivation, competition detection, description generation timing
5. **Strength-order rendering**: drives render by intensity regardless of activation status
6. **Full scenario run**: 900 events, cognitive dumps for 5 primary characters, classified at 300 and 900 event windows

### Key findings

- **Multi-channel descriptions work**: gallantry and proving-worth have genuinely different vocabulary across somatic, attentional, temporal, and action tendency channels
- **Micro test 4/5**: PP gallantry door-stuck scores well in isolation (up from 2/5 pre-pipeline)
- **Full scenario 2.50**: gap is not description quality but reactive drive opportunity — gallantry needs someone in trouble, but most events are proving-worth territory
- **Score decay past 300**: all drives decay without a termination signal. Needs boredom/satiation mechanism
- **Below-par pattern**: gallantry (2.50), gloating (2.37, 2.85), adventure (2.84) are all reactive drives. Proactive drives (scheming 5.0, curiosity 4.45) score well

### Meta-rule: strength-ordered rendering

Within any multi-item section of the cognitive observation, items must appear in strength/priority order. LLM primacy bias means position matters. Currently applied to drives. Should extend to beliefs, social rules.

## Branch State

- `issue-115-drive-expression-gap` (examples) — active, 24 commits ahead of main
- `issue-115-weighted-situation-vocabulary` (neocortex) — active, 4 new commits this session
- Paused branch stack has this examples branch

## Neocortex Commits This Session

All on branch `issue-115-weighted-situation-vocabulary`:
- `33235928` fix(#584): DriveProfileDerivationPhase works without SituationClassifier/CapsTopology
- `9c7ddea6` fix(#585,#586): multi-channel drive discrimination + cross-phase pipeline verification
- `dd65ac70` wip: add pipeline observability logging — phase timing + per-agent progress
- `8b519bbb` wip: pipeline observability — per-agent profile/competition/description logging

## Immediate Next Steps

1. **Strength-order all sections** — apply meta-rule to beliefs and social rules
2. **Re-run with strength-order fix** — need fresh 300-event comparison (fix was applied after this run)
3. **Scenario termination signal** — boredom/satiation from neocortex to stop at ~300 events
4. **Reactive drive scenarios** — ensure scripted playbook includes enough trigger situations for gallantry, gloating, adventure

## Baseline Transcripts — DO NOT DELETE

See eval directory for all baseline transcripts. Key comparison:
- B0 baseline: mean 3.96
- This run @300: mean 3.68
- This run @900: mean 3.14 (decay)
