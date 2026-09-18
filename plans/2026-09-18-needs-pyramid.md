# Needs Pyramid Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #68 — Needs pyramid — satisfaction tracking with decay across competing need tiers
**Issue group:** #63, #66, #67, #68, #69

**Goal:** Track satisfaction across five competing need tiers, constrain drive adaptation via saturation, and render satisfaction as qualitative prose in the observation pipeline.

**Architecture:** Satisfaction flows through the existing drive reinforcement layer via a global drive→tier mapping (D30). DriveAdaptationPhase is extended (D33) to accumulate tier satisfaction from raw PAD values, apply decay toward configurable resting levels, and constrain its own learning rate by tier saturation. A new NeedsPyramidPromptSection renders reachable tiers as qualitative prose.

**Tech Stack:** Java 26, Quarkus 3.32+, JUnit 5, Smallrye Config, MindMapStore (neocortex)

## Global Constraints

- All new types in `io.casehub.blocks.agentic.social.need` package (blocks-core)
- All tests use `InMemoryMindMapStore` — no Quarkus test profile needed
- Build blocks: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml` from `/Users/mdproctor/claude/casehub/slots/196/blocks`
- Build wacky-manor: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml` from `/Users/mdproctor/claude/casehub/slots/196/examples`
- Commit every task to the appropriate repo (blocks or examples) with `Refs #68`

---

## Batch 1: Foundation — types, config, and CognitionConfig wiring

### Task 1: NeedTier enum, NeedTierMappingProvider SPI, NeedSatisfactionConfig

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/NeedTier.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/NeedTierMappingProvider.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/NeedSatisfactionConfig.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/need/NeedTierTest.java`

**Interfaces:**
- Produces: `NeedTier` enum with 5 values (`SAFETY`, `TASKS`, `SOCIAL`, `SELF_EXPRESSION`, `UNDERSTANDING`)
- Produces: `NeedTierMappingProvider.tierMapping()` → `Map<String, Set<NeedTier>>`
- Produces: `NeedSatisfactionConfig` with `satisfactionIncrement()`, `dissatisfactionIncrement()`, `initialSatisfaction()`, `decay()`, `resting()`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social.need;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class NeedTierTest {

    @Test
    void allFiveTiersExist() {
        assertEquals(5, NeedTier.values().length);
        assertNotNull(NeedTier.SAFETY);
        assertNotNull(NeedTier.TASKS);
        assertNotNull(NeedTier.SOCIAL);
        assertNotNull(NeedTier.SELF_EXPRESSION);
        assertNotNull(NeedTier.UNDERSTANDING);
    }

    @Test
    void valueOfRoundTrips() {
        for (NeedTier tier : NeedTier.values()) {
            assertEquals(tier, NeedTier.valueOf(tier.name()));
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=NeedTierTest
```
Expected: compilation failure — `NeedTier` does not exist.

- [ ] **Step 3: Create NeedTier enum**

Create `blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/NeedTier.java`:

```java
package io.casehub.blocks.agentic.social.need;

public enum NeedTier {
    SAFETY, TASKS, SOCIAL, SELF_EXPRESSION, UNDERSTANDING;
}
```

- [ ] **Step 4: Create NeedTierMappingProvider SPI**

Create `blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/NeedTierMappingProvider.java`:

```java
package io.casehub.blocks.agentic.social.need;

import java.util.Map;
import java.util.Set;

public interface NeedTierMappingProvider {
    Map<String, Set<NeedTier>> tierMapping();
}
```

- [ ] **Step 5: Create NeedSatisfactionConfig**

Create `blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/NeedSatisfactionConfig.java`:

```java
package io.casehub.blocks.agentic.social.need;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.needs-pyramid")
public interface NeedSatisfactionConfig {

    @WithDefault("0.05")
    double satisfactionIncrement();

    @WithDefault("0.05")
    double dissatisfactionIncrement();

    @WithDefault("0.5")
    double initialSatisfaction();

    DecayConfig decay();

    interface DecayConfig {
        @WithDefault("0.15")
        double safety();

        @WithDefault("0.10")
        double tasks();

        @WithDefault("0.08")
        double social();

        @WithDefault("0.05")
        double selfExpression();

        @WithDefault("0.03")
        double understanding();
    }

    RestingConfig resting();

    interface RestingConfig {
        @WithDefault("0.6")
        double safety();

        @WithDefault("0.3")
        double tasks();

        @WithDefault("0.4")
        double social();

        @WithDefault("0.4")
        double selfExpression();

        @WithDefault("0.4")
        double understanding();
    }

    default double decayRate(NeedTier tier) {
        return switch (tier) {
            case SAFETY -> decay().safety();
            case TASKS -> decay().tasks();
            case SOCIAL -> decay().social();
            case SELF_EXPRESSION -> decay().selfExpression();
            case UNDERSTANDING -> decay().understanding();
        };
    }

    default double restingLevel(NeedTier tier) {
        return switch (tier) {
            case SAFETY -> resting().safety();
            case TASKS -> resting().tasks();
            case SOCIAL -> resting().social();
            case SELF_EXPRESSION -> resting().selfExpression();
            case UNDERSTANDING -> resting().understanding();
        };
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=NeedTierTest
```
Expected: PASS (2 tests).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/src/main/java/io/casehub/blocks/agentic/social/need/ blocks/src/test/java/io/casehub/blocks/agentic/social/need/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#68): NeedTier enum, NeedTierMappingProvider SPI, NeedSatisfactionConfig

Refs casehubio/examples#68"
```

### Task 2: CognitionConfig + CognitionCore extension

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionConfig.java`
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitionCore.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/CognitionConfigTest.java` (new)

**Interfaces:**
- Consumes: `NeedTier`, `NeedTierMappingProvider`, `NeedSatisfactionConfig` (from Task 1)
- Produces: `CognitionConfig.needsPyramidEnabled()` boolean
- Produces: `CognitionCore.promptSections()` includes `NeedsPyramidPromptSection` when enabled

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.blocks.agentic.social;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class CognitionConfigTest {

    @Test
    void allIncludesNeedsPyramid() {
        var config = CognitionConfig.all();
        assertTrue(config.needsPyramidEnabled());
    }

    @Test
    void noneExcludesNeedsPyramid() {
        var config = CognitionConfig.none();
        assertFalse(config.needsPyramidEnabled());
    }

    @Test
    void withNeedsPyramidToggles() {
        var config = CognitionConfig.none().with("needsPyramid", true);
        assertTrue(config.needsPyramidEnabled());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitionConfigTest
```
Expected: compilation failure — `needsPyramidEnabled()` does not exist.

