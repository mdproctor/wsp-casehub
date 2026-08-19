# InnerLife Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #119 — InnerLife pattern — background thought loop with proactive initiation
**Issue group:** #119, #126

**Goal:** Build a background thought loop that enables agents to initiate conversations proactively via a three-stage evaluation pipeline (civility → content quality → LLM motivation scoring).

**Architecture:** InnerLifeOrchestrator composes ReflectionOrchestrator (neocortex) and AgentProvider (platform) into a tick()-based pipeline gated by composable CivilityConstraint SPI and a novelty-based content quality gate. Produces sealed InnerLifeTick (Silent/Initiated) for consumer dispatch.

**Tech Stack:** Java 21, JUnit 5, Mockito, CDI (jakarta.inject/enterprise), neocortex-memory-api, platform-agent-api

## Global Constraints

- All types in `io.casehub.blocks.agentic.personality` (same package as PersonalityEvolution)
- No Quarkus runtime — plain JUnit 5 + Mockito tests
- No CDI container in tests — construct beans manually
- Follow established patterns: sealed return types, @FunctionalInterface SPIs, tick() CDI beans
- AgentProvider.invoke() → collect TextDelta → parse JSON (same pattern as RoutingSupport, LlmDecomposition)

---

## Batch 1: Foundation types — SPI, records, sealed interfaces, novelty scorer

### Task 1: Create CivilityConstraint SPI, CivilityCheck, InitiationContext, InnerLifeTick, InnerLifeConfig, ContentQualityGate, TokenJaccardDistance

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/CivilityConstraint.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/CivilityCheck.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/InitiationContext.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/InnerLifeTick.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/InnerLifeConfig.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/ContentQualityGate.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/TokenJaccardDistance.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/InnerLifeTickTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/TokenJaccardDistanceTest.java`

**Interfaces:**
- Produces: `CivilityConstraint` — `@FunctionalInterface`: `permitInitiation(InitiationContext) → CivilityCheck`
- Produces: `CivilityCheck` — sealed: `Permitted`, `Denied(String reason)`
- Produces: `InitiationContext(Instant lastInitiationTimestamp, int initiationsInWindow, int consecutiveInitiationsWithoutResponse, AgentDescriptor descriptor)`
- Produces: `InnerLifeTick` — sealed: `Silent(@Nullable String reason)`, `Initiated(String content, @Nullable String channelHint, double motivationScore)`
- Produces: `InnerLifeConfig(double motivationThreshold, double noveltyThreshold, int minObservations, Duration quietPeriodBypass, int maxReflectionSources, int maxObservationsInPrompt, Duration windowDuration, Duration evictionTimeout)` with `defaults()`
- Produces: `ContentQualityGate(double noveltyThreshold, int minObservations, Duration quietPeriodBypass)` with `defaults()`
- Produces: `TokenJaccardDistance.distance(String a, String b) → double` — package-private

- [ ] **Step 1: Write InnerLifeTick test**

```java
package io.casehub.blocks.agentic.personality;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class InnerLifeTickTest {

    @Test
    void silentWithReason() {
        var tick = new InnerLifeTick.Silent("civility denied");
        assertThat(tick.reason()).isEqualTo("civility denied");
    }

    @Test
    void silentWithNullReason() {
        var tick = new InnerLifeTick.Silent(null);
        assertThat(tick.reason()).isNull();
    }

    @Test
    void initiatedCarriesAllFields() {
        var tick = new InnerLifeTick.Initiated("Hello!", "#general", 0.85);
        assertThat(tick.content()).isEqualTo("Hello!");
        assertThat(tick.channelHint()).isEqualTo("#general");
        assertThat(tick.motivationScore()).isEqualTo(0.85);
    }

    @Test
    void initiatedWithNullChannelHint() {
        var tick = new InnerLifeTick.Initiated("Hi", null, 0.7);
        assertThat(tick.channelHint()).isNull();
    }

    @Test
    void exhaustiveSwitchCoversAllVariants() {
        List<InnerLifeTick> ticks = List.of(
                new InnerLifeTick.Silent("quiet"),
                new InnerLifeTick.Initiated("msg", "#ch", 0.9));
        for (InnerLifeTick t : ticks) {
            String label = switch (t) {
                case InnerLifeTick.Silent s -> "silent";
                case InnerLifeTick.Initiated i -> "initiated";
            };
            assertThat(label).isNotBlank();
        }
    }
}
```

- [ ] **Step 2: Write TokenJaccardDistance test**

```java
package io.casehub.blocks.agentic.personality;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

