# Cognitive Architecture Audit — Full Stack Review

**Date:** 2026-09-19
**Scope:** All three phases (A, B, C) across blocks, examples/wacky-manor, and neocortex
**Branch state:** blocks `issue-283-directive-minimal-architecture` has 14 unmerged commits; examples `issue-63-fix-cdi-config-beans` merged to main

---

## 1. Consolidation Pipeline

### Status: Complete (with caveats)

**Platform phases (in neocortex/engine jars):**

| Priority | Phase | Source | Output consumed by | Tests |
|----------|-------|--------|-------------------|-------|
| @10 | AccessFrequencyPhase | neocortex | Internal scoring | Platform |
| @15 | ExperienceConsolidationPhase | neocortex | BeliefRevision, DriveAdaptation, RelationshipStage (all read graduated nodes) | Platform |
| @20 | MergeDetectionPhase | neocortex | Internal deduplication | Platform |
| @22 | TrustConsolidationPhase | engine | CharacterCognition.renderTrustSections() reads overlay trust properties | Platform |
| @30 | CommunitySummaryPhase | neocortex | Unknown — needs investigation | Platform |

**Blocks phases (on feature branch `issue-283-directive-minimal-architecture`):**

| Priority | Phase | Output consumed by | Tests |
|----------|-------|-------------------|-------|
| @16 | BeliefRevisionPhase | CharacterCognition.renderCognitiveSections() reads Belieflike nodes with `provenance: "belief-revision"` | Yes — DriveAdaptationPhaseNeedsSatisfactionTest pattern |
| @17 | DriveAdaptationPhase (+ need satisfaction) | CharacterDrivePromptSection reads drive nodes; NeedsPyramidPromptSection reads satisfaction nodes; ManorGoalRevisionStrategy reads satisfaction | Yes |
| @18 | RelationshipStagePhase | PerceptionTranslator receives stage for rendering; ManorContextStrategy.shouldDisclose/shouldCooperate use stage | Yes |

**Additional platform phases not in the original list:**

| Priority | Phase | Notes |
|----------|-------|-------|
| ? | CuriosityRefreshPhase | neocortex — refreshes curiosity-related nodes |
| ? | SchemaDiscoveryPhase | neocortex — discovers schema patterns |

**Caveat:** CuriosityRefreshPhase and SchemaDiscoveryPhase priorities are unknown — they're in the neocortex jar. If they overlap with the blocks @16-@18 range, ordering conflicts could occur silently.

---

## 2. CognitionConfig Subsystem Coverage

### Status: Gap — blocks branch not merged

**On blocks main (10 fields):** moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts

**On blocks feature branch (12 fields):** + characterDrivesEnabled, needsPyramidEnabled

**Wiring in CognitionCore.promptSections() (feature branch):**

| Config flag | PromptSection | Data source | Populated by |
|-------------|---------------|-------------|-------------|
| moodEnabled | MoodPromptSection | MoodOrchestrator (in-memory per-tick) | CognitionCore.tick() |
| drivesEnabled | DrivePromptSection | DriveOrchestrator (4 SDT axes, in-memory) | CognitionCore.tick() |
| narrativeEnabled | NarrativePromptSection | NarrativeOrchestrator (in-memory) | CognitionCore.tick() |
| userModelEnabled | UserModelPromptSection | UserModelOrchestrator (in-memory) | CognitionCore.recordInteraction() |
| mentalModelEnabled | MentalModelPromptSection | MentalModelOrchestrator (in-memory) | CognitionCore.recordInteraction() |
| strategyEnabled | StrategyPromptSection | StrategyLearningOrchestrator (in-memory) | CognitionCore.tick() |
| goalsEnabled | GoalPromptSection | GoalProposalOrchestrator (in-memory) | Goal formation/revision |
| characterDrivesEnabled | CharacterDrivePromptSection | MindMapStore (drive-intensity nodes) | ManorCognitiveSeeder + DriveAdaptationPhase |
| needsPyramidEnabled | NeedsPyramidPromptSection | MindMapStore (need-satisfaction nodes) | ManorCognitiveSeeder + DriveAdaptationPhase |
| directivePrompts | DirectiveSection wrapper | N/A (wraps other sections) | N/A |
| memoryHygieneEnabled | (no PromptSection) | MemoryHygieneOrchestrator | CognitionCore.tick() |
| innerLifeEnabled | (no PromptSection) | InnerLifeOrchestrator | CognitionCore.tick() |

