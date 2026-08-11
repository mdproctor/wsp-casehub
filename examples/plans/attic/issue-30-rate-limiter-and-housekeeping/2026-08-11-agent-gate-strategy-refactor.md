# Agent Gate Strategy Refactor — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #30 — epic: Platform LLM rate limiter — extract from wacky-manor to AgentProvider
**Issue group:** #30, #1, #6

**Goal:** Refactor `agent-gate` to use composable `AdmissionStrategy` implementations, then migrate wacky-manor and trellis to use the platform module.

**Architecture:** The existing hardcoded token bucket + semaphore in `GatedAgentProvider` becomes a list of `AdmissionStrategy` implementations. Each strategy declares a `Scope` (INVOCATION or SESSION) that determines when it participates. Three strategies: `ConcurrencyStrategy` (semaphore), `TokenBucketStrategy` (token bucket), `SlidingWindowStrategy` (sliding window from trellis). Config moves to nested groups.

**Tech Stack:** Java 26, Quarkus, SmallRye Config, SmallRye Mutiny, JUnit 5, AssertJ

## Global Constraints

- All strategies must be thread-safe (concurrent `invoke()` calls from virtual threads)
- `tryAcquire()` is blocking — `invoke()` path must use `runSubscriptionOn` for the reactive bridge
- Acquisition order: cheapest-to-reject first (sliding window → token bucket → concurrency)
- Config prefix: `casehub.platform.agent.gate.<strategy>.*`
- Pre-release: breaking config changes are acceptable

## Repos

| Repo | Path | Tasks |
|---|---|---|
| platform | `/Users/mdproctor/claude/casehub/platform` | 1–5 |
| examples | `/Users/mdproctor/claude/casehub/slots/111/examples` | 6 |
| trellis | `/Users/mdproctor/claude/hortora/trellis` | 7 |

---

### Task 1: AdmissionStrategy interface + ConcurrencyStrategy

**Files:**
- Create: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/AdmissionStrategy.java`
- Create: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/ConcurrencyStrategy.java`
- Test: `platform/agent-gate/src/test/java/io/casehub/platform/agent/gate/ConcurrencyStrategyTest.java`

**Interfaces:**
- Produces: `AdmissionStrategy` interface with `Scope scope()`, `boolean tryAcquire(Duration)`, `void release()`, `void rollback()`. `ConcurrencyStrategy(int maxConcurrent)` constructor.

- [ ] **Step 1: Write AdmissionStrategy interface**

```java
package io.casehub.platform.agent.gate;

import java.time.Duration;

public interface AdmissionStrategy {

    enum Scope { INVOCATION, SESSION }

    Scope scope();

    boolean tryAcquire(Duration timeout) throws InterruptedException;

    void release();

    void rollback();
}
```

- [ ] **Step 2: Write ConcurrencyStrategy tests**

```java
package io.casehub.platform.agent.gate;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

import static org.assertj.core.api.Assertions.assertThat;

class ConcurrencyStrategyTest {

    @Test
    void scopeIsSession() {
        var strategy = new ConcurrencyStrategy(1);
        assertThat(strategy.scope()).isEqualTo(AdmissionStrategy.Scope.SESSION);
    }

    @Test
    void acquireAndRelease() throws Exception {
        var strategy = new ConcurrencyStrategy(1);
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        strategy.release();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
    }

    @Test
    void acquireAndRollback() throws Exception {
        var strategy = new ConcurrencyStrategy(1);
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        strategy.rollback();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
    }

    @Test
    void timeoutWhenAllPermitsHeld() throws Exception {
        var strategy = new ConcurrencyStrategy(1);
        strategy.tryAcquire(Duration.ofSeconds(1));
        assertThat(strategy.tryAcquire(Duration.ofMillis(50))).isFalse();
    }

    @Test
    void blocksUntilPermitReleased() throws Exception {
        var strategy = new ConcurrencyStrategy(1);
        strategy.tryAcquire(Duration.ofSeconds(1));
        var acquired = new AtomicBoolean(false);
        var latch = new CountDownLatch(1);
        Thread.ofVirtual().start(() -> {
            try {
                acquired.set(strategy.tryAcquire(Duration.ofSeconds(5)));
            } catch (InterruptedException ignored) {}
            latch.countDown();
        });
        Thread.sleep(50);
        strategy.release();
        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(acquired.get()).isTrue();
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=ConcurrencyStrategyTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL — `ConcurrencyStrategy` not found

- [ ] **Step 4: Implement ConcurrencyStrategy**

```java
package io.casehub.platform.agent.gate;

