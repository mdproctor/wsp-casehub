# Relationship Stage Thresholds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #70 — Relationship stage thresholds — interaction volume gates social behavior
**Issue group:** #63, #283, #66, #70 (current branch covers all)

**Goal:** Add familiarity-based relationship stages that gate trust-related social behaviors and perception depth based on accumulated interaction volume.

**Architecture:** A new `RelationshipStagePhase` consolidation phase in blocks-core computes familiarity per agent pair using the existing `computeFamiliarity()` formula and persists stage on overlay nodes. ManorContextStrategy gains trust-gating methods. PerceptionTranslator renders stage-appropriate perception depth.

**Tech Stack:** Java 26, Quarkus, CDI, MindMapStore (neocortex), CaseMemoryStore (neocortex), InMemoryMindMapStore (test)

## Global Constraints

- Use existing `RelationshipStageConfig`, `StageTier`, `computeFamiliarity()` from blocks-core — no new formula
- 5-tier model: stranger (0.0), acquaintance (0.2), familiar (0.4), friend (0.6), confidant (0.8)
- Adversarial behaviors (scheming, suspicion) stay drive-gated — NOT stage-gated
- Only trust-unlocked behaviors (disclosure, cooperation) are stage-gated
- `RelationshipStagePhase` @Priority(18) — after DriveAdaptation@17, before Merge@20
- PAD pleasure classification: `> 0.1` → positive, `< -0.1` → negative, else neutral

---

## Batch 1: Platform infrastructure (blocks-core)

### Task 1: OverlayFamiliarityPropertyModel + RelationshipStageConfigProvider

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/OverlayFamiliarityPropertyModel.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/RelationshipStageConfigProvider.java`

**Interfaces:**
- Produces: `OverlayFamiliarityPropertyModel.FAMILIARITY_SCORE`, `.FAMILIARITY_STAGE`, `.FAMILIARITY_INTERACTION_COUNT` (String constants)
- Produces: `RelationshipStageConfigProvider` (`@FunctionalInterface`, `RelationshipStageConfig forAgent(String agentId)`)

- [ ] **Step 1: Write the failing test for OverlayFamiliarityPropertyModel constants**

Create test in wacky-manor (the test will verify integration later; for now verify constants are accessible):

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class OverlayFamiliarityPropertyModelTest {
    @Test
    void constantsAreDefined() {
        assertThat(OverlayFamiliarityPropertyModel.FAMILIARITY_SCORE).isEqualTo("familiarity-score");
        assertThat(OverlayFamiliarityPropertyModel.FAMILIARITY_STAGE).isEqualTo("familiarity-stage");
        assertThat(OverlayFamiliarityPropertyModel.FAMILIARITY_INTERACTION_COUNT).isEqualTo("familiarity-interaction-count");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=OverlayFamiliarityPropertyModelTest`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement OverlayFamiliarityPropertyModel**

Use `ide_create_file` to create:

```java
package io.casehub.blocks.agentic.social;

public final class OverlayFamiliarityPropertyModel {
    public static final String FAMILIARITY_SCORE = "familiarity-score";
    public static final String FAMILIARITY_STAGE = "familiarity-stage";
    public static final String FAMILIARITY_INTERACTION_COUNT = "familiarity-interaction-count";

    private OverlayFamiliarityPropertyModel() {}
}
```

- [ ] **Step 4: Implement RelationshipStageConfigProvider**

Use `ide_create_file` to create:

```java
package io.casehub.blocks.agentic.social;

@FunctionalInterface
public interface RelationshipStageConfigProvider {
    RelationshipStageConfig forAgent(String agentId);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=OverlayFamiliarityPropertyModelTest`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/OverlayFamiliarityPropertyModel.java
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/RelationshipStageConfigProvider.java
git commit -m "feat(#70): OverlayFamiliarityPropertyModel + RelationshipStageConfigProvider"
```

Note: `OverlayFamiliarityPropertyModelTest` is in wacky-manor and will be committed with the examples repo later.

### Task 2: RelationshipStagePhase consolidation

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/RelationshipStagePhase.java`

