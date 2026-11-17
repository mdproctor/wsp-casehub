## D1: Pipeline placement for situational personality modulator

**Choice:** New tick participant — `SituationalPersonalityParticipant`
**Alternatives:**
- Extend PersonalityEvolutionOrchestrator — keeps personality logic together but mixes slow (evolution) and fast (situational) time scales in one component
- Extend BehavioralSynthesisPhase — minimal new code but couples to consolidation cycle, not per-tick
**Rationale:** Clean separation of concerns. The modulator runs every tick, reads multi-signal state (CAPS attractors, mood PAD, drive profile, appraisal results), computes personality pull toward situational attractors, updates runtime personality state, and triggers CDE re-derivation. The rest of the pipeline is unaware.
**Trade-offs:** New component to maintain vs extending existing ones. More moving parts.
**Sources:** CAPS settling pipeline (BehavioralSynthesisPhase.java), PersonalityEvolutionOrchestrator.java, CognitiveDerivationEngine.java
**Exploration:** quick
**Status:** captured

## D2: Elastic band state representation

**Choice:** Home + elasticity — three values per facet: `currentFacets` (where the personality IS), `homeFacets` (where it gravitates when no situation active), `elasticity` (how well it returns, 0.0–1.0)
**Alternatives:**
- Exposure accumulator — simpler but home is always Eidos baseline, no permanent personality change modeled
- Dual drift rates — mathematically equivalent but no explicit elasticity parameter, less intuitive
**Rationale:** Clear physical meaning maps directly to the elastic band metaphor. Elasticity degrades with exposure intensity/duration, recovers during rest. Home drifts toward current at a rate proportional to (1 - elasticity). Three numbers per facet, all testable.
**Trade-offs:** More state to persist (3 values × 13 facets = 39 numbers per agent). But the state is interpretable and debuggable.
**Sources:** Elastic band metaphor from user, CAPS attractor model (CapsAttractor.java)
**Exploration:** quick
**Depends on:** D1 (pipeline placement determines who reads/writes this state)
**Status:** captured

## D3: CDE integration path

**Choice:** Modified DescriptorView — feed modulated facets into CDE through the existing DescriptorView → derive path
**Alternatives:**
- Post-CDE modification of CognitiveDefaults directly — bypasses CDE's derivation logic (PID-5→FFM→cognitive chains wouldn't run)
- Wrapper store between Eidos and bridge — clean abstraction but adds indirection layer
**Rationale:** CDE is oblivious. It receives different facet scores and derives cognitive defaults naturally through all 14+ pathways. The existing merge path (deriveAndMerge) handles explicit-wins-over-derived. All the PID-5→FFM→cognitive-parameter chains from #845 run on the modulated values without modification.
**Trade-offs:** CDE re-derivation happens every tick when situational state changes, not just on evolution events. Potential performance cost — but CDE is stateless and fast (pure arithmetic).
**Sources:** EidosCognitiveProfileBridge.toDescriptorView(), CDE.deriveAndMerge(), #845 design spec
**Exploration:** quick
**Depends on:** D1, D2 (tick participant computes effectiveFacets from elastic state)
**Status:** captured

## D4: Situational mode detection design

**Choice:** Weighted threshold classifier — weighted sum of CAPS attractors, mood PAD, drive states, appraisal results per mode, compared to threshold
**Alternatives:**
- Rule-based classifier — explicit if/then rules per mode. More readable but rigid, adding signals means more rules
- Secondary CAPS settling network — reuses settling algorithm but over-engineered for a classification task
**Rationale:** Each mode's detection is a pure function: weighted sum of already-resolved signals > threshold → mode active at that intensity. Weights and thresholds defined in the situational topology YAML alongside attractor profiles. Fully deterministic, tunable via eval results, testable in isolation.
**Trade-offs:** Less readable than explicit rules. Weight tuning requires eval runs to calibrate.
**Sources:** CAPS attractor detection (CapsSettler.extractAttractors), DriveCompetitionDetector.DRIVE_CAPS_MAPPINGS, MoodOrchestrator
**Exploration:** quick
**Depends on:** D1 (tick participant reads these signals)
**Status:** captured

## D5: Persistence of runtime personality state

**Choice:** New neocortex-local SituationalPersonalityStore (SQLite) — co-located with consumer, avoids cross-repo API changes
**Alternatives:**
- Extend DispositionSignalStore (eidos-api) — keeps all personality state in one place but the store's API operates on Jungian function terms, not PID-5 facets; this is a new storage domain, not a simple extension
- MindMap cognitive nodes — follows drive-intensity pattern but violates #845 principle ("MindMap stores runtime effects, not personality state")
**Rationale:** DispositionSignalStore's existing API is incompatible (Jungian terms vs PID-5 facets). A neocortex-local store avoids cross-repo coupling while keeping the data close to SituationalPersonalityParticipant. Schema: agent_id, facet_name, current_value, home_value, elasticity, elasticity_floor, last_updated.
**Trade-offs:** Personality runtime state now spans two stores (DispositionSignalStore for Jungian evolution, SituationalPersonalityStore for facet elasticity). But these are genuinely different domains.
**Sources:** DispositionSignalStore API (eidos-api), SqliteCapsEngine pattern (caps-engine), #845 design spec §Part 0c
**Exploration:** quick
**Depends on:** D2 (state representation defines what needs persisting)
**Status:** revised (decision review — cross-repo coupling concern)
