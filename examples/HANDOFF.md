# HANDOFF — 2026-07-28

## Last Session

Wacky Manor Phase 2.5 — autonomous character validation. Built switchable SCRIPTED/AUTONOMOUS mode, goals in observations, narrative event builder, auto-positioning, USE/TAKE consequences. Validated with live LLM runs. Discovered the narrative-mechanical gap: LLMs reason correctly but generate wrong structured actions. Fixed with affordance grounding chain (identity + action tags + consequence links). Scenario ends POISONED — verdict gate passes.

## Immediate Next Step

User is working on eidos-107 — may provide enhancements applicable to wacky-manor. When returning: file the blocks AffordanceRenderer issue (`wacky-manor/docs/issues/blocks-affordance-renderer.md`) once `gh auth` is fixed. Then either proceed to Phase 2.6 (observation accumulator wiring with casehub-blocks) or implement AffordanceRenderer in blocks first.

## What's Left

- File blocks AffordanceRenderer issue on casehubio/blocks — ready at `wacky-manor/docs/issues/blocks-affordance-renderer.md`, blocked on gh auth · XS · Low
- Push garden entry GE-20260728-f7ad43 — committed locally, pre-push hook blocked · XS · Low
- Wacky-manor not in parent pom modules list — `mvn` requires `-f wacky-manor/pom.xml` · XS · Low

## What's Next

| Phase | Description | Scale | Complexity | Notes |
|---|---|---|---|---|
| 2.6 | ObservationAccumulator — wire casehub-blocks, build AffordanceRenderer | M | Med | Build AffordanceRenderer in blocks first, then wire |
| 2.7 | Live LLM narrator — wire NarratorAgent | S | Low | NarratorAgent class exists, not wired |
| 2.8 | NPC system — Tier 2/3 scripted fixtures, player/NPC split | M | Med | RPG framing |
| 2.9 | Scale to 6 rooms — Library, Laboratory, Tower | L | Med | After 2.5 validates |
| 3.0 | Platform integration — memory, trust, HiL, replay | XL | High | Full casehub exercise |

## References

- Spec: `wacky-manor/docs/specs/2026-07-27-phase-2.5-autonomous-characters-design.md`
- Landscape: `wacky-manor/docs/llm-autonomy-landscape-2026.md`
- Blog: `~/claude/mdproctor.github.io/_notes/2026-07-28-mdp01-the-hooded-claw-schemes-perfectly.md`
- Garden: `GE-20260728-f7ad43` — affordance grounding chain technique
- Memory: `project_wacky-manor.md`