import java.time.Duration;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public final class ConcurrencyStrategy implements AdmissionStrategy {

    private final Semaphore gate;

    public ConcurrencyStrategy(int maxConcurrent) {
        this.gate = new Semaphore(maxConcurrent, true);
    }

    @Override
    public Scope scope() {
        return Scope.SESSION;
    }

    @Override
    public boolean tryAcquire(Duration timeout) throws InterruptedException {
        return gate.tryAcquire(timeout.toMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public void release() {
        gate.release();
    }

    @Override
    public void rollback() {
        gate.release();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=ConcurrencyStrategyTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS (5 tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add agent-gate/src/main/java/io/casehub/platform/agent/gate/AdmissionStrategy.java agent-gate/src/main/java/io/casehub/platform/agent/gate/ConcurrencyStrategy.java agent-gate/src/test/java/io/casehub/platform/agent/gate/ConcurrencyStrategyTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#30): AdmissionStrategy interface + ConcurrencyStrategy

Refs casehubio/examples#30"
```

---

### Task 2: TokenBucketStrategy

**Files:**
- Create: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/TokenBucketStrategy.java`
- Test: `platform/agent-gate/src/test/java/io/casehub/platform/agent/gate/TokenBucketStrategyTest.java`

**Interfaces:**
- Consumes: `AdmissionStrategy` interface (Task 1)
- Produces: `TokenBucketStrategy(double permitsPerSecond, int burstCapacity)` constructor

- [ ] **Step 1: Write TokenBucketStrategy tests**

```java
package io.casehub.platform.agent.gate;

import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

class TokenBucketStrategyTest {

    @Test
    void scopeIsInvocation() {
        var strategy = new TokenBucketStrategy(1.0, 1);
        assertThat(strategy.scope()).isEqualTo(AdmissionStrategy.Scope.INVOCATION);
    }

    @Test
    void acquireConsumesToken() throws Exception {
        var strategy = new TokenBucketStrategy(1.0, 1);
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        assertThat(strategy.tryAcquire(Duration.ofMillis(50))).isFalse();
    }

    @Test
    void releaseIsNoOp() throws Exception {
        var strategy = new TokenBucketStrategy(1.0, 1);
        strategy.tryAcquire(Duration.ofSeconds(1));
        strategy.release();
        assertThat(strategy.tryAcquire(Duration.ofMillis(50))).isFalse();
    }

    @Test
    void rollbackReturnsToken() throws Exception {
        var strategy = new TokenBucketStrategy(1.0, 1);
        strategy.tryAcquire(Duration.ofSeconds(1));
        strategy.rollback();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
    }

    @Test
    void burstAllowsMultipleAcquisitions() throws Exception {
        var strategy = new TokenBucketStrategy(1.0, 3);
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        assertThat(strategy.tryAcquire(Duration.ofMillis(50))).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=TokenBucketStrategyTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement TokenBucketStrategy**

```java
package io.casehub.platform.agent.gate;

import java.time.Duration;

public final class TokenBucketStrategy implements AdmissionStrategy {

    private final TokenBucket bucket;

    public TokenBucketStrategy(double permitsPerSecond, int burstCapacity) {
        this.bucket = new TokenBucket(permitsPerSecond, burstCapacity);
    }

    @Override
    public Scope scope() {
        return Scope.INVOCATION;
    }

    @Override
    public boolean tryAcquire(Duration timeout) throws InterruptedException {
        return bucket.tryAcquire(timeout);
    }

    @Override
    public void release() {
        // no-op — token consumed at admission
    }

    @Override
    public void rollback() {
        bucket.release();
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=TokenBucketStrategyTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add agent-gate/src/main/java/io/casehub/platform/agent/gate/TokenBucketStrategy.java agent-gate/src/test/java/io/casehub/platform/agent/gate/TokenBucketStrategyTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#30): TokenBucketStrategy — wraps TokenBucket as AdmissionStrategy

Refs casehubio/examples#30"
```

---

### Task 3: SlidingWindowStrategy

**Files:**
- Create: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/SlidingWindowStrategy.java`
- Test: `platform/agent-gate/src/test/java/io/casehub/platform/agent/gate/SlidingWindowStrategyTest.java`

**Interfaces:**
- Consumes: `AdmissionStrategy` interface (Task 1)
- Produces: `SlidingWindowStrategy(int maxActions, Duration windowSize)` constructor

- [ ] **Step 1: Write SlidingWindowStrategy tests**

```java
package io.casehub.platform.agent.gate;

import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

class SlidingWindowStrategyTest {

    @Test
    void scopeIsInvocation() {
        var strategy = new SlidingWindowStrategy(5, Duration.ofSeconds(60));
        assertThat(strategy.scope()).isEqualTo(AdmissionStrategy.Scope.INVOCATION);
    }

    @Test
    void admitsWithinLimit() throws Exception {
        var strategy = new SlidingWindowStrategy(3, Duration.ofSeconds(60));
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
    }

    @Test
    void rejectsWhenLimitReached() throws Exception {
        var strategy = new SlidingWindowStrategy(2, Duration.ofSeconds(60));
        strategy.tryAcquire(Duration.ofSeconds(1));
        strategy.tryAcquire(Duration.ofSeconds(1));
        assertThat(strategy.tryAcquire(Duration.ofMillis(50))).isFalse();
    }

    @Test
    void rollbackRemovesTimestamp() throws Exception {
        var strategy = new SlidingWindowStrategy(1, Duration.ofSeconds(60));
        strategy.tryAcquire(Duration.ofSeconds(1));
        strategy.rollback();
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
    }

    @Test
    void releaseIsNoOp() throws Exception {
        var strategy = new SlidingWindowStrategy(1, Duration.ofSeconds(60));
        strategy.tryAcquire(Duration.ofSeconds(1));
        strategy.release();
        assertThat(strategy.tryAcquire(Duration.ofMillis(50))).isFalse();
    }

    @Test
    void expiredTimestampsArePruned() throws Exception {
        var strategy = new SlidingWindowStrategy(1, Duration.ofMillis(100));
        strategy.tryAcquire(Duration.ofSeconds(1));
        assertThat(strategy.tryAcquire(Duration.ofMillis(10))).isFalse();
        Thread.sleep(150);
        assertThat(strategy.tryAcquire(Duration.ofSeconds(1))).isTrue();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=SlidingWindowStrategyTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement SlidingWindowStrategy**

```java
package io.casehub.platform.agent.gate;

import java.time.Duration;
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public final class SlidingWindowStrategy implements AdmissionStrategy {

    private final ReentrantLock lock = new ReentrantLock(true);
    private final Condition slotAvailable = lock.newCondition();
    private final Deque<Long> timestamps = new ArrayDeque<>();
    private final int maxActions;
    private final long windowNanos;

    public SlidingWindowStrategy(int maxActions, Duration windowSize) {
        this.maxActions = maxActions;
        this.windowNanos = windowSize.toNanos();
    }

    @Override
    public Scope scope() {
        return Scope.INVOCATION;
    }

    @Override
    public boolean tryAcquire(Duration timeout) throws InterruptedException {
        long deadlineNanos = System.nanoTime() + timeout.toNanos();
        lock.lockInterruptibly();
        try {
            while (true) {
                pruneExpired();
                if (timestamps.size() < maxActions) {
                    timestamps.addLast(System.nanoTime());
                    return true;
                }
                long remainingNanos = deadlineNanos - System.nanoTime();
                if (remainingNanos <= 0) {
                    return false;
                }
                long oldestExpiry = timestamps.peekFirst() + windowNanos;
                long waitNanos = Math.min(oldestExpiry - System.nanoTime(), remainingNanos);
                if (waitNanos > 0) {
                    slotAvailable.await(waitNanos, TimeUnit.NANOSECONDS);
                }
            }
        } finally {
            lock.unlock();
        }
    }

    @Override
    public void release() {
        // no-op — counts admissions, not completions
    }

    @Override
    public void rollback() {
        lock.lock();
        try {
            if (!timestamps.isEmpty()) {
                timestamps.pollLast();
                slotAvailable.signal();
            }
        } finally {
            lock.unlock();
        }
    }

    private void pruneExpired() {
        long cutoff = System.nanoTime() - windowNanos;
        while (!timestamps.isEmpty() && timestamps.peekFirst() < cutoff) {
            timestamps.pollFirst();
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=SlidingWindowStrategyTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add agent-gate/src/main/java/io/casehub/platform/agent/gate/SlidingWindowStrategy.java agent-gate/src/test/java/io/casehub/platform/agent/gate/SlidingWindowStrategyTest.java
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#30): SlidingWindowStrategy — rolling count rate limiter

Refs casehubio/examples#30"
```

---

### Task 4: GatedAgentProvider + config refactor

**Files:**
- Modify: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/AgentGateProperties.java`
- Modify: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/GatedAgentProvider.java`
- Modify: `platform/agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentProviderTest.java`
- Modify: `platform/agent-gate/src/test/resources/application.properties`
- Test: `platform/agent-gate/src/test/java/io/casehub/platform/agent/gate/StrategyCompositionTest.java` (new)

**Interfaces:**
- Consumes: `AdmissionStrategy`, `ConcurrencyStrategy` (Task 1), `TokenBucketStrategy` (Task 2), `SlidingWindowStrategy` (Task 3)
- Produces: Refactored `GatedAgentProvider` that uses `List<AdmissionStrategy>`

- [ ] **Step 1: Write composition test — rollback on partial acquisition failure**

```java
package io.casehub.platform.agent.gate;

import org.junit.jupiter.api.Test;

import java.time.Duration;

import static org.assertj.core.api.Assertions.assertThat;

class StrategyCompositionTest {

    @Test
    void rollbackPriorStrategiesWhenLaterFails() throws Exception {
        var tokenBucket = new TokenBucketStrategy(1.0, 1);
        var concurrency = new ConcurrencyStrategy(0);

        var strategies = java.util.List.<AdmissionStrategy>of(tokenBucket, concurrency);
        boolean acquired = acquireAll(strategies, Duration.ofMillis(50));

        assertThat(acquired).isFalse();
        // token should have been rolled back
        assertThat(tokenBucket.tryAcquire(Duration.ofMillis(50))).isTrue();
    }

    @Test
    void acquireAllSucceedsWhenAllStrategiesSucceed() throws Exception {
        var tokenBucket = new TokenBucketStrategy(10.0, 10);
        var concurrency = new ConcurrencyStrategy(5);

        var strategies = java.util.List.<AdmissionStrategy>of(tokenBucket, concurrency);
        boolean acquired = acquireAll(strategies, Duration.ofSeconds(1));

        assertThat(acquired).isTrue();
    }

    @Test
    void acquisitionOrderIsPreserved() throws Exception {
        var order = new java.util.ArrayList<String>();
        var s1 = new TrackingStrategy("first", order, true);
        var s2 = new TrackingStrategy("second", order, true);

        acquireAll(java.util.List.of(s1, s2), Duration.ofSeconds(1));

        assertThat(order).containsExactly("first:acquire", "second:acquire");
    }

    private static boolean acquireAll(java.util.List<AdmissionStrategy> strategies,
                                       Duration timeout) throws InterruptedException {
        long deadlineNanos = System.nanoTime() + timeout.toNanos();
        for (int i = 0; i < strategies.size(); i++) {
            long remainingNanos = deadlineNanos - System.nanoTime();
            Duration remaining = remainingNanos > 0 ? Duration.ofNanos(remainingNanos) : Duration.ZERO;
            if (!strategies.get(i).tryAcquire(remaining)) {
                for (int j = i - 1; j >= 0; j--) {
                    strategies.get(j).rollback();
                }
                return false;
            }
        }
        return true;
    }

    private record TrackingStrategy(String name, java.util.List<String> log,
                                     boolean shouldSucceed) implements AdmissionStrategy {
        @Override public Scope scope() { return Scope.INVOCATION; }
        @Override public boolean tryAcquire(Duration timeout) {
            log.add(name + ":acquire");
            return shouldSucceed;
        }
        @Override public void release() { log.add(name + ":release"); }
        @Override public void rollback() { log.add(name + ":rollback"); }
    }
}
```

- [ ] **Step 2: Run composition tests to verify they pass** (these test the strategy primitives, not the provider yet)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -Dtest=StrategyCompositionTest -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: PASS

- [ ] **Step 3: Refactor AgentGateProperties to nested config groups**

Replace the contents of `AgentGateProperties.java`:

```java
package io.casehub.platform.agent.gate;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

import java.time.Duration;

@ConfigMapping(prefix = "casehub.platform.agent.gate")
public interface AgentGateProperties {

    @WithDefault("PT30S")
    Duration acquireTimeout();

    @WithDefault("PT5S")
    Duration queryAcquireTimeout();

    Concurrency concurrency();
    TokenBucketConfig tokenBucket();
    SlidingWindow slidingWindow();

    interface Concurrency {
        @WithDefault("0")
        int max();
    }

    interface TokenBucketConfig {
        @WithDefault("0.0")
        double permitsPerSecond();

        @WithDefault("0")
        int burstCapacity();
    }

    interface SlidingWindow {
        @WithDefault("0")
        int maxActions();

        @WithDefault("60")
        int windowSeconds();
    }
}
```

- [ ] **Step 4: Update test application.properties**

```properties
casehub.platform.agent.gate.concurrency.max=1
casehub.platform.agent.gate.token-bucket.permits-per-second=100.0
casehub.platform.agent.gate.acquire-timeout=PT1S
casehub.platform.agent.gate.query-acquire-timeout=PT1S
```

- [ ] **Step 5: Refactor GatedAgentProvider to use List\<AdmissionStrategy\>**

Replace the body of `GatedAgentProvider`. The class keeps `@Decorator @Priority(APPLICATION)`. Key changes:
- `@PostConstruct init()` builds the strategy list from config
- `invoke()` acquires all strategies (shared deadline), wraps stream with release-on-termination
- `openSession()` acquires all strategies, returns `GatedAgentSession` with partitioned lists
- Rollback on partial acquisition failure
- `runSubscriptionOn(Infrastructure.getDefaultWorkerPool())` preserved

```java
package io.casehub.platform.agent.gate;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentRateLimitException;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.platform.agent.AgentSessionLimitException;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.infrastructure.Infrastructure;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.Priority;
import jakarta.decorator.Decorator;
import jakarta.decorator.Delegate;
import jakarta.enterprise.inject.Any;
import jakarta.inject.Inject;
import jakarta.interceptor.Interceptor;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

@Decorator
@Priority(Interceptor.Priority.APPLICATION)
public class GatedAgentProvider implements AgentProvider {

    @Inject @Delegate @Any AgentProvider delegate;
    @Inject AgentGateProperties properties;

    private List<AdmissionStrategy> strategies = List.of();
    private List<AdmissionStrategy> sessionStrategies = List.of();
    private List<AdmissionStrategy> invocationStrategies = List.of();
    private Duration acquireTimeout;
    private Duration queryAcquireTimeout;
    private boolean active;

    protected GatedAgentProvider() {}

    GatedAgentProvider(AgentProvider delegate, List<AdmissionStrategy> strategies,
                       Duration acquireTimeout, Duration queryAcquireTimeout) {
        this.delegate = delegate;
        this.acquireTimeout = acquireTimeout;
        this.queryAcquireTimeout = queryAcquireTimeout;
        setStrategies(strategies);
    }

    @PostConstruct
    void init() {
        if (properties == null) return;
        this.acquireTimeout = properties.acquireTimeout();
        this.queryAcquireTimeout = properties.queryAcquireTimeout();

        var built = new ArrayList<AdmissionStrategy>();
        var sw = properties.slidingWindow();
        if (sw.maxActions() > 0) {
            built.add(new SlidingWindowStrategy(sw.maxActions(),
                    Duration.ofSeconds(sw.windowSeconds())));
        }
        var tb = properties.tokenBucket();
        if (tb.permitsPerSecond() > 0) {
            int burst = tb.burstCapacity() > 0 ? tb.burstCapacity()
                    : (int) Math.ceil(tb.permitsPerSecond());
            built.add(new TokenBucketStrategy(tb.permitsPerSecond(), burst));
        }
        var cc = properties.concurrency();
        if (cc.max() > 0) {
            built.add(new ConcurrencyStrategy(cc.max()));
        }
        setStrategies(built);
    }

    private void setStrategies(List<AdmissionStrategy> all) {
        this.strategies = List.copyOf(all);
        this.sessionStrategies = all.stream()
                .filter(s -> s.scope() == AdmissionStrategy.Scope.SESSION)
                .toList();
        this.invocationStrategies = all.stream()
                .filter(s -> s.scope() == AdmissionStrategy.Scope.INVOCATION)
                .toList();
        this.active = !all.isEmpty();
    }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        if (!active) {
            return delegate.invoke(config);
        }
        return Multi.createFrom().<AgentEvent>deferred(() -> {
            acquireAll(strategies, acquireTimeout);
            try {
                Multi<AgentEvent> result = delegate.invoke(config);
                return result.onTermination()
                        .invoke(() -> releaseAll(strategies));
            } catch (Exception e) {
                releaseAll(strategies);
                throw e;
            }
        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        if (!active) {
            return delegate.openSession(init);
        }
        acquireAll(strategies, acquireTimeout);
        try {
            AgentSession session = delegate.openSession(init);
            return new GatedAgentSession(session, sessionStrategies,
                    invocationStrategies, queryAcquireTimeout);
        } catch (Exception e) {
            releaseAll(strategies);
            throw e;
        }
    }

    static void acquireAll(List<AdmissionStrategy> strategies,
                            Duration timeout) {
        long deadlineNanos = System.nanoTime() + timeout.toNanos();
        for (int i = 0; i < strategies.size(); i++) {
            long remainingNanos = deadlineNanos - System.nanoTime();
            Duration remaining = remainingNanos > 0
                    ? Duration.ofNanos(remainingNanos) : Duration.ZERO;
            try {
                if (!strategies.get(i).tryAcquire(remaining)) {
                    rollbackPrior(strategies, i);
                    throw exceptionFor(strategies.get(i));
                }
            } catch (InterruptedException e) {
                rollbackPrior(strategies, i);
                Thread.currentThread().interrupt();
                throw new RuntimeException(
                        "Interrupted during admission acquisition", e);
            }
        }
    }

    static void releaseAll(List<AdmissionStrategy> strategies) {
        for (int i = strategies.size() - 1; i >= 0; i--) {
            strategies.get(i).release();
        }
    }

    private static void rollbackPrior(List<AdmissionStrategy> strategies,
                                       int failedIndex) {
        for (int j = failedIndex - 1; j >= 0; j--) {
            strategies.get(j).rollback();
        }
    }

    private static RuntimeException exceptionFor(AdmissionStrategy strategy) {
        if (strategy instanceof ConcurrencyStrategy) {
            return new AgentSessionLimitException(0);
        }
        return new AgentRateLimitException(0);
    }
}
```

- [ ] **Step 6: Refactor GatedAgentProviderTest to use new config/constructors**

Update the factory helpers to build the provider with strategy lists instead of raw parameters. The test assertions remain the same — only the setup changes. Replace `createGated(...)` methods:

```java
private GatedAgentProvider createGated(AgentProvider delegate,
                                       int maxConcurrent,
                                       double permitsPerSecond,
                                       int burstCapacity) {
    return createGated(delegate, maxConcurrent, permitsPerSecond,
            burstCapacity, Duration.ofSeconds(30), Duration.ofSeconds(5));
}

private GatedAgentProvider createGated(AgentProvider delegate,
                                       int maxConcurrent,
                                       double permitsPerSecond,
                                       int burstCapacity,
                                       Duration acquireTimeout,
                                       Duration queryAcquireTimeout) {
    var strategies = new java.util.ArrayList<AdmissionStrategy>();
    if (permitsPerSecond > 0) {
        int burst = burstCapacity > 0 ? burstCapacity
                : (int) Math.ceil(permitsPerSecond);
        strategies.add(new TokenBucketStrategy(permitsPerSecond, burst));
    }
    if (maxConcurrent > 0) {
        strategies.add(new ConcurrencyStrategy(maxConcurrent));
    }
    return new GatedAgentProvider(delegate, strategies,
            acquireTimeout, queryAcquireTimeout);
}
```

- [ ] **Step 7: Run all agent-gate tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add agent-gate/src
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#30): refactor GatedAgentProvider to composable AdmissionStrategy list

Config moves to nested groups. Acquisition ordering: cheapest-to-reject
first. Rollback on partial acquisition failure.

Refs casehubio/examples#30"
```

---

### Task 5: GatedAgentSession refactor

**Files:**
- Modify: `platform/agent-gate/src/main/java/io/casehub/platform/agent/gate/GatedAgentSession.java`
- Modify: `platform/agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentSessionTest.java`

**Interfaces:**
- Consumes: `AdmissionStrategy`, `GatedAgentProvider.acquireAll()`, `GatedAgentProvider.releaseAll()` (Task 4)

- [ ] **Step 1: Refactor GatedAgentSession to use partitioned strategy lists**

```java
package io.casehub.platform.agent.gate;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentSession;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;

import java.time.Duration;
import java.util.List;

final class GatedAgentSession implements AgentSession {

    private final AgentSession delegate;
    private final List<AdmissionStrategy> sessionStrategies;
    private final List<AdmissionStrategy> queryStrategies;
    private final Duration queryAcquireTimeout;

    GatedAgentSession(AgentSession delegate,
                      List<AdmissionStrategy> sessionStrategies,
                      List<AdmissionStrategy> queryStrategies,
                      Duration queryAcquireTimeout) {
        this.delegate = delegate;
        this.sessionStrategies = sessionStrategies;
        this.queryStrategies = queryStrategies;
        this.queryAcquireTimeout = queryAcquireTimeout;
    }

    @Override
    public Multi<AgentEvent> query(String prompt) {
        if (!queryStrategies.isEmpty()) {
            GatedAgentProvider.acquireAll(queryStrategies, queryAcquireTimeout);
        }
        return delegate.query(prompt);
    }

    @Override
    public Uni<Void> interrupt() {
        return delegate.interrupt();
    }

    @Override
    public void close(Duration maxWait) {
        try {
            delegate.close(maxWait);
        } finally {
            GatedAgentProvider.releaseAll(sessionStrategies);
        }
    }

    @Override
    public void close() {
        close(Duration.ofSeconds(30));
    }
}
```

- [ ] **Step 2: Refactor GatedAgentSessionTest**

Update test factory methods to construct with strategy lists instead of raw `TokenBucket`/`Semaphore`. Key mapping:
- `rateLimitActive=true` → add `TokenBucketStrategy` to queryStrategies
- `concurrencyActive=true` → add `ConcurrencyStrategy` to sessionStrategies

```java
private static GatedAgentSession sessionWith(AgentSession delegate,
                                              TokenBucketStrategy tokenBucket,
                                              ConcurrencyStrategy concurrency,
                                              Duration queryTimeout) {
    var sessionStrategies = concurrency != null
            ? java.util.List.<AdmissionStrategy>of(concurrency) : java.util.List.<AdmissionStrategy>of();
    var queryStrategies = tokenBucket != null
            ? java.util.List.<AdmissionStrategy>of(tokenBucket) : java.util.List.<AdmissionStrategy>of();
    return new GatedAgentSession(delegate, sessionStrategies, queryStrategies, queryTimeout);
}
```

Update each test to use the new factory. Assertions remain the same.

- [ ] **Step 3: Run all agent-gate tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl agent-gate -f /Users/mdproctor/claude/casehub/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/platform add agent-gate/src
git -C /Users/mdproctor/claude/casehub/platform commit -m "feat(#30): refactor GatedAgentSession — partitioned session/query strategies

Session strategies (concurrency) acquired at open, released at close.
Query strategies (rate limit) acquired per query.

Refs casehubio/examples#30"
```

---

### Task 6: Wacky-manor migration

**Files:**
- Modify: `examples/wacky-manor/pom.xml` — add `casehub-platform-agent-gate` dependency
- Modify: `examples/wacky-manor/src/main/resources/application.properties` — add gate config
- Delete: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/GatedAgentProvider.java`
- Modify: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — remove manual wrapping

**Interfaces:**
- Consumes: Platform `agent-gate` CDI decorator (Tasks 1–5)

- [ ] **Step 1: Add agent-gate dependency to wacky-manor pom.xml**

Add to the `<dependencies>` section:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-gate</artifactId>
</dependency>
```

- [ ] **Step 2: Add gate config to application.properties**

Add:

```properties
casehub.platform.agent.gate.concurrency.max=${manor.agent.max-concurrent:5}
casehub.platform.agent.gate.acquire-timeout=PT120S
```

This reuses the existing `manor.agent.max-concurrent` config property.

- [ ] **Step 3: Remove manual GatedAgentProvider wrapping from ScenarioOrchestrator**

In `ScenarioOrchestrator.java`:
- Remove the `private volatile AgentProvider gatedProvider;` field
- Remove line 86: `this.gatedProvider = new GatedAgentProvider(agentProvider, maxConcurrentAgents, java.time.Duration.ofSeconds(120));`
- Replace all usages of `gatedProvider` with `agentProvider` (the injected field — CDI decorator handles gating)
- Remove the `manor.agent.max-concurrent` config property injection (line 59-60) since it's now in platform config

- [ ] **Step 4: Delete the local GatedAgentProvider**

Use `ide_refactor_safe_delete` to remove `io.casehub.examples.manor.agent.GatedAgentProvider`.

- [ ] **Step 5: Build and run wacky-manor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -f /Users/mdproctor/claude/casehub/slots/111/examples/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/111/examples add wacky-manor/pom.xml wacky-manor/src
git -C /Users/mdproctor/claude/casehub/slots/111/examples commit -m "feat(#30): migrate wacky-manor to platform agent-gate

Deletes local GatedAgentProvider. CDI decorator handles concurrency
gating transparently.

Refs #30"
```

---

### Task 7: Trellis migration

**Files:**
- Modify: `trellis/sidecar/pom.xml` — add `casehub-platform-agent-gate` dependency
- Modify: `trellis/sidecar/src/main/resources/application.properties` — add gate config
- Modify: `trellis/sidecar/src/main/java/io/hortora/trellis/coordinator/ActionService.java` — remove sliding window code

**Interfaces:**
- Consumes: Platform `agent-gate` CDI decorator (Tasks 1–5)

- [ ] **Step 1: Add agent-gate dependency to trellis sidecar pom.xml**

Add to the `<dependencies>` section:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-gate</artifactId>
</dependency>
```

- [ ] **Step 2: Add gate config to application.properties**

Add:

```properties
casehub.platform.agent.gate.sliding-window.max-actions=5
casehub.platform.agent.gate.sliding-window.window-seconds=60
```

- [ ] **Step 3: Remove sliding window code from ActionService**

Remove from `ActionService.java`:
- The `autonomousTimestamps` field (likely a `Deque<Instant>`)
- `isWithinRateLimit()` method (lines 272-276)
- `recordAutonomousExecution()` method (lines 278-280)
- `pruneOldTimestamps()` method (lines 282-288)
- `resetRateLimit()` method (lines 290-292)
- All call sites of these methods — replace `isWithinRateLimit()` checks with unconditional proceed (the platform gate handles it)

- [ ] **Step 4: Build and run trellis tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl sidecar -f /Users/mdproctor/claude/hortora/trellis/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/hortora/trellis add sidecar/pom.xml sidecar/src
git -C /Users/mdproctor/claude/hortora/trellis commit -m "feat(#30): migrate to platform agent-gate sliding window

Removes bespoke sliding window from ActionService. Platform CDI
decorator handles rate limiting transparently.

Refs casehubio/examples#30"
```
