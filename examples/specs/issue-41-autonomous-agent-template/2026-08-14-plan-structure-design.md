# Plan Structure — Replace currentPlan String with Structured Plans

**Issue:** casehubio/examples#44
**Date:** 2026-08-14
**Depends on:** #43 (goal lifecycle — plans decompose goals into steps)
**Engine dependency:** `casehub-engine-api` already on classpath (added in #43)

## Design Principle

Three-layer cognitive model, mapping to System 1 / System 2:

| Layer | Concern | Persistence | Author | Update cadence |
|-------|---------|------------|--------|---------------|
| **Goals** (strategic) | What to pursue | AgentDescriptor | GoalFormationStrategy (reflection-driven) | On reflection |
| **Plans** (tactical) | How to pursue a goal | CharacterState | ManorPlanFormationStrategy (goal-driven) | On goal change + revision |
| **Thinking** (reactive) | What to do this tick | CharacterState | Per-tick LLM response | Every tick |

Goals emerge from reflection. Plans decompose goals into steps. Thinking
reasons about the immediate situation using goals and plans as context.
Each layer has a different author, cadence, and persistence model.

## Architecture

### Data Flow

```
Goal lifecycle (from #43):
  Reflection → ManorGoalFormationStrategy → new goals on AgentDescriptor
                                                    ↓
Plan lifecycle (this issue):                        ↓
  New goal formed ──────────────────→ ManorPlanFormationStrategy
                                       → per-goal plan on CharacterState
                                                    ↓
  Action failure (reactive) ────────→ ManorPlanRevisionStrategy
  Reflection (deliberative) ────────→   → revised plan on CharacterState
                                                    ↓
Per-tick:                                           ↓
  ObservationBuilder reads goals + plans ───→ observation prompt
  LLM produces thinking + action ───→ stored on CharacterState
```

### Plan Data Model

```java
public record AgentPlan(
    String goalName,
    List<PlanStep> steps,
    String rationale,
    int creationTick,
    int lastRevisionTick,
    int revisionGeneration
) {}

public record PlanStep(
    String id,
    String description,
    PlanStepStatus status
) {}

public enum PlanStepStatus {
    PENDING, IN_PROGRESS, COMPLETED, FAILED
}
```

Plans are stored on `CharacterState` as `Map<String, AgentPlan>` keyed by
goal name, replacing the `String currentPlan` field. The `currentPlan()`
/ `setCurrentPlan()` methods are removed. A new `plans()` accessor returns
the map; `setPlan(String goalName, AgentPlan plan)` and
`removePlan(String goalName)` handle updates.

`PlanStep` maps to the engine's `PlanStepDescriptor` (id, description,
capabilityName). `capabilityName` is set to null — the manor has no
worker capabilities. `PlanStepStatus` is a manor-local enum; the engine's
`PlanItemStatus` has 8 states tied to case execution. The manor needs 4.

### Plan-Goal Lifecycle Coupling

Plans are tied to goal lifecycle:
- **Goal formed** → plan formation triggered for that goal
- **Goal revised** → plan revision triggered (goal description changed)
- **Goal abandoned** → plan removed from CharacterState
- **Goal completed** → plan removed from CharacterState

This coupling is managed by `ManorGoalEvaluator`, which already handles
goal lifecycle events. After processing goal formation/revision/abandonment,
the evaluator chains plan formation/revision/removal.

## New Components

### ManorPlanFormationStrategy

Manor-local strategy (no engine SPI for plan formation). Uses
`AgentProvider` for LLM calls. Called by `ManorGoalEvaluator` after
a new goal is registered.

```java
public class ManorPlanFormationStrategy {

    public AgentPlan formPlan(String agentId, String tenancyId,
            AgentGoal goal, List<AgentGoal> allGoals,
            List<RetrievedMemory> memories, int currentTick) {
        // Build prompt: "Given this goal and the agent's current context,
        //   decompose the goal into 2-5 concrete, actionable steps."
        // Parse response into AgentPlan
    }
}
```

**Input:** The goal to decompose, all current goals (for cross-goal
awareness), recent memories (for world context), current tick.

**Output:** `AgentPlan` with 2-5 steps, all initially `PENDING`.