class TokenJaccardDistanceTest {

    @Test
    void identicalTextReturnsZero() {
        assertThat(TokenJaccardDistance.distance("hello world", "hello world")).isEqualTo(0.0);
    }

    @Test
    void completelyDisjointReturnsOne() {
        assertThat(TokenJaccardDistance.distance("hello world", "foo bar")).isEqualTo(1.0);
    }

    @Test
    void partialOverlap() {
        double d = TokenJaccardDistance.distance("hello world foo", "hello bar baz");
        // intersection: {hello} = 1, union: {hello, world, foo, bar, baz} = 5
        // similarity = 1/5 = 0.2, distance = 0.8
        assertThat(d).isCloseTo(0.8, within(0.001));
    }

    @Test
    void emptyStringsReturnZero() {
        assertThat(TokenJaccardDistance.distance("", "")).isEqualTo(0.0);
    }

    @Test
    void oneEmptyReturnsOne() {
        assertThat(TokenJaccardDistance.distance("hello", "")).isEqualTo(1.0);
        assertThat(TokenJaccardDistance.distance("", "hello")).isEqualTo(1.0);
    }

    @Test
    void caseInsensitive() {
        assertThat(TokenJaccardDistance.distance("Hello World", "hello world")).isEqualTo(0.0);
    }

