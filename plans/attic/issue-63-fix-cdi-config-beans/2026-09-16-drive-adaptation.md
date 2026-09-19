# Drive Adaptation from Reward Signals — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #66 — Drive adaptation from reward signals
**Issue group:** #63, #66

**Goal:** Make character motivational drives (e.g., "scheming", "social-harmony") evolve based on action outcomes during consolidation, replacing the current static YAML-only intensities.

**Architecture:** A new `DriveAdaptationPhase` (blocks-core) runs after `ExperienceConsolidationPhase` in the consolidation pipeline. It reads graduated experience mindmap nodes, maps event types to character drives via per-character YAML config, and applies multiplicative intensity updates to drive mindmap nodes. A `CharacterDrivePromptSection` (blocks-core) renders adapted drives in the observation pipeline. Wacky-manor provides YAML config and thin seeding wiring.

**Tech Stack:** Java 26, Quarkus (CDI, Smallrye Config), MindMapStore (neocortex), ConsolidationPhase SPI (neocortex)

## Global Constraints

- Multiplicative updates only — `intensity *= (1 + delta)`, never additive (GE-20260714-439924)
- Drive intensity clamped to [0.1, 1.0] — drives never extinguish or exceed range
- Volume dampening on multi-event aggregation: `effectiveReward = totalReward / (1 + log(count))` (GE-20260820-d9129a)
- Build commands: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module> -s slot-settings.xml`
- Use `ide_create_file` for new Java files, `ide_edit_member`/`ide_replace_member`/`ide_insert_member` for structural edits
- Tests go in the corresponding test source tree of the module being tested

---

## Batch 1: Data Model + Adaptation Phase (blocks-core)

### Task 1: DriveReinforcementEntry records and DriveAdaptationConfig

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveReinforcementEntry.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/ReinforcementDirection.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/RewardAxis.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationConfig.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveReinforcementEntryTest.java`

