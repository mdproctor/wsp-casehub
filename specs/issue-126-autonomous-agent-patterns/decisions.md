## D1: Pattern scope

**Choice:** Signal-driven orchestrator
**Alternatives:**
- Full feedback loop — bypasses eidos JPAF pipeline, computes trait deltas directly. Risks duplicating DefaultDispositionEvolution logic.
- Thin orchestrator — just wires SPIs with config, no signal translation SPI or safety rails. Too thin to be useful.
**Rationale:** Eidos already implements the full JPAF evaluation pipeline (DispositionSignalStore → DefaultDispositionHealth.probe() → DefaultDispositionEvolution.evaluate()). What's missing is the signal translation layer that maps real interaction outcomes (BehavioralSignal, QualitySignal, GoalOutcomeCounts) to disposition activation recordings, the orchestration that drives the periodic probe cycle, and safety rails the academic literature identifies as essential.
**Trade-offs:** Blocks depends on eidos SPI stability (DispositionSignalStore, DispositionHealth). If eidos's JPAF implementation changes, blocks must adapt.
**Sources:** DefaultDispositionHealth.java (eidos), DefaultDispositionEvolution.java (eidos), DispositionSignalStore.java (eidos), JPAF paper (arXiv:2601.10025), LLMPTBench (NeurIPS 2025), BFI-Adapt (arXiv:2608.06485)
**Exploration:** deep-analysis
**Status:** captured

## D2: Signal-to-activation mapping strategy

**Choice:** Layered mapping (profile-term-aware + vocabulary-structural)
**Alternatives:**
- Vocabulary-aware mapping only — maps via VocabularyTerm.opposite() for function-level activation. Only meaningful for JungianFunctionTerm; other 12 vocabulary frameworks return Optional.empty(), excluding their agents from personality evolution entirely.
- Axis-level mapping — maps to 5 DispositionAxis values. Architecturally incoherent: DefaultDispositionHealth.probe() iterates over dispositionProfile terms, not per-axis fields. Axis names recorded as activation terms would silently produce zero matches via counts.getOrDefault(dv.term(), 0).
- Raw event passthrough — passes raw domain events as activation terms, bypasses eidos JPAF infrastructure entirely.
**Rationale:** Two layers, both operating on dispositionProfile terms (the field consumed by DefaultDispositionHealth.probe()):
- Layer 1 (universal): Profile-term-aware mapping — the signal translator receives the AgentDescriptor, reads dispositionProfile terms, and maps domain events directly to those terms. Works for ANY vocabulary: Big Five agents activate "Openness"/"Conscientiousness", DISC agents activate "Dominance"/"Influence", Jungian agents activate "Ti"/"Ne". Every agent with a populated dispositionProfile participates in personality evolution.
- Layer 2 (enrichment): Vocabulary-structural mapping — for agents whose vocabulary supports structural navigation (e.g., JungianFunctionTerm.opposite()), the translator additionally infers indirect activations from vocabulary structure (e.g., negative event on Ti also activates shadow Fe). This enrichment requires vocabulary-specific structural knowledge not derivable from profile terms alone.
The translator already has access to AgentDescriptor (required for agentId/tenancyId in recordActivation), so reading dispositionProfile terms adds no dependencies.
**Trade-offs:** Two-layer mapping is more complex than either layer alone. Profile-term-aware mapping is direct but vocabulary-agnostic — agents with vocabulary structural enrichment get richer evolution trajectories (shadow function awareness). The mapping from domain event to specific profile term is SPI-defined, requiring per-domain-event-type implementations.
**Sources:** DispositionSignalStore.java (eidos), VocabularyTerm.opposite() (eidos API), AgentDisposition.dispositionProfile (eidos API), DefaultDispositionHealth.probe() (eidos runtime), GE-20260728-a53632 (vocabulary-generic structural navigation technique)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope)
**Status:** revised (R1-06, R1-07, R2-02: profile-term-aware mapping replaces axis-level mapping — aligns with probe pipeline's use of dispositionProfile terms)

## D3: Probe timing model

**Choice:** Batched periodic
**Alternatives:**
- Threshold-triggered — probe runs when activation count crosses threshold. Adapts to activity but no time-based decay anchor.
- Dual threshold + periodic — more complex, handles both high and low activity.
**Rationale:** Matches JPAF reflection model where evolution happens at episode boundaries, not per-event. Prevents micro-oscillation. Configurable frequency (per-case, daily, weekly) lets consumers tune to their domain. Time-based scheduling provides a natural anchor for decay application.
**Trade-offs:** Low-activity agents may accumulate stale activations between ticks. Mitigated by time-based decay running on the same schedule.
**Sources:** JPAF Algorithm 1 (reflection at episode boundaries), DefaultDispositionHealth.probe() (eidos)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope)
**Status:** captured

