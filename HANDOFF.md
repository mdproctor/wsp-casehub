# Session Handover — 2026-09-16

## What happened

Phase C #65 (trust evolution from experience) completed — design through implementation across 3 repos.

**Design phase:**
- Brainstormed trust evolution approach — user identified ledger's `TrustScoreComputer` as the right computation engine (Bayesian Beta with temporal decay, not a custom heuristic)
- User pushed for maximum reusability: all framework code in blocks (YAML-configurable), wacky-manor provides zero trust-specific Java
- 11 design decisions captured, refined through 3-round adversarial decision review (9 decisions, revised 6, added 2)
- Design spec written and reviewed (3-round spec review, 22 findings, all resolved)
- Key architectural insight: per-relationship trust from per-actor model via attestor-filtered scoring

**Implementation (4 batches, 6 tasks):**
- **blocks-core:** `TrustEvolutionConfig` record + `OverlayTrustPropertyModel` constants
- **blocks:** `TrustRelevantAction` CDI event + `TrustEventRecorder` async observer
- **blocks-agentic-yaml:** `TrustEvolutionConfigSpec` (YAML spec type)
- **neocortex-mindmap-intelligence:** `TrustConsolidationPhase` @Priority(22) — reads ledger attestations, computes per-relationship Bayesian Beta scores, writes to overlay nodes
- **wacky-manor:** `trust-evolution.yaml` config, `ManorTrustEvolutionConfigLoader`, `TrustEvolutionConfigProducer` CDI bean, `ScenarioOrchestrator` CDI event wiring, `CharacterCognition` trust rendering via `CognitiveObservationSections.trustSection()`

**Artifacts:**
- ADR 0001: Lightweight ledger trust scoring for cognitive agents
- 3 garden entries: OverlayRef gotcha, per-relationship trust technique, lightweight ledger technique
- Diary: "When Trust Learns to Decay"
- Follow-up issues: #71 (remove ManorTrustEvents), #72 (personality-driven trust), #73 (per-capability trust), #74 (YAML DSL migration), #75 (action model extension)

## Decisions

- Ledger TrustScoreComputer with in-memory persistence — not custom heuristic
- Per-relationship trust via attestor-filtered scoring — filter attestation map by attestorId before passing to compute()
- All framework in blocks (blocks-core/blocks/agentic-yaml), consolidation phase in neocortex-mindmap-intelligence
- AgentTrustProvider SPI unchanged (global trust only) — per-relationship trust reads directly from overlay node properties
- ManorTrustEvents superseded by CDI event + ledger attestation model
- Evidence gate: alpha + beta <= 2 → TrustLevel.UNKNOWN (no-evidence is not an opinion)
- Wacky-manor's YAML DSL migration deferred to follow-up #74

## Known issues

- **#63 (CDI config beans):** 18 unsatisfied deployment errors in Quarkus test context — blocks cognitive features (PersonalityEvolutionConfig, StrategyLearningConfig, NarrativeConfig, etc.) lack CDI producers in wacky-manor. Blocks integration testing for ALL cognitive features. This is the active issue on the branch.
- **blocks landing:** work-end reported merge conflict for blocks — pushed manually to local repo. Verify blocks main is clean.
- **Integration tests:** Trust evolution unit tests pass (13 across 3 repos). End-to-end Quarkus integration tests blocked by #63.

## Next action

Branch `issue-63-fix-cdi-config-beans` is scaffolded with the full Phase C queue:

| Position | Issue | Scale | Repo |
|----------|-------|-------|------|
| 0 (active) | **#63** — Fix CDI config bean failures | S | examples |
| 1 | **blocks#283** — Directive-minimal architecture | L / High | blocks |
| 2 | **#66** — Drive adaptation from reward signals | L / High | examples → blocks |
| 3 | **#70** — Relationship stage thresholds | S / Low | examples → blocks |
| 4 | **#67** — Belief revision from contradicting evidence | M / Med | examples → blocks |
| 5 | **#68** — Needs pyramid | L / High | examples → blocks |
| 6 | **#69** — Goal prioritization from unmet needs | L / High | examples → blocks |

Start with `work continue` — the branch and queue are ready.

**blocks#283** is the architectural backbone: shifting cognitive data from the briefing directive to neocortex subsystems. The briefing becomes identity + cognitive instructions ("you have beliefs, trust perceptions, drives — use them"); the actual cognitive state flows through observation sections fed by consolidation. Without #283, the cognitive feedback loops (#65-#70) are supplementary context that the LLM's briefing directive overpowers.

## References

| Artifact | Path |
|----------|------|
| Trust evolution spec | specs/issue-65-trust-evolution/2026-09-16-trust-evolution-design.md |
| Trust evolution decisions | specs/issue-65-trust-evolution/decisions.md |
| ADR 0001 | docs/adr/0001-lightweight-ledger-trust-scoring.md |
| Diary entry | examples/blog/2026-09-16-mdp03-when-trust-learns-to-decay.md |
| Phase C epic | casehubio/examples#64 |
| Trust evolution plan | plans/2026-09-16-trust-evolution.md |
