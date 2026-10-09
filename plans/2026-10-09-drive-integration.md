# Drive Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #118 — Wire drive adaptation into CAPS settling and appraisal
**Issue group:** #115, #118, #119

**Goal:** Connect three disconnected cognitive subsystems so that drive adaptation produces visible behavioral change through CAPS attractors, emotional modulation, and memory salience.

**Architecture:** Drives become persistent activations in the CAPS network (Read & Miller 2002), modulate emotional appraisal via SEC relevance checks (Scherer 2001), and gate formation memory salience via PAD proximity (Klinger 1971, Bower 1981). Implementation follows strict phasing: wire computation → verify → then remove prescriptive text.

**Tech Stack:** Java 26, Quarkus, neocortex (cognition, cognition-api, mindmap-intelligence), wacky-manor

## Global Constraints

- Literature-first: every mapping value must be Tier 2+ sourced (D2)
- Runtime configurable: all strategy choices and thresholds exposed as Quarkus config (D4a)
- No raw values exposed to LLM: drive labels, intensity numbers, CAPS internals never in prompt
- Emergence: no prescriptive behavioral text; behavior emerges from computational enrichment
- Cross-repo: neocortex changes require `mvn install` before wacky-manor can consume them
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <module>/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f <module>/pom.xml test -s .mvn/slot-settings.xml`

---

## Batch 1: Drive → CAPS Persistent Activations

Wire drive-intensity nodes into BehavioralSynthesisPhase as persistent activations alongside situation activations. After this batch, CAPS attractors reflect drive state.

### Task 1: Drive-CAPS activation mapping config

**Files:**
- Create: `neocortex/mindmap-intelligence/src/main/resources/META-INF/drive-caps-mapping.yaml`
- Create: `neocortex/mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveCapsMapping.java`
- Test: `neocortex/mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveCapsMappingTest.java`

**Interfaces:**
- Produces: `DriveCapsMapping.activationsFor(String driveType, double intensity, double dampening) → Map<String, Double>`

- [ ] **Step 1: Write the failing test**

```java
@Test
void gallantryDriveProducesExpectedActivations() {
    var mapping = DriveCapsMapping.load();
    var activations = mapping.activationsFor("gallantry", 0.9, 0.3);
    assertFalse(activations.isEmpty());
    assertTrue(activations.containsKey("acceptance"));
    assertTrue(activations.containsKey("other_safety"));
    // activation = mapping_weight * intensity * dampening
    // acceptance: 0.6 * 0.9 * 0.3 = 0.162
    assertEquals(0.162, activations.get("acceptance"), 0.01);
}

