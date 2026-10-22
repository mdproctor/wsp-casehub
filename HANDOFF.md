# HANDOFF — casehub-examples (slot 196)

## Last Session

Closed neocortex#656 + examples#143 — CDI path: orchestrators get null SessionPipeline. Landed, pushed, issues closed. All repos on main, clean.

## Immediate Next Step

Start examples#122 Batch 1 — Drive expression pipeline (get working). Then:
1. examples#122 Batch 1 — DriveProfileDerivationPhase not producing profiles (upstream)
2. examples#111 — CAPS settling dampens too aggressively (downstream, unblocks #102)
3. examples#112 — Unresolvable disposition modifiers
4. examples#102 — Pipeline tuning (unblocked by #111)

All pre-run blockers from epic #109 + #122. Fix before next scenario runs.

## References

- `wacky-manor/.../ManorCognitivePipelineBuilder.java` — composable builder
- `wacky-manor/.../ManorCognitivePipelineTest.java` — 9 tests
- `wacky-manor/.../PlaybookOrchestrator.java` — refactored to use builder
- `wsp-casehub/specs/issue-115-drive-expression-gap/` — drive expression research
