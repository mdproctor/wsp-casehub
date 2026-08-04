# HANDOFF — 2026-08-04

## Last Session

Closed #29 — dev mode game loop fix. Root cause: ScenarioOrchestrator validated all 17 world characters for Eidos descriptors before the active-characters filter, silently killing the scenario-loop thread. Also compacted the UI: removed title, smaller rooms, chat panels for all 6 rooms. Two garden entries submitted (GE-20260804-b4cb6a validation ordering gotcha, GE-20260804-d378f3 dual-process port gotcha).

## Immediate Next Step

Pick up #16 (scale testing) or scenario design work. Dev mode now works — start with `mvn quarkus:dev -Dquarkus.http.port=8180` + `npm run dev` in webui.

## What's Left

- Garden entry GE-20260804-c21841 push failed from last session — committed locally, needs rebase+push · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Scenario design — character groups with distinct plot devices | M | Med | User idea: 4-5 chars per scenario, episodic acts |
| #16 | Scale testing beyond 5 agents | M | Med | No longer blocked — dev mode works |
| #11 | Epic: personality calibration gap | L | Med | 6 child issues (3 local, 3 eidos) |

## References

- LLM smoke test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/LlmSmokeTest.java`
- Garden: GE-20260804-b4cb6a — Eidos validation ordering gotcha; GE-20260804-d378f3 — dual-process port gotcha
