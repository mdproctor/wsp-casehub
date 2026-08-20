# MentalModel Design — Theory of Mind with BDI Tracking

**Issue:** casehubio/blocks#123
**Date:** 2026-08-20
**Decisions:** D23-D31

## Overview

MentalModel maintains a BDI (Beliefs, Desires, Intentions) model per actor the agent interacts with. It tracks what others know, want, and plan to do — enabling the agent to reason about other people's cognitive states, not just its own goals.

**Distinction from UserModel:** UserModel tracks stable behavioral traits (communication style, preferences, topics of interest) — identity-level, slow-changing. MentalModel tracks dynamic cognitive state (current beliefs, desires, intentions) — situation-level, fast-changing. "User prefers concise communication" is UserModel. "User currently believes the deployment is risky" is MentalModel. Both are keyed by (agentId, subjectId, tenantId) but serve different consumer needs (D30).

## Architecture

### Orchestrator: `MentalModelOrchestrator`

`@ApplicationScoped` CDI bean following the established social cognition pattern. Per-subject BDI state in `ConcurrentHashMap<String, SubjectMentalState>`. Per-subject `ReentrantLock` for tick concurrency. Three public methods:

- **`record(MentalStateSignal, agentId, subjectId, tenantId)`** — accumulates signals. Heuristic extraction of explicit cues happens immediately (O(1), no LLM, no store write). Appends signal text to a buffer for later LLM inference.

- **`tick(agentId, subjectId, tenantId) → MentalModelTick`** — re-evaluates all three BDI dimensions. Applies confidence decay. Optionally invokes LLM when enough new signal has accumulated (gated by `minSignalsForInference` and `inferenceCooldown`). Persists via `MentalModelStore`. Returns sealed outcome: `Unchanged`, `Updated`, `Inferred` (LLM ran).

- **`project(agentId, subjectId, tenantId) → List<MentalProjection>`** — projects current BDI state into a list of condition/confidence pairs. Each `MentalProjection` carries: `conditionKey` (String), `value` (boolean), `confidence` (double [0,1]), `dimension` (BDI enum). Consumers filter by confidence threshold and merge into `GoapWorldState` via `new GoapWorldState(conditions)` or `.with(key, value)`.

### BDI State Container: `SubjectMentalState`

Package-private mutable state per subject:

```
SubjectMentalState:
  agentId, subjectId, tenantId     — identity triple
  beliefs: BeliefSet<String>       — AGM-style, key=topic, value=attributed belief text
  desires: List<AttributedDesire>  — what the subject wants
  intentions: List<AttributedIntention> — what the subject plans to do
  pendingSignals: int              — count since last inference
  textBuffer: StringBuilder        — raw signal text for LLM
  lastSignalTimestamp: Instant     — for confidence decay
  lastInferenceTimestamp: Instant  — for cooldown gating
  lastActivityTimestamp: Instant   — for eviction
  currentSnapshot: MentalModelSnapshot — latest persisted state
```

### BDI Dimension Types

**Beliefs** — use existing `BeliefSet<String>` from `blocks.agentic.belief`:
- Key = topic (e.g., "deployment_risk", "team_capacity")
- Value = attributed belief text (e.g., "subject thinks deployments are risky")
- Entrenchment = reinforcement count (increases on each confirming signal)
- Confidence = temporal freshness [0,1], decays with time
- AGM revision via `BeliefSet.revise()` when contradictory evidence arrives

**Desires** — new `AttributedDesire` record:
```java
record AttributedDesire(
    String key,           // e.g., "quick_resolution"
    String description,   // "subject wants a quick resolution"
    double confidence,    // [0,1], decays faster than beliefs
    Instant lastReinforced
) {}
```

**Intentions** — new `AttributedIntention` record:
```java
record AttributedIntention(
    String key,           // e.g., "escalate_to_manager"
    String description,   // "subject plans to escalate to their manager"
    double confidence,    // [0,1], decays fastest
    Instant lastReinforced
) {}
```

### Confidence Decay

Each BDI dimension decays at a configurable rate (D27):
- **Beliefs** decay slowly (default halfLife: 7 days). Well-established beliefs persist.
- **Desires** decay moderately (default halfLife: 1 day). Wants shift with context.
- **Intentions** decay quickly (default halfLife: 4 hours). Plans change frequently.

Decay formula: `confidence × Math.pow(0.5, elapsed / halfLife)`

