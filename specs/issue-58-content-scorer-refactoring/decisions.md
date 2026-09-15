## D1: ContentScorer as a new interface alongside ConfidenceScorer/GraduationScorer

**Choice:** Introduce `ContentScorer` + `ScoreableContent` in neocortex-memory-api as a new domain-agnostic scoring interface. SurpriseScorer/ArousalScorer implement both ContentScorer and ConfidenceScorer (adapter pattern). ManorGraduationScorer composes ContentScorer instances for graduation.
**Alternatives:**
- Refactor ConfidenceScorer itself to operate on ScoreableContent — eliminates ConfidenceScorer's CBR-specific signature but large blast radius across blocks YAML pipeline, MemoryHygieneOrchestrator, CompositeConfidenceScorer, WeightedScorer, ConfidenceScorerRegistry
- Unify ConfidenceScorer and GraduationScorer into a single interface — cleanest long-term but requires moving the unified type to memory-api and updating all consumers in both repos
**Rationale:** Approach A adds the abstraction without disrupting either existing scoring pipeline. SurpriseScorer/ArousalScorer implement both interfaces — ContentScorer for the new graduation path, ConfidenceScorer as a thin adapter for backward compat. Zero changes needed in MemoryHygieneOrchestrator, CompositeConfidenceScorer, or the YAML pipeline.
**Trade-offs:** Two scoring interfaces coexist (ContentScorer and ConfidenceScorer). Acceptable pre-release — can unify later if the duplication becomes friction.
**Sources:** io.casehub.blocks.memory.ConfidenceScorer, io.casehub.neocortex.memory.experience.GraduationScorer, GE-20260912-be7c74 (three-tier memory model), casehubio/examples#58
**Exploration:** quick
**Status:** captured

## D2: actionImportance is a new game-specific ContentScorer, not a renamed SurpriseScorer

**Choice:** ActionImportanceScorer is a new ContentScorer in examples/wacky-manor that scores event-type attributes from Memory metadata. SurpriseScorer is refactored to ContentScorer for reuse but is not in the graduation formula.
**Alternatives:**
- Rename SurpriseScorer to ActionImportanceScorer and adapt its logic — conflates two different scoring heuristics (feature diversity vs action significance)
- Use SurpriseScorer directly in the formula — surprise (feature diversity) doesn't map to action importance semantically
**Rationale:** actionImportance scores the game-level significance of events (conflict resolution > idle chat), which is conceptually distinct from surprise (feature diversity). SurpriseScorer remains available as a composable ContentScorer for other uses.
**Trade-offs:** One more scorer class in examples, but semantics are clearer.
**Sources:** casehubio/examples#58 score formula
**Exploration:** quick
**Status:** captured

## D3: ScoreableContent extraction — static factory on ScoreableContent for Memory, inline in blocks for ScoredCbrCase

**Choice:** ScoreableContent.fromMemory(Memory) as a static factory in memory-api. ScoredCbrCase extraction lives in the blocks adapter (SurpriseScorer/ArousalScorer ConfidenceScorer methods) since it depends on FeatureValue stringification.
**Alternatives:**
- ScoreableContent.fromScoredCbrCase() in memory-api — couples ScoreableContent to CbrCase details (FeatureValue types)
- Separate extractor functions/interfaces — over-engineered for two extraction paths
**Rationale:** Memory is in memory-api alongside ScoreableContent, so fromMemory() is natural. ScoredCbrCase extraction uses FeatureValue (a CBR-specific type), so it belongs in blocks where the dependency already exists.
**Trade-offs:** Extraction logic split across two locations. Acceptable — each location has the right dependencies.
**Sources:** io.casehub.neocortex.memory.Memory, io.casehub.neocortex.memory.cbr.ScoredCbrCase, io.casehub.neocortex.memory.cbr.CbrCase
**Exploration:** quick
**Status:** captured
