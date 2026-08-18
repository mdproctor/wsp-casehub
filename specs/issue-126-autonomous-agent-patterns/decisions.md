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

**Choice:** Layered mapping (axis-level + vocabulary-level)
**Alternatives:**
- Vocabulary-aware mapping only — maps via VocabularyTerm.opposite() for function-level activation. Only meaningful for JungianFunctionTerm; other 12 vocabulary frameworks return Optional.empty(), excluding their agents from personality evolution entirely.
- Axis-level mapping only — maps to 5 disposition axes (socialOrient, ruleFollowing, etc.) rather than individual functions. Universal but loses Jungian vocabulary detail.
- Raw event passthrough — passes raw domain events as activation terms, bypasses eidos JPAF infrastructure entirely.
**Rationale:** Every agent with a disposition profile has 5 DispositionAxis values (SOCIAL_ORIENTATION, RULE_FOLLOWING, RISK_APPETITE, AUTONOMY, CONFLICT_MODE) regardless of vocabulary framework. Axis-level mapping provides universal personality evolution for all agents. Vocabulary-level mapping via VocabularyTerm.opposite() enriches agents whose vocabulary supports oppositional semantics (currently Jungian only) with shadow-function-aware activation. The SPI declares both layers; implementors provide at minimum the axis-level mapping.
**Trade-offs:** Two-layer mapping is more complex than either layer alone. Axis-level mapping is coarser than vocabulary-level — agents with vocabulary enrichment get richer evolution trajectories. Some domain events may not have a natural axis-level mapping (the translation SPI returns Optional to handle this).
**Sources:** DispositionSignalStore.java (eidos), VocabularyTerm.opposite() (eidos API), DispositionAxis.java (eidos API), GE-20260728-a53632 (vocabulary-generic structural navigation technique)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope)
**Status:** revised (R1-06, R1-07: layered mapping replaces vocabulary-only mapping to avoid excluding agents without Jungian vocabulary)

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
**Depends on:** D1 (signal-driven orchestrator scope), D2 (layered mapping)
**Status:** revised (R1-13: moved dampening from recording-time to probe-time to enable retroactive calibration)

## D5: Bounded displacement enforcement

**Choice:** L2 norm ceiling on effective weights
**Alternatives:**
- Per-function activation cap — caps max activation count per function. Doesn't account for different base weights.
- Weight-range clamping — clamps effective weights to JPAF differentiation ranges. Constrains output, not input.
- Weighted norm — assigns smaller penalty to dominant/auxiliary displacement (expected to be strong) and larger penalty to inferior displacement. More psychologically coherent but embeds Jungian theory into the safety rail.
**Rationale:** The L2 displacement metric is already exposed through the DispositionHealth SPI: DispositionStatus.Drifted(effectiveWeights, mostActivated, driftMagnitude) includes driftMagnitude which is the L2 distance. The orchestrator checks the ceiling via probe results — when probe returns Drifted with driftMagnitude >= L2Ceiling, a halt flag is set. When probe returns Aligned, the flag clears. Between ticks, recording is gated by the halt flag (AtomicBoolean). No direct access to the private computeL2 method is needed.
**Complementary relationship with overReinforcementThreshold:** eidos's overReinforcementThreshold (default 0.50) is a per-function ceiling on the dominant's effective weight — fires when ONE function is disproportionately strong. The L2 ceiling is a global displacement bound — fires when the OVERALL profile drifts too far, catching distributed drift across multiple dimensions that no single per-function ceiling would detect. These are distinct geometric properties (per-dimension max vs global euclidean distance). Both mechanisms fire independently and serve complementary safety roles.
**Trade-offs:** L2 norm treats all axes equally — a shift in dominant function contributes the same as a shift in undifferentiated. This is intentional: the L2 ceiling is a safety rail, not a personality model. The JPAF pipeline handles psychological correctness (evolution detection); the L2 ceiling handles the "displacement accumulated without triggering evolution" case, which is pathological by definition. Bounded overshoot between probe ticks is an accepted trade-off of D3's periodic model.
**Sources:** DispositionStatus.Drifted.driftMagnitude (eidos API), JPAF Equations 1-3 (arXiv:2601.10025), DispositionPreferenceKeys.OVER_REINFORCEMENT_THRESHOLD (eidos, default 0.50)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D3 (batched periodic timing)
**Status:** revised (R1-16, R1-17: clarified L2 check mechanism via Drifted.driftMagnitude rather than computeL2 reimplementation; documented complementary relationship with overReinforcementThreshold)

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