**Interfaces:**
- Consumes: `ConsolidationPhase` (neocortex-mindmap-intelligence), `MindMapStore` (neocortex-mindmap-api), `CaseMemoryStore` (neocortex-memory-api), `RelationshipStageConfigProvider` (Task 1), `OverlayFamiliarityPropertyModel` (Task 1), `UserModelOrchestrator.computeFamiliarity(int, int, int, RelationshipStageConfig, long)`, `RelationshipStageConfig.resolveStage(double)`
- Produces: `RelationshipStagePhase` — ConsolidationPhase @Priority(18). Constructor: `(MindMapStore, CaseMemoryStore, RelationshipStageConfigProvider, String peopleSubgraphName, long consolidationIntervalMs)`

- [ ] **Step 1: Write the failing test**

Create test in wacky-manor following DriveAdaptationIntegrationTest pattern:

```java
package io.casehub.examples.manor.engine;

import io.casehub.blocks.agentic.social.OverlayFamiliarityPropertyModel;
import io.casehub.blocks.agentic.social.RelationshipStageConfig;
import io.casehub.blocks.agentic.social.RelationshipStagePhase;
import io.casehub.examples.manor.agent.ManorCognitiveSeeder;
import io.casehub.examples.manor.agent.ManorSocialConfigLoader;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.MemoryDomain;
import io.casehub.neocortex.memory.MemoryOrder;
import io.casehub.neocortex.memory.MemoryQuery;
import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.MemoryInput;
import io.casehub.neocortex.memory.Subject;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RelationshipStageIntegrationTest {

    private static final String TENANT = "stage-test";
    private static final String AGENT = "hooded-claw";
    private static final String TARGET = "penelope-pitstop";

    @Test
    void fullLifecycle_seedInteractConsolidate() {
        var mindMapStore = new InMemoryMindMapStore();
        var seeder = new ManorCognitiveSeeder(mindMapStore);
        var allConfigs = ManorSocialConfigLoader.load();

        seeder.seedPeople(allConfigs, TENANT);

        var memories = buildInteractionMemories(20);
        var memoryStore = new StubCaseMemoryStore(memories);

        var phase = new RelationshipStagePhase(
            mindMapStore, memoryStore,
            agentId -> RelationshipStageConfig.defaults(),
            ManorCognitiveSeeder.PEOPLE_SUBGRAPH,
            60_000L);

        phase.run(TENANT, List.of());

        var subgraphs = mindMapStore.listSubgraphs(TENANT);
        var peopleSg = subgraphs.stream()
            .filter(sg -> ManorCognitiveSeeder.PEOPLE_SUBGRAPH.equals(sg.name()))
            .findFirst().orElseThrow();
        var nodes = mindMapStore.nodesIn(peopleSg.id(), TENANT);

        var overlay = nodes.stream()
            .filter(n -> n.traits().contains("overlay"))
            .filter(n -> AGENT.equals(n.property(
                io.casehub.neocortex.mindmap.OverlayRef.AGENT_ID).orElse(null)))
            .findFirst().orElseThrow();

        assertThat(overlay.property(OverlayFamiliarityPropertyModel.FAMILIARITY_SCORE))
            .isPresent();
        assertThat(overlay.property(OverlayFamiliarityPropertyModel.FAMILIARITY_STAGE))
            .isPresent()
            .hasValueSatisfying(stage -> assertThat(stage).isNotEqualTo("stranger"));
        assertThat(overlay.property(OverlayFamiliarityPropertyModel.FAMILIARITY_INTERACTION_COUNT))
            .isPresent()
            .hasValue("20");
    }

    @Test
    void noMemories_stageRemainsStranger() {
        var mindMapStore = new InMemoryMindMapStore();
        var seeder = new ManorCognitiveSeeder(mindMapStore);
        var allConfigs = ManorSocialConfigLoader.load();

        seeder.seedPeople(allConfigs, TENANT);

        var memoryStore = new StubCaseMemoryStore(List.of());

        var phase = new RelationshipStagePhase(
            mindMapStore, memoryStore,
            agentId -> RelationshipStageConfig.defaults(),
            ManorCognitiveSeeder.PEOPLE_SUBGRAPH,
            60_000L);

        phase.run(TENANT, List.of());

        var subgraphs = mindMapStore.listSubgraphs(TENANT);
        var peopleSg = subgraphs.stream()
            .filter(sg -> ManorCognitiveSeeder.PEOPLE_SUBGRAPH.equals(sg.name()))
            .findFirst().orElseThrow();
        var nodes = mindMapStore.nodesIn(peopleSg.id(), TENANT);

        var overlay = nodes.stream()
            .filter(n -> n.traits().contains("overlay"))
            .filter(n -> AGENT.equals(n.property(
                io.casehub.neocortex.mindmap.OverlayRef.AGENT_ID).orElse(null)))
            .findFirst().orElseThrow();

        assertThat(overlay.property(OverlayFamiliarityPropertyModel.FAMILIARITY_STAGE))
            .hasValue("stranger");
    }

    private List<Memory> buildInteractionMemories(int count) {
        var memories = new java.util.ArrayList<Memory>();
        for (int i = 0; i < count; i++) {
            memories.add(new Memory(
                "mem-" + i, Subject.of("agent", AGENT),
                new MemoryDomain("relationship", null), TENANT,
                null, "interaction " + i, Map.of(),
                Instant.now().minusSeconds(count - i),
                null, 0.5, 0.3, 0.0, null, null));
        }
        return memories;
    }

    static class StubCaseMemoryStore implements CaseMemoryStore {
        private final List<Memory> memories;
        StubCaseMemoryStore(List<Memory> memories) { this.memories = memories; }

        @Override
        public String store(MemoryInput input) { return "stub"; }

        @Override
        public List<Memory> query(MemoryQuery query) { return memories; }

        @Override
        public int erase(io.casehub.neocortex.memory.EraseRequest request) { return 0; }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=RelationshipStageIntegrationTest`