- [ ] **Step 3: Modify CognitionConfig**

Use `ide_replace_member` on `CognitionConfig` to add the `needsPyramidEnabled` field. The record gains a 12th boolean parameter. Update `all()`, `none()`, `withDirectives()`, `with()`, and `without()` accordingly.

Updated record:
```java
public record CognitionConfig(
        boolean moodEnabled,
        boolean drivesEnabled,
        boolean mentalModelEnabled,
        boolean userModelEnabled,
        boolean strategyEnabled,
        boolean narrativeEnabled,
        boolean goalsEnabled,
        boolean memoryHygieneEnabled,
        boolean innerLifeEnabled,
        boolean directivePrompts,
        boolean characterDrivesEnabled,
        boolean needsPyramidEnabled
) {
    public static CognitionConfig all() {
        return new CognitionConfig(true, true, true, true, true, true, true, true, true, false, false, true);
    }

    public static CognitionConfig none() {
        return new CognitionConfig(false, false, false, false, false, false, false, false, false, false, false, false);
    }

    public CognitionConfig withDirectives() {
        return new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled,
                userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled,
                memoryHygieneEnabled, innerLifeEnabled, true, characterDrivesEnabled,
                needsPyramidEnabled);
    }

    public CognitionConfig with(String subsystem, boolean enabled) {
        return switch (subsystem) {
            case "mood" -> new CognitionConfig(enabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "drives" -> new CognitionConfig(moodEnabled, enabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "mentalModel" -> new CognitionConfig(moodEnabled, drivesEnabled, enabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "userModel" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, enabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "strategy" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, enabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "narrative" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, enabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "goals" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, enabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "memoryHygiene" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, enabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "innerLife" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, enabled, directivePrompts, characterDrivesEnabled, needsPyramidEnabled);
            case "directivePrompts" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, enabled, characterDrivesEnabled, needsPyramidEnabled);
            case "characterDrives" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, enabled, needsPyramidEnabled);
            case "needsPyramid" -> new CognitionConfig(moodEnabled, drivesEnabled, mentalModelEnabled, userModelEnabled, strategyEnabled, narrativeEnabled, goalsEnabled, memoryHygieneEnabled, innerLifeEnabled, directivePrompts, characterDrivesEnabled, enabled);
            default -> throw new IllegalArgumentException("Unknown subsystem: " + subsystem);
        };
    }

    public CognitionConfig without(String... subsystems) {
        var config = this;
        for (var s : subsystems) {
            config = config.with(s, false);
        }
        return config;
    }
}
```

- [ ] **Step 4: Update CognitionCore.promptSections()**

Add NeedsPyramidPromptSection registration after the `characterDrivesEnabled` block (line ~401 in `promptSections()`). Use `ide_replace_member` on `promptSections`.

Add this block after the `characterDrivesEnabled` check:
```java
if (config.needsPyramidEnabled() && mindMapStore != null) {
    sections.add(new NeedsPyramidPromptSection(mindMapStore, needTierMapping));
}
```

Note: `needTierMapping` is a new field on CognitionCore — a `Map<String, Set<NeedTier>>` injected from `NeedTierMappingProvider`. Add it as a constructor parameter and field. This wiring will compile once NeedsPyramidPromptSection is created in Task 4. For now, the field and import can exist with the class not yet created — the test only verifies CognitionConfig behavior.

- [ ] **Step 5: Fix compilation — update all CognitionConfig call sites**

Search for all callers of `CognitionConfig.all()`, `CognitionConfig.none()`, and direct `new CognitionConfig(...)` constructors. Each needs the 12th boolean added. Use `ide_find_references` on CognitionConfig to locate all call sites.

Key locations to update:
- `SocialAvatarCognition` (creates CognitionConfig via `all()` or `none()`)
- `AgenticYamlProcessor.discoverCognitionConfig()` (build-time discovery)
- Any test files that construct CognitionConfig directly

- [ ] **Step 6: Run test to verify it passes**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=CognitionConfigTest
```
Expected: PASS (3 tests).

- [ ] **Step 7: Run full blocks-core test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml
```
Expected: all existing tests pass (CognitionConfig changes are backward-compatible via `all()`/`none()`).

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/ blocks/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#68): CognitionConfig needsPyramidEnabled + CognitionCore wiring

Refs casehubio/examples#68"
```

---

## Batch 2: Core logic — satisfaction tracking and rendering

### Task 3: DriveAdaptationPhase extension — satisfaction accumulation, decay, and constraint

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhase.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhaseNeedsSatisfactionTest.java` (new)

**Interfaces:**
- Consumes: `NeedTier`, `NeedTierMappingProvider`, `NeedSatisfactionConfig` (from Task 1)
- Consumes: `DriveReinforcementEntry`, `DriveAdaptationConfig` (existing)
- Produces: MindMap nodes with `cognitiveKind: "need-satisfaction"` per tier per agent, updated each consolidation cycle
- Produces: Saturation constraint on drive learning rate via `effectiveLR = learningRate × (1 - avgTierSatisfaction)`

- [ ] **Step 1: Write the failing test — positive event increases tier satisfaction**

Create `blocks/src/test/java/io/casehub/blocks/agentic/social/drive/adaptation/DriveAdaptationPhaseNeedsSatisfactionTest.java`:

```java
package io.casehub.blocks.agentic.social.drive.adaptation;

import io.casehub.blocks.agentic.social.need.NeedSatisfactionConfig;
import io.casehub.blocks.agentic.social.need.NeedTier;
import io.casehub.blocks.agentic.social.need.NeedTierMappingProvider;
import io.casehub.neocortex.cognitive.Confidence;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.SubgraphInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class DriveAdaptationPhaseNeedsSatisfactionTest {

    private InMemoryMindMapStore store;
    private static final String TENANT = "test-tenant";
    private static final String AGENT = "hooded-claw";
    private String subgraphId;

    record TestDriveConfig(
        double learningRate, double arousalWeight,
        double minIntensity, double maxIntensity,
        int maxPerPass
    ) implements DriveAdaptationConfig {}

    record TestDecayConfig(
        double safety, double tasks, double social,
        double selfExpression, double understanding
    ) implements NeedSatisfactionConfig.DecayConfig {}

    record TestRestingConfig(
        double safety, double tasks, double social,
        double selfExpression, double understanding
    ) implements NeedSatisfactionConfig.RestingConfig {}

    record TestNeedConfig(
        double satisfactionIncrement, double dissatisfactionIncrement,
        double initialSatisfaction,
        NeedSatisfactionConfig.DecayConfig decay,
        NeedSatisfactionConfig.RestingConfig resting
    ) implements NeedSatisfactionConfig {
        @Override public double decayRate(NeedTier tier) {
            return switch (tier) {
                case SAFETY -> decay.safety();
                case TASKS -> decay.tasks();
                case SOCIAL -> decay.social();
                case SELF_EXPRESSION -> decay.selfExpression();
                case UNDERSTANDING -> decay.understanding();
            };
        }
        @Override public double restingLevel(NeedTier tier) {
            return switch (tier) {
                case SAFETY -> resting.safety();
                case TASKS -> resting.tasks();
                case SOCIAL -> resting.social();
                case SELF_EXPRESSION -> resting.selfExpression();
                case UNDERSTANDING -> resting.understanding();
            };
        }
    }

    private NeedSatisfactionConfig defaultNeedConfig() {
        return new TestNeedConfig(
            0.1, 0.1, 0.5,
            new TestDecayConfig(0.15, 0.10, 0.08, 0.05, 0.03),
            new TestRestingConfig(0.6, 0.3, 0.4, 0.4, 0.4)
        );
    }

    @BeforeEach
    void setUp() {
        store = new InMemoryMindMapStore();
        subgraphId = store.createSubgraph(
            new SubgraphInput("beliefs-" + AGENT, "cognitive", null), TENANT);

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

        for (NeedTier tier : NeedTier.values()) {
            store.addNode(NodeInput.of("need-" + tier.name().toLowerCase(), subgraphId)
                .withConfidence(Confidence.stated(0.8, Instant.now()))
                .withProvenance("need-satisfaction")
                .withProperties(Map.of(
                    "cognitiveKind", "need-satisfaction",
                    "agent-id", AGENT,
                    "tier", tier.name(),
                    "satisfaction", "0.5",
                    "resting-level", String.valueOf(tier == NeedTier.SAFETY ? 0.6 : 0.4))),
                TENANT);
        }
    }

    private DriveAdaptationPhase createPhase(
            Map<String, Map<String, List<DriveReinforcementEntry>>> reinforcement,
            Map<String, Set<NeedTier>> tierMapping) {
        var driveConfig = new TestDriveConfig(0.1, 0.3, 0.1, 1.0, 20);
        var needConfig = defaultNeedConfig();
        NeedTierMappingProvider mappingProvider = () -> tierMapping;
        return new DriveAdaptationPhase(
            store, reinforcement, driveConfig, mappingProvider, needConfig);
    }

    @Test
    void positiveEventIncreasesTierSatisfaction() {
        var reinforcement = Map.of(
            AGENT, Map.of(
                "conflict_resolution", List.of(
                    new DriveReinforcementEntry("scheming", null, null)
                )
            )
        );
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION));
        var phase = createPhase(reinforcement, tierMapping);

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
        var selfExprNode = nodes.stream()
            .filter(n -> "need-satisfaction".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "SELF_EXPRESSION".equals(n.properties().get("tier")))
            .findFirst().orElseThrow();

        double satisfaction = Double.parseDouble(selfExprNode.properties().get("satisfaction"));
        assertTrue(satisfaction > 0.5,
            "SELF_EXPRESSION satisfaction should increase from 0.5, got " + satisfaction);
    }

    @Test
    void negativeEventDecreasesTierSatisfaction() {
        var reinforcement = Map.of(
            AGENT, Map.of(
                "trust_change", List.of(
                    new DriveReinforcementEntry("scheming", null, null)
                )
            )
        );
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION));
        var phase = createPhase(reinforcement, tierMapping);

        store.addNode(NodeInput.of("trust event", subgraphId)
            .withProvenance("experience-consolidation")
            .withProperties(Map.of(
                "cognitiveKind", "experience",
                "agent-id", AGENT,
                "event-type", "trust_change"))
            .withPleasure(-0.7).withArousal(0.5),
            TENANT);

        phase.run(TENANT, List.of());

        var nodes = store.nodesIn(subgraphId, TENANT);
        var selfExprNode = nodes.stream()
            .filter(n -> "need-satisfaction".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "SELF_EXPRESSION".equals(n.properties().get("tier")))
            .findFirst().orElseThrow();

        double satisfaction = Double.parseDouble(selfExprNode.properties().get("satisfaction"));
        assertTrue(satisfaction < 0.5,
            "SELF_EXPRESSION satisfaction should decrease from 0.5, got " + satisfaction);
    }

    @Test
    void decayTowardRestingLevel() {
        var reinforcement = Map.<String, Map<String, List<DriveReinforcementEntry>>>of(
            AGENT, Map.of());
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION));
        var phase = createPhase(reinforcement, tierMapping);

        phase.run(TENANT, List.of());

        var nodes = store.nodesIn(subgraphId, TENANT);
        var safetyNode = nodes.stream()
            .filter(n -> "need-satisfaction".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "SAFETY".equals(n.properties().get("tier")))
            .findFirst().orElseThrow();

        double satisfaction = Double.parseDouble(safetyNode.properties().get("satisfaction"));
        assertTrue(satisfaction > 0.5,
            "Safety (resting 0.6) should decay UP from 0.5, got " + satisfaction);
        assertTrue(satisfaction < 0.6,
            "Safety should not exceed resting level in one cycle, got " + satisfaction);
    }

    @Test
    void saturationConstraintReducesLearningRate() {
        var reinforcement = Map.of(
            AGENT, Map.of(
                "conflict_resolution", List.of(
                    new DriveReinforcementEntry("scheming", null, null)
                )
            )
        );
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION));

        // Set SELF_EXPRESSION satisfaction very high (0.9)
        var nodes = store.nodesIn(subgraphId, TENANT);
        var selfExprNode = nodes.stream()
            .filter(n -> "need-satisfaction".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "SELF_EXPRESSION".equals(n.properties().get("tier")))
            .findFirst().orElseThrow();
        store.updateNode(selfExprNode.id(),
            io.casehub.neocortex.mindmap.NodeUpdate.empty().withPropertiesToSet(
                Map.of("satisfaction", "0.9")),
            TENANT);

        var phaseHigh = createPhase(reinforcement, tierMapping);

        store.addNode(NodeInput.of("conflict event high", subgraphId)
            .withProvenance("experience-consolidation")
            .withProperties(Map.of(
                "cognitiveKind", "experience",
                "agent-id", AGENT,
                "event-type", "conflict_resolution"))
            .withPleasure(0.6).withArousal(0.5),
            TENANT);

        phaseHigh.run(TENANT, List.of());

        var driveNodeHigh = store.nodesIn(subgraphId, TENANT).stream()
            .filter(n -> "drive-intensity".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "scheming".equals(n.properties().get("drive-type")))
            .findFirst().orElseThrow();
        double highSatIntensity = Double.parseDouble(driveNodeHigh.properties().get("intensity"));

        // The intensity change should be small because tier satisfaction was 0.9
        // effectiveLR = 0.1 * (1 - 0.9) = 0.01
        assertTrue(highSatIntensity < 0.92,
            "With high satisfaction, drive should barely change, got " + highSatIntensity);
        assertTrue(highSatIntensity > 0.9,
            "Drive should still increase slightly, got " + highSatIntensity);
    }

    @Test
    void unmappedDriveSkipsSatisfactionUpdate() {
        var reinforcement = Map.of(
            AGENT, Map.of(
                "conflict_resolution", List.of(
                    new DriveReinforcementEntry("scheming", null, null)
                )
            )
        );
        // Empty tier mapping — scheming is not mapped to any tier
        var tierMapping = Map.<String, Set<NeedTier>>of();
        var phase = createPhase(reinforcement, tierMapping);

        store.addNode(NodeInput.of("conflict event", subgraphId)
            .withProvenance("experience-consolidation")
            .withProperties(Map.of(
                "cognitiveKind", "experience",
                "agent-id", AGENT,
                "event-type", "conflict_resolution"))
            .withPleasure(0.6).withArousal(0.5),
            TENANT);

        phase.run(TENANT, List.of());

        // All satisfaction nodes should be unchanged (only decay applied)
        var selfExprNode = store.nodesIn(subgraphId, TENANT).stream()
            .filter(n -> "need-satisfaction".equals(n.properties().get("cognitiveKind")))
            .filter(n -> "SELF_EXPRESSION".equals(n.properties().get("tier")))
            .findFirst().orElseThrow();

        double satisfaction = Double.parseDouble(selfExprNode.properties().get("satisfaction"));
        // With no events touching SELF_EXPRESSION, it should only experience decay
        // Starting at 0.5, resting at 0.4, decayRate 0.05:
        // 0.4 + (0.5 - 0.4) * (1 - 0.05) = 0.4 + 0.095 = 0.495
        assertTrue(satisfaction < 0.5,
            "With no events, satisfaction should decay, got " + satisfaction);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=DriveAdaptationPhaseNeedsSatisfactionTest
```
Expected: compilation failure — `DriveAdaptationPhase` constructor does not accept `NeedTierMappingProvider` and `NeedSatisfactionConfig`.

