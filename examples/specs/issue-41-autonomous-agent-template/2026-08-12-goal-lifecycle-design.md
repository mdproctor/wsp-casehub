# Goal Lifecycle — Replace DynamicGoal with Engine-Backed AgentGoal

**Issue:** casehubio/examples#43
**Date:** 2026-08-12
**Depends on:** #42 (memory stack — reflection must produce insights before goal formation can consume them)
**Engine prerequisites:** casehubio/engine#897 (remove CaseDefinition from contexts), casehubio/engine#903 (GoalRevisionAction enum)

## Design Principle

Goals are the strategic layer. Plans are the tactical layer. The `currentPlan`
(persisted thinking) handles per-tick reactive intent — "I see poison, I'll
take it to the ballroom." Goals emerge from reflection — "Based on my
experiences, I should focus on protecting Penelope." This maps to System 1
(plan, reactive, per-tick) vs System 2 (goals, deliberative, post-reflection).

## Architecture

The goal lifecycle replaces the manor's hand-rolled `DynamicGoal` management
with the engine's cognitive stack. Three new components wire into the existing
reflection pipeline:

```
Tick loop → memory ingest → reflection trigger fires →
  ManorReflectionSynthesizer (insights + goal outcome assessments) →
  ManorGoalEvaluator:
    1. GoalFormationStrategy.propose() → new goals
    2. GoalRevisionStrategy.revise() → revisions / abandonments / completions
    3. AgentRegistry.register(updatedDescriptor)
  → goals visible on next tick via ObservationBuilder
```

The full chain runs on a single virtual thread, post-reflection. Three LLM
calls total (reflection synthesis + goal formation + goal revision). Latency
is 6-30 seconds depending on model speed. During this time the tick loop
continues — characters act on their current plan. Goals update asynchronously
and appear in the next tick's observation after the chain completes.

## Removals

| Component | What | Why |
|-----------|------|-----|
| `DynamicGoal.java` | Delete record | Replaced by `AgentGoal` on `AgentRegistry` |
| `CharacterState.dynamicGoals` | Delete field + methods | Goals no longer stored on character state |
| `AgentResponse.newGoals()` | Delete field | Goals come from reflection, not LLM response |
| `AgentResponse.dropGoals()` | Delete field | Goal lifecycle handled by revision strategy |
| `AgentResponse.GoalEntry` | Delete inner record | No longer needed |
| `ScenarioOrchestrator` lines 324-334 | Delete goal processing block | newGoals/dropGoals no longer in response |
| `RESPONSE_FORMAT_INSTRUCTION` | Remove newGoals/dropGoals | Not in response format |
| `CharacterAgentLoop.RESPONSE_FORMAT_INSTRUCTION` | Remove newGoals/dropGoals | Same |

## New Components

### ManorGoalFormationStrategy

