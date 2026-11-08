# Proactive Regulation: Pre-Appraisal Situation Avoidance from Emotional Memory

**Issue:** casehubio/neocortex#819
**Branch:** issue-851-dynamic-regulatory-capacity
**Date:** 2026-11-08

## Overview

Characters currently regulate emotions only reactively — after appraisal detects an emotional discrepancy. Proactive regulation adds anticipatory avoidance: a character who was humiliated in the library last tick feels reluctance to enter the library this tick, before appraisal fires.

This implements Gross's (1998) situation selection strategy as a memory-driven, pre-appraisal mechanism. A new `ProactiveRegulationParticipant` in the FOUNDATION phase queries past regulation episodes from MindMap, evaluates whether the current context matches a previously-triggering situation, and generates avoidance goals via the existing goal pipeline.

## Literature Foundation

| Model | Contribution | How used |
|-------|-------------|----------|
| Gross 1998 | Situation selection as antecedent-focused regulation | Core mechanism — avoidance before emotion fires |
| Gross 2015 (EPM) | Identification stage — recognising a situation as emotionally relevant before engaging | Justifies FOUNDATION placement — recognition before appraisal |
| Bonanno & Burton 2013 | Regulatory flexibility — rigid avoidance is maladaptive | Temporal decay prevents permanent avoidance bias |
| Bosse, Pontier & Treur 2010 | Quantitative dynamics for regulation thresholds | Threshold calibration via redirectTendency |

## Architecture

### Pipeline Placement

A new `ProactiveRegulationParticipant` runs in the FOUNDATION phase of `CognitionCore.tick()`, after the existing `SubThoughtTickParticipant` and before SOURCE phase processing.

```
CognitionCore.tick() — FOUNDATION phase:
  mood.tick()                              ← built-in
  memoryHygiene()                          ← built-in
  runCustomParticipants(FOUNDATION):
    1. SubThoughtTickParticipant            ← existing (extracts sub-thoughts from observation)
    2. ProactiveRegulationParticipant       ← NEW (queries regulation history, generates avoidance goals)
```

**Ordering constraint:** Must run after SubThoughtTickParticipant (insertion order via `addParticipant`). A unit test should verify ordering in the FOUNDATION participant list.

### Context Provision

A new `RegulationContext` record carries structured context into the tick pipeline:

```java
public record RegulationContext(
    @Nullable String location,
    @Nullable List<String> interactingAgents,
    @Nullable String topic
) {}
```