@Test
void unknownDriveTypeReturnsEmpty() {
    var mapping = DriveCapsMapping.load();
    var activations = mapping.activationsFor("nonexistent", 0.5, 0.3);
    assertTrue(activations.isEmpty());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest=DriveCapsMappingTest -s .mvn/slot-settings.xml`
Expected: FAIL — class does not exist

- [ ] **Step 3: Create the YAML mapping file**

```yaml
# drive-caps-mapping.yaml
# Universal drive-type → CAPS activation mapping
# Each drive type maps to CAPS input nodes with weights
# Sources: See design spec §3.2 for literature citations per mapping
#
# activation = mapping_weight × drive_intensity × dampening_factor

drive-type-activations:
  gallantry:
    acceptance: 0.6
    other_safety: 0.8
    approach: 0.5
  proving-worth:
    mastery: 0.8
    competence_recognition: 0.6
    self_efficacy: 0.5
  protection:
    other_safety: 0.9
    physical_threat: 0.4
    approach: 0.6
  scheming:
    dominance: 0.7
    social_cognition: 0.5
    distrust: 0.4
  gloating:
    dominance: 0.5
    reward: 0.6
  self-preservation:
    physical_threat: 0.7
    withdraw: 0.4
    self_efficacy: 0.3
  social-harmony:
    acceptance: 0.7
    secure_attachment: 0.5
    approach: 0.4
```

- [ ] **Step 4: Implement DriveCapsMapping**

```java
package io.casehub.neocortex.mindmap.intelligence.consolidation;

import org.yaml.snakeyaml.Yaml;
import java.io.InputStream;
import java.util.*;

public final class DriveCapsMapping {
    private final Map<String, Map<String, Double>> mappings;

    private DriveCapsMapping(Map<String, Map<String, Double>> mappings) {
        this.mappings = mappings;
    }

    public static DriveCapsMapping load() {
        try (InputStream is = DriveCapsMapping.class.getResourceAsStream(
                "/META-INF/drive-caps-mapping.yaml")) {
            if (is == null) return new DriveCapsMapping(Map.of());
            var yaml = new Yaml();
            Map<String, Object> root = yaml.load(is);
            @SuppressWarnings("unchecked")
            var raw = (Map<String, Map<String, Double>>) root.get("drive-type-activations");
            return new DriveCapsMapping(raw != null ? raw : Map.of());
        } catch (Exception e) {
            return new DriveCapsMapping(Map.of());
        }
    }

    public Map<String, Double> activationsFor(String driveType, double intensity, double dampening) {
        var weights = mappings.get(driveType);
        if (weights == null) return Map.of();
        var result = new HashMap<String, Double>();
        for (var entry : weights.entrySet()) {
            result.put(entry.getKey(), entry.getValue() * intensity * dampening);
        }
        return result;
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest=DriveCapsMappingTest -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C neocortex add mindmap-intelligence/src/
git -C neocortex commit -m "feat(#118): drive-CAPS activation mapping — literature-derived universal mappings"
```

### Task 2: Inject drive activations into BehavioralSynthesisPhase

**Files:**
- Modify: `neocortex/mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/BehavioralSynthesisPhase.java:72-115`
- Test: `neocortex/mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/BehavioralSynthesisPhaseTest.java`

**Interfaces:**
- Consumes: `DriveCapsMapping.activationsFor(String, double, double) → Map<String, Double>` (Task 1)
- Produces: drive-enriched `combinedActivations` fed to `capsEngine.settle()`

- [ ] **Step 1: Write the failing test**

```java
@Test
void settlingIncludesDriveActivationsWhenPresent() {
    // Set up MindMapStore with a drive-intensity node for the agent
    var driveNode = createNode("drive-proving-worth", Map.of(
        "cognitiveKind", "drive-intensity",
        "drive-type", "proving-worth",
        "intensity", "0.8",
        "agent-id", AGENT_ID));
    store.addNode(COGNITIVE_SG, driveNode);

    // Set up a graduated cognitive node (triggers situation activation)
    var graduatedNode = createGraduatedNode("experience-1", AGENT_ID);
    store.addNode(COGNITIVE_SG, graduatedNode);

    phase.run(TENANT_ID, List.of());

    // Verify capsEngine.settle was called with combined activations
    // that include BOTH situation activations AND drive activations
    var capturedActivations = capsEngine.lastSettleActivations();
    assertNotNull(capturedActivations);
    // Drive activations: proving-worth maps to mastery (0.8 * 0.8 * 0.3 = 0.192)
    assertTrue(capturedActivations.containsKey("mastery"),
        "Drive activation 'mastery' should be present from proving-worth drive");
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — BehavioralSynthesisPhase doesn't read drive nodes yet

- [ ] **Step 3: Modify BehavioralSynthesisPhase.run()**

In `run()` method, after building `combinedActivations` from SituationClassifier (line 110) and before the `if (combinedActivations.isEmpty())` check (line 112), add drive activation reading:

```java
// After line 110 (end of situation activation loop)
// Inject persistent drive activations (Read & Miller 2002)
var driveActivations = readDriveActivations(tenantId, agentId);
for (var da : driveActivations.entrySet()) {
    combinedActivations.merge(da.getKey(), da.getValue(), Double::sum);
}
```

Add the helper method:

```java
private static final DriveCapsMapping DRIVE_MAPPING = DriveCapsMapping.load();
private static final double DEFAULT_DAMPENING = 0.3;

private Map<String, Double> readDriveActivations(String tenantId, String agentId) {
    var subgraphs = store.listSubgraphs(tenantId);
    var driveNodes = subgraphs.stream()
        .filter(s -> "cognitive".equals(s.type()))
        .flatMap(s -> store.nodesIn(s.id(), tenantId).stream())
        .filter(n -> "drive-intensity".equals(n.properties().get("cognitiveKind")))
        .filter(n -> agentId.equals(n.properties().get("agent-id")))
        .toList();

    var combined = new HashMap<String, Double>();
    for (var node : driveNodes) {
        var driveType = node.properties().get("drive-type");
        var intensityStr = node.properties().get("intensity");
        if (driveType == null || intensityStr == null) continue;
        double intensity = Double.parseDouble(intensityStr);
        var activations = DRIVE_MAPPING.activationsFor(driveType, intensity, DEFAULT_DAMPENING);
        for (var entry : activations.entrySet()) {
            combined.merge(entry.getKey(), entry.getValue(), Double::sum);
        }
    }
    return combined;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -Dtest=BehavioralSynthesisPhaseTest -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Run full module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml test -s .mvn/slot-settings.xml`
Expected: All tests pass

- [ ] **Step 6: Commit**

```bash
git -C neocortex add mindmap-intelligence/src/
git -C neocortex commit -m "feat(#118): inject drive activations into CAPS settling — persistent dispositional activations per Read & Miller 2002"
```

- [ ] **Step 7: Install neocortex for downstream consumption**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml
```

---

## Batch 2: Drive → Appraisal Wiring

Wire character drives into the SEC appraisal checks via call-site merge in AppraisalTickParticipant. After this batch, character drives modulate emotional intensity.

### Task 3: Extend appraisal Drive record for keyword triggers

**Files:**
- Modify: `neocortex/cognition-api/src/main/java/io/casehub/neocortex/cognition/appraisal/Drive.java:5-18`
- Test: `neocortex/cognition-api/src/test/java/io/casehub/neocortex/cognition/appraisal/DriveTest.java`

**Interfaces:**
- Produces: `Drive(String name, DriveCategory category, double intensity, String trigger, List<String> triggers)` — `triggers` is the keyword list; `trigger` kept for backward compat

- [ ] **Step 1: Write the failing test**

```java
@Test
void driveWithTriggersKeywordList() {
    var drive = new Drive("gallantry", DriveCategory.CHARACTER, 0.9, "",
        List.of("help", "courtesy", "rescue", "protect"));
    assertEquals(4, drive.triggers().size());
    assertTrue(drive.triggers().contains("protect"));
}

@Test
void legacyDriveConstructorStillWorks() {
    var drive = new Drive("curiosity", DriveCategory.BASELINE, 0.5, "curious");
    assertNotNull(drive.triggers());
    assertTrue(drive.triggers().isEmpty());
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — Drive record has no `triggers` field

- [ ] **Step 3: Extend Drive record**

```java
public record Drive(
        String name,
        DriveCategory category,
        double intensity,
        String trigger,
        List<String> triggers) {
    public Drive {
        Objects.requireNonNull(name, "name");
        Objects.requireNonNull(category, "category");
        if (intensity < 0.0 || intensity > 1.0) {
            throw new IllegalArgumentException("intensity must be in [0, 1]: " + intensity);
        }
        if (trigger == null) trigger = "";
        if (triggers == null) triggers = List.of();
    }

    public Drive(String name, DriveCategory category, double intensity, String trigger) {
        this(name, category, intensity, trigger, List.of());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Install cognition-api**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition-api/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml
```

- [ ] **Step 6: Commit**

```bash
git -C neocortex add cognition-api/src/
git -C neocortex commit -m "feat(#118): extend Drive record with triggers keyword list for character drive relevance detection"
```

### Task 4: Wire character drives into AppraisalTickParticipant

**Files:**
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/appraisal/AppraisalTickParticipant.java:18-84`
- Test: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/appraisal/AppraisalTickParticipantTest.java`

**Interfaces:**
- Consumes: `Drive(name, DriveCategory.CHARACTER, intensity, "", triggers)` (Task 3)
- Produces: merged drive list (SDT + CHARACTER) flowing through SEC checks

- [ ] **Step 1: Write the failing test**

```java
@Test
void tickMergesCharacterDrivesWithSdtDrives() {
    // Set up MindMapStore with a character drive-intensity node
    store.addDriveNode("gallantry", 0.9, AGENT_ID);

    participant.tick(createContext(AGENT_ID, TENANT_ID, "Someone needs help"));

    var result = participant.currentResult(AGENT_ID, TENANT_ID);
    assertTrue(result.isPresent());
    // Character drive should amplify relevance for "help" keyword
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — AppraisalTickParticipant doesn't read MindMap drives

- [ ] **Step 3: Add MindMapStore to AppraisalTickParticipant**

Add `MindMapStore` field and constructor parameter. In `tick()`, after line 54 (`var drives = profileOpt.get().allDrives()`), read character drives and merge:

```java
// After line 54
var characterDrives = readCharacterDrives(context.agentId(), context.tenantId());
if (!characterDrives.isEmpty()) {
    var merged = new java.util.ArrayList<>(drives);
    merged.addAll(characterDrives);
    drives = List.copyOf(merged);
}
```

Add helper:

```java
private final MindMapStore mindMapStore; // add to constructor

private List<Drive> readCharacterDrives(String agentId, String tenantId) {
    if (mindMapStore == null) return List.of();
    var subgraphs = mindMapStore.listSubgraphs(tenantId);
    return subgraphs.stream()
        .filter(s -> "cognitive".equals(s.type()))
        .flatMap(s -> mindMapStore.nodesIn(s.id(), tenantId).stream())
        .filter(n -> "drive-intensity".equals(n.properties().get("cognitiveKind")))
        .filter(n -> agentId.equals(n.properties().get("agent-id")))
        .map(n -> new Drive(
            n.properties().getOrDefault("drive-type", "unknown"),
            DriveCategory.CHARACTER,
            Double.parseDouble(n.properties().getOrDefault("intensity", "0")),
            "",
            parseTriggers(n.properties().getOrDefault("drive-type", ""))))
        .toList();
}
```

Note: `parseTriggers` will need a lookup from drive type to keyword list. This could come from a config or from social-config.yaml. For the neocortex layer, the simplest approach is to pass triggers as a node property or use the drive type name as the trigger (the wacky-manor layer will enrich later).

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Run full cognition module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s .mvn/slot-settings.xml
```

- [ ] **Step 6: Install cognition**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml
```

- [ ] **Step 7: Commit**

```bash
git -C neocortex add cognition/src/
git -C neocortex commit -m "feat(#118): wire character drives into appraisal — call-site merge in AppraisalTickParticipant per Scherer 2001"
```

### Task 5: Add triggers to social-config.yaml drives

**Files:**
- Modify: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java:65-73`
- Modify: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java`
- Modify: `examples/wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- Test: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java`

**Interfaces:**
- Produces: `SocialConfig.Drive(type, intensity, description, triggers, pad)` with keyword lists and PAD profiles

- [ ] **Step 1: Write the failing test**

```java
@Test
void drivesParsedWithTriggersAndPad() {
    var config = loader.loadForAgent("peter-perfect");
    var gallantry = config.drives().stream()
        .filter(d -> "gallantry".equals(d.type())).findFirst().orElseThrow();
    assertFalse(gallantry.triggers().isEmpty());
    assertTrue(gallantry.triggers().contains("help"));
    assertNotEquals(0.0, gallantry.pleasure());
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Extend SocialConfig.Drive record**

```java
public record Drive(String type, double intensity, String description,
                    List<String> triggers, double pleasure, double arousal, double dominance) {
    public Drive {
        Objects.requireNonNull(type);
        if (description == null) description = "";
        if (triggers == null) triggers = List.of();
        if (intensity < 0.0 || intensity > 1.0) {
            throw new IllegalArgumentException("intensity must be in [0,1], got " + intensity);
        }
    }

    public Drive(String type, double intensity, String description) {
        this(type, intensity, description, List.of(), 0.0, 0.0, 0.0);
    }
}
```

- [ ] **Step 4: Update social-config.yaml with triggers and PAD for PP drives**

```yaml
drives:
  - type: gallantry
    intensity: 0.9
    triggers: ["help", "courtesy", "rescue", "protect", "defend", "kindness", "manners"]
    pad: { pleasure: 0.7, arousal: 0.4, dominance: 0.5 }
    description: >-
      A warmth moves through you when courtesy matters...
  - type: proving-worth
    intensity: 0.7
    triggers: ["capable", "competent", "skill", "prove", "accomplish", "succeed", "achieve"]
    pad: { pleasure: 0.5, arousal: 0.6, dominance: 0.7 }
    description: >-
      There's a tension in your hands...
  - type: protection
    intensity: 0.8
    triggers: ["threat", "danger", "protect", "guard", "safety", "defend", "vulnerable"]
    pad: { pleasure: 0.6, arousal: 0.5, dominance: 0.6 }
    description: >-
      You notice broken things...
```

- [ ] **Step 5: Update ManorSocialConfigLoader to parse triggers and PAD**

- [ ] **Step 6: Update all callers of Drive constructor** (use ide_find_references on old constructor)

- [ ] **Step 7: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -s .mvn/slot-settings.xml
```

- [ ] **Step 8: Commit**

```bash
git add wacky-manor/src/
git commit -m "feat(#118): add triggers and PAD profiles to SocialConfig.Drive — keyword lists for appraisal, PAD for memory salience"
```

---

## Batch 3: Drive → Memory Salience

Implement Klinger's current concerns theory — active drives modulate formation memory salience via PAD proximity. After this batch, memory rendering responds to drive state.

### Task 6: Salience computation in renderFormationMemories

**Files:**
- Modify: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:219-226`
- Create: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/MemorySalience.java`
- Test: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/agent/MemorySalienceTest.java`

**Interfaces:**
- Consumes: `SocialConfig.Drive(type, intensity, ..., pleasure, arousal, dominance)` (Task 5)
- Consumes: `SocialConfig.FormationMemory(age, episode, pleasure, arousal, dominance)` (existing)
- Produces: `MemorySalience.score(Drive, FormationMemory) → double`
- Produces: `MemorySalience.rank(List<Drive>, List<FormationMemory>) → List<FormationMemory>` (sorted by salience)

- [ ] **Step 1: Write the failing test**

```java
@Test
void memoryWithSimilarPadToStrongDriveScoresHigher() {
    var drive = new SocialConfig.Drive("protection", 0.8, "", List.of(),
        0.6, 0.5, 0.6);  // PAD centroid
    var protectionMemory = new SocialConfig.FormationMemory(
        16, "You intervened when she was bullied", 0.9, 0.8, 0.7);
    var neutralMemory = new SocialConfig.FormationMemory(
        10, "You watched a film with your mother", -0.3, 0.4, -0.5);

    double protScore = MemorySalience.score(drive, protectionMemory);
    double neutralScore = MemorySalience.score(drive, neutralMemory);

    assertTrue(protScore > neutralScore,
        "Protection-congruent memory should score higher than neutral");
}

@Test
void rankOrdersMemoriesBySalience() {
    var drives = List.of(
        new SocialConfig.Drive("protection", 0.8, "", List.of(), 0.6, 0.5, 0.6));
    var memories = List.of(
        new SocialConfig.FormationMemory(10, "Film", -0.3, 0.4, -0.5),
        new SocialConfig.FormationMemory(16, "Intervened", 0.9, 0.8, 0.7),
        new SocialConfig.FormationMemory(5, "Father's lesson", 0.7, 0.4, 0.3));

    var ranked = MemorySalience.rank(drives, memories, 0.5, 0.25, 0.25);
    assertEquals("Intervened", ranked.get(0).episode().substring(0, 10));
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement MemorySalience**

```java
package io.casehub.examples.manor.agent;

import java.util.*;

public final class MemorySalience {

    public static double score(SocialConfig.Drive drive, SocialConfig.FormationMemory memory) {
        return score(drive, memory, 0.5, 0.25, 0.25);
    }

    public static double score(SocialConfig.Drive drive, SocialConfig.FormationMemory memory,
                               double wP, double wA, double wD) {
        double dot = wP * drive.pleasure() * memory.pleasure()
                   + wA * drive.arousal() * memory.arousal()
                   + wD * drive.dominance() * memory.dominance();
        return dot * drive.intensity();
    }

    public static List<SocialConfig.FormationMemory> rank(
            List<SocialConfig.Drive> drives,
            List<SocialConfig.FormationMemory> memories,
            double wP, double wA, double wD) {
        return memories.stream()
            .sorted(Comparator.comparingDouble(
                (SocialConfig.FormationMemory m) -> drives.stream()
                    .mapToDouble(d -> score(d, m, wP, wA, wD))
                    .max().orElse(0.0))
                .reversed())
            .toList();
    }

    private MemorySalience() {}
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Wire into renderFormationMemories**

Replace the unconditional rendering in `CharacterCognition.renderFormationMemories()`:

```java
private ObservationSection renderFormationMemories() {
    var memories = socialConfig.formationMemories();
    if (memories.isEmpty()) return null;

    var drives = resolveAdaptedDrives();
    if (!drives.isEmpty() && drives.stream().anyMatch(d -> d.pleasure() != 0.0 || d.arousal() != 0.0)) {
        memories = MemorySalience.rank(drives, memories, 0.5, 0.25, 0.25);
    }

    var sb = new StringBuilder();
    sb.append("These are the experiences that made you who you are.\n\n");
    for (var mem : memories) {
        sb.append("**Age ").append(mem.age()).append(":** ").append(mem.episode()).append("\n\n");
    }
    return ObservationSection.text("Where You Come From", sb.toString().strip());
}
```

- [ ] **Step 6: Run full wacky-manor tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -s .mvn/slot-settings.xml
```

- [ ] **Step 7: Commit**

```bash
git add wacky-manor/src/
git commit -m "feat(#119): memory salience via PAD proximity — Klinger current concerns + Bower mood-congruent memory"
```

---

## Batch 4: Content Enrichment + Rendering Cleanup

Enrich formation memories to the proven 5/5 format. Remove somatic drive text from renderInternalState. This batch MUST NOT start until Batches 1-3 are verified via micro experiments.

### Task 7: Enrich PP formation memories

**Files:**
- Modify: `examples/wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml:412-457`
- Test: manual eval — `RelationalModelEvalTest`

**Interfaces:**
- Produces: enriched formation memories with behavioral response + emotional payoff

- [ ] **Step 1: Enrich each PP memory**

Add behavioral response and emotional payoff to each memory WITHOUT naming drives. Follow the proven format from eval tests. Example:

Age 5 (before): "Your father took you to watch a race..."
Age 5 (after): "...You spent the rest of the afternoon pretending to fix things around the house, showing him each one: 'Look, Dad, I fixed it.'"

Apply to all 9 PP memories (ages 5, 7, 8, 10, 12, 13, 14, 16, 17).

- [ ] **Step 2: Run eval test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest=RelationalModelEvalTest -s .mvn/slot-settings.xml
```

Expected: PP ambiguous scenarios score ≥4/5

- [ ] **Step 3: Commit**

```bash
git add wacky-manor/src/main/resources/
git commit -m "feat(#118): enrich PP formation memories — behavioral response + emotional payoff without naming drives"
```

### Task 8: Remove somatic drive text from renderInternalState

**GATE: Batches 1-3 must be verified via micro experiments before starting this task.**

**Files:**
- Modify: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:238-274`
- Test: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: CAPS attractor nodes from behavioral subgraph (existing)
- Produces: "What Pulls at You" section with CAPS attractors only (no somatic drive text)

- [ ] **Step 1: Write the test**

```java
@Test
void renderInternalStateContainsOnlyAttractorsNotSomaticDrives() {
    var section = cognition.renderInternalState();
    if (section != null) {
        String content = section.content();
        // Should NOT contain somatic drive descriptions
        assertFalse(content.contains("warmth moves through you"),
            "Somatic drive text should be removed");
        assertFalse(content.contains("tension in your hands"),
            "Somatic drive text should be removed");
    }
}
```

- [ ] **Step 2: Modify renderInternalState**

Remove the somatic drive rendering (lines 239-244). Keep only the CAPS attractor rendering (lines 246-268):

```java
private ObservationSection renderInternalState() {
    if (mindMapStore == null || tenantId == null) return null;

    var subgraphs = mindMapStore.listSubgraphs(tenantId);
    var attractorLines = subgraphs.stream()
        .filter(s -> "behavioral".equals(s.type()))
        .flatMap(s -> mindMapStore.nodesIn(s.id(), tenantId).stream())
        .filter(n -> n.traits().contains("CapsGenerated"))
        .filter(n -> agentId.equals(n.properties().get("agent-id")))
        .filter(n -> {
            var str = n.properties().get("strength");
            return str != null && Double.parseDouble(str) > 0.1;
        })
        .sorted(java.util.Comparator.comparingDouble(
            (io.casehub.neocortex.mindmap.MindMapNode n) -> {
                var str = n.properties().get("strength");
                return str != null ? Double.parseDouble(str) : 0.0;
            }).reversed())
        .limit(5)
        .map(this::renderAttractorAsFelt)
        .toList();

    if (attractorLines.isEmpty()) return null;
    return ObservationSection.text("What Pulls at You", String.join("\n\n", attractorLines));
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -s .mvn/slot-settings.xml
```

- [ ] **Step 4: Run drive expression eval** (verify no regression)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -s .mvn/slot-settings.xml
```

Expected: drive expression scores maintained or improved vs pre-removal baseline

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/
git commit -m "feat(#118): remove somatic drive text from renderInternalState — CAPS attractors only, verified via micro experiments"
```

---

## References

- [2026-10-09-drive-integration-architecture-design.md] — design spec this plan implements
- [decisions.md] — D1-D8 design decisions with literature citations
- [BehavioralSynthesisPhase.java:72-186] — CAPS settling, drive activation injection point
- [AppraisalTickParticipant.java:46-84] — appraisal tick, character drive merge point
- [CharacterCognition.java:219-274] — formation memory and internal state rendering
- [SocialConfig.java:65-73, 150-158] — Drive and FormationMemory records
- [Drive.java:5-18] — appraisal Drive record (cognition-api)
- [DriveCategory.java:3-6] — BASELINE/CHARACTER enum
- [caps-topology.yaml] — CAPS network topology, input node IDs
- [GitHub #118] — Wire drive adaptation into CAPS settling and appraisal
- [GitHub #119] — Klinger's current concerns theory
- [GitHub #115] — Close the drive expression gap
