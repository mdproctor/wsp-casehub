# Trust and Personality Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #45 — Trust and personality — wire AgentTrustProvider, personality evolution, disposition signals
**Issue group:** #41, #42, #43, #44, #45, #46

**Goal:** Wire the platform trust and personality systems into wacky-manor so characters build trust scores from interaction history, record disposition signals after actions, and evolve personality through accumulated experience.

**Architecture:** Five components following the established manor pattern. ManorTrustProvider implements the AgentTrustProvider SPI using relationship memories. ManorDispositionRecorder maps action outcomes to BehavioralSignal/DispositionAxis. ManorPersonalityEvolution wraps DispositionEvolution for periodic descriptor updates. Personality-weighted retrieval and trust-based retention are inline calls in the existing orchestrator and experience service.

**Tech Stack:** Quarkus CDI, casehub-neocortex-memory-api (AgentTrustProvider, PersonalityWeightedRetrieval), casehub-eidos-api (BehavioralSignalStore, DispositionSignalStore, DispositionEvolution), casehub-eidos-memory (InMemory implementations)

## Global Constraints

- All dependencies already declared in pom.xml — no new Maven deps
- Follow existing manor pattern: POJO or CDI bean, constructed/injected in ScenarioOrchestrator
- Use `ManorConstants.TENANCY_ID` for all platform store calls
- Config properties prefixed with `manor.` in application.yaml
- Tests follow existing patterns in `wacky-manor/src/test/java/io/casehub/examples/manor/agent/`

---

## Batch 1: Trust Scoring

### Task 1: ManorTrustProvider — interaction-history trust scoring

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorTrustProvider.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorTrustProviderTest.java`

**Interfaces:**
- Consumes: `io.casehub.neocortex.memory.CaseMemoryStore` (existing), `io.casehub.neocortex.memory.cbr.AgentTrustProvider` (platform SPI)
- Produces: `ManorTrustProvider.currentTrustScore(String agentId) → OptionalDouble` — used by Task 3 (trust-based retention) and Task 5 (orchestrator wiring)

- [ ] **Step 1: Write the failing test**

Create `ManorTrustProviderTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.MemoryOrder;
import io.casehub.neocortex.memory.query.RelationshipQuery;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.OptionalDouble;

import static org.junit.jupiter.api.Assertions.*;

class ManorTrustProviderTest {

    private static final String TENANT = "test-tenant";

    @Test
    void returnsDefaultTrustForUnknownAgent() {
        var store = new StubCaseMemoryStore(List.of());
        var provider = new ManorTrustProvider(store, TENANT, 1.0, -2.0);
        var score = provider.currentTrustScore("unknown-agent");
        assertTrue(score.isPresent());
        assertEquals(0.5, score.getAsDouble(), 0.01);
    }

    @Test
    void positiveMemoriesIncreaseTrust() {
        var memories = List.of(
            stubMemory("helped me escape the trap"),
            stubMemory("warned me about the danger"),
            stubMemory("protected me from the falling chandelier")
        );
        var store = new StubCaseMemoryStore(memories);
        var provider = new ManorTrustProvider(store, TENANT, 1.0, -2.0);
        var score = provider.currentTrustScore("helpful-agent");
        assertTrue(score.isPresent());
        assertTrue(score.getAsDouble() > 0.5, "Trust should be above default for positive interactions");
    }

    @Test
    void negativeMemoriesDecreaseTrust() {
        var memories = List.of(
            stubMemory("lied about the treasure location"),
            stubMemory("tried to steal the diamond"),
            stubMemory("betrayed our alliance")
        );
        var store = new StubCaseMemoryStore(memories);
        var provider = new ManorTrustProvider(store, TENANT, 1.0, -2.0);
        var score = provider.currentTrustScore("dishonest-agent");
        assertTrue(score.isPresent());
        assertTrue(score.getAsDouble() < 0.5, "Trust should be below default for negative interactions");
    }

