# Scale Testing — Beyond 5 Agents and 60 Turns

**Issue:** casehubio/examples#16
**Date:** 2026-08-05
**Status:** Reviewed (light — 4 dimensions, 34 findings, all resolved)

## Goal

Demonstrate wacky-manor running 10 agents for 200 logical turns with <30s average turn latency, maintained personality coherence, and at least one emergent social dynamic not seen in 5-agent runs. Validate across both the deterministic test runner (personality benchmarks) and the concurrent production orchestrator (latency/throughput stress testing).

**Scope note:** Issue #16 defines four scale dimensions. This spec addresses agent count (10), turn count (200+), and concurrent LLM calls. Room/location scaling (6→10 rooms) is deferred to a follow-up issue — the existing 6 rooms provide sufficient spatial diversity for the 10-agent test.

## Prerequisites (both shipped)

- **neocortex#184** — Agent experience stream (CLOSED). Structured event ingestion per agent with query by agent/time/type/relevance.
- **blocks#68** — Observation accumulator (CLOSED). Tiered rendering (verbatim/grouped/summarised) already integrated via `PartitionedObservationService`.

## Approach

Build and validate everything in wacky-manor first. Extract the rate limiter to the platform as a follow-up (#30).

Four components, built in order:

0. WorldState thread safety (prerequisite)
1. Agent invocation service (rate limiting, retry, metrics)
2. Agent experience service (ingest + recall)
3. Scale testing harness (both runners)

---

## 0. WorldState Thread Safety (Prerequisite)

### Problem

`WorldState` uses unsynchronized `HashMap`, `HashSet`, and `ArrayList` for shared mutable state (`rooms`, `characters`, `firedTriggers`, `takenObjects`, `eventLog`, etc.). With 5 agents and narrow timing windows, races are improbable. With 10 concurrent virtual threads, they become likely: double-take races on portable objects, `ArrayList` corruption on `eventLog`, and `ConcurrentModificationException` during observation iteration.

### Fix

- Replace `HashMap` fields with `ConcurrentHashMap`
- Replace `ArrayList eventLog` with a synchronized list or `CopyOnWriteArrayList`
- Synchronize `takenObjects` checks + mutations (check-then-act atomicity for TAKE actions)
- `characters()` returns an unmodifiable snapshot, not the raw mutable map

This is a prerequisite step — scale testing without it produces unreliable results.

---

## 1. AgentInvocationService

### Problem

Agent invocation logic is duplicated across `CharacterAgentLoop.callAgentWithRetry()` and `AutonomousScenarioRunner.callAgentWithRetry()` — same retry count, same timeout, same invoke→filter→collect→parse pipeline. The spec adds rate limiting, experience ingestion, and metrics, which would compound the duplication into two divergent copies with five concerns each.

### Component

`AgentInvocationService` — a single class owning the full agent call lifecycle:

1. Acquire concurrency semaphore (rate limiting)
2. Call `AgentProvider.invoke()` with timeout
3. Retry with exponential backoff + jitter on transient failure
4. Parse response
5. Ingest experience event (fire-and-forget)
6. Collect per-call metrics (latency, token count, success/failure)

Both `CharacterAgentLoop` and `AutonomousScenarioRunner` delegate to this service instead of calling `AgentProvider` directly. This resolves:
- Retry duplication
- CDI decorator gap (rate limiting composes inside the service, not via CDI decoration)
- Thundering herd (jitter added once, applied everywhere)
- Test runner rate limiting (service is manually constructable, not CDI-only)

### Rate Limiting: Concurrency Semaphore

Single control: `java.util.concurrent.Semaphore` capping max in-flight LLM calls.

**Why semaphore only (no token bucket):** At 10 agents with `max-concurrent=5` and ~8s average LLM call time, back-of-envelope:
- Agents 1-5: 0s wait + 8s call = 8s
- Agents 6-10: ~8s wait + 8s call = 16s
- Average turn latency: ~12s — well within the 30s target

Adding a token bucket on top creates multiplicative interaction with the semaphore and risks pushing worst-case latency past 30s. Token bucket is deferred to platform extraction (#31) where it makes sense for sustained multi-tenant throughput.

### Retry Strategy

Exponential backoff with jitter: `base_delay * 2^attempt + random(0, base_delay)`. Prevents thundering herd when multiple agents fail simultaneously. Max 2 retries, then fall back to idle response.

### Configuration

```properties
manor.agent.rate-limit.max-concurrent=5
manor.agent.invocation.timeout-seconds=60
manor.agent.invocation.max-retries=2
manor.agent.invocation.base-retry-delay-ms=2000
```

### Not in Scope (follow-up issues on #30)

- Token bucket / sustained throughput control (#31)
- Circuit breakers and fallback chains (#32)
- Multi-model and multi-tenant awareness (#33)

---

## 2. AgentExperienceService

### Problem

Without persistent memory, agents only see events since their last observation drain. At turn 150, an agent has no recall of turns 1-100. This makes 200+ turn runs meaningless.

### Component

`AgentExperienceService` — owns both ingestion and recall, injectable and testable in isolation.

### Ingestion — Fire-and-Forget

Called by `AgentInvocationService` after each successful LLM call (step 5 in the invocation pipeline). Writes are wrapped in try-catch — failures are logged, never propagated. An experience stream outage degrades memory but does not kill agents.

Experience event fields:
- **Who:** character agent ID
- **What:** action taken + result, dialogue spoken
- **Where:** current room
- **When:** timestamp
- **Why:** the `thinking` field from the LLM response

One record per agent per turn. At 10 agents x 200 turns = 2,000 records per run.

### Recall — Pre-Queried, Not Inline

`ObservationBuilder` stays pure (static, no service dependencies). The caller (`AgentInvocationService` or the agent loop) queries `AgentExperienceService.recall(agentId, currentRoom, limit)` before building the observation, then passes the result as an additional parameter to `ObservationBuilder.buildObservation()`.

Recall query:
- Uses neocortex's query API: filter by agent ID, ordered by recency
- Relevance boost for same room, same characters, same objects
- Timeout: 2s — degrades to empty recall on timeout (no stall)
- Budget: cap returned text at ~500 tokens (character-count heuristic: ~2000 chars)

### Observation Section

New section named **"Past Experience"** (not "Memory" — avoids collision with existing "Remembered" section which shows compacted observation-drain data). Placed between "Remembered" and "Last Action Result".

### Interaction with Observation Accumulator

The observation accumulator (blocks#68) and experience stream serve overlapping but distinct purposes:
- **Accumulator ("Remembered"):** compacted recent events from rooms previously visited — short-term, event-level
- **Experience stream ("Past Experience"):** episodic memories from the agent's own history — long-term, action-level

No deduplication needed — they operate at different granularities. The accumulator shows "what happened while you were away" (other agents' actions); the experience stream shows "what you did before" (your own past actions and reasoning).

---

## 3. Scale Testing Harness

### Turn Semantics Fix

`ScenarioOrchestrator.turnCount` currently increments per action polled — with 10 agents, "200 turns" terminates after ~20 logical turns per agent. Fix: count a logical turn as one full cycle where all active agents have acted (track via a set of agents-who-acted-this-cycle, increment turn counter when set is full, reset).

Both runners use the same definition: one turn = all active agents act once.

### AutonomousScenarioRunner (Deterministic Benchmarks)

- **Configurable character list** — accept a `List<String>` parameter. Default: discover from `AgentRegistry`.
- **Descriptor profile:** composite (`descriptors-composite.yaml`) — the only profile with all 17 characters.
- **Wire observation pipeline** — real `ObservationService` instead of empty `PartitionedDrain`.
- **Delegate to AgentInvocationService** — replaces inline `callAgentWithRetry`. Gets rate limiting, experience ingestion, and metrics for free.
- **ScaleReport** — extends `TranscriptRecorder.RunResult` with: avg/p95/p99 turn latency, per-agent interaction count, memory recall hit rate. Output: JSON to file + console summary log.

### ScenarioOrchestrator (Concurrency Stress Test)

- **Delegate to AgentInvocationService** — `CharacterAgentLoop` calls the service instead of its own `callAgentWithRetry`.
- **PendingAction timeout** — `awaitResult()` gets a 60s timeout. On timeout, agent logs warning and continues with a "your action timed out" result. Orphaned actions from crashed agents are cleaned up by the orchestrator (skip actions for inactive characters).
- **Metrics collection** — per-agent turn latency, rate limiter wait time, action queue depth. Logged at scenario end.

### Personality Coherence Baseline

Wiring the real observation pipeline changes what agents see (from bare prompts to rich observations + memory). Existing 5-agent coherence baselines are invalidated. Before measuring 10-agent coherence:

1. Re-baseline at 5 agents with the wired pipeline (new baseline)
2. Then scale to 10 agents and compare against the new baseline

Personality coherence is measured in **both runners** — concurrent execution may degrade coherence in ways the sequential runner wouldn't catch.

### Character Selection for 10-Agent Benchmark

17 characters across 6 rooms, all with composite descriptors. Core 5 + 5 with diverse starts and think delays:

| Character | Start Room | thinkDelayMs | Rationale |
|-----------|-----------|-------------|-----------|
| penelope-pitstop | entrance-hall | default | Core 5 |
| hooded-claw | entrance-hall | default | Core 5 |
| ant-hill-mob | entrance-hall | default | Core 5 |
| dick-dastardly | entrance-hall | default | Core 5 |
| peter-perfect | entrance-hall | default | Core 5 |
| muttley | entrance-hall | 2000 | Fast thinker, pairs with dastardly |
| pat-pending | entrance-hall | 4000 | Inventor archetype, different goals |
| lazy-luke | ballroom | 8000 | Slow thinker — stress tests staggering |
| rock-slag | library | 2000 | Different starting room |
| rufus-ruffcut | laboratory | 3000 | Different starting room |

4 starting rooms (entrance-hall, ballroom, library, laboratory), spread of think delays (2s-8s).

---

## Success Criteria

| Criterion | Measurement | Runner |
|-----------|------------|--------|
| 10 agents, 200 logical turns, no crashes | Run completes without exceptions | Both |
| Turn latency < 30s average | Per-turn wall clock in ScaleReport | ScenarioOrchestrator |
| Personality coherence maintained | MBTI judges against 5-agent wired-pipeline baseline | Both |
| Emergent social dynamic | Manual transcript review | ScenarioOrchestrator |

## Issue Tree

- **#16** — this issue (scale testing, agent invocation service, experience service)
- **#30** — epic: Extract rate limiter to platform AgentProvider (after #16 validates it)
  - **#31** — Token-aware (TPM) rate limiting
  - **#32** — Circuit breakers and fallback chains
  - **#33** — Multi-model and multi-tenant awareness
- **#34** — Room/location scaling (6→10 rooms, deferred from #16)

## Review Log

Light review, 4 dimensions, $5.12 total. 34 findings → 8 themes, all accepted:

1. **Shared agent invocation** → extracted `AgentInvocationService`
2. **Experience stream architecture** → extracted `AgentExperienceService`, ObservationBuilder stays pure
3. **WorldState thread safety** → added as prerequisite step (§0)
4. **Turn semantics** → fixed to logical turns, consistent across runners
5. **Rate limiter simplification** → dropped token bucket, semaphore-only
6. **Scope gaps** → room scaling deferred, room count corrected
7. **Personality coherence baseline** → re-baseline at 5 agents, measure in both runners
8. **PendingAction safety** → added timeout, orphan cleanup

## References

- POC spec: `wacky-manor/docs/POC-SPEC.md`
- Vision: `wacky-manor/docs/VISION.md`
- Trellis rate limiter: `sidecar/src/main/java/io/hortora/trellis/coordinator/ActionService.java` (lines 272-292)
- Industry research: token bucket recommended for bursty multi-agent LLM traffic
- LangChain4j has no built-in rate limiting — platform-level is a differentiator
