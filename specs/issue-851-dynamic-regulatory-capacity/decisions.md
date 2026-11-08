## D1: Context anchor for proactive avoidance

**Choice:** Multi-dimensional context with query-scope modes
**Alternatives:**
- Observation text only — generalises beyond location but loses structured queryability
- Structured location only — simpler but limited to spatial avoidance
**Rationale:** Store all dimensions (location, interacting agents, topic) on every regulation node. Mode controls which dimensions the proactive participant queries. Start with location-only mode. Historical data retroactively gains value when new dimensions are promoted.
**Trade-offs:** Storage cost per regulation node increases (3 extra string properties). All properties are optional — null when unparseable.
**Sources:** EmotionRegulationMindMapAdapter.java:29-48 (existing property storage), issue #203 (spatial model), issue #805 (social regulation)
**Exploration:** quick
**Status:** captured

## D2: Location source — how the system knows where the agent is

**Choice:** Parse from observation text using a lightweight extractor
**Alternatives:**
- Add location to CognitionTickContext — clean API but requires changes to CognitionCore and all callers
- Query MindMap for current location — adds a MindMap read per tick
**Rationale:** The observation string describes what/where the agent is. A rule-based extractor parses location and interacting agents from it. No API changes needed. Falls back gracefully (null context) when unparseable.
**Trade-offs:** Extraction accuracy depends on observation format consistency. If observations change format, extractor needs updating.
**Sources:** CognitionTickContext.java:7 (observation field), EmotionRegulationTickParticipant.java:140 (existing observation access pattern)
**Exploration:** quick
**Status:** captured

## D3: Output mechanism — how avoidance signals reach the goal system

**Choice:** registerGoals() from FOUNDATION phase
**Alternatives:**
- Modify observation text — inject avoidance context before appraisal; organic but fragile text injection
- New ProactiveAvoidanceSignal mechanism — more explicit but requires new API surface and consumers
**Rationale:** Call GoalProposalOrchestrator.registerGoals() early in FOUNDATION. Proposals accumulate and are processed when TERMINAL fires. Simple, uses existing API. The timing gap (FOUNDATION→TERMINAL) is fine because proposals are additive.
**Trade-offs:** Avoidance goals compete with drive-based goals in the normal priority system. No special "avoidance" priority — just driveIntensity-based sorting.
**Sources:** EmotionRegulationGoalParticipant.java:37 (existing registerGoals pattern), GoalProposalOrchestrator.java:116 (registerGoals API)
**Exploration:** quick
**Depends on:** D1 (context dimensions determine what avoidance goals reference)
**Status:** captured

## D4: Personality modulation — avoidance threshold sensitivity

**Choice:** Derive avoidance sensitivity from existing DispositionAxes
**Alternatives:**
- Add proactiveAvoidanceSensitivity to RegulationDefaults — explicit but grows the record
- Reuse redirectTendency as proactive threshold — pragmatic but semantically loose
**Rationale:** Map existing axes to an avoidance-sensitivity score: conservative risk + avoidant conflict = high sensitivity (low threshold), bold risk + competitive conflict = low sensitivity (high threshold). Follows the CognitiveDerivationEngine derivation pattern. No new personality data needed.
**Trade-offs:** Sensitivity is derived, not independently tunable per character. But the axes already capture the relevant personality variation.
**Sources:** CognitiveDerivationEngine.java:502-526 (existing derivation pattern), RegulationStrategyPredictor.java (existing personality→regulation mapping), Gross 1998 (situation selection as antecedent-focused)
**Exploration:** quick
**Status:** captured
