# Decisions — #118 Drive Integration Architecture

## D1: Drive → CAPS Mapping Mechanism

**Choice:** PAD-space geometric derivation (PAD Bridge)
**Alternatives:**
- Hand-crafted config mapping (caps-activations per drive type in YAML) — arbitrary, not traceable, no derivation chain
- New CAPS input nodes per drive type — requires topology changes, conflates motivational and perceptual constructs
- Reverse reinforcement mapping — indirect, sparse activation, still requires event-type-to-CAPS mapping
**Rationale:** PAD (Pleasure-Arousal-Dominance) space is already the shared mathematical foundation across drives, formation memories, and affect theory. CAPS input nodes represent psychological constructs with inherent PAD profiles derivable from published literature. The mapping becomes geometric similarity in PAD space — computed, not assigned. The derivation chain is fully traceable: drive reinforcement history → PAD centroid → similarity to CAPS node PAD profiles → activation weight.
**Trade-offs:** Requires PAD profiles for all ~30 CAPS input nodes (one-time effort, literature-grounded). PAD space is low-dimensional (3D) so drives with similar PAD profiles will produce similar CAPS activations — acceptable because psychologically similar drives SHOULD have similar behavioral influence.
**Sources:** Mehrabian (1996) PAD model, Bradley & Lang (1999) ANEW, Fontaine et al. (2007) GRID, Mischel & Shoda (1995) CAPS theory
**Exploration:** deep-analysis
**Status:** captured

## D2: Model Value Sourcing Principle

**Choice:** All model values (PAD profiles, similarity thresholds, dampening factors) must be sourced from published psychological literature with citations
**Alternatives:**
- Hand-tuned values based on intuition — no derivation, not reproducible
- LLM-generated values — plausible but not citable
**Rationale:** The system is a computational implementation of established psychological models. Values sourced from literature are reproducible, defensible, and traceable. Where a construct lacks a direct published PAD value, derive from the closest published construct with the derivation documented.
**Trade-offs:** Requires literature review effort. Some constructs may need interpolation from related published values.
**Sources:** Design principle from user
**Exploration:** quick
**Status:** captured

## D3: Drive → Appraisal Wiring

**Choice:** DriveProfile adapter — extend DriveProfile.allDrives() to merge SDT drives + character drives (from MindMap drive-intensity nodes)
**Alternatives:**
- Separate AppraisalTickParticipant path — keeps SDT and character drive paths separated but duplicates wiring
- Unified drive model — replace dual drive system entirely; largest change, conflicts with Phase D SDT design
**Rationale:** DriveCategory.CHARACTER enum already exists but was never wired. RelevanceCheck already keyword-matches on drive.name() and drive.trigger(). Minimal code change to connect character drives through existing appraisal infrastructure.
**Trade-offs:** Couples character drives into the SDT drive list. RelevanceCheck's keyword matching may need tuning for character drive names (gallantry, proving-worth, protection vs curiosity, competence, affiliation, autonomy).
**Sources:** AppraisalTickParticipant.java, SchererAppraisalStrategy.java, DriveProfile.java
**Exploration:** quick
**Status:** captured

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

## D5: Design Scope

**Choice:** Full architecture — all 3 integration points (Drive→CAPS, Drive→Appraisal, Drive→Memory Salience) + content changes (enriched formation memories) + rendering cleanup (remove somatic drives from renderInternalState)
**Alternatives:**
- CAPS integration + rendering only — defers appraisal and memory salience
- Everything including detailed eval strategy — comprehensive but significantly larger spec
**Rationale:** The three integration points are architecturally related and should be designed as a coherent system. Implementation can be phased.
**Trade-offs:** Larger design scope means longer brainstorming, but phased implementation keeps each step testable.
**Sources:** Issue #118 implementation order
**Exploration:** quick
**Status:** captured
