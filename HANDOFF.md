# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-28
**Branch:** issue-156-computed-mode-quality (examples)

## What was done this session

### Phase 1 — Quick fixes (#164, #165)

1. **#165 Ungate formation memories** — removed `briefingMode` guard from formation memory rendering in `CharacterCognition.renderCognitiveSections()`. Formation memories are biographical data, not prose templates — they should render in both modes.

2. **#164 GENERIC agentId leak** — multiple fixes:
   - Entity IDs now use slugified display names (`clara-bellingham`) instead of cartoon agentIds (`penelope-pitstop`)
   - Added `WorldState.resolveCharacter(ref)` — tries direct agentId, then slug-matched display name
   - `ActionResolver`, `PlaybookOrchestrator` response parsing, `NarrativeEventBuilder` all use resolved names
   - Descriptor names now applied to ALL characters at startup (not just active ones)
   - Non-active characters (Muttley, Prof Pat Pending, etc.) marked inactive so they don't appear in room observations
   - Result: **zero cartoon name leaks** in 103-event eval run

3. **Barrett V3 instruction** — replaced "Do not conclude. Do not explain. Discover." with "FROM BODY TO ACTION" bridge. High-intensity drives now move directly into words and actions without cognitive mediation.

### Phase 2 — Baseline comparison

Cross-baseline scorecard established (B0, B1, V2, Computed, Fixed):

| Baseline | Quality | Register | Notes |
|----------|---------|----------|-------|
| B0 (Oct 2) | 3.64 | 1.52 | Original generic colloquial |
| B1 (Oct 4) | 3.95 | 2.72 | Emotional core baseline |
| V2 (Oct 26) | 3.88 | 3.35 | Full cognitive run (briefing mode) |
| Pre-fix computed | 3.78 | 2.94 | Before fixes |
| Fixed computed | 3.76 | 3.61 | After #164/#165 fixes — **register surpasses V2** |

### Phase 3 — Iteration (Barrett V3)

Barrett V3 iteration running. Eval pending.

Key remaining gaps vs V2:
- HC gloating quality: 2.50 vs 4.00 — characters observe somatic gloating but don't express it overtly
- Peter gallantry quality: 3.00 vs 4.00 — better with formation memories but still below V2

### Commits on branch

| SHA | Description |
|-----|-------------|
| 0804d893 | fix(#164,#165): ungate formation memories + fix GENERIC agentId leak |
| 173d2d24 | fix(#164): apply descriptor name to CharacterState at startup |
| 58e6521a | fix(#164): mark non-active characters inactive + apply name overrides to all |
| 28d464f3 | data(#156): cross-baseline scorecard |
| 57451552 | data(#156): computed-fixes eval — register 2.94→3.61 |
| 2edb0799 | feat(#156): Barrett V3 — bridge somatic awareness to behavioral expression |

## Key References

- Barrett instruction: `CharacterCognition.java` lines 208-248 (in `!briefingMode` block)
- WorldState.resolveCharacter: `WorldState.java` line 39
- Entity ID slug: `ManorWorldObservationProvider.java` line 106
- Scorecard: `wacky-manor/docs/eval/scorecard.json`
- Fixed eval: `wacky-manor/docs/eval/computed-fixes-run-20261028/`
- Toggle: `-Dmanor.cognitive.briefing-mode=false`
- Active chars: `-Dmanor.scenario.active-characters=penelope-pitstop,hooded-claw,ant-hill-mob,dick-dastardly,peter-perfect`

## Next Steps

1. Evaluate Barrett V3 results
2. If HC gloating quality remains low: investigate consolidated drive profile vs V2 prose
3. If Peter gallantry quality remains low: check formation memory rendering + consolidated profile
4. Full 300-tick run when short runs match or exceed V2
