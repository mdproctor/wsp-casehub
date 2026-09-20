# HANDOFF — casehub-examples

## Last Session

Completed #79 (Wire CognitionCore.promptSections into wacky-manor) and #84 (CognitionConfig.all() defaults).

### #79 — CognitionCore pipeline wiring

CharacterCognition.renderCognitiveSections() now calls cognitionCore.promptSections() after building app-level sections (beliefs, norms, social awareness, trust). Each PromptSection is adapted to ObservationSection.TextBlock via adaptPromptSection() — extracts `## Header` from the contributed text as the TextBlock header, rest as content.

ScenarioOrchestrator passes mmStore and new ManorNeedTierMappingProvider() to CognitionCore constructor (positions 12-13), with characterDrives and needsPyramid enabled in config.

Verified quantifiably: hooded-claw's scheming renders at 90%, self-preservation and dominance both appear, need tiers SELF_EXPRESSION and SAFETY computed from ManorNeedTierMappingProvider.

3 new tests in CharacterCognitionTest: content assertions on drive intensities and need tiers, null-cognitionCore safety, driveless-character filtering.

540 tests pass (1 pre-existing failure: SocialCognitionIntegrationTest expects "Your Drives" which moved to CognitionCore in Phase C).

### #84 — CognitionConfig.all() default

Fixed in blocks-core (slot clone at slots/196/blocks): changed characterDrivesEnabled from false to true in all(). One-line change + test assertion. Installed to local Maven repo.

Note: wacky-manor doesn't use all() — it uses explicit CognitionConfig.none().with(...) from #79. The fix is for SocialAvatarCognition and any future callers of all().

## Immediate Next Step

Start #81 — shouldCompareSocially read adapted intensities (S / Low). From audit §8: ManorContextStrategy.shouldCompareSocially() reads from static SocialConfig.Drive records, not from adapted drive intensities in MindMap nodes. D22 says it must read adapted intensities. A character whose scheming drive has decayed to 0.1 still gets social awareness based on the original 0.9. Adaptation is functionally inert for behavioral decisions.

The fix: ManorContextStrategy needs access to MindMapStore and tenantId. Read drive-intensity nodes filtered by agent-id, use the adapted intensity instead of the static SocialConfig.Drive.intensity().

## Key Discovery — blocks-core CDI (from prior session, still relevant)

blocks-core jar has NO Jandex index. All CDI registration must be in the application (wacky-manor), not in blocks-core. ManorConsolidationBeans handles this with self-contained @Produces @Singleton methods.

## Queue

Position 4/9. All under epic #77.

| # | Issue | Scale | Complexity | Blocked by | Status |
|---|-------|-------|------------|------------|--------|
| 78 | CDI registration for blocks consolidation phases | S | Med | — | Done |
| 79 | Wire CognitionCore.promptSections into wacky-manor | M | Med | #78 | Done |
| 84 | CognitionConfig.all() defaults | XS | Low | — | Done |
| 81 | shouldCompareSocially adapted intensities | S | Low | — | Next |
| 82 | Wire shouldDisclose/shouldCooperate | S | Low | — | — |
| 83 | CognitivePreambleGenerator disambiguation | S | Low | — | — |
| 85 | Render goals in observation pipeline | S | Low | #79 | — |
| 86 | NeedTierMappingProvider @DefaultBean | XS | Low | — | — |

## Cross-Module

- **blocks** branch `issue-283-directive-minimal-architecture` (slot clone at slots/196/blocks) — 1 new commit this session: CognitionConfig.all() characterDrivesEnabled fix. Installed to local Maven repo.
- **neocortex** — no changes this session.

## Test Suite Note

Full wacky-manor suite (540 tests) hangs when run online due to GitHub Packages 401 on SNAPSHOT metadata resolution. Use `-o` (offline mode): `mvn test -pl wacky-manor -s slot-settings.xml -o`. Takes ~55 seconds offline.

## Blog Guidance

User wants blog entries to teach, advocate, and demonstrate practical relevance. Entries should be meaningful, engaging, and informative — show readers why this work matters to them, not just what was built.

## References

| Artifact | Path |
|----------|------|
| Audit report | audits/2026-09-19-cognitive-architecture-audit.md |
| Decisions (D1-D45) | specs/issue-63-fix-cdi-config-beans/decisions.md |
| Previous blog | blog/2026-09-19-mdp01-needs-pyramid-agent-cognition.md |