## D4: Asymmetric dampening strategy

**Choice:** Probe-time dampening with valence-tagged activations
**Alternatives:**
- Recording-time dampening — applies dampening factor at signal translation time, records already-dampened counts. Simple but permanently distorts data; retroactive recalibration impossible.
- Sentiment-aware decay rate — negative activations decay faster. Requires tracking signal source alongside counts, adds state to DispositionSignalStore.
- Fixed ratio cap — caps negative-to-positive activation ratio. Less granular than per-signal weighting.
**Rationale:** LLMPTBench (NeurIPS 2025) shows agentic frameworks exhibit exaggerated shifts under negative events. The signal translation SPI records activations with a valence tag (positive/negative). DispositionSignalStore gains a valence concept to distinguish positive from negative activation counts. DefaultDispositionHealth.probe() applies the dampening factor when computing effective weights: effectiveWeight = baseWeight + (positiveCount + negativeCount × dampeningFactor) × reinforcementDelta. The dampening factor is a configurable preference (per tenant), retroactively adjustable — changing the factor changes the next probe's computation without loss of historical data. This enables A/B testing of dampening factors across agent cohorts and analytical comparison of raw vs dampened signal distributions.
**Trade-offs:** Requires eidos SPI enhancement (valence parameter on recordActivation or separate positive/negative counts). More complex than recording-time dampening. Justified because preserving raw signal data with metadata is architecturally superior to permanently distorting it. Signal severity gradations are handled by the signal translation SPI (magnitude = how many activations to record) rather than the dampening mechanism (asymmetry = how to weight negative vs positive). Spec must document cascaded attenuation with default parameters: reinforcementDelta=0.06, overReinforcementThreshold=0.50, dampeningFactor=0.5.
**Sources:** LLMPTBench (NeurIPS 2025, OpenReview kVXePuKReA), BFI-Adapt (arXiv:2608.06485)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D2 (profile-term-aware + vocabulary-structural mapping)
**Status:** revised (R1-13: moved dampening from recording-time to probe-time to enable retroactive calibration)

## D5: Bounded displacement enforcement

**Choice:** L2 norm ceiling on effective weights
**Alternatives:**
- Per-function activation cap — caps max activation count per function. Doesn't account for different base weights.
- Weight-range clamping — clamps effective weights to JPAF differentiation ranges. Constrains output, not input.
- Weighted norm — assigns smaller penalty to dominant/auxiliary displacement (expected to be strong) and larger penalty to inferior displacement. More psychologically coherent but embeds Jungian theory into the safety rail.
**Rationale:** The L2 displacement metric is already exposed through the DispositionHealth SPI: DispositionStatus.Drifted(effectiveWeights, mostActivated, driftMagnitude) includes driftMagnitude which is the L2 distance. The orchestrator checks the ceiling via probe results. Between ticks, recording is gated by the halt flag (AtomicBoolean). No direct access to the private computeL2 method is needed.
**Halt flag state machine:** Complete coverage of the sealed DispositionStatus:
- Aligned → clear halt flag (no drift, recording continues)
- Drifted with driftMagnitude >= L2Ceiling → set halt flag (displacement too high)
- Drifted with driftMagnitude < L2Ceiling → clear halt flag (drift within bounds)
- EvolutionPending → trigger DispositionEvolution.evaluate(), then:
  - Evolved → apply new profile, call DispositionSignalStore.clear(), clear halt flag (new baseline = zero displacement)
  - Dampened → call DispositionSignalStore.decay() with factor, leave halt flag unchanged (next tick re-evaluates with reduced activations)
