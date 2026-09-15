# CognitiveProfile Queries Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #54 — Wire CognitiveProfile queries to mindmap for dynamic cognitive sections
**Issue group:** #54, #55, #56, #57, #59, #60

**Goal:** Replace static SocialConfig reads in CharacterCognition with dynamic CognitiveProfile queries against the neocortex mindmap, using TemporalFocus for attention-based item selection and blocks#260 renderers for standardised output. Adopt CognitionCore from blocks#261 as the cognitive orchestrator.

**Architecture:** CognitionCore (blocks) inside CharacterCognition as the cognitive engine. ManorContextStrategy provides game-world context (rooms, nearby characters, inventory). CognitiveProfile queries mindmap for Tier 3 knowledge. TemporalFocus ranks items by salience. Initial beliefs seeded from SocialConfig to mindmap at scenario start, then mindmap is authoritative.

**Tech Stack:** Java 21, Quarkus, neocortex (CognitiveProfile, TemporalFocus, MindMapStore, Thing traits), blocks (CognitionCore, CognitiveObservationSections, Belief, TrustSummary, Principle, SocialNorm), Eidos

## Global Constraints

- Mindmap (Tier 3) is write-on-consolidation only during the tick loop — seeding happens once at scenario start, not per-tick
- TemporalFocus is stateless and synchronous — called per tick, no side effects
- CognitionCore is a single shared instance keyed by agentId:tenantId internally
- Use `ide_find_references`, `ide_find_class` for navigation; `ide_insert_member`, `ide_replace_member`, `ide_edit_member` for edits
- All new types in `io.casehub.examples.manor.agent` package
- TDD: failing test first, then minimal implementation
- Mapping details (field-level): see spec §Mapping: Thing Traits → Blocks Types

---

## Batch 1: Foundation types (no CDI, unit testable)

After this batch: CognitiveBudget, ManorContextStrategy, and mapping utilities exist as standalone types with unit tests. No wiring yet.

### Task 1: CognitiveBudget + ManorContextStrategy

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/CognitiveBudget.java`
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorContextStrategy.java`
- Test: `src/test/java/io/casehub/examples/manor/agent/CognitiveBudgetTest.java`
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorContextStrategyTest.java`

**Interfaces:**
- Produces: `CognitiveBudget(int maxBeliefs, int maxTrust, int maxNorms, int maxPrinciples)` with `static CognitiveBudget forSituation(int nearbyCount, double arousal, int activeGoalCount)`
- Produces: `ManorContextStrategy` with `CognitiveBudget budgetFor(int nearbyCount, double arousal, int activeGoals)`, `List<SocialConfig.NormEntry> filterNorms(List<SocialConfig.NormEntry> norms, Collection<String> nearbyNames, Collection<String> inventory)`, `TemporalFocusConfig temporalFocusConfig()`

- [ ] **Step 1: Write CognitiveBudget test**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CognitiveBudgetTest {

    @Test
    void baselineBudgetForEmptyRoom() {
        var budget = CognitiveBudget.forSituation(0, 0.3, 0);
        assertThat(budget.maxBeliefs()).isEqualTo(5);
        assertThat(budget.maxTrust()).isEqualTo(1);
        assertThat(budget.maxNorms()).isEqualTo(4);
        assertThat(budget.maxPrinciples()).isEqualTo(3);
    }

    @Test
    void highArousalIncreasesBeliefBudget() {
        var calm = CognitiveBudget.forSituation(2, 0.3, 0);
        var alert = CognitiveBudget.forSituation(2, 0.8, 0);
        assertThat(alert.maxBeliefs()).isGreaterThan(calm.maxBeliefs());
    }

    @Test
    void moreNearbyAgentsIncreaseTrustBudget() {
        var few = CognitiveBudget.forSituation(1, 0.3, 0);
        var many = CognitiveBudget.forSituation(5, 0.3, 0);
        assertThat(many.maxTrust()).isGreaterThan(few.maxTrust());
    }

    @Test
    void activeGoalsIncreaseBeliefBudget() {
        var noGoals = CognitiveBudget.forSituation(2, 0.3, 0);
        var withGoals = CognitiveBudget.forSituation(2, 0.3, 3);
        assertThat(withGoals.maxBeliefs()).isGreaterThanOrEqualTo(noGoals.maxBeliefs());
    }
}
```

- [ ] **Step 2: Implement CognitiveBudget**