Below a configurable floor (default 0.1), entries are evicted regardless of entrenchment. Entrenchment only protects beliefs during AGM revision — it determines which beliefs survive when contradictory evidence forces inconsistency resolution.

### GOAP Projection

`MentalProjection` record:
```java
record MentalProjection(
    String conditionKey,      // e.g., "subject_stressed"
    boolean value,            // projected boolean
    double confidence,        // from BDI entry
    BdiDimension dimension    // BELIEF, DESIRE, INTENTION
) {}
```

Consumers decide their own threshold:
```java
var projections = mentalModel.project(agentId, subjectId, tenantId);
var conditions = new HashMap<String, Boolean>();
for (var p : projections) {
    if (p.confidence() >= 0.5) {
        conditions.put(p.conditionKey(), p.value());
    }
}
var worldState = new GoapWorldState(conditions);
```

The projection method is a pure function of current BDI state — no side effects, no engine dependency.

## Signal Model

### `MentalStateSignal` — sealed interface (D25, D31)

Per-orchestrator signal type, consistent with the social cognition pattern. Each variant carries cognitive cues the orchestrator can extract:

```java
sealed interface MentalStateSignal {
    String content();  // raw text for LLM inference

    record VerbalCue(String content, CueType type) implements MentalStateSignal {}
    // Explicit verbal statements: "I think X", "I want Y", "I plan to Z"
    // CueType: BELIEF_STATEMENT, DESIRE_EXPRESSION, INTENTION_DECLARATION

    record BehavioralCue(String content, String actionType) implements MentalStateSignal {}
    // Observable actions that imply mental state
    // e.g., repeatedly checking a dashboard → desire for status visibility

    record ContextualCue(String content, Map<String, String> metadata) implements MentalStateSignal {}
    // Contextual information: deadlines, stress indicators, environmental factors
}
```

### Heuristic Extraction (Tier 1)

On `record()`, extract explicit cues from `VerbalCue` signals:
- "I think/believe/know X" → expand belief (key derived from X, high confidence)
- "I want/need/wish X" → add desire (high confidence)
- "I plan to/will/am going to X" → add intention (high confidence)

Heuristic extraction is keyword-based and runs at O(1). False positives from figurative language are tolerated — LLM inference on the next tick can correct them.

### LLM Inference (Tier 2)

On `tick()`, when `pendingSignals >= minSignalsForInference` and `inferenceCooldown` has elapsed:

1. Assemble prompt with current BDI state + accumulated signal text
2. Ask LLM to infer: what does this person currently believe, desire, and intend?
3. Parse structured JSON response into BDI updates
4. Apply updates via `BeliefSet.revise()` for beliefs, replace for desires/intentions

System prompt template:
```
You are analysing conversation signals to infer another person's mental state.
Given the current attributed beliefs, desires, and intentions, plus recent signals,
produce an updated BDI assessment.

Respond with JSON only:
{"beliefs": [{"key": "...", "text": "...", "confidence": 0.8}],
 "desires": [{"key": "...", "text": "...", "confidence": 0.7}],
 "intentions": [{"key": "...", "text": "...", "confidence": 0.5}]}

Only include entries with non-trivial confidence (>= 0.3).
For unchanged entries, omit them — previous state is preserved.
```

## Persistence

### `MentalModelStore` SPI (D28)

```java
public interface MentalModelStore {
    void store(MentalModelSnapshot snapshot);
    Optional<MentalModelSnapshot> lookup(String agentId, String subjectId, String tenantId);
    List<MentalModelSnapshot> findByAgent(String agentId, String tenantId);
    void eraseSubject(String subjectId, String tenantId);
}
```

### `MentalModelSnapshot` — persisted state

```java
record MentalModelSnapshot(
    String agentId,
    String subjectId,
    String tenantId,
    List<SnapshotBelief> beliefs,
    List<AttributedDesire> desires,
    List<AttributedIntention> intentions,
    Instant lastSignal,
    Instant lastInference,
    Instant snapshotCreated
) {}

record SnapshotBelief(
    String key,
    String value,
    int entrenchment,
    double confidence,
    Instant lastReinforced
) {}
```

### `CbrMentalModelStore` — `@DefaultBean` adapter