- [ ] **Step 3: Extend DriveAdaptationPhase constructor**

Add `NeedTierMappingProvider` and `NeedSatisfactionConfig` as new constructor parameters. Keep the old 3-parameter constructor for backward compatibility with existing tests:

```java
private final Map<String, Set<NeedTier>> tierMapping;
private final NeedSatisfactionConfig needConfig;

public DriveAdaptationPhase(
        MindMapStore mindMapStore,
        Map<String, Map<String, List<DriveReinforcementEntry>>> reinforcementMap,
        DriveAdaptationConfig config) {
    this(mindMapStore, reinforcementMap, config, null, null);
}

public DriveAdaptationPhase(
        MindMapStore mindMapStore,
        Map<String, Map<String, List<DriveReinforcementEntry>>> reinforcementMap,
        DriveAdaptationConfig config,
        NeedTierMappingProvider tierMappingProvider,
        NeedSatisfactionConfig needConfig) {
    this.mindMapStore = mindMapStore;
    this.reinforcementMap = reinforcementMap;
    this.config = config;
    this.tierMapping = tierMappingProvider != null ? tierMappingProvider.tierMapping() : Map.of();
    this.needConfig = needConfig;
}
```

- [ ] **Step 4: Add satisfaction tracking to processSubgraph()**

In `processSubgraph()`, after loading drive and experience nodes, also load need-satisfaction nodes:

```java
var needSatisfactionNodes = new HashMap<NeedTier, MindMapNode>();

for (var node : nodes) {
    var kind = node.properties().get("cognitiveKind");
    if (DRIVE_KIND.equals(kind)) {
        var driveType = node.properties().get("drive-type");
        if (driveType != null) driveNodes.put(driveType, node);
    } else if (EXPERIENCE_PROVENANCE.equals(node.provenance())) {
        experienceNodes.add(node);
    } else if ("need-satisfaction".equals(kind)
               && agentId != null
               && agentId.equals(node.properties().get("agent-id"))) {
        var tierName = node.properties().get("tier");
        if (tierName != null) {
            try {
                needSatisfactionNodes.put(NeedTier.valueOf(tierName), node);
            } catch (IllegalArgumentException ignored) {}
        }
    }
    // cursor handling unchanged
}
```

After `accumulateRewards()` and before `applyUpdates()`, add tier satisfaction accumulation:

```java
var currentSatisfaction = loadCurrentSatisfaction(needSatisfactionNodes);
accumulateTierSatisfaction(currentSatisfaction, newExperiences, agentReinforcement);
applyUpdates(driveNodes, rewardAccumulator, currentSatisfaction, tenantId);
applyDecay(currentSatisfaction);
saveSatisfaction(needSatisfactionNodes, currentSatisfaction, tenantId);
```