**Also rendered unconditionally (no config flag):**
- PersonalityPromptSection — from AgentDescriptor.disposition()
- ConstraintPromptSection — from AgentDescriptor.constraints() (soft only)

**Flags always false in practice:**
- `directivePrompts` — defaults to false in `all()`, only enabled via `withDirectives()`
- `characterDrivesEnabled` — defaults to false in `all()` — **this is wrong**, should be true for wacky-manor. `all()` returns false for both characterDrives and directivePrompts, which means characters using `CognitionConfig.all()` do NOT get character drive rendering. This needs investigation.

---

## 3. Observation Pipeline — THE CRITICAL GAP

### Status: **CRITICAL GAP — CognitionCore.promptSections() is not called by wacky-manor**

**Two parallel rendering pipelines exist:**

**Pipeline A — CognitionCore.promptSections() (blocks-level)**
- Returns `List<PromptSection>` (blocks-speech API)
- Wired into `SocialAvatarCognition.buildSections()` which feeds into the eidos `SpeechPromptAssembler` CDI pipeline
- Contains: Personality, Constraints, Mood, SDT Drives, Narrative, UserModel, MentalModel, Strategy, Goals, CharacterDrives, NeedsPyramid
- **NOT used by wacky-manor** — `SocialAvatarCognition` is never referenced in wacky-manor production code

**Pipeline B — CharacterCognition.renderCognitiveSections() (wacky-manor-level)**
- Returns `List<ObservationSection>` (blocks-summarisation API)
- Called from `ScenarioOrchestrator` line 271 via `.withCognitiveSections()`
- Contains: Beliefs, Norms, Social Awareness, Trust Perceptions
- **This is what the game actually uses**

**Consequence:** Every PromptSection built in Phase C (CharacterDrivePromptSection, NeedsPyramidPromptSection) and every platform-level section (MoodPromptSection, DrivePromptSection, NarrativePromptSection, etc.) is **invisible to wacky-manor characters**. The character's LLM prompt never sees:
- Mood state
- SDT psychological drives
- Character motivational drives (the adapted ones)
- Needs pyramid satisfaction
- Narrative arc
- User model
- Mental model
- Strategy reflections
- Goals (via the prompt — goals are used for action selection via GoalEvaluator, but not rendered in the observation pipeline)

**The `cognitionCore` field in CharacterCognition is dead code** — assigned in constructor, never called.

**Root cause:** Wacky-manor uses a manual rendering approach (ObservationSection) via its own `CharacterCognition` class, bypassing the CDI-managed `SocialAvatarCognition` → `SpeechPromptAssembler` pipeline. This was likely pragmatic during early development but means all CognitionCore prompt sections are orphaned from the game's actual prompt construction.

**Fix options:**
1. **Wire CognitionCore sections into CharacterCognition** — call `cognitionCore.promptSections()`, adapt each `PromptSection` to `ObservationSection`, and include them in `renderCognitiveSections()`. This is the minimal change.
2. **Migrate to SocialAvatarCognition pipeline** — replace CharacterCognition's rendering with the platform CDI pipeline. Larger change but eliminates the parallel pipeline problem permanently.
3. **Hybrid** — keep CharacterCognition for application-level sections (beliefs, norms, social awareness, trust) but delegate to CognitionCore.promptSections() for platform-level sections. Add an adapter from PromptSection → ObservationSection.

