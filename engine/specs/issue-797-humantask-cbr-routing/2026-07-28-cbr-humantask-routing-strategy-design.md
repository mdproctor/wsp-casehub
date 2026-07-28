# CbrHumanTaskRoutingStrategy Design

Refs: engine#754 (parent: engine#797)

## Context

engine#741 shipped the `HumanTaskRoutingStrategy` SPI: interface, sealed result type,
`ExperienceAnalyser` generalisation with predicate overload, retention changes for
humanTask traces (`bindingName` as the matching key, nullable `capabilityName`), and
handler integration in `CaseContextChangedEventHandler.publishHumanTaskSchedule()`.

The `NoOpHumanTaskRoutingStrategy` (`@DefaultBean`, id `"default"`) returns `Unchanged`.

This spec covers the first concrete implementation: a CBR-based strategy that scores
candidate users using plan trace history.

## Placement

`runtime/src/main/java/io/casehub/engine/internal/routing/CbrHumanTaskRoutingStrategy.java`

**Why runtime, not blocks:**

- `HumanTaskRoutingStrategy` is an engine SPI resolved by `EngineStrategyResolver` and
  called from `CaseContextChangedEventHandler`. It is engine planning infrastructure.
- The strategy only depends on engine-api types (`ExperienceAnalyser`,
  `HumanTaskRoutingStrategy`, `HumanTaskRoutingContext`, `HumanTaskCandidates`,
  `HumanTaskRoutingResult`, `RoutingOutcome`). No eidos, ledger, or trust deps.
- `CbrAgentRoutingStrategy` is in blocks because it composes trust (ledger) + graph
  (eidos) + CBR — dependency chains that pull it out of engine. The humanTask
  strategy has no such pull.
- blocks#60 draws the line: engine owns planning/routing infrastructure, blocks owns
  problem-solving techniques. A routing strategy is engine infrastructure.
- Runtime already hosts the CBR pipeline: `CbrRetrievalService` (produces experiences),
  `EngineStrategyResolver` (discovers strategies), `NoOpHumanTaskRoutingStrategy`.
- The shared CBR scoring logic is in `ExperienceAnalyser` (engine-api) — no duplication
  between agent and humanTask strategies.

## Design

### Class

```java
@ApplicationScoped
public class CbrHumanTaskRoutingStrategy implements HumanTaskRoutingStrategy {

  @Override
  public String id() { return "cbr"; }

  @Override
  public HumanTaskRoutingResult select(
      HumanTaskRoutingContext context, HumanTaskCandidates candidates) { ... }
}
```

No constructor injection. No CDI dependencies. Pure function from context + candidates
to result.

### Algorithm

1. If `context.experiences()` is empty, return `Unchanged`.
2. Extract eligible user IDs from `candidates.users()`.
3. If no eligible users, return `Unchanged`.
4. Call `ExperienceAnalyser.workerSuccessRates(experiences, eligibleUsers,
   step -> context.bindingName().equals(step.bindingName()),
   ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS)`.
5. If scores map is empty (no matching plan trace data), return `Unchanged`.
6. Return `Enriched(candidates.groups(), candidates.users(), scores)`.

### Matching key: bindingName

Agent routing matches plan trace steps by `capabilityName`. HumanTask routing matches
by `bindingName`. This is because humanTask traces have null `capabilityName` —
humanTasks are identified by their binding name, not a capability.

The `ExperienceAnalyser` predicate overload (added in #741) supports this:
`step -> bindingName.equals(step.bindingName())`.

### Result semantics

- `Unchanged`: no CBR data available, or no matching trace data. Candidates pass through
  to `HumanTaskScheduleEvent` with empty scores. This is the safe default — human tasks
  always have recipients.
- `Enriched`: groups and users pass through unchanged (the strategy enriches, not filters).
  `candidateScores` keys are from `candidateUsers` only — group scoring requires group
  membership resolution (engine#757).

Unlike `CbrAgentRoutingStrategy` which returns `Unresolvable` when it cannot select,
this strategy never blocks dispatch. Human tasks always proceed.

### Outcome weights

Uses `ExperienceAnalyser.DEFAULT_OUTCOME_WEIGHTS` directly:
- SUCCESS: 1.0
- GATE_EXPIRED: 0.5
- GATE_REJECTED: 0.25
- FAILURE: 0.0

Same values as blocks' `DefaultCbrOutcomeWeights`. No configurable weights SPI.
If a domain needs custom humanTask outcome weights, a `HumanTaskCbrOutcomeWeights`
interface can be added later — YAGNI now.

### Activation

Case definitions activate via YAML:

```yaml
humanTaskRouting: "cbr"
```

`EngineStrategyResolver` resolves by ID. When `humanTaskRouting` is null (not set),
the `@DefaultBean` `NoOpHumanTaskRoutingStrategy` (id `"default"`) is used.

### What this strategy does NOT do

- No trust-based filtering (humans, not AI agents)
- No graph query fallback (no agent graph for humans)
- No signal assembly (engine#754 scope — signals are an agent routing concept)
- No group scoring (engine#757 — requires group membership resolution)
- No candidate filtering (enrichment only — groups and users pass through)

## Tests

`runtime/src/test/java/io/casehub/engine/internal/routing/CbrHumanTaskRoutingStrategyTest.java`

Plain JUnit 5 + AssertJ. No `@QuarkusTest` needed — no CDI injection.

| Test | Assertion |
|------|-----------|
| `idIsCbr` | `id()` returns `"cbr"` |
| `emptyExperiencesReturnsUnchanged` | No CBR data → `Unchanged` |
| `emptyUsersReturnsUnchanged` | No candidate users → `Unchanged` |
| `scoresUsersByBindingName` | Scores computed using `bindingName` match |
| `selectsUserWithHighestSuccessRate` | Highest-scored user in `candidateScores` |
| `ignoresUsersNotInCandidateSet` | Scores only contain eligible user IDs |
| `ignoresStepsWithDifferentBindingName` | Steps for other bindings excluded |
| `addedStepsExcluded` | ADDED adaptation steps skipped |
| `substitutedStepsExcluded` | SUBSTITUTED adaptation steps skipped |
| `similarityWeightingApplied` | High-similarity experiences count more |
| `groupsPassThroughUnchanged` | `Enriched.candidateGroups()` equals input groups |
| `noMatchingTraceDataReturnsUnchanged` | Experiences exist but no steps match → `Unchanged` |

## Downstream impact

- No engine wiring changes. `EngineStrategyResolver` already discovers
  `HumanTaskRoutingStrategy` beans.
- No handler changes. `publishHumanTaskSchedule()` already threads strategy
  results to `HumanTaskScheduleEvent`.
- No CLAUDE.md changes needed for the strategy itself — the SPI section
  already documents `HumanTaskRoutingStrategy`.

## Future work

- engine#755: Constraint-based `HumanTaskRoutingStrategy` (separate strategy, id TBD)
- engine#757: Group scoring via group membership resolution
- engine#756: Work repo consumption of experiences and scores from `HumanTaskScheduleEvent`
