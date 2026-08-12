# Goal Lifecycle — Replace DynamicGoal with Engine-Backed AgentGoal

**Issue:** casehubio/examples#43
**Date:** 2026-08-12
**Depends on:** #42 (memory stack — reflection must produce insights before goal formation can consume them)
**Engine prerequisites:** casehubio/engine#897 (remove CaseDefinition from contexts), casehubio/engine#903 (GoalRevisionAction enum)
**Implementation sequencing:** Goal formation can be implemented immediately. Goal revision is blocked on engine#903 — implement as a second commit after #903 ships.

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

**CDI wiring path:** `ScenarioOrchestrator` (`@ApplicationScoped`) injects
`ManorGoalFormationStrategy` and `ManorGoalRevisionStrategy` via `@Inject`.
It constructs `ManorGoalEvaluator` with these strategies + `AgentRegistry`
+ config, then passes the evaluator to `AgentExperienceService`.

## Removals

| Component | What | Why |
|-----------|------|-----|
| `DynamicGoal.java` | Delete record | Replaced by `AgentGoal` on `AgentRegistry` |
| `CharacterState.dynamicGoals` | Delete field + methods | Goals no longer stored on character state |
| `AgentResponse.newGoals()` | Delete field | Goals come from reflection, not LLM response |
| `AgentResponse.dropGoals()` | Delete field | Goal lifecycle handled by revision strategy |
| `AgentResponse.GoalEntry` | Delete inner record | No longer needed |
| `ScenarioOrchestrator` goal processing block | Delete newGoals/dropGoals processing in `runAutonomousTicks()` | No longer in response |
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
`AgentProvider` for LLM calls. Blocked on engine#903 — implement after
the `GoalRevisionAction` enum ships.

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
Not a CDI bean — constructed by `ScenarioOrchestrator` with injected
strategies and passed to `AgentExperienceService`.

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
    private final ConcurrentHashMap<String, ReentrantLock> agentLocks;

    public void evaluate(String agentId, List<String> insights,
                         Map<String, GoalOutcomeCounts> goalOutcomes) {
        // 0. Per-agent lock — prevents concurrent descriptor updates
        //    from parallel reflection chains
        // 1. Cooldown check — skip if last formation was within N minutes
        // 2. Read current descriptor from AgentRegistry
        // 3. Retrieve recent memories from CaseMemoryStore
        //    Convert Memory → RetrievedMemory (memoryId, text, domain,
        //    createdAt, attributes)
        // 4. FORMATION: build GoalFormationContext, call strategy.propose()
        //    - Validate: no duplicate names, within capacity, name/desc length
        //    - Default priority: SECONDARY
        //    - Default visibility: PRIVATE (formed goals are internal)
        //    - Default capabilities: empty list
        // 5. REVISION (when available — blocked on engine#903):
        //    build GoalRevisionContext, call strategy.revise()
        //    - REVISE: update description
        //    - ABANDON: remove goal, ingest "Abandoned goal: [name]" to memory
        //    - COMPLETE: remove goal, ingest "Completed goal: [name]" to memory
        // 6. Build updated descriptor: descriptor.toBuilder().goals(finalGoals).build()
        // 7. Register: agentRegistry.register(updated)
    }
}
```

**Concurrency protection:** Per-agent `ReentrantLock` in the evaluator
prevents the read-modify-write race on `AgentDescriptor`. The existing
reflection trigger can spawn parallel chains for the same agent (the
`shouldReflect()` check and `Thread.start()` are not atomic). Previously
this was low-impact (duplicate insight writes are idempotent). With the
expanded chain writing to the descriptor, last-write-wins would clobber
goals. The lock ensures only one chain modifies the descriptor at a time.
The cooldown provides the first line of defense (skip re-formation within
N minutes); the lock handles the edge case where two chains slip through.

**AgentGoal construction defaults:** `ProposedGoal` has `name`,
`description`, `suggestedPriority`, `formationReason` — but `AgentGoal`
also requires `visibility` and `capabilities`. Defaults:
- `visibility: Visibility.PRIVATE` — formed goals are the agent's internal
  objectives, not visible to other agents or the system prompt renderer's
  public goal summary
- `capabilities: List.of()` — formed goals don't require specific
  capabilities
- `priority: suggestedPriority != null ? suggestedPriority : GoalPriority.SECONDARY`

**Memory type conversion:** `CaseMemoryStore.query()` returns
`List<Memory>` (neocortex type). `GoalFormationContext.recentMemories`
requires `List<RetrievedMemory>` (engine-api type). The evaluator converts:
```java
var retrieved = memories.stream()
    .map(m -> new RetrievedMemory(m.memoryId(), m.text(),
         m.domain().name(), m.createdAt(), m.attributes()))
    .toList();
```

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

The existing `synthesize()` method remains for use when goal evaluation
is disabled. When enabled, `runReflection()` calls
`synthesizeWithGoalAssessment()` instead.

### AgentExperienceService

Chains `ManorGoalEvaluator` after reflection. The synthesizer field type
changes from `ReflectionSynthesizer` (interface) to
`ManorReflectionSynthesizer` (concrete) to access the new
`synthesizeWithGoalAssessment()` method. This is acceptable: the manor
constructs everything manually — no SPI discovery is involved.

The `runReflection()` method expands:

```java
private void runReflection(String agentId, Instant since) {
    var sources = store.query(...);
    if (sources.isEmpty()) return;

    if (goalEvaluator != null) {
        // New path: combined insights + goal assessment
        var currentGoals = /* read from AgentRegistry */;
        var result = synthesizer.synthesizeWithGoalAssessment(
                agentId, tenantId, sources, currentGoals, 1);

        // Store reflection events
        for (var event : result.insights()) {
            store.store(ReflectionEvents.toMemoryInput(event));
        }

        // Chain goal evaluation
        var insightTexts = result.insights().stream()
                .map(ReflectionEvent::insight).toList();
        goalEvaluator.evaluate(agentId, insightTexts, result.goalOutcomes());
    } else {
        // Original path: insights only
        var events = synthesizer.synthesize(agentId, tenantId, sources, 1);
        for (var event : events) {
            store.store(ReflectionEvents.toMemoryInput(event));
        }
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

- `runAutonomousTicks()`: remove the newGoals/dropGoals processing block.
  Goal lifecycle is now handled by the evaluator post-reflection.
- `runScenario()`: inject `ManorGoalFormationStrategy` and
  `ManorGoalRevisionStrategy` via CDI `@Inject`. Construct
  `ManorGoalEvaluator` with strategies + `agentRegistry` + config.
  Pass evaluator to `AgentExperienceService`.

**Scripted mode:** `runScripted()` does not get goal evaluation. Scripted
mode is legacy/test-only — characters keep their initial Eidos goals.
This is intentional: scripted mode uses `CharacterAgentLoop` with trigger-
driven scenes, not autonomous reflection.

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
  - Per-agent lock prevents concurrent descriptor clobbering
  - Capacity cap respected
  - Duplicate goal names rejected
  - Memory → RetrievedMemory conversion correct
  - Default visibility PRIVATE, capabilities empty
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