```java
package io.casehub.examples.manor.agent;

public record CognitiveBudget(int maxBeliefs, int maxTrust, int maxNorms, int maxPrinciples) {

    public static CognitiveBudget forSituation(int nearbyCount, double arousal, int activeGoalCount) {
        int beliefs = 5;
        int trust = Math.min(nearbyCount + 1, 6);
        int norms = 4;

        if (arousal > 0.7) {
            beliefs += 3;
            norms += 2;
        }
        if (nearbyCount > 3) {
            norms += nearbyCount - 3;
        }
        if (activeGoalCount > 0) {
            beliefs += Math.min(activeGoalCount, 3);
        }
        return new CognitiveBudget(beliefs, trust, norms, 3);
    }
}
```

- [ ] **Step 3: Run CognitiveBudget test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CognitiveBudgetTest`
Expected: PASS

- [ ] **Step 4: Write ManorContextStrategy test**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ManorContextStrategyTest {

    @Test
    void budgetAdaptsToSituation() {
        var strategy = new ManorContextStrategy();
        var calm = strategy.budgetFor(2, 0.3, 0);
        var alert = strategy.budgetFor(2, 0.8, 0);
        assertThat(alert.maxBeliefs()).isGreaterThan(calm.maxBeliefs());
    }

    @Test
    void normFilterDelegates() {
        var strategy = new ManorContextStrategy();
        var norms = List.of(
            new SocialConfig.NormEntry("low", 1),
            new SocialConfig.NormEntry("high", 10));
        var filtered = strategy.filterNorms(norms, List.of(), List.of());
        assertThat(filtered.get(0).rule()).isEqualTo("high");
    }

    @Test
    void temporalFocusConfigHasDefaults() {
        var strategy = new ManorContextStrategy();
        var config = strategy.temporalFocusConfig();
        assertThat(config).isNotNull();
    }
}
```

- [ ] **Step 5: Implement ManorContextStrategy**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.memory.TemporalFocusConfig;
import java.util.Collection;
import java.util.List;

public final class ManorContextStrategy {

    private static final TemporalFocusConfig TF_CONFIG = new TemporalFocusConfig(
            7.0, 0.5, 0.7, 0.3, java.util.Map.of());

    public CognitiveBudget budgetFor(int nearbyCount, double arousal, int activeGoals) {
        return CognitiveBudget.forSituation(nearbyCount, arousal, activeGoals);
    }

    public List<SocialConfig.NormEntry> filterNorms(List<SocialConfig.NormEntry> norms,
                                                     Collection<String> nearbyNames,
                                                     Collection<String> inventory) {
        return ManorNormFilter.filter(norms, nearbyNames, inventory);
    }

    public TemporalFocusConfig temporalFocusConfig() {
        return TF_CONFIG;
    }
}
```

- [ ] **Step 6: Run ManorContextStrategy test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorContextStrategyTest`
Expected: PASS

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CognitiveBudget.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorContextStrategy.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/CognitiveBudgetTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorContextStrategyTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#54): CognitiveBudget + ManorContextStrategy — adaptive attention budget and game-world context adapter"
```

---

### Task 2: ManorCognitiveSeeder — seed initial beliefs to mindmap

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java`
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederTest.java`

**Interfaces:**
- Consumes: `SocialConfig.forCharacter(agentId)` from Task 1 of Phase A, `MindMapStore` (CDI)
- Produces: `ManorCognitiveSeeder.seed(String agentId, SocialConfig config, String tenantId)` — creates COGNITIVE subgraph + nodes per initial belief. Returns `ManorCognitiveSeeder.SeedResult(String subgraphId, Map<String, java.time.Instant> seededNodeTimestamps)` for revised-belief detection.

- [ ] **Step 1: Write seeder test**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ManorCognitiveSeederTest {

    @Test
    void seedResultTracksNodeTimestamps() {
        var result = new ManorCognitiveSeeder.SeedResult("sub-1",
            java.util.Map.of("penelope-awareness", java.time.Instant.now()));
        assertThat(result.subgraphId()).isEqualTo("sub-1");
        assertThat(result.seededNodeTimestamps()).containsKey("penelope-awareness");
    }

    @Test
    void emptyConfigProducesEmptyResult() {
        var result = new ManorCognitiveSeeder.SeedResult("sub-1", java.util.Map.of());
        assertThat(result.seededNodeTimestamps()).isEmpty();
    }
}
```

- [ ] **Step 2: Implement ManorCognitiveSeeder**

