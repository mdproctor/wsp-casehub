# HANDOFF — 2026-08-04

## Last Session

Implemented #27 (pause/play/speed controls) and #28 (character profile panel) on branch `issue-27-pause-play-profiles`. 10 commits: WorldState pause/speed, CharacterAgentLoop pause gate, WebSocket control events, REST endpoints (pause/resume/speed/profile), CharacterProfileDTO, 12 new SVG sprites (all 17 characters), character-profile Lit component, configurable active-characters filter (%dev runs core 5). Also wired eidos BriefingCoherenceValidator and deleted legacy CharacterProfile/CharacterProfileLoader. 232 tests green. LLM smoke test proves full pipeline works. UI verified via Playwright.

## Immediate Next Step

Debug why Quarkus dev mode character threads die on first LLM call despite `LlmSmokeTest` proving the pipeline works in test mode. Start with `LlmSmokeTest.java` (committed) as baseline — it renders a real descriptor and gets a valid LLM response. The dev mode issue is environmental, not code.

## What's Left

- Dev mode LLM integration not working — characters don't produce events despite test proving pipeline works · M · Med
- Branch `issue-27-pause-play-profiles` not closed — needs work-end after dev mode fix · XS · Low
- Garden entry GE-20260804-c21841 push failed — committed locally, needs rebase+push · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Scenario design — character groups with distinct plot devices | M | Med | User idea: 4-5 chars per scenario, episodic acts |
| #16 | Scale testing beyond 5 agents | M | Med | Blocked on dev mode LLM fix |
| #11 | Epic: personality calibration gap | L | Med | 6 child issues (3 local, 3 eidos) |

## References

- Spec: `specs/issue-27-pause-play-profiles/2026-08-04-pause-play-profiles-design.md`
- Plan: `plans/2026-08-04-pause-play-profiles.md`
- LLM smoke test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/LlmSmokeTest.java`
- Garden: GE-20260804-c21841 — ide_edit_member record body wipe gotcha