Expected: FAIL — `RelationshipStagePhase` does not exist

- [ ] **Step 3: Implement RelationshipStagePhase**

Use `ide_create_file` in blocks-core:

```java
package io.casehub.blocks.agentic.social;

import io.casehub.neocortex.memory.CaseMemoryStore;
import io.casehub.neocortex.memory.Memory;
import io.casehub.neocortex.memory.relationship.RelationshipQuery;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import io.casehub.neocortex.mindmap.NodeUpdate;
import io.casehub.neocortex.mindmap.OverlayRef;
import io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase;
import jakarta.annotation.Priority;

import java.time.Duration;
import java.time.Instant;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.logging.Logger;

@Priority(18)
public class RelationshipStagePhase implements ConsolidationPhase {

    private static final Logger LOG = Logger.getLogger(RelationshipStagePhase.class.getName());
    private static final double POSITIVE_THRESHOLD = 0.1;
    private static final double NEGATIVE_THRESHOLD = -0.1;

    private final MindMapStore mindMapStore;
    private final CaseMemoryStore memoryStore;
    private final RelationshipStageConfigProvider configProvider;
    private final String peopleSubgraphName;
    private final long consolidationIntervalMs;

    public RelationshipStagePhase(
            MindMapStore mindMapStore,
            CaseMemoryStore memoryStore,
            RelationshipStageConfigProvider configProvider,
            String peopleSubgraphName,
            long consolidationIntervalMs) {
        this.mindMapStore = mindMapStore;
        this.memoryStore = memoryStore;
        this.configProvider = configProvider;
        this.peopleSubgraphName = peopleSubgraphName;
        this.consolidationIntervalMs = consolidationIntervalMs;
    }

    @Override
    public String name() {
        return "relationship-stage";
    }

    @Override
    public void run(String tenantId, List<String> subgraphPriority) {
        var subgraphs = mindMapStore.listSubgraphs(tenantId);
        var peopleSg = subgraphs.stream()
            .filter(sg -> peopleSubgraphName.equals(sg.name()))
            .findFirst();

        if (peopleSg.isEmpty()) {
            LOG.warning("No people subgraph '" + peopleSubgraphName
                + "' found in tenant " + tenantId);
            return;
        }

        var nodes = mindMapStore.nodesIn(peopleSg.get().id(), tenantId);

        var sharedNodes = new HashMap<String, MindMapNode>();
        for (var node : nodes) {
            if (!node.traits().contains("overlay")) {
                node.property("agentId").ifPresent(aid -> sharedNodes.put(node.id(), node));
            }
        }

        for (var overlay : nodes) {
            if (!overlay.traits().contains("overlay")) continue;
            var observerId = overlay.property(OverlayRef.AGENT_ID).orElse(null);
            if (observerId == null) continue;

            var targetNodeId = OverlayRef.sharedNodeId(overlay).orElse(null);
            if (targetNodeId == null) continue;
            var sharedNode = sharedNodes.get(targetNodeId);
            if (sharedNode == null) continue;
            var targetAgentId = sharedNode.property("agentId").orElse(null);
            if (targetAgentId == null) continue;

            updateFamiliarity(overlay, observerId, targetAgentId, tenantId);
        }
    }

    private void updateFamiliarity(MindMapNode overlay, String observerId,
                                    String targetAgentId, String tenantId) {
        var memories = memoryStore.query(
            RelationshipQuery.forPair(observerId, targetAgentId, tenantId)
                .withLimit(500));

        int positive = 0, negative = 0, neutral = 0;
        Instant latestTimestamp = null;

        for (var memory : memories) {
            Double pleasure = memory.pleasure();
            if (pleasure != null && pleasure > POSITIVE_THRESHOLD) positive++;
            else if (pleasure != null && pleasure < NEGATIVE_THRESHOLD) negative++;
            else neutral++;

            if (latestTimestamp == null || (memory.createdAt() != null
                && memory.createdAt().isAfter(latestTimestamp))) {
                latestTimestamp = memory.createdAt();
            }
        }

        long ticksSinceLastInteraction = 0;
        if (latestTimestamp != null && consolidationIntervalMs > 0) {
            long elapsedMs = Duration.between(latestTimestamp, Instant.now()).toMillis();
            ticksSinceLastInteraction = Math.max(0, elapsedMs / consolidationIntervalMs);
        }

        var stageConfig = configProvider.forAgent(observerId);
        double score = UserModelOrchestrator.computeFamiliarity(
            positive, negative, neutral, stageConfig, ticksSinceLastInteraction);
        String stage = stageConfig.resolveStage(score);

        mindMapStore.updateNode(overlay.id(),
            NodeUpdate.empty().withPropertiesToSet(Map.of(
                OverlayFamiliarityPropertyModel.FAMILIARITY_SCORE, String.valueOf(score),
                OverlayFamiliarityPropertyModel.FAMILIARITY_STAGE, stage,
                OverlayFamiliarityPropertyModel.FAMILIARITY_INTERACTION_COUNT,
                    String.valueOf(positive + negative + neutral))),
            tenantId);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=RelationshipStageIntegrationTest`
