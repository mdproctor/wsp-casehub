# Decisions — Emotion Regulation (#590)

## D1: Strategy scope

**Choice:** All five Gross strategies — design for the complete process model
**Alternatives:**
- Three post-appraisal only (distraction, reappraisal, suppression) — simpler but incomplete model, paints into a corner if goal-level strategies need different abstractions later
- Two core only (reappraisal + suppression) — misses distraction which Petter et al. show dominates at high intensity
**Rationale:** Designing for the full model ensures the abstraction handles both post-appraisal (pipeline-level) and antecedent (goal-level) strategies cleanly. Implementation batched: Batch 1 = post-appraisal three, Batch 2 = goal-level two.
**Trade-offs:** Larger design scope; goal-level strategies touch GoalFormationStrategy which is a different pipeline integration point
**Sources:** Gross 1998 five-strategy process model; casehubio/neocortex#590 issue body
**Exploration:** quick
**Status:** captured

## D2: Computation approach

**Choice:** LLM-driven with literature-structured prompting. Bosse/Petter models provide the framework (strategy families, intensity thresholds, personality mappings) as structured prompt context; the LLM performs actual regulation reasoning. Mechanize later based on empirical evidence of what becomes formulaic.
**Alternatives:**
- Fully mechanical (Bosse differential equations + Petter RL selector) — deterministic and testable, but risks producing rigid/unconvincing regulation like prior keyword filtering issues
- Hybrid (mechanical selection, LLM rendering) — reasonable compromise but premature optimization before we know what quality looks like
**Rationale:** Quality first, optimize later. Published equations look clean but we don't know if they produce convincing character moments empirically. Prior bad experience with mechanical keyword filtering. The LLM gets the literature as structured input, not as a replacement for judgment.
**Trade-offs:** Non-deterministic initially; harder to unit test; higher token cost per tick. Mitigated by eval framework (drive expression eval can measure regulation quality).
**Sources:** Bosse et al. 2010 quantitative model; Petter et al. 2025 strategy choice; prior keyword filtering experience
**Exploration:** quick
**Status:** captured

## D3: LLM integration pattern

**Choice:** Dedicated RegulationGenerationPhase — LLM call in the regulation participant producing structured regulation output (strategy chosen, intensity dampening, narrative). Same pattern as DriveDescriptionGenerationPhase.
**Alternatives:**
- Prompt-only (no generation call) — add regulation to renderInternalState() prompt sections, let character LLM handle it implicitly. Cheaper but regulation becomes unobservable and untestable
- Multi-phase generation (separate calls for identification, selection, implementation) — most faithful to Gross 2015 cycle but 3x token cost per tick
**Rationale:** Follows established generation phase pattern. Produces observable, testable regulation output stored on MindMap. Single LLM call balances quality with cost. Strategy identification + selection can be combined in one prompt.
**Trade-offs:** One extra LLM call per tick; regulation narrative quality depends on prompt engineering
**Sources:** DriveDescriptionGenerationPhase pattern; Gross 2015 extended process model
**Exploration:** quick
**Status:** captured

## D4: Pipeline placement

**Choice:** New RegulationTickParticipant in DERIVED phase, after AppraisalTickParticipant
**Alternatives:**
- Inside AppraisalTickParticipant — conflates emotion construction with regulation, grows an already complex class
- New phase between DERIVED and TERMINAL — maximum isolation but adds pipeline complexity, and regulation needs data from within DERIVED (CAPS output)
**Rationale:** Same pattern as existing participants. Clean separation of concerns — appraisal constructs, regulation modulates. Reads AppraisalResult, writes RegulationResult to MindMap for downstream consumption by DriveOrchestrator and renderInternalState().
**Trade-offs:** Ordering dependency within DERIVED phase — must run after appraisal
**Sources:** CognitionCore tick phase architecture; AppraisalTickParticipant pattern
**Exploration:** quick
**Status:** captured

## D5: Personality-strategy connection

**Choice:** Both personality-trait tendencies and current-state intensity/load as structured prompt context. The LLM integrates them rather than a mechanical priority rule.
**Alternatives:**
- Personality-primary (traits set default, intensity overrides only at extremes) — more predictable per-character but may miss dynamic moments
- Intensity-primary (Petter model drives selection, personality tilts tie-breakers) — more responsive but characters feel personality-less in regulation
**Rationale:** Consistent with D2 (quality first). Both dimensions are empirically validated — personality tendencies (Soleimani & Kobti extension of Bosse) and intensity-driven selection (Petter 2025). Letting the LLM integrate avoids premature mechanical weighting.
**Trade-offs:** Less predictable than a hard rule; regulation may vary across runs for same personality+intensity
**Depends on:** D2 (computation approach)
**Sources:** Petter et al. 2025 intensity model; Big Five → regulation strategy literature; Soleimani & Kobti personality-parameterized Bosse model
**Exploration:** quick
**Status:** captured

## D6: Rendering in internal state

**Choice:** Regulation section + modified drives. A new 'Regulation' ObservationSection shows active regulation (strategy deployed, target emotion, effectiveness). Drive snapshot text is also modified to reflect dampening ("you feel the pull but hold it back").
**Alternatives:**
- Regulation section only — drives unchanged, regulation advisory only. Simpler but drives feel disconnected from regulation
- Modified drives only — regulation baked into drive descriptions. Saves prompt tokens but regulation becomes invisible as a distinct cognitive process
**Rationale:** Both channels reinforce the regulation signal. The separate section makes regulation observable and testable (eval can check for regulation presence). Modified drives ensure the character LLM doesn't ignore regulation when acting on drives.
**Trade-offs:** More prompt tokens per tick; requires coordination between regulation output and drive rendering
**Depends on:** D3 (RegulationGenerationPhase produces the content), D4 (pipeline placement determines data flow)
**Sources:** Issue #590 "regulation-colored drive text"; CognitionObservationBuilder.renderInternalState() architecture
**Exploration:** quick
**Status:** captured
