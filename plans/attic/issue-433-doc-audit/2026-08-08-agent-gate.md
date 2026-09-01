# Agent Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/examples#30 — Platform LLM rate limiter
**Issue group:** #30, #31, #32, #33

**Goal:** Add a CDI `@Decorator` module (`agent-gate/`) that transparently
wraps any `AgentProvider` with token bucket rate limiting and concurrency
gating.

**Architecture:** New optional module `casehub-platform-agent-gate` with
a CDI `@Decorator` that wraps whichever `AgentProvider` CDI resolves.
Token bucket (throughput) + Semaphore (concurrency) applied before
delegation. Deferred admission in `invoke()` via `Multi.createFrom().deferred()`.
Provider semaphores subsumed via config override when gate is present.

**Tech Stack:** Java 21, Quarkus 3.32 (ArC CDI), SmallRye Config,
Mutiny, JUnit 5, AssertJ

## Global Constraints

- `agent-api/` must remain zero-dependency beyond Mutiny — no Quarkus, no CDI
- `agent-gate/` is an optional Jandex library module — no `quarkus:build` goal
- All new types use `io.casehub.platform.agent` or `io.casehub.platform.agent.gate` packages
- Tests use JUnit 5 + AssertJ; plain Java unit tests except the CDI smoke test
- Every commit references `Refs casehubio/examples#30`

---

### Task 1: AgentRateLimitException in agent-api

**Files:**
- Create: `agent-api/src/main/java/io/casehub/platform/agent/AgentRateLimitException.java`
- Test: `agent-api/src/test/java/io/casehub/platform/agent/AgentRateLimitExceptionTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `AgentRateLimitException(double permitsPerSecond)` — extends `RuntimeException`, exposes `permitsPerSecond()` and `retryAfterMillis()`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.platform.agent;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AgentRateLimitExceptionTest {

    @Test
    void messageContainsRate() {
        var ex = new AgentRateLimitException(2.0);
        assertThat(ex.getMessage()).contains("2.0");
        assertThat(ex.getMessage()).contains("permits/sec");
    }

    @Test
    void retryAfterMillisComputedFromRate() {
        var ex = new AgentRateLimitException(2.0);
        assertThat(ex.retryAfterMillis()).isEqualTo(500L);
    }

    @Test
    void retryAfterMillisRoundsUp() {
        var ex = new AgentRateLimitException(3.0);
        assertThat(ex.retryAfterMillis()).isEqualTo(334L);
    }

    @Test
    void zeroRateDefaultsToOneSecond() {
        var ex = new AgentRateLimitException(0.0);
        assertThat(ex.retryAfterMillis()).isEqualTo(1000L);
    }

    @Test
    void permitsPerSecondExposed() {
        var ex = new AgentRateLimitException(5.0);
        assertThat(ex.permitsPerSecond()).isEqualTo(5.0);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl agent-api -Dtest=AgentRateLimitExceptionTest`
Expected: Compilation failure — class not found

- [ ] **Step 3: Write the implementation**

Create `agent-api/src/main/java/io/casehub/platform/agent/AgentRateLimitException.java`:

```java
package io.casehub.platform.agent;

public class AgentRateLimitException extends RuntimeException {

    private final double permitsPerSecond;
    private final long retryAfterMillis;

    public AgentRateLimitException(double permitsPerSecond) {
        super("Agent rate limit exceeded (" + permitsPerSecond + " permits/sec)");
        this.permitsPerSecond = permitsPerSecond;
        this.retryAfterMillis = permitsPerSecond > 0
                ? (long) Math.ceil(1000.0 / permitsPerSecond)
                : 1000L;
    }

    public double permitsPerSecond() {
        return permitsPerSecond;
    }

    public long retryAfterMillis() {
        return retryAfterMillis;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl agent-api -Dtest=AgentRateLimitExceptionTest`
Expected: All 5 tests PASS

- [ ] **Step 5: Commit**

```
feat(#30): add AgentRateLimitException to agent-api

Refs casehubio/examples#30
```

---

### Task 2: TokenBucket

**Files:**
- Create: `agent-gate/pom.xml`
- Create: `agent-gate/src/main/java/io/casehub/platform/agent/gate/TokenBucket.java`
- Create: `agent-gate/src/test/java/io/casehub/platform/agent/gate/TokenBucketTest.java`
- Modify: `pom.xml` (parent — add `<module>agent-gate</module>`)

**Interfaces:**
- Consumes: nothing
- Produces: `TokenBucket(double permitsPerSecond, int burstCapacity)` with `boolean tryAcquire(Duration timeout) throws InterruptedException`, `void release()`, `double availablePermits()`

- [ ] **Step 1: Create the module POM**

Create `agent-gate/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-platform-agent-gate</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform Agent Gate</name>
    <description>CDI @Decorator rate limiter for AgentProvider — token bucket + concurrency gate.
        Activates by classpath presence. No quarkus:build goal (library module).</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>

        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.awaitility</groupId>
            <artifactId>awaitility</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-code</goal>
                            <goal>generate-code-tests</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add module to parent POM**

Add `<module>agent-gate</module>` after `<module>preferences-editor</module>` in the parent `pom.xml` modules list.

- [ ] **Step 3: Create source directories**

```bash
mkdir -p agent-gate/src/main/java/io/casehub/platform/agent/gate
mkdir -p agent-gate/src/test/java/io/casehub/platform/agent/gate
mkdir -p agent-gate/src/main/resources/META-INF
```

- [ ] **Step 4: Write the failing tests**

Create `agent-gate/src/test/java/io/casehub/platform/agent/gate/TokenBucketTest.java`:

```java
package io.casehub.platform.agent.gate;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;

