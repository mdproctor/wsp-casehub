# Decisions — #118 Drive Integration Architecture

## D1: Drive → CAPS Mapping Mechanism

**Choice:** CAPS connection weight modulation via disposition_modifiers
**Supersedes:** PAD-space geometric derivation (PAD Bridge) — rejected after decision review R1-01 identified PAD as an emotion model, not a motivation bridge. Domain mismatch made PAD profiles for CAPS nodes Tier 2-3 at best.
**Alternatives:**
- PAD bridge (input activation via geometric similarity) — domain mismatch: PAD is for emotions not motivations; requires ~30 CAPS node PAD profiles most of which lack published values; adds second activation pathway with unspecified interaction semantics
- Hand-crafted config mapping (caps-activations per drive type) — arbitrary, not traceable
- New CAPS input nodes per drive type — conflates motivational and perceptual constructs
- Reverse reinforcement mapping — indirect, sparse activation
**Rationale:** CAPS already supports disposition-based weight modulation via `disposition_modifiers` in caps-topology.yaml. Personality dimensions (socialOrient, rulefollowing) already scale connection weights — drives extend this mechanism. Drives modulate HOW situations are processed (connection weight bias), not WHAT situations are perceived (input activation). This is psychologically accurate: motivation creates processing biases, not phantom perceptions. No CapsNode schema extension needed. No PAD profiles for CAPS nodes needed. Uses `CapsEngine.updateWeights()` for dynamic modulation as drive intensities change.
**Drive-to-connection mapping:** Each drive type maps to connection groups it strengthens/weakens, defined universally (same for all characters). The mapping is from drives to processing paths (input→mediating→output connections), not to perceptual nodes. Example: gallantry strengthens social_input→approach paths, weakens social_input→withdraw paths. Mapping defined alongside drive type definitions, literature-sourced per D2 tiers.
**Dynamic modulation:** Current disposition_modifiers are static (set at agent creation). Drive-based modifiers are dynamic — recalculated each tick from current drive intensities via `CapsEngine.updateWeights()`. Weight change = `connection_modifier × drive_intensity × dampening_factor`. Dampening configurable via `manor.caps.drive-dampening` (default 0.3).
**Trade-offs:** Still requires defining which connections each drive type modulates (a mapping). But this mapping is more natural than PAD-to-input: "gallantry biases prosocial processing paths" is more defensible than "gallantry activates the acceptance input node." Mapping grounded in drive's psychological meaning, not PAD geometry.
**Sources:** Mischel & Shoda (1995) CAPS theory, caps-topology.yaml disposition_modifiers mechanism, CapsEngine.updateWeights()
**Surfaced by:** Decision review R1-01
**Exploration:** deep-analysis
**Status:** revised (R2: weight modulation replaces PAD bridge)

## D2: Model Value Sourcing Principle

