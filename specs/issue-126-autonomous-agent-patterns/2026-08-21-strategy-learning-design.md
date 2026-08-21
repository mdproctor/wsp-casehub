# StrategyLearning Design — Multi-Level Reflection on Interaction Strategies

**Issue:** casehubio/blocks#124
**Package:** `io.casehub.blocks.agentic.social`
**Decisions:** D32–D41

## Overview

StrategyLearning is the fifth social cognition orchestrator. It evaluates engagement outcomes, detects what interaction strategies work (per-subject and globally), and produces actionable strategy guidelines for agent prompts.

**Cognitive boundary:** UserModel = perceive ("what IS this person like"). StrategyLearning = learn-and-adapt ("what WORKS with this person and in general"). MentalModel = reason about minds ("what does this person THINK"). PersonalityEvolution = self-model ("how am I CHANGING"). InnerLife = self-expression ("should I SPEAK now").

## Composition

| Component | Source | Role |
|-----------|--------|------|
| `ReflectionOrchestrator` | neocortex-memory-api | Generates reflective insights from accumulated memories (tier 3) |
| `ContentSummariser<EngagementSignal>` | blocks/summarisation | Textual summary of per-conversation engagement (tier 2) |
| `CbrCaseMemoryStore` | neocortex-memory-api | Stores engagement evidence cases with features + dimensional snapshots |
| `TrendAnalyzer` | neocortex-memory-api | Detects engagement trends (slope, delta, volatility) over case history |
| `AgentProvider` | platform-agent-api | LLM synthesis for strategy profile updates (tier 3) |
| `EngagementEvent` | neocortex-memory-api | Engagement signal data (responded, responseTimeMs, sentimentShift, etc.) |

## Types

### EngagementSignal (sealed interface)

Input signal type for `record()`. Two variants:

```java
public sealed interface EngagementSignal {
    String subjectId();

    record TurnOutcome(
        EngagementEvent event,
        Map<String, Double> dimensionalSnapshot,
        String responseExcerpt,
        String subjectId
    ) implements EngagementSignal {
        // dimensionalSnapshot: agent's current strategy dimensional scores
        //   at time of response (from StrategyProfile.dimensions()).
        //   Provides correlation data: "when verbosity was 0.7, engagement was X."
        // responseExcerpt: brief excerpt of agent's response (for LLM reflection context)
    }

    record ConversationOutcome(
        String conversationId,
        String subjectId,
        String conversationSummary,
        int turnCount
    ) implements EngagementSignal {
        // Conversation boundary marker — triggers tier 2 case storage
    }
}
```

### StrategyProfile (record)

Output of the orchestrator. Per-agent strategy.

```java
public record StrategyProfile(
    String agentId,
    String tenantId,
    Map<String, Double> dimensions,
    List<String> guidelines,
    Instant lastReflection,
    int evidenceCount,
    Map<String, String> metadata
) {
    // dimensions: verbosity, formality, initiative, directness, questionRate
    //   each [0,1], default 0.5
    // guidelines: ranked LLM-generated textual strategy advice
    //   e.g. "Keep responses under 3 sentences", "Ask follow-up questions"
    //   May include per-subject insights: "With User-X: use humor"
}
```

**Default dimensions:**

| Dimension | Range | Default | Meaning |
|-----------|-------|---------|---------|
| `verbosity` | [0,1] | 0.5 | concise ↔ elaborate |
| `formality` | [0,1] | 0.5 | casual ↔ formal |
| `initiative` | [0,1] | 0.5 | reactive ↔ proactive |
| `directness` | [0,1] | 0.5 | indirect ↔ direct |
| `questionRate` | [0,1] | 0.5 | statements ↔ questions |

### StrategyLearningConfig (record)

```java
public record StrategyLearningConfig(
    int minSignalsForConversationCase,
    int minCasesForReflection,
    int maxReflectionSources,
    int maxGuidelines,
    double defaultDimensionValue,
    int maxBufferSize,
    Duration staleStateTimeout
) {
    public StrategyLearningConfig {
        if (minSignalsForConversationCase < 1)
            throw new IllegalArgumentException("minSignalsForConversationCase must be >= 1");
        if (minCasesForReflection < 1)
            throw new IllegalArgumentException("minCasesForReflection must be >= 1");
        if (defaultDimensionValue < 0.0 || defaultDimensionValue > 1.0)
            throw new IllegalArgumentException("defaultDimensionValue must be in [0,1]");
    }

    public static StrategyLearningConfig defaults() {
        return new StrategyLearningConfig(3, 5, 50, 10, 0.5, 100,
            Duration.ofHours(24));
    }
}
```

