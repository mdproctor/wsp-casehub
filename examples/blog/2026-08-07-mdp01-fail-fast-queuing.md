---
layout: post
title: "Fail-Fast Is Not Your Problem — Queuing Is"
date: 2026-08-07
entry_type: note
subtype: diary
projects: [casehub-examples]
tags: [concurrency, wacky-manor, agent-provider, mutiny]
---

When Wacky Manor's autonomous tick loop fires ten character agents simultaneously, the platform's Claude subprocess semaphore has six permits. Four callers beyond the character agents — narrator, observation summariser, scene beat calls — share the same semaphore. With a batch-of-3 workaround capping character concurrency, the system stayed under six most of the time. "Most of the time" is a terrible SLA for a game loop.

The root cause was architectural: four independent callers competing for a shared resource with no coordination between them, and the resource's backpressure signal treated as an error rather than a queue hint.

The platform's `ClaudeAgentClient` uses `tryAcquire()` — non-blocking, fail-fast. When the semaphore is full, it returns `AgentSessionLimitException` immediately. This is correct platform behaviour. The platform can't know whether its caller runs on a Vert.x IO thread (must not block) or a virtual thread (can block). It signals; the caller decides.

The problem: no caller decided well. `AgentInvocationService` caught the exception with exponential backoff — the wrong strategy for transient backpressure. After two retries the character fell back to idle, meaning "do nothing this turn." The narrator and summariser let it propagate as failures. Scene beat calls returned `"[character] is speechless"`. Silent degradation across the board.

And `AgentInvocationService` had a `maxConcurrent` constructor parameter that was accepted, stored nowhere, and used by nothing. Dead code giving the false impression that concurrency was handled at the application layer.

The fix was a `GatedAgentProvider` — a decorator wrapping `AgentProvider` with `tryAcquire(timeout)` (blocking, fair, with timeout). All four callers receive the gated provider instead of the raw CDI-injected one. The gate is sized at 5 (one below the platform's 6), so the platform semaphore is never exhausted.

```java
public Multi<AgentEvent> invoke(AgentSessionConfig config) {
    if (!gate.tryAcquire(acquireTimeout.toMillis(), TimeUnit.MILLISECONDS)) {
        return Multi.createFrom().failure(new AgentSessionLimitException(maxConcurrent));
    }
    try {
        return delegate.invoke(config)
                .onTermination().invoke(() -> gate.release());
    } catch (Exception e) {
        gate.release();
        return Multi.createFrom().failure(e);
    }
}
```

Mutiny's `onTermination()` fires on completion, failure, or cancellation — one handler covers all three cleanup paths. The synchronous catch handles the case where `delegate.invoke()` throws before returning a Multi.

With the gate in place, the batch loop becomes unnecessary. Instead of processing agents in groups of three with artificial synchronisation barriers between batches, all agents per tick fire concurrently. The semaphore pipelines them — as each call completes, the next queued call starts immediately. No batch-boundary stalls.

The performance arithmetic is straightforward. With 10 agents and a batch of 3, you get `ceil(10/3) = 4` batches, each waiting for its slowest agent. With a semaphore of 5, the first 5 start immediately and the remaining 5 start as slots free up — pipeline parallelism instead of synchronised waves.

The transferable principle: fail-fast at the platform layer and blocking at the application layer is the right two-tier design. The platform signals "I'm full" instantly — safe on any thread. The application decides whether to queue (blocking acquire on a virtual thread) or propagate (reactive IO thread). The mistake is treating the platform's backpressure as an error to retry, when it's a flow-control signal to respect.