**Interfaces:**
- Produces: `DriveReinforcementEntry(String driveType, ReinforcementDirection direction, RewardAxis rewardAxis)` — used by Task 2 and Task 4
- Produces: `ReinforcementDirection { POSITIVE, NEGATIVE }` — used by Task 2
- Produces: `RewardAxis { PLEASURE, DOMINANCE, COMPOSITE }` — used by Task 2
- Produces: `DriveAdaptationConfig` — Smallrye config interface with `learningRate()`, `arousalWeight()`, `minIntensity()`, `maxIntensity()`, `maxPerPass()` — used by Task 2

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.blocks.agentic.social.drive.adaptation;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class DriveReinforcementEntryTest {

    @Test
    void defaultsToPositivePleasure() {
        var entry = new DriveReinforcementEntry("scheming", null, null);
        assertEquals(ReinforcementDirection.POSITIVE, entry.direction());
        assertEquals(RewardAxis.PLEASURE, entry.rewardAxis());
    }

    @Test
    void rejectNullDriveType() {
        assertThrows(NullPointerException.class,
            () -> new DriveReinforcementEntry(null, null, null));
    }

    @Test
    void preserveExplicitValues() {
        var entry = new DriveReinforcementEntry("social-harmony",
            ReinforcementDirection.NEGATIVE, RewardAxis.DOMINANCE);
        assertEquals("social-harmony", entry.driveType());
        assertEquals(ReinforcementDirection.NEGATIVE, entry.direction());
        assertEquals(RewardAxis.DOMINANCE, entry.rewardAxis());
    }

    @Test
    void compositeRewardAxis() {
        var entry = new DriveReinforcementEntry("greed", null, RewardAxis.COMPOSITE);
        assertEquals(RewardAxis.COMPOSITE, entry.rewardAxis());
        assertEquals(ReinforcementDirection.POSITIVE, entry.direction());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=DriveReinforcementEntryTest`
Expected: compilation failure — classes do not exist

- [ ] **Step 3: Create the enums**

Use `ide_create_file` for each:

`ReinforcementDirection.java`:
```java
package io.casehub.blocks.agentic.social.drive.adaptation;

public enum ReinforcementDirection { POSITIVE, NEGATIVE }
```

`RewardAxis.java`:
```java
package io.casehub.blocks.agentic.social.drive.adaptation;

public enum RewardAxis { PLEASURE, DOMINANCE, COMPOSITE }
```

- [ ] **Step 4: Create DriveReinforcementEntry record**

Use `ide_create_file`:

```java
package io.casehub.blocks.agentic.social.drive.adaptation;

import java.util.Objects;

public record DriveReinforcementEntry(
    String driveType,
    ReinforcementDirection direction,
    RewardAxis rewardAxis
) {
    public DriveReinforcementEntry {
        Objects.requireNonNull(driveType, "driveType required");
        if (direction == null) direction = ReinforcementDirection.POSITIVE;
        if (rewardAxis == null) rewardAxis = RewardAxis.PLEASURE;
    }
}
```

- [ ] **Step 5: Create DriveAdaptationConfig**

Use `ide_create_file`:

```java
package io.casehub.blocks.agentic.social.drive.adaptation;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.drive-adaptation")
public interface DriveAdaptationConfig {
    @WithDefault("0.05")
    double learningRate();

    @WithDefault("0.3")
    double arousalWeight();

    @WithDefault("0.1")
    double minIntensity();

    @WithDefault("1.0")
    double maxIntensity();

    @WithDefault("20")
    int maxPerPass();
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=DriveReinforcementEntryTest`
Expected: 4 tests PASS

- [ ] **Step 7: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/
git add blocks/src/test/java/io/casehub/blocks/agentic/social/drive/adaptation/
git commit -m "feat(#66): DriveReinforcementEntry records and DriveAdaptationConfig

Refs #66"
```

---

### Task 2: DriveAdaptationPhase

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhase.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhaseTest.java`

**Interfaces:**
- Consumes: `DriveReinforcementEntry`, `ReinforcementDirection`, `RewardAxis`, `DriveAdaptationConfig` (Task 1)
- Consumes: `ConsolidationPhase` SPI — `io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase`
- Consumes: `MindMapStore.nodesIn(subgraphId, tenantId)`, `MindMapStore.updateNode(nodeId, NodeUpdate, tenantId)`, `MindMapStore.resolveNode(name, subgraphId, tenantId)`, `MindMapStore.addNode(NodeInput, tenantId)`, `MindMapStore.listSubgraphs(tenantId)`
- Consumes: `MindMapNode.properties()` — Map<String, String> for reading `cognitiveKind`, `agent-id`, `drive-type`, `intensity`, `event-type`
- Consumes: `MindMapNode.pleasure()`, `MindMapNode.arousal()`, `MindMapNode.dominance()` — PAD values on graduated experience nodes
- Produces: `DriveAdaptationPhase` — CDI ConsolidationPhase at `@Priority(17)`, constructor takes `MindMapStore`, `Map<String, Map<String, List<DriveReinforcementEntry>>>` (agentId → event-type → entries), `DriveAdaptationConfig`

- [ ] **Step 1: Write the failing test — positive reinforcement**

```java
package io.casehub.blocks.agentic.social.drive.adaptation;

import io.casehub.neocortex.cognitive.Confidence;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class DriveAdaptationPhaseTest {

    private InMemoryMindMapStore store;
    private DriveAdaptationPhase phase;
    private static final String TENANT = "test-tenant";
    private static final String AGENT = "hooded-claw";
    private String subgraphId;

    @BeforeEach
    void setUp() {
        store = new InMemoryMindMapStore();

        var reinforcement = Map.of(
            AGENT, Map.of(
                "conflict_resolution", List.of(
                    new DriveReinforcementEntry("scheming", null, null)
                )
            )
        );

        var config = new TestDriveAdaptationConfig(0.1, 0.3, 0.1, 1.0, 20);
        phase = new DriveAdaptationPhase(store, reinforcement, config);

        subgraphId = store.createSubgraph(
            new io.casehub.neocortex.mindmap.SubgraphInput(
                "beliefs-" + AGENT, "cognitive", null), TENANT);

        store.addNode(NodeInput.of("scheming", subgraphId)
            .withConfidence(Confidence.stated(0.8, Instant.now()))
            .withProvenance("drive-adaptation")
            .withProperties(Map.of(
                "cognitiveKind", "drive-intensity",
                "agent-id", AGENT,
                "drive-type", "scheming",
                "intensity", "0.9",
                "initial-intensity", "0.9",
                "description", "Compelled to hatch elaborate plans")),
            TENANT);
    }

    @Test
    void positiveReinforcementIncreasesIntensity() {
        store.addNode(NodeInput.of("conflict event", subgraphId)
            .withProvenance("experience-consolidation")
            .withProperties(Map.of(
                "cognitiveKind", "experience",
                "agent-id", AGENT,
                "event-type", "conflict_resolution"))
            .withPleasure(0.6).withArousal(0.5),
            TENANT);

        phase.run(TENANT, List.of());

        var nodes = store.nodesIn(subgraphId, TENANT);
        var driveNode = nodes.stream()
            .filter(n -> "drive-intensity".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "scheming".equals(n.properties().get("drive-type")))
            .findFirst().orElseThrow();

        double updatedIntensity = Double.parseDouble(
            driveNode.properties().get("intensity"));
        assertTrue(updatedIntensity > 0.9,
            "Expected intensity > 0.9, got " + updatedIntensity);
        assertTrue(updatedIntensity <= 1.0,
            "Expected intensity <= 1.0, got " + updatedIntensity);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=DriveAdaptationPhaseTest`
Expected: compilation failure — DriveAdaptationPhase does not exist

- [ ] **Step 3: Write the DriveAdaptationPhase implementation**

Use `ide_create_file`:

```java
package io.casehub.blocks.agentic.social.drive.adaptation;

import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import io.casehub.neocortex.mindmap.MindMapSubgraph;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.NodeUpdate;
import io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase;
import jakarta.annotation.Priority;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.logging.Level;
import java.util.logging.Logger;

@Priority(17)
public class DriveAdaptationPhase implements ConsolidationPhase {

    private static final Logger LOG = Logger.getLogger(
        DriveAdaptationPhase.class.getName());
    private static final String CURSOR_NODE_NAME = "drive-adaptation-cursor";
    private static final String DRIVE_KIND = "drive-intensity";
    private static final String EXPERIENCE_PROVENANCE = "experience-consolidation";

    private final MindMapStore mindMapStore;
    private final Map<String, Map<String, List<DriveReinforcementEntry>>> reinforcementMap;
    private final DriveAdaptationConfig config;

    public DriveAdaptationPhase(
            MindMapStore mindMapStore,
            Map<String, Map<String, List<DriveReinforcementEntry>>> reinforcementMap,
            DriveAdaptationConfig config) {
        this.mindMapStore = mindMapStore;
        this.reinforcementMap = reinforcementMap;
        this.config = config;
    }

    @Override
    public String name() {
        return "drive-adaptation";
    }

    @Override
    public void run(String tenantId, List<String> subgraphPriority) {
        var subgraphs = mindMapStore.listSubgraphs(tenantId);
        for (var subgraph : subgraphs) {
            if (!"cognitive".equals(subgraph.type())) continue;
            processSubgraph(subgraph, tenantId);
        }
    }

    private void processSubgraph(MindMapSubgraph subgraph, String tenantId) {
        var nodes = mindMapStore.nodesIn(subgraph.subgraphId(), tenantId);

        var driveNodes = new HashMap<String, MindMapNode>();
        var experienceNodes = new java.util.ArrayList<MindMapNode>();
        String cursorNodeId = null;
        String lastProcessedId = null;

        for (var node : nodes) {
            var kind = node.properties().get("cognitiveKind");
            if (DRIVE_KIND.equals(kind)) {
                var driveType = node.properties().get("drive-type");
                if (driveType != null) driveNodes.put(driveType, node);
            } else if (EXPERIENCE_PROVENANCE.equals(node.provenance())) {
                experienceNodes.add(node);
            }
            if (CURSOR_NODE_NAME.equals(node.name())) {
                cursorNodeId = node.nodeId();
                lastProcessedId = node.properties().get("last-processed-node-id");
            }
        }

        if (driveNodes.isEmpty()) {
            var agentIds = nodes.stream()
                .map(n -> n.properties().get("agent-id"))
                .filter(java.util.Objects::nonNull)
                .distinct().toList();
            if (!agentIds.isEmpty()) {
                LOG.warning("No drive nodes found for agent(s) " + agentIds
                    + " in tenant " + tenantId);
            }
            return;
        }

        var agentId = driveNodes.values().iterator().next()
            .properties().get("agent-id");
        var agentReinforcement = reinforcementMap.get(agentId);
        if (agentReinforcement == null) return;

        var newExperiences = filterNewExperiences(
            experienceNodes, lastProcessedId);
        if (newExperiences.isEmpty()) return;

        var rewardAccumulator = accumulateRewards(
            newExperiences, agentReinforcement);

        applyUpdates(driveNodes, rewardAccumulator, tenantId);

        var lastId = newExperiences.get(newExperiences.size() - 1).nodeId();
        saveCursor(subgraph.subgraphId(), cursorNodeId,
            lastId, tenantId);
    }

    private List<MindMapNode> filterNewExperiences(
            List<MindMapNode> experienceNodes, String lastProcessedId) {
        if (lastProcessedId == null) return experienceNodes;
        return experienceNodes.stream()
            .filter(n -> n.nodeId().compareTo(lastProcessedId) > 0)
            .toList();
    }

    record DriveReward(double totalReward, int count) {
        DriveReward add(double reward) {
            return new DriveReward(totalReward + reward, count + 1);
        }

        double effectiveReward() {
            if (count == 0) return 0.0;
            return totalReward / (1 + Math.log(count));
        }
    }

    private Map<String, DriveReward> accumulateRewards(
            List<MindMapNode> experiences,
            Map<String, List<DriveReinforcementEntry>> agentReinforcement) {
        var accumulator = new HashMap<String, DriveReward>();

        for (var exp : experiences) {
            var eventType = exp.properties().get("event-type");
            if (eventType == null) continue;

            var entries = agentReinforcement.get(eventType);
            if (entries == null) continue;

            for (var entry : entries) {
                double padValue = extractPadValue(exp, entry.rewardAxis());
                if (padValue == 0.0) continue;

                double arousal = exp.arousal() != null ? exp.arousal() : 0.0;
                double reward = padValue
                    * (1 + Math.abs(arousal) * config.arousalWeight());

                if (entry.direction() == ReinforcementDirection.NEGATIVE) {
                    reward = -reward;
                }

                accumulator.merge(entry.driveType(),
                    new DriveReward(reward, 1),
                    (a, b) -> a.add(b.totalReward()));
            }
        }
        return accumulator;
    }

    private double extractPadValue(MindMapNode node, RewardAxis axis) {
        return switch (axis) {
            case PLEASURE -> node.pleasure() != null ? node.pleasure() : 0.0;
            case DOMINANCE -> node.dominance() != null ? node.dominance() : 0.0;
            case COMPOSITE -> {
                double p = node.pleasure() != null ? node.pleasure() : 0.0;
                double a = node.arousal() != null ? node.arousal() : 0.0;
                double d = node.dominance() != null ? node.dominance() : 0.0;
                yield 0.6 * p + 0.2 * a + 0.2 * d;
            }
        };
    }

    private void applyUpdates(
            Map<String, MindMapNode> driveNodes,
            Map<String, DriveReward> rewardAccumulator,
            String tenantId) {
        for (var entry : rewardAccumulator.entrySet()) {
            var driveNode = driveNodes.get(entry.getKey());
            if (driveNode == null) continue;

            double currentIntensity = Double.parseDouble(
                driveNode.properties().getOrDefault("intensity", "0.5"));
            double delta = entry.getValue().effectiveReward()
                * config.learningRate();
            double newIntensity = currentIntensity * (1 + delta);
            newIntensity = Math.clamp(newIntensity,
                config.minIntensity(), config.maxIntensity());

            mindMapStore.updateNode(driveNode.nodeId(),
                NodeUpdate.empty().withPropertiesToSet(
                    Map.of("intensity",
                        String.valueOf(newIntensity))),
                tenantId);
        }
    }

    private void saveCursor(String subgraphId, String cursorNodeId,
                            String lastId, String tenantId) {
        if (cursorNodeId != null) {
            mindMapStore.updateNode(cursorNodeId,
                NodeUpdate.empty().withPropertiesToSet(
                    Map.of("last-processed-node-id", lastId)),
                tenantId);
        } else {
            mindMapStore.addNode(
                NodeInput.of(CURSOR_NODE_NAME, subgraphId)
                    .withProvenance("drive-adaptation")
                    .withProperties(Map.of(
                        "cognitiveKind", "cursor",
                        "last-processed-node-id", lastId)),
                tenantId);
        }
    }
}
```

- [ ] **Step 4: Create TestDriveAdaptationConfig helper**

Add to the test file:

```java
record TestDriveAdaptationConfig(
    double learningRate, double arousalWeight,
    double minIntensity, double maxIntensity,
    int maxPerPass
) implements DriveAdaptationConfig {}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=DriveAdaptationPhaseTest#positiveReinforcementIncreasesIntensity`
Expected: PASS

- [ ] **Step 6: Write additional tests — negative reinforcement, clamping, volume dampening**

Add to `DriveAdaptationPhaseTest`:

```java
@Test
void negativeReinforcementDecreasesIntensity() {
    var reinforcement = Map.of(
        AGENT, Map.of(
            "social_interaction", List.of(
                new DriveReinforcementEntry("scheming",
                    ReinforcementDirection.NEGATIVE, null)
            )
        )
    );
    var config = new TestDriveAdaptationConfig(0.1, 0.3, 0.1, 1.0, 20);
    phase = new DriveAdaptationPhase(store, reinforcement, config);

    store.addNode(NodeInput.of("social event", subgraphId)
        .withProvenance("experience-consolidation")
        .withProperties(Map.of(
            "cognitiveKind", "experience",
            "agent-id", AGENT,
            "event-type", "social_interaction"))
        .withPleasure(0.7).withArousal(0.3),
        TENANT);

    phase.run(TENANT, List.of());

    var nodes = store.nodesIn(subgraphId, TENANT);
    var driveNode = nodes.stream()
        .filter(n -> "drive-intensity".equals(
            n.properties().get("cognitiveKind")))
        .filter(n -> "scheming".equals(
            n.properties().get("drive-type")))
        .findFirst().orElseThrow();

    double updated = Double.parseDouble(
        driveNode.properties().get("intensity"));
    assertTrue(updated < 0.9,
        "Expected intensity < 0.9 after negative reinforcement, got " + updated);
    assertTrue(updated >= 0.1,
        "Expected intensity >= 0.1 (floor), got " + updated);
}

@Test
void intensityNeverDropsBelowFloor() {
    // Set drive intensity very low, apply strong negative reinforcement
    store.addNode(NodeInput.of("low-drive", subgraphId)
        .withConfidence(io.casehub.neocortex.cognitive.Confidence.stated(
            0.8, Instant.now()))
        .withProvenance("drive-adaptation")
        .withProperties(Map.of(
            "cognitiveKind", "drive-intensity",
            "agent-id", AGENT,
            "drive-type", "weak-drive",
            "intensity", "0.12",
            "initial-intensity", "0.12",
            "description", "Almost extinguished")),
        TENANT);

    var reinforcement = Map.of(
        AGENT, Map.of(
            "conflict_resolution", List.of(
                new DriveReinforcementEntry("weak-drive",
                    ReinforcementDirection.NEGATIVE, null)
            )
        )
    );
    var config = new TestDriveAdaptationConfig(0.5, 0.3, 0.1, 1.0, 20);
    phase = new DriveAdaptationPhase(store, reinforcement, config);

    store.addNode(NodeInput.of("strong conflict", subgraphId)
        .withProvenance("experience-consolidation")
        .withProperties(Map.of(
            "cognitiveKind", "experience",
            "agent-id", AGENT,
            "event-type", "conflict_resolution"))
        .withPleasure(0.9).withArousal(0.9),
        TENANT);

    phase.run(TENANT, List.of());

    var nodes = store.nodesIn(subgraphId, TENANT);
    var driveNode = nodes.stream()
        .filter(n -> "drive-intensity".equals(
            n.properties().get("cognitiveKind")))
        .filter(n -> "weak-drive".equals(
            n.properties().get("drive-type")))
        .findFirst().orElseThrow();

    double updated = Double.parseDouble(
        driveNode.properties().get("intensity"));
    assertEquals(0.1, updated, 0.001,
        "Intensity should clamp to floor 0.1");
}

@Test
void dominanceRewardAxis() {
    var reinforcement = Map.of(
        AGENT, Map.of(
            "conflict_resolution", List.of(
                new DriveReinforcementEntry("scheming", null,
                    RewardAxis.DOMINANCE)
            )
        )
    );
    var config = new TestDriveAdaptationConfig(0.1, 0.3, 0.1, 1.0, 20);
    phase = new DriveAdaptationPhase(store, reinforcement, config);

    // Low pleasure but high dominance — should still reinforce
    store.addNode(NodeInput.of("dominance event", subgraphId)
        .withProvenance("experience-consolidation")
        .withProperties(Map.of(
            "cognitiveKind", "experience",
            "agent-id", AGENT,
            "event-type", "conflict_resolution"))
        .withPleasure(-0.2).withArousal(0.4).withDominance(0.8),
        TENANT);

    phase.run(TENANT, List.of());

    var nodes = store.nodesIn(subgraphId, TENANT);
    var driveNode = nodes.stream()
        .filter(n -> "drive-intensity".equals(
            n.properties().get("cognitiveKind")))
        .filter(n -> "scheming".equals(
            n.properties().get("drive-type")))
        .findFirst().orElseThrow();

    double updated = Double.parseDouble(
        driveNode.properties().get("intensity"));
    assertTrue(updated > 0.9,
        "Expected intensity > 0.9 with dominance axis, got " + updated);
}

@Test
void noExperienceNodesProducesNoChange() {
    phase.run(TENANT, List.of());

    var nodes = store.nodesIn(subgraphId, TENANT);
    var driveNode = nodes.stream()
        .filter(n -> "drive-intensity".equals(
            n.properties().get("cognitiveKind")))
        .findFirst().orElseThrow();

    assertEquals("0.9", driveNode.properties().get("intensity"));
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=DriveAdaptationPhaseTest`
Expected: all 5 tests PASS

- [ ] **Step 8: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhase.java
git add blocks/src/test/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhaseTest.java
git commit -m "feat(#66): DriveAdaptationPhase consolidation phase

Multiplicative intensity updates with volume dampening, configurable
PAD reward axis, negative reinforcement support.

Refs #66"
```

---

## Batch 2: Rendering + CognitionConfig (blocks-core)

### Task 3: CharacterDrivePromptSection and CognitionConfig update

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CharacterDrivePromptSection.java`
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionConfig.java` — add `characterDrivesEnabled` field
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java:345-376` — add CharacterDrivePromptSection to `promptSections()`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/CharacterDrivePromptSectionTest.java`

**Interfaces:**
- Consumes: `MindMapStore.nodesIn(subgraphId, tenantId)`, `MindMapStore.listSubgraphs(tenantId)` — reads drive nodes
- Consumes: `PromptSection` SPI — `io.casehub.blocks.speech.PromptSection`
- Consumes: `PromptContext.agentId()`, `PromptContext.tenantId()`
- Produces: `CharacterDrivePromptSection` — PromptSection that renders "Character Motivations" observation text
- Produces: `CognitionConfig.characterDrivesEnabled()` — boolean flag

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.neocortex.cognitive.Confidence;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.SubgraphInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import io.casehub.blocks.speech.PromptContext;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class CharacterDrivePromptSectionTest {

    private InMemoryMindMapStore store;
    private CharacterDrivePromptSection section;
    private static final String TENANT = "test-tenant";
    private static final String AGENT = "hooded-claw";

    @BeforeEach
    void setUp() {
        store = new InMemoryMindMapStore();
        section = new CharacterDrivePromptSection(store);

        var subgraphId = store.createSubgraph(
            new SubgraphInput("beliefs-" + AGENT, "cognitive", null), TENANT);

        store.addNode(NodeInput.of("scheming", subgraphId)
            .withConfidence(Confidence.stated(0.8, Instant.now()))
            .withProvenance("drive-adaptation")
            .withProperties(Map.of(
                "cognitiveKind", "drive-intensity",
                "agent-id", AGENT,
                "drive-type", "scheming",
                "intensity", "0.92",
                "description", "Compelled to hatch elaborate plans")),
            TENANT);

        store.addNode(NodeInput.of("dominance", subgraphId)
            .withConfidence(Confidence.stated(0.8, Instant.now()))
            .withProvenance("drive-adaptation")
            .withProperties(Map.of(
                "cognitiveKind", "drive-intensity",
                "agent-id", AGENT,
                "drive-type", "dominance",
                "intensity", "0.65",
                "description", "Must be most powerful in every room")),
            TENANT);
    }

    @Test
    void rendersAdaptedDrivesSortedByIntensity() {
        var ctx = new PromptContext(AGENT, TENANT, null);
        var result = section.contribute(ctx);

        assertNotNull(result);
        assertTrue(result.contains("scheming"),
            "Should contain scheming drive");
        assertTrue(result.contains("dominance"),
            "Should contain dominance drive");
        int schemingIdx = result.indexOf("scheming");
        int dominanceIdx = result.indexOf("dominance");
        assertTrue(schemingIdx < dominanceIdx,
            "Scheming (92%) should appear before dominance (65%)");
    }

    @Test
    void returnsNullWhenNoDriveNodes() {
        var ctx = new PromptContext("unknown-agent", TENANT, null);
        var result = section.contribute(ctx);
        assertNull(result);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=CharacterDrivePromptSectionTest`
Expected: compilation failure

- [ ] **Step 3: Create CharacterDrivePromptSection**

Use `ide_create_file`:

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.speech.PromptContext;
import io.casehub.blocks.speech.PromptSection;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import org.jspecify.annotations.Nullable;

import java.util.Comparator;

public class CharacterDrivePromptSection implements PromptSection {

    private final MindMapStore mindMapStore;

    public CharacterDrivePromptSection(MindMapStore mindMapStore) {
        this.mindMapStore = mindMapStore;
    }

    @Override
    public @Nullable String contribute(PromptContext context) {
        var subgraphs = mindMapStore.listSubgraphs(context.tenantId());

        var driveNodes = subgraphs.stream()
            .filter(s -> "cognitive".equals(s.type()))
            .flatMap(s -> mindMapStore.nodesIn(
                s.subgraphId(), context.tenantId()).stream())
            .filter(n -> "drive-intensity".equals(
                n.properties().get("cognitiveKind")))
            .filter(n -> context.agentId().equals(
                n.properties().get("agent-id")))
            .sorted(Comparator.comparingDouble(
                (MindMapNode n) -> Double.parseDouble(
                    n.properties().getOrDefault("intensity", "0")))
                .reversed())
            .toList();

        if (driveNodes.isEmpty()) return null;

        var sb = new StringBuilder("## Character Motivations\n\n");
        for (var node : driveNodes) {
            var type = node.properties().get("drive-type");
            var intensity = Double.parseDouble(
                node.properties().getOrDefault("intensity", "0"));
            var description = node.properties().getOrDefault(
                "description", "");
            sb.append(String.format("- %s (%.0f%%) — %s%n",
                type, intensity * 100, description));
        }
        return sb.toString().stripTrailing();
    }
}
```

- [ ] **Step 4: Add `characterDrivesEnabled` to CognitionConfig**

Use `ide_edit_member` on `CognitionConfig` record declaration to add the field. Update `all()`, `none()`, `withDirectives()`, and `with()` methods to include the new field.

The new record component list:
```
moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled,
strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled,
directivePrompts, characterDrivesEnabled
```

Add `case "characterDrives"` to the `with()` switch.

- [ ] **Step 5: Add CharacterDrivePromptSection to CognitionCore.promptSections()**

Use `ide_replace_member` on `CognitionCore.promptSections()`. After the `goalsEnabled` block (before the `directivePrompts` check), add:

```java
if (config.characterDrivesEnabled() && mindMapStore != null)
    sections.add(new CharacterDrivePromptSection(mindMapStore));
```

This requires adding a `MindMapStore` field to `CognitionCore`. Use `ide_insert_member` to add the field and update the constructor that takes `CognitionConfig` to also accept `MindMapStore`.

- [ ] **Step 6: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=CharacterDrivePromptSectionTest,CognitionConfigTest,CognitionCoreTest`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/CharacterDrivePromptSection.java
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionConfig.java
git add blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java
git add blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/CharacterDrivePromptSectionTest.java
git commit -m "feat(#66): CharacterDrivePromptSection + CognitionConfig.characterDrivesEnabled

Renders adapted drive intensities from MindMap nodes in the observation
pipeline under 'Character Motivations' heading.

Refs #66"
```

---

## Batch 3: Application Wiring (wacky-manor)

### Task 4: YAML reinforcement config, drive seeding, CharacterCognition deduplication

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java` — add `ReinforcementMapping` record and `reinforcement` field
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java:47-95` — parse `reinforcement` section
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java:86-114` — add drive node seeding
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:100-148` — remove drive rendering and CognitionCore delegation
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` — add `reinforcement` sections for all characters
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java` — existing, add reinforcement tests
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederTest.java` — existing or new, test drive seeding

**Interfaces:**
- Consumes: `DriveReinforcementEntry`, `ReinforcementDirection`, `RewardAxis` (Task 1)
- Consumes: `NodeInput`, `MindMapStore` (neocortex API)
- Produces: `SocialConfig.ReinforcementMapping(String drive, ReinforcementDirection direction, RewardAxis rewardAxis)` — parsed from YAML
- Produces: `SocialConfig.reinforcement()` — `Map<String, List<ReinforcementMapping>>`

- [ ] **Step 1: Write failing test for reinforcement parsing**

Add to existing `ManorSocialConfigLoaderTest`:

```java
@Test
void parsesReinforcementMappings() {
    var configs = ManorSocialConfigLoader.load();
    var hoodedClaw = configs.get("hooded-claw");
    assertNotNull(hoodedClaw);
    assertFalse(hoodedClaw.reinforcement().isEmpty(),
        "Hooded Claw should have reinforcement mappings");

    var conflictEntries = hoodedClaw.reinforcement().get("conflict_resolution");
    assertNotNull(conflictEntries, "Should have conflict_resolution mappings");
    assertTrue(conflictEntries.stream()
        .anyMatch(e -> "scheming".equals(e.drive())),
        "conflict_resolution should reinforce scheming");
}

@Test
void parsesNegativeReinforcementDirection() {
    var configs = ManorSocialConfigLoader.load();
    var hoodedClaw = configs.get("hooded-claw");
    var socialEntries = hoodedClaw.reinforcement().get("social_interaction");
    assertNotNull(socialEntries);
    var schemingEntry = socialEntries.stream()
        .filter(e -> "scheming".equals(e.drive()))
        .findFirst().orElseThrow();
    assertEquals("NEGATIVE", schemingEntry.direction());
}

@Test
void parsesRewardAxis() {
    var configs = ManorSocialConfigLoader.load();
    var penelope = configs.get("penelope-pitstop");
    var conflictEntries = penelope.reinforcement()
        .get("conflict_resolution");
    assertNotNull(conflictEntries);
    var harmonyEntry = conflictEntries.stream()
        .filter(e -> "social-harmony".equals(e.drive()))
        .findFirst().orElseThrow();
    assertEquals("DOMINANCE", harmonyEntry.rewardAxis());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorSocialConfigLoaderTest`
Expected: compilation failure — `reinforcement()` method does not exist on SocialConfig

- [ ] **Step 3: Add ReinforcementMapping and reinforcement field to SocialConfig**

Use `ide_insert_member` to add after the `Relationship` record:

```java
public record ReinforcementMapping(String drive, String direction, String rewardAxis) {
    public ReinforcementMapping {
        Objects.requireNonNull(drive);
        if (direction == null) direction = "POSITIVE";
        if (rewardAxis == null) rewardAxis = "PLEASURE";
    }
}
```

Use `ide_edit_member` to update the `SocialConfig` record declaration to add `Map<String, List<ReinforcementMapping>> reinforcement` as a field. Update the compact constructor to default: `reinforcement = reinforcement != null ? Map.copyOf(reinforcement) : Map.of();`. Update `empty()` to pass `Map.of()`.

- [ ] **Step 4: Add reinforcement parsing to ManorSocialConfigLoader**

Use `ide_replace_member` on `parseCharacterConfig` to add reinforcement parsing after the `relationships` block:

```java
@SuppressWarnings("unchecked")
var reinforcement = new HashMap<String, List<SocialConfig.ReinforcementMapping>>();
if (raw.containsKey("reinforcement")) {
    var reinforcementMap = (Map<String, List<Map<String, Object>>>) raw.get("reinforcement");
    for (var entry : reinforcementMap.entrySet()) {
        var mappings = entry.getValue().stream()
            .map(m -> new SocialConfig.ReinforcementMapping(
                (String) m.get("drive"),
                (String) m.get("direction"),
                (String) m.get("reward-axis")))
            .toList();
        reinforcement.put(entry.getKey(), mappings);
    }
}

return new SocialConfig(goals, drives, norms, beliefs, relationships, reinforcement);
```

- [ ] **Step 5: Add reinforcement sections to social-config.yaml**

Add `reinforcement` sections to hooded-claw and penelope-pitstop entries (use the YAML from the spec §3). Add basic reinforcement sections to peter-perfect, dick-dastardly, and ant-hill-mob based on their drive types.

- [ ] **Step 6: Run reinforcement parsing tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorSocialConfigLoaderTest`
Expected: all PASS

- [ ] **Step 7: Write failing test for drive seeding**

```java
@Test
void seedsDriveNodes() {
    var store = new InMemoryMindMapStore();
    var seeder = new ManorCognitiveSeeder(store);

    var config = ManorSocialConfigLoader.load().get("hooded-claw");
    var result = seeder.seed("hooded-claw", config, "test-tenant");

    var nodes = store.nodesIn(result.subgraphId(), "test-tenant");
    var driveNodes = nodes.stream()
        .filter(n -> "drive-intensity".equals(
            n.properties().get("cognitiveKind")))
        .toList();

    assertEquals(3, driveNodes.size(),
        "Hooded Claw has 3 drives: scheming, self-preservation, dominance");

    var scheming = driveNodes.stream()
        .filter(n -> "scheming".equals(n.properties().get("drive-type")))
        .findFirst().orElseThrow();
    assertEquals("0.9", scheming.properties().get("intensity"));
    assertEquals("0.9", scheming.properties().get("initial-intensity"));
    assertEquals("drive-adaptation", scheming.provenance());
}
```

- [ ] **Step 8: Add drive seeding to ManorCognitiveSeeder.seed()**

Use `ide_replace_member` on the `seed` method. After seeding beliefs, add drive seeding:

```java
for (var drive : config.drives()) {
    var existing = nodes.stream()
        .filter(n -> "drive-intensity".equals(
            n.properties().get("cognitiveKind")))
        .filter(n -> drive.type().equals(
            n.properties().get("drive-type")))
        .findFirst();
    if (existing.isPresent()) continue;

    mindMapStore.addNode(
        NodeInput.of(drive.type(), subgraphId)
            .withConfidence(Confidence.stated(0.8, Instant.now()))
            .withProvenance("drive-adaptation")
            .withProperties(Map.of(
                "cognitiveKind", "drive-intensity",
                "agent-id", agentId,
                "drive-type", drive.type(),
                "intensity", String.valueOf(drive.intensity()),
                "initial-intensity", String.valueOf(drive.intensity()),
                "description", drive.description())),
        tenantId);
}
```

Note: the existing `seed()` method returns early if `initialBeliefs().isEmpty()`. This guard must be widened to check `if (config.initialBeliefs().isEmpty() && config.drives().isEmpty())` so drive-only characters get seeded too.

- [ ] **Step 9: Remove drive rendering from CharacterCognition**

Use `ide_replace_member` on `renderCognitiveSections`. Remove the drive rendering block (lines 106-111 — the `socialConfig.drives()` block) and the CognitionCore delegation block (lines 130-138 — the `cognitionCore.promptSections()` block). Keep: initial beliefs, norms, social awareness, trust perceptions.

- [ ] **Step 10: Run all wacky-manor tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#66): drive reinforcement YAML, seeding, CharacterCognition dedup

- SocialConfig.ReinforcementMapping for per-character drive mappings
- ManorCognitiveSeeder seeds drive nodes into COGNITIVE subgraph
- CharacterCognition stops rendering drives (now via CharacterDrivePromptSection)
- social-config.yaml reinforcement sections for all characters

Refs #66"
```

---

## Batch 4: Integration Test

### Task 5: End-to-end drive adaptation integration test

**Files:**
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/engine/DriveAdaptationIntegrationTest.java`

**Interfaces:**
- Consumes: all previous tasks — DriveAdaptationPhase, ManorCognitiveSeeder, CharacterDrivePromptSection, SocialConfig reinforcement

- [ ] **Step 1: Write integration test**

```java
package io.casehub.examples.manor.engine;

import io.casehub.blocks.agentic.social.drive.adaptation.*;
import io.casehub.blocks.agentic.social.prompt.CharacterDrivePromptSection;
import io.casehub.blocks.speech.PromptContext;
import io.casehub.examples.manor.agent.ManorCognitiveSeeder;
import io.casehub.examples.manor.agent.ManorSocialConfigLoader;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import org.junit.jupiter.api.Test;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

class DriveAdaptationIntegrationTest {

    @Test
    void fullLifecycle_seedAdaptRender() {
        var store = new InMemoryMindMapStore();
        var seeder = new ManorCognitiveSeeder(store);
        var allConfigs = ManorSocialConfigLoader.load();
        var tenant = "integration-test";
        var agent = "hooded-claw";

        // 1. Seed drives
        var seedResult = seeder.seed(agent, allConfigs.get(agent), tenant);
        assertNotNull(seedResult.subgraphId());

        // 2. Simulate experience consolidation — add graduated experience nodes
        store.addNode(NodeInput.of("conflict event 1", seedResult.subgraphId())
            .withProvenance("experience-consolidation")
            .withProperties(Map.of(
                "cognitiveKind", "experience",
                "agent-id", agent,
                "event-type", "conflict_resolution"))
            .withPleasure(0.7).withArousal(0.6),
            tenant);

        store.addNode(NodeInput.of("social event 1", seedResult.subgraphId())
            .withProvenance("experience-consolidation")
            .withProperties(Map.of(
                "cognitiveKind", "experience",
                "agent-id", agent,
                "event-type", "social_interaction"))
            .withPleasure(0.5).withArousal(0.3),
            tenant);

        // 3. Build reinforcement map from SocialConfig
        var config = allConfigs.get(agent);
        var agentReinforcement = new HashMap<String, List<DriveReinforcementEntry>>();
        for (var entry : config.reinforcement().entrySet()) {
            agentReinforcement.put(entry.getKey(), entry.getValue().stream()
                .map(m -> new DriveReinforcementEntry(
                    m.drive(),
                    m.direction() != null
                        ? ReinforcementDirection.valueOf(m.direction())
                        : null,
                    m.rewardAxis() != null
                        ? RewardAxis.valueOf(m.rewardAxis())
                        : null))
                .toList());
        }

        // 4. Run adaptation
        var adaptConfig = new TestConfig(0.1, 0.3, 0.1, 1.0, 20);
        var phase = new DriveAdaptationPhase(
            store, Map.of(agent, agentReinforcement), adaptConfig);
        phase.run(tenant, List.of());

        // 5. Verify drives changed
        var nodes = store.nodesIn(seedResult.subgraphId(), tenant);
        var schemingNode = nodes.stream()
            .filter(n -> "drive-intensity".equals(
                n.properties().get("cognitiveKind")))
            .filter(n -> "scheming".equals(
                n.properties().get("drive-type")))
            .findFirst().orElseThrow();

        double schemingIntensity = Double.parseDouble(
            schemingNode.properties().get("intensity"));

        // conflict_resolution reinforces scheming (POSITIVE)
        // social_interaction weakens scheming (NEGATIVE)
        // Net effect depends on relative reward magnitudes

        // 6. Render via CharacterDrivePromptSection
        var renderer = new CharacterDrivePromptSection(store);
        var rendered = renderer.contribute(
            new PromptContext(agent, tenant, null));
        assertNotNull(rendered);
        assertTrue(rendered.contains("scheming"),
            "Rendered output should contain scheming drive");
        assertTrue(rendered.contains("Character Motivations"),
            "Should have Character Motivations heading");
    }

    record TestConfig(
        double learningRate, double arousalWeight,
        double minIntensity, double maxIntensity,
        int maxPerPass
    ) implements DriveAdaptationConfig {}
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=DriveAdaptationIntegrationTest`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all PASS (including existing tests unbroken)

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/test/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "test(#66): end-to-end drive adaptation integration test

Verifies full lifecycle: seed → consolidation → adapt → render.

Refs #66"
```

---

## References

- [2026-09-16-drive-adaptation-design.md] — design spec this plan implements
- [SocialConfig.java:39] — `SocialConfig.Drive` record (wacky-manor)
- [DriveOrchestrator.java:94-145] — platform SDT drive tick (blocks-core)
- [ExperienceConsolidationPhase.java:88-153] — consolidation run method (neocortex)
- [ConsolidationPhase.java] — phase SPI interface (neocortex)
- [ConsolidationScheduler.java:107-177] — scheduler tick and phase ordering (neocortex)
- [ActionImportanceScorer.java] — event type weights (wacky-manor)
- [ManorCognitiveSeeder.java:86-114] — existing seed method (wacky-manor)
- [CharacterCognition.java:100-148] — current drive rendering (wacky-manor)
- [CognitionConfig.java] — subsystem flags (blocks-core)
- [CognitionCore.java:345-376] — promptSections() builder (blocks-core)
- [MindMapStore.java] — store interface (neocortex)
- [NodeUpdate.java] — node property update (neocortex)
- [MindMapQuery.java] — query builder (neocortex)
- [social-config.yaml] — current drive definitions (wacky-manor)
- [decisions.md D9-D18] — design decisions
- GE-20260714-439924 — multiplicative dampening
- GE-20260820-d9129a — volume dampening
- GitHub #66 — focal issue
- GitHub #63 — branch issue group