Note: probe() returns Drifted (not EvolutionPending) when the dominant exceeds overReinforcementThreshold, so EvolutionPending fires only when displacement has crossed an evolution threshold but NOT the over-reinforcement ceiling. EvolutionPending is an evolution trigger, not a halt condition.
**Complementary relationship with overReinforcementThreshold:** eidos's overReinforcementThreshold (default 0.50) is a per-function ceiling on the dominant's effective weight — fires when ONE function is disproportionately strong. The L2 ceiling is a global displacement bound — fires when the OVERALL profile drifts too far, catching distributed drift across multiple dimensions that no single per-function ceiling would detect. These are distinct geometric properties (per-dimension max vs global euclidean distance). Both mechanisms fire independently and serve complementary safety roles.
**Trade-offs:** L2 norm treats all axes equally — a shift in dominant function contributes the same as a shift in undifferentiated. This is intentional: the L2 ceiling is a safety rail, not a personality model. The JPAF pipeline handles psychological correctness (evolution detection); the L2 ceiling handles the "displacement accumulated without triggering evolution" case, which is pathological by definition. Bounded overshoot between probe ticks is an accepted trade-off of D3's periodic model.
**Sources:** DispositionStatus.Drifted.driftMagnitude (eidos API), JPAF Equations 1-3 (arXiv:2601.10025), DispositionPreferenceKeys.OVER_REINFORCEMENT_THRESHOLD (eidos, default 0.50)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D3 (batched periodic timing)
**Status:** revised (R1-16, R1-17: clarified L2 check mechanism via Drifted.driftMagnitude; R2-03: complete halt flag state machine covering all three DispositionStatus variants and post-evolution behavior)

## D6: Integration point

**Choice:** Application-driven CDI bean with tick() interface
**Alternatives:**
- Engine-coupled — driven by WorkOrchestrator or JobScheduler after task completion. Creates hard dependency between blocks and engine lifecycle. Blocks shouldn't know about engine.
- Standalone @Scheduled — CDI @Scheduled bean in blocks with fixed interval. Forces a specific scheduling framework; not all consuming apps use the same scheduler.
- Fully passive — triggered by signal recording (probe on every Nth recording). Contradicts D3's periodic model; couples probe cadence to interaction volume.
**Rationale:** The orchestrator is a CDI @ApplicationScoped bean that consuming apps wire into their lifecycle. It exposes a tick() method that applications call at their chosen cadence. This follows blocks' established pattern: SummarisationRunner, KeyedSummarisationRunner, and ObservationAccumulator all provide tick()/collect()/drain() methods that consuming apps invoke from their own scheduling mechanism (CDI @Scheduled, engine's JobScheduler, or explicit application code). Blocks is a library, not a framework — it doesn't own scheduling or lifecycle.
**Trade-offs:** Each consuming app must wire the tick call into its own scheduler. No out-of-the-box scheduling. Accepted because this is the same integration cost as SummarisationRunner and is consistent with blocks' architecture.
**Sources:** SummarisationRunner.tick() (blocks), KeyedSummarisationRunner.tick() (blocks), ObservationAccumulator.drain() (blocks)
**Exploration:** surfaced (R1-21: implicit decision made explicit)
**Depends on:** D1 (signal-driven orchestrator scope), D3 (batched periodic timing)
**Status:** captured

---

# MemoryHygiene (#120) Design Decisions

## D7: Scope — all five operations