**Choice:** All model values (PAD profiles, similarity thresholds, dampening factors) must be traceable to published psychological literature — with sourcing tier documented per value
**Alternatives:**
- Hand-tuned values based on intuition — no derivation, not reproducible
- LLM-generated values — plausible but not citable
- Single "literature-sourced" requirement without tiers — aspirational for constructs where published norms don't exist
**Rationale:** The system is a computational implementation of established psychological models. Values must be reproducible, defensible, and traceable. Three sourcing tiers acknowledge the practical reality that published PAD norms cover emotion words and concepts (ANEW, GRID) but not all CAPS constructs or motivational drives:
  - **Tier 1 — Direct:** Published normed values (e.g., ANEW word ratings, GRID emotion profiles). Cite source and table/page.
  - **Tier 2 — Derived:** Derived from published research with documented derivation chain (e.g., secure_attachment PAD from attachment theory's characterization of secure base affect). Cite source theory and derivation reasoning.
  - **Tier 3 — Constrained:** Researcher judgment within literature-informed bounds (e.g., PAD profile for "competence_recognition" bounded by published achievement emotion profiles). Cite bounding sources and state that it is researcher-derived.
All three tiers are acceptable; Tier 3 values should be flagged for empirical validation through D4a's micro-experiments.
**Trade-offs:** Requires literature review effort. Most CAPS input nodes will be Tier 2 or 3. This is honest about the derivation quality rather than claiming false precision.
**Sources:** Design principle from user, Bradley & Lang (1999) ANEW, Fontaine et al. (2007) GRID
**Exploration:** quick
**Status:** revised (R1: three-tier sourcing replaces single "literature-sourced" requirement)

## D3: Drive → Appraisal Wiring

**Choice:** Call-site merge — AppraisalTickParticipant merges SDT drives (from DriveOrchestrator) with character drives (from MindMapStore drive-intensity nodes) at the point of use. Character drives use keyword lists for relevance detection.
**Alternatives:**
- DriveProfile.allDrives() extension — creates wrong-layer dependency (cognition-api → mindmap-api)
- Separate AppraisalTickParticipant path — keeps SDT and character drive paths separated but duplicates wiring
- Unified drive model — replace dual drive system entirely; largest change, conflicts with Phase D SDT design
**Rationale:** DriveProfile lives in cognition-api (a pure API module). MindMap drive-intensity nodes are in MindMapStore (mindmap-api runtime). Merging in DriveProfile.allDrives() would create a cross-layer dependency. AppraisalTickParticipant already has DriveOrchestrator injected; adding MindMapStore injection and merging the two drive lists preserves layer boundaries. DriveCategory.CHARACTER enum already exists and distinguishes character from SDT drives in the merged list.
**Relevance detection:** Character drives require keyword lists (not drive-name substring matching). The Drive record's trigger field is extended from single String to List<String> to hold situationally-relevant terms (e.g., scheming → ["plan", "trap", "ambush", "outwit", "scheme"]). RelevanceCheck.computeDriveRelevance() matches these keywords against the narrative using the same pattern as RuleBasedSituationClassifier. SDT drives ("curiosity", "competence", "affiliation", "autonomy") naturally appear in narratives; character drives need curated keyword lists in social-config.yaml.
**Sources:** AppraisalTickParticipant.java, RelevanceCheck.java, RuleBasedSituationClassifier.java, DriveProfile.java (cognition-api module boundaries)
**Exploration:** quick
**Status:** revised (R1: call-site merge replaces DriveProfile extension; keyword lists replace name-only matching)

## D4: Emergence Philosophy

**Choice:** Always opt for inferred emergence. Drives influence perception and feeling; behavior emerges from the LLM interpreting enriched context. Use micro experiments to validate each integration point empirically.
**Alternatives:**
- Prescriptive/deterministic — drives directly script behavioral output (rejected: violates emergence constraints)
- Pure emergence with no computational influence — drives have no effect on any pipeline (rejected: adaptation becomes computation with no output)
**Rationale:** The architecture should create memories, use emotions, and use the underlying character profile to infer and emerge behavior. Explicit behavioral declarations (somatic cues, tendency text) should be removed. Computational influence is acceptable when the derivation is principled, traceable, and literature-grounded — not when it's arbitrary assignment.
**Trade-offs:** Emergent behavior is harder to guarantee/predict. Micro experiments needed to verify each integration point actually produces measurable behavioral change.
**Sources:** Design principle from user
**Exploration:** deep-analysis
**Status:** captured

## D4a: Runtime Configurability Principle

**Choice:** Every design choice involving alternative strategies or "how much" parameters must be exposed as runtime configuration. Support hybrid modes where strategies compose. Tune empirically through training/eval tests. Never bake in a single approach if alternatives are viable.
**Alternatives:**
- Fixed implementations chosen at design time — loses experimental flexibility, assumes we can predict the right answer before observing behavior
**Rationale:** This is research. The system is a platform for discovering what works, not a product with known requirements. Every knob that can be turned should be turnable. The optimal configuration is discovered empirically, not designed theoretically.
**Applies to:** D1 (dampening factor, similarity threshold, activation scaling), D3 (modulation strength, which SEC checks use character drives), D5 (PAD similarity function, centroid decay rate), D6 (salience strategies), rendering (somatic text on/off alongside CAPS attractors)
**Config pattern:** Quarkus config properties (manor.caps.dampening, manor.appraisal.character-drive-weight, manor.salience.*, etc.)
**Sources:** Design principle from user — empirical over upfront design
**Exploration:** quick
**Status:** captured

## D5: Memory-Drive Association Mechanism

**Choice:** PAD proximity — associate formation memories with drives by geometric similarity in PAD space, with initial PAD centroids for cold start
**Alternatives:**
- Explicit YAML field (associated-drives per memory) — manually assigned, not emergent
- Content-inferred via LLM — emergent but expensive (LLM call per memory per tick) and non-deterministic
**Rationale:** Formation memories already have PAD values (pleasure, arousal, dominance in social-config.yaml). Drives have PAD centroids (from D1). Similarity between memory PAD and drive PAD centroid determines association strength. Derived, not declared. Consistent with PAD bridge theme.
**Cold-start strategy:** At cold start (no reinforcement history), drive PAD centroids are undefined. Initial PAD profiles are specified in social-config.yaml drive entries (e.g., scheming: pleasure=0.7, arousal=0.5, dominance=0.9 — derived from the character's formation memories that shaped the drive). These seed values serve as the centroid until reinforcement history accumulates, at which point the centroid is a weighted blend of initial values and experienced PAD (initial weight decays via manor.drives.initial-pad-decay config). This ensures formation memories — the character's backstory — are MOST relevant at cold start and gradually yield to runtime experience.
**Trade-offs:** PAD space is 3D so some memories with different narrative content but similar PAD profiles may associate with the same drives. Acceptable — psychologically similar experiences SHOULD associate similarly. Initial PAD profiles require per-drive authoring in social-config.yaml.
**Sources:** Existing FormationMemory PAD values in social-config.yaml, D1 PAD bridge
**Depends on:** D2 (literature sourcing for initial PAD profiles). Note: D1 revision to weight modulation decoupled D5 from D1 — memory-drive association uses formation memory PAD values + drive initial PAD profiles independently of CAPS mapping mechanism.
**Exploration:** quick
**Status:** revised (R1: cold-start strategy with initial PAD profiles in social-config.yaml)

## D6: Memory Salience Rendering Strategy

**Choice:** All three strategies (prominence ordering, threshold filtering, enrichment injection) available at runtime as tunable global settings. Optimal hybridization discovered empirically through micro experiments.
**Alternatives:**
- Pick one strategy upfront — loses experimental flexibility
**Rationale:** This is research. The right combination of salience strategies is an empirical question, not a design question. Runtime configurability lets us isolate each variable's contribution and find the optimal blend. Same pattern as CognitionConfig's progressive activation levels.
**Trade-offs:** More implementation work than a single strategy. But the experimental value far outweighs the cost — and the strategies are independently simple.
**Config shape:** `manor.salience.ordering=true`, `manor.salience.threshold=0.3`, `manor.salience.enrichment=false`
**Sources:** Design principle from user — empirical over upfront design
**Exploration:** quick
**Status:** captured

## D7: Design Scope

**Choice:** Full architecture — all 3 integration points (Drive→CAPS, Drive→Appraisal, Drive→Memory Salience) + content changes (enriched formation memories) + rendering cleanup (remove somatic drives from renderInternalState). Implementation follows strict phasing order.
**Alternatives:**
- CAPS integration + rendering only — defers appraisal and memory salience
- Everything including detailed eval strategy — comprehensive but significantly larger spec
**Rationale:** The three integration points are architecturally related and should be designed as a coherent system. Implementation is phased to prevent regression.
**Implementation phasing (strict order):**
  1. Wire computational pipeline (D1 PAD bridge, D3 appraisal wiring, D5 memory association)
  2. Verify each integration point via D4a micro-experiments — drive expression must be measurably present
  3. THEN remove prescriptive rendering (somatic drive text from renderInternalState)
Rendering cleanup MUST NOT precede verified computational pipeline delivery. Issue #115's data showed gloating collapsed -1.52 when prescriptive text was removed without computational replacement. The same failure mode must not repeat.
**Trade-offs:** Larger design scope means longer brainstorming, but phased implementation keeps each step testable and prevents regression.
**Sources:** Issue #118 implementation order, Issue #115 regression data
**Exploration:** quick
**Status:** revised (R1: explicit phasing order to prevent #115-style regression)

## D8: CAPS Topology Stability Under Drive-Based Activation

**Choice:** CAPS topology (connection weights, convergence parameters, distortion mechanics) remains unchanged. Drive-based activation bias is managed through dampening and clamping, not topology retuning.
**Alternatives:**
- Retune convergence parameters (max_iterations, epsilon) for persistent drive activations — premature without empirical evidence of settling problems
- Separate settling parameters for drive-biased vs pure-situation activation — unnecessary complexity, existing clamping handles magnitude
**Rationale:** Drive activations are persistent low-level biases (dampened to ~30% of raw value via D1's dampening config), structurally different from event-triggered situation activations. Existing safeguards — activation clamping [-1, 1], convergence epsilon 0.001, max 100 iterations — are designed for arbitrary input patterns and handle additive bias without modification. If micro-experiments (D4a) reveal settling anomalies, the dampening factor and activation clamp are the first tuning targets, not the topology itself.
**Trade-offs:** If drive bias causes unexpected attractor patterns, the fix is dampening/config tuning rather than topology redesign. D4a's runtime configurability ensures this is adjustable without code changes.
**Sources:** caps-topology.yaml convergence parameters, CapsEngine.settle() interface
**Surfaced by:** R1-09 (reviewer identified implicit decision)
**Exploration:** quick
**Status:** captured
