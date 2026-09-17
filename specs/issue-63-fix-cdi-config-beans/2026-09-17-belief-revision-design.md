# Design Spec — Belief Revision from Contradicting Evidence (#67)

## Summary

Characters have seeded beliefs (e.g. "Penelope is naive and trusts too easily") that never change regardless of gameplay. This adds a consolidation phase that uses LLM-assessed semantic contradiction detection to gradually erode belief confidence when accumulated evidence contradicts a belief, and ultimately supersedes stale beliefs with LLM-generated revised beliefs when confidence drops below a threshold.

## What Exists

**Already in neocortex (platform):**
- `ConsolidationPhase` — SPI interface: `String name()`, `void run(String tenantId, List<String> subgraphPriority)`
- `ConsolidationScheduler` — runs phases sequentially on a daemon thread, per-phase try-catch, gated by `IdleTracker.isIdle(Duration.ofMinutes(1))`
- `ExperienceConsolidationPhase` (@Priority 15) — graduates memories into cognitive nodes via `GraduationClassifier` and `GraduationScorer`
- `MindMapStore.supersede(targetId, supersedingId, reason, tenantId)` — marks target node as superseded; history tracked via `SupersessionStatus`
- `Confidence` record — `(ConfidenceOrigin origin, double value, Instant decayReference)` with `withValue(double)` for adjustment. Origins: STATED, INFERRED, SPECULATED, UNKNOWN
- `SupersessionStatus` — tracks supersession history: target, superseder, reason, timestamps, reinstatement
- `GraduationClassifier` — classifies memories into cognitive kinds; `GraduationResult(String cognitiveKind, ConfidenceOrigin confidenceOrigin, Map<String,String> properties)`

**Already in blocks-core:**
- `AgentProvider` — platform SPI for LLM invocation, already used by `UserModelOrchestrator` for LLM synthesis
- `DriveAdaptationPhase` (@Priority 17) — consolidation phase pattern to follow

**Already in wacky-manor:**
- `ManorCognitiveSeeder.seed()` — seeds Belieflike nodes in per-agent `cognitive-{agentId}` subgraphs with `Confidence.stated(0.8, now)`, trait `Belieflike`, property `subject`, provenance `manor-seed`
- Initial beliefs in `social-config.yaml`: hooded-claw ("Penelope is naive and trusts too easily", "Peter Perfect is protective but predictable"), penelope-pitstop ("Sylvester Sneekly is a helpful estate manager", "Everyone here means well"), etc.

## Architecture

### 1. BeliefRevisionPhase (D24, D27)

New `BeliefRevisionPhase` in blocks-core, `@Priority(16)` — after ExperienceConsolidation@15 (which creates graduated nodes), before DriveAdaptation@17.

**Input:** `MindMapStore` (belief and graduated nodes), `AgentProvider` (LLM calls), `BeliefRevisionConfig` (decay/threshold parameters).

**Algorithm per consolidation cycle:**
1. List all `cognitive`-type subgraphs in the tenant
2. For each subgraph, collect Belieflike nodes (trait `Belieflike`, not superseded) and newly graduated experience nodes (provenance `experience-consolidation`)
3. Use a cursor node (same pattern as `DriveAdaptationPhase`) to track which graduated nodes have already been processed — only process nodes newer than the cursor
4. Group beliefs and new evidence by agent (using `agent-id` property on nodes and `principalId` on beliefs)
5. If an agent has both active beliefs and new evidence, invoke LLM contradiction detection (§2)
6. Apply confidence decay for detected contradictions (§3)
7. If any belief's confidence drops below threshold, supersede it (§4)
8. Update cursor

**Failure handling (D29):** LLM call failures are caught, logged as WARNING, and the phase returns without modifying beliefs. ConsolidationScheduler's per-phase try-catch ensures subsequent phases (DriveAdaptation@17, RelationshipStage@18) proceed normally.

### 2. LLM Contradiction Detection (D25)

One LLM call per agent per consolidation cycle. The prompt presents all active beliefs and all new evidence in a single batch.

**System prompt:**
```
You are analyzing a character's beliefs against recent evidence from their experiences.
For each belief that is contradicted by the evidence, identify the contradiction.
Respond with JSON only. If no contradictions are found, return {"contradictions": []}.
```

