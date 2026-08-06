# HANDOFF — 2026-08-06

## Last Session

Scale testing #16: built tick-based game loop (snapshot turns, cadence pacing, batched LLM calls), thread-safe WorldState, AgentInvocationService (retry with jitter, metrics), AgentExperienceService, ObservationBuilder Past Experience section, 18th character (Red Max). 4-dimension adversarial design review ($5.12, 34 findings, all resolved). Tick-based orchestrator validated with 5 agents (perfectly fair — 9 events each). Scaling beyond 5 blocked by Claude API concurrent call limits, not app code.

## Immediate Next Step

Run `/work` to resume branch `issue-16-scale-testing` for #16. Investigate Claude CLI concurrent call limit — 5 agents works, 6+ produces 0 events. Check `casehub.platform.agent.claude.max-concurrent-sessions` interaction with Anthropic API tier rate limits. Try sequential batching (batch size 1) to isolate whether it's concurrency or total call volume.

## Forage — deferred to next session

4 garden entries identified but not captured (session too long for full pipeline):
1. [gotcha] Double semaphore starvation — app + platform semaphores caused convoy
2. [technique] Tick-based game loop for LLM agents — snapshot turns, cadence pacing
3. [gotcha] ide_edit_member HTML entity corruption — angle brackets become `& lt;`
4. [undocumented] Claude CLI concurrent call limit — silently fails at 6+ concurrent
