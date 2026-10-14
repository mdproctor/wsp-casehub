# Decisions — #115/#118 Drive Integration Architecture

## D1: Drive → CAPS Mapping Mechanism

**Choice:** Persistent dispositional activations — drives injected as persistent baseline activations in the CAPS network, with literature-derived mappings per drive type
**Supersedes:**
- R3: CAPS weight modulation via disposition_modifiers — rejected after literature review. Read & Miller (2002) and Read et al. (2018) establish that in computational CAPS models, motivational states operate as activations flowing through the weighted network, NOT as modulators that change connection weights. Weights represent stable personality traits from developmental/learning history; they don't fluctuate with momentary motivation. Weight changes are reserved for long-term learning (Hebbian + error-correcting).
- R1: PAD-space geometric derivation (PAD Bridge) — rejected after decision review R1-01 identified PAD as an emotion model, not a motivation bridge.
**Alternatives:**
- Weight modulation (R2) — intuitively appealing but contradicted by published CAPS computational models (Read & Miller 2002). Weights encode stable personality, not dynamic motivation.
- PAD bridge (R1) — domain mismatch: PAD is for emotions not motivations
- Hand-crafted config mapping — arbitrary, not traceable
- Reverse reinforcement mapping — indirect, sparse activation
**Rationale:** In Mischel & Shoda's CAPS theory, personality is the pattern of connection weights AND the pattern of activations across cognitive-affective units (CAUs). Goals and motivating forces are one of the five CAU types. Read & Miller's *Virtual Personalities* (2002) — the most elaborated computational CAPS implementation — treats goals as activations that spread through the weighted network to bias behavior selection. Drives in our system follow this model: they are persistent activations (re-injected each tick based on current drive intensity) that bias the network's settling dynamics alongside transient situation activations.
**Three activation types in the CAPS network:**
  - **Connection weights** = stable personality (from disposition_modifiers, changed only by long-term learning)
  - **Situation activations** = transient event-triggered signals (from SituationClassifier, existing)
  - **Drive activations** = persistent motivational bias (from DriveAdaptationPhase, NEW — re-injected each tick)
**Drive-to-activation mapping:** Each drive type maps to CAPS nodes it activates, defined universally. Mappings derived from published motivational psychology literature for each specific drive construct:
  - Gallantry/prosocial motivation: Batson (1991) empathy-altruism, Penner et al. (2005) prosocial behavior → `other_safety`, `acceptance`, `approach`
  - Proving-worth/achievement motivation: McClelland (1961) need for achievement → `mastery`, `competence_recognition`, `self_efficacy`
  - Protection/caregiving: George & Solomon (2008) caregiving behavioral system → `other_safety`, `physical_threat` sensitivity, `approach`
  - Scheming/Machiavellianism: Christie & Geis (1970) Machiavellianism → `dominance`, `strategic cognition`, `distrust`
  Each mapping is Tier 2 sourcing (D2) — derived from published research with documented derivation chain.
**Activation merge:** Drive activations are added to the `combinedActivations` map alongside situation activations before `capsEngine.settle()`. Merge strategy: additive with configurable dampening (`manor.caps.drive-dampening`, default 0.3). Drive activations are persistent low-level biases; situation activations are transient high-level signals. Existing activation clamping [-1, 1] prevents saturation.
**Trade-offs:** Still requires defining which CAPS nodes each drive type activates. But the mapping is now derived from specific motivational construct literature, not from PAD geometry or intuition. Adding a new drive type requires a literature review of that motivational construct to identify its activation pattern.
**Sources:** Mischel & Shoda (1995) CAPS theory, Read & Miller (2002) Virtual Personalities, Read et al. (2018) Personality Neuroscience, Batson (1991), McClelland (1961), George & Solomon (2008), Christie & Geis (1970)
**Exploration:** deep-analysis + literature-review
**Status:** revised (R3: persistent activations replaces weight modulation, per published CAPS computational models)

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