`CognitionTickContext` gains a `@Nullable RegulationContext regulationContext` field via a new telescoping constructor. Zero existing callers break — they continue using existing constructors that pass null for the new field. Callers that can provide context (e.g., wacky-manor's tick driver) construct and pass a RegulationContext; callers that can't pass null.

### Write Path — Storing Context on Regulation Episodes

`EmotionRegulationMindMapAdapter.write()` gains three optional string properties on each regulation node:

| Property | Source | Purpose |
|----------|--------|---------|
| `regulationLocation` | `RegulationContext.location()` | Room/place where regulation occurred |
| `regulationAgents` | CSV of `RegulationContext.interactingAgents()` | Characters present during regulation |
| `regulationTopic` | `RegulationContext.topic()` | Topic/subject that triggered the emotion |

All three are nullable — omitted when RegulationContext is null or a dimension is null. This is a write-time enrichment: every regulation episode automatically captures its full context regardless of the current proactive avoidance mode. Historical data retroactively gains value when new dimensions are promoted.

### Read Path — Proactive Avoidance Evaluation

`ProactiveRegulationParticipant.tick()`:

1. **Guard:** Skip if RegulationContext is null, or if `CognitiveDefaultsRegistry` has no defaults for this agent (no personality → no threshold).
2. **Query:** Load recent regulation episodes from MindMap via `EmotionRegulationMindMapAdapter.loadHistory()` (limit: 20, unfiltered).
3. **Filter by mode:** For each episode, check whether the active dimensions match the current context:
   - `LOCATION_ONLY` — compare `regulationLocation` to current `location`
   - `LOCATION_AND_AGENTS` — also compare `regulationAgents` intersection
   - `FULL` — also compare `regulationTopic`
4. **Filter by recency:** Compute decay weight: `1.0 - (ticksSinceEpisode / decayHorizon)`. Skip episodes with decay weight ≤ 0 (fully decayed).
5. **Filter by threshold:** Compute avoidance score: `(1.0 - effectiveness) * decayWeight`. The `(1.0 - effectiveness)` term means failed regulation (low effectiveness) produces stronger avoidance. Compare against `redirectTendency` from `RegulationDefaults` — agents with high redirectTendency (avoidant, conservative) avoid at lower scores.
6. **Generate goals:** For each surviving episode, create a `DriveGoalProposal` with:
   - `axis`: the episode's `regulationAxis` (or `AUTONOMY` as fallback)
   - `goalName`: `"proactive-avoidance-<location>"`
   - `goalDescription`: `"Avoid <location> — previous emotional distress (<emotionType>)"`
   - `formationReason`: `"Proactive regulation: memory-based anticipatory avoidance"`
   - `driveIntensity`: the avoidance score (0.0-1.0)
7. **Register:** Call `GoalProposalOrchestrator.appendGoals()` with the generated proposals.

### Proactive Avoidance Mode

```java
public enum ProactiveAvoidanceMode {
    LOCATION_ONLY,
    LOCATION_AND_AGENTS,
    FULL
}
```

Configurable per application via Quarkus config (`manor.regulation.proactive.mode`). Default: `LOCATION_ONLY`. Controls which context dimensions the proactive participant queries — all dimensions are always stored.

### Temporal Decay

Avoidance weight decays linearly from 1.0 to 0.0 over a configurable horizon (default: 50 ticks). An episode at tick 0 has full weight; at tick 50 it has zero weight and is ignored.

This prevents the permanent avoidance bias identified in the decision review (R1-15, Bonanno & Burton 2013). Characters naturally "forget" avoidance dispositions unless the triggering situation recurs. If the agent encounters the situation again and regulates successfully (high effectiveness), the new positive episode counterbalances the old negative one.

The decay horizon is configurable (`manor.regulation.proactive.decayTicks`, default 50) for empirical tuning.

### Personality Modulation

The avoidance threshold uses `redirectTendency` from `RegulationDefaults` (already derived by `CognitiveDerivationEngine` from all five disposition axes):

- **High redirectTendency** (avoidant conflict, conservative risk, low autonomy) → low threshold → proactive avoidance fires readily
- **Low redirectTendency** (analytical conflict, bold risk, high autonomy) → high threshold → most episodes don't trigger avoidance

The threshold comparison: `avoidanceScore > (1.0 - redirectTendency)`. An agent with `redirectTendency = 0.8` avoids when score > 0.2 (easily triggered). An agent with `redirectTendency = 0.2` avoids when score > 0.8 (rarely triggered).

### Goal Output

A new `appendGoals()` method on `GoalProposalOrchestrator` provides additive semantics:

```java
public void appendGoals(String agentId, String tenantId, List<DriveGoalProposal> goals) {
    var state = stateFor(agentId, tenantId);
    var merged = new ArrayList<>(state.registeredGoals);
    merged.addAll(goals);
    state.registeredGoals = List.copyOf(merged);
}
```

The existing `registerGoals()` keeps its replace semantics — `EmotionRegulationGoalParticipant` (TERMINAL) continues using it for reactive regulation goals. Proactive goals (FOUNDATION) use `appendGoals()` so they accumulate without being clobbered.

Registered goals bypass the priority/capacity system (they're concatenated in `currentProposals()` without sorting or limiting). This is acceptable: avoidance goals should always render regardless of drive-based priority. Unbounded growth is mitigated by temporal decay naturally limiting the active set.

### Interaction with Reactive Regulation

If the proactive participant generates an avoidance goal for the library, and the agent enters the library anyway (bold personality overrides avoidance), reactive regulation may fire and select SITUATION_SELECTION again. Both signals reach the goal system:

- Proactive: via `appendGoals()` from FOUNDATION — persists across ticks
- Reactive: via `registerGoals()` from TERMINAL — replaced each tick

No deduplication needed — the goal pipeline renders both, and the agent's action selection naturally combines them. The reactive signal reinforces the proactive one.

The FOUNDATION placement means proactive avoidance goals are visible to `CognitiveLoadEstimator` within the same tick. This feedback loop is intentional: an agent carrying avoidance dispositions has higher cognitive load, which can shift the reactive predictor toward low-cost strategies via `predictUnderHighLoad()`.

## New Components

| Component | Module | Type |
|-----------|--------|------|
| `RegulationContext` | cognition-api | record |
| `ProactiveAvoidanceMode` | cognition-api | enum |
| `ProactiveRegulationParticipant` | cognition | CognitionTickParticipant |
| `appendGoals()` | cognition | method on GoalProposalOrchestrator |
| CognitionTickContext constructor | cognition-api | telescoping constructor |
| 3 properties on regulation nodes | cognition | write-path enrichment |

## Configuration

| Property | Default | Purpose |
|----------|---------|---------|
| `manor.regulation.proactive.enabled` | `true` | Feature toggle |
| `manor.regulation.proactive.mode` | `LOCATION_ONLY` | Which dimensions to query |
| `manor.regulation.proactive.decayTicks` | `50` | Decay horizon for avoidance weight |
| `manor.regulation.proactive.historyLimit` | `20` | Max regulation episodes to query |

## Testing Strategy

1. **Unit tests (ProactiveRegulationParticipant):**
   - Context match: location matches → avoidance goal generated
   - Context miss: different location → no goal
   - Threshold: high redirectTendency → avoidance fires; low → doesn't
   - Decay: episode at decay horizon → no avoidance; recent episode → avoidance
   - Mode: LOCATION_ONLY ignores agents/topic dimensions
   - Null context: graceful skip

2. **Unit tests (appendGoals):**
   - Additive semantics: appendGoals + registerGoals both visible in currentProposals
   - registerGoals replaces; appendGoals accumulates

3. **Integration test (pipeline ordering):**
   - ProactiveRegulationParticipant runs in FOUNDATION after SubThoughtTickParticipant
   - Goals registered in FOUNDATION visible in TERMINAL

4. **Validation test (scenario-level):**
   - Run 30-tick scenario with proactive regulation enabled
   - Verify avoidance goals appear for characters who had high-intensity regulation episodes
   - Verify decay: avoidance goals weaken over ticks

## Phase 2 — Future Extensions

- **CBR migration:** Replace MindMap in-memory filtering with CBR indexed retrieval when scale demands it (R1-09)
- **Spatial integration (#203):** When x-position rooms land, RegulationContext.location becomes structured coordinates instead of room names
- **Social regulation (#805):** Promote `LOCATION_AND_AGENTS` mode — avoidance of specific people in specific contexts
- **Probabilistic avoidance:** Instead of binary threshold, generate avoidance with probability proportional to score. Allows some exposure for learning.

## References

- `EmotionRegulationMindMapAdapter.java:29-48` — existing regulation node property storage
- `EmotionRegulationGoalParticipant.java:22-37` — existing reactive regulation→goal bridge
- `GoalProposalOrchestrator.java:116-120` — registerGoals() replace semantics
- `CognitiveDerivationEngine.java:516-521` — redirectTendency derivation from disposition axes
- `CognitionTickContext.java:7-35` — telescoping constructor pattern
- `CognitionCore.java:255-261,793-799` — FOUNDATION phase execution and SubThought registration
- `RegulationStrategyPredictor.java` — personality→regulation mapping (coherence model)
- casehubio/neocortex#590 — reactive regulation design spec
- casehubio/neocortex#203 — spatial room model (future integration)
- casehubio/neocortex#805 — social regulation (future integration)
- R1-02, R1-06, R1-12, R1-15 — decision review findings that shaped revisions