**User prompt:**
```
Character: {agentId}

Current beliefs:
1. [{beliefNodeId}] "{beliefText}" (confidence: {confidence})
2. [{beliefNodeId}] "{beliefText}" (confidence: {confidence})
...

Recent evidence:
- "{graduatedNodeText}" (event-type: {eventType})
- "{graduatedNodeText}" (event-type: {eventType})
...

For each belief contradicted by this evidence, provide:
- which belief (by ID)
- what evidence contradicts it
- how strong the contradiction is (0.0 = barely relevant, 1.0 = directly disproven)
- what the character should now believe (one sentence, from their perspective)
```

**Structured JSON output schema:**
```json
{
  "contradictions": [
    {
      "beliefNodeId": "node-id-123",
      "beliefText": "Penelope is naive and trusts too easily",
      "contradictingEvidence": "Penelope saw through the disguise and alerted Peter",
      "reasoning": "Penelope demonstrated perceptiveness, directly contradicting the belief that she is naive",
      "contradictionStrength": 0.8,
      "revisedBelief": "Penelope is more perceptive than she appears"
    }
  ]
}
```

The `contradictionStrength` field (0.0–1.0) modulates the confidence decay (§3). The `revisedBelief` field is used only when confidence drops below the supersession threshold — it is pre-generated in every response to avoid a second LLM call.

### 3. Confidence Decay (D26)

For each contradiction detected by the LLM:

```
effectiveDecay = beliefDecayPerContradiction × contradictionStrength
newConfidence = currentConfidence - effectiveDecay
```

- `beliefDecayPerContradiction` (default: 0.15) — configurable base decay per contradiction
- `contradictionStrength` (0.0–1.0) — from LLM output
- Confidence is clamped to [0.0, 1.0]

The decay is applied via `Confidence.withValue(newConfidence)` and persisted by updating the node's confidence in MindMapStore.

**Example progression** (default config, starting at 0.8):
- Strong contradiction (0.8): `0.8 - (0.15 × 0.8) = 0.68`
- Another strong (0.8): `0.68 - 0.12 = 0.56`
- Weak contradiction (0.3): `0.56 - 0.045 = 0.515`
- Strong contradiction (0.9): `0.515 - 0.135 = 0.38`
- Strong contradiction (0.8): `0.38 - 0.12 = 0.26` → below threshold (0.3), triggers supersession

### 4. Belief Supersession (D26, D28)

When a belief's confidence drops below `beliefSupersessionThreshold` (default: 0.3):

1. Create a new Belieflike node in the same subgraph:
   - Name: the `revisedBelief` text from the LLM's JSON output
   - Trait: `Belieflike`
   - Property `subject`: same as the superseded belief's subject
   - Confidence: `Confidence.inferred(0.6, Instant.now())` — INFERRED origin distinguishes evolved beliefs from authored (STATED) ones. Starting at 0.6 (not 0.8) reflects that the revised belief is less certain than an authored seed.
   - Provenance: `belief-revision`
   - PrincipalId: same agent as the superseded belief

2. Supersede the old belief: `mindMapStore.supersede(oldNodeId, newNodeId, reason, tenantId)`
   - Reason: the LLM's `reasoning` text (e.g., "Penelope demonstrated perceptiveness, directly contradicting the belief that she is naive")

3. The superseded belief remains in the mindmap with `SupersessionStatus.superseded = true`. Characters can "remember what they used to believe" — the history is queryable.

### 5. Configuration (D27)

New `BeliefRevisionConfig` record in blocks-core:

```java
public record BeliefRevisionConfig(
    double beliefDecayPerContradiction,
    double beliefSupersessionThreshold,
    double revisedBeliefInitialConfidence
) {
    public static BeliefRevisionConfig defaults() {
        return new BeliefRevisionConfig(0.15, 0.3, 0.6);
    }
}
```

The `AgentProvider` for LLM calls is injected via CDI — wacky-manor already wires this. No application-level configuration needed beyond providing `BeliefRevisionConfig` (defaults are sufficient for v1).

## Data Flow

