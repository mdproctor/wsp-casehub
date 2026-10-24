# HANDOFF — casehub-examples (slot 196)

## Last Session

examples#146 — Consolidate pre-era and cross-repo cognitive research content.

### What was done
- **neocortex repo** (branch `issue-146-consolidate-pre-era-cross-repo-research`, rebased on #145 → #144):
  - Relocated 3 pre-era research files from wsp-casehub blog to `docs/research/`:
    - needs-pyramid (2026-09-19), wiring-problem (2026-09-20), proving-emergence (2026-09-28)
  - Relocated 5 cognitive architecture blog entries to `docs/blog/`:
    - unified-relational-personality-model (from examples, project tag updated)
    - the-mind-that-reads-minds, the-social-brain, the-mood-that-colours-memory (from agency)
    - your-agent-doesnt-know-how-it-feels (from agency)
  - Added Supporting Research Files section to era2 (7 cross-references)
  - Added 3 entries to era3 Supporting Research Files table
  - Deduplication audit verified: all 6 pre-era entries UNIQUE, all 3 neocortex blog entries UNIQUE, agency findings already relocated (#144), agency blog subset of findings spec

### Not yet done
- Source repos (examples, agency) still have the original copies — removal deferred until merge
- Engine cognitive stack inventory (issue-1104) — engine not in slot, deferred
- 7 CROSS-CUTTING agency blog entries stay in agency (cognitive concept but tightly coupled to blocks framework)
- #144, #145, #146 branches not merged — chain dependency: #146 → #145 → #144

### Relocation totals for #146
- 3 pre-era research files (wsp-casehub → neocortex/docs/research/)
- 12 cognitive blog entries (1 examples + 11 agency → neocortex/docs/blog/)
- Cross-references added to era2 (7 entries) and era3 (3 entries) Supporting Research Files

---

## Prior Session

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