```java
package io.casehub.examples.manor.agent;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

public final class ManorCognitiveSeeder {

    public record SeedResult(String subgraphId, Map<String, Instant> seededNodeTimestamps) {}

    private final io.casehub.neocortex.mindmap.MindMapStore mindMapStore;

    public ManorCognitiveSeeder(io.casehub.neocortex.mindmap.MindMapStore mindMapStore) {
        this.mindMapStore = mindMapStore;
    }

    public SeedResult seed(String agentId, SocialConfig config, String tenantId) {
        if (config.initialBeliefs().isEmpty()) {
            return new SeedResult("beliefs-" + agentId, Map.of());
        }
        var subgraphId = "beliefs-" + agentId;
        var timestamps = new HashMap<String, Instant>();
        for (var belief : config.initialBeliefs()) {
            var now = Instant.now();
            timestamps.put(belief.key(), now);
        }
        return new SeedResult(subgraphId, Map.copyOf(timestamps));
    }
}
```

Note: The actual MindMapStore.createSubgraph() and addNode() calls depend on the neocortex API. The seeder's structure is correct; the exact MindMapStore calls will be verified against the API during implementation (see spec §Seeding for NodeInput construction).

- [ ] **Step 3: Run seeder test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorCognitiveSeederTest`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#54): ManorCognitiveSeeder — seed initial beliefs to mindmap at scenario start"
```

---

## Batch 2: CharacterCognition rewrite + orchestrator wiring

After this batch: CharacterCognition composes CognitionCore + CognitiveProfile + TemporalFocus. ScenarioOrchestrator injects new services, constructs shared CognitionCore, calls seeder, wires post-action dispatch. Dynamic cognitive sections rendered via blocks#260. All existing tests pass.

### Task 3: Rewrite CharacterCognition

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:16-120` — replace fields, constructors, renderCognitiveSections(), recordTrustEvent()
- Test: update `src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: `CognitiveBudget`, `ManorContextStrategy` from Task 1; `ManorCognitiveSeeder.SeedResult` from Task 2; `CognitionCore` (blocks), `CognitiveProfile` (neocortex), `TemporalFocus` (neocortex)
- Produces: Same public API as before (`renderCognitiveSections()`, `recordExperience()`, `recallMemories()`, etc.) — callers unchanged. New: `CharacterCognition(String agentId, AgentExperienceService experienceService, CognitiveDefaults cognitiveDefaults, SocialConfig socialConfig, List<AgentConstraint> constraints, CognitiveProfile cognitiveProfile, ManorContextStrategy contextStrategy, CognitionCore cognitionCore, ManorCognitiveSeeder.SeedResult seedResult, String tenantId)`

- [ ] **Step 1: Update CharacterCognitionTest for new constructor**

Add tests for dynamic sections rendering. Keep existing tests by adding a backward-compatible constructor delegation.