- [ ] **Step 5: Implement accumulateTierSatisfaction()**

```java
private void accumulateTierSatisfaction(
        Map<NeedTier, Double> currentSatisfaction,
        List<MindMapNode> experiences,
        Map<String, List<DriveReinforcementEntry>> agentReinforcement) {
    if (needConfig == null || tierMapping.isEmpty()) return;

    var tierRaw = new HashMap<NeedTier, Double>();
    var tierCount = new HashMap<NeedTier, Integer>();

    for (var exp : experiences) {
        var eventType = exp.properties().get("event-type");
        if (eventType == null) continue;
        var entries = agentReinforcement.get(eventType);
        if (entries == null) continue;

        for (var entry : entries) {
            var tiers = tierMapping.get(entry.driveType());
            if (tiers == null) continue;

            double padValue = extractPadValue(exp, entry.rewardAxis());
            if (padValue > 0) {
                for (var tier : tiers) {
                    tierRaw.merge(tier, needConfig.satisfactionIncrement() * padValue, Double::sum);
                    tierCount.merge(tier, 1, Integer::sum);
                }
            } else if (padValue < 0) {
                for (var tier : tiers) {
                    tierRaw.merge(tier, -(needConfig.dissatisfactionIncrement() * Math.abs(padValue)), Double::sum);
                    tierCount.merge(tier, 1, Integer::sum);
                }
            }
        }
    }

    for (var entry : tierRaw.entrySet()) {
        var tier = entry.getKey();
        int count = tierCount.getOrDefault(tier, 1);
        double effectiveDelta = entry.getValue() / (1 + Math.log(count));
        double current = currentSatisfaction.getOrDefault(tier, 0.5);
        currentSatisfaction.put(tier, Math.clamp(current + effectiveDelta, 0.0, 1.0));
    }
}
```

- [ ] **Step 6: Implement applyDecay()**

```java
private void applyDecay(Map<NeedTier, Double> currentSatisfaction) {
    if (needConfig == null) return;
    for (NeedTier tier : NeedTier.values()) {
        double satisfaction = currentSatisfaction.getOrDefault(tier, 0.5);
        double restingLevel = needConfig.restingLevel(tier);
        double decayRate = needConfig.decayRate(tier);
        double decayed = restingLevel + (satisfaction - restingLevel) * (1 - decayRate);
        currentSatisfaction.put(tier, Math.clamp(decayed, 0.0, 1.0));
    }
}
```

- [ ] **Step 7: Modify applyUpdates() for saturation constraint**

Change signature to accept `Map<NeedTier, Double> currentSatisfaction`. Before computing delta, apply the scaling factor:

```java
private void applyUpdates(
        Map<String, MindMapNode> driveNodes,
        Map<String, DriveReward> rewardAccumulator,
        Map<NeedTier, Double> currentSatisfaction,
        String tenantId) {
    for (var entry : rewardAccumulator.entrySet()) {
        var driveNode = driveNodes.get(entry.getKey());
        if (driveNode == null) continue;

        double currentIntensity = Double.parseDouble(
            driveNode.properties().getOrDefault("intensity", "0.5"));

        double effectiveLR = config.learningRate();
        var tiers = tierMapping.get(entry.getKey());
        if (tiers != null && !tiers.isEmpty() && !currentSatisfaction.isEmpty()) {
            double avgSatisfaction = tiers.stream()
                .mapToDouble(t -> currentSatisfaction.getOrDefault(t, 0.5))
                .average().orElse(0.5);
            effectiveLR = config.learningRate() * (1 - avgSatisfaction);
        }

        double delta = entry.getValue().effectiveReward() * effectiveLR;
        double newIntensity = currentIntensity * (1 + delta);
        newIntensity = Math.clamp(newIntensity, config.minIntensity(), config.maxIntensity());

        mindMapStore.updateNode(driveNode.id(),
            NodeUpdate.empty().withPropertiesToSet(
                Map.of("intensity", String.valueOf(newIntensity))),
            tenantId);
    }
}
```

- [ ] **Step 8: Implement helper methods**

```java
private Map<NeedTier, Double> loadCurrentSatisfaction(Map<NeedTier, MindMapNode> needNodes) {
    var result = new HashMap<NeedTier, Double>();
    for (var entry : needNodes.entrySet()) {
        double val = Double.parseDouble(
            entry.getValue().properties().getOrDefault("satisfaction", "0.5"));
        result.put(entry.getKey(), val);
    }
    return result;
}

private void saveSatisfaction(
        Map<NeedTier, MindMapNode> needNodes,
        Map<NeedTier, Double> currentSatisfaction,
        String tenantId) {
    for (var entry : currentSatisfaction.entrySet()) {
        var node = needNodes.get(entry.getKey());
        if (node != null) {
            mindMapStore.updateNode(node.id(),
                NodeUpdate.empty().withPropertiesToSet(
                    Map.of("satisfaction", String.valueOf(entry.getValue()))),
                tenantId);
        }
    }
}
```

- [ ] **Step 9: Run tests to verify they pass**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=DriveAdaptationPhaseNeedsSatisfactionTest
```
Expected: PASS (5 tests).

- [ ] **Step 10: Verify existing DriveAdaptationPhaseTest still passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=DriveAdaptationPhaseTest
```
Expected: PASS (all existing tests — backward-compatible via old 3-param constructor).

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/ blocks/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#68): DriveAdaptationPhase — satisfaction tracking, decay, saturation constraint

Integrated need satisfaction into drive adaptation:
- Accumulates tier satisfaction from raw PAD values with volume dampening
- Bidirectional: positive events increase, negative events decrease
- Decays toward configurable resting levels per tier
- Constrains drive learning rate by tier saturation

Refs casehubio/examples#68"
```

### Task 4: NeedsPyramidPromptSection

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/NeedsPyramidPromptSection.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/NeedsPyramidPromptSectionTest.java`

**Interfaces:**
- Consumes: `MindMapStore` (reads `cognitiveKind: "need-satisfaction"` and `cognitiveKind: "drive-intensity"` nodes)
- Consumes: `Map<String, Set<NeedTier>>` tier mapping (for reachable-tier filtering)
- Produces: `String` observation section under "## Inner Needs" heading, or `null` if no reachable tiers