class TokenBucketTest {

    @Test
    void burstAllowsImmediateConsumption() throws Exception {
        var bucket = new TokenBucket(1.0, 3);
        assertThat(bucket.tryAcquire(Duration.ZERO)).isTrue();
        assertThat(bucket.tryAcquire(Duration.ZERO)).isTrue();
        assertThat(bucket.tryAcquire(Duration.ZERO)).isTrue();
        assertThat(bucket.tryAcquire(Duration.ZERO)).isFalse();
    }

    @Test
    void refillAddsTokensOverTime() throws Exception {
        var bucket = new TokenBucket(10.0, 1);
        assertThat(bucket.tryAcquire(Duration.ZERO)).isTrue();
        assertThat(bucket.tryAcquire(Duration.ZERO)).isFalse();
        Thread.sleep(150);
        assertThat(bucket.tryAcquire(Duration.ZERO)).isTrue();
    }

    @Test
    void tryAcquireBlocksUntilTokenAvailable() throws Exception {
        var bucket = new TokenBucket(10.0, 1);
        bucket.tryAcquire(Duration.ZERO);
        long start = System.nanoTime();
        assertThat(bucket.tryAcquire(Duration.ofMillis(500))).isTrue();
        long elapsed = (System.nanoTime() - start) / 1_000_000;
        assertThat(elapsed).isBetween(50L, 300L);
    }

    @Test
    void tryAcquireTimesOutWhenNoTokens() throws Exception {
        var bucket = new TokenBucket(0.5, 1);
        bucket.tryAcquire(Duration.ZERO);
        assertThat(bucket.tryAcquire(Duration.ofMillis(100))).isFalse();
    }

    @Test
    void releaseCreditsOneToken() throws Exception {
        var bucket = new TokenBucket(1.0, 2);
        bucket.tryAcquire(Duration.ZERO);
        bucket.tryAcquire(Duration.ZERO);
        assertThat(bucket.availablePermits()).isLessThan(1.0);
        bucket.release();
        assertThat(bucket.availablePermits()).isGreaterThanOrEqualTo(1.0);
    }

    @Test
    void releaseCappedAtBurstCapacity() throws Exception {
        var bucket = new TokenBucket(1.0, 2);
        bucket.release();
        bucket.release();
        assertThat(bucket.availablePermits()).isLessThanOrEqualTo(2.0);
    }