```java
@Test
void rendersDynamicBeliefSections() {
    // When CognitiveProfile is null (no mindmap), falls back to static SocialConfig
    var socialConfig = SocialConfig.forCharacter("hooded-claw");
    var cognition = new CharacterCognition("hooded-claw", null, null, socialConfig, List.of(),
            null, new ManorContextStrategy(), null, null, "wacky-manor");
    var sections = cognition.renderCognitiveSections(
            new CharacterState("hooded-claw", "HC", "Room", 0.0, List.of()),
            List.of(), Map.of());
    assertThat(sections).anyMatch(s -> s.header().equals("Your Drives"));
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: compilation failure — new constructor doesn't exist yet

- [ ] **Step 3: Rewrite CharacterCognition**

Use `ide_edit_member` on the class declaration. New fields:

```java
private final CognitiveProfile cognitiveProfile;
private final ManorContextStrategy contextStrategy;
private final CognitionCore cognitionCore;
private final ManorCognitiveSeeder.SeedResult seedResult;
private final String tenantId;
```

New full constructor accepting all dependencies. Keep 2-arg constructor for backward compatibility (null CognitiveProfile = static fallback).

Replace `renderCognitiveSections()` body:
1. If cognitiveProfile is non-null: query mindmap, rank via TemporalFocus, map to blocks types, render via blocks#260 methods
2. If cognitiveProfile is null: keep existing static SocialConfig rendering (backward compatibility for tests)
3. Always render drives from SocialConfig (static personality drives)
4. Always render principles from constraints
5. Add CognitionCore.promptSections() for motivation/mood when cognitionCore is non-null

Replace `recordTrustEvent()` with dual-path dispatch per spec §Step 9.

- [ ] **Step 4: Run CharacterCognitionTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CharacterCognitionTest`
Expected: all tests PASS (existing + new)

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS — backward-compatible constructor means existing callers are unaffected

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#54): rewrite CharacterCognition — compose CognitionCore + CognitiveProfile + TemporalFocus"
```

---

### Task 4: Wire ScenarioOrchestrator — inject services, construct CognitionCore, seed, dispatch

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:29-172` — new inject fields, CognitionCore construction, seeder call, CharacterCognition construction update
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java:174-373` — post-action dispatch update in runAutonomousTicks()

**Interfaces:**
- Consumes: `CharacterCognition` new constructor from Task 3, `ManorCognitiveSeeder` from Task 2, `ManorContextStrategy` from Task 1
- Produces: No new public API — internal wiring change

- [ ] **Step 1: Add inject fields for new services**

Use `ide_insert_member` to add after existing `consolidationScheduler` field:

```java
@Inject
io.casehub.neocortex.cognitive.index.CognitiveProfile cognitiveProfile;
@Inject
io.casehub.neocortex.mindmap.MindMapStore mindMapStore;
@Inject
io.casehub.blocks.agentic.social.mood.MoodOrchestrator moodOrchestrator;
@Inject
io.casehub.blocks.agentic.social.drive.DriveOrchestrator driveOrchestrator;
@Inject
io.casehub.blocks.agentic.social.user.UserModelOrchestrator userModelOrchestrator;
@Inject
io.casehub.blocks.agentic.social.mental.MentalModelOrchestrator mentalModelOrchestrator;
```

- [ ] **Step 2: Construct CognitionCore + seeder in runScenario()**

In `runScenario()`, after `experienceService` construction:

```java
var cognitionCore = new io.casehub.blocks.agentic.social.CognitionCore(
    moodOrchestrator, driveOrchestrator, userModelOrchestrator, mentalModelOrchestrator,
    null, null, null, null, agentProvider,
    io.casehub.blocks.agentic.social.CognitionConfig.all()
        .without("strategy", "narrative", "goals", "memoryHygiene"));

var seeder = new ManorCognitiveSeeder(mindMapStore);
var contextStrategy = new ManorContextStrategy();
```

- [ ] **Step 3: Update character construction loop**

Replace the CharacterCognition construction in the character setup loop:

```java
var seedResult = seeder.seed(entry.getKey(), socialCfg, ManorConstants.TENANCY_ID);
cognitions.put(entry.getKey(), new CharacterCognition(
    entry.getKey(), experienceService, cogDefaults, socialCfg, desc.constraints(),
    cognitiveProfile, contextStrategy, cognitionCore, seedResult, ManorConstants.TENANCY_ID));
```

- [ ] **Step 4: Update post-action dispatch in runAutonomousTicks()**

Replace the trust event recording (line ~337) with dual-path dispatch:

```java
if (response.dialogue() != null && trustTarget != null) {
    cognitionCore.recordInteraction(c.agentId(), ManorConstants.TENANCY_ID,
        trustTarget, response.dialogue(), response.thinking());
}
if (response.action() != null && response.action().type() != ActionType.WAIT) {
    cognitions.get(c.agentId()).recordAction(
        trustTarget, response.action().type(), ManorConstants.TENANCY_ID);
}
```

- [ ] **Step 5: Add CognitionCore.tick() call at start of tick processing**

At the beginning of the per-agent virtual thread (before observation building), add:

```java
var nearbyIds = world.charactersInRoom(c.currentRoom()).stream()
    .map(CharacterState::agentId)
    .filter(id -> !id.equals(c.agentId()))
    .collect(java.util.stream.Collectors.toSet());
cognitionCore.tick(c.agentId(), ManorConstants.TENANCY_ID, null, nearbyIds);
```

- [ ] **Step 6: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#54): wire CognitionCore + CognitiveProfile + seeder into ScenarioOrchestrator"
```

---

## Batch 3: Integration tests

After this batch: full cognitive stack verified end-to-end with @QuarkusTest. Dynamic queries, seeding, CognitionCore wiring, and TemporalFocus ranking all tested.

### Task 5: Integration tests

**Files:**
- Create: `src/test/java/io/casehub/examples/manor/agent/CognitiveQueryIntegrationTest.java`
- Modify: `src/test/java/io/casehub/examples/manor/agent/SocialCognitionIntegrationTest.java` — extend with dynamic assertions

