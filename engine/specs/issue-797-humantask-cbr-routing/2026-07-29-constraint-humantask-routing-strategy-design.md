# ConstraintHumanTaskRoutingStrategy Design

Refs: engine#755 (parent: engine#797)

## Context

engine#741 shipped the `HumanTaskRoutingStrategy` SPI. engine#754 added the
CBR-based implementation scoring candidates by historical plan trace data.

This spec covers a constraint-based strategy that uses declarative rules for
candidate filtering and scoring, with both context-driven conditions and
workload/fairness balancing.

## Two constraint types

Context constraints and workload constraints have genuinely different evaluation
semantics. Context constraints are global (evaluated once against the case state,
effect applies to named groups/users). Workload constraints are per-candidate
(evaluated for each candidate user against their operational state). Unifying them
into a single evaluation model adds artificial complexity.

### ContextConstraint

Global condition evaluated against the case context (working layer). When the
condition is true, the effect applies to named groups or users.

**Java DSL:**
```java
ContextConstraint.builder()
    .when(ctx -> ctx.layer(WORKING).getDouble("transaction.amount") > 10000)
    .preferUsers(Set.of("senior-reviewer-a", "senior-reviewer-b"))
    .weight(0.8)
    .build()
```

**YAML:**
```yaml
humanTaskConstraints:
  context:
    - when: ".transaction.amount > 10000"
      preferGroups: ["senior-reviewers"]
      weight: 0.8
    - when: ".risk.level == \"high\""
      excludeUsers: ["junior-analysts"]
```

**Record fields:**
- `condition` — `ExpressionEvaluator` (JQ, MVEL, or Lambda)
- `effect` — sealed: `Prefer(Set<String> groups, Set<String> users)` |
  `Exclude(Set<String> groups, Set<String> users)`
- `weight` — `double`, 0.0–1.0 (scoring magnitude for `Prefer`; ignored for `Exclude`)

**Effect semantics:**
- `Prefer` — additive score boost. Each matching user gets `+weight` added to their
  score. Multiple constraints stack.
- `Exclude` — hard filter. Matching users are removed from candidates before scoring.
  Irreversible within this evaluation cycle.

**Condition evaluation:** the `condition` is evaluated against
`context.layer(ContextLayer.WORKING).asJsonNode()` for JQ/MVEL, or
`context` directly for Lambda. Boolean result. Non-boolean or evaluation
failure → condition treated as false (logged, not thrown).

**Location:** `api/src/main/java/io/casehub/api/model/routing/ContextConstraint.java`

### WorkloadConstraint

Per-candidate thresholds applied using data from `WorkloadDataProvider`.

**Java DSL:**
```java
WorkloadConstraint.builder()
    .maxActiveTaskCount(5)
    .loadBalanceWeight(0.3)
    .build()
```

**YAML:**
```yaml
humanTaskConstraints:
  workload:
    maxActiveTaskCount: 5
    loadBalanceWeight: 0.3
```

**Record fields:**
- `maxActiveTaskCount` — `Integer`, nullable. Exclude users with active task count
  above this threshold. Hard filter.
- `loadBalanceWeight` — `Double`, nullable, 0.0–1.0. Scoring weight for load
  balancing. Lower active count → higher score, normalised across candidates.
  Score formula: `weight * (1.0 - (userCount / maxCountAmongCandidates))`.

**Location:** `api/src/main/java/io/casehub/api/model/routing/WorkloadConstraint.java`

## WorkloadDataProvider SPI

Provides operational workload snapshots for candidate users.

```java
public interface WorkloadDataProvider extends NamedStrategy {
    Map<String, WorkloadSnapshot> getWorkload(Set<String> userIds, String tenancyId);
}

public record WorkloadSnapshot(int activeTaskCount, Instant lastAssignedAt) {}
```

**Location:** `api/src/main/java/io/casehub/api/spi/routing/WorkloadDataProvider.java`
and `api/src/main/java/io/casehub/api/spi/routing/WorkloadSnapshot.java`

**Default:** `NoOpWorkloadDataProvider` (`@DefaultBean @ApplicationScoped @Unremovable`,
id `"default"`) in `runtime/internal/routing/`. Returns empty map. When active,
workload constraints degrade gracefully — `maxActiveTaskCount` excludes nobody,
`loadBalanceWeight` produces no scores.