| Field | Default | Purpose |
|-------|---------|---------|
| `minSignalsForConversationCase` | 3 | Minimum TurnOutcome signals before storing a conversation case |
| `minCasesForReflection` | 5 | Minimum engagement cases before reflect() produces insights |
| `maxReflectionSources` | 50 | Max CBR cases to feed into reflect() LLM synthesis |
| `maxGuidelines` | 10 | Max guidelines in StrategyProfile |
| `defaultDimensionValue` | 0.5 | Starting value for each dimension |
| `maxBufferSize` | 100 | Max pending signals before dropping oldest |
| `staleStateTimeout` | 24h | Evict agent states with no activity |

### StrategyLearningTick (sealed interface — from tick())

Hierarchical: `Learned ⊃ Observed ⊃ NoChange`.

```java
public sealed interface StrategyLearningTick {
    record NoChange(String reason) implements StrategyLearningTick {}

    record Observed(
        int signalsProcessed,
        double engagementRate,
        double meanSentiment
    ) implements StrategyLearningTick {}

    record Learned(
        int signalsProcessed,
        double engagementRate,
        double meanSentiment,
        List<String> conversationsStored,
        int casesStored
    ) implements StrategyLearningTick {}
}
```

### StrategyReflection (sealed interface — from reflect())

```java
public sealed interface StrategyReflection {
    record NoChange(String reason) implements StrategyReflection {}

    record Reflected(
        StrategyProfile profile,
        List<String> newGuidelines,
        TrendProfile trends,
        int evidenceCases
    ) implements StrategyReflection {}
}
```

### StrategyStore (SPI)

```java
public interface StrategyStore {
    void store(StrategyProfile profile);
    Optional<StrategyProfile> lookup(String agentId, String tenantId);
    List<String> subjectInsights(String agentId, String subjectId, String tenantId);
    void eraseAgent(String agentId, String tenantId);
}
```

`CbrStrategyStore` (`@DefaultBean`) backs onto `CbrCaseMemoryStore`:
- Profile stored as CbrCase: guidelines as problem text, dimensions as features, placeholder solution (`"-"`)
- `subjectInsights()` calls `retrieveSimilar()` with subjectId feature filter, extracts strategy-relevant features from matching cases, formats as textual insights

## Orchestrator

### StrategyLearningOrchestrator

Three entry points: `record()`, `tick()`, `reflect()`.

**Constructor dependencies:**
- `StrategyStore strategyStore`
- `CbrCaseMemoryStore cbrStore` — for engagement evidence cases
- `ReflectionOrchestrator reflectionOrchestrator` — tier 3 reflection
- `AgentProvider agentProvider` — tier 3 LLM synthesis
- `@Nullable ContentSummariser<EngagementSignal> summariser` — optional tier 2 text summary
- `StrategyLearningConfig config`

**Internal state:** `ConcurrentHashMap<String, AgentLearningState>` keyed by `agentId:tenantId`.

```
AgentLearningState:
  - pendingSignals: ArrayDeque<TurnOutcome>  (bounded by maxBufferSize)
  - pendingConversations: ArrayDeque<ConversationOutcome>
  - totalSignals: int
  - totalResponded: int
  - sentimentSum: double
  - lastSignalTimestamp: Instant
  - lastTickTimestamp: Instant
  - lastReflectTimestamp: Instant
  - lastActivityTimestamp: Instant
  - currentProfile: StrategyProfile (loaded from store on first access)
```

### record(EngagementSignal signal, String agentId, String tenantId)

O(1). Appends to pending queue. If `TurnOutcome`: increments counters. If `ConversationOutcome`: appends to conversation queue.

### tick(String agentId, String tenantId) → StrategyLearningTick

Cheap. Runs tiers 1-2 under per-agent `ReentrantLock`.

**Tier 1 (always, if pending signals exist):**
1. Drain pending TurnOutcome signals
2. Update aggregate counters: totalSignals, totalResponded, sentimentSum
3. Compute engagement metrics: `engagementRate = totalResponded / totalSignals`, `meanSentiment = sentimentSum / totalSignals`
4. Return `Observed` if no conversations pending

**Tier 2 (if pending conversations exist or signal threshold reached):**
1. Drain pending ConversationOutcome signals
2. For each conversation, collect the TurnOutcome signals with matching subjectId
3. Extract structured features from TurnOutcome signals:
   - `avgResponseLength`: mean of `event.responseLength()`
   - `continuationRate`: fraction where `event.continued() == true`
   - `meanSentimentShift`: mean of `event.sentimentShift()`
   - `avgDimensionalSnapshot_<dim>`: mean of snapshot values per dimension
   - `turnCount`: number of signals