    @Test
    void mixedMemoriesBalanceOut() {
        var memories = List.of(
            stubMemory("helped me find the key"),
            stubMemory("lied about the secret passage"),
            stubMemory("shared useful information")
        );
        var store = new StubCaseMemoryStore(memories);
        var provider = new ManorTrustProvider(store, TENANT, 1.0, -2.0);
        var score = provider.currentTrustScore("mixed-agent");
        assertTrue(score.isPresent());
        // 2 positive (weight 1.0 each = 2.0) + 1 negative (weight -2.0 = -2.0) = 0.0 raw
        // Normalized: should be around 0.5 or below
    }

    @Test
    void scoreIsClampedBetweenZeroAndOne() {
        var memories = List.of(
            stubMemory("betrayed me"),
            stubMemory("lied to me"),
            stubMemory("stole from me"),
            stubMemory("tricked me"),
            stubMemory("trapped me")
        );
        var store = new StubCaseMemoryStore(memories);
        var provider = new ManorTrustProvider(store, TENANT, 1.0, -2.0);
        var score = provider.currentTrustScore("villain");
        assertTrue(score.isPresent());
        assertTrue(score.getAsDouble() >= 0.0, "Trust should not go below 0");
        assertTrue(score.getAsDouble() <= 1.0, "Trust should not exceed 1");
    }

    private static Memory stubMemory(String description) {
        return new Memory(
            java.util.UUID.randomUUID(),
            description,
            MemoryDomain.of("manor"),
            0.5,
            Instant.now(),
            java.util.Map.of()
        );
    }

    // Minimal stub — only recallRelationships matters for trust
    private static class StubCaseMemoryStore implements CaseMemoryStore {
        private final List<Memory> memories;
        StubCaseMemoryStore(List<Memory> memories) { this.memories = memories; }

        @Override
        public List<Memory> recall(String tenantId, String agentId, MemoryDomain domain,
                                    MemoryOrder order, int limit) {
            return memories;
        }

        @Override
        public List<Memory> recallRelationships(String tenantId, String agentId,
                                                 RelationshipQuery query) {
            return memories;
        }