Expected: PASS

- [ ] **Step 5: Commit both repos**

blocks repo:
```bash
git -C /path/to/blocks add blocks-core/src/main/java/io/casehub/blocks/agentic/social/RelationshipStagePhase.java
git -C /path/to/blocks commit -m "feat(#70): RelationshipStagePhase consolidation @Priority(18)"
```

examples repo:
```bash
git add wacky-manor/src/test/java/io/casehub/blocks/agentic/social/OverlayFamiliarityPropertyModelTest.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/engine/RelationshipStageIntegrationTest.java
git commit -m "test(#70): RelationshipStagePhase integration tests Refs #70"
```

---

## Batch 2: Configuration + behavioral gating (wacky-manor)

### Task 3: SocialConfig stageConfig field + YAML parsing

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java` — add `RelationshipStageConfig stageConfig` field
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java` — parse `familiarity-thresholds` section
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` — add per-character thresholds for hooded-claw and penelope-pitstop
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java`

**Interfaces:**
- Consumes: `RelationshipStageConfig`, `StageTier` (blocks-core)
- Produces: `SocialConfig.stageConfig()` — returns `RelationshipStageConfig`

- [ ] **Step 1: Write the failing test**

Add test to `ManorSocialConfigLoaderTest`:

```java
@Test
void parsesCustomFamiliarityThresholds() {
    var configs = ManorSocialConfigLoader.load();
    var hoodedClaw = configs.get("hooded-claw");
    assertThat(hoodedClaw.stageConfig()).isNotNull();
    assertThat(hoodedClaw.stageConfig().tiers()).hasSize(5);
    assertThat(hoodedClaw.stageConfig().tiers().get(1).threshold()).isGreaterThan(0.2);
}

@Test
void defaultThresholdsWhenNotSpecified() {
    var configs = ManorSocialConfigLoader.load();
    var dickDastardly = configs.get("dick-dastardly");
    assertThat(dickDastardly.stageConfig())
        .isEqualTo(RelationshipStageConfig.defaults());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorSocialConfigLoaderTest#parsesCustomFamiliarityThresholds`