    @Test
    void nullInputsTreatedAsEmpty() {
        assertThat(TokenJaccardDistance.distance(null, null)).isEqualTo(0.0);
        assertThat(TokenJaccardDistance.distance(null, "hello")).isEqualTo(1.0);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn --batch-mode test -pl blocks -Dtest="InnerLifeTickTest,TokenJaccardDistanceTest" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: FAIL — classes not found

- [ ] **Step 4: Create all foundation types**

Create `CivilityConstraint.java`:
```java
package io.casehub.blocks.agentic.personality;

@FunctionalInterface
public interface CivilityConstraint {
    CivilityCheck permitInitiation(InitiationContext context);
}
```

Create `CivilityCheck.java`:
```java
package io.casehub.blocks.agentic.personality;

public sealed interface CivilityCheck {
    record Permitted() implements CivilityCheck {}
    record Denied(String reason) implements CivilityCheck {}
}
```

Create `InitiationContext.java`:
```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import java.time.Instant;

public record InitiationContext(
        Instant lastInitiationTimestamp,
        int initiationsInWindow,
        int consecutiveInitiationsWithoutResponse,
        AgentDescriptor descriptor) {}
```

Create `InnerLifeTick.java`:
```java
package io.casehub.blocks.agentic.personality;

import org.jspecify.annotations.Nullable;

public sealed interface InnerLifeTick {
    record Silent(@Nullable String reason) implements InnerLifeTick {}
    record Initiated(String content, @Nullable String channelHint,
                     double motivationScore) implements InnerLifeTick {}
}
```

Create `ContentQualityGate.java`:
```java
package io.casehub.blocks.agentic.personality;

import java.time.Duration;

public record ContentQualityGate(double noveltyThreshold,
                                  int minObservations,
                                  Duration quietPeriodBypass) {
    public static final double DEFAULT_NOVELTY_THRESHOLD = 0.3;
    public static final int DEFAULT_MIN_OBSERVATIONS = 3;
    public static final Duration DEFAULT_QUIET_PERIOD = Duration.ofMinutes(30);

    public ContentQualityGate {
        if (noveltyThreshold < 0.0 || noveltyThreshold > 1.0) {
            throw new IllegalArgumentException("noveltyThreshold must be in [0.0, 1.0]");
        }
        if (minObservations < 0) {
            throw new IllegalArgumentException("minObservations must be >= 0");
        }
        if (quietPeriodBypass == null) {
            throw new IllegalArgumentException("quietPeriodBypass must not be null");
        }
    }

    public static ContentQualityGate defaults() {
        return new ContentQualityGate(DEFAULT_NOVELTY_THRESHOLD,
                DEFAULT_MIN_OBSERVATIONS, DEFAULT_QUIET_PERIOD);
    }
}
```

Create `InnerLifeConfig.java`:
```java
package io.casehub.blocks.agentic.personality;

import java.time.Duration;

public record InnerLifeConfig(
        double motivationThreshold,
        ContentQualityGate contentQualityGate,
        int maxReflectionSources,
        int maxObservationsInPrompt,
        Duration windowDuration,
        Duration evictionTimeout) {

    public static final double DEFAULT_MOTIVATION_THRESHOLD = 0.6;
    public static final int DEFAULT_MAX_REFLECTION_SOURCES = 10;
    public static final int DEFAULT_MAX_OBSERVATIONS_IN_PROMPT = 50;
    public static final Duration DEFAULT_WINDOW_DURATION = Duration.ofHours(1);
    public static final Duration DEFAULT_EVICTION_TIMEOUT = Duration.ofHours(24);

    public InnerLifeConfig {
        if (motivationThreshold < 0.0 || motivationThreshold > 1.0) {
            throw new IllegalArgumentException("motivationThreshold must be in [0.0, 1.0]");
        }
        if (contentQualityGate == null) {
            throw new IllegalArgumentException("contentQualityGate must not be null");
        }
        if (maxReflectionSources <= 0) {
            throw new IllegalArgumentException("maxReflectionSources must be positive");
        }
        if (maxObservationsInPrompt <= 0) {
            throw new IllegalArgumentException("maxObservationsInPrompt must be positive");
        }
    }

    public static InnerLifeConfig defaults() {
        return new InnerLifeConfig(
                DEFAULT_MOTIVATION_THRESHOLD,
                ContentQualityGate.defaults(),
                DEFAULT_MAX_REFLECTION_SOURCES,
                DEFAULT_MAX_OBSERVATIONS_IN_PROMPT,
                DEFAULT_WINDOW_DURATION,
                DEFAULT_EVICTION_TIMEOUT);
    }
}
```

Create `TokenJaccardDistance.java`:
```java
package io.casehub.blocks.agentic.personality;

import java.util.HashSet;
import java.util.Set;

final class TokenJaccardDistance {

    private TokenJaccardDistance() {}

    static double distance(String a, String b) {
        Set<String> tokensA = tokenize(a);
        Set<String> tokensB = tokenize(b);
        if (tokensA.isEmpty() && tokensB.isEmpty()) {
            return 0.0;
        }
        if (tokensA.isEmpty() || tokensB.isEmpty()) {
            return 1.0;
        }
        Set<String> intersection = new HashSet<>(tokensA);
        intersection.retainAll(tokensB);
        Set<String> union = new HashSet<>(tokensA);
        union.addAll(tokensB);
        return 1.0 - ((double) intersection.size() / union.size());
    }

    private static Set<String> tokenize(String text) {
        if (text == null || text.isBlank()) {
            return Set.of();
        }
        Set<String> tokens = new HashSet<>();
        for (String token : text.toLowerCase().split("\\s+")) {
            if (!token.isEmpty()) {
                tokens.add(token);
            }
        }
        return tokens;
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="InnerLifeTickTest,TokenJaccardDistanceTest" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/ blocks/src/test/java/io/casehub/blocks/agentic/personality/
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#119): add InnerLife foundation types — CivilityConstraint SPI, InnerLifeTick, ContentQualityGate, TokenJaccardDistance Refs #119"
```

---

## Batch 2: Default civility constraints

### Task 2: Implement MinimumGapConstraint, MaxPerWindowConstraint, ConsecutiveInitiationCooldownConstraint

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/MinimumGapConstraint.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/MaxPerWindowConstraint.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/ConsecutiveInitiationCooldownConstraint.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/CivilityConstraintTest.java`

**Interfaces:**
- Consumes: `CivilityConstraint`, `CivilityCheck`, `InitiationContext` (Task 1)
- Produces: Three default `CivilityConstraint` implementations

- [ ] **Step 1: Write CivilityConstraint test**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;

class CivilityConstraintTest {

    private static final AgentDescriptor DESCRIPTOR = mock(AgentDescriptor.class);

    // --- MinimumGapConstraint ---

    @Test
    void gapPermittedWhenEnoughTimeElapsed() {
        var constraint = new MinimumGapConstraint(Duration.ofMinutes(5));
        var ctx = new InitiationContext(
                Instant.now().minus(Duration.ofMinutes(10)), 0, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Permitted.class);
    }

    @Test
    void gapDeniedWhenTooSoon() {
        var constraint = new MinimumGapConstraint(Duration.ofMinutes(5));
        var ctx = new InitiationContext(
                Instant.now().minus(Duration.ofMinutes(2)), 0, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Denied.class);
    }

    @Test
    void gapPermittedOnFirstInitiation() {
        var constraint = new MinimumGapConstraint(Duration.ofMinutes(5));
        var ctx = new InitiationContext(Instant.EPOCH, 0, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Permitted.class);
    }

    // --- MaxPerWindowConstraint ---

    @Test
    void windowPermittedWhenUnderLimit() {
        var constraint = new MaxPerWindowConstraint(3);
        var ctx = new InitiationContext(Instant.now(), 2, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Permitted.class);
    }

    @Test
    void windowDeniedWhenAtLimit() {
        var constraint = new MaxPerWindowConstraint(3);
        var ctx = new InitiationContext(Instant.now(), 3, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Denied.class);
    }

    @Test
    void windowPermittedWhenZeroInitiations() {
        var constraint = new MaxPerWindowConstraint(3);
        var ctx = new InitiationContext(Instant.EPOCH, 0, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Permitted.class);
    }

    // --- ConsecutiveInitiationCooldownConstraint ---

    @Test
    void cooldownPermittedWhenUnderLimit() {
        var constraint = new ConsecutiveInitiationCooldownConstraint(2);
        var ctx = new InitiationContext(Instant.now(), 0, 1, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Permitted.class);
    }

    @Test
    void cooldownDeniedWhenAtLimit() {
        var constraint = new ConsecutiveInitiationCooldownConstraint(2);
        var ctx = new InitiationContext(Instant.now(), 0, 2, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Denied.class);
    }

    @Test
    void cooldownPermittedWhenZeroConsecutive() {
        var constraint = new ConsecutiveInitiationCooldownConstraint(2);
        var ctx = new InitiationContext(Instant.now(), 0, 0, DESCRIPTOR);
        assertThat(constraint.permitInitiation(ctx)).isInstanceOf(CivilityCheck.Permitted.class);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl blocks -Dtest=CivilityConstraintTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: FAIL

- [ ] **Step 3: Implement MinimumGapConstraint**

```java
package io.casehub.blocks.agentic.personality;

import java.time.Duration;
import java.time.Instant;

public class MinimumGapConstraint implements CivilityConstraint {

    private final Duration minimumGap;

    public MinimumGapConstraint(Duration minimumGap) {
        this.minimumGap = minimumGap;
    }

    @Override
    public CivilityCheck permitInitiation(InitiationContext context) {
        if (context.lastInitiationTimestamp().equals(Instant.EPOCH)) {
            return new CivilityCheck.Permitted();
        }
        Duration elapsed = Duration.between(context.lastInitiationTimestamp(), Instant.now());
        if (elapsed.compareTo(minimumGap) < 0) {
            return new CivilityCheck.Denied("minimum gap not met: " + elapsed.toSeconds() + "s < " + minimumGap.toSeconds() + "s");
        }
        return new CivilityCheck.Permitted();
    }
}
```

- [ ] **Step 4: Implement MaxPerWindowConstraint**

```java
package io.casehub.blocks.agentic.personality;

public class MaxPerWindowConstraint implements CivilityConstraint {

    private final int maxInitiations;

    public MaxPerWindowConstraint(int maxInitiations) {
        this.maxInitiations = maxInitiations;
    }

    @Override
    public CivilityCheck permitInitiation(InitiationContext context) {
        if (context.initiationsInWindow() >= maxInitiations) {
            return new CivilityCheck.Denied("rate limit exceeded: " + context.initiationsInWindow() + " >= " + maxInitiations);
        }
        return new CivilityCheck.Permitted();
    }
}
```

- [ ] **Step 5: Implement ConsecutiveInitiationCooldownConstraint**

```java
package io.casehub.blocks.agentic.personality;

public class ConsecutiveInitiationCooldownConstraint implements CivilityConstraint {

    private final int maxConsecutive;

    public ConsecutiveInitiationCooldownConstraint(int maxConsecutive) {
        this.maxConsecutive = maxConsecutive;
    }

    @Override
    public CivilityCheck permitInitiation(InitiationContext context) {
        if (context.consecutiveInitiationsWithoutResponse() >= maxConsecutive) {
            return new CivilityCheck.Denied("consecutive cooldown: " + context.consecutiveInitiationsWithoutResponse() + " unanswered initiations");
        }
        return new CivilityCheck.Permitted();
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn --batch-mode test -pl blocks -Dtest=CivilityConstraintTest -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/ blocks/src/test/java/io/casehub/blocks/agentic/personality/CivilityConstraintTest.java
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#119): add default civility constraints — MinimumGap, MaxPerWindow, ConsecutiveInitiationCooldown Refs #119"
```

---

## Batch 3: Orchestrator — observe, tick pipeline, per-agent state

### Task 3: Implement InnerLifeOrchestrator

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/InnerLifeOrchestrator.java`
- Create: `blocks/src/main/java/io/casehub/blocks/agentic/personality/MotivationAssessment.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/InnerLifeOrchestratorTest.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/personality/ObserveAndStateTest.java`

**Interfaces:**
- Consumes: All foundation types (Task 1), all civility constraints (Task 2)
- Consumes: `ReflectionOrchestrator` (neocortex), `AgentProvider` (platform-agent-api)
- Consumes: `AgentDescriptor`, `AgentDisposition` (eidos-api)
- Consumes: `LevelEvent` (blocks/summarisation)
- Produces: `InnerLifeOrchestrator` CDI bean with `observe(LevelEvent, AgentDescriptor)`, `observeResponse(AgentDescriptor)`, `tick(AgentDescriptor, String channelContext) → InnerLifeTick`

- [ ] **Step 1: Write orchestrator test — civility denied returns Silent**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.neocortex.memory.reflection.ReflectionOrchestrator;
import io.casehub.platform.agent.api.AgentProvider;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class InnerLifeOrchestratorTest {

    private ReflectionOrchestrator reflectionOrchestrator;
    private AgentProvider agentProvider;
    private AgentDescriptor descriptor;
    private InnerLifeOrchestrator orchestrator;

    @SuppressWarnings("unchecked")
    @BeforeEach
    void setUp() {
        reflectionOrchestrator = mock(ReflectionOrchestrator.class);
        agentProvider = mock(AgentProvider.class);
        descriptor = mock(AgentDescriptor.class);
        when(descriptor.agentId()).thenReturn("agent-1");
        when(descriptor.tenancyId()).thenReturn("tenant-1");
        when(descriptor.name()).thenReturn("TestAgent");
        when(descriptor.briefing()).thenReturn("A test agent");
        var disposition = mock(io.casehub.eidos.api.AgentDisposition.class);
        when(disposition.dispositionProfile()).thenReturn(List.of());
        when(descriptor.disposition()).thenReturn(disposition);

        // Default: civility denied (too soon — gap constraint with 5 min, no prior initiation but we'll test specific paths)
        CivilityConstraint alwaysDeny = ctx -> new CivilityCheck.Denied("test deny");
        Instance<CivilityConstraint> constraints = mock(Instance.class);
        when(constraints.stream()).thenReturn(java.util.stream.Stream.of(alwaysDeny));

        orchestrator = new InnerLifeOrchestrator(
                reflectionOrchestrator, agentProvider, constraints,
                InnerLifeConfig.defaults());
    }

    @Test
    void tickReturnsSilentWhenCivilityDenied() {
        var result = orchestrator.tick(descriptor, "channel context");
        assertThat(result).isInstanceOf(InnerLifeTick.Silent.class);
        assertThat(((InnerLifeTick.Silent) result).reason()).contains("test deny");
        verifyNoInteractions(reflectionOrchestrator);
        verifyNoInteractions(agentProvider);
    }
}
```

- [ ] **Step 2: Write orchestrator test — low novelty returns Silent**

Add to `InnerLifeOrchestratorTest`:
```java
    @Test
    void tickReturnsSilentWhenNoObservations() {
        // Override with permissive civility
        CivilityConstraint alwaysPermit = ctx -> new CivilityCheck.Permitted();
        @SuppressWarnings("unchecked")
        Instance<CivilityConstraint> constraints = mock(Instance.class);
        when(constraints.stream()).thenReturn(java.util.stream.Stream.of(alwaysPermit));

        var permissiveOrch = new InnerLifeOrchestrator(
                reflectionOrchestrator, agentProvider, constraints,
                InnerLifeConfig.defaults());

        var result = permissiveOrch.tick(descriptor, "context");
        assertThat(result).isInstanceOf(InnerLifeTick.Silent.class);
        verifyNoInteractions(reflectionOrchestrator);
    }
```

- [ ] **Step 3: Write orchestrator test — full pipeline returns Initiated**

This test requires mocking AgentProvider.invoke() to return a Multi<AgentEvent> with TextDelta events that form a JSON MotivationAssessment. The exact mocking depends on the AgentProvider API surface. Write the test after implementing enough of the orchestrator to see the API shape.

- [ ] **Step 4: Create MotivationAssessment**

```java
package io.casehub.blocks.agentic.personality;

record MotivationAssessment(double score, String content, String channelHint) {
    MotivationAssessment {
        if (score < 0.0 || score > 1.0) {
            throw new IllegalArgumentException("score must be in [0.0, 1.0]");
        }
    }
}
```

- [ ] **Step 5: Implement InnerLifeOrchestrator**

Create `blocks/src/main/java/io/casehub/blocks/agentic/personality/InnerLifeOrchestrator.java` with:
- Constructor: inject ReflectionOrchestrator, AgentProvider, Instance<CivilityConstraint>, InnerLifeConfig
- `observe(LevelEvent, AgentDescriptor)`: append to per-agent event buffer + raw text
- `observeResponse(AgentDescriptor)`: reset consecutiveWithoutResponse
- `tick(AgentDescriptor, String)`: full pipeline — snapshot buffers → civility → content quality → reflect → LLM score → threshold → output
- Per-agent state in ConcurrentHashMap with ReentrantLock for tick()

The implementation follows the spec's tick steps 0-6 and the prompt design. AgentProvider.invoke() follows the text-collection-and-parse pattern from RoutingSupport.

- [ ] **Step 6: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl blocks -Dtest="InnerLifeOrchestratorTest,ObserveAndStateTest" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 7: Write ObserveAndStateTest**

```java
package io.casehub.blocks.agentic.personality;

import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.LevelEvent;
import io.casehub.eidos.api.AgentDescriptor;
import io.casehub.neocortex.memory.reflection.ReflectionOrchestrator;
import io.casehub.platform.agent.api.AgentProvider;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ObserveAndStateTest {

    @SuppressWarnings("unchecked")
    private InnerLifeOrchestrator makeOrchestrator(CivilityConstraint... constraints) {
        Instance<CivilityConstraint> cdi = mock(Instance.class);
        when(cdi.stream()).thenReturn(java.util.stream.Stream.of(constraints));
        return new InnerLifeOrchestrator(
                mock(ReflectionOrchestrator.class),
                mock(AgentProvider.class),
                cdi, InnerLifeConfig.defaults());
    }

    private AgentDescriptor descriptor() {
        var d = mock(AgentDescriptor.class);
        when(d.agentId()).thenReturn("a");
        when(d.tenancyId()).thenReturn("t");
        return d;
    }

    @Test
    void observeAccumulatesEvents() {
        var orch = makeOrchestrator();
        var d = descriptor();
        var event = new LevelEvent<>("hello world", System.currentTimeMillis(), new EventLevel("L1", 1));
        orch.observe(event, d);
        orch.observe(event, d);
        // Two observations recorded — verified indirectly via content quality gate
        // (minObservations=3 by default, so 2 observations → Silent)
        var result = orch.tick(d, "ctx");
        assertThat(result).isInstanceOf(InnerLifeTick.Silent.class);
    }

    @Test
    void observeResponseResetsConsecutiveCount() {
        var orch = makeOrchestrator();
        var d = descriptor();
        // observeResponse should not throw when called before any initiation
        orch.observeResponse(d);
    }
}
```

- [ ] **Step 8: Run full personality test suite**

Run: `mvn --batch-mode test -pl blocks -Dtest="io.casehub.blocks.agentic.personality.*" -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: PASS

- [ ] **Step 9: Run full blocks build**

Run: `mvn --batch-mode install -pl blocks -f /Users/mdproctor/claude/casehub/slots/134/blocks/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 10: Update CLAUDE.md**

Add InnerLife to the personality sub-package description in CLAUDE.md Key Directories table.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/134/blocks add blocks/src/main/java/io/casehub/blocks/agentic/personality/ blocks/src/test/java/io/casehub/blocks/agentic/personality/ CLAUDE.md
git -C /Users/mdproctor/claude/casehub/slots/134/blocks commit -m "feat(#119): implement InnerLifeOrchestrator — background thought loop with civility, novelty, and LLM motivation scoring Refs #119"
```

---

## References

- `specs/issue-126-autonomous-agent-patterns/2026-08-18-inner-life-design.md` — design spec
- `specs/issue-126-autonomous-agent-patterns/119-decisions.md` — 6 design decisions (D1–D6)
- `docs/research/2026-08-16-autonomous-agent-patterns-landscape.md` — research doc §2.1, §2.8
- `ReflectionOrchestrator` (neocortex) — reflection generation
- `AgentProvider.invoke()` (platform-agent-api) — LLM invocation
- `PersonalityEvolutionOrchestrator` (blocks) — established tick() pattern
- `RoutingSupport.invokeAndCollect()` (blocks) — text-collection-and-parse pattern
- GitHub #119 — InnerLife pattern
- GitHub #126 — epic: autonomous agent patterns