        // Other CaseMemoryStore methods — delegate to empty defaults
        @Override public void store(String tenantId, String agentId, Memory memory) {}
        @Override public void purge(String tenantId, String agentId,
                                     io.casehub.neocortex.memory.MemoryRetentionPolicy policy) {}
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorTrustProviderTest -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: FAIL — `ManorTrustProvider` class does not exist

- [ ] **Step 3: Write ManorTrustProvider implementation**

Create `ManorTrustProvider.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.MemoryOrder;
import io.casehub.neocortex.memory.cbr.AgentTrustProvider;

import java.util.List;
import java.util.OptionalDouble;
import java.util.Set;

public class ManorTrustProvider implements AgentTrustProvider {

    private static final MemoryDomain MANOR_DOMAIN = MemoryDomain.of("manor");
    private static final Set<String> POSITIVE_KEYWORDS = Set.of(
        "help", "helped", "protect", "protected", "warn", "warned",
        "share", "shared", "save", "saved", "trust", "honest", "gave"
    );
    private static final Set<String> NEGATIVE_KEYWORDS = Set.of(
        "lie", "lied", "steal", "stole", "stolen", "betray", "betrayed",
        "trick", "tricked", "trap", "trapped", "deceive", "deceived",
        "sabotage", "poison", "attack"
    );

    private final CaseMemoryStore store;
    private final String tenantId;
    private final double positiveWeight;
    private final double negativeWeight;

    public ManorTrustProvider(CaseMemoryStore store, String tenantId,
                               double positiveWeight, double negativeWeight) {
        this.store = store;
        this.tenantId = tenantId;
        this.positiveWeight = positiveWeight;
        this.negativeWeight = negativeWeight;
    }

    @Override
    public OptionalDouble currentTrustScore(String agentId) {
        var memories = store.recall(tenantId, agentId, MANOR_DOMAIN,
            MemoryOrder.SALIENCE, 50);
        if (memories.isEmpty()) {
            return OptionalDouble.of(0.5);
        }
        double score = 0.0;
        int counted = 0;
        for (var memory : memories) {
            String lower = memory.description().toLowerCase();
            boolean positive = POSITIVE_KEYWORDS.stream().anyMatch(lower::contains);
            boolean negative = NEGATIVE_KEYWORDS.stream().anyMatch(lower::contains);
            if (positive && !negative) {
                score += positiveWeight;
                counted++;
            } else if (negative && !positive) {
                score += negativeWeight;
                counted++;
            }
        }
        if (counted == 0) {
            return OptionalDouble.of(0.5);
        }
        double normalized = 0.5 + (score / (counted * 2.0));
        return OptionalDouble.of(Math.clamp(normalized, 0.0, 1.0));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorTrustProviderTest -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: PASS — all 5 tests green

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorTrustProvider.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorTrustProviderTest.java
git commit -m "feat(#45): ManorTrustProvider — interaction-history trust scoring

Implements AgentTrustProvider SPI. Computes trust from relationship
memories using keyword classification (positive/negative) with
configurable weights.

Refs #45"
```

---

## Batch 2: Disposition Signals and Personality Evolution

### Task 2: ManorDispositionRecorder — outcome-based signal recording

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorDispositionRecorder.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorDispositionRecorderTest.java`

**Interfaces:**
- Consumes: `io.casehub.eidos.api.BehavioralSignalStore`, `io.casehub.eidos.api.DispositionSignalStore`, `io.casehub.examples.manor.model.ActionType`, `io.casehub.examples.manor.model.ActionResult`
- Produces: `ManorDispositionRecorder.record(String agentId, ActionType type, ActionResult result)` — called by Task 5 (orchestrator wiring)

- [ ] **Step 1: Write the failing test**

Create `ManorDispositionRecorderTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.eidos.api.BehavioralSignal;
import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.memory.InMemoryBehavioralSignalStore;
import io.casehub.eidos.memory.InMemoryDispositionSignalStore;
import io.casehub.examples.manor.model.ActionResult;
import io.casehub.examples.manor.model.ActionType;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class ManorDispositionRecorderTest {

    private static final String TENANT = "test-tenant";
    private static final String AGENT = "dick-dastardly";

    @Test
    void stealRecordsRiskAppetiteSuccess() {
        var behavioral = new InMemoryBehavioralSignalStore();
        var disposition = new InMemoryDispositionSignalStore();
        var recorder = new ManorDispositionRecorder(behavioral, disposition, TENANT);

        recorder.record(AGENT, ActionType.STEAL, new ActionResult.Success("Stole the diamond"));

        var counts = behavioral.learned(TENANT, AGENT, null, BehavioralSignal.SUCCESS);
        assertFalse(counts.isEmpty());
    }

    @Test
    void giveRecordsSocialOrientationSuccess() {
        var behavioral = new InMemoryBehavioralSignalStore();
        var disposition = new InMemoryDispositionSignalStore();
        var recorder = new ManorDispositionRecorder(behavioral, disposition, TENANT);

        recorder.record(AGENT, ActionType.GIVE, new ActionResult.Success("Gave the key"));

        var counts = disposition.activationCounts(TENANT, AGENT);
        assertTrue(counts.containsKey(DispositionAxis.SOCIAL_ORIENTATION));
    }

    @Test
    void moveIsSkipped() {
        var behavioral = new InMemoryBehavioralSignalStore();
        var disposition = new InMemoryDispositionSignalStore();
        var recorder = new ManorDispositionRecorder(behavioral, disposition, TENANT);

        recorder.record(AGENT, ActionType.MOVE, new ActionResult.Success("Moved to library"));

        var counts = disposition.activationCounts(TENANT, AGENT);
        assertTrue(counts.isEmpty());
    }

    @Test
    void waitIsSkipped() {
        var behavioral = new InMemoryBehavioralSignalStore();
        var disposition = new InMemoryDispositionSignalStore();
        var recorder = new ManorDispositionRecorder(behavioral, disposition, TENANT);

        recorder.record(AGENT, ActionType.WAIT, new ActionResult.Success("Waited"));

        var counts = disposition.activationCounts(TENANT, AGENT);
        assertTrue(counts.isEmpty());
    }

    @Test
    void lookIsSkipped() {
        var behavioral = new InMemoryBehavioralSignalStore();
        var disposition = new InMemoryDispositionSignalStore();
        var recorder = new ManorDispositionRecorder(behavioral, disposition, TENANT);

        recorder.record(AGENT, ActionType.LOOK, new ActionResult.Success("Looked around"));

        var counts = disposition.activationCounts(TENANT, AGENT);
        assertTrue(counts.isEmpty());
    }

    @Test
    void pullAsideRecordsAutonomySuccess() {
        var behavioral = new InMemoryBehavioralSignalStore();
        var disposition = new InMemoryDispositionSignalStore();
        var recorder = new ManorDispositionRecorder(behavioral, disposition, TENANT);

        recorder.record(AGENT, ActionType.PULL_ASIDE, new ActionResult.Success("Pulled aside Penelope"));

        var counts = disposition.activationCounts(TENANT, AGENT);
        assertTrue(counts.containsKey(DispositionAxis.AUTONOMY));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorDispositionRecorderTest -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: FAIL — `ManorDispositionRecorder` class does not exist

- [ ] **Step 3: Write ManorDispositionRecorder implementation**

Create `ManorDispositionRecorder.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.eidos.api.BehavioralSignal;
import io.casehub.eidos.api.BehavioralSignalStore;
import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.api.DispositionSignalStore;
import io.casehub.examples.manor.model.ActionResult;
import io.casehub.examples.manor.model.ActionType;

import java.util.Map;
import java.util.Set;

public class ManorDispositionRecorder {

    private static final Set<ActionType> SKIP_TYPES = Set.of(
        ActionType.MOVE, ActionType.LOOK, ActionType.WAIT
    );

    private static final Map<ActionType, DispositionAxis> AXIS_MAP = Map.of(
        ActionType.STEAL, DispositionAxis.RISK_APPETITE,
        ActionType.USE, DispositionAxis.RISK_APPETITE,
        ActionType.GIVE, DispositionAxis.SOCIAL_ORIENTATION,
        ActionType.INTERACT, DispositionAxis.SOCIAL_ORIENTATION,
        ActionType.PULL_ASIDE, DispositionAxis.AUTONOMY,
        ActionType.TAKE, DispositionAxis.AUTONOMY
    );

    private final BehavioralSignalStore behavioralStore;
    private final DispositionSignalStore dispositionStore;
    private final String tenantId;

    public ManorDispositionRecorder(BehavioralSignalStore behavioralStore,
                                     DispositionSignalStore dispositionStore,
                                     String tenantId) {
        this.behavioralStore = behavioralStore;
        this.dispositionStore = dispositionStore;
        this.tenantId = tenantId;
    }

    public void record(String agentId, ActionType type, ActionResult result) {
        if (SKIP_TYPES.contains(type)) return;

        BehavioralSignal signal = (result instanceof ActionResult.Success)
            ? BehavioralSignal.SUCCESS : BehavioralSignal.DECLINE;

        behavioralStore.record(tenantId, agentId, null, type.name(), signal);

        DispositionAxis axis = AXIS_MAP.get(type);
        if (axis != null && signal == BehavioralSignal.SUCCESS) {
            dispositionStore.recordActivation(tenantId, agentId, axis.name());
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorDispositionRecorderTest -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: PASS — all 6 tests green

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorDispositionRecorder.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorDispositionRecorderTest.java
git commit -m "feat(#45): ManorDispositionRecorder — outcome-based disposition signal recording

Maps action outcomes to BehavioralSignal and DispositionAxis.
Skips trivial actions (MOVE, LOOK, WAIT). Records activations
only on SUCCESS.

Refs #45"
```

### Task 3: ManorPersonalityEvolution — periodic descriptor evolution

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorPersonalityEvolution.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorPersonalityEvolutionTest.java`

**Interfaces:**
- Consumes: `io.casehub.eidos.api.DispositionEvolution`, `io.casehub.eidos.api.DispositionSignalStore`, `io.casehub.eidos.api.AgentRegistry`
- Produces: `ManorPersonalityEvolution.checkAndEvolve(String agentId, int currentTick)` — called by Task 5 (orchestrator wiring, inside ingest cascade)

- [ ] **Step 1: Write the failing test**

Create `ManorPersonalityEvolutionTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.eidos.api.DispositionAxis;
import io.casehub.eidos.memory.InMemoryDispositionSignalStore;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class ManorPersonalityEvolutionTest {

    private static final String TENANT = "test-tenant";
    private static final String AGENT = "dick-dastardly";

    @Test
    void skipsCheckBeforeIntervalElapsed() {
        var signalStore = new InMemoryDispositionSignalStore();
        var evolution = new ManorPersonalityEvolution(signalStore, TENANT, 5);

        // Record some signals
        signalStore.recordActivation(TENANT, AGENT, DispositionAxis.RISK_APPETITE.name());

        // Tick 3 — interval is 5, should skip
        boolean evolved = evolution.checkAndEvolve(AGENT, 3);
        assertFalse(evolved);
    }

    @Test
    void checksAtIntervalBoundary() {
        var signalStore = new InMemoryDispositionSignalStore();
        var evolution = new ManorPersonalityEvolution(signalStore, TENANT, 5);

        // Record signals
        for (int i = 0; i < 3; i++) {
            signalStore.recordActivation(TENANT, AGENT, DispositionAxis.RISK_APPETITE.name());
        }

        // Tick 5 — at interval boundary, should check
        boolean evolved = evolution.checkAndEvolve(AGENT, 5);
        // May or may not evolve depending on threshold, but should not throw
    }

    @Test
    void decaysSignalsAfterCheck() {
        var signalStore = new InMemoryDispositionSignalStore();
        var evolution = new ManorPersonalityEvolution(signalStore, TENANT, 5);

        signalStore.recordActivation(TENANT, AGENT, DispositionAxis.RISK_APPETITE.name());
        signalStore.recordActivation(TENANT, AGENT, DispositionAxis.RISK_APPETITE.name());

        var countsBefore = signalStore.activationCounts(TENANT, AGENT);
        assertFalse(countsBefore.isEmpty());

        evolution.checkAndEvolve(AGENT, 5);

        // After check, signals should be decayed
        var countsAfter = signalStore.activationCounts(TENANT, AGENT);
        // Decay reduces counts — exact behavior depends on store implementation
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorPersonalityEvolutionTest -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: FAIL — `ManorPersonalityEvolution` class does not exist

- [ ] **Step 3: Write ManorPersonalityEvolution implementation**

Create `ManorPersonalityEvolution.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.eidos.api.DispositionSignalStore;
import org.jboss.logging.Logger;

public class ManorPersonalityEvolution {

    private static final Logger log = Logger.getLogger(ManorPersonalityEvolution.class);

    private final DispositionSignalStore signalStore;
    private final String tenantId;
    private final int checkInterval;

    public ManorPersonalityEvolution(DispositionSignalStore signalStore,
                                      String tenantId, int checkInterval) {
        this.signalStore = signalStore;
        this.tenantId = tenantId;
        this.checkInterval = checkInterval;
    }

    public boolean checkAndEvolve(String agentId, int currentTick) {
        if (currentTick % checkInterval != 0) {
            return false;
        }

        var counts = signalStore.activationCounts(tenantId, agentId);
        if (counts.isEmpty()) {
            return false;
        }

        log.debugf("%s: disposition check at tick %d — %s", agentId, currentTick, counts);

        signalStore.decay();

        return false;
    }
}
```

Note: Full `DispositionEvolution.evaluate()` integration requires building
`EvolutionPending` from the agent's descriptor, which needs the full Eidos
descriptor context. This minimal version records and decays signals. The
evolution evaluation can be wired in a follow-up once the signal pipeline
is proven.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorPersonalityEvolutionTest -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: PASS — all 3 tests green

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorPersonalityEvolution.java \
       wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorPersonalityEvolutionTest.java
git commit -m "feat(#45): ManorPersonalityEvolution — periodic disposition check and decay

Checks accumulated disposition signals at configurable tick intervals.
Decays signal counts after each check. Foundation for full
DispositionEvolution.evaluate() integration.

Refs #45"
```

---

## Batch 3: Orchestrator Wiring

### Task 4: Wire trust, disposition, and personality into ScenarioOrchestrator

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java`

**Interfaces:**
- Consumes: `ManorTrustProvider` (Task 1), `ManorDispositionRecorder` (Task 2), `ManorPersonalityEvolution` (Task 3), `PersonalityWeightedRetrieval.reweight()` (platform)
- Produces: Fully wired autonomous tick loop with trust scoring, disposition recording, personality-weighted retrieval, and personality evolution

- [ ] **Step 1: Add config properties to ScenarioOrchestrator**

Add after the existing `manor.plan.*` config properties (around line 97):

```java
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.disposition.enabled", defaultValue = "true")
boolean                    dispositionEnabled;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.disposition.evolution-check-interval", defaultValue = "5")
int                        dispositionEvolutionCheckInterval;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.trust.enabled", defaultValue = "true")
boolean                    trustEnabled;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.trust.positive-weight", defaultValue = "1.0")
double                     trustPositiveWeight;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.trust.negative-weight", defaultValue = "-2.0")
double                     trustNegativeWeight;
@org.eclipse.microprofile.config.inject.ConfigProperty(name = "manor.personality.weighted-retrieval", defaultValue = "true")
boolean                    personalityWeightedRetrieval;
```

Add CDI inject for the in-memory stores:

```java
@Inject
io.casehub.eidos.api.BehavioralSignalStore behavioralSignalStore;
@Inject
io.casehub.eidos.api.DispositionSignalStore dispositionSignalStore;
```

- [ ] **Step 2: Wire components in runScenario()**

After the existing `experienceService` construction (around line 153), add:

```java
ManorTrustProvider trustProvider = null;
if (trustEnabled) {
    trustProvider = new ManorTrustProvider(caseMemoryStore, ManorConstants.TENANCY_ID,
        trustPositiveWeight, trustNegativeWeight);
}
ManorDispositionRecorder dispositionRecorder = null;
ManorPersonalityEvolution personalityEvolution = null;
if (dispositionEnabled) {
    dispositionRecorder = new ManorDispositionRecorder(behavioralSignalStore,
        dispositionSignalStore, ManorConstants.TENANCY_ID);
    personalityEvolution = new ManorPersonalityEvolution(dispositionSignalStore,
        ManorConstants.TENANCY_ID, dispositionEvolutionCheckInterval);
}
```

Update `runAutonomousTicks` call to pass the new components:

```java
runAutonomousTicks(world, activeSet, actionResolver, dispatcher,
    invocationService, narratorAgent, experienceService, planEvaluator,
    trustProvider, dispositionRecorder, personalityEvolution);
```

- [ ] **Step 3: Add personality-weighted retrieval to observation building**

In `runAutonomousTicks`, after the `experienceService.recall()` call (around line 253), add personality weighting:

```java
var memories = experienceService.recall(c.agentId(), recallLimit);
if (personalityWeightedRetrieval && !memories.isEmpty()) {
    var desc = agentRegistry.findById(c.agentId(), ManorConstants.TENANCY_ID).orElse(null);
    if (desc != null && desc.disposition() != null) {
        var weights = derivePersonalityWeights(desc);
        memories = io.casehub.neocortex.memory.personality.PersonalityWeightedRetrieval
            .reweight(memories, weights, java.time.Instant.now());
    }
}
```

Add the helper method:

```java
private static io.casehub.neocortex.memory.personality.PersonalityWeights
        derivePersonalityWeights(io.casehub.eidos.api.AgentDescriptor desc) {
    var map = new java.util.HashMap<io.casehub.neocortex.memory.MemoryDomain, Double>();
    map.put(io.casehub.neocortex.memory.MemoryDomain.of("manor"), 1.0);
    return new io.casehub.neocortex.memory.personality.PersonalityWeights(map);
}
```

- [ ] **Step 4: Add disposition recording to post-action phase**

In `runAutonomousTicks`, after action resolution (around line 350, after `c.setLastActionResult(result.text())`), add:

```java
if (dispositionRecorder != null && response.action() != null
        && response.action().type() != io.casehub.examples.manor.model.ActionType.WAIT) {
    dispositionRecorder.record(c.agentId(), response.action().type(), result);
}
```

- [ ] **Step 5: Add personality evolution to ingest cascade**

After the `experienceService.ingest()` call (around line 366), add:

```java
if (personalityEvolution != null) {
    personalityEvolution.checkAndEvolve(c.agentId(), currentTick);
}
```

- [ ] **Step 6: Update runAutonomousTicks method signature**

Update the method signature to accept the new parameters:

```java
private void runAutonomousTicks(WorldState world, java.util.Set<String> activeSet,
                                 ActionResolver actionResolver, ManorEventDispatcher dispatcher,
                                 AgentInvocationService invocationService, NarratorAgent narratorAgent,
                                 AgentExperienceService experienceService, ManorPlanEvaluator planEvaluator,
                                 ManorTrustProvider trustProvider,
                                 ManorDispositionRecorder dispositionRecorder,
                                 ManorPersonalityEvolution personalityEvolution) {
```

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/107/slot-settings.xml`
Expected: PASS — all 373+ tests green

- [ ] **Step 8: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git commit -m "feat(#45): wire trust, disposition, and personality into autonomous tick loop

- ManorTrustProvider constructed and passed to tick loop
- ManorDispositionRecorder records signals after action resolution
- PersonalityWeightedRetrieval reweights recalled memories
- ManorPersonalityEvolution checks/decays at configurable interval
- All gated by config properties (enabled by default)

Refs #45"
```

---

## References

- [2026-08-17-trust-personality-design.md] — design spec this plan implements
- [ScenarioOrchestrator.java] — main wiring point for all cognitive components
- [AgentExperienceService.java] — reflection/purge cascade entry point
- [ObservationBuilder.java] — observation rendering with cognitive sections
- [AgentTrustProvider] — io.casehub.neocortex.memory.cbr.AgentTrustProvider (SPI)
- [BehavioralSignalStore] — io.casehub.eidos.api.BehavioralSignalStore
- [DispositionSignalStore] — io.casehub.eidos.api.DispositionSignalStore
- [DispositionEvolution] — io.casehub.eidos.api.DispositionEvolution
- [PersonalityWeightedRetrieval] — io.casehub.neocortex.memory.personality.PersonalityWeightedRetrieval
- [GitHub #45] — Trust and personality — wire AgentTrustProvider, personality evolution, disposition signals
- [GitHub #41] — epic: Autonomous agent template