Expected: FAIL — `stageConfig()` method does not exist

- [ ] **Step 3: Add stageConfig to SocialConfig record**

Use `ide_edit_member` on `SocialConfig` to add the field:

Change the record declaration to:
```java
public record SocialConfig(
        List<GoalConfig> goals,
        List<Drive> drives,
        List<NormEntry> norms,
        List<InitialBelief> initialBeliefs,
        List<Relationship> relationships,
        Map<String, List<ReinforcementMapping>> reinforcement,
        RelationshipStageConfig stageConfig
)
```

Update the compact constructor to default `stageConfig`:
```java
if (stageConfig == null) {stageConfig = RelationshipStageConfig.defaults();}
```

Update `SocialConfig.empty()`:
```java
return new SocialConfig(List.of(), List.of(), List.of(), List.of(), List.of(), Map.of(), null);
```

- [ ] **Step 4: Fix all callers of SocialConfig constructor**

Search for all places that construct `SocialConfig` and add the `stageConfig` parameter. Key locations:
- `ManorSocialConfigLoader.parseCharacterConfig()` — add stageConfig parsing (see next step)
- Any test constructors — add `RelationshipStageConfig.defaults()` or `null`

Use `ide_find_references` on `SocialConfig` constructor to find all callers.

- [ ] **Step 5: Add familiarity-thresholds parsing to ManorSocialConfigLoader**

Add parsing at the end of `parseCharacterConfig()`, before the return statement:

```java
RelationshipStageConfig stageConfig = null;
if (raw.containsKey("familiarity-thresholds")) {
    var ftRaw = (Map<String, Object>) raw.get("familiarity-thresholds");
    var tiers = ftRaw.containsKey("tiers")
        ? ((List<Map<String, Object>>) ftRaw.get("tiers")).stream()
            .map(t -> new io.casehub.blocks.agentic.social.StageTier(
                (String) t.get("name"),
                ((Number) t.get("threshold")).doubleValue()))
            .toList()
        : RelationshipStageConfig.defaults().tiers();
    double decayRate = ftRaw.containsKey("decay-rate")
        ? ((Number) ftRaw.get("decay-rate")).doubleValue()
        : RelationshipStageConfig.defaults().decayRate();
    double positiveWeight = ftRaw.containsKey("positive-weight")
        ? ((Number) ftRaw.get("positive-weight")).doubleValue()
        : RelationshipStageConfig.defaults().positiveWeight();
    double negativeWeight = ftRaw.containsKey("negative-weight")
        ? ((Number) ftRaw.get("negative-weight")).doubleValue()
        : RelationshipStageConfig.defaults().negativeWeight();
    stageConfig = new RelationshipStageConfig(tiers, decayRate, positiveWeight, negativeWeight);
}
```

Update the return to include `stageConfig`:
```java
return new SocialConfig(goals, drives, norms, beliefs, relationships, reinforcement, stageConfig);
```

- [ ] **Step 6: Add YAML config for hooded-claw and penelope-pitstop**

In `social-config.yaml`, add under `hooded-claw:`:

```yaml
  familiarity-thresholds:
    tiers:
      - name: stranger
        threshold: 0.0
      - name: acquaintance
        threshold: 0.3
      - name: familiar
        threshold: 0.5
      - name: friend
        threshold: 0.75
      - name: confidant
        threshold: 0.9
    decay-rate: 0.02
    positive-weight: 0.8
    negative-weight: 0.7
```

Under `penelope-pitstop:`:

```yaml
  familiarity-thresholds:
    tiers:
      - name: stranger
        threshold: 0.0
      - name: acquaintance
        threshold: 0.15
      - name: familiar
        threshold: 0.3
      - name: friend
        threshold: 0.5
      - name: confidant
        threshold: 0.7
    positive-weight: 1.2
```

- [ ] **Step 7: Run tests to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java
git add wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml
git add wacky-manor/src/test/
git commit -m "feat(#70): SocialConfig.stageConfig + familiarity-thresholds YAML parsing Refs #70"
```

### Task 4: ManorContextStrategy stage gating + PerceptionTranslator stage rendering

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorContextStrategy.java` — add `shouldDisclose(String)`, `shouldCooperate(String)`, `stageOrdinal(String)`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/PerceptionTranslator.java` — add `stage` parameter, stage-gated rendering
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java` — read familiarity-stage from overlay, pass to PerceptionTranslator
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorContextStrategyTest.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/PerceptionTranslatorTest.java`

**Interfaces:**
- Consumes: `OverlayFamiliarityPropertyModel` (Task 1), `RelationshipStageConfig.defaults()` (blocks-core)
- Produces: `ManorContextStrategy.shouldDisclose(String stage) → boolean`, `ManorContextStrategy.shouldCooperate(String stage) → boolean`, `PerceptionTranslator.translate(PerspectivalComparison, PrincipalId, PrincipalId, String, String) → Optional<String>` (added `stage` parameter)

- [ ] **Step 1: Write the failing tests for ManorContextStrategy**

Add to `ManorContextStrategyTest`:

```java
@Test
void shouldDisclose_requiresFriendOrHigher() {
    var strategy = new ManorContextStrategy();
    assertThat(strategy.shouldDisclose("stranger")).isFalse();
    assertThat(strategy.shouldDisclose("acquaintance")).isFalse();
    assertThat(strategy.shouldDisclose("familiar")).isFalse();
    assertThat(strategy.shouldDisclose("friend")).isTrue();
    assertThat(strategy.shouldDisclose("confidant")).isTrue();
}