**Prompt constraints:** Steps must be concrete actions the character can
perform (MOVE, TAKE, USE, INTERACT, etc.), not abstract objectives.
Each step should be achievable in 1-3 ticks.

### ManorPlanRevisionStrategy

Implements `PlanRevisionStrategy` from `casehub-engine-api`. Uses
`AgentProvider` for LLM calls.

```java
@ApplicationScoped
public class ManorPlanRevisionStrategy implements PlanRevisionStrategy {

    @Override
    public String id() { return "manor-llm"; }

    @Override
    public RevisedPlan revise(RevisionContext context) {
        // Build prompt from context:
        //   - goalName (from AdaptationContext)
        //   - completedSteps, pendingSteps (current plan state)
        //   - cause (what triggered revision)
        //   - memories (recent context)
        // Ask LLM: "Given what succeeded, what failed, and why,
        //   propose a revised plan for this goal."
        // Parse response into RevisedPlan (List<PlanStepDescriptor> + rationale)
    }
}
```

**AdaptationContext construction (case fields nulled):**

| Field | Value |
|-------|-------|
| `caseId` | `null` |
| `tenancyId` | `"wacky-manor"` |
| `compoundId` | `null` |
| `goalName` | The AgentGoal being pursued |
| `completedSteps` | Steps with status COMPLETED, mapped to `CompletedStep` |
| `pendingSteps` | Steps with status PENDING/IN_PROGRESS, mapped to `PlanStepDescriptor` |
| `runningSteps` | Empty list (manor doesn't have concurrent step execution) |
| `currentContext` | `null` |
| `definition` | `null` |
| `latestStatus` | `null` |
| `latestBindingName` | `null` |
| `adaptationGeneration` | `plan.revisionGeneration()` |

**RevisionContext construction:**

| Field | Value |
|-------|-------|
| `adaptationContext` | As above |
| `cause` | Manor-local `AdaptationCause` impl (action failure or reflection trigger) |
| `capabilities` | Empty list |
| `memories` | Recent memories from `CaseMemoryStore` |

**Output mapping:** `RevisedPlan.steps()` (List<PlanStepDescriptor>) mapped
back to `List<PlanStep>` with all statuses reset to PENDING.
`PlanStepDescriptor.capabilityName()` is ignored.

## Plan Revision Triggers (D5)

### Reactive: Action Failure

In `ScenarioOrchestrator.runAutonomousTicks()`, after action resolution:

```java
if (result.isFailure()) {
    // Find which goal's plan the action likely relates to
    // (or revise all plans — the LLM in the strategy can assess relevance)
    planRevisionService.reviseOnFailure(c.agentId(), result, currentTick);
}
```

`ActionResult` needs a failure detection mechanism. Currently the
orchestrator assigns `result.text()` to `lastActionResult` without
inspecting success/failure. The action resolution returns text like
"You can't reach that" or "The door is locked." A simple approach:
`ActionResolver.resolve()` returns an `ActionOutcome` with a `success`
boolean alongside the text. Failed outcomes trigger plan revision.

**AdaptationCause for reactive revision:**

```java
public record ActionFailureCause(
    String actionType,
    String target,
    String failureReason
) implements AdaptationCause {}
```

### Deliberative: Reflection-Driven

During reflection, after step status assessment (D7), trigger plan
revision for goals whose plans have stale or failed steps.

Piggybacking on the existing reflection chain in `AgentExperienceService`:

```
reflectionSynthesizer.synthesize() → insights
  → goalEvaluator.evaluate() → goal formation/revision
    → NEW: planStepAssessment() → update step statuses
    → NEW: planRevisionService.reviseOnReflection() → revise plans with stale/failed steps
```

**AdaptationCause for deliberative revision:**

```java
public record ReflectionCause(
    List<String> insights,
    int ticksSinceLastRevision
) implements AdaptationCause {}
```

## Step Status Assessment (D7)

Step status is assessed during reflection by expanding the reflection
synthesis prompt. Same pattern as `GoalOutcomeCounts` from #43.

The synthesizer prompt gains:

```
For each plan step below, assess its current status based on recent
actions and outcomes:
- COMPLETED: the agent performed actions that achieved this step
- IN_PROGRESS: the agent is actively working on this step
- FAILED: the agent attempted this step and it failed
- PENDING: not yet attempted

Plan for goal "protect-penelope":
  1. find-poison: "Locate the poison bottle" [PENDING]
  2. dispose-poison: "Remove the poison from play" [PENDING]
```

The response includes a `planAssessments` object:

```json
{
  "insights": [...],
  "goalAssessments": {...},
  "planAssessments": {
    "protect-penelope": {
      "find-poison": "COMPLETED",
      "dispose-poison": "PENDING"
    }
  }
}
```

Step statuses are updated on `CharacterState` after parsing.

**Status lag:** Step status may lag by up to `maxUnreflected` ticks
(default: 5). This is acceptable — the character's per-tick behavior
is driven by the thinking field, which naturally tracks its own progress.
The structured status serves plan revision, not per-tick action selection.

## Modified Components

### CharacterState

Replace `String currentPlan` with structured plan storage:

```java
// Remove
private volatile String currentPlan;
public String currentPlan() { ... }
public void setCurrentPlan(String plan) { ... }

// Add
private final ConcurrentHashMap<String, AgentPlan> plans = new ConcurrentHashMap<>();
public Map<String, AgentPlan> plans() { return Collections.unmodifiableMap(plans); }
public void setPlan(String goalName, AgentPlan plan) { plans.put(goalName, plan); }
public void removePlan(String goalName) { plans.remove(goalName); }
```

### ObservationBuilder

Replace `currentPlanSection(CharacterState)` with `planSections(CharacterState)`.
Returns a list of `ObservationSection` — one per goal with an active plan.

```java
private static List<ObservationSection> planSections(CharacterState character) {
    if (character.plans().isEmpty()) return List.of();
    return character.plans().entrySet().stream()
        .sorted(Map.Entry.comparingByKey())
        .map(e -> {
            var goal = e.getKey();
            var plan = e.getValue();
            var items = plan.steps().stream()
                .map(s -> "[" + s.status().name() + "] " + s.description())
                .toList();
            return ObservationSection.items(
                "Plan: " + goal, plan.rationale(), items);
        })
        .toList();
}
```

**Token budget consideration:** With up to 10 goals and 2-5 steps each,
plans could add 20-50 lines to the observation. Plans are rendered after
goals and before recent activity. If token pressure becomes an issue,
limit rendering to PRIMARY goals' plans or the top-3 by priority.

### AgentResponse / RESPONSE_FORMAT_INSTRUCTION

The `thinking` field's prompt changes from:

```
"thinking": "your persistent strategic plan — shown to you next turn
  as 'Your Current Plan'. Write strategy, not stream-of-consciousness"
```

To:

```
"thinking": "your immediate reasoning about the current situation —
  what you notice, how it affects your plans, what to do next.
  Shown to you next turn as 'Your Current Thinking'"
```

The `thinking` field becomes a reactive scratchpad. Strategic planning
is handled by the structured plan model. The thinking field is still
stored on `CharacterState` (as a separate field, not `currentPlan`) and
shown back next turn as "Your Current Thinking" to maintain continuity
of the character's internal reasoning.

**New field on CharacterState:**

```java
private volatile String currentThinking;
public String currentThinking() { return currentThinking; }
public void setCurrentThinking(String thinking) { this.currentThinking = thinking; }
```

`ObservationBuilder.currentPlanSection()` becomes `currentThinkingSection()`
and renders "Your Current Thinking" instead of "Your Current Plan."

### ScenarioOrchestrator

In `runAutonomousTicks()`:

```java
// Replace
if (response.thinking() != null) {
    c.setCurrentPlan(response.thinking());
}

// With
if (response.thinking() != null) {
    c.setCurrentThinking(response.thinking());
}

// Add after action resolution
if (result.isFailure()) {
    planRevisionService.reviseOnFailure(c.agentId(), result, currentTick);
}
```

### ExchangeRunner

Same replacement — `setCurrentPlan()` → `setCurrentThinking()`.

### ManorGoalEvaluator

Expanded to chain plan formation after goal formation and plan removal
after goal abandonment/completion. Gains a `ManorPlanFormationStrategy`
dependency and access to `CharacterState` (via a lookup function or
direct reference).

After goal formation:
```java
for (AgentGoal newGoal : newlyFormedGoals) {
    AgentPlan plan = planFormationStrategy.formPlan(
        agentId, tenancyId, newGoal, finalGoals, memories, currentTick);
    if (plan != null && !plan.steps().isEmpty()) {
        characterStateLookup.apply(agentId).setPlan(newGoal.name(), plan);
    }
}
```

After goal abandonment/completion:
```java
characterStateLookup.apply(agentId).removePlan(abandonedGoal.name());
```

### AgentExperienceService

Expanded to chain plan step assessment and plan revision after reflection.
Gains `ManorPlanRevisionStrategy` and plan-related dependencies.

In `runReflection()`, after goal evaluation:
```java
// Assess plan step statuses from reflection insights
var planAssessments = reflectionResult.planAssessments();
if (planAssessments != null) {
    updatePlanStepStatuses(agentId, planAssessments);
}

// Trigger deliberative plan revision for goals with failed/stale steps
planRevisionService.reviseOnReflection(agentId, insights, currentTick);
```

## LLM Call Budget

Full reflection chain with plan lifecycle:

| Call | When | Cost |
|------|------|------|
| Reflection synthesis | Every reflection | Existing |
| Goal formation | When new goals proposed | Existing |
| Goal revision | Every reflection | Existing |
| **Plan formation** | Per new goal (0-2 per reflection) | **New** |
| **Plan revision** | Per goal with failed/stale steps | **New** |

**Worst case per reflection:** 3 existing + 2 plan formations + 3 plan
revisions = 8 LLM calls. Reflection fires every 5+ ticks. Each call is
a short prompt (plan context, not full scenario context). All run on the
same virtual thread, serialized. Total latency: 20-60 seconds for the
full chain, during which the tick loop continues.

**Batching option (future optimization):** Plan formation for multiple
new goals could be batched into a single LLM call. Similarly, plan
revision could assess all goals' plans in one call. Deferred — optimize
only if latency becomes a problem.

## Configuration

```properties
# Plan formation
manor.plan.enabled=true
manor.plan.max-steps-per-goal=5
manor.plan.min-steps-per-goal=2

# Plan revision
manor.plan.revision.enabled=true
manor.plan.revision.max-generation=5
```

`max-generation` caps the number of times a single goal's plan can be
revised. After 5 revisions, the plan is considered stable — further
revision triggers are ignored. This prevents infinite revision loops
where the LLM keeps proposing plans that fail.

## Testing Strategy

### Unit Tests

- **ManorPlanFormationStrategyTest** — mock AgentProvider, verify prompt
  includes goal + context, verify plan parsing (valid JSON, malformed,
  empty steps), verify step count constraints
- **ManorPlanRevisionStrategyTest** — mock AgentProvider, verify
  AdaptationContext construction (null case fields, correct step mapping),
  verify RevisedPlan parsing, verify step status reset
- **CharacterStateTest** — verify plans CRUD operations, verify
  currentPlan removal, verify currentThinking field
- **ObservationBuilderTest** — verify plan sections render per-goal with
  status prefixes, verify "Your Current Thinking" replaces "Your Current
  Plan", verify empty plans omitted
- **AgentResponseTest** — verify thinking field unchanged (still parsed
  from JSON)
- **ManorGoalEvaluatorTest** — verify plan formation chains after goal
  formation, verify plan removal on goal abandon/complete, verify plan
  revision on goal revision

### Integration / Regression

Full test suite (`mvn test -pl wacky-manor`) verifies no regressions
from currentPlan removal and thinking field relabeling.

### LLM Eval Tests

- **Plan formation from goal** — scenario where goal forms, verify plan
  appears in subsequent observations with named steps
- **Plan revision on failure** — character attempts a plan step that
  fails (locked door, item taken), verify plan revises
- **Plan removal on goal completion** — goal completes, verify plan
  removed from observations
- **Thinking field independence** — verify thinking field contains
  reactive reasoning (not strategic planning) when plans exist
