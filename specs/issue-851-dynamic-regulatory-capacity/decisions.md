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
