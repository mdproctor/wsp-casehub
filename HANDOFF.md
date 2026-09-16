# HANDOFF — casehub-examples

## Last Session

Completed blocks#283 directive-minimal architecture — Batch 4 (Tasks 7-8). Removed constraint rendering from CharacterCognition (now split: HARD → Prime Directives in system prompt, SOFT → ConstraintPromptSection via CognitionCore). Wired GoalProposalOrchestrator into ScenarioOrchestrator bootstrap with goal seeding from social-config.yaml. Integration test verifies full prompt pipeline. Also installed blocks branch to local Maven to verify cross-module compilation.

## Immediate Next Step

Run `work next` to advance the queue past blocks#283 to the next examples issue (#66 Drive adaptation). Before that, the blocks branch `issue-283-directive-minimal-architecture` needs work-end (4 commits, no PR yet).

## Cross-Module

- **blocks** branch `issue-283-directive-minimal-architecture` has 4 commits (renderer, preamble, constraint filtering, personality dedup). Not yet merged — needs work-end.
- **blocks#286** filed: user guide for neurocortex seeding best practices (not just mechanical how-to, but content placement decisions, goal axis selection, constraint severity, initial beliefs, worked examples).
- Pre-existing broken integration tests in wacky-manor: `ManorResourceProfileTest`, `CognitiveQueryIntegrationTest`, `SocialCognitionIntegrationTest` — class resolution failures unrelated to this work.

## References

| Artifact | Path |
|----------|------|
| Design spec | specs/issue-63-fix-cdi-config-beans/2026-09-16-directive-minimal-architecture-design.md |
| Decisions | specs/issue-63-fix-cdi-config-beans/decisions.md |
| Implementation plan | plans/2026-09-16-directive-minimal-architecture.md |