4. Optionally run `ContentSummariser<EngagementSignal>` for text summary (stored as StringVal feature)
5. Store CbrCase via `cbrStore.store()` with:
   - problem: conversation summary text
   - solution: `"-"` (placeholder per GE-20260820-d4e011)
   - features: structured engagement features + subjectId + agentId
   - producerAgentId: agentId
6. Return `Learned` with tier 1 + tier 2 data

### reflect(String agentId, String tenantId) → StrategyReflection

Expensive. Runs tier 3 under per-agent `ReentrantLock`.

**Tier 3:**
1. Load current StrategyProfile from store (or use defaults if none)
2. Query engagement CBR cases: `retrieveSimilar()` with empty features, `minSimilarity(0.0)`, filtered by producerAgentId post-retrieval
3. If cases < `minCasesForReflection` → return `NoChange("insufficient evidence")`
4. Run `TrendAnalyzer.analyze()` on case feature history — produces TrendProfile with engagement trend metrics (slope on sentiment, volatility on response length, etc.)
5. Run `ReflectionOrchestrator.reflect(agentId, tenantId, since, maxReflectionSources)` — produces reflective insights from accumulated memories
6. Build LLM synthesis prompt with:
   - Current dimensional scores
   - Trend analysis results (what's improving/declining)
   - Reflection insights
   - Per-subject engagement patterns (group cases by subjectId, compare engagement rates)
   - Current guidelines (for revision)
7. Invoke `agentProvider` with synthesis prompt → produces revised guidelines + dimensional adjustments
8. Parse LLM output: extract new guidelines (List<String>) and dimensional deltas (Map<String, Double>)
9. Apply dimensional deltas to current scores (clamped to [0,1])
10. Build updated StrategyProfile
11. Store via `strategyStore.store()`
12. Return `Reflected` with new profile, guidelines, trends

### CBR Feature Schema

Engagement evidence cases use this feature layout:

| Feature | Type | Range | Purpose |
|---------|------|-------|---------|
| `subjectId` | StringVal | — | Per-subject filtering |
| `agentId` | StringVal | — | Per-agent filtering |
| `turnCount` | NumberVal | [1, 1000] | Conversation length |
| `avgResponseLength` | NumberVal | [0, 10000] | Mean response length in chars |
| `continuationRate` | NumberVal | [0, 1] | Fraction of turns where subject continued |
| `meanSentimentShift` | NumberVal | [-1, 1] | Mean sentiment change |
| `avgSnapshot_verbosity` | NumberVal | [0, 1] | Mean agent verbosity score during conversation |
| `avgSnapshot_formality` | NumberVal | [0, 1] | Mean agent formality score |
| `avgSnapshot_initiative` | NumberVal | [0, 1] | Mean agent initiative score |
| `avgSnapshot_directness` | NumberVal | [0, 1] | Mean agent directness score |
| `avgSnapshot_questionRate` | NumberVal | [0, 1] | Mean agent question rate |
| `conversationSummary` | StringVal | — | Optional text summary from ContentSummariser |

For TrendAnalyzer integration, the numeric features are declared as `FeatureField.TimeSeries` with `TrendSpec` containing `SLOPE`, `DELTA`, and `VOLATILITY` trend types. This enables detection of engagement trends like "sentiment is declining" (negative slope) or "response length is volatile" (high volatility).

### LLM Synthesis Prompt (tier 3)

```
You are a metacognitive strategy advisor for an AI agent. Analyze the agent's
interaction history and recommend strategy adjustments.

Current strategy dimensions:
{dimensions as key=value list}

Current guidelines:
{numbered guideline list, or "None yet" if first reflection}

Engagement trend analysis:
{TrendProfile metrics — slope, delta, volatility per feature}

Reflective insights:
{ReflectionOrchestrator output — list of reflective observations}

Per-subject engagement patterns:
{grouped by subjectId: engagement rate, mean sentiment, notable patterns}

Based on this evidence:
1. List up to {maxGuidelines} ranked strategy guidelines (most impactful first).
   Include per-subject insights where patterns differ significantly from global.
2. Suggest dimensional adjustments as JSON: {"verbosity": delta, "formality": delta, ...}
   where delta is in [-0.2, +0.2]. Only include dimensions that should change.

Respond as JSON:
{"guidelines": ["...", ...], "dimensionDeltas": {"verbosity": -0.1, ...}}
```

### Prompt Delivery — StrategyPromptSection

A `RoutingPromptSection` implementation (analogous to `OptimisedFewShotSection` and `CbrRoutingPromptSection`) that injects strategy context into agent system prompts:

```java
@ApplicationScoped
public class StrategyPromptSection implements RoutingPromptSection {
    @Inject StrategyStore strategyStore;

    @Override
    public String id() { return "strategy"; }

    @Override
    public @Nullable String render(AgentRoutingContext context) {
        var profile = strategyStore.lookup(
            context.agentRef().name(), context.tenantId());
        if (profile.isEmpty()) return null;

        var sb = new StringBuilder("## Interaction Strategy\n\n");
        for (String guideline : profile.get().guidelines()) {
            sb.append("- ").append(guideline).append('\n');
        }
        return sb.toString();
    }
}
```

Consumers that need per-subject strategy can query `strategyStore.subjectInsights()` and append subject-specific guidance at the prompt level.

## Type Summary

| Type | Kind | Description |
|------|------|-------------|
| `EngagementSignal` | sealed interface | Input signal: TurnOutcome, ConversationOutcome |
| `StrategyProfile` | record | Per-agent strategy: dimensions + guidelines |
| `StrategyLearningConfig` | record | Configuration with validation |
| `StrategyLearningTick` | sealed interface | tick() outcome: NoChange, Observed, Learned |
| `StrategyReflection` | sealed interface | reflect() outcome: NoChange, Reflected |
| `StrategyStore` | interface (SPI) | Strategy profile persistence |
| `CbrStrategyStore` | class (@DefaultBean) | CbrCaseMemoryStore-backed StrategyStore |
| `StrategyLearningOrchestrator` | class | Composition root: record() + tick() + reflect() |
| `StrategyPromptSection` | class (@ApplicationScoped) | Injects strategy into agent prompts |

9 new types. ~500 lines of production code estimated.

## Testing Strategy

| Test | What it verifies |
|------|-----------------|
| EngagementSignal validation | TurnOutcome requires non-null event, non-null snapshot; ConversationOutcome requires non-blank conversationId |
| StrategyProfile defaults | Default dimensions are all 0.5, empty guidelines |
| StrategyLearningConfig validation | Rejects invalid config (negative thresholds, out-of-range defaults) |
| CbrStrategyStore round-trip | store() → lookup() returns equivalent profile; eraseAgent() clears |
| CbrStrategyStore subjectInsights | Returns insights filtered by subjectId |
| tick() with no signals | Returns NoChange |
| tick() with TurnOutcome signals | Returns Observed with correct engagement metrics |
| tick() with ConversationOutcome | Returns Learned, stores CBR case with correct features |
| tick() feature extraction | avgResponseLength, continuationRate, meanSentimentShift computed correctly |
| tick() dimensional snapshot averaging | avgSnapshot_* features correctly averaged across signals |
| reflect() with insufficient cases | Returns NoChange |
| reflect() LLM synthesis | Guidelines extracted, dimensions adjusted, profile stored |
| reflect() dimensional clamping | Deltas that push dimensions outside [0,1] are clamped |
| reflect() TrendAnalyzer integration | TrendProfile produced from engagement case history |
| reflect() per-subject grouping | Cases grouped by subjectId for per-subject analysis |
| Concurrency | Concurrent record() + tick() on same agent — no lost signals |
| Stale state eviction | Agent states evicted after staleStateTimeout |
| StrategyPromptSection rendering | Formats guidelines as markdown list |

~45 tests estimated.

## References

- Research §2.6 (StrategyLearning pattern definition)
- Metacognitive Learning (ICML 2025, Liu & Van Der Schaar)
- Self-Learning Agents Enhanced by Multi-level Reflection (EMNLP 2025)
- Reflexion: Language Agents with Verbal Reinforcement Learning (NeurIPS 2023, Shinn et al.)
- MIRROR: Cognitive Inner Monologue Between Conversational Turns (2025, arXiv:2506.00430)
- EngagementEvent (neocortex-memory-api: `io.casehub.neocortex.memory.engagement`)
- ReflectionOrchestrator (neocortex-memory-api: `io.casehub.neocortex.memory.reflection`)
- TrendAnalyzer / TrendProfile (neocortex-memory-api: `io.casehub.neocortex.memory.cbr`)
- ContentSummariser (blocks: `io.casehub.blocks.summarisation`)
- UserModelOrchestrator (blocks: `io.casehub.blocks.agentic.social`) — tiered pattern precedent
- MemoryHygieneScheduler (blocks: `io.casehub.blocks.memory`) — tick()+maintain() dual-cadence precedent
- CbrUserProfileStore / CbrMentalModelStore — CbrCase adapter pattern precedent
- GE-20260820-c19b68 (producerAgentId post-filtering)
- GE-20260804-eb75e0 (scan() returns summaries — use retrieveSimilar)
- GE-20260820-d4e011 (non-blank solution placeholder)
- GE-20260813-keyed-summarisation-runner-api (KeyedSummarisationRunner API differences)
- Decisions D32–D41 in `decisions.md`
- Decision review R1-01 through R1-11
