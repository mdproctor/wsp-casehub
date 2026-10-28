# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-28
**Branch:** issue-156-computed-mode-quality (examples)

## What was done this session

### Phase 1 — Quick fixes (#164, #165) — COMPLETE

1. **#165 Ungate formation memories** — removed `briefingMode` guard from formation memory rendering in `CharacterCognition.renderCognitiveSections()`. Formation memories are biographical data, not prose templates — they render in both modes now.

2. **#164 GENERIC agentId leak** — 6 leak points fixed:
   - Entity IDs: slugified display names (`clara-bellingham`) instead of cartoon agentIds
   - `WorldState.resolveCharacter(ref)`: tries direct agentId → slug-matched display name
   - `ActionResolver`, `PlaybookOrchestrator` response parsing use resolved names
   - `NarrativeEventBuilder.describeRich/describeDirectedDialogue` use display names
   - Descriptor names applied to ALL characters at startup (not just active ones)
   - Non-active characters marked inactive → don't appear in room observations
   - **Result: zero cartoon name leaks** in eval runs

### Phase 2 — Baseline comparison — COMPLETE

Cross-baseline scorecard (B0, B1, V2, Pre-fix, Fixed):

| Baseline | Quality | Register | Notes |
|----------|---------|----------|-------|
| B0 (Oct 2) | 3.64 | 1.52 | Original generic colloquial |
| B1 (Oct 4) | 3.95 | 2.72 | Emotional core baseline |
| V2 (Oct 26) | 3.88 | 3.35 | Full cognitive run (briefing mode) |
| Pre-fix computed | 3.78 | 2.94 | Computed mode before fixes |
| **Fixed computed** | **3.76** | **3.61** | **After #164/#165 — register surpasses V2** |
| Barrett V3 | 3.70 | 3.47 | Instruction change regressed — reverted |

### Phase 3 — Barrett V3 iteration — TESTED, REVERTED

Barrett V3 ("FROM BODY TO ACTION" bridge replacing "Do not conclude. Do not explain. Discover.") was tested and reverted. It reduced register (3.61→3.47) and didn't improve quality on target drives. The eval's small N per drive (3-6) makes single-run differences unreliable.

### Key Fix1 improvements vs pre-fix computed (the shipped configuration):

| Drive | Metric | Pre-fix | Fixed | Δ |
|-------|--------|---------|-------|---|
| Peter gallantry | register | 1.17 | 3.50 | **+2.33** |
| HC gloating | register | 1.75 | 3.17 | **+1.42** |
| PP social-harmony | register | 2.67 | 4.00 | **+1.33** |
| DD recognition | register | 1.00 | 2.20 | **+1.20** |
| Peter protection | register | 2.33 | 3.50 | **+1.17** |
| Peter proving-worth | register | 2.33 | 3.50 | **+1.17** |

### Commits on branch

| SHA | Description |
|-----|-------------|
| 0804d893 | fix(#164,#165): ungate formation memories + fix GENERIC agentId leak |
| 173d2d24 | fix(#164): apply descriptor name to CharacterState at startup |
| 58e6521a | fix(#164): mark non-active characters inactive + apply name overrides |
| 28d464f3 | data(#156): cross-baseline scorecard |
| 57451552 | data(#156): computed-fixes eval — register 2.94→3.61 |
| 2edb0799 | feat(#156): Barrett V3 (reverted) |
| df7bf48d | data(#156): Barrett V3 eval data |
| 2979519a | Revert Barrett V3 |

## Remaining quality gaps

| Drive | V2 quality | Fixed quality | Gap |
|-------|-----------|---------------|-----|
| HC gloating | 4.00 | 2.50 | -1.50 |
| Peter gallantry | 4.00 | 3.00 | -1.00 |
| Peter protection | 4.60 | 3.33 | -1.27 |

These are QUALITY gaps (not register). The consolidated drive descriptions are good (rich Barrett-style somatic profiles) but the character doesn't translate somatic awareness into explicitly evaluable behavior. This might resolve with larger N in a 300-tick run, or might need targeted improvements to the consolidation prompt for these specific drives.

## Next Steps (for continuation)

1. **Full 300-tick run** — use Fix1 config (Barrett V2 instruction, formation memories ungated, GENERIC names). Start with: `-Dmanor.cognitive.briefing-mode=false -Dmanor.scenario.profile=generic -Dmanor.scenario.mode=autonomous -Dmanor.scenario.max-turns=300 -Dmanor.scenario.active-characters=penelope-pitstop,hooded-claw,ant-hill-mob,dick-dastardly,peter-perfect`
2. **If quality gaps persist at scale** — investigate consolidated drive profiles for HC gloating and Peter gallantry/protection in `DriveDescriptionGenerationPhase` (neocortex). The somatic descriptions are rich but might not translate to evaluable behavior.
3. **Branch closure** — when 300-tick results are satisfactory, squash and merge to main via work-end.

## Key References

- Barrett instruction: `CharacterCognition.java` lines 208-242 (in `!briefingMode` block)
- WorldState.resolveCharacter: `WorldState.java` line 39
- Entity ID slug: `ManorWorldObservationProvider.java` line 106
- Scorecard: `wacky-manor/docs/eval/scorecard.json`
- Fixed eval: `wacky-manor/docs/eval/computed-fixes-run-20261028/`
- V3 eval: `wacky-manor/docs/eval/barrett-v3-run-20261028/`
- Toggle: `-Dmanor.cognitive.briefing-mode=false`
- Active chars: `-Dmanor.scenario.active-characters=penelope-pitstop,hooded-claw,ant-hill-mob,dick-dastardly,peter-perfect`
