# HANDOFF — casehub-examples (slot 196)

## Last Session

examples#144 — Relocate cognitive research docs from wacky-manor to neocortex.

### What was done
- **examples repo** (branch `issue-144-relocate-cognitive-research-docs`):
  - Removed 13 research files from `wacky-manor/docs/` and `wacky-manor/docs/research/`
  - Added `research/RELOCATED.md` pointer
  - Updated CLAUDE.md: research/journals go to foundation repo

- **neocortex repo** (branch `issue-144-relocate-cognitive-research-docs`):
  - Added 15 research files to `docs/research/` (10 from examples, 3 more from examples, 2 from agency)
  - Phase 10 added to CHARACTER-EMERGENCE-FINDINGS with principles renumbered P85-P88
  - Era3 consolidated with 4 new phases (P89-P93): multi-channel expression, discriminative profiles, consolidation trigger, channel weighting
  - Supporting research file links added to era3

- **workspace** (branch `issue-144-relocate-cognitive-research-docs`):
  - `cognitive-research-inventory.md` — complete audit of 57 documents across 8 repos

### Issues filed
- examples#145 — Era2 gap: 7 un-consolidated blog entries from issue-115 series
- examples#146 — Pre-era and cross-repo cognitive research consolidation

### Not yet done
- Neither branch merged to main yet — needs review
- Engine cognitive-stack-inventory not relocated (engine not in this slot)

## Immediate Next Step

Review and merge #144 branches in both examples and neocortex. Then continue with:
1. examples#122 Batch 1 — Drive expression pipeline (get working)
2. examples#111 — CAPS settling dampens too aggressively
3. examples#112 — Unresolvable disposition modifiers
4. examples#102 — Pipeline tuning (unblocked by #111)

## References

- `wacky-manor/.../ManorCognitivePipelineBuilder.java` — composable builder
- `wacky-manor/.../ManorCognitivePipelineTest.java` — 9 tests
- `wacky-manor/.../PlaybookOrchestrator.java` — refactored to use builder
- `wsp-casehub/specs/issue-115-drive-expression-gap/` — drive expression research