Implements `GoalFormationStrategy` from `casehub-engine-api`. Uses
`AgentProvider` for LLM calls (not the engine's `ChatModelProvider`).

```java
@ApplicationScoped
public class ManorGoalFormationStrategy implements GoalFormationStrategy {

    @Override
    public String id() { return "manor-llm"; }

    @Override
    public GoalFormationProposal propose(GoalFormationContext context) {
        // Build prompt from context:
        //   - agentId, remainingCapacity
        //   - existingGoals (name, description, priority)
        //   - reflectionInsights
        //   - recentMemories
        // Ask LLM: "Propose new goals that represent genuinely new
        //   objectives — not refinements of existing goals."
        // Parse response into GoalFormationProposal
    }
}
```

**Input:** `GoalFormationContext(agentId, tenancyId, reflectionInsights,
existingGoals, recentMemories, remainingCapacity)`

**Output:** `GoalFormationProposal(goals: List<ProposedGoal>, rationale)`
where each `ProposedGoal` has `name`, `description`, `suggestedPriority`,
`formationReason`.

### ManorGoalRevisionStrategy

Implements `GoalRevisionStrategy` from `casehub-engine-api`. Uses
`AgentProvider` for LLM calls.

```java
@ApplicationScoped
public class ManorGoalRevisionStrategy implements GoalRevisionStrategy {

    @Override
    public String id() { return "manor-llm"; }

    @Override
    public GoalRevisionProposal revise(GoalRevisionContext context) {
        // Build prompt from context:
        //   - agentId
        //   - goals with outcome counts (success rate per goal)
        // Ask LLM: "For each goal, recommend an action:
        //   REVISE (refine description), ABANDON (drop — unachievable
        //   or irrelevant), or COMPLETE (goal achieved). Only act
        //   on goals with clear signals."
        // Parse response into GoalRevisionProposal with GoalRevisionAction
    }
}
```

**Input:** `GoalRevisionContext(agentId, tenancyId, goals, counts)`
where `counts` is `Map<String, GoalOutcomeCounts>` produced by the
expanded reflection synthesizer.

**Output:** `GoalRevisionProposal(revisions: List<RevisedGoal>, rationale)`
where each `RevisedGoal` has `goalName`, `action` (REVISE/ABANDON/COMPLETE),
`revisedDescription`, `revisionReason`.

### ManorGoalEvaluator

Orchestrator — called after reflection completes, on the same virtual thread.
Not a CDI bean — constructed by `AgentExperienceService` with its dependencies.

```java
public class ManorGoalEvaluator {

    private static final int MAX_GOALS = 10;
    private final GoalFormationStrategy formationStrategy;
    private final GoalRevisionStrategy revisionStrategy;
    private final AgentRegistry agentRegistry;
    private final CaseMemoryStore memoryStore;
    private final String tenancyId;
    private final long cooldownMinutes;
    private final ConcurrentHashMap<String, Instant> lastFormationTime;

    public void evaluate(String agentId, List<String> insights,
                         Map<String, GoalOutcomeCounts> goalOutcomes) {
        // 1. Cooldown check — skip if last formation was within N minutes
        // 2. Read current descriptor from AgentRegistry
        // 3. Retrieve recent memories from CaseMemoryStore
        // 4. FORMATION: build GoalFormationContext, call strategy.propose()
        //    - Validate: no duplicate names, within capacity, name/desc length
        //    - Default priority: SECONDARY
        // 5. REVISION: build GoalRevisionContext, call strategy.revise()
        //    - REVISE: update description
        //    - ABANDON: remove goal, ingest "Abandoned goal: [name]" to memory
        //    - COMPLETE: remove goal, ingest "Completed goal: [name]" to memory
        // 6. Build updated descriptor: descriptor.toBuilder().goals(finalGoals).build()
        // 7. Register: agentRegistry.register(updated)
    }
}
```

**Cooldown:** Configurable via `manor.goal.cooldown-minutes` (default 5).
Prevents concurrent-write clobbering when reflection triggers fire in
quick succession.

**Memory ingestion on lifecycle transitions:** When a goal is abandoned or
completed, the evaluator ingests a memory event recording the transition.
This ensures the agent's memory reflects strategic changes — "I gave up on
protecting the tea because the threat passed" or "Successfully poisoned the
tea." These memories influence future reflections and goal formation.

## Modified Components

### ManorReflectionSynthesizer

Currently returns `List<ReflectionEvent>` (insights only). Expanded with
a new method that also produces goal outcome assessments:

```java
public record ReflectionWithGoalAssessment(
    List<ReflectionEvent> insights,
    Map<String, GoalOutcomeCounts> goalOutcomes
) {}

public ReflectionWithGoalAssessment synthesizeWithGoalAssessment(
        String agentId, String tenantId,
        List<Memory> sources, List<AgentGoal> currentGoals, int targetLevel) {
    // Single LLM call with expanded prompt:
    //   - "Analyze recent experiences for patterns and insights" (existing)
    //   - "For each of these goals, assess how many recent experiences
    //      represent success vs failure toward that goal" (new)
    // Returns structured JSON:
    //   { "insights": [...], "goalAssessments": {"goal-name": {"success": N, "failure": M}} }
}
```

The existing `synthesize()` method remains unchanged for backward
compatibility. The evaluator calls `synthesizeWithGoalAssessment()`.

### AgentExperienceService

Chains `ManorGoalEvaluator` after reflection. The `runReflection()` method
expands:

```java
private void runReflection(String agentId, Instant since) {
    var sources = store.query(...);
    if (sources.isEmpty()) return;

    // Existing: synthesize insights
    // New: use synthesizeWithGoalAssessment() instead of synthesize()
    var currentGoals = agentRegistry.findById(agentId, tenantId)
            .map(AgentDescriptor::goals).orElse(List.of());
    var result = synthesizer.synthesizeWithGoalAssessment(
            agentId, tenantId, sources, currentGoals, 1);

    // Store reflection events (existing)
    for (var event : result.insights()) {
        store.store(ReflectionEvents.toMemoryInput(event));
    }

    // New: chain goal evaluation
    if (goalEvaluator != null) {
        var insightTexts = result.insights().stream()
                .map(ReflectionEvent::insight).toList();
        goalEvaluator.evaluate(agentId, insightTexts, result.goalOutcomes());
    }

    // Existing: memory decay
    if (decayEnabled) { ... }
}
```

`AgentExperienceService` constructor gains `ManorGoalEvaluator` parameter
(nullable — goal evaluation is optional, like reflection).

### ObservationBuilder.goalsSection()

Simplifies to accept only `List<AgentGoal>`. No more `DynamicGoal` rendering:

```java
private static ObservationSection goalsSection(List<AgentGoal> goals) {
    var items = goals.stream()
        .sorted(Comparator.comparing(AgentGoal::priority)
                           .thenComparing(AgentGoal::name))
        .map(g -> "[" + g.priority().name() + "] " + g.description())
        .toList();
    if (items.isEmpty()) {
        return ObservationSection.items("Your Goals", "No specific goals.", List.of());
    }
    return ObservationSection.items("Your Goals", null, items);
}
```

The `character` parameter is removed from `goalsSection()` — no longer
needs access to `CharacterState.dynamicGoals()`. Call sites in
`buildObservation()` update accordingly.

### ScenarioOrchestrator

- `runAutonomousTicks()`: remove the newGoals/dropGoals processing block
  (currently lines 324-334). Goal lifecycle is now handled by the evaluator
  post-reflection.
- `runScenario()`: pass `AgentRegistry` and goal configuration to the
  `ManorGoalEvaluator` constructor. Pass the evaluator to
  `AgentExperienceService`.

### pom.xml

Add `casehub-engine-api` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-engine-api</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

## Configuration

```properties
# Goal formation
manor.goal.enabled=true
manor.goal.cooldown-minutes=5
manor.goal.max-new-per-reflection=2
manor.goal.max-goals=10

# Goal revision
manor.goal.revision.enabled=true
```

## Testing Strategy

### Unit Tests

- **ManorGoalFormationStrategyTest** — mock AgentProvider, verify prompt
  construction includes insights/goals/memories, verify proposal parsing
  (valid JSON, malformed JSON, empty goals)
- **ManorGoalRevisionStrategyTest** — mock AgentProvider, verify prompt
  includes goals with outcome counts, verify parsing of REVISE/ABANDON/COMPLETE
  actions
- **ManorGoalEvaluatorTest** — mock strategies and registry:
  - Formation proposes goals → registered on descriptor
  - Revision with REVISE → description updated
  - Revision with ABANDON → goal removed, memory ingested
  - Revision with COMPLETE → goal removed, memory ingested
  - Cooldown prevents double-formation
  - Capacity cap respected
  - Duplicate goal names rejected
- **ManorReflectionSynthesizerTest** — verify `synthesizeWithGoalAssessment()`
  parses both insights and goal outcome counts from LLM response
- **ObservationBuilderTest** — verify goals section renders only AgentGoal
  with priority prefix, no DynamicGoal references
- **AgentResponseTest** — verify newGoals/dropGoals fields removed, existing
  parse tests updated
- **CharacterStateTest** — verify dynamicGoals methods removed

### Regression

Full test suite (`mvn test -pl wacky-manor`) verifies no regressions from
DynamicGoal removal and AgentResponse field changes.

### LLM Eval Tests

- **Goal formation from reflection** — run a scenario where a character
  accumulates experiences, triggers reflection, and verify that new goals
  appear in subsequent observations
- **Goal abandonment** — character has a goal that becomes impossible,
  verify it gets abandoned after reflection assesses failure
- **Goal completion** — character achieves a goal, verify it gets marked
  complete

## Platform Extraction

The goal lifecycle pattern (reflection → formation → revision → registry)
is domain-independent. The manor-specific parts are the LLM prompts and
the `AgentProvider` usage. The pattern extracts as:

- `GoalFormationStrategy` SPI — already in engine-api
- `GoalRevisionStrategy` SPI — already in engine-api
- `GoalRevisionAction` enum — engine#903
- Evaluator orchestration — generalizable as an engine service

The wacky-manor implementation demonstrates the pattern; the engine
provides the reusable contracts.