    @Test
    void releaseWakesBlockedWaiter() throws Exception {
        var bucket = new TokenBucket(0.1, 1);
        bucket.tryAcquire(Duration.ZERO);
        var acquired = new AtomicBoolean(false);
        var latch = new CountDownLatch(1);
        Thread.ofVirtual().start(() -> {
            try {
                acquired.set(bucket.tryAcquire(Duration.ofSeconds(5)));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            latch.countDown();
        });
        Thread.sleep(50);
        bucket.release();
        assertThat(latch.await(2, TimeUnit.SECONDS)).isTrue();
        assertThat(acquired.get()).isTrue();
    }

    @Test
    void interruptedThreadGetsInterruptedException() {
        var bucket = new TokenBucket(0.1, 1);
        bucket.tryAcquire(Duration.ZERO);
        var thread = Thread.ofVirtual().start(() -> {
            try {
                bucket.tryAcquire(Duration.ofSeconds(10));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
        thread.interrupt();
    }

    @Test
    void concurrentAccessRespectsThroughput() throws Exception {
        var bucket = new TokenBucket(20.0, 5);
        var acquired = new AtomicInteger();
        var done = new CountDownLatch(10);
        for (int i = 0; i < 10; i++) {
            Thread.ofVirtual().start(() -> {
                try {
                    if (bucket.tryAcquire(Duration.ofSeconds(2))) {
                        acquired.incrementAndGet();
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                done.countDown();
            });
        }
        assertThat(done.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(acquired.get()).isEqualTo(10);
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=TokenBucketTest`
Expected: Compilation failure — TokenBucket not found

- [ ] **Step 6: Write the implementation**

Create `agent-gate/src/main/java/io/casehub/platform/agent/gate/TokenBucket.java`:

```java
package io.casehub.platform.agent.gate;

import java.time.Duration;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

final class TokenBucket {

    private final ReentrantLock lock = new ReentrantLock(true);
    private final Condition tokensAvailable = lock.newCondition();
    private final double permitsPerSecond;
    private final double maxPermits;
    private double storedPermits;
    private long lastRefillNanos;

    TokenBucket(double permitsPerSecond, int burstCapacity) {
        if (permitsPerSecond <= 0) {
            throw new IllegalArgumentException("permitsPerSecond must be > 0");
        }
        if (burstCapacity <= 0) {
            throw new IllegalArgumentException("burstCapacity must be > 0");
        }
        this.permitsPerSecond = permitsPerSecond;
        this.maxPermits = burstCapacity;
        this.storedPermits = burstCapacity;
        this.lastRefillNanos = System.nanoTime();
    }

    boolean tryAcquire(Duration timeout) throws InterruptedException {
        long deadlineNanos = System.nanoTime() + timeout.toNanos();
        lock.lockInterruptibly();
        try {
            while (true) {
                refill();
                if (storedPermits >= 1.0) {
                    storedPermits -= 1.0;
                    tokensAvailable.signal();
                    return true;
                }
                long remainingNanos = deadlineNanos - System.nanoTime();
                if (remainingNanos <= 0) {
                    return false;
                }
                double deficit = 1.0 - storedPermits;
                long waitNanos = (long) Math.ceil(
                        deficit / permitsPerSecond * 1_000_000_000L);
                long actualWait = Math.min(waitNanos, remainingNanos);
                tokensAvailable.await(actualWait, java.util.concurrent.TimeUnit.NANOSECONDS);
            }
        } finally {
            lock.unlock();
        }
    }

    void release() {
        lock.lock();
        try {
            storedPermits = Math.min(maxPermits, storedPermits + 1.0);
            tokensAvailable.signal();
        } finally {
            lock.unlock();
        }
    }

    double availablePermits() {
        lock.lock();
        try {
            refill();
            return storedPermits;
        } finally {
            lock.unlock();
        }
    }

    private void refill() {
        long now = System.nanoTime();
        if (now > lastRefillNanos) {
            double elapsed = (now - lastRefillNanos) / 1_000_000_000.0;
            storedPermits = Math.min(maxPermits,
                    storedPermits + elapsed * permitsPerSecond);
            lastRefillNanos = now;
        }
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=TokenBucketTest`
Expected: All 9 tests PASS

- [ ] **Step 8: Commit**

```
feat(#30): add TokenBucket with Condition-based wakeup

Refs casehubio/examples#30
```

---

### Task 3: GatedAgentSession

**Files:**
- Create: `agent-gate/src/main/java/io/casehub/platform/agent/gate/GatedAgentSession.java`
- Create: `agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentSessionTest.java`

**Interfaces:**
- Consumes: `TokenBucket.tryAcquire(Duration)`, `TokenBucket.release()`, `AgentSession`, `Semaphore`
- Produces: `GatedAgentSession implements AgentSession` — wraps delegate session, per-query rate limiting, concurrency release on close

- [ ] **Step 1: Write the failing tests**

Create `agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentSessionTest.java`:

```java
package io.casehub.platform.agent.gate;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentRateLimitException;
import io.casehub.platform.agent.AgentSession;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.Semaphore;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GatedAgentSessionTest {

    @Test
    void queryConsumesTokenWhenRateLimitActive() throws Exception {
        var bucket = new TokenBucket(1.0, 1);
        var gate = new Semaphore(1, true);
        var session = new GatedAgentSession(
                stubSession("hello"), bucket, gate,
                Duration.ofSeconds(5), true, true);

        String result = collectText(session.query("prompt"));
        assertThat(result).isEqualTo("hello");
        assertThat(bucket.availablePermits()).isLessThan(1.0);
    }

    @Test
    void querySkipsTokenWhenRateLimitInactive() {
        var gate = new Semaphore(1, true);
        var session = new GatedAgentSession(
                stubSession("hello"), null, gate,
                Duration.ofSeconds(5), false, true);

        String result = collectText(session.query("prompt"));
        assertThat(result).isEqualTo("hello");
    }

    @Test
    void queryFailsWithRateLimitExceptionWhenBucketEmpty() throws Exception {
        var bucket = new TokenBucket(0.5, 1);
        bucket.tryAcquire(Duration.ZERO);
        var session = new GatedAgentSession(
                stubSession("hello"), bucket, null,
                Duration.ofMillis(50), true, false);

        assertThatThrownBy(() -> collectText(session.query("prompt")))
                .isInstanceOf(AgentRateLimitException.class);
    }

    @Test
    void closeReleasesConcurrencyPermit() {
        var gate = new Semaphore(1, true);
        gate.tryAcquire();
        assertThat(gate.availablePermits()).isEqualTo(0);
        var session = new GatedAgentSession(
                stubSession("x"), null, gate,
                Duration.ofSeconds(5), false, true);

        session.close();
        assertThat(gate.availablePermits()).isEqualTo(1);
    }

    @Test
    void closeReleasesPermitEvenWhenDelegateThrows() {
        var gate = new Semaphore(1, true);
        gate.tryAcquire();
        var failing = new StubSession() {
            @Override
            public void close(Duration maxWait) {
                throw new RuntimeException("close failed");
            }
        };
        var session = new GatedAgentSession(
                failing, null, gate,
                Duration.ofSeconds(5), false, true);

        try { session.close(); } catch (RuntimeException ignored) {}
        assertThat(gate.availablePermits()).isEqualTo(1);
    }

    @Test
    void closeSkipsGateReleaseWhenConcurrencyInactive() {
        var session = new GatedAgentSession(
                stubSession("x"), null, null,
                Duration.ofSeconds(5), false, false);
        session.close();
    }

    @Test
    void interruptDelegatesToSession() {
        var delegate = new StubSession();
        var session = new GatedAgentSession(
                delegate, null, null,
                Duration.ofSeconds(5), false, false);
        session.interrupt();
        assertThat(delegate.interrupted).isTrue();
    }

    // --- helpers ---

    private static String collectText(Multi<AgentEvent> multi) {
        return multi
                .filter(e -> e instanceof AgentEvent.TextDelta)
                .map(e -> ((AgentEvent.TextDelta) e).text())
                .collect().with(Collectors.joining())
                .await().atMost(Duration.ofSeconds(5));
    }

    private static AgentSession stubSession(String text) {
        return new StubSession() {
            @Override
            public Multi<AgentEvent> query(String prompt) {
                return Multi.createFrom().item(new AgentEvent.TextDelta(text));
            }
        };
    }

    private static class StubSession implements AgentSession {
        boolean interrupted = false;

        @Override
        public Multi<AgentEvent> query(String prompt) {
            return Multi.createFrom().empty();
        }

        @Override
        public Uni<Void> interrupt() {
            interrupted = true;
            return Uni.createFrom().voidItem();
        }

        @Override
        public void close(Duration maxWait) {}

        @Override
        public void close() { close(Duration.ofSeconds(30)); }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=GatedAgentSessionTest`
Expected: Compilation failure — GatedAgentSession not found

- [ ] **Step 3: Write the implementation**

Create `agent-gate/src/main/java/io/casehub/platform/agent/gate/GatedAgentSession.java`:

```java
package io.casehub.platform.agent.gate;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentRateLimitException;
import io.casehub.platform.agent.AgentSession;
import io.smallrye.mutiny.Multi;
import io.smallrye.mutiny.Uni;

import java.time.Duration;
import java.util.concurrent.Semaphore;

final class GatedAgentSession implements AgentSession {

    private final AgentSession delegate;
    private final TokenBucket tokenBucket;
    private final Semaphore concurrencyGate;
    private final Duration queryAcquireTimeout;
    private final boolean rateLimitActive;
    private final boolean concurrencyActive;

    GatedAgentSession(AgentSession delegate,
                      TokenBucket tokenBucket,
                      Semaphore concurrencyGate,
                      Duration queryAcquireTimeout,
                      boolean rateLimitActive,
                      boolean concurrencyActive) {
        this.delegate = delegate;
        this.tokenBucket = tokenBucket;
        this.concurrencyGate = concurrencyGate;
        this.queryAcquireTimeout = queryAcquireTimeout;
        this.rateLimitActive = rateLimitActive;
        this.concurrencyActive = concurrencyActive;
    }

    @Override
    public Multi<AgentEvent> query(String prompt) {
        if (rateLimitActive) {
            try {
                if (!tokenBucket.tryAcquire(queryAcquireTimeout)) {
                    return Multi.createFrom().failure(
                            new AgentRateLimitException(0));
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return Multi.createFrom().failure(
                        new RuntimeException("Interrupted during rate limit acquisition", e));
            }
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
            if (concurrencyActive) {
                concurrencyGate.release();
            }
        }
    }

    @Override
    public void close() {
        close(Duration.ofSeconds(30));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=GatedAgentSessionTest`
Expected: All 7 tests PASS

- [ ] **Step 5: Commit**

```
feat(#30): add GatedAgentSession with per-query rate limiting

Refs casehubio/examples#30
```

---

### Task 4: GatedAgentProvider decorator + AgentGateProperties

**Files:**
- Create: `agent-gate/src/main/java/io/casehub/platform/agent/gate/AgentGateProperties.java`
- Create: `agent-gate/src/main/java/io/casehub/platform/agent/gate/GatedAgentProvider.java`
- Create: `agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentProviderTest.java`
- Create: `agent-gate/src/main/resources/META-INF/microprofile-config.properties`

**Interfaces:**
- Consumes: `TokenBucket`, `GatedAgentSession`, `AgentGateProperties`, `AgentProvider`, `AgentRateLimitException`, `AgentSessionLimitException`
- Produces: CDI `@Decorator` wrapping `AgentProvider` — transparently gates `invoke()` and `openSession()`

- [ ] **Step 1: Create AgentGateProperties**

Create `agent-gate/src/main/java/io/casehub/platform/agent/gate/AgentGateProperties.java`:

```java
package io.casehub.platform.agent.gate;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;
import java.time.Duration;

@ConfigMapping(prefix = "casehub.platform.agent.gate")
public interface AgentGateProperties {

    @WithDefault("0")
    int maxConcurrent();

    @WithDefault("0.0")
    double permitsPerSecond();

    @WithDefault("0")
    int burstCapacity();

    @WithDefault("PT30S")
    Duration acquireTimeout();

    @WithDefault("PT5S")
    Duration queryAcquireTimeout();
}
```

- [ ] **Step 2: Create microprofile-config.properties**

Create `agent-gate/src/main/resources/META-INF/microprofile-config.properties`:

```properties
# Ordinal 200 — overrides provider defaults (ordinal 100) when gate module is on classpath.
# Deployers can re-enable provider semaphores with ordinal 300+.
config_ordinal=200
casehub.platform.agent.claude.max-concurrent-sessions=0
casehub.platform.agent.langchain4j.max-concurrent-sessions=0
```

- [ ] **Step 3: Write the failing tests**

Create `agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentProviderTest.java`:

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
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class GatedAgentProviderTest {

    @Test
    void passthroughWhenBothLimitsZero() {
        var delegate = stubProvider("hello");
        var gated = createGated(delegate, 0, 0.0, 0);

        String result = collectText(gated.invoke(config()));
        assertThat(result).isEqualTo("hello");
    }

    @Test
    void concurrencyOnlyBlocksExcessCalls() throws Exception {
        var holdFirst = new CountDownLatch(1);
        var firstStarted = new CountDownLatch(1);
        AgentProvider delayed = new StubProvider() {
            final AtomicInteger callOrder = new AtomicInteger();
            @Override
            public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                if (callOrder.incrementAndGet() == 1) {
                    return Multi.createFrom().emitter(em -> {
                        firstStarted.countDown();
                        try { holdFirst.await(); } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                        em.emit(new AgentEvent.TextDelta("first"));
                        em.complete();
                    });
                }
                return Multi.createFrom().item(new AgentEvent.TextDelta("second"));
            }
        };
        var gated = createGated(delayed, 1, 0.0, 0);
        var allDone = new CountDownLatch(2);
        var results = new java.util.concurrent.ConcurrentLinkedQueue<String>();

        Thread.ofVirtual().start(() -> {
            results.add(collectText(gated.invoke(config())));
            allDone.countDown();
        });
        assertThat(firstStarted.await(5, TimeUnit.SECONDS)).isTrue();

        Thread.ofVirtual().start(() -> {
            results.add(collectText(gated.invoke(config())));
            allDone.countDown();
        });

        holdFirst.countDown();
        assertThat(allDone.await(10, TimeUnit.SECONDS)).isTrue();
        assertThat(results).containsExactlyInAnyOrder("first", "second");
    }

    @Test
    void concurrencyTimeoutReturnsSessionLimitFailure() throws Exception {
        var holdForever = new CountDownLatch(1);
        AgentProvider slow = new StubProvider() {
            @Override
            public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                return Multi.createFrom().emitter(em -> {
                    try { holdForever.await(); } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    em.emit(new AgentEvent.TextDelta("done"));
                    em.complete();
                });
            }
        };
        var gated = createGated(slow, 1, 0.0, 0,
                Duration.ofMillis(200), Duration.ofSeconds(5));

        Thread.ofVirtual().start(() -> collectText(gated.invoke(config())));
        Thread.sleep(100);

        assertThatThrownBy(() -> collectText(gated.invoke(config())))
                .isInstanceOf(AgentSessionLimitException.class);
        holdForever.countDown();
    }

    @Test
    void rateLimitOnlyControlsThroughput() throws Exception {
        var delegate = stubProvider("ok");
        var gated = createGated(delegate, 0, 2.0, 2);

        assertThat(collectText(gated.invoke(config()))).isEqualTo("ok");
        assertThat(collectText(gated.invoke(config()))).isEqualTo("ok");
    }

    @Test
    void rateLimitTimeoutReturnsRateLimitFailure() {
        var delegate = stubProvider("ok");
        var gated = createGated(delegate, 0, 0.5, 1,
                Duration.ofMillis(100), Duration.ofSeconds(5));

        collectText(gated.invoke(config()));
        assertThatThrownBy(() -> collectText(gated.invoke(config())))
                .isInstanceOf(AgentRateLimitException.class);
    }

    @Test
    void permitReleasedOnStreamCompletion() {
        var delegate = stubProvider("ok");
        var gated = createGated(delegate, 1, 0.0, 0);

        collectText(gated.invoke(config()));
        collectText(gated.invoke(config()));
    }

    @Test
    void permitReleasedOnStreamFailure() {
        AgentProvider failing = new StubProvider() {
            @Override
            public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                return Multi.createFrom().failure(new RuntimeException("boom"));
            }
        };
        var gated = createGated(failing, 1, 0.0, 0);

        assertThatThrownBy(() -> collectText(gated.invoke(config())))
                .hasMessageContaining("boom");
        collectText(createGated(stubProvider("ok"), 1, 0.0, 0).invoke(config()));
    }

    @Test
    void permitReleasedOnSynchronousThrow() {
        AgentProvider exploding = new StubProvider() {
            @Override
            public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                throw new IllegalStateException("sync explosion");
            }
        };
        var gated = createGated(exploding, 1, 0.0, 0);

        assertThatThrownBy(() -> collectText(gated.invoke(config())))
                .hasMessageContaining("sync explosion");
    }

    @Test
    void tokenRefundedOnConcurrencyFailure() {
        var delegate = stubProvider("ok");
        var gated = createGated(delegate, 0, 1.0, 1,
                Duration.ofMillis(50), Duration.ofSeconds(5));

        collectText(gated.invoke(config()));
    }

    @Test
    void openSessionGatesConcurrency() {
        var delegate = new StubProvider() {
            @Override
            public AgentSession openSession(AgentSessionInit init) {
                return stubSession();
            }
        };
        var gated = createGated(delegate, 1, 0.0, 0);
        var session = gated.openSession(AgentSessionInit.of("sys"));
        session.close();
    }

    @Test
    void openSessionConcurrencyTimeoutThrows() throws Exception {
        var holdForever = new CountDownLatch(1);
        var delegate = new StubProvider() {
            @Override
            public AgentSession openSession(AgentSessionInit init) {
                return new StubAgentSession() {
                    @Override
                    public void close(Duration maxWait) {
                        holdForever.countDown();
                    }
                };
            }
        };
        var gated = createGated(delegate, 1, 0.0, 0,
                Duration.ofMillis(200), Duration.ofSeconds(5));
        var session = gated.openSession(AgentSessionInit.of("sys"));

        assertThatThrownBy(() ->
                gated.openSession(AgentSessionInit.of("sys")))
                .isInstanceOf(AgentSessionLimitException.class);
        session.close();
    }

    // --- factory helpers ---

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
        return new GatedAgentProvider(delegate, maxConcurrent,
                permitsPerSecond, burstCapacity, acquireTimeout,
                queryAcquireTimeout);
    }

    private static AgentSessionConfig config() {
        return AgentSessionConfig.of("system", "user");
    }

    private static String collectText(Multi<AgentEvent> multi) {
        return multi
                .filter(e -> e instanceof AgentEvent.TextDelta)
                .map(e -> ((AgentEvent.TextDelta) e).text())
                .collect().with(Collectors.joining())
                .await().atMost(Duration.ofSeconds(30));
    }

    private static AgentProvider stubProvider(String text) {
        return new StubProvider() {
            @Override
            public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                return Multi.createFrom().item(new AgentEvent.TextDelta(text));
            }
        };
    }

    private static AgentSession stubSession() {
        return new StubAgentSession();
    }

    private static abstract class StubProvider implements AgentProvider {
        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            return Multi.createFrom().empty();
        }
        @Override
        public AgentSession openSession(AgentSessionInit init) {
            throw new UnsupportedOperationException();
        }
    }

    private static class StubAgentSession implements AgentSession {
        @Override
        public Multi<AgentEvent> query(String prompt) {
            return Multi.createFrom().empty();
        }
        @Override
        public Uni<Void> interrupt() {
            return Uni.createFrom().voidItem();
        }
        @Override
        public void close(Duration maxWait) {}
        @Override
        public void close() { close(Duration.ofSeconds(30)); }
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=GatedAgentProviderTest`
Expected: Compilation failure — GatedAgentProvider not found (with test constructor)

- [ ] **Step 5: Write the implementation**

Create `agent-gate/src/main/java/io/casehub/platform/agent/gate/GatedAgentProvider.java`:

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
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

@Decorator
@Priority(Interceptor.Priority.APPLICATION)
public class GatedAgentProvider implements AgentProvider {

    @Inject @Delegate @Any AgentProvider delegate;
    @Inject AgentGateProperties config;

    private TokenBucket tokenBucket;
    private Semaphore concurrencyGate;
    private Duration acquireTimeout;
    private Duration queryAcquireTimeout;
    private boolean rateLimitActive;
    private boolean concurrencyActive;
    private boolean active;

    protected GatedAgentProvider() {}

    GatedAgentProvider(AgentProvider delegate, int maxConcurrent,
                       double permitsPerSecond, int burstCapacity,
                       Duration acquireTimeout, Duration queryAcquireTimeout) {
        this.delegate = delegate;
        this.acquireTimeout = acquireTimeout;
        this.queryAcquireTimeout = queryAcquireTimeout;
        this.rateLimitActive = permitsPerSecond > 0;
        this.concurrencyActive = maxConcurrent > 0;
        this.active = rateLimitActive || concurrencyActive;
        if (rateLimitActive) {
            int burst = burstCapacity > 0 ? burstCapacity
                    : (int) Math.ceil(permitsPerSecond);
            this.tokenBucket = new TokenBucket(permitsPerSecond, burst);
        }
        if (concurrencyActive) {
            this.concurrencyGate = new Semaphore(maxConcurrent, true);
        }
    }

    @PostConstruct
    void init() {
        if (config == null) return;
        int maxConcurrent = config.maxConcurrent();
        double permitsPerSecond = config.permitsPerSecond();
        int burstCapacity = config.burstCapacity();
        this.acquireTimeout = config.acquireTimeout();
        this.queryAcquireTimeout = config.queryAcquireTimeout();

        if (maxConcurrent < 0) {
            throw new IllegalStateException(
                    "casehub.platform.agent.gate.max-concurrent must be >= 0");
        }
        if (permitsPerSecond < 0) {
            throw new IllegalStateException(
                    "casehub.platform.agent.gate.permits-per-second must be >= 0");
        }
        if (burstCapacity > 0 && permitsPerSecond <= 0) {
            throw new IllegalStateException(
                    "casehub.platform.agent.gate.burst-capacity > 0 requires "
                    + "permits-per-second > 0");
        }

        this.rateLimitActive = permitsPerSecond > 0;
        this.concurrencyActive = maxConcurrent > 0;
        this.active = rateLimitActive || concurrencyActive;

        if (rateLimitActive) {
            int burst = burstCapacity > 0 ? burstCapacity
                    : (int) Math.ceil(permitsPerSecond);
            this.tokenBucket = new TokenBucket(permitsPerSecond, burst);
        }
        if (concurrencyActive) {
            this.concurrencyGate = new Semaphore(maxConcurrent, true);
        }
    }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        if (!active) {
            return delegate.invoke(config);
        }
        return Multi.createFrom().deferred(() -> {
            long deadlineNanos = System.nanoTime() + acquireTimeout.toNanos();

            if (rateLimitActive) {
                try {
                    Duration remaining = durationUntil(deadlineNanos);
                    if (!tokenBucket.tryAcquire(remaining)) {
                        throw new AgentRateLimitException(
                                this.config != null
                                    ? this.config.permitsPerSecond()
                                    : 0);
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(
                            "Interrupted during rate limit acquisition", e);
                }
            }

            if (concurrencyActive) {
                try {
                    Duration remaining = durationUntil(deadlineNanos);
                    if (!concurrencyGate.tryAcquire(
                            remaining.toMillis(), TimeUnit.MILLISECONDS)) {
                        if (rateLimitActive) tokenBucket.release();
                        throw new AgentSessionLimitException(
                                this.config != null
                                    ? this.config.maxConcurrent()
                                    : 0);
                    }
                } catch (InterruptedException e) {
                    if (rateLimitActive) tokenBucket.release();
                    Thread.currentThread().interrupt();
                    throw new RuntimeException(
                            "Interrupted during concurrency acquisition", e);
                }
            }

            try {
                Multi<AgentEvent> result = delegate.invoke(config);
                if (concurrencyActive) {
                    result = result.onTermination()
                            .invoke(() -> concurrencyGate.release());
                }
                return result;
            } catch (Exception e) {
                if (concurrencyActive) concurrencyGate.release();
                throw e;
            }
        }).runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        if (!active) {
            return delegate.openSession(init);
        }
        long deadlineNanos = System.nanoTime() + acquireTimeout.toNanos();

        if (rateLimitActive) {
            try {
                Duration remaining = durationUntil(deadlineNanos);
                if (!tokenBucket.tryAcquire(remaining)) {
                    throw new AgentRateLimitException(
                            config != null ? config.permitsPerSecond() : 0);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(
                        "Interrupted during rate limit acquisition", e);
            }
        }

        if (concurrencyActive) {
            try {
                Duration remaining = durationUntil(deadlineNanos);
                if (!concurrencyGate.tryAcquire(
                        remaining.toMillis(), TimeUnit.MILLISECONDS)) {
                    if (rateLimitActive) tokenBucket.release();
                    throw new AgentSessionLimitException(
                            config != null ? config.maxConcurrent() : 0);
                }
            } catch (InterruptedException e) {
                if (rateLimitActive) tokenBucket.release();
                Thread.currentThread().interrupt();
                throw new RuntimeException(
                        "Interrupted during concurrency acquisition", e);
            }
        }

        try {
            AgentSession session = delegate.openSession(init);
            return new GatedAgentSession(session, tokenBucket,
                    concurrencyGate, queryAcquireTimeout,
                    rateLimitActive, concurrencyActive);
        } catch (Exception e) {
            if (concurrencyActive) concurrencyGate.release();
            throw e;
        }
    }

    private static Duration durationUntil(long deadlineNanos) {
        long remaining = deadlineNanos - System.nanoTime();
        return remaining > 0 ? Duration.ofNanos(remaining) : Duration.ZERO;
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=GatedAgentProviderTest`
Expected: All 11 tests PASS

- [ ] **Step 7: Commit**

```
feat(#30): add GatedAgentProvider CDI @Decorator with token bucket + concurrency gate

Refs casehubio/examples#30
```

---

### Task 5: Provider semaphore subsumption

**Files:**
- Modify: `agent-claude/src/main/java/io/casehub/platform/agent/claude/ClaudeAgentClient.java` — change validation from `< 1` to `< 0`, `== 0` creates `Semaphore(Integer.MAX_VALUE)`
- Modify: `agent-langchain4j/src/main/java/io/casehub/platform/agent/langchain4j/ChatModelAgentProvider.java` — same change
- Modify: `agent-claude/src/test/java/io/casehub/platform/agent/claude/ClaudeAgentClientTest.java` — update zero-validation test
- Modify: `agent-langchain4j/src/test/java/io/casehub/platform/agent/langchain4j/ChatModelAgentProviderTest.java` — update zero-validation test

**Interfaces:**
- Consumes: existing `ClaudeAgentProperties.maxConcurrentSessions()`, `AgentLangchain4jProperties.maxConcurrentSessions()`
- Produces: `maxConcurrentSessions=0` now creates `Semaphore(Integer.MAX_VALUE)` instead of throwing

- [ ] **Step 1: Update ClaudeAgentClient validation**

In `ClaudeAgentClient.java`, both the `@Inject` constructor (line 69) and the test constructor (line 101) have identical validation. Change both from:

```java
if (maxSessions < 1) {
    throw new IllegalStateException(
        "casehub.platform.agent.claude.max-concurrent-sessions must be >= 1, got " + maxSessions);
}
this.semaphore = new Semaphore(maxSessions);
```

to:

```java
if (maxSessions < 0) {
    throw new IllegalStateException(
        "casehub.platform.agent.claude.max-concurrent-sessions must be >= 0, got " + maxSessions);
}
this.semaphore = new Semaphore(maxSessions == 0 ? Integer.MAX_VALUE : maxSessions);
```

- [ ] **Step 2: Update ChatModelAgentProvider validation**

In `ChatModelAgentProvider.java` `init()` method (line 49), change from:

```java
if (maxSessions < 1) {
    throw new IllegalStateException(
        "casehub.platform.agent.langchain4j.max-concurrent-sessions must be >= 1, got " + maxSessions);
}
semaphore = new Semaphore(maxSessions);
```

to:

```java
if (maxSessions < 0) {
    throw new IllegalStateException(
        "casehub.platform.agent.langchain4j.max-concurrent-sessions must be >= 0, got " + maxSessions);
}
semaphore = new Semaphore(maxSessions == 0 ? Integer.MAX_VALUE : maxSessions);
```

- [ ] **Step 3: Update ClaudeAgentClientTest**

In `ClaudeAgentClientTest.java`, update the test at line 136:

Change `constructor_zeroMaxConcurrentSessions_throwsIllegalState` to:

```java
@Test
void constructor_zeroMaxConcurrentSessions_createsAlwaysPermitSemaphore() {
    var client = new ClaudeAgentClient(props(0), config -> Multi.createFrom().empty());
    assertThat(client.availablePermits()).isEqualTo(Integer.MAX_VALUE);
}
```

Keep the negative test unchanged (just update message assertion from `>= 1` to `>= 0`).

- [ ] **Step 4: Update ChatModelAgentProviderTest**

In `ChatModelAgentProviderTest.java`, update the test at line 308:

Change `init_withZeroMaxConcurrentSessions_throws` to:

```java
@Test
void init_withZeroMaxConcurrentSessions_createsAlwaysPermitSemaphore() {
    when(properties.maxConcurrentSessions()).thenReturn(0);
    provider.init();
    assertThat(provider.availablePermits()).isEqualTo(Integer.MAX_VALUE);
}
```

Update negative test message assertion from `>= 1` to `>= 0`.

- [ ] **Step 5: Run all affected tests**

Run: `mvn --batch-mode test -pl agent-claude,agent-langchain4j`
Expected: All tests PASS

- [ ] **Step 6: Commit**

```
feat(#30): accept max-concurrent-sessions=0 in Claude and LangChain4j providers

Gate module ships config override that disables provider semaphores when active.
Zero creates Semaphore(Integer.MAX_VALUE) — always-permit, zero downstream changes.

Refs casehubio/examples#30
```

---

### Task 6: AgentProvider Javadoc update + full build verification

**Files:**
- Modify: `agent-api/src/main/java/io/casehub/platform/agent/AgentProvider.java` — update `openSession()` Javadoc

**Interfaces:**
- Consumes: nothing new
- Produces: updated Javadoc removing "immediately" timing guarantee

- [ ] **Step 1: Update AgentProvider.openSession() Javadoc**

In `AgentProvider.java` line 47, change the `@throws` clause from:

```java
@throws AgentSessionLimitException immediately (not via onFailure) if the concurrent-session
        cap is reached
```

to:

```java
@throws AgentSessionLimitException (not via onFailure) if concurrent-session
        admission fails — may block up to the configured admission timeout
        when a gate decorator is present
```

- [ ] **Step 2: Run the full build**

Run: `mvn --batch-mode install`
Expected: All modules compile and all tests pass, including the new `agent-gate` module

- [ ] **Step 3: Commit**

```
docs(#30): update AgentProvider.openSession() Javadoc for gate admission semantics

Refs casehubio/examples#30
```

---

### Task 7: CDI integration smoke test

**Files:**
- Create: `agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentProviderCdiTest.java`
- Create: `agent-gate/src/test/resources/application.properties`

**Interfaces:**
- Consumes: CDI runtime, `GatedAgentProvider`, `AgentGateProperties`, stub `AgentProvider`
- Produces: verification that `@Decorator` activates correctly via CDI

- [ ] **Step 1: Create test application.properties**

Create `agent-gate/src/test/resources/application.properties`:

```properties
casehub.platform.agent.gate.max-concurrent=1
casehub.platform.agent.gate.permits-per-second=100.0
casehub.platform.agent.gate.acquire-timeout=PT1S
casehub.platform.agent.gate.query-acquire-timeout=PT1S
```

- [ ] **Step 2: Write the CDI test**

Create `agent-gate/src/test/java/io/casehub/platform/agent/gate/GatedAgentProviderCdiTest.java`:

```java
package io.casehub.platform.agent.gate;

import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSession;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.agent.AgentSessionInit;
import io.casehub.platform.agent.AgentSessionLimitException;
import io.quarkus.test.junit.QuarkusTest;
import io.smallrye.mutiny.Multi;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class GatedAgentProviderCdiTest {

    @Inject
    AgentProvider provider;

    @Test
    void decoratorIsActiveAndGatesConcurrency() throws Exception {
        var hold = new CountDownLatch(1);
        var started = new CountDownLatch(1);

        Thread.ofVirtual().start(() -> {
            provider.invoke(AgentSessionConfig.of("sys", "hold")).subscribe().with(
                    e -> { started.countDown(); try { hold.await(); } catch (InterruptedException ignored) {} },
                    t -> started.countDown()
            );
        });

        Thread.sleep(200);

        assertThatThrownBy(() ->
                provider.invoke(AgentSessionConfig.of("sys", "second"))
                        .collect().with(Collectors.joining())
                        .await().atMost(Duration.ofSeconds(3)))
                .hasCauseInstanceOf(AgentSessionLimitException.class);

        hold.countDown();
    }

    @Alternative
    @Priority(1)
    @ApplicationScoped
    public static class TestAgentProvider implements AgentProvider {
        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            return Multi.createFrom().item(
                    new AgentEvent.TextDelta(config.userPrompt()));
        }

        @Override
        public AgentSession openSession(AgentSessionInit init) {
            throw new UnsupportedOperationException();
        }
    }
}
```

- [ ] **Step 3: Run the CDI test**

Run: `mvn --batch-mode test -pl agent-gate -Dtest=GatedAgentProviderCdiTest`
Expected: Test PASSES — decorator activates, gates concurrency at 1

- [ ] **Step 4: Commit**

```
test(#30): add CDI integration smoke test for GatedAgentProvider decorator

Refs casehubio/examples#30
```

---

### Task 8: Documentation — ARC42STORIES + CLAUDE.md

**Files:**
- Modify: `ARC42STORIES.MD` — add agent-gate to §5 (L8 container) and §7 (deployment table)
- Modify: `CLAUDE.md` — add agent-gate module entry

**Interfaces:**
- Consumes: nothing
- Produces: updated documentation reflecting the new module

- [ ] **Step 1: Update ARC42STORIES.MD**

Add `agent-gate/` to the L8: Agent Infrastructure section in §5 (building block view),
listing it alongside `agent-claude/` and `agent-langchain4j/`. Add deployment entry
in §7.

- [ ] **Step 2: Update CLAUDE.md**

Add the `agent-gate/` module entry to the Modules table:

```
| `agent-gate/` | `casehub-platform-agent-gate` | CDI @Decorator rate limiter for AgentProvider — token bucket (throughput) + Semaphore (concurrency). @Priority(APPLICATION = 2000) wraps any AgentProvider. Deferred admission via Multi.createFrom().deferred() + runSubscriptionOn(workerPool). Per-query rate limiting in GatedAgentSession. Config: casehub.platform.agent.gate.{max-concurrent, permits-per-second, burst-capacity, acquire-timeout, query-acquire-timeout}. Ships ordinal-200 config override disabling provider semaphores. No quarkus:build goal |
```

Update the Package Structure section to add:
```
  .gate          — AgentGateProperties (@ConfigMapping), TokenBucket, GatedAgentProvider (@Decorator), GatedAgentSession
```

Add `AgentRateLimitException` to the existing agent exception list in the package structure.

- [ ] **Step 3: Commit**

```
docs(#30): add agent-gate to ARC42STORIES and CLAUDE.md

Refs casehubio/examples#30
```

- [ ] **Step 4: Run final full build**

Run: `mvn --batch-mode install`
Expected: All modules compile and all tests pass
