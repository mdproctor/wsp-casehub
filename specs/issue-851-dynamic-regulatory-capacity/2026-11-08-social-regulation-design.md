# Social Emotion Regulation: Interpersonal Strategies

**Issue:** casehubio/neocortex#805
**Branch:** issue-851-dynamic-regulatory-capacity
**Date:** 2026-11-08

## Overview

Characters can regulate their own emotions (intrapersonal, #590) but cannot use social strategies. This design adds interpersonal regulation strategies from Zaki & Williams (2013): both intrinsic (using social means to regulate one's own emotions) and extrinsic (regulating another's emotions). These are the most dramatically observable regulation strategies in a social simulation — a character confiding in a friend to manage anxiety, withdrawing from a group to cool down, or soothing a distressed companion.

The design extends the existing regulation system rather than creating a parallel one: new enum values on `EmotionRegulationStrategy`, a `targetAgentId` field on `EmotionRegulationResult`, social strategy descriptions gated into the LLM prompt when social context is available, and personality modulation via a new `socialRegulationTendency` in `RegulationDefaults`.

## Literature Foundation

| Model | Contribution | How used |
|-------|-------------|----------|
| Zaki & Williams 2013 | Intrinsic vs. extrinsic interpersonal regulation taxonomy | Strategy classification — which direction does regulation flow |
| Gross 1998, 2015 | Process model and extended process model | Extends the existing 5-strategy framework with social variants |
| Niven et al. 2009 | Interpersonal affect regulation strategies inventory | Specific strategy definitions for co-regulation and provocation |
| Rimé 2009 | Social sharing of emotion — disclosure drives connection | SOCIAL_SHARING strategy specification |

## Architecture

### New Strategies

Five social strategies added to `EmotionRegulationStrategy`:

| Strategy | Direction | Description | Goal output |
|----------|-----------|-------------|-------------|
| `SOCIAL_SUPPORT_SEEKING` | Intrinsic | Seek comfort, reassurance, or advice from a specific agent | Approach goal targeting the agent |
| `SOCIAL_SHARING` | Intrinsic | Share emotional experience with another — disclosure, not advice-seeking | Interaction goal (talk to agent about topic) |
| `SOCIAL_WITHDRAWAL` | Intrinsic | Withdraw from social interaction to self-regulate in isolation | Avoidance goal (leave current location/interaction) |
| `CO_REGULATION` | Extrinsic | Actively soothe, comfort, or calm a distressed agent | Approach goal targeting the distressed agent |
| `DELIBERATE_PROVOCATION` | Extrinsic | Intentionally provoke an emotional response in another agent | Interaction goal targeting the agent |

**Enum update:**

```java
public enum EmotionRegulationStrategy {
    // Gross 1998 intrapersonal strategies (1-5)
    SITUATION_SELECTION,
    SITUATION_MODIFICATION,
    ATTENTIONAL_DEPLOYMENT,
    COGNITIVE_CHANGE,
    RESPONSE_MODULATION,
    // Zaki & Williams 2013 social strategies
    SOCIAL_SUPPORT_SEEKING,
    SOCIAL_SHARING,
    SOCIAL_WITHDRAWAL,
    CO_REGULATION,
    DELIBERATE_PROVOCATION
}
```

### Strategy Classification

`EmotionRegulationResult` gains two new classification methods:

```java
public boolean isSocialStrategy() {
    return strategy == SOCIAL_SUPPORT_SEEKING
        || strategy == SOCIAL_SHARING
        || strategy == SOCIAL_WITHDRAWAL
        || strategy == CO_REGULATION
        || strategy == DELIBERATE_PROVOCATION;
}

public boolean isExtrinsicStrategy() {
    return strategy == CO_REGULATION
        || strategy == DELIBERATE_PROVOCATION;
}
```

The existing `isEnvironmentalStrategy()` is unchanged — SITUATION_SELECTION and SITUATION_MODIFICATION remain environmental. Social strategies are a new category that routes through its own goal construction logic.

### EmotionRegulationResult — targetAgentId

A new nullable field `targetAgentId` identifies the social target:

```java
public record EmotionRegulationResult(
    EmotionRegulationStrategy strategy,
    List<String> targetEmotions,
    List<String> targetDrives,
    double peakOriginalIntensity,
    double regulatedIntensity,
    double effectiveness,
    String narrative,
    @Nullable DriveAxis regulationAxis,
    @Nullable String environmentalDirective,
    @Nullable String targetAgentId          // NEW
) { ... }
```

**Validation:** The `EmotionRegulationTickParticipant` validates `targetAgentId` against `RegulationContext.interactingAgents()` after LLM generation. If the LLM outputs an agent ID not in the available list, the result is discarded and regulation falls back to no-regulation for that tick. This prevents hallucinated agent IDs from producing invalid goals.

**Backward compatibility:** Existing constructors and callers pass `null` for `targetAgentId`. A telescoping constructor chain (following the CognitionTickContext pattern) ensures zero breakage.

### Unified Strategy Selection — Social Gate

The `EmotionRegulationGenerationPhase` system prompt is extended with social strategy descriptions. These are **conditionally included** based on `RegulationContext`:

1. If `context.regulationContext()` is null or `interactingAgents()` is empty → only the existing 5 Gross strategies appear in the prompt
2. If `interactingAgents()` is non-empty → all 10 strategies appear, with available agents listed by ID
3. If `RegulationTracker` trajectory data exists for nearby agents → distress trajectories are included as context for extrinsic strategy selection

**Additional prompt content (appended when social context available):**

```
SOCIAL REGULATION STRATEGIES (Zaki & Williams 2013):

6. SOCIAL_SUPPORT_SEEKING — Seek comfort, reassurance, or advice from
   a specific person present. Intrinsic: regulates YOUR emotions through
   social connection. Select when: emotional intensity is moderate,
   a trusted/affiliated person is available.
7. SOCIAL_SHARING — Share your emotional experience with another person.
   Not advice-seeking — disclosure for connection and relief. Intrinsic.
   Select when: emotion needs expression, audience is receptive.
8. SOCIAL_WITHDRAWAL — Remove yourself from social interaction to
   self-regulate alone. Intrinsic. Select when: social presence
   amplifies distress, or emotion is too intense for social strategies.
9. CO_REGULATION — Actively comfort or soothe a person you observe
   to be in distress. Extrinsic: regulates THEIR emotions. Select when:
   you observe distress in someone you care about, and your own
   emotional state allows helping.
10. DELIBERATE_PROVOCATION — Intentionally provoke an emotional response
    in another person. Extrinsic. Select when: strategic provocation
    serves a goal (e.g., provoking a rival, testing loyalty). Rare —
    personality must support it (low agreeableness, high scheming).

Available agents: [agent-id-1, agent-id-2, ...]
For social strategies, set targetAgentId to ONE of the available agents.

OBSERVED DISTRESS (for extrinsic strategies):
[agent-id: trajectory (IMPROVING/STABLE/WORSENING), delta]
```

**JSON schema extension:**

```json
{
  "strategy": "STRATEGY_NAME",
  "targetEmotions": ["emotion1"],
  "targetDrives": ["drive1"],
  "peakOriginalIntensity": 0.85,
  "regulatedIntensity": 0.4,
  "effectiveness": 0.6,
  "narrative": "Second-person present-tense description",
  "regulationAxis": "AXIS_NAME or null",
  "environmentalDirective": "directive text or null",
  "targetAgentId": "agent-id or null"
}
```

### Goal Routing for Social Strategies

The existing `EmotionRegulationGoalParticipant` is extended to handle social strategies alongside environmental ones:

```java
@Override
public void tick(CognitionTickContext context) {
    var result = regulationParticipant.currentResult(context.agentId(), context.tenantId());
    if (result.isEmpty()) return;

    var reg = result.get();
    if (reg.isEnvironmentalStrategy()) {
        // Existing environmental goal logic (unchanged)
        if (reg.regulationAxis() == null || reg.environmentalDirective() == null) return;
        var proposal = new DriveGoalProposal(
                reg.regulationAxis(),
                "regulation-" + reg.strategy().name().toLowerCase().replace('_', '-'),
                reg.environmentalDirective(),
                "Emotion regulation: " + reg.narrative(),
                reg.regulatedIntensity());
        goalOrchestrator.registerGoals(context.agentId(), context.tenantId(), List.of(proposal));
    } else if (reg.isSocialStrategy()) {
        // New social goal logic
        var axis = reg.regulationAxis() != null ? reg.regulationAxis() : DriveAxis.AFFILIATION;
        String goalName = "social-regulation-" + reg.strategy().name().toLowerCase().replace('_', '-');
        String description = reg.narrative();
        if (reg.targetAgentId() != null) {
            description += " [target: " + reg.targetAgentId() + "]";
        }
        var proposal = new DriveGoalProposal(
                axis, goalName, description,
                "Social regulation: " + reg.strategy().name(),
                reg.regulatedIntensity());
        goalOrchestrator.appendGoals(context.agentId(), context.tenantId(), List.of(proposal));
    }
}
```

**Key difference:** Social goals use `appendGoals()` (additive, from #819) instead of `registerGoals()` (replace). This prevents social regulation goals from clobbering proactive avoidance goals or other registered goals.

### Personality Modulation — socialRegulationTendency

`RegulationDefaults` gains a fourth field:

```java
public record RegulationDefaults(
    double reappraisalCapacity,
    double suppressionCapacity,
    double redirectTendency,
    double socialRegulationTendency    // NEW
) {}
```

**Derivation in CognitiveDerivationEngine:**

```java
double social = 0.5
    + SOCIAL_REG_SOCIAL.getOrDefault(lower(axes.socialOrient()), 0.0)
    + SOCIAL_REG_CONFLICT.getOrDefault(lower(axes.conflictMode()), 0.0)
    + SOCIAL_REG_AUTONOMY.getOrDefault(lower(axes.autonomy()), 0.0);
```

Weight maps (initial values for empirical tuning):

| Axis value | Weight | Rationale |
|-----------|--------|-----------|
| socialOrient: gregarious | +0.20 | Gregarious characters seek social interaction for regulation |
| socialOrient: reserved | -0.15 | Reserved characters prefer solitary regulation |
| socialOrient: selective | +0.05 | Selective — slight preference for targeted social support |
| conflictMode: harmonizing | +0.15 | Harmonizers value social connection, seek comfort |
| conflictMode: confrontational | -0.10 | Confrontational — less likely to seek social support |
| conflictMode: analytical | 0.00 | Neutral |
| autonomy: independent | -0.10 | Independent characters self-regulate |
| autonomy: collaborative | +0.10 | Collaborative characters co-regulate |
| autonomy: dependent | +0.15 | Dependent characters seek support readily |

**Prompt integration:** `socialRegulationTendency` is provided to the LLM as a selection factor:
- High (>0.6) → "Character prefers social regulation strategies when others are available"
- Low (<0.4) → "Character prefers intrapersonal strategies even in social contexts"
- Middle → "Character selects based on situation"

### Interaction with Existing Systems

**Proactive regulation (#819):** `ProactiveRegulationParticipant` evaluates avoidance in FOUNDATION phase. Social regulation fires later in DERIVED phase. A character may have both a proactive avoidance goal (from #819) and a social regulation goal (from #805) in the same tick — `appendGoals()` ensures both coexist.

**Reactive regulation (#590):** Social strategies are a natural extension. The activation gate, fatigue tracking, and effectiveness decay all apply to social strategies identically. A character who repeatedly seeks comfort will see diminishing effectiveness (Bosse 2010 adaptation).

**RegulationTracker (Theory of Mind):** The tracker observes distress trajectories for other agents. This data is included in the LLM prompt for extrinsic strategy selection. No changes to RegulationTracker itself — it already provides the trajectory API.

**RegulationContext (#819):** The `interactingAgents` field, originally designed for proactive avoidance agent matching, becomes the social gate: non-empty → social strategies available.

### Backward Compatibility

- `EmotionRegulationResult` gains a 10th field via telescoping constructor. Existing callers pass `null` for `targetAgentId`.
- `RegulationDefaults` gains a 4th field. Existing 3-arg constructors still work (add a 3-arg constructor that passes 0.5 for `socialRegulationTendency`).
- `EmotionRegulationStrategy` enum gains 5 values. Existing switch/if-else chains use `isEnvironmentalStrategy()` and won't break — they simply won't match the new values.
- The LLM prompt extension is conditional on `RegulationContext` — callers without social context see no change.

## New Components

| Component | Module | Type | Change |
|-----------|--------|------|--------|
| 5 new enum values | cognition-api | enum extension | Extend `EmotionRegulationStrategy` |
| `isSocialStrategy()` | cognition-api | method | Add to `EmotionRegulationResult` |
| `isExtrinsicStrategy()` | cognition-api | method | Add to `EmotionRegulationResult` |
| `targetAgentId` field | cognition-api | record field | Add to `EmotionRegulationResult` |
| `socialRegulationTendency` | cognitive-index | record field | Add to `RegulationDefaults` |
| Social derivation weights | cognitive-index | static maps | Add to `CognitiveDerivationEngine` |
| Social prompt extension | cognition | prompt section | Extend `EmotionRegulationGenerationPhase` |
| Social goal routing | cognition | participant extension | Extend `EmotionRegulationGoalParticipant` |
| targetAgentId validation | cognition | validation | Add to `EmotionRegulationTickParticipant` |
| Social context in user message | cognition | prompt content | Extend `EmotionRegulationGenerationPhase.buildUserMessage()` |

## Configuration

| Property | Default | Purpose |
|----------|---------|---------|
| `manor.regulation.social.enabled` | `true` | Feature toggle for social strategies |

No other new config — social strategies use the existing regulation config (`regulationEnabled`, `regulationThreshold`, etc.).

## Testing Strategy

1. **Unit tests (EmotionRegulationStrategy):** `isSocialStrategy()` and `isExtrinsicStrategy()` classification
2. **Unit tests (EmotionRegulationResult):** Telescoping constructors, backward compat, targetAgentId accessors
3. **Unit tests (RegulationDefaults):** 4-arg constructor, backward-compat 3-arg constructor
4. **Unit tests (CognitiveDerivationEngine):** socialRegulationTendency derivation for gregarious/reserved/mixed profiles
5. **Unit tests (EmotionRegulationGoalParticipant):** Social strategy → goal proposal with targetAgentId, appendGoals semantics
6. **Unit tests (EmotionRegulationTickParticipant):** targetAgentId validation against RegulationContext
7. **Unit tests (EmotionRegulationGenerationPhase):** Social prompt gating — verify social strategies appear only when interactingAgents present
8. **Integration test:** Full tick with social context → social strategy selected → social goal produced

## Phase 2 — Future Extensions

- **Reciprocal co-regulation:** When agent A co-regulates agent B, agent B's next tick should reflect the social support (requires cross-agent regulation state propagation)
- **Social regulation memory:** Store social regulation episodes with target in MindMap — enables relationship-aware regulation ("I've sought comfort from this agent before, it worked well")
- **Group dynamics (#806):** Social regulation in CAPS — social strategy dispositions as CAUs
- **Spatial integration (#203):** SOCIAL_WITHDRAWAL produces movement goals with spatial coordinates

## References

- `EmotionRegulationStrategy.java:4-11` — existing 5-strategy enum
- `EmotionRegulationResult.java:7-24` — existing result record with isEnvironmentalStrategy()
- `EmotionRegulationGoalParticipant.java:10-39` — existing environmental goal bridge
- `EmotionRegulationGenerationPhase.java:27-74` — system prompt with strategy descriptions
- `EmotionRegulationTickParticipant.java:88-182` — tick() method with generation call
- `RegulationDefaults.java:18-22` — existing 3-field record
- `CognitiveDerivationEngine.java:502-525` — deriveRegulationDefaults() with disposition axis weights
- `RegulationTracker.java:9-36` — distress trajectory observation
- `RegulationContext.java` — location, interactingAgents, topic (from #819)
- `GoalProposalOrchestrator.java:121-128` — appendGoals() additive semantics (from #819)
- `ProactiveRegulationParticipant.java` — FOUNDATION-phase proactive regulation (from #819)
- Zaki, J. & Williams, W.C. (2013) — interpersonal emotion regulation taxonomy
- Niven, K. et al. (2009) — interpersonal affect regulation strategies inventory
- Rimé, B. (2009) — social sharing of emotion
- casehubio/neocortex#590 — intrapersonal regulation design spec
- casehubio/neocortex#819 — proactive regulation (RegulationContext, appendGoals)
- casehubio/neocortex#805 — this issue
- decisions-805.md — D1-D6 decisions captured during brainstorming