- [ ] **Step 1: Write the failing test**

Create `blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/NeedsPyramidPromptSectionTest.java`:

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.need.NeedTier;
import io.casehub.blocks.speech.PromptContext;
import io.casehub.neocortex.cognitive.Confidence;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.SubgraphInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.Map;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

class NeedsPyramidPromptSectionTest {

    private InMemoryMindMapStore store;
    private static final String TENANT = "test-tenant";
    private static final String AGENT = "hooded-claw";
    private String subgraphId;

    @BeforeEach
    void setUp() {
        store = new InMemoryMindMapStore();
        subgraphId = store.createSubgraph(
            new SubgraphInput("beliefs-" + AGENT, "cognitive", null), TENANT);

        store.addNode(NodeInput.of("scheming", subgraphId)
            .withConfidence(Confidence.stated(0.8, Instant.now()))
            .withProvenance("drive-adaptation")
            .withProperties(Map.of(
                "cognitiveKind", "drive-intensity",
                "agent-id", AGENT,
                "drive-type", "scheming",
                "intensity", "0.9")),
            TENANT);
    }

    private void seedSatisfaction(NeedTier tier, double value) {
        store.addNode(NodeInput.of("need-" + tier.name().toLowerCase(), subgraphId)
            .withConfidence(Confidence.stated(0.8, Instant.now()))
            .withProvenance("need-satisfaction")
            .withProperties(Map.of(
                "cognitiveKind", "need-satisfaction",
                "agent-id", AGENT,
                "tier", tier.name(),
                "satisfaction", String.valueOf(value))),
            TENANT);
    }

