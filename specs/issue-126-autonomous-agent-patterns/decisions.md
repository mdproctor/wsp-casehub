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

**Choice:** Vocabulary-aware mapping
**Alternatives:**
- Axis-level mapping — maps to 5 disposition axes (socialOrient, ruleFollowing, etc.) rather than individual functions. Simpler but loses Jungian vocabulary detail.
- Raw event passthrough — passes raw domain events as activation terms, bypasses eidos JPAF infrastructure entirely.
**Rationale:** DispositionSignalStore.recordActivation() takes a functionTerm (Jungian cognitive function). The SPI must translate domain events into function activations that are meaningful within the agent's disposition vocabulary. Vocabulary-generic navigation via VocabularyTerm.opposite() (already in eidos API) enables this without compile-time vocabulary coupling.
**Trade-offs:** Requires agents to have a registered disposition vocabulary for full function-level mapping. Agents without vocabulary fall back to simpler behavior (no activation recording).
**Sources:** DispositionSignalStore.java (eidos), VocabularyTerm.opposite() (eidos API), GE-20260728-a53632 (vocabulary-generic structural navigation technique)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope)
**Status:** captured

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

**Choice:** Configurable weight per signal valence
**Alternatives:**
- Sentiment-aware decay rate — negative activations decay faster. Requires tracking signal source alongside counts, adds state to DispositionSignalStore.
- Fixed ratio cap — caps negative-to-positive activation ratio. Less granular than per-signal weighting.
**Rationale:** LLMPTBench (NeurIPS 2025) shows agentic frameworks exhibit exaggerated shifts under negative events. Applying a dampening factor (default 0.5) to negative signals before recording is simple, transparent, and tunable per agent/tenant. No changes needed to DispositionSignalStore — dampening is applied at signal translation time, not storage time.
**Trade-offs:** Dampening is applied at recording time and becomes permanent — cannot retroactively adjust. Accepted because the alternative (tracking valence per activation) adds significant complexity to eidos's store.
**Sources:** LLMPTBench (NeurIPS 2025, OpenReview kVXePuKReA), BFI-Adapt (arXiv:2608.06485)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D2 (vocabulary-aware mapping)
**Status:** captured

## D5: Bounded displacement enforcement

**Choice:** L2 norm ceiling on effective weights
**Alternatives:**
- Per-function activation cap — caps max activation count per function. Doesn't account for different base weights.
- Weight-range clamping — clamps effective weights to JPAF differentiation ranges. Constrains output, not input.
**Rationale:** DefaultDispositionHealth already computes L2 distance between base and effective weights (computeL2 method). Using the same metric for bounding is coherent. When L2 displacement exceeds a configurable ceiling, the orchestrator stops recording activations until decay brings displacement back within range. This preserves the existing probe logic unchanged.
**Trade-offs:** L2 norm treats all axes equally — a shift in dominant function contributes the same as a shift in undifferentiated. Accepted because per-function bounding adds complexity without clear benefit at this stage.
**Sources:** DefaultDispositionHealth.computeL2() (eidos), JPAF Equations 1-3 (arXiv:2601.10025)
**Exploration:** quick
**Depends on:** D1 (signal-driven orchestrator scope), D3 (batched periodic timing)
**Status:** captured