Backs onto `CbrCaseMemoryStore`. Mental model snapshot stored as a CbrCase:
- `problem` = serialized BDI summary text
- `caseType` = "mental-model"
- Features: agentId, subjectId, tenantId (for lookup), beliefs/desires/intentions as JSON StringVal
- Supersession for versioning (each tick that persists supersedes the previous snapshot)

Same adapter pattern as `CbrUserProfileStore` (D20). The CbrCase impedance mismatch is hidden behind the typed SPI.

## Configuration

### `MentalModelConfig`

```java
record MentalModelConfig(
    Duration beliefHalfLife,           // default: 7 days
    Duration desireHalfLife,           // default: 1 day
    Duration intentionHalfLife,        // default: 4 hours
    double confidenceFloor,            // default: 0.1
    int minSignalsForInference,        // default: 3
    Duration inferenceCooldown,        // default: 5 minutes
    int maxSignalsInPrompt,            // default: 20
    Duration evictionTimeout,          // default: 24 hours (for in-memory state)
    Duration expectedTickInterval      // default: 1 minute
) {}
```

## Tick Outcome

### `MentalModelTick` — sealed interface

```java
sealed interface MentalModelTick {
    record Unchanged(String reason) implements MentalModelTick {}
    record Updated(MentalModelSnapshot snapshot) implements MentalModelTick {}
    record Inferred(MentalModelSnapshot snapshot,
                    MentalModelSnapshot previous) implements MentalModelTick {}
}
```

## Package Placement

`io.casehub.blocks.agentic.social` — alongside PersonalityEvolutionOrchestrator, InnerLifeOrchestrator, and UserModelOrchestrator. MentalModel is the fourth member of the social cognition family: how the agent models others' cognitive states (D22 extended).

## Type Inventory

| Type | Kind | What it does |
|------|------|-------------|
| `MentalModelOrchestrator` | `@ApplicationScoped` CDI bean | Composition root: record()+tick()+project() |
| `MentalStateSignal` | Sealed interface | Input signal: VerbalCue, BehavioralCue, ContextualCue |
| `CueType` | Enum | BELIEF_STATEMENT, DESIRE_EXPRESSION, INTENTION_DECLARATION |
| `AttributedDesire` | Record | Inferred desire with confidence and reinforcement timestamp |
| `AttributedIntention` | Record | Inferred intention with confidence and reinforcement timestamp |
| `MentalProjection` | Record | GOAP projection: conditionKey + value + confidence + dimension |
| `BdiDimension` | Enum | BELIEF, DESIRE, INTENTION |
| `MentalModelTick` | Sealed interface | Tick outcome: Unchanged, Updated, Inferred |
| `MentalModelSnapshot` | Record | Persisted BDI state |
| `SnapshotBelief` | Record | Persisted belief with entrenchment + confidence |
| `MentalModelStore` | Interface (SPI) | Persistence: store, lookup, findByAgent, eraseSubject |
| `CbrMentalModelStore` | `@DefaultBean @ApplicationScoped` | CbrCaseMemoryStore adapter |
| `MentalModelConfig` | Record | Configuration: decay rates, thresholds, cooldowns |

13 types total (8 new records/enums, 2 sealed interfaces, 1 CDI bean, 1 SPI interface, 1 default bean).

## Dependencies

**Compile (existing):** `casehub-neocortex-memory-api` (CbrCaseMemoryStore, RelationshipEvent, QualitySignal)
**Provided (existing):** `casehub-platform-agent-api` (AgentProvider for LLM inference)
**Internal (existing):** `blocks.agentic.belief` (BeliefSet, Belief, ConsistencyChecker)

No new external dependencies.

## References

- Research §2.5 — Mental Model (Theory of Mind) pattern definition
- ToMA (Findings of ACL 2026) — Infusing Theory of Mind into LLM agents
- Beyond Words (Findings of ACL 2025) — ToM-informed alignment
- TimeToM (Findings of ACL 2024) — Temporal reasoning in Theory of Mind
- DPT-Agent (2025) — Dual Process Theory with Theory of Mind
- BeliefSet.java (blocks/agentic/belief) — AGM belief revision infrastructure
- EpistemicRule.java (blocks/conversation) — epistemic status classification
- GoapPlanner.java (engine-api) — A* planner API
- GoapWorldState.java (engine-api) — world state record with .with() builder
- UserModelOrchestrator.java (blocks/agentic/social) — precedent orchestrator
- InteractionSignal.java (blocks/agentic/social) — precedent signal model
- D23-D31 decisions — validated design choices