**Choice:** Include all five operations: consolidation, forgetting, importance scoring, temporal versioning (§2.8), and cross-linking (§2.8)
**Alternatives:**
- Core three only (consolidation + forgetting + importance scoring) — smaller scope but defers P2-P3 fold-ins to separate issues
- Core three + integrity — adds integrity checks but still defers versioning and cross-linking
**Rationale:** Complete coverage avoids fragmented follow-up issues. The fold-ins (temporal versioning via existing supersession, cross-linking via feature maps) are lightweight additions that leverage existing store APIs.
**Trade-offs:** Larger implementation surface in a single issue
**Sources:** Research §2.2, §2.8; CbrCaseMemoryStore.supersede()/features()
**Exploration:** quick
**Status:** captured

## D8: Importance scoring placement — blocks SPIs

**Choice:** Define ImportanceScorer as a @FunctionalInterface SPI in blocks with ArousalScorer and SurpriseScorer as provided implementations
**Alternatives:**
- In neocortex as retention extensions — closer to memory data but couples blocks to neocortex internals
- Split SPI in blocks / impl in neocortex — clean separation but adds cross-module coordination
**Rationale:** Blocks owns the orchestration pattern; importance scoring is a pluggable strategy within that pattern. Consumers override via CDI. Consistent with how blocks defines other SPIs (ConvergencePolicy, AcceptancePolicy, etc.).
**Trade-offs:** Implementations can't directly access neocortex internals without injection
**Sources:** blocks scope criteria (CLAUDE.md); PersonalityEvolution TraitPressureSource pattern
**Exploration:** quick
**Status:** captured

## D9: Orchestrator architecture — tick + idle scheduler, single class

**Choice:** Single MemoryHygieneOrchestrator with dual entry points: tick() for on-demand and maintain() for idle-time full pipeline. Separate MemoryHygieneScheduler discovers agents/tenants and calls maintain().
**Alternatives:**
- Tick-only orchestrator — consistent with PersonalityEvolution but misses idle-time processing
- Batch scheduler only — closer to CbrRetentionScheduler but no on-demand path
- Separate orchestrator + scheduler with shared pipeline stages — more flexible but more types
**Rationale:** Tick and maintain share the same pipeline stages at different depths. Internal methods, not separate types. Mirrors InnerLifeOrchestrator's depth-controlled design. Scheduler is a thin wrapper that iterates agents/tenants.
**Trade-offs:** maintain() is a superset of tick() — some stage overlap, but avoids type explosion
**Sources:** PersonalityEvolutionOrchestrator; InnerLifeOrchestrator; CbrRetentionScheduler
**Exploration:** quick
**Status:** captured

## D10: Consolidation strategy — two-pass (summarise + reflect)

**Choice:** Two-pass consolidation: Pass 1 (ContentSummariser) merges raw memories into consolidated entries. Pass 2 (ReflectionOrchestrator) generates higher-level insights from consolidated set. Tick runs pass 1 only; idle scheduler runs both.
**Alternatives:**
- SummarisationRunner composition only — simpler but misses knowledge synthesis
- ReflectionOrchestrator delegation only — generates insights but doesn't merge/reduce raw memories
**Rationale:** Two passes serve different purposes: data reduction (100→20 memories) vs knowledge synthesis ("user raises security concerns after deployments"). Tick mode stays cheap (heuristic summariser, no LLM). Idle mode leverages Sleeptime concept for full LLM-powered reflection.
**Trade-offs:** Idle scheduler has higher LLM cost; reflection output stored as CBR cases ("reflection" case type)
**Sources:** Research §2.2 (LUFY, MemGPT Sleeptime); ContentSummariser; TieredContentSummariser; ReflectionOrchestrator
**Exploration:** deep-analysis
**Status:** captured

## D11: Eviction policy — composite score threshold