**Recommendation:** Option 3 (hybrid) — preserves the application-level rendering while connecting the platform pipeline. This is probably a new issue.

---

## 4. Drive System Coherence

### Status: Gap — preamble disambiguation missing, `all()` defaults wrong

**Two systems verified:**
- SDT drives: DriveOrchestrator (4 axes) → DrivePromptSection (rendered under "Psychological Needs")
- Character drives: SocialConfig.Drive → MindMap nodes → CharacterDrivePromptSection (rendered under "Character Motivations")

**Gaps:**
1. **CognitivePreambleGenerator does not disambiguate the two systems (D3 revision).** The preamble says "You have motivational drives" but doesn't distinguish SDT vs character drives. D3's revision specified: "Your psychological needs — curiosity, competence, affiliation, autonomy — shift based on your interactions. Separately, your character motivations — the drives that define who you are — evolve based on your experiences." This text is missing.

2. **CognitivePreambleGenerator still has state-dependent language (D3 constraint).** Line: "some are stronger than others right now" — D3's preamble wording constraint says this violates the system prompt caching strategy. Should be architecture-only language.

3. **CognitivePreambleGenerator does not check `config.characterDrivesEnabled()` or `config.needsPyramidEnabled()`.** It only checks `drivesEnabled` (SDT drives). When character drives or needs pyramid are active, the preamble should mention them.

4. **`CognitionConfig.all()` returns `characterDrivesEnabled=false`.** This means applications using `all()` don't get character drive rendering. The design decision D15 intended character drives to be rendered when enabled — `all()` should return true for it.

5. **NeedsPyramidPromptSection references character drives, not SDT drives** — confirmed correct per D30.

---

## 5. Seeder Coverage

### Status: Complete

ManorCognitiveSeeder.seed() creates:
- Belief nodes (Belieflike trait, from initialBeliefs) ✓
- Drive intensity nodes (cognitiveKind: "drive-intensity") ✓
- Need satisfaction nodes (cognitiveKind: "need-satisfaction") ✓

ManorCognitiveSeeder.seedGoals() creates goals via GoalProposalOrchestrator ✓

ManorCognitiveSeeder.seedPeople() creates:
- Person entity nodes (Entitylike trait) ✓
- Perspectival overlay nodes (overlay trait, per-observer PAD) ✓

**No orphaned seeds.** Every seeded node type is consumed by at least one phase or rendering section:
- Belief nodes → BeliefRevisionPhase reads, CharacterCognition renders
- Drive nodes → DriveAdaptationPhase reads/updates, CharacterDrivePromptSection renders
- Need nodes → DriveAdaptationPhase reads/updates, NeedsPyramidPromptSection renders
- Person/overlay nodes → RelationshipStagePhase updates familiarity, CharacterCognition reads trust + stage

---

## 6. YAML Config Coverage

### Status: Complete

All social-config.yaml sections are parsed and consumed:

| Section | Parsed by | Consumed by |
|---------|-----------|-------------|
| goals | ManorSocialConfigLoader | ManorCognitiveSeeder.seedGoals() → GoalProposalOrchestrator |
| drives | ManorSocialConfigLoader | ManorCognitiveSeeder.seed() → drive MindMap nodes |
| reinforcement | ManorSocialConfigLoader | ScenarioOrchestrator → DriveAdaptationPhase config |
| norms | ManorSocialConfigLoader | CharacterCognition.renderCognitiveSections() via ManorNormFilter |
| initial-beliefs | ManorSocialConfigLoader | ManorCognitiveSeeder.seed() → belief MindMap nodes |
| relationships | ManorSocialConfigLoader | ManorCognitiveSeeder.seedPeople() → overlay PAD values |
| familiarity-thresholds | ManorSocialConfigLoader | SocialConfig.stageConfig → RelationshipStagePhase |

**Characters with drives but no reinforcement:**
- Only the 5 "major" characters (hooded-claw, penelope-pitstop, peter-perfect, dick-dastardly, ant-hill-mob) have drives and reinforcement.
- The remaining 12 characters (muttley through red-max) have only goals — no drives, no reinforcement. This is intentional (simpler characters).

