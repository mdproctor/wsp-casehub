# HANDOFF — 2026-08-05

## Last Session

Closed #12, #13, #14 (epic #11 personality calibration gap). Built BriefingCoherenceJudge in eidos-eval (LLM judge for briefing/disposition coherence), added toBuilder() to AgentDescriptor in eidos-api, and extended wacky-manor's PromptQualityTest with a 4×4 briefing richness matrix and a 3-mechanism experiment loop. Design spec underwent standard adversarial review (4 dimensions, $29.65, 51 issues, 0 unresolved). One garden entry revised (GE-20260529-182916: ctx.py CWD vs git root fast-path false negative).

## Immediate Next Step

Run the experiments. Execute `mvn test -pl wacky-manor -Pllm-eval` to collect the full TAA matrix. Start with a targeted run: `-Deval.layers=composite -Deval.briefings=empty,name_only -Deval.characters=hooded-claw` to validate the harness before the full matrix.

## What's Left

- Garden entry push failed — committed locally, needs rebase+push · XS · Low
- Eidos cross-repo commits (toBuilder, BriefingCoherenceJudge) on eidos main — need push · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Run full experiment matrix and analyse results | M | Low | Harness built, just needs execution |
| — | Scenario design — character groups with distinct plot devices | M | Med | 4-5 chars per scenario, episodic acts |
| #16 | Scale testing beyond 5 agents | M | Med | Dev mode works, no longer blocked |

## References

- Spec: `wacky-manor/docs/specs/` (promoted by design review)
- Plan: workspace `plans/2026-08-04-personality-calibration.md`
- Garden: GE-20260529-182916 — ctx.py CWD fast-path false negatives (failure mode 4 added)