**Choice:** Compute composite retention score from importance (arousal, surprise), recency (TemporalDecay), scope (ScopeDecay), and trust. Evict below configurable threshold.
**Alternatives:**
- Staged filters (age → importance → capacity) — simpler but less nuanced
- Configurable EvictionStrategy SPI with composite default — adds SPI overhead for a single-strategy concern
**Rationale:** Single unified score is easier to tune and reason about. All signal types already produce [0,1] factors. Composite = weighted product of factors. Threshold is the one knob operators adjust.
**Trade-offs:** Weight tuning requires experimentation; no independent stage-level visibility
**Sources:** TemporalDecay; ScopeDecay; CbrRetentionPolicy; LUFY retention target (<10%)
**Exploration:** quick
**Status:** captured

## D12: Temporal versioning — supersession-based

**Choice:** Use existing CbrCaseMemoryStore.supersede(caseId, supersedingCaseId, reason) for temporal versioning. Old memory is marked superseded; new consolidated version replaces it. reinstate() available for rollback.
**Alternatives:**
- Validity windows (validFrom/validTo) — richer bi-temporal model but requires store API changes
- Soft-delete with archive domain — simple but grows storage
**Rationale:** Zero new store API needed. Supersession already provides invalidate-not-delete semantics. SupersessionStatus tracking gives audit trail. reinstate() gives rollback.
**Trade-offs:** No explicit time windows — supersession is binary (active/superseded), not temporal-range-based
**Sources:** CbrCaseMemoryStore.supersede()/reinstate()/getSupersessionStatus(); Research §2.8
**Exploration:** quick
**Status:** captured

## D13: Cross-linking — feature-based links

**Choice:** Store cross-links as feature values on CbrCase (e.g., "related_cases" → comma-separated caseIds). Read via retrieveSimilar with feature matching.
**Alternatives:**
- Dedicated MemoryLinkStore SPI — richer graph model but adds persistence contract
- Supersession chains — limited to parent-child, no peer links
**Rationale:** No store API change. Feature maps are extensible by design. Cross-links are metadata on the memory, not a separate graph. Consolidation step writes links as part of the merged case.
**Trade-offs:** No graph traversal — link queries are feature-match queries, not path queries
**Sources:** CbrCase.features(); FeatureValue; Research §2.8 (Zettelkasten/A-MEM)
**Exploration:** quick
**Status:** captured

## D14: Integrity checking — IntegrityChecker SPI with hybrid default

**Choice:** IntegrityChecker SPI. Default implementation does structural checks (orphaned supersessions, duplicate caseIds, missing features, stale undecayed memories), flags anomalies, escalates to optional SemanticIntegrityChecker (LLM-backed, @DefaultBean no-op). Consumers override semantic checker to activate.
**Alternatives:**
- Structural checks only — cheaper but misses contradictory memories
- Single strategy with escalation config — simpler but less flexible
- TieredIntegrityChecker — follows TieredContentSummariser pattern but integrity isn't batch-size-dependent
**Rationale:** Structural checks are always cheap and catch data consistency issues. Semantic escalation is opt-in — consumers who need contradiction detection activate it. SPI gives domain-specific flexibility.
**Trade-offs:** Default semantic checker is no-op — consumers must wire LLM-backed implementation to get semantic checks
**Sources:** Research §2.8 (memory integrity/safety); TieredContentSummariser pattern; IntakeClassifier SPI pattern
**Exploration:** quick
**Status:** captured

## D15: Package placement — agentic.memory

**Choice:** New sub-package io.casehub.blocks.agentic.memory
**Alternatives:**
- agentic.personality — keeps cognitive patterns together but conflates memory lifecycle with personality
- agentic.personality.memory — nested sub-package, awkward nesting
**Rationale:** Memory lifecycle is a distinct concern from personality evolution. Clean separation with own test directory. Consistent with how blocks organises sub-packages (belief, coalition, intention are peer packages under agentic).
**Trade-offs:** More packages under agentic — but the package tree is already wide
**Sources:** Existing package structure: agentic.belief, agentic.coalition, agentic.intention, agentic.personality
**Exploration:** quick
**Status:** captured
