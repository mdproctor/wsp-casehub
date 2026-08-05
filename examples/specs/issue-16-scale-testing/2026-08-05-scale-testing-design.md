# Scale Testing — Beyond 5 Agents and 60 Turns

**Issue:** casehubio/examples#16
**Date:** 2026-08-05
**Status:** Draft

## Goal

Demonstrate wacky-manor running 10 agents for 200 turns with <30s average turn latency, maintained personality coherence, and at least one emergent social dynamic not seen in 5-agent runs. Validate across both the deterministic test runner (personality benchmarks) and the concurrent production orchestrator (latency/throughput stress testing).

## Prerequisites (both shipped)

- **neocortex#184** — Agent experience stream (CLOSED). Structured event ingestion per agent with query by agent/time/type/relevance.
- **blocks#68** — Observation accumulator (CLOSED). Tiered rendering (verbatim/grouped/summarised) already integrated via `PartitionedObservationService`.

## Approach

Build and validate everything in wacky-manor first. Extract the rate limiter to the platform as a follow-up (#30).

Three components, built in order:

1. Rate-limited AgentProvider wrapper
2. Experience stream integration (ingest + recall)
3. Scale testing harness (both runners)

---

## 1. Rate-Limited AgentProvider Wrapper

### Problem

All character agents call `AgentProvider.invoke()` concurrently via virtual threads. With 10+ agents, this means 10+ simultaneous LLM API calls per turn cycle — no concurrency control, no throughput management.

### Algorithm: Token Bucket + Concurrency Cap

Two orthogonal controls, each solving a different problem:

- **Concurrency cap** (`java.util.concurrent.Semaphore`) — max simultaneous in-flight LLM calls. Agents block on the semaphore (virtual threads make blocking cheap) rather than getting rejected. Prevents overwhelming the API provider.
- **Token bucket** — refills at a fixed rate. Each call consumes one token. If bucket is empty, thread blocks until a token is available. Controls sustained throughput over time, allows controlled bursts.

### Why Token Bucket Over Sliding Window

Agents are inherently bursty — all N fire simultaneously at the start of each turn cycle, then idle during action resolution. Token bucket handles this naturally: burst up to bucket capacity, then meter the overflow. Sliding window would reject agents that could safely be queued. Trellis's sliding window pattern is correct for its use case (human-interactive action approval) but wrong for LLM call batching.

### Component

`RateLimitedAgentProvider` — CDI decorator wrapping `AgentProvider`.

ScenarioOrchestrator and CharacterAgentLoop inject `AgentProvider` unchanged — they get the rate-limited version transparently.

### Configuration

```properties
manor.agent.rate-limit.max-concurrent=4
manor.agent.rate-limit.bucket-capacity=8
manor.agent.rate-limit.refill-rate=10
manor.agent.rate-limit.refill-period-seconds=60
```

### Not in Scope (follow-up issues on #30)

- Token-aware (TPM) rate limiting (#31)
- Circuit breakers and fallback chains (#32)
- Multi-model and multi-tenant awareness (#33)

---

## 2. Experience Stream Integration

### Problem

Without persistent memory, agents only see events since their last observation drain. At turn 150, an agent has no recall of turns 1-100. This makes 200+ turn runs meaningless — agents repeat themselves and lose narrative coherence.

### Ingestion — After Each Turn

In `CharacterAgentLoop.run()`, after action resolution, write an experience event to the neocortex stream:

- **Who:** character agent ID
- **What:** action taken + result, dialogue spoken, observations received
- **Where:** current room
- **When:** timestamp
- **Why:** the `thinking` field from the LLM response

One experience record per agent per turn. At 10 agents x 200 turns = 2,000 records per run.

### Recall — Into Observation Context

In `ObservationBuilder.buildObservation()`, add a "Memory" section between "Remembered" and "Last Action Result":

- Query the experience stream for this agent's relevant past experiences
- Filter by recency + relevance (same room, same characters, same objects)
- Render as summarised past experiences
- Budget: cap at ~500 tokens to avoid context bloat

### Test Harness Impact

`AutonomousScenarioRunner` currently passes an empty `PartitionedDrain`. With experience stream integration, the runner must wire the same ingestion/recall pipeline so benchmarks reflect real memory behaviour.

---

## 3. Scale Testing Harness

### AutonomousScenarioRunner (Deterministic Benchmarks)

Currently hardcodes 5 characters in `CHARACTER_ORDER`. Changes:

- **Configurable character list** — accept a `List<String>` instead of the hardcoded constant. Default to all characters with Eidos descriptors (discover from `AgentRegistry`).
- **Wire observation pipeline** — currently passes empty `PartitionedDrain`. Needs real `ObservationService` for memory to work.
- **Wire experience stream** — same ingestion/recall as production path.
- **Per-turn metrics** — LLM call latency, prompt token count, response parse success rate.
- **ScaleReport output** — total duration, avg/p95/p99 turn latency, personality coherence scores, agent interaction count, memory recall hits.

### ScenarioOrchestrator (Concurrency Stress Test)

Already supports configurable `active-characters` and concurrent virtual threads. Changes:

- **Wire experience stream** — same integration as above.
- **Metrics collection** — per-agent turn latency, rate limiter wait time, action queue depth.
- **ScaleTestProfile** — Quarkus test profile that activates 10 agents, 200 turn limit, and injects a test `AgentProvider` wired with the rate limiter.
- **Metrics endpoint** — expose via REST or log summary at scenario end.

### Character Selection for 10-Agent Benchmark

17 characters exist across 6 rooms. For the 10-agent target — core 5 + 5 with diverse room starts and think delays:

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

3 starting rooms, spread of think delays (2s-8s), character pairs with natural interaction dynamics.

---

## Success Criteria

| Criterion | Measurement | Runner |
|-----------|------------|--------|
| 10 agents, 200 turns, no crashes | Run completes without exceptions | Both |
| Turn latency < 30s average | Per-turn wall clock in ScaleReport | ScenarioOrchestrator |
| Personality coherence maintained | MBTI alignment judges (existing PromptQualityTest) | AutonomousScenarioRunner |
| Emergent social dynamic | Manual transcript review | ScenarioOrchestrator |

## Issue Tree

- **#16** — this issue (scale testing, manor-local rate limiter, experience stream integration)
- **#30** — epic: Extract rate limiter to platform AgentProvider (after #16 validates it)
  - **#31** — Token-aware (TPM) rate limiting
  - **#32** — Circuit breakers and fallback chains
  - **#33** — Multi-model and multi-tenant awareness

## References

- POC spec: `wacky-manor/docs/POC-SPEC.md`
- Vision: `wacky-manor/docs/VISION.md`
- Trellis rate limiter: `sidecar/src/main/java/io/hortora/trellis/coordinator/ActionService.java` (lines 272-292)
- Industry research: token bucket recommended for bursty multi-agent LLM traffic
- LangChain4j has no built-in rate limiting — platform-level is a differentiator
