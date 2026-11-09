## D1: Context anchor for proactive avoidance

**Choice:** Multi-dimensional context with query-scope modes
**Alternatives:**
- Observation text only — generalises beyond location but loses structured queryability
- Structured location only — simpler but limited to spatial avoidance
**Rationale:** Store all dimensions (location, interacting agents, topic) on every regulation node. Mode controls which dimensions the proactive participant queries. Start with location-only mode. Historical data retroactively gains value when new dimensions are promoted.
**Trade-offs:** Storage cost per regulation node increases (3 extra string properties). All properties are optional — null when unparseable. CBR subsystem (R1-09) is a better long-term retrieval mechanism — MindMap in-memory filtering scales to ~200 nodes; CBR provides indexed vector search. Deferred to Phase 2 when scale demands it.
**Sources:** EmotionRegulationMindMapAdapter.java:29-48 (existing property storage), issue #203 (spatial model), issue #805 (social regulation)
**Exploration:** quick
**Status:** captured

## D2: Location source — how the system knows where the agent is

**Choice:** Add optional RegulationContext to CognitionTickContext
**Alternatives:**
- Parse from observation text — no API changes but fragile extraction, silent degradation when format changes (R1-07)
- Query MindMap for current location — adds MindMap read per tick
**Rationale:** CognitionTickContext already uses telescoping constructors (4→5→6→7 args). Adding a @Nullable RegulationContext adds one more constructor — zero existing callers break (R1-06). Explicit API makes context provision visible at compile time. Callers that can provide context do; callers that can't pass null. RegulationContext is a simple record: location, interactingAgents, topic.
**Trade-offs:** Adds a new record type (RegulationContext) and one more constructor to CognitionTickContext. Minimal — the telescoping pattern already absorbs this.
**Sources:** CognitionTickContext.java:7 (telescoping constructors), R1-06/R1-07 (decision review findings)
**Exploration:** quick → revised after decision review
**Status:** revised

## D3: Output mechanism — how avoidance signals reach the goal system

**Choice:** appendGoals() from FOUNDATION phase (new method)
**Alternatives:**
- registerGoals() — has replace semantics (R1-02), would clobber proactive goals when reactive regulation calls it from TERMINAL
- Modify observation text — fragile text injection
- New ProactiveAvoidanceSignal mechanism — unnecessary new API surface
**Rationale:** Add `appendGoals()` method to GoalProposalOrchestrator with additive semantics. Proactive avoidance calls appendGoals() from FOUNDATION. Reactive regulation (EmotionRegulationGoalParticipant) keeps calling registerGoals() from TERMINAL. No clobbering. Registered goals bypass the priority system entirely (R1-03) — they accumulate without capacity limiting. This is acceptable for avoidance goals: they should always render regardless of drive priority.
**Trade-offs:** Registered goals (both proactive and reactive) bypass capacity limiting. Unbounded growth possible in theory — mitigated by temporal decay (D5) which naturally limits the active set.
**Sources:** GoalProposalOrchestrator.java:116 (registerGoals replace semantics), R1-02/R1-03 (decision review findings)
**Exploration:** quick → revised after decision review
**Depends on:** D1 (context dimensions determine what avoidance goals reference)
**Status:** revised

## D4: Personality modulation — avoidance threshold sensitivity

**Choice:** Use existing redirectTendency from RegulationDefaults
**Alternatives:**
- Derive new avoidance-sensitivity from DispositionAxes subset — duplicates redirectTendency derivation (R1-12)
- Add proactiveAvoidanceSensitivity to RegulationDefaults — explicit but unnecessary growth
**Rationale:** redirectTendency (CognitiveDerivationEngine line ~520) is already derived from the same axes: avoidant(+0.25), conservative(+0.20), low autonomy(+0.10). Situation selection IS redirection — they capture the same personality dimension. Agents with high redirectTendency proactively avoid at lower memory intensity thresholds.
**Trade-offs:** No independent tuning knob for proactive vs reactive redirect. Acceptable because the personality variation is the same in both cases.
**Sources:** CognitiveDerivationEngine.java:516-521 (redirectTendency derivation), R1-12 (decision review finding), 2026-11-07-regulation-continuous-weights-literature-review.md
**Exploration:** quick → revised after decision review
**Status:** revised