@Test
void shouldCooperate_requiresAcquaintanceOrHigher() {
    var strategy = new ManorContextStrategy();
    assertThat(strategy.shouldCooperate("stranger")).isFalse();
    assertThat(strategy.shouldCooperate("acquaintance")).isTrue();
    assertThat(strategy.shouldCooperate("familiar")).isTrue();
    assertThat(strategy.shouldCooperate("friend")).isTrue();
    assertThat(strategy.shouldCooperate("confidant")).isTrue();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorContextStrategyTest#shouldDisclose_requiresFriendOrHigher`
Expected: FAIL — method does not exist

- [ ] **Step 3: Implement stage gating in ManorContextStrategy**

Use `ide_insert_member` to add to `ManorContextStrategy`:

```java
private static final java.util.List<String> STAGE_ORDER = java.util.List.of(
    "stranger", "acquaintance", "familiar", "friend", "confidant");

public boolean shouldDisclose(String stage) {
    return stageOrdinal(stage) >= stageOrdinal("friend");
}

public boolean shouldCooperate(String stage) {
    return stageOrdinal(stage) >= stageOrdinal("acquaintance");
}

static int stageOrdinal(String stage) {
    int idx = STAGE_ORDER.indexOf(stage);
    return idx >= 0 ? idx : 0;
}
```

- [ ] **Step 4: Run ManorContextStrategy tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorContextStrategyTest`
Expected: ALL PASS

- [ ] **Step 5: Write the failing tests for PerceptionTranslator**

Add to `PerceptionTranslatorTest`:

```java
@Test
void strangerStageReturnsEmpty() {
    var comparison = buildComparison(-0.6, 0.4, 0.1, 0.1, 0.0, 0.0);
    assertThat(PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME, "stranger"))
        .isEmpty();
}

@Test
void acquaintanceStageReturnsDominantOnly() {
    var comparison = buildComparison(-0.6, 0.4, 0.1, 0.1, 0.0, 0.0);
    var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME, "acquaintance");
    assertThat(result).isPresent();
    assertThat(result.get()).doesNotContain("widening").doesNotContain("converging");
}

@Test
void friendStageIncludesTrajectory() {
    var comparison = buildDivergentComparison(-0.6, 0.4, 0.1, 0.1, 0.0, 0.0);
    var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME, "friend");
    assertThat(result).isPresent();
    assertThat(result.get()).contains("widening");
}
```

- [ ] **Step 6: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=PerceptionTranslatorTest#strangerStageReturnsEmpty`
Expected: FAIL — wrong method signature

- [ ] **Step 7: Add stage parameter to PerceptionTranslator.translate()**

Use `ide_edit_member` to change the method signature and add stage-gating logic:

```java
public static Optional<String> translate(
        PerspectivalComparison comparison,
        PrincipalId self,
        PrincipalId other,
        String otherName,
        String stage) {

    if ("stranger".equals(stage)) {
        return Optional.empty();
    }

    if (comparison.unassessedAgents().contains(self)
            || comparison.unassessedAgents().contains(other)) {
        return Optional.empty();
    }

    var pair = AgentPair.of(self, other);
    double distance = comparison.distances().distance(self, other);
    if (distance < DISTANCE_THRESHOLD) {
        return Optional.empty();
    }

    PadDimension dominant = dominantDimension(comparison, self, other);
    double diff = comparison.dimensionDifferences().get(dominant)
            .difference(self, other);

    String statement = switch (dominant) {
        case PLEASURE -> diff < 0
                ? otherName + " seems to view this more positively than you do"
                : "You feel more positively about " + otherName + " than they feel about themselves";
        case AROUSAL -> diff > 0
                ? "You're more alert around " + otherName + " than they seem to be"
                : otherName + " seems more on edge than you'd expect";
        case DOMINANCE -> diff > 0
                ? "You feel more in control around " + otherName + " than they do"
                : otherName + " seems more confident in this interaction than you are";
    };

    int stageOrd = ManorContextStrategy.stageOrdinal(stage);
    boolean includeTrajectory = stageOrd >= ManorContextStrategy.stageOrdinal("friend");

    if (includeTrajectory) {
        var trajectory = comparison.trajectoryAlignment();
        var agreement = trajectory.agreements().get(pair);
        if (agreement == TrendAgreement.DIVERGENT) {
            statement += " (and this gap is widening)";
        } else if (agreement == TrendAgreement.ALIGNED) {
            statement += " (though you're converging)";
        }
    }

    return Optional.of(otherName + " (" + stage + "): " + statement);
}
```