**Choice:** Motivational salience via PAD proximity — grounded in Klinger's current concerns theory, operationalized through PAD-space geometric similarity, with initial PAD centroids for cold start
**Alternatives:**
- Explicit YAML field (associated-drives per memory) — manually assigned, not emergent
- Content-inferred via LLM — emergent but expensive (LLM call per memory per tick) and non-deterministic
- Pure semantic/keyword matching — computationally cheap but doesn't capture affective similarity
**Rationale:** Klinger's current concerns theory (1971, 2004) establishes that active motivational states (goal pursuits, drives) directly increase cognitive salience for related stimuli — this is the theoretical foundation. PAD proximity is the computational operationalization: formation memories have PAD values, drives have PAD centroids, and geometric similarity determines association strength. Derived, not declared. Literature (Rubin & Talarico 2009) confirms dimensional models outperform categorical for autobiographical memory retrieval. Valence should be weighted more heavily than arousal/dominance in the similarity function (PMC mood-congruent memory review 2023).
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
  1. Wire computational pipeline (D1 persistent drive activations, D3 appraisal wiring, D5 memory association)
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

## D9: Formation Memory Concern Vocabulary

**Choice:** Drive type names as the `serves` vocabulary — individual-specific concerns connected to CAPS nodes via D1 mappings
**Alternatives:**
- CAPS mediating node names directly (self_worth, BAS_activation) — too granular, loses the motivational meaning. A memory serves a CONCERN (goal pursuit), not a network node.
- McClelland's three needs (achievement, affiliation, power) — too coarse. Gloating, scheming, and proving-worth all map to "achievement," losing the behavioral signature.
- SDT basic needs (autonomy, competence, relatedness) — also too coarse for distinguishing character-specific drive expression.
- Murray's 20 needs — right granularity but many didn't survive empirical testing. Not a maintained vocabulary.
- New bespoke concern vocabulary — unnecessary when drive types already serve this function.
**Rationale:** Singer (2013) and Klinger (1971) both establish that concerns are individual-specific, not universal categories. Drive type names ARE concerns in Klinger's sense — specific motivational goal-pursuits. D1 already maps each drive type to CAPS nodes with literature sourcing (Batson 1991 for gallantry, McClelland 1961 for proving-worth, etc.). Higher-level themes (insecurity, need-for-validation) are derivable compositions of drive types + facet configurations — they emerge from the graph, not from a separate vocabulary.
**Trade-offs:** Drive names are character-specific (PP has gallantry, DD doesn't). Cross-character comparison requires going through D1's CAPS node mappings. Acceptable — the behavioral signature IS character-specific per CAPS theory.
**Sources:** Singer et al. (2013) SDMs, Klinger (1971/2004) current concerns, D1 drive-to-CAPS mappings, D2 sourcing principle
**Exploration:** deep-analysis (literature review of Murray/McClelland/SDT/Singer/Klinger taxonomies)
**Status:** captured

## D10: Two-Layer Memory-Concern Association

**Choice:** Static structural association (`serves` annotations, Singer 2013) + dynamic activation modulation (PAD-proximity/Klinger salience, D5) as complementary mechanisms at different levels
**Alternatives:**
- `serves` replaces PAD-proximity — loses the dynamic, context-dependent activation that Conway's SMS and Klinger's theory predict
- `serves` is testing-only metadata — loses the structural association that Singer's SDM model establishes as a stable personality property
- PAD-proximity only (status quo per D5) — loses static testability and authorial intent documentation
**Rationale:** Empirically established as two distinct mechanisms. Singer (2013): "the pool of self-defining memories possesses a synchronic existence within narrative identity; they remain relatively static." Conway (2005): "the working self modulates access to long-term knowledge" based on current goals. Both are validated. `serves` captures the structural link (which drive a memory is about — this doesn't change with context). PAD-proximity/Klinger captures the dynamic activation (which memories surface depends on current drive intensity). Neither replaces the other.
**Trade-offs:** Two mechanisms to maintain. But they're at different layers — `serves` is authoring-time/testing, PAD-proximity is runtime. No implementation coupling.
**Sources:** Singer et al. (2013), Conway (2005), Conway & Pleydell-Pearce (2000), Klinger (1971)
**Exploration:** deep-analysis (literature review)
**Status:** captured

## D11: Annotation Syntax and Degree Scale

**Choice:** Inline `{{serves: drive|degree: text}}` annotations within formation memory episode text, with optional whole-memory `serves` field. Degree scale: faint, moderate, strong, defining.
**Alternatives:**
- Separate sub-memory records — adds structural complexity to the YAML, breaks the narrative flow
- Whole-memory-only labeling — too coarse for memories that serve multiple drives in different passages
- No degree — loses the intensity information that distinguishes a passing reference from a formative experience
**Rationale:** Inline annotations preserve the memory as a single readable narrative while enabling fine-grained drive association. The parser extracts annotations for testing (Level 0) and strips them for prompt rendering (LLM sees clean text). Degree scale from Pillemer (2003): directive memories "inform, guide, motivate, and inspire" — the degree captures which function the passage serves. "Defining" = the memory IS the formation of this concern (Pillemer's "originating event").
**Trade-offs:** Inline syntax adds parsing complexity. Acceptable — the parser is simple regex extraction.
**Sources:** Pillemer (2003) directive function taxonomy, Singer (2013) SDM properties (vivid, emotionally evocative, linked to enduring concerns)
**Exploration:** quick (iterative design with user)
**Status:** captured

## D12: Inference-Based Drive Differentiation

**Choice:** Memory-traced drive differentiation during consolidation — classify formation memories (not drive descriptions) through the vocabulary classifier to infer per-drive CAPS profiles, detect competition via pathway overlap, and derive S6-level grounded differentiation text from unique formation memories
**Supersedes:**
- Static `DRIVE_CAPS_MAPPINGS` in `DriveCompetitionDetector` — world-specific, not universal, not grounded in developmental history
- Generic `DIFFERENTIATION_INSTRUCTION` — operates at S3 (Semerari MAS-A), insufficient for differentiation per Barrett (2017) and Kashdan et al. (2015)
**Alternatives:**
- Classify drive descriptions (not memories) — §10.10.4 original vision. Rejected: drive descriptions are phenomenological prose; formation memories are situational episodes that match the vocabulary classifier's design. Barrett (2017): the concept is populated by its instances (memories), not its description.
- LLM-generated differentiation text during consolidation — emergent but expensive, non-deterministic, and unnecessary. The structured data (mediating node poles, formation memory episodes) is sufficient for template-based composition.
- Per-character static differentiation text — what Attempt 2 was (§10.10.2). Prescriptive, hand-authored, not universal.
**Rationale:** Three independent literature traditions converge: differentiation requires grounded conceptual knowledge, not generic metacognitive instruction. Barrett (2017): emotion concepts = populations of experiential instances; differentiation requires applying different concepts to the same activation. Kashdan et al. (2015): conceptual knowledge enables differentiation as a deployable skill; richer concepts → differentiation follows. Semerari et al. (2003): MAS-A hierarchy distinguishes S3 (generic: "attend to differences") from S6 (grounded: "this feeling traces through this experience"). MIT therapy (Dimaggio et al. 2015) validates biographical episodes as substrate for differentiation.
**Implementation:** `DriveProfileDerivationPhase` as a new `ConsolidationPhase`. Two-layer output: (1) inferred drive INPUT activation profiles stored as `DriveProfileInferred` nodes, consumed by `mergeDriveIntensities()` in place of static table; (2) grounded differentiation text stored as `DriveCompetitionInferred` nodes, consumed by preamble/render pipeline in place of generic instruction. Cold-start fallback to static table and generic instruction until first consolidation completes.
**Trade-offs:** More consolidation-time computation. But runs once per sleep cycle (5 min default), result is cached, and the existing vocabulary classifier and topology tracing are O(n) per memory. Requires formation memories to have `serves` annotations (already present from Layer B work). New worlds must annotate formation memories before inference works — but this is required for Layer B calibration anyway.
**Sources:** Barrett (2017) *How Emotions Are Made*, Kashdan et al. (2015) *Current Directions in Psychological Science* 24(1) 10-16, Semerari et al. (2003) *Clinical Psychology & Psychotherapy* 10 238-261, Dimaggio & Lysaker (2010), Dimaggio et al. (2015) MIT treatment manual, Vedernikova et al. (2021) *Frontiers in Psychology*
**Full design:** research doc §10.10.6-7
**Exploration:** deep-analysis (literature review of three traditions + first-principles design)
**Status:** captured