## D5: Temporal decay — learning path to safety

**Choice:** Time-based decay of avoidance weight
**Alternatives:**
- No decay (permanent avoidance) — creates one-way ratchet, violates regulatory flexibility (Bonanno & Burton 2013)
- Probabilistic avoidance — avoid with probability proportional to severity, allowing some exposure
- Explicit "approach despite avoidance" via personality override — personality already modulates threshold (D4)
**Rationale:** Avoidance weight for a past regulation episode decays linearly with ticks since the episode. After N ticks (configurable, default ~50) without re-exposure, the avoidance lapses. If the agent encounters the situation again and regulates successfully, a positive outcome resets the avoidance (or reduces it). This prevents permanent avoidance bias (R1-15) while still providing short-term protection.
**Trade-offs:** Decay rate is a tuning parameter. Too fast = no protection, too slow = permanent avoidance. Start with 50 ticks (empirical tuning expected).
**Sources:** R1-15 (decision review finding), Bonanno & Burton 2013 (regulatory flexibility), Gross 2015 (extended process model with identification→selection→implement→monitor)
**Exploration:** quick (surfaced by decision review)
**Status:** captured

## D6: Wire up RegulationStrategyPredictor in eval tests

**Choice:** Provide CognitiveDefaultsRegistry so the predictor pins strategies deterministically
**Alternatives:**
- LLM-only (no predictor) — all 5 strategies depend on LLM judgment; more naturalistic but less deterministic for all strategies
**Rationale:** The predictor is signal-based (numerical capacities, categorical disposition axes, emotion types) — not keyword-based. It applies Petter 2025 personality × emotion-type correlations to structured LLM appraisal output. Pins 4/5 strategies deterministically. SITUATION_MODIFICATION falls through to LLM by design (predictor can't pin it). Tests the production prediction pipeline.
**Trade-offs:** Tests the predictor-pinned path, not the LLM's free-choice path. Acceptable — the predictor IS the production strategy selection mechanism.
**Sources:** RegulationStrategyPredictor.java, EmotionRegulationTickParticipant.java:123 (predictor null-check), CognitionCore.java:755 (defaultsRegistry wiring)
**Exploration:** quick
**Status:** captured

## D7: Separate test class for per-strategy tests

**Choice:** New `PerStrategyRegulationEvalTest` class
**Alternatives:**
- Extend existing `RegulationFullPipelineEvalTest` — mixes two different CognitionCore configurations (with/without defaults registry), complicates an already-passing test
**Rationale:** The existing test uses `CognitionCoreTestBuilder` without a `CognitiveDefaultsRegistry`. Per-strategy tests need a different builder setup with the registry wired. Separate class keeps both tests independently runnable and avoids regression risk.
**Trade-offs:** Some duplication of test infrastructure (scenario runner method). Acceptable — the setup difference is fundamental, not cosmetic.
**Sources:** RegulationFullPipelineEvalTest.java, CognitionCoreTestBuilder.java
**Exploration:** quick
**Status:** captured

## D8: Strategy-specific assertions

**Choice:** Assert specific strategy per scenario, with environmental field checks
**Alternatives:**
- Assert only "regulation activated" (any strategy) — doesn't verify per-strategy coverage, which is the whole point of #822
**Rationale:** Each scenario asserts the expected strategy. For SITUATION_MODIFICATION (LLM free choice), accept SITUATION_MODIFICATION or SITUATION_SELECTION (both environmental). For environmental strategies (1-2), also assert regulationAxis != null and environmentalDirective != null — required for DriveGoalProposal production. Keep N=3 with ≥2/3 threshold from existing test pattern.
**Trade-offs:** SITUATION_MODIFICATION assertion is looser (accepts either environmental strategy). This reflects genuine predictor design — it can't pin modification, so we verify environmental strategy selection, not the specific sub-type.
**Sources:** EmotionRegulationResult.java (isEnvironmentalStrategy), EmotionRegulationGoalParticipant.java (null checks for axis/directive), issue #822 acceptance criteria
**Exploration:** quick
**Depends on:** D6 (predictor wiring determines which strategies can be asserted precisely)
**Status:** captured

## D9: Separate test class for per-modality eval tests

**Choice:** New `PerModalityRegulationEvalTest` class in caps-llm-classifier eval package
**Alternatives:**
- Extend `PerStrategyRegulationEvalTest` — different test structure (multi-tick vs 2-tick), different assertions (behavioral change over time vs single-tick strategy), mixes concerns
- Add to `RegulationFullPipelineEvalTest` — different CognitionCore lifecycle (persistent vs fresh per run)
**Rationale:** Per-modality tests require keeping a single CognitionCore across multiple ticks (fatigue, monitor, failure signals persist within the instance). Per-strategy tests create a fresh core per run. The scenario runner captures per-tick results rather than a single final result. Fundamentally different test structure requires separate class.
**Trade-offs:** Three eval test classes in the same package. Acceptable — each tests a distinct dimension of the regulation pipeline.
**Sources:** PerStrategyRegulationEvalTest.java (2-tick pattern), EmotionRegulationTickParticipant.java:80-81 (fatigueTracker/regulationMonitor persist in instance), CognitionCoreTestBuilder.java:100-108 (4-arg builder)
**Exploration:** quick
**Depends on:** D7 (established separate-class pattern for different CognitionCore configurations)
**Status:** captured

## D10: CognitionCore.tick() overload for RegulationContext

**Choice:** Add 7-arg tick() overload accepting RegulationContext, flowing it to the 8-arg CognitionTickContext constructor
**Alternatives:**
- Test ProactiveRegulationParticipant directly with mocks (unit-level, not eval-level) — loses pipeline integration, doesn't prove the write path persists location to MindMap
- Parse location from observation text in ProactiveRegulationParticipant — fragile, already rejected in D2
**Rationale:** Two-directional production gap, not just a test gap. (1) EmotionRegulationTickParticipant.write() at line 182 passes context.regulationContext() to MindMapAdapter — always null, so location is never persisted. (2) ProactiveRegulationParticipant.tick() at line 46-47 short-circuits on null regulationContext — proactive avoidance never fires. CognitionTickContext already has the 8-arg canonical constructor with RegulationContext (line 8-17 of CognitionTickContext.java). CognitionCore.tick() at line 253 uses the 7-arg constructor that passes null. Fix: one new overload, one changed constructor call.
**Trade-offs:** Adds one more tick() overload to CognitionCore (already has 3). Callers without RegulationContext continue using existing overloads unchanged.
**Sources:** CognitionCore.java:253 (7-arg constructor call), CognitionTickContext.java:39-46 (7-arg → 8-arg with null), ProactiveRegulationParticipant.java:46-47 (null guard), EmotionRegulationTickParticipant.java:182-188 (write with null context)
**Exploration:** quick
**Depends on:** D2 (RegulationContext record already exists in CognitionTickContext)
**Status:** captured

## D11: Fatigue test — ~20 ticks for deterministic predictor override

**Choice:** Run ~20 ticks to accumulate fatigue > 0.7, verify predictor forces ATTENTIONAL_DEPLOYMENT
**Alternatives:**
- 5-tick test per issue spec — fatigue ≈ 0.30 after 5 ticks (convergent series: effort = alpha × discrepancy / D_MAX where alpha = 0.5 × (1 - fatigue)), nowhere near any threshold
- Combined fatigue + monitor test (5-7 ticks) — shorter but depends entirely on LLM generating declining effectiveness, not deterministic
**Rationale:** At fatigue > 0.7, RegulationStrategyPredictor.predict() line 36-39 unconditionally returns ATTENTIONAL_DEPLOYMENT with confidence 0.85. This is deterministic — no LLM dependency for the strategy override. Set up a character with high reappraisalCapacity that normally selects COGNITIVE_CHANGE. After ~20 ticks of sustained regulation, verify strategy switches to ATTENTIONAL_DEPLOYMENT. The LLM still generates narrative/effectiveness but the strategy is predictor-pinned.
**Trade-offs:** 20 ticks × N=3 = 60 LLM calls per test. Comparable to existing eval suite total (~30 calls across 5 tests). Test runtime ~10-15 min. Acceptable for llm-eval profile.
**Sources:** RegulationFatigueTracker.java:8-9 (FATIGUE_RATE=0.15, formula), EmotionRegulationTickParticipant.java:100-104 (effort = alpha × discrepancy / D_MAX, negative feedback loop), RegulationStrategyPredictor.java:36-39 (fatigue > 0.7 override)
**Exploration:** deep-analysis (first-principles arithmetic verification of convergent series)
**Status:** captured

## D12: Automatic regulation — strategy concordance, no timing

**Choice:** Test strategy concordance between predictor and LLM free choice; drop timing assertion
**Alternatives:**
- Include timing assertion — LLM is called in both paths (generationPhase.generate() at EmotionRegulationGenerationPhase.java:148 invokes LLM for narrative/effectiveness regardless of pinnedStrategy), no meaningful latency difference
- Test only predictor path — already covered by #822 (PerStrategyRegulationEvalTest), adds nothing new
**Rationale:** The predictor's value is selecting the SAME strategy the LLM would choose, deterministically and without the strategy-selection component of the prompt. Run same scenario twice: (1) with defaultsRegistry → predictor pins strategy, (2) without defaultsRegistry → LLM freely chooses. Assert strategies match ≥2/3 runs. This validates the predictor's personality → strategy mapping against the LLM's independent judgment.
**Trade-offs:** Concordance is probabilistic (LLM may legitimately choose differently). ≥2/3 threshold is lenient. If concordance is consistently low, the finding is that the predictor's mapping diverges from LLM judgment — a real signal, not a test failure.
**Sources:** EmotionRegulationGenerationPhase.java:148-149 (LLM called in both paths), EmotionRegulationGenerationPhase.java:357-359 (pinnedStrategy prompt), EmotionRegulationTickParticipant.java:121-138 (predictor flow)
**Exploration:** deep-analysis (first-principles verification of LLM invocation path)
**Status:** captured

## D13: Monitoring — 5-tick signal consumption cycle

**Choice:** 5 ticks with pinned strategy, assert strategy change on tick 5
**Alternatives:**
- 4 ticks — signal fires on tick 4 but is consumed on tick 5 (failureSignals stored on tick N, consumed on tick N+1 via failureSignals.remove() at line 112)
- Direct monitor unit test — already exists (RegulationMonitorTest), doesn't prove pipeline signal flow
**Rationale:** Pin RESPONSE_MODULATION via predictor (high suppressionCapacity). Use escalating emotional observations across ticks to encourage declining effectiveness from the LLM. The LLM also receives increasing fatigue (regulationFatigue > 0.01 → prompt includes "Regulation fatigue: X%"). Monitor's consecutiveDeclines counter requires same strategy + declining effectiveness for 3 consecutive ticks (RegulationMonitor.java:31-33). Signal generated on tick 4, consumed on tick 5 — at which point pendingSignal != null bypasses the predictor (line 123) and the LLM receives "⚠ STRATEGY FAILING" prompt (line 389). Assert: strategy on tick 5 differs from ticks 1-4.
**Trade-offs:** Effectiveness decline is LLM-generated and non-deterministic. Escalating observations and fatigue prompt provide pressure but no guarantee. N=3, ≥1/3 threshold acknowledges this. If the test consistently fails, the observation escalation needs redesign, not the pipeline.
**Sources:** EmotionRegulationTickParticipant.java:112 (signal consumption), EmotionRegulationTickParticipant.java:178-180 (signal generation), RegulationMonitor.java:26-48 (decline tracking), EmotionRegulationGenerationPhase.java:382-396 (failure signal prompt)
**Exploration:** deep-analysis (first-principles trace of signal flow through tick lifecycle)
**Depends on:** D6 (predictor wiring for strategy pinning across ticks)
**Status:** captured

## D14: Proactive — 3-tick eval with RegulationContext and seeded memory

**Choice:** 3-tick full-pipeline test with RegulationContext flowing through CognitionCore
**Alternatives:**
- Direct ProactiveRegulationParticipant test with mocked MindMap — already exists (ProactiveRegulationParticipantTest.java), doesn't prove pipeline integration (write path → MindMap → read path)
- Seed MindMap directly without running tick 1 — bypasses the write path, doesn't prove EmotionRegulationMindMapAdapter persists location data
**Rationale:** Requires D10 (tick overload). Tick 1: high-shame observation at RegulationContext(location="library") → regulation fires → EmotionRegulationMindMapAdapter.write() persists result with regulationLocation="library". Tick 2: different observation (recovery tick — emotion below baseline). Tick 3: agent re-enters "library" (same RegulationContext) → ProactiveRegulationParticipant reads MindMap in FOUNDATION phase, finds matching episode, produces DriveGoalProposal for avoidance. Character has high redirectTendency (0.8) so threshold = 1.0 - 0.8 = 0.2 (easy to exceed with low-effectiveness regulation in tick 1). Assert: GoalProposalOrchestrator receives avoidance goal with goalName containing "proactive-avoidance" and "library".
**Trade-offs:** Depends on tick 1 producing a regulation result with effectiveness < 0.8 (so avoidanceScore > 0.2 × decayWeight). The low threshold (0.2) makes this robust — only very effective regulation in tick 1 would block the avoidance signal. Temporal decay: ticks 1-3 happen within seconds, well within the 50-tick decay horizon.
**Sources:** ProactiveRegulationParticipant.java:55-89 (avoidance score calculation), EmotionRegulationMindMapAdapter.java:32-73 (write with regulationContext), ProactiveRegulationParticipantTest.java:58-78 (existing component test pattern), GoalProposalOrchestrator.appendGoals() (D3)
**Exploration:** deep-analysis (first-principles trace of write path → MindMap → read path)
**Depends on:** D10 (CognitionCore.tick() overload for RegulationContext), D4 (redirectTendency as threshold)
**Status:** captured

## D15: State model — extend CognitionSnapshot, not new record

**Choice:** Extend the existing CognitionSnapshot record with missing pipeline fields (appraisal, CAPS settling, regulation, concern salience, prompt sections)
**Alternatives:**
- New CognitiveStateDump record — purpose-built trace chain structure but duplicates fields already in CognitionSnapshot
- Layered snapshot + trace overlay — compose CognitionSnapshot with a lightweight CognitiveTrace record, adds composition complexity without clear benefit
**Rationale:** CognitionSnapshot already captures ~5 of the 9 trace chain stages (mood, drives, mental models, strategy, narrative, goals, attention). CognitionMetrics wraps it and adds prompt section content. Extending the canonical record keeps one representation, avoids duplication, and lets CognitionDelta.diffFrom() automatically cover new fields.
**Trade-offs:** Larger record size. All callers of CognitionSnapshot get the new fields whether they need them or not. Acceptable — the record is already a point-in-time aggregate.
**Sources:** CognitionSnapshot.java (existing record), CognitionMetrics.java (wraps snapshot + prompt sections), CognitionDelta.java (diffFrom)
**Exploration:** quick
**Status:** captured

## D16: Access model — inject CognitionCore into CognitionService

**Choice:** Add CognitionCore as a dependency to CognitionService, add new @PlatformQuery method to CognitionApi
**Alternatives:**
- New CognitivePipelineApi @McpDomain — separate API surface for pipeline tracing, splits diagnostic surface across two domains
- CognitionCore fires CDI event with snapshot after each tick — decoupled but adds event infrastructure for simple query
**Rationale:** CognitionService already aggregates multiple dependencies (MindMapStore, SnapshotStore, CognitiveProfile, CaseMemoryStore, etc.). Adding CognitionCore follows the same pattern. Keeps all cognitive observability under one @McpDomain("neocortex/cognition").
**Trade-offs:** Creates a dependency from the observability module to the cognition module. This is a new dependency direction — currently observability is downstream (receives events) not upstream (queries state). Acceptable because the query is read-only and CognitionCore already exposes public current*() methods.
**Sources:** CognitionApi.java (@McpDomain interface), CognitionService.java (implementation with multiple dependencies), CognitionBeans.java (CDI wiring)
**Exploration:** quick
**Status:** captured

## D17: Scope — live inspection + per-tick snapshot storage

**Choice:** Both live inspection of current tick state AND per-tick snapshot persistence for historical replay and before/after comparison
**Alternatives:**
- Live inspection only — query current state via current*() methods, no storage, historical replay as follow-up
- Live + on-demand capture — live by default, manual capture mutation for point-in-time snapshots, lower storage cost
**Rationale:** The issue explicitly mentions both live inspection and historical replay. Per-tick storage enables eval postmortem analysis (what happened at each pipeline stage for each agent across a 300-tick scenario run). The eval framework already captures transcripts — pipeline snapshots add the "why" to the "what."
**Trade-offs:** Storage volume: 17 agents × 300 ticks = 5100 snapshots per scenario run. Each snapshot is ~2-5 KB serialized. ~10-25 MB per run. Acceptable for SQLite. Retention policy needed for production use.
**Sources:** Issue #520 ("live inspection and historical replay"), CognitionSnapshot.java (partial capture already exists), SnapshotStore.java (existing keyframe/mutation pattern)
**Exploration:** quick
**Status:** captured

## D18: Storage — new PipelineSnapshotStore with SQLite impl

**Choice:** New PipelineSnapshotStore interface in cognitive-observability-core with SqlitePipelineSnapshotStore implementation
**Alternatives:**
- Extend SnapshotStore with pipeline methods — mixes graph topology snapshots (nodes/edges) with pipeline state (mood values, CAPS activations), confuses the data model
- JSON append log — simplest but no query capability, not suitable for time-range queries or agent-scoped retrieval
**Rationale:** SnapshotStore is designed for graph-level data (GraphSnapshot, NodeSnapshot, EdgeSnapshot, GraphMutation). Pipeline snapshots are fundamentally different — transient computational state (scalar values, enum results, maps) not graph topology. Separate interface follows the same module pattern: interface in -core, SQLite impl in -sqlite, in-memory impl in -testing.
**Trade-offs:** New interface + two implementations. Schema optimized for time-series query (agent, tenant, tick range). More code but clean separation of concerns.
**Sources:** SnapshotStore.java (existing interface pattern), SqliteSnapshotStore.java (existing SQLite impl pattern), InMemorySnapshotStore.java (existing test impl pattern)
**Exploration:** quick
**Status:** captured

## D19: CAPS capture — CapsEngine stores last settling result

**Choice:** Add lastResult(agentId, tenantId) to CapsEngine, storing per-agent SettlingResult + input activations in a ConcurrentHashMap
**Alternatives:**
- Caller stores and exposes — keeps CapsEngine stateless but couples capture to call site, harder to find
- CDI event on settle — decoupled but adds event infrastructure for a simple stateful query
**Rationale:** Follows the same pattern used throughout the cognitive pipeline: DriveOrchestrator.currentDrives(), AppraisalTickParticipant.currentResult(), MoodOrchestrator.currentMood(). Per-agent state in ConcurrentHashMap, cleared on agent removal. CognitionCore already knows the agent/tenant context when querying. Input activations stored alongside result for complete trace chain coverage (stages 3-4).
**Trade-offs:** CapsEngine becomes stateful (previously pure function). The state is per-agent and read-only from the observability perspective. Acceptable — all other pipeline components are already stateful per-agent.
**Sources:** CapsEngine.java (settle() method), CapsSettler.java (implementation), DriveOrchestrator.java (currentDrives pattern), AppraisalTickParticipant.java (currentResult pattern)
**Exploration:** quick
**Status:** captured