- [ ] **Step 8: Fix existing PerceptionTranslator callers**

Use `ide_find_references` on `PerceptionTranslator#translate` to find all callers.
The only production caller is `CharacterCognition.renderSocialAwareness()`.
Update it to read familiarity-stage from the overlay and pass it:

In `renderSocialAwareness()`, add overlay node lookup before the nearby-agent loop. Read familiarity-stage for each nearby agent from the people subgraph overlays (same pattern as `renderTrustSections()`):

```java
// Build stage map from overlay nodes
var stageMap = new java.util.HashMap<String, String>();
if (mindMapStore != null && tenantId != null) {
    var subgraphs = mindMapStore.listSubgraphs(tenantId);
    var peopleSg = subgraphs.stream()
        .filter(sg -> "people".equals(sg.name())).findFirst();
    if (peopleSg.isPresent()) {
        var nodes = mindMapStore.nodesIn(peopleSg.get().id(), tenantId);
        var sharedNodes = new java.util.HashMap<String, String>();
        for (var node : nodes) {
            if (!node.traits().contains("overlay")) {
                node.property("agentId").ifPresent(aid -> sharedNodes.put(node.id(), aid));
            }
        }
        for (var node : nodes) {
            if (!node.traits().contains("overlay")) continue;
            if (!agentId.equals(node.property(io.casehub.neocortex.mindmap.OverlayRef.AGENT_ID).orElse(null))) continue;
            var targetNodeId = io.casehub.neocortex.mindmap.OverlayRef.sharedNodeId(node).orElse(null);
            if (targetNodeId == null) continue;
            var targetId = sharedNodes.get(targetNodeId);
            if (targetId == null) continue;
            var stage = node.property(io.casehub.blocks.agentic.social.OverlayFamiliarityPropertyModel.FAMILIARITY_STAGE).orElse("stranger");
            stageMap.put(targetId, stage);
        }
    }
}
```

Then in the for loop, pass stage to `PerceptionTranslator.translate()`:

```java
String stage = stageMap.getOrDefault(nearbyId, "stranger");
PerceptionTranslator.translate(comparison, selfPrincipal, otherPrincipal, otherName, stage)
        .ifPresent(lines::add);
```

- [ ] **Step 9: Fix existing test calls**

Update all existing `PerceptionTranslator.translate()` test calls to include a stage parameter. Use `"friend"` for existing tests that expect full output, to maintain backward compatibility of test assertions.

- [ ] **Step 10: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: ALL PASS

- [ ] **Step 11: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorContextStrategy.java
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/PerceptionTranslator.java
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java
git add wacky-manor/src/test/
git commit -m "feat(#70): stage-gated behavioral gating + perception rendering Refs #70"
```

---

## References

- [2026-09-16-relationship-stage-thresholds-design.md] — design spec this plan implements
- [RelationshipStageConfig.java] (blocks-core) — existing 5-tier config
- [UserModelOrchestrator.computeFamiliarity()] (blocks-core:110-121) — existing formula
- [DriveAdaptationPhase.java] (blocks-core) — ConsolidationPhase pattern
- [DriveAdaptationIntegrationTest.java] (wacky-manor) — integration test pattern
- [ManorCognitiveSeeder.seedPeople()] (wacky-manor:27-76) — overlay creation
- [CharacterCognition.renderTrustSections()] (wacky-manor:131-187) — overlay reading pattern
- [PerceptionTranslator.translate()] (wacky-manor:17-59) — PAD-based perception
- [FamiliarityScoreTest.java] (blocks) — existing computeFamiliarity tests
- GitHub #70
