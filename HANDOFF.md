# HANDOFF — casehub-examples

## Last Session

Completed #78 (CDI registration for blocks consolidation phases) — the first issue in the #77 cognitive wiring audit queue. Key discovery: blocks-core has no Jandex index, so Quarkus never discovers its CDI beans. All CDI registration must be in the application (wacky-manor), not in blocks-core.

Created `ManorConsolidationBeans` with self-contained `@Produces @Singleton` methods for all three consolidation phases (DriveAdaptationPhase, BeliefRevisionPhase, RelationshipStagePhase). Config records constructed inline — no CDI dependency on unindexed blocks-core types.

Fixed pre-existing CDI context failures: excluded engine/eidos runtime beans missing platform SPIs, fixed `TrustEvolutionConfigProducer` record-proxy incompatibility (`@Singleton` instead of `@ApplicationScoped`).

536/537 tests pass. One pre-existing failure: `SocialCognitionIntegrationTest.characterCognitionRendersSectionsWithSocialConfig` expects "Your Drives" from CharacterCognition, which moved to CognitionCore pipeline in Phase C.

Also created `ManorRelationshipStageConfigProvider` (per-character familiarity thresholds from YAML).

## Immediate Next Step

Start #79 — Wire CognitionCore.promptSections() into wacky-manor (M / Med). This is the critical gap identified in audit §3: wacky-manor uses CharacterCognition.renderCognitiveSections() (returning ObservationSection) instead of CognitionCore.promptSections() (returning PromptSection). All Phase C rendering is invisible to characters.

The audit recommends Option 3 (hybrid): keep CharacterCognition for application-level sections (beliefs, norms, social awareness, trust) but delegate to CognitionCore.promptSections() for platform-level sections. Add an adapter from PromptSection → ObservationSection.

## Key Discovery — blocks-core CDI

blocks-core jar has NO Jandex index. `SocialCognitionDefaultBeans` and all `@ConfigMapping` types (`DriveAdaptationConfig`, `NeedSatisfactionConfig`) are invisible to Quarkus. Attempting `quarkus.index-dependency` for blocks-core causes cascading ambiguity errors (MoodOrchestrator duplicates, etc.). The working approach: construct all blocks-core types directly in wacky-manor producers with inline config records.

The blocks revert commit (`3a56e9f`) on `issue-283-directive-minimal-architecture` is safe — it just undid an unnecessary `@ApplicationScoped` addition to `BeliefRevisionPhase` that never would have been discovered anyway.

## Queue

Position 1/9 (advancing to #79 next). All under epic #77.

| # | Issue | Scale | Complexity | Blocked by | Status |
|---|-------|-------|------------|------------|--------|
| 78 | CDI registration for blocks consolidation phases | S | Med | — | Done |
| 79 | Wire CognitionCore.promptSections into wacky-manor | M | Med | #78 | Next |
| 84 | CognitionConfig.all() defaults | XS | Low | — | — |
| 81 | shouldCompareSocially adapted intensities | S | Low | — | — |
| 82 | Wire shouldDisclose/shouldCooperate | S | Low | — | — |
| 83 | CognitivePreambleGenerator disambiguation | S | Low | — | — |
| 85 | Render goals in observation pipeline | S | Low | #79 | — |
| 86 | NeedTierMappingProvider @DefaultBean | XS | Low | — | — |

## Cross-Module

- **blocks** branch `issue-283-directive-minimal-architecture` — has a revert commit from this session (reverted unnecessary `@ApplicationScoped` on BeliefRevisionPhase). 2 extra commits vs last session (the original change + its revert). Net effect: no change from previous session's state.
- **neocortex** — no changes this session.

## Blog Guidance

User wants blog entries to teach, advocate, and demonstrate practical relevance. Entries should be meaningful, engaging, and informative — show readers why this work matters to them, not just what was built.

## References

| Artifact | Path |
|----------|------|
| Audit report | audits/2026-09-19-cognitive-architecture-audit.md |
| Decisions (D1-D45) | specs/issue-63-fix-cdi-config-beans/decisions.md |
| Previous blog | blog/2026-09-19-mdp01-needs-pyramid-agent-cognition.md |