**Real implementations** (follow-on, not #755 scope):
- `casehub-engine-actor-state` — backed by `WorkerExecutionManager.getActiveCaseIds()`
- `casehub-work-engine-adapter` — backed by WorkItem query

**EngineStrategyResolver:** already discovers `NamedStrategy` beans via the
`registerRemainingStrategies` catch-all. No resolver changes needed.

## ConstraintHumanTaskRoutingStrategy

**Location:** `runtime/src/main/java/io/casehub/engine/internal/routing/ConstraintHumanTaskRoutingStrategy.java`

**CDI:** `@ApplicationScoped @Unremovable`, id `"constraint"`.

**Dependencies:** `WorkloadDataProvider` (optional via `Instance<>`),
`JQEvaluator` for JQ expression evaluation.

### Algorithm

1. Copy `candidates.users()` to a mutable `eligibleUsers` set.
2. **Context constraints (global):** for each `ContextConstraint` on the
   case definition:
   a. Evaluate `condition` against the case context working layer.
   b. If false, skip.
   c. If `Exclude` effect: remove named users from `eligibleUsers`. Group-based
      exclusion deferred to engine#757 (no-op until group membership resolution).
   d. If `Prefer` effect: accumulate `+weight` for each named user present in
      `eligibleUsers`. Group-based preference deferred to engine#757.
3. If `eligibleUsers` is empty after exclusions → return
   `Escalated("all candidates excluded by context constraints")`.
4. **Workload constraints (per-candidate):** if `WorkloadConstraint` is configured
   AND `WorkloadDataProvider` returns non-empty data:
   a. Query `provider.getWorkload(eligibleUsers, tenancyId)`.
   b. If `maxActiveTaskCount` set: exclude users above threshold from `eligibleUsers`.
   c. If `eligibleUsers` empty after workload exclusion → return
      `Escalated("all candidates excluded by workload constraints")`.
   d. If `loadBalanceWeight` set: compute normalised load-balance scores for
      remaining candidates. Formula: `weight * (1.0 - (count / maxCount))`.
      Users without workload data get score 0.0 (no boost, not excluded).
5. **Combine scores:** context preference scores + workload balance scores
   (additive). If combined scores map is empty (no Prefer constraints matched,
   no workload scoring) → return `Unchanged`.
6. Return `Enriched(candidates.groups(), eligibleUsers, combinedScores)`.

**Key differences from CbrHumanTaskRoutingStrategy:**
- CAN filter candidates (Exclude, maxActiveTaskCount)
- CAN escalate (all candidates excluded)
- Modifies `candidateUsers` in the Enriched result (excluded users removed)
- Does not use `ExperienceAnalyser` — complementary to CBR, not overlapping

### Expression evaluation

The strategy needs to evaluate `ExpressionEvaluator` conditions against the case
context. `HumanTaskRoutingContext.caseContext()` is a `JsonNode` (working layer).

For JQ: inject `JQEvaluator` (engine-common, `@ApplicationScoped`). Call
`evaluator.evaluateBoolean(expression, caseContext)`.

For MVEL: not yet implemented in the engine's evaluator infrastructure. JQ and
Lambda cover the YAML and Java DSL paths respectively. MVEL support is additive
when needed.

For Lambda: `LambdaExpressionEvaluator.test(caseContext)` — but the context type
is `CaseContext`, not `JsonNode`. The strategy receives `JsonNode` in
`HumanTaskRoutingContext`. Lambda constraints need the full `CaseContext`.

**Resolution:** the strategy accepts an additional `CaseContext` parameter via
a new field on `HumanTaskRoutingContext`, or evaluates Lambda constraints by
reconstructing a `CaseContext` from the `JsonNode`. The simpler path: add
`CaseContext caseContextObj` to `HumanTaskRoutingContext` alongside the existing
`JsonNode caseContext`. The handler already has the `CaseContext` — threading it
costs nothing. JQ evaluates against the `JsonNode`; Lambda evaluates against the
`CaseContext`. No reconstruction needed.

**HumanTaskRoutingContext change:**
```java
public record HumanTaskRoutingContext(
    UUID caseId,
    String bindingName,
    String tenancyId,
    JsonNode caseContext,
    CaseContext caseContextObj,
    List<RetrievedExperience> experiences) {}
```

This is a breaking change to the record (new parameter). All construction sites
update. Pre-release platform — the cost is trivial.

## CaseDefinition changes

`CaseDefinition` gains:
- `List<ContextConstraint> humanTaskContextConstraints` (empty by default)
- `WorkloadConstraint humanTaskWorkloadConstraint` (nullable)

**Builder:**
```java
.humanTaskContextConstraint(ContextConstraint.builder()
    .when(".transaction.amount > 10000")
    .preferGroups(Set.of("senior-reviewers"))
    .weight(0.8)
    .build())
.humanTaskWorkloadConstraint(WorkloadConstraint.builder()
    .maxActiveTaskCount(5)
    .loadBalanceWeight(0.3)
    .build())
```

**YAML mapper:** `humanTaskConstraints:` block in `CaseDefinitionYamlMapper` with
`context:` array and `workload:` object sub-blocks.

## Activation

```yaml
humanTaskRouting: "constraint"
humanTaskConstraints:
  context:
    - when: ".transaction.amount > 10000"
      preferGroups: ["senior-reviewers"]
      weight: 0.8
  workload:
    maxActiveTaskCount: 5
    loadBalanceWeight: 0.3
```

`EngineStrategyResolver` resolves by id `"constraint"`.

## Group membership limitation

`Exclude(groups)` and `Prefer(groups)` need to know which candidate users belong
to which groups. The strategy receives `HumanTaskCandidates(groups, users)` — flat
sets, no membership mapping.

For #755: group-based effects are stored on the constraint but have no effect at
evaluation time. `excludeUsers`/`preferUsers` work immediately. Group evaluation
is deferred to engine#757 (group scoring via group membership resolution). The
constraint model is complete; the group evaluation is deferred.

## Tests

### Unit tests — ContextConstraint

`api/src/test/java/io/casehub/api/model/routing/ContextConstraintTest.java`

| Test | Assertion |
|------|-----------|
| `preferUsersEffect` | Builder creates Prefer with users |
| `preferGroupsEffect` | Builder creates Prefer with groups |
| `excludeUsersEffect` | Builder creates Exclude with users |
| `excludeGroupsEffect` | Builder creates Exclude with groups |
| `weightClamped` | Weight outside 0.0–1.0 clamped or rejected |
| `conditionRequired` | Null condition rejected |
| `effectRequired` | No effect set rejected |

### Unit tests — WorkloadConstraint

`api/src/test/java/io/casehub/api/model/routing/WorkloadConstraintTest.java`

| Test | Assertion |
|------|-----------|
| `maxActiveTaskCountOnly` | Builds with threshold, null weight |
| `loadBalanceWeightOnly` | Builds with weight, null threshold |
| `bothSet` | Both fields populated |
| `emptyConstraintRejected` | Neither field set → rejected |

### Unit tests — ConstraintHumanTaskRoutingStrategy

`runtime/src/test/java/io/casehub/engine/internal/routing/ConstraintHumanTaskRoutingStrategyTest.java`

| Test | Assertion |
|------|-----------|
| `idIsConstraint` | `id()` returns `"constraint"` |
| `noConstraintsReturnsUnchanged` | No constraints configured → Unchanged |
| `preferUsersBoostsScores` | Matching Prefer adds weight to named users |
| `excludeUsersRemovesCandidates` | Matching Exclude removes named users |
| `falseConditionSkipped` | Condition evaluates false → no effect |
| `multipleConstraintsStack` | Multiple Prefer weights are additive |
| `allExcludedEscalates` | All users excluded → Escalated |
| `workloadExcludesAboveThreshold` | Users above maxActiveTaskCount removed |
| `workloadLoadBalanceScoring` | Lower load → higher score |
| `workloadAllExcludedEscalates` | All users above threshold → Escalated |
| `noWorkloadProviderSkipsWorkload` | No provider → workload constraints have no effect |
| `combinedContextAndWorkload` | Both constraint types combine scores additively |
| `groupEffectsDeferredNoOp` | preferGroups/excludeGroups stored but no effect (pending #757) |
| `lambdaConditionEvaluated` | Lambda ExpressionEvaluator works |

### YAML mapper tests

`api/src/test/java/io/casehub/api/model/converter/CaseDefinitionYamlMapperTest.java` — extend:

| Test | Assertion |
|------|-----------|
| `humanTaskConstraints_contextParsed` | YAML context constraints parsed |
| `humanTaskConstraints_workloadParsed` | YAML workload constraint parsed |

## Downstream impact

- **HumanTaskRoutingContext** gains `caseContextObj` field — all construction sites
  in `CaseContextChangedEventHandler.publishHumanTaskSchedule()` update. Pre-release
  breaking change.
- **CaseDefinition** gains two fields — additive, no existing code breaks.
- **CaseDefinitionYamlMapper** gains `humanTaskConstraints:` parsing.
- **EngineStrategyResolver** — no changes (auto-discovers `NamedStrategy` beans).
- **WorkloadDataProvider** SPI + no-op default — additive.

## Future work

- engine#757: Group membership resolution — enables group-based Prefer/Exclude effects
- Follow-on: Real `WorkloadDataProvider` implementation (actor-state or work adapter)
- Follow-on: MVEL expression evaluator support in the constraint evaluation path