    @Test
    void rendersCorrectBandLabels() {
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION, NeedTier.SAFETY));

        seedSatisfaction(NeedTier.SELF_EXPRESSION, 0.9);
        seedSatisfaction(NeedTier.SAFETY, 0.1);

        var section = new NeedsPyramidPromptSection(store, tierMapping);
        var result = section.contribute(new PromptContext(AGENT, TENANT, null));

        assertNotNull(result);
        assertTrue(result.contains("Inner Needs"), "Should have heading");
        assertTrue(result.contains("self-expression"), "Should render SELF_EXPRESSION");
        assertTrue(result.contains("safety"), "Should render SAFETY");
        assertTrue(result.contains("fulfilled") || result.contains("true to yourself"),
            "0.9 should render as fulfilled");
        assertTrue(result.contains("neglected") || result.contains("uneasy"),
            "0.1 should render as critically neglected");
    }

    @Test
    void filtersUnreachableTiers() {
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION));

        seedSatisfaction(NeedTier.SELF_EXPRESSION, 0.5);
        seedSatisfaction(NeedTier.TASKS, 0.3);

        var section = new NeedsPyramidPromptSection(store, tierMapping);
        var result = section.contribute(new PromptContext(AGENT, TENANT, null));

        assertNotNull(result);
        assertTrue(result.contains("self-expression"), "Reachable tier should appear");
        assertFalse(result.contains("task"), "Unreachable TASKS should be filtered out");
    }

    @Test
    void returnsNullForDrivelessAgent() {
        var tierMapping = Map.of("scheming", Set.of(NeedTier.SELF_EXPRESSION));

        var section = new NeedsPyramidPromptSection(store, tierMapping);
        var result = section.contribute(new PromptContext("unknown-agent", TENANT, null));

        assertNull(result, "Driveless agent should get null (no Inner Needs)");
    }

    @Test
    void rendersAllFiveBands() {
        var tierMapping = Map.of("scheming",
            Set.of(NeedTier.SAFETY, NeedTier.TASKS, NeedTier.SOCIAL,
                   NeedTier.SELF_EXPRESSION, NeedTier.UNDERSTANDING));

        seedSatisfaction(NeedTier.SAFETY, 0.1);       // critically neglected
        seedSatisfaction(NeedTier.TASKS, 0.3);         // neglected
        seedSatisfaction(NeedTier.SOCIAL, 0.5);        // adequate
        seedSatisfaction(NeedTier.UNDERSTANDING, 0.7); // well-met
        seedSatisfaction(NeedTier.SELF_EXPRESSION, 0.9); // fulfilled

        var section = new NeedsPyramidPromptSection(store, tierMapping);
        var result = section.contribute(new PromptContext(AGENT, TENANT, null));

        assertNotNull(result);
        // Each tier should appear
        assertTrue(result.contains("safety"), "SAFETY should appear");
        assertTrue(result.contains("task"), "TASKS should appear");
        assertTrue(result.contains("social"), "SOCIAL should appear");
        assertTrue(result.contains("curiosity"), "UNDERSTANDING should appear");
        assertTrue(result.contains("self-expression"), "SELF_EXPRESSION should appear");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=NeedsPyramidPromptSectionTest
```
Expected: compilation failure — `NeedsPyramidPromptSection` does not exist.

- [ ] **Step 3: Implement NeedsPyramidPromptSection**

Create `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/NeedsPyramidPromptSection.java`:

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.agentic.social.need.NeedTier;
import io.casehub.blocks.speech.PromptContext;
import io.casehub.blocks.speech.PromptSection;
import io.casehub.neocortex.mindmap.MindMapStore;
import org.jspecify.annotations.Nullable;

import java.util.EnumMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

public class NeedsPyramidPromptSection implements PromptSection {

    private final MindMapStore mindMapStore;
    private final Map<String, Set<NeedTier>> tierMapping;

    public NeedsPyramidPromptSection(MindMapStore mindMapStore, Map<String, Set<NeedTier>> tierMapping) {
        this.mindMapStore = mindMapStore;
        this.tierMapping = tierMapping;
    }

    @Override
    public @Nullable String contribute(PromptContext context) {
        var subgraphs = mindMapStore.listSubgraphs(context.tenantId());

        var driveTypes = new HashSet<String>();
        var satisfactionByTier = new EnumMap<NeedTier, Double>(NeedTier.class);

        for (var sg : subgraphs) {
            if (!"cognitive".equals(sg.type())) continue;
            for (var node : mindMapStore.nodesIn(sg.id(), context.tenantId())) {
                if (!context.agentId().equals(node.properties().get("agent-id"))) continue;

                var kind = node.properties().get("cognitiveKind");
                if ("drive-intensity".equals(kind)) {
                    var dt = node.properties().get("drive-type");
                    if (dt != null) driveTypes.add(dt);
                } else if ("need-satisfaction".equals(kind)) {
                    var tierName = node.properties().get("tier");
                    var satStr = node.properties().get("satisfaction");
                    if (tierName != null && satStr != null) {
                        try {
                            satisfactionByTier.put(NeedTier.valueOf(tierName), Double.parseDouble(satStr));
                        } catch (IllegalArgumentException ignored) {}
                    }
                }
            }
        }

        if (driveTypes.isEmpty()) return null;

        var reachableTiers = new HashSet<NeedTier>();
        for (var dt : driveTypes) {
            var tiers = tierMapping.get(dt);
            if (tiers != null) reachableTiers.addAll(tiers);
        }
        if (reachableTiers.isEmpty()) return null;

        var sb = new StringBuilder("## Inner Needs\n\n");
        for (NeedTier tier : NeedTier.values()) {
            if (!reachableTiers.contains(tier)) continue;
            double satisfaction = satisfactionByTier.getOrDefault(tier, 0.5);
            sb.append(renderTier(tier, satisfaction)).append('\n');
        }
        return sb.toString().stripTrailing();
    }

    private String renderTier(NeedTier tier, double satisfaction) {
        String noun = tierNoun(tier);
        String band = bandLabel(satisfaction);
        String note = contextNote(tier, satisfaction);

        return switch (band) {
            case "critically neglected" -> "Your " + noun + " feels critically neglected" +
                (note.isEmpty() ? "." : " — " + note + ".");
            case "neglected" -> "Your " + noun + " feels neglected" +
                (note.isEmpty() ? "." : " — " + note + ".");
            case "adequate" -> "Your " + noun + " feels adequate.";
            case "well-met" -> "Your " + noun + " feels well-met.";
            case "fulfilled" -> "Your " + noun + " feels fulfilled" +
                (note.isEmpty() ? "." : " — " + note + ".");
            default -> "Your " + noun + " feels adequate.";
        };
    }

    private static String tierNoun(NeedTier tier) {
        return switch (tier) {
            case SAFETY -> "sense of safety";
            case TASKS -> "task commitments";
            case SOCIAL -> "social bonds";
            case SELF_EXPRESSION -> "self-expression";
            case UNDERSTANDING -> "curiosity";
        };
    }

    private static String bandLabel(double satisfaction) {
        if (satisfaction < 0.2) return "critically neglected";
        if (satisfaction < 0.4) return "neglected";
        if (satisfaction < 0.6) return "adequate";
        if (satisfaction < 0.8) return "well-met";
        return "fulfilled";
    }

    private static String contextNote(NeedTier tier, double satisfaction) {
        if (satisfaction >= 0.4 && satisfaction < 0.8) return "";
        boolean positive = satisfaction >= 0.8;
        return switch (tier) {
            case SAFETY -> positive ? "you feel secure in your surroundings" : "recent events have left you uneasy";
            case TASKS -> positive ? "you're on top of your responsibilities" : "obligations are piling up";
            case SOCIAL -> positive ? "your relationships feel strong" : "you've been isolated lately";
            case SELF_EXPRESSION -> positive ? "you've been true to yourself lately" : "you haven't been true to yourself";
            case UNDERSTANDING -> positive ? "the world around you makes sense" : "there's much you don't understand yet";
        };
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run from `/Users/mdproctor/claude/casehub/slots/196/blocks`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=NeedsPyramidPromptSectionTest
```
Expected: PASS (4 tests).

- [ ] **Step 5: Run full blocks-core test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml
```
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/ blocks/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#68): NeedsPyramidPromptSection — qualitative prose rendering

Renders satisfaction levels as 'Inner Needs' observation section.
Filters to reachable tiers only, suppresses for driveless characters.
Five bands: critically neglected → fulfilled.

Refs casehubio/examples#68"
```

---

## Batch 3: Wiring — wacky-manor integration

### Task 5: ManorNeedTierMappingProvider + ManorCognitiveSeeder extension

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorNeedTierMappingProvider.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorNeedTierMappingProviderTest.java` (new)
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederTest.java` (modify — add need seeding test)

**Interfaces:**
- Consumes: `NeedTier`, `NeedTierMappingProvider` (from Task 1)
- Produces: CDI `@ApplicationScoped` bean providing the 13-entry drive→tier mapping
- Produces: `ManorCognitiveSeeder.seed()` creates 5 need-satisfaction nodes per agent

- [ ] **Step 1: Write the failing test — mapping provider**

Create `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorNeedTierMappingProviderTest.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.need.NeedTier;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class ManorNeedTierMappingProviderTest {

    @Test
    void maps13DriveTypes() {
        var provider = new ManorNeedTierMappingProvider();
        var mapping = provider.tierMapping();
        assertEquals(13, mapping.size(), "Should map all 13 drive types");
    }

    @Test
    void schemingMapToSelfExpression() {
        var provider = new ManorNeedTierMappingProvider();
        var tiers = provider.tierMapping().get("scheming");
        assertNotNull(tiers);
        assertTrue(tiers.contains(NeedTier.SELF_EXPRESSION));
    }

    @Test
    void adventureMapsToMultipleTiers() {
        var provider = new ManorNeedTierMappingProvider();
        var tiers = provider.tierMapping().get("adventure");
        assertNotNull(tiers);
        assertEquals(2, tiers.size(), "adventure should map to 2 tiers");
        assertTrue(tiers.contains(NeedTier.UNDERSTANDING));
        assertTrue(tiers.contains(NeedTier.SOCIAL));
    }

    @Test
    void allTiersCovered() {
        var provider = new ManorNeedTierMappingProvider();
        var mapping = provider.tierMapping();
        var allTiers = new java.util.HashSet<NeedTier>();
        mapping.values().forEach(allTiers::addAll);
        for (NeedTier tier : NeedTier.values()) {
            assertTrue(allTiers.contains(tier),
                "Tier " + tier + " should be reachable by at least one drive");
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run from `/Users/mdproctor/claude/casehub/slots/196/examples`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorNeedTierMappingProviderTest
```
Expected: compilation failure — `ManorNeedTierMappingProvider` does not exist.

- [ ] **Step 3: Implement ManorNeedTierMappingProvider**

Create `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorNeedTierMappingProvider.java`:

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.agentic.social.need.NeedTier;
import io.casehub.blocks.agentic.social.need.NeedTierMappingProvider;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;
import java.util.Set;

import static io.casehub.blocks.agentic.social.need.NeedTier.*;
import static java.util.Map.entry;

@ApplicationScoped
public class ManorNeedTierMappingProvider implements NeedTierMappingProvider {

    @Override
    public Map<String, Set<NeedTier>> tierMapping() {
        return Map.ofEntries(
            entry("scheming",          Set.of(SELF_EXPRESSION)),
            entry("self-preservation", Set.of(SAFETY)),
            entry("dominance",         Set.of(SELF_EXPRESSION)),
            entry("curiosity",         Set.of(UNDERSTANDING)),
            entry("social-harmony",    Set.of(SOCIAL)),
            entry("adventure",         Set.of(UNDERSTANDING, SOCIAL)),
            entry("gallantry",         Set.of(SOCIAL, TASKS)),
            entry("proving-worth",     Set.of(TASKS, SELF_EXPRESSION)),
            entry("protection",        Set.of(SAFETY, SOCIAL)),
            entry("greed",             Set.of(SELF_EXPRESSION)),
            entry("recognition",       Set.of(SELF_EXPRESSION, SOCIAL)),
            entry("loyalty",           Set.of(SOCIAL, TASKS)),
            entry("suspicion",         Set.of(SAFETY))
        );
    }
}
```

- [ ] **Step 4: Run mapping provider tests**

Run from `/Users/mdproctor/claude/casehub/slots/196/examples`:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorNeedTierMappingProviderTest
```
Expected: PASS (4 tests).

- [ ] **Step 5: Write the failing test — seeder extension**

Add to existing `ManorCognitiveSeederTest.java` (or create if it doesn't exist with appropriate test):

```java
@Test
void seedCreatesNeedSatisfactionNodes() {
    var store = new InMemoryMindMapStore();
    var seeder = new ManorCognitiveSeeder(store);

    var config = new SocialConfig(
        List.of(), 
        List.of(new SocialConfig.Drive("scheming", 0.9, "Test")),
        List.of(), List.of(), List.of(), Map.of(), null);

    var result = seeder.seed("test-agent", config, "tenant");

    var nodes = store.nodesIn(result.subgraphId(), "tenant");
    var needNodes = nodes.stream()
        .filter(n -> "need-satisfaction".equals(n.properties().get("cognitiveKind")))
        .toList();

    assertEquals(5, needNodes.size(), "Should create 5 need satisfaction nodes");

    var safetyNode = needNodes.stream()
        .filter(n -> "SAFETY".equals(n.properties().get("tier")))
        .findFirst().orElseThrow();
    assertEquals("0.5", safetyNode.properties().get("satisfaction"));
    assertEquals("test-agent", safetyNode.properties().get("agent-id"));
}
```

- [ ] **Step 6: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorCognitiveSeederTest#seedCreatesNeedSatisfactionNodes
```
Expected: FAIL — no need nodes created.

- [ ] **Step 7: Extend ManorCognitiveSeeder.seed()**

In the `seed()` method, after the drive seeding loop and before the return statement, add need tier seeding:

```java
for (NeedTier tier : NeedTier.values()) {
    mindMapStore.addNode(
            NodeInput.of("need-" + tier.name().toLowerCase(), subgraphId)
                     .withConfidence(Confidence.stated(0.8, now))
                     .withProvenance("need-satisfaction")
                     .withProperties(Map.of(
                             "cognitiveKind", "need-satisfaction",
                             "agent-id", agentId,
                             "tier", tier.name(),
                             "satisfaction", "0.5",
                             "resting-level", String.valueOf(switch (tier) {
                                 case SAFETY -> 0.6;
                                 case TASKS -> 0.3;
                                 default -> 0.4;
                             }))),
            tenantId);
    timestamps.put("need:" + tier.name(), now);
}
```

Add the import: `import io.casehub.blocks.agentic.social.need.NeedTier;`

Also update the early-return guard: the current guard is `if (config.initialBeliefs().isEmpty() && config.drives().isEmpty())`. This should remain unchanged — if there are no beliefs and no drives, there's no cognitive subgraph to seed need tiers into. Need tiers are only meaningful for characters with drives.

- [ ] **Step 8: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorCognitiveSeederTest
```
Expected: PASS (all tests including new one).

- [ ] **Step 9: Run full wacky-manor test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml
```
Expected: all tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#68): ManorNeedTierMappingProvider + seeder need-tier nodes

13-entry drive→tier mapping for wacky-manor.
ManorCognitiveSeeder seeds 5 need-satisfaction nodes per agent at
bootstrap with initial satisfaction 0.5.

Refs #68"
```

---

## References

- [2026-09-18-needs-pyramid-design.md] — design spec this plan implements
- [DriveAdaptationPhase.java] — blocks-core, extended in Task 3
- [DriveAdaptationConfig.java] — blocks-core, config pattern reference
- [DriveAdaptationPhaseTest.java] — blocks, test pattern reference
- [CharacterDrivePromptSection.java] — blocks-core, rendering pattern reference
- [CharacterDrivePromptSectionTest.java] — blocks, test pattern reference
- [CognitionConfig.java] — blocks-core, modified in Task 2
- [CognitionCore.java:376-405] — blocks-core, promptSections() modified in Task 2
- [ManorCognitiveSeeder.java] — wacky-manor, extended in Task 5
- [SocialConfig.java] — wacky-manor, defines Drive record
- [social-config.yaml] — wacky-manor, 13 drive types for tier mapping
- Decisions D30–D42 — specs/issue-63-fix-cdi-config-beans/decisions.md
- GitHub #68 — Needs pyramid
- GitHub #69 — Goal prioritization (downstream consumer)
- GitHub #66 — Drive adaptation (upstream dependency)
- GE-20260714-439924 — multiplicative dampening
- GE-20260820-d9129a — volume factor