**No mismatches found** — every character with drives also has reinforcement config.

---

## 7. Goal Lifecycle

### Status: Complete (with rendering gap from §3)

- **Formation:** ManorGoalFormationStrategy (LLM-driven) → GoalProposalOrchestrator → ManorGoalEvaluator
- **Seeding:** ManorCognitiveSeeder.seedGoals() → DriveGoalProposal → GoalProposalOrchestrator.registerGoals()
- **Revision:** ManorGoalRevisionStrategy (LLM-driven, now needs-aware with satisfaction levels) → GoalRevisionProposal
- **Reprioritization:** REPRIORITIZE action → ManorGoalEvaluator maps Double → GoalPriority (PRIMARY/SECONDARY)
- **Completion:** COMPLETE action → ManorGoalEvaluator removes from active list
- **Abandonment:** ABANDON action → ManorGoalEvaluator removes from active list
- **Plan formation:** ManorPlanFormationStrategy → ManorPlanEvaluator (when config.plan.enabled)

**Gap:** GoalPromptSection exists in CognitionCore.promptSections() but is not rendered in wacky-manor's actual prompt (see §3). Goals influence action selection via ManorGoalEvaluator but the character never "sees" its goals in its observation pipeline. The character can't reason about its own goals in dialogue.

---

## 8. Cross-Subsystem Integration Points

### DriveAdaptation → NeedSatisfaction (D33)
**Status: Complete** — integrated into DriveAdaptationPhase on the feature branch.

### NeedSatisfaction → GoalRevision (D43, D44)
**Status: Complete** — ManorGoalRevisionStrategy reads satisfaction nodes via MindMapStore, includes numeric values in LLM prompt.

### BeliefRevision → CharacterCognition rendering (#67)
**Status: Complete** — CharacterCognition reads Belieflike nodes from MindMapStore, maps to Belief<T>, renders via CognitiveObservationSections.beliefsSection() with [REVISED] marking for provenance "belief-revision".

### RelationshipStage → PerceptionTranslator stage-gated rendering (#70)
**Status: Complete** — PerceptionTranslator.translate() receives stage parameter, filters by stage (stranger = skip, acquaintance = dominant dimension only, friend = full with trajectory).

### TrustConsolidation → CharacterCognition trust rendering
**Status: Complete** — CharacterCognition.renderTrustSections() reads overlay trust properties from MindMap people subgraph.

### Drive adaptation → shouldCompareSocially behavioral gating
**Status: GAP** — ManorContextStrategy.shouldCompareSocially() reads from **static SocialConfig.Drive records**, not from adapted drive intensities in MindMap nodes. D22's "Adapted intensity requirement" says: "shouldCompareSocially() must read adapted drive intensities from MindMap nodes, not from static SocialConfig.Drive records." This was explicitly called out in the design but never implemented.