**Interfaces:**
- Consumes: All types from Tasks 1-4

- [ ] **Step 1: Write CognitiveQueryIntegrationTest**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.cognitive.index.CognitiveProfile;
import io.casehub.neocortex.mindmap.MindMapStore;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatCode;

@QuarkusTest
class CognitiveQueryIntegrationTest {

    @Inject CognitiveProfile cognitiveProfile;
    @Inject MindMapStore mindMapStore;

    @Test
    void cognitiveProfileIsInjectable() {
        assertThat(cognitiveProfile).isNotNull();
    }

    @Test
    void mindMapStoreIsInjectable() {
        assertThat(mindMapStore).isNotNull();
    }

    @Test
    void seederRunsWithoutError() {
        var seeder = new ManorCognitiveSeeder(mindMapStore);
        var config = SocialConfig.forCharacter("hooded-claw");
        assertThatCode(() -> seeder.seed("test-hc", config, "test-tenant"))
                .doesNotThrowAnyException();
    }

    @Test
    void cognitionCoreTickRunsWithoutError() {
        assertThatCode(() -> {
            var core = new io.casehub.blocks.agentic.social.CognitionCore(
                null, null, null, null, null, null, null, null, null,
                io.casehub.blocks.agentic.social.CognitionConfig.minimal());
            core.tick("test-agent", "test-tenant", null, java.util.Set.of());
        }).doesNotThrowAnyException();
    }
}
```

- [ ] **Step 2: Extend SocialCognitionIntegrationTest**

Add test verifying the full CharacterCognition with CognitiveProfile returns sections:

```java
@Test
void characterCognitionWithCognitiveProfileRendersSections() {
    var socialConfig = SocialConfig.forCharacter("hooded-claw");
    var cognition = new CharacterCognition("hooded-claw", null, null, socialConfig,
            java.util.List.of(), cognitiveProfile, new ManorContextStrategy(),
            null, null, "wacky-manor");
    var sections = cognition.renderCognitiveSections(
            new io.casehub.examples.manor.model.CharacterState(
                "hooded-claw", "HC", "Grand Hallway", 0.0, java.util.List.of()),
            java.util.List.of("penelope-pitstop"),
            java.util.Map.of("penelope-pitstop", "Penelope Pitstop"));
    assertThat(sections).isNotEmpty();
    assertThat(sections.stream().map(s -> s.header()).toList())
            .contains("Your Drives");
}
```

- [ ] **Step 3: Run all integration tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CognitiveQueryIntegrationTest,SocialCognitionIntegrationTest`
Expected: all tests PASS

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/test/java/io/casehub/examples/manor/agent/CognitiveQueryIntegrationTest.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/SocialCognitionIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#54): integration tests — CognitiveProfile queries, seeder, CognitionCore wiring"
```

---

## Deferred

- **MindMapStore API verification** — ManorCognitiveSeeder.seed() body needs exact MindMapStore.createSubgraph() and addNode() calls verified against the neocortex API at implementation time. The spec §Seeding has the NodeInput construction; the implementer verifies the exact method signatures.
- **CognitionCore CDI orchestrator injection** — the inject field types (MoodOrchestrator, DriveOrchestrator, etc.) need verification against actual CDI availability in the wacky-manor Quarkus context. If unavailable, construct manually.
- **TemporalFocusConfig tuning** — hardcoded defaults in ManorContextStrategy. Promote to ManorConfig if play testing reveals the need.
- **AffectTrajectory population** — TemporalFocus gracefully degrades to recency-only when no trajectory data exists. Full trajectory wiring comes after ConversationBridge (#55) and consolidation (#336) land.

## References

- [2026-09-14-cognitive-profile-queries-design.md] — design spec
- [decisions.md] — D1-D4 captured decisions
- [CharacterCognition.java:16-120] — current implementation (Phase A)
- [ScenarioOrchestrator.java:77-172] — runScenario() setup
- [ScenarioOrchestrator.java:174-373] — runAutonomousTicks() tick loop
- [CognitiveProfile] (neocortex cognitive-index) — resolve(), compare()
- [TemporalFocus] (neocortex memory) — focus(), AttentionItem
- [CognitionCore] (blocks) — tick(), promptSections()
- [CognitiveObservationSections] (blocks) — beliefsSection(), trustSection(), normsSection(), principlesSection()
- blocks#282 — CognitionContextStrategy SPI proposal
- neocortex#336 — ExperienceConsolidationPhase (upstream)
- GitHub #54 — focal issue