```
scenario bootstrap
  └─ ManorCognitiveSeeder.seed()
       └─ Belieflike nodes created in cognitive-{agentId} subgraph
          confidence: 0.8, origin: STATED, provenance: manor-seed

interaction loop
  └─ recordExperience() stores memories

consolidation (sleep cycle)
  ├─ ExperienceConsolidationPhase (@Priority 15)
  │    └─ graduates memories → cognitive nodes (provenance: experience-consolidation)
  │
  ├─ BeliefRevisionPhase (@Priority 16)
  │    ├─ scans cognitive subgraphs for Belieflike + new graduated nodes
  │    ├─ groups by agent
  │    ├─ LLM call: batch contradiction detection per agent
  │    │    └─ returns: [{beliefNodeId, contradictionStrength, revisedBelief}]
  │    ├─ for each contradiction:
  │    │    ├─ decay confidence: current - (baseDecay × strength)
  │    │    └─ if below threshold → supersede + create revised belief
  │    └─ update cursor
  │
  ├─ DriveAdaptationPhase (@Priority 17)
  └─ RelationshipStagePhase (@Priority 18)

rendering (prompt construction)
  └─ CharacterCognition.renderCognitiveSections()
       └─ reads Belieflike nodes → renders active beliefs
          (superseded beliefs are filtered out by SupersessionStatus)
```

## Files Changed

### blocks-core (new)
| File | Change |
|------|--------|
| `BeliefRevisionPhase.java` | New ConsolidationPhase, @Priority(16) — LLM contradiction detection, confidence decay, supersession |
| `BeliefRevisionConfig.java` | Configuration record: decay, threshold, initial revised confidence |

### wacky-manor (tests)
| File | Change |
|------|--------|
| `BeliefRevisionIntegrationTest.java` | End-to-end: seed beliefs → add graduated evidence → run phase → verify confidence decay and supersession |

## Trade-offs Acknowledged

- **LLM in consolidation (D29):** BeliefRevisionPhase is the first consolidation phase to invoke an LLM. ConsolidationScheduler's per-phase try-catch and idle-time gating handle failure isolation and latency. LLM calls that hang block subsequent phases until the next scheduler tick — AgentProvider implementations must enforce their own call timeouts.
- **Non-deterministic contradiction detection:** The LLM may assess the same (belief, evidence) pair differently across runs. Acceptable — belief revision is inherently subjective. The `contradictionStrength` modulates a configurable base decay, bounding extreme variation.
- **Batch prompt limitations (D25):** One LLM call per agent amortizes cost but may miss subtle pairwise contradictions. Characters have 4-6 beliefs and consolidation graduates ~5-20 nodes per cycle, so the prompt is bounded. Can switch to pairwise for high-value beliefs in a future iteration.
- **Uniform base decay (D26):** While `contradictionStrength` varies, the `baseDecay` is uniform across all beliefs. A deeply-held belief and a casual assumption both decay at the same base rate. Acceptable for v1 — per-belief decay rates would add configuration complexity with marginal benefit.
- **Revised belief confidence (D28):** Revised beliefs start at 0.6 (INFERRED), not 0.8 (STATED). This means a revised belief can itself be superseded more quickly by further contradicting evidence. Intentional — evolved beliefs should be less entrenched than authored ones.
- **Pre-generated revised text:** The `revisedBelief` field is generated in every LLM response, even when confidence hasn't reached the threshold yet. The unused text is discarded. This avoids a second LLM call when supersession does trigger, at the cost of generating text that may never be used.

## References

- `ConsolidationPhase.java` (neocortex-mindmap-intelligence) — SPI interface
- `ConsolidationScheduler` (neocortex-mindmap-intelligence) — per-phase try-catch, IdleTracker gating
- `ExperienceConsolidationPhase.java` (neocortex-mindmap-intelligence:88-153) — graduation pattern, @Priority(15)
- `DriveAdaptationPhase.java` (blocks-core) — consolidation phase pattern, cursor mechanism
- `MindMapStore.supersede()` (neocortex-mindmap-api) — supersession API
- `Confidence.java` (neocortex-cognitive-api) — confidence model with `withValue()`, STATED/INFERRED origins
- `SupersessionStatus.java` (neocortex-mindmap-api) — supersession history tracking
- `GraduationClassifier.java` (neocortex-memory-api) — classification interface
- `ManorCognitiveSeeder.seed()` (wacky-manor:100-109) — belief seeding with Belieflike trait
- `AgentProvider.java` (platform-agent-api) — LLM invocation SPI
- `UserModelOrchestrator.java` (blocks-core) — LLM synthesis pattern in blocks-core
- Issue #67, #52