**Impact:** Drive adaptation (#66) modifies intensities that are rendered (CharacterDrivePromptSection) but never influence behavioral gating. A character whose scheming drive has decayed to 0.1 from consistently negative outcomes still gets social awareness rendering based on the original static intensity of 0.9. Adaptation is functionally inert for behavioral decisions.

### shouldDisclose / shouldCooperate (D22)
**Status: GAP — defined but never called.** These methods exist in ManorContextStrategy with tests, but are never called from any production code. The stage-gated behavioral gates designed in D22 are dead code.

---

## 9. Opportunities

### 9a. CognitionCore pipeline wiring (CRITICAL — new issue required)
The entire blocks-level observation pipeline is disconnected from wacky-manor. This is the single biggest integration gap. All of Phase C's rendering work (CharacterDrivePromptSection, NeedsPyramidPromptSection) is invisible to characters. Fixing this would immediately give characters:
- Mood awareness
- SDT drive awareness
- Character drive awareness (adapted)
- Needs pyramid feelings
- Narrative arc
- Strategy reflections
- Goal awareness in dialogue

### 9b. shouldCompareSocially reads adapted intensities (issue from D22)
Wire ManorContextStrategy to read from MindMap drive-intensity nodes instead of static SocialConfig. This makes drive adaptation affect behavioral gating.

### 9c. shouldDisclose / shouldCooperate wiring (issue from D22)
Wire the stage-gated behavioral gates into CharacterCognition or the prompt construction path so stage influences character behavior beyond just perception rendering.

### 9d. CognitivePreambleGenerator updates (from D3)
- Add two-system disambiguation text
- Remove state-dependent "some are stronger" language
- Check characterDrivesEnabled and needsPyramidEnabled flags

### 9e. CognitionConfig.all() defaults
- `characterDrivesEnabled` should be true in `all()`
- Alternatively, ScenarioOrchestrator should explicitly enable the subsystems it needs rather than relying on `all()`

### 9f. Norms are static
Norms are loaded from YAML and never evolve. Characters can't learn new social rules from experience. This could be a future Phase D feature — norm discovery from repeated interaction patterns, similar to how beliefs are revised from contradicting evidence.

### 9g. CognitionCore.tick() is never called in wacky-manor
The CognitionCore on the ScenarioOrchestrator is created with `CognitionConfig.none().with("goals", true)` and is used only for goal orchestration. `CognitionCore.tick()` — which updates mood, SDT drives, narrative, strategy, mental model, user model — is never called. All these orchestrators are null or inactive.

### 9h. Performance consideration
CharacterCognition.renderSocialAwareness() and renderTrustSections() both call `mindMapStore.listSubgraphs()` and `mindMapStore.nodesIn()` separately, scanning the same subgraphs twice per character per tick. These could be consolidated into a single scan.

### 9i. Blocks feature branch unmerged
The `issue-283-directive-minimal-architecture` branch in blocks has 14 commits (the entire Phase C blocks implementation) that are not merged to blocks' main. Until this merges, the installed blocks-core jar doesn't include: CognitivePreambleGenerator, CognitiveSystemPromptRenderer, DriveAdaptationPhase, BeliefRevisionPhase, RelationshipStagePhase, CharacterDrivePromptSection, NeedsPyramidPromptSection, NeedTier, NeedSatisfactionConfig, or the CognitionConfig extensions. Wacky-manor's pom.xml references the 0.2-SNAPSHOT which resolves to whatever's in the local Maven repo — if blocks-core was installed from the feature branch, it works; if someone runs `mvn install` from blocks main, all Phase C features disappear.

---

## 10. CDI Discovery — Consolidation Phase Registration

### Status: **CRITICAL GAP — Blocks consolidation phases are not CDI beans**

Platform consolidation phases (e.g., `ExperienceConsolidationPhase`) have `@ApplicationScoped @Priority(15)` — they are CDI beans discovered by `ConsolidationScheduler` via `Instance<ConsolidationPhase>`.

The three blocks-level consolidation phases on the feature branch have `@Priority` but **NOT** `@ApplicationScoped`:
- `DriveAdaptationPhase` — `@Priority(17)` only
- `BeliefRevisionPhase` — `@Priority(16)` only
- `RelationshipStagePhase` — `@Priority(18)` only

**Consequence:** These phases are never discovered by CDI, never instantiated by the container, and never run. Drive adaptation, belief revision, need satisfaction tracking, and relationship stage computation are dead code in the running application.

No CDI producer for these phases exists in `SocialCognitionDefaultBeans`, `BlocksBeans`, or anywhere else.

**Fix:** Either:
1. Add `@ApplicationScoped` to each phase class (they need constructor injection for MindMapStore, reinforcement config, etc.)
2. Add `@Produces @ApplicationScoped` producers in a blocks-core beans class that constructs each phase with its dependencies

Option 2 is likely necessary because the phases have complex constructor dependencies (reinforcement maps from per-character YAML, tier mappings from the application) that don't map cleanly to CDI injection.

**Intersection with neocortex#362 (orphaned SPIs):** This gap means the blocks ConsolidationPhase implementations are effectively orphaned — the SPI exists, implementations exist, but nothing connects them to the CDI runtime.

---

## 11. GA Audit Intersections (neocortex#355)

### #362 — Orphaned SPIs
- `AgentTrustProvider` — used by wacky-manor via `TrustEvolutionConfigProducer` CDI bean. Not orphaned.
- The consolidation phase implementations themselves are effectively orphaned (see §10).

### #361 — mindmap→cognitive-index upward dependency
- `CognitionCore` imports from both `neocortex.mindmap` (MindMapStore) and `neocortex.cognitive.index` (CognitiveProfile). CharacterCognition in wacky-manor also imports both. This is the upstream→downstream pattern flagged by #361. Not a blocks issue per se but the wacky-manor integration crosses this boundary.

### #363 — API consistency
- ConsolidationPhase SPI: `run(String tenantId, List<String> subgraphPriority)` — consistent across all implementations.
- The `subgraphPriority` parameter is never populated by any caller in wacky-manor (always `List.of()`). It may be dead API surface.

### #367 — SPI completeness (@DefaultBean + contract tests)
- `NeedTierMappingProvider` — no `@DefaultBean`. Applications must provide an implementation or the phase silently skips (null check). Should have a no-op default that returns empty map.
- `NeedSatisfactionConfig` — Smallrye ConfigMapping, auto-discovered. No @DefaultBean needed.
- `DriveReinforcementConfig` — not a formal SPI. The reinforcement map is passed as a constructor argument (`Map<String, Map<String, List<DriveReinforcementEntry>>>`). No CDI contract.

### ConfidenceDecayDecorator pattern
- Not directly relevant to the cognitive pipeline, but the pattern (CDI bean exists but is silently inactive) applies to the consolidation phases (§10). The three blocks phases are the same pattern — they exist, they implement the SPI, but CDI never discovers them.

---

## Summary of Findings

| # | Finding | Severity | Area |
|---|---------|----------|------|
| 1 | Blocks consolidation phases lack @ApplicationScoped — never instantiated by CDI, never run | **CRITICAL** | §10 |
| 2 | CognitionCore.promptSections() never called from wacky-manor — entire blocks observation pipeline orphaned | **CRITICAL** | §3 |
| 3 | CognitionCore.tick() never called — mood, SDT drives, narrative, strategy, mental model all inactive | **HIGH** | §9g |
| 4 | shouldCompareSocially reads static drive intensities, not adapted ones (D22 requirement) | **HIGH** | §8 |
| 5 | Blocks feature branch has 14 unmerged commits — Phase C blocks code not on main | **HIGH** | §9i |
| 6 | shouldDisclose/shouldCooperate defined but never called from production code | **MEDIUM** | §8 |
| 7 | CognitivePreambleGenerator missing drive disambiguation text (D3 revision) | **MEDIUM** | §4 |
| 8 | CognitivePreambleGenerator has state-dependent language violating caching constraint | **MEDIUM** | §4 |
| 9 | CognitionConfig.all() returns characterDrivesEnabled=false | **MEDIUM** | §4 |
| 10 | Goals not rendered in observation pipeline — characters can't reason about own goals in dialogue | **MEDIUM** | §7 |
| 11 | NeedTierMappingProvider has no @DefaultBean — missing SPI default | **MEDIUM** | §11 |
| 12 | Norms are static — no evolution mechanism | **LOW** | §9f |
| 13 | Duplicate MindMapStore scans in CharacterCognition rendering | **LOW** | §9h |
| 14 | CuriosityRefreshPhase/SchemaDiscoveryPhase priority overlap risk with blocks phases | **LOW** | §1 |
| 15 | subgraphPriority parameter never populated — potentially dead API surface | **LOW** | §11 |
