# Targeted Somatic Drives — Pipeline Re-architecture

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives
**Issue group:** #175

**Goal:** Re-architect the cognitive pipeline so drive descriptions are relationally targeted — generated per-drive-per-role during consolidation, matched to specific characters at scene time, and rendered as `DRIVE (intensity) → Name: [targeted somatic description]`.

**Architecture:** Three-layer change in neocortex: (1) `DriveDescriptionGenerationPhase` reads relational schemas and generates per-drive-per-role Barrett descriptions alongside untargeted ones. (2) `CognitionObservationBuilder` resolves drive→target at scene time using nearby agents' overlay schemas. (3) Rendering composes targeted drives with role-specific descriptions. All cognitive sections preserved — no abilities removed.

**Tech Stack:** Java 26, Quarkus, MindMapStore overlay nodes, LLM consolidation via SessionPipeline/ParallelLlmExecutor, InMemoryMindMapStore for tests

## Global Constraints

- All pipeline changes in neocortex repos (`mindmap-intelligence`, `cognition`). Examples app provides character/world config only — no changes.
- Barrett 4-channel system prompt structure unchanged — somatic register is non-negotiable.
- No cognitive sections removed — relational schemas, attachment, trust, theory of mind sections all stay.
- Consolidation cache must be cleared before eval (schemas change consolidation output).
- TDD: every change starts with a failing test.
- One LLM call per drive (same as current) — schemas are context, not separate calls per role.

---

## Batch 1: Pipeline — Schema-Aware Description Generation

### Task 1: Extend DriveDescriptionGenerationPhase with relational schema context

**Files:**
- Modify: `neocortex/mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhase.java`
- Modify: `neocortex/mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/DriveDescriptionGenerationPhaseTest.java`

**Interfaces:**
- Consumes: `relational-schema` cognitive nodes (`cognitiveKind=relational-schema`, properties: `agent-id`, `target-id`, `rel.role`, `rel.description`) — same data `OtherModelDerivationPhase` reads at line 122-124
- Produces: `DriveDescriptionGenerated` nodes with additional `role` property and compound node name `drive-description-{driveType}-{role}`. Untargeted `drive-description-{driveType}` nodes still produced as before.

- [ ] **Step 1: Write failing test — targeted description generated when schemas present**

Add to `DriveDescriptionGenerationPhaseTest`:

```java
@Test
void generatesTargetedDescriptionWhenRelationalSchemasPresent() {
    seedDriveNode("gallantry", 0.9);
    seedFormationMemory("Your father knelt to help the stranger", "gallantry");
    seedRelationalSchema(AGENT, "penelope-pitstop", "beloved",
            "She means everything — when she smiles, something in you settles");

    phase.run(TENANT, List.of());

    var targetedDescriptions = findDescriptionNodes().stream()
        .filter(n -> "beloved".equals(n.properties().get("role")))
        .toList();
    assertThat(targetedDescriptions).hasSize(1);
    assertThat(targetedDescriptions.getFirst().property("drive-type")).hasValue("gallantry");
    assertThat(targetedDescriptions.getFirst().property("role")).hasValue("beloved");
    assertThat(targetedDescriptions.getFirst().property("generated-description")).isPresent();
}
```

- [ ] **Step 2: Write failing test — untargeted description still generated alongside targeted**

```java
@Test
void untargetedDescriptionStillGeneratedAlongsideTargeted() {
    seedDriveNode("gallantry", 0.9);
    seedFormationMemory("Your father knelt to help the stranger", "gallantry");
    seedRelationalSchema(AGENT, "penelope-pitstop", "beloved",
            "She means everything");

    phase.run(TENANT, List.of());

    var untargeted = findDescriptionNodes().stream()
        .filter(n -> n.properties().get("role") == null)
        .filter(n -> "gallantry".equals(n.properties().get("drive-type")))
        .toList();
    assertThat(untargeted).as("untargeted description must still be generated").hasSize(1);

    var targeted = findDescriptionNodes().stream()
        .filter(n -> "beloved".equals(n.properties().get("role")))
        .toList();
    assertThat(targeted).as("targeted description also generated").hasSize(1);
}
```

- [ ] **Step 3: Write failing test — prompt includes relationship context**

```java
@Test
void promptIncludesRelationalSchemaContext() {
    seedDriveNode("gallantry", 0.9);
    seedFormationMemory("Your father knelt to help the stranger", "gallantry");
    seedRelationalSchema(AGENT, "penelope-pitstop", "beloved",
            "She means everything");

    phase.run(TENANT, List.of());

    var descPrompts = capturedUserPrompts.stream()
        .filter(p -> p.contains("gallantry"))
        .toList();
    assertThat(descPrompts).isNotEmpty();
    assertThat(descPrompts.getFirst()).contains("beloved");
    assertThat(descPrompts.getFirst()).contains("She means everything");
}
```

- [ ] **Step 4: Write failing test — skips targeted descriptions on idempotent rerun**

```java
@Test
void skipsTargetedDescriptionsOnIdempotentRerun() {
    seedDriveNode("gallantry", 0.9);
    seedFormationMemory("Your father knelt to help the stranger", "gallantry");
    seedRelationalSchema(AGENT, "penelope-pitstop", "beloved",
            "She means everything");

    phase.run(TENANT, List.of());
    int firstRunCalls = capturedUserPrompts.size();
    assertThat(findDescriptionNodes().stream()
        .filter(n -> "beloved".equals(n.properties().get("role")))
        .toList()).hasSize(1);

    phase.run(TENANT, List.of());
    assertThat(capturedUserPrompts).as("no new LLM calls on rerun")
        .hasSize(firstRunCalls);
}
```

- [ ] **Step 5: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: FAIL — `seedRelationalSchema` helper doesn't exist, no role-aware generation logic.

- [ ] **Step 6: Add test helper — seedRelationalSchema**

Add to `DriveDescriptionGenerationPhaseTest`:

```java
private void seedRelationalSchema(String agentId, String targetId, String role, String description) {
    store.addNode(
        NodeInput.of("rel-" + agentId + "-" + targetId, cognitiveSubgraphId)
            .withProperties(Map.of(
                "cognitiveKind", "relational-schema",
                "agent-id", agentId,
                "target-id", targetId,
                "rel.role", role,
                "rel.description", description)),
        TENANT);
}
```

- [ ] **Step 7: Update test stub to return targeted format**

Update the `setUp()` stub's response to include targeted descriptions. The existing `parseDescription` will still extract the `description` field, so old tests pass:

```java
// In the stub's query() method, update the non-discrimination branch:
if (isDiscrimination) {
    // ... unchanged
} else {
    String userPrompt2 = userPrompt;
    boolean hasRelationships = userPrompt2.contains("Relationships");
    if (hasRelationships) {
        return Multi.createFrom().items(
            new AgentEvent.TextDelta("{\"description\": \"A warmth rises through your chest, "),
            new AgentEvent.TextDelta("spreading into your shoulders.\", \"targeted\": {\"beloved\": \""),
            new AgentEvent.TextDelta("Something warm and fierce rises when she is near.\"}}"));
    }
    return Multi.createFrom().items(
        new AgentEvent.TextDelta("{\"description\": \"A warmth rises through your chest, "),
        new AgentEvent.TextDelta("spreading into your shoulders.\"}"));
}
```

- [ ] **Step 8: Implement schema-aware description generation**

In `DriveDescriptionGenerationPhase.run()`:

8a. After collecting `formativeNodes` (~line 103), add relational schema collection:
```java
List<MindMapNode> relationalNodes = allCognitive.stream()
    .filter(n -> "relational-schema".equals(n.properties().get("cognitiveKind")))
    .toList();
```

8b. In the per-agent loop (~line 140), collect agent's schemas:
```java
List<MindMapNode> agentSchemas = relationalNodes.stream()
    .filter(n -> agentId.equals(n.properties().get("agent-id")))
    .toList();
```

8c. Update the idempotency check (~line 123) to also track described roles:
```java
Set<String> describedKeys = allCognitive.stream()
    .filter(n -> n.traits().contains(DESCRIPTION_TRAIT))
    .filter(n -> agentId.equals(n.properties().get("agent-id")))
    .map(n -> {
        String dt = n.properties().get("drive-type");
        String role = n.properties().get("role");
        return role != null ? dt + "-" + role : dt;
    })
    .filter(java.util.Objects::nonNull)
    .collect(Collectors.toSet());
```

Check: skip if untargeted + all role combinations already exist.

8d. Pass `agentSchemas` through to `generateAndStore` and `callLlm` (add parameter).

8e. In `callLlm`, after the formation memories section, add schema context:
```java
if (!agentSchemas.isEmpty()) {
    promptBuilder.append("Relationships this character has:\n");
    for (var schema : agentSchemas) {
        String role = schema.properties().get("rel.role");
        String desc = schema.properties().getOrDefault("rel.description", "");
        promptBuilder.append("  - ").append(role);
        if (!desc.isEmpty()) promptBuilder.append(": ").append(desc);
        promptBuilder.append("\n");
    }
    promptBuilder.append("\n");
}
```

8f. Update the response format instruction at the end of the user prompt:
```java
if (!agentSchemas.isEmpty()) {
    promptBuilder.append("Given this character's personality, memories, and relationships, describe how ")
        .append("this pull TYPICALLY manifests in their body — without naming the pull. ")
        .append("Weave all four channels together. Second person, no behavioral instructions.\n\n")
        .append("Also describe how it feels differently when directed at each relationship type ")
        .append("where this pull naturally activates. Not every pull targets every relationship.\n\n")
        .append("Respond with JSON: {\"description\": \"...\", \"targeted\": {\"role\": \"...\"}}");
} else {
    promptBuilder.append("Given this character's personality and formation memories, describe how this pull ")
        .append("TYPICALLY manifests in their body — without naming the pull. ")
        .append("Weave all four channels together. Second person, no behavioral instructions.");
}
```

8g. Update `generateAndStore` to also store targeted descriptions:
```java
// After storing the untargeted description (existing code):
Map<String, String> targeted = parseTargeted(responseText);
for (var entry : targeted.entrySet()) {
    String role = entry.getKey();
    String targetedDesc = entry.getValue();
    if (targetedDesc == null || targetedDesc.isBlank()) continue;
    store.addNode(
        NodeInput.of("drive-description-" + driveType + "-" + role, subgraphId)
            .withTraits(Set.of(DESCRIPTION_TRAIT))
            .withProperties(Map.of(
                "agent-id", agentId,
                "drive-type", driveType,
                "role", role,
                "drive-intensity", intensity,
                "generated-description", targetedDesc)),
        tenantId);
}
```

8h. Add `parseTargeted` method:
```java
private Map<String, String> parseTargeted(String json) {
    int idx = json.indexOf("\"targeted\"");
    if (idx < 0) return Map.of();
    int braceStart = json.indexOf('{', idx);
    if (braceStart < 0) return Map.of();
    // Find matching closing brace
    int depth = 0;
    int braceEnd = -1;
    for (int i = braceStart; i < json.length(); i++) {
        if (json.charAt(i) == '{') depth++;
        else if (json.charAt(i) == '}') { depth--; if (depth == 0) { braceEnd = i; break; } }
    }
    if (braceEnd < 0) return Map.of();
    String inner = json.substring(braceStart + 1, braceEnd).strip();
    var result = new java.util.HashMap<String, String>();
    // Parse key-value pairs: "role": "description"
    java.util.regex.Matcher m = java.util.regex.Pattern
        .compile("\"([^\"]+)\"\\s*:\\s*\"((?:[^\"\\\\]|\\\\.)*)\"")
        .matcher(inner);
    while (m.find()) {
        result.put(m.group(1), m.group(2).replace("\\\"", "\"").replace("\\n", "\n"));
    }
    return result;
}
```

- [ ] **Step 9: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/mindmap-intelligence/pom.xml test -Dtest="DriveDescriptionGenerationPhaseTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: PASS — all existing tests pass, new targeted tests pass.

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add mindmap-intelligence/src/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat: schema-aware drive description generation — per-drive-per-role Barrett descriptions (#175)

DriveDescriptionGenerationPhase now reads relational-schema cognitive nodes
and includes relationship context in the LLM prompt. Generates targeted
descriptions (keyed by drive-type + role) alongside untargeted fallbacks.

Refs casehubio/examples#175"
```

---

## Batch 2: Appraisal + Composition — Targeted Drive Rendering

### Task 2: Add TargetedDrive model and target resolution to CognitionObservationBuilder

**Files:**
- Create: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/TargetedDrive.java`
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilder.java`
- Create: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilderTargetingTest.java`

**Interfaces:**
- Consumes: `DriveSnapshot` (existing), `RelationalSchema` from overlay nodes (existing record in `cognitive-index`), `DriveDescriptionGenerated` nodes with `role` property (from Task 1)
- Produces: `TargetedDrive` records used by `renderInternalState()` for rendering `DRIVE (intensity) → Name: [description]`

- [ ] **Step 1: Write failing test — TargetedDrive record**

Create `CognitionObservationBuilderTargetingTest`:

```java
package io.casehub.neocortex.cognition.prompt.observation;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class TargetedDriveTest {

    @Test
    void targetedDriveCarriesTargetInfo() {
        var drive = new DriveSnapshot("gloating", 0.7, "jaw tightens");
        var targeted = new TargetedDrive(drive, "penelope-pitstop", "Penelope", "prey");

        assertThat(targeted.hasTarget()).isTrue();
        assertThat(targeted.drive().type()).isEqualTo("gloating");
        assertThat(targeted.targetAgentId()).isEqualTo("penelope-pitstop");
        assertThat(targeted.targetDisplayName()).isEqualTo("Penelope");
        assertThat(targeted.role()).isEqualTo("prey");
    }

    @Test
    void untargetedDriveHasNoTarget() {
        var drive = new DriveSnapshot("curiosity", 0.4, "pull behind your eyes");
        var untargeted = TargetedDrive.untargeted(drive);

        assertThat(untargeted.hasTarget()).isFalse();
        assertThat(untargeted.targetAgentId()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognition/pom.xml test -Dtest="TargetedDriveTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: FAIL — `TargetedDrive` class doesn't exist.

- [ ] **Step 3: Implement TargetedDrive record**

Create `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/TargetedDrive.java`:

```java
package io.casehub.neocortex.cognition.prompt.observation;

public record TargetedDrive(
        DriveSnapshot drive,
        String targetAgentId,
        String targetDisplayName,
        String role
) {
    public static TargetedDrive untargeted(DriveSnapshot drive) {
        return new TargetedDrive(drive, null, null, null);
    }

    public boolean hasTarget() {
        return targetAgentId != null;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognition/pom.xml test -Dtest="TargetedDriveTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: PASS.

- [ ] **Step 5: Write failing test — resolveRelationalTargets matches drives to nearby schemas**

Create `CognitionObservationBuilderTargetingTest` (full test class with MindMapStore setup):

```java
package io.casehub.neocortex.cognition.prompt.observation;

import io.casehub.neocortex.cognitive.index.RelationalSchema;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.SubgraphInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import io.casehub.neocortex.mindmap.OverlayRef;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class CognitionObservationBuilderTargetingTest {

    private InMemoryMindMapStore store;
    private static final String TENANT = "t1";
    private static final String OBSERVER = "hooded-claw";
    private static final String TARGET = "penelope-pitstop";

    @BeforeEach
    void setUp() {
        store = new InMemoryMindMapStore();
        seedPersonOverlay(OBSERVER, TARGET, "prey", -0.3, 0.0, 0.8,
            "She inherited everything you built");
    }

    @Test
    void resolvesTargetForDriveWithMatchingSchema() {
        var builder = createBuilder(List.of(
            new DriveSnapshot("gloating", 0.7, "jaw tightens")));

        var targeted = builder.resolveRelationalTargets(
            List.of(new DriveSnapshot("gloating", 0.7, "jaw tightens")),
            Set.of(TARGET),
            Map.of(TARGET, "Penelope"));

        assertThat(targeted).hasSize(1);
        assertThat(targeted.getFirst().hasTarget()).isTrue();
        assertThat(targeted.getFirst().targetAgentId()).isEqualTo(TARGET);
        assertThat(targeted.getFirst().targetDisplayName()).isEqualTo("Penelope");
        assertThat(targeted.getFirst().role()).isEqualTo("prey");
    }

    @Test
    void leavesNonSocialDriveUntargeted() {
        var builder = createBuilder(List.of(
            new DriveSnapshot("curiosity", 0.4, "pull behind your eyes")));

        var targeted = builder.resolveRelationalTargets(
            List.of(new DriveSnapshot("curiosity", 0.4, "pull behind your eyes")),
            Set.of(TARGET),
            Map.of(TARGET, "Penelope"));

        assertThat(targeted).hasSize(1);
        assertThat(targeted.getFirst().hasTarget()).isFalse();
    }

    // --- helpers ---

    private CognitionObservationBuilder createBuilder(List<DriveSnapshot> drives) {
        return new CognitionObservationBuilder(OBSERVER, TENANT, store, null, null, drives);
    }

    private void seedPersonOverlay(String observerId, String targetId,
                                    String role, double trust, double intimacy, double utility,
                                    String description) {
        String peopleSgId = store.createSubgraph(
            new SubgraphInput("people", "people", null), TENANT);
        String targetNodeId = store.addNode(
            NodeInput.of(targetId, peopleSgId)
                .withProperties(Map.of("agentId", targetId)),
            TENANT);
        var schema = new RelationalSchema(targetId, role, trust, intimacy, utility, description);
        var overlayProps = new java.util.HashMap<>(schema.toProperties());
        overlayProps.put(OverlayRef.AGENT_ID, observerId);
        overlayProps.put(OverlayRef.SHARED_NODE_ID, targetNodeId);
        store.addNode(
            NodeInput.of("overlay-" + observerId + "-" + targetId, peopleSgId)
                .withTraits(Set.of("overlay"))
                .withProperties(Map.copyOf(overlayProps)),
            TENANT);
    }
}
```

- [ ] **Step 6: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognition/pom.xml test -Dtest="CognitionObservationBuilderTargetingTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: FAIL — `resolveRelationalTargets` method doesn't exist on `CognitionObservationBuilder`.

- [ ] **Step 7: Implement resolveRelationalTargets**

Add to `CognitionObservationBuilder`:

```java
public List<TargetedDrive> resolveRelationalTargets(
        List<DriveSnapshot> filteredDrives,
        Collection<String> nearbyAgentIds,
        Map<String, String> agentNames) {
    if (mindMapStore == null || tenantId == null || nearbyAgentIds.isEmpty()) {
        return filteredDrives.stream().map(TargetedDrive::untargeted).toList();
    }

    // Resolve schemas for all nearby agents (pattern from AppraisalTickParticipant)
    Map<String, RelationalSchema> nearbySchemas = new java.util.HashMap<>();
    var peopleSg = mindMapStore.listSubgraphs(tenantId).stream()
        .filter(sg -> "people".equals(sg.type()) || "people".equals(sg.name()))
        .findFirst();
    if (peopleSg.isPresent()) {
        var nodes = mindMapStore.nodesIn(peopleSg.get().id(), tenantId);
        Map<String, String> agentToNodeId = new java.util.HashMap<>();
        for (var node : nodes) {
            if (node.traits().contains("overlay")) continue;
            node.property("agentId").ifPresent(aid -> {
                if (nearbyAgentIds.contains(aid)) agentToNodeId.put(aid, node.id());
            });
        }
        for (var overlay : nodes) {
            if (!overlay.traits().contains("overlay")) continue;
            if (!agentId.equals(overlay.property(OverlayRef.AGENT_ID).orElse(null))) continue;
            var sharedId = OverlayRef.sharedNodeId(overlay).orElse(null);
            for (var entry : agentToNodeId.entrySet()) {
                if (entry.getValue().equals(sharedId)) {
                    var schema = RelationalSchema.fromProperties(overlay.properties());
                    if (schema != null) nearbySchemas.put(entry.getKey(), schema);
                }
            }
        }
    }

    if (nearbySchemas.isEmpty()) {
        return filteredDrives.stream().map(TargetedDrive::untargeted).toList();
    }

    // Match each drive to the most salient nearby target
    var result = new java.util.ArrayList<TargetedDrive>();
    for (var drive : filteredDrives) {
        TargetedDrive best = findBestTarget(drive, nearbySchemas, agentNames);
        result.add(best != null ? best : TargetedDrive.untargeted(drive));
    }
    return result;
}

private TargetedDrive findBestTarget(DriveSnapshot drive,
                                      Map<String, RelationalSchema> nearbySchemas,
                                      Map<String, String> agentNames) {
    // Find the nearby agent whose schema role has highest salience for this drive
    String bestAgentId = null;
    RelationalSchema bestSchema = null;
    double bestSalience = 0.0;

    for (var entry : nearbySchemas.entrySet()) {
        var schema = entry.getValue();
        double salience = computeDriveRoleSalience(drive.type(), schema);
        if (salience > bestSalience) {
            bestSalience = salience;
            bestAgentId = entry.getKey();
            bestSchema = schema;
        }
    }
    if (bestSchema == null || bestSalience <= 0.0) return null;

    String displayName = agentNames.getOrDefault(bestAgentId, bestAgentId);
    return new TargetedDrive(drive, bestAgentId, displayName, bestSchema.role());
}

private double computeDriveRoleSalience(String driveType, RelationalSchema schema) {
    // Salience = how naturally this drive activates toward this role
    // Uses the role's absolute trust distance from neutral + utility as signal
    String role = schema.role();
    double salience = Math.abs(schema.trust()) + schema.utility();

    // Boost for roles that are natural targets of specific drive categories
    // This is the semantic bridge between drive types and relationship roles
    return switch (driveType) {
        case "gloating", "dominance" ->
            "prey".equals(role) || "rival".equals(role) ? salience + 1.0 : 0.0;
        case "scheming" ->
            "obstacle".equals(role) || "threat".equals(role) || "rival".equals(role) || "prey".equals(role)
                ? salience + 1.0 : 0.0;
        case "gallantry" ->
            "beloved".equals(role) || "romantic-interest".equals(role) || "ward".equals(role)
                ? salience + 1.0 : 0.0;
        case "protection" ->
            "ward".equals(role) || "beloved".equals(role) || "family".equals(role)
                ? salience + 1.0 : 0.0;
        case "loyalty" ->
            "ally".equals(role) || "family".equals(role) || "guardian".equals(role)
                ? salience + 1.0 : 0.0;
        case "recognition", "proving-worth" ->
            "rival".equals(role) || "obstacle".equals(role) || "suspected-rival".equals(role)
                ? salience + 1.0 : 0.0;
        default -> 0.0;  // Non-social drives — curiosity, adventure, greed, etc.
    };
}
```

Note: `computeDriveRoleSalience` returns 0.0 for drives that have no natural social target, leaving them untargeted. This is the semantic bridge — it lives in neocortex, not in the examples app.

- [ ] **Step 8: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognition/pom.xml test -Dtest="CognitionObservationBuilderTargetingTest,TargetedDriveTest" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat: scene-level drive target resolution via relational schemas (#175)

Add TargetedDrive record and resolveRelationalTargets() to
CognitionObservationBuilder. Matches active drives to nearby agents'
relational schemas using drive-role salience. Non-social drives remain
untargeted.

Refs casehubio/examples#175"
```

### Task 3: Update renderInternalState to render targeted drives

**Files:**
- Modify: `neocortex/cognition/src/main/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilder.java`
- Modify: `neocortex/cognition/src/test/java/io/casehub/neocortex/cognition/prompt/observation/CognitionObservationBuilderTargetingTest.java`

**Interfaces:**
- Consumes: `TargetedDrive` from `resolveRelationalTargets()` (Task 2), role-keyed `DriveDescriptionGenerated` nodes (Task 1)
- Produces: `ObservationSection` with drive lines formatted as `DRIVE (intensity) → DisplayName: [description]`

- [ ] **Step 1: Write failing test — renderInternalState includes → Name for targeted drives**

Add to `CognitionObservationBuilderTargetingTest`:

```java
@Test
void renderInternalStateIncludesTargetAnnotation() {
    // Seed a role-keyed description node
    String cogSgId = store.createSubgraph(
        new SubgraphInput("cognitive-" + OBSERVER, "cognitive", null), TENANT);
    store.addNode(
        NodeInput.of("drive-description-gloating-prey", cogSgId)
            .withTraits(Set.of("DriveDescriptionGenerated"))
            .withProperties(Map.of(
                "agent-id", OBSERVER,
                "drive-type", "gloating",
                "role", "prey",
                "generated-description", "Your jaw loosens watching them fumble")),
        TENANT);
    // Also seed untargeted fallback
    store.addNode(
        NodeInput.of("drive-description-gloating", cogSgId)
            .withTraits(Set.of("DriveDescriptionGenerated"))
            .withProperties(Map.of(
                "agent-id", OBSERVER,
                "drive-type", "gloating",
                "generated-description", "Your jaw loosens, lazy warmth")),
        TENANT);

    var builder = createBuilder(List.of(
        new DriveSnapshot("gloating", 0.7, "gloating desc")));

    var section = builder.renderInternalState(
        List.of(), "", Set.of(TARGET), null);

    assertThat(section).isNotNull();
    String text = section.text();
    assertThat(text).contains("→ Penelope");
    assertThat(text).contains("Your jaw loosens watching them fumble");
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognition/pom.xml test -Dtest="CognitionObservationBuilderTargetingTest#renderInternalStateIncludesTargetAnnotation" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: FAIL — `renderInternalState` doesn't use targeting or role-keyed descriptions.

- [ ] **Step 3: Update readGeneratedDriveDescriptions to return role-keyed descriptions**

Modify `readGeneratedDriveDescriptions()` in `CognitionObservationBuilder` to include role-keyed entries:

```java
private Map<String, String> readGeneratedDriveDescriptions() {
    if (mindMapStore == null || tenantId == null) { return Map.of(); }
    var result = new HashMap<String, String>();
    mindMapStore.listSubgraphs(tenantId).stream()
            .filter(s -> "cognitive".equals(s.type()))
            .flatMap(s -> mindMapStore.nodesIn(s.id(), tenantId).stream())
            .filter(n -> n.traits().contains("DriveDescriptionGenerated"))
            .filter(n -> agentId.equals(n.properties().get("agent-id")))
            .forEach(n -> {
                String driveType = n.properties().get("drive-type");
                String desc = n.properties().get("generated-description");
                String role = n.properties().get("role");
                if (driveType != null && desc != null && !desc.isBlank()) {
                    String key = role != null ? driveType + "-" + role : driveType;
                    result.put(key, desc);
                }
            });
    return result;
}
```

- [ ] **Step 4: Update renderInternalState to use targeting**

In `renderInternalState()`, after `appraisalFilterDrives`, add target resolution and update the description lookup:

```java
// After: var filtered = appraisalFilterDrives(eligible, situationContext, nearbyAgentIds);

// Resolve targets — need agentNames, pass through from caller or build from nearby IDs
var targeted = resolveRelationalTargets(filtered, nearbyAgentIds,
    nearbyAgentIds.stream().collect(Collectors.toMap(id -> id, id -> id)));

Function<TargetedDrive, String> targetedDescriptionFor = td -> {
    var d = td.drive();
    String label = d.type().toUpperCase().replace('-', ' ')
                   + " (" + String.format("%.1f", d.intensity()) + ")";
    if (td.hasTarget()) {
        label += " → " + td.targetDisplayName();
    }
    // Try role-specific description first, fall back to untargeted
    String generated = null;
    if (td.hasTarget() && td.role() != null) {
        generated = generatedDescriptions.get(d.type() + "-" + td.role());
    }
    if (generated == null) {
        generated = generatedDescriptions.get(d.type());
    }
    if (generated != null) {
        return label + ": " + generated;
    }
    return label;
};

var driveDescriptions = targeted.stream()
        .sorted(Comparator.comparingDouble((TargetedDrive td) -> td.drive().intensity()).reversed())
        .map(targetedDescriptionFor)
        .toList();
```

Note: `renderInternalState` currently doesn't receive `agentNames`. The method signature needs updating to accept `Map<String, String> agentNames`. Callers pass the display name map they already have.

- [ ] **Step 5: Update renderInternalState signature**

Add `Map<String, String> agentNames` parameter to `renderInternalState()`:

```java
public ObservationSection renderInternalState(List<String> activatedCategories,
                                               String situationText,
                                               Collection<String> nearbyAgentIds,
                                               SituationContext situationContext,
                                               Map<String, String> agentNames)
```

Find all callers via `ide_find_references` and update them to pass the agent names map. The primary caller is `CharacterCognition.renderCognitiveSections()` which already has `agentNames`.

- [ ] **Step 6: Run all tests to verify nothing broke**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/neocortex/cognition/pom.xml test -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: PASS — all existing + new tests green.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat: render targeted drives with → Name and role-specific descriptions (#175)

renderInternalState now resolves relational targets and renders drives as
DRIVE (intensity) → DisplayName: [targeted somatic description]. Falls back
to untargeted description when no role-specific match exists.

Refs casehubio/examples#175"
```

---

## Batch 3: Evolved Schema Rendering

### Task 4: Update renderRelationalSchemas to use evolved overlay data

**Files:**
- Modify: `examples/wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:369-385`
- Modify: `examples/wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: MindMapStore overlay nodes (people subgraph) with `rel.*` properties from `RelationalSchemaEvolutionPhase`
- Produces: `ObservationSection` with evolved (not static YAML) descriptions for "How You Feel About Those Present"

- [ ] **Step 1: Write failing test — renderRelationalSchemas uses evolved overlay descriptions**

Add to `CharacterCognitionTest`:

```java
@Test
void renderRelationalSchemas_usesEvolvedOverlayDescriptions() {
    // Seed an overlay with evolved description different from YAML
    seedPersonOverlay(agentId, "penelope-pitstop", "prey",
        "She is weaker than you expected — the contempt deepens");

    var sections = characterCognition.renderCognitiveSections(
        Set.of("penelope-pitstop"), Map.of("penelope-pitstop", "Penelope"),
        null, null);

    var relationalSection = sections.stream()
        .filter(s -> "How You Feel About Those Present".equals(s.heading()))
        .findFirst();
    assertThat(relationalSection).isPresent();
    assertThat(relationalSection.get().text())
        .contains("She is weaker than you expected");
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/examples/wacky-manor/pom.xml test -Dtest="CharacterCognitionTest#renderRelationalSchemas_usesEvolvedOverlayDescriptions" -Pfast -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: FAIL — `renderRelationalSchemas` reads static YAML, not overlay data.

- [ ] **Step 3: Update renderRelationalSchemas to read overlay data**

Replace the current implementation (~line 369-385) with overlay-reading version:

```java
private ObservationSection renderRelationalSchemas(Collection<String> nearbyAgentIds, Map<String, String> agentNames) {
    if (mindMapStore == null || tenantId == null) {
        // Fallback to static YAML when no MindMapStore (e.g., unit tests without full setup)
        return renderRelationalSchemasFromConfig(nearbyAgentIds, agentNames);
    }

    var peopleSg = mindMapStore.listSubgraphs(tenantId).stream()
        .filter(sg -> "people".equals(sg.name()))
        .findFirst();
    if (peopleSg.isEmpty()) {
        return renderRelationalSchemasFromConfig(nearbyAgentIds, agentNames);
    }

    var nodes = mindMapStore.nodesIn(peopleSg.get().id(), tenantId);
    Map<String, String> agentToNodeId = new java.util.HashMap<>();
    for (var node : nodes) {
        if (!node.traits().contains("overlay")) {
            node.property("agentId").ifPresent(aid -> {
                if (nearbyAgentIds.contains(aid)) agentToNodeId.put(aid, node.id());
            });
        }
    }

    var sb = new StringBuilder();
    boolean found = false;
    for (var overlay : nodes) {
        if (!overlay.traits().contains("overlay")) continue;
        if (!agentId.equals(overlay.property(io.casehub.neocortex.mindmap.OverlayRef.AGENT_ID).orElse(null))) continue;
        var sharedId = io.casehub.neocortex.mindmap.OverlayRef.sharedNodeId(overlay).orElse(null);
        for (var entry : agentToNodeId.entrySet()) {
            if (entry.getValue().equals(sharedId)) {
                var schema = io.casehub.neocortex.cognitive.index.RelationalSchema.fromProperties(overlay.properties());
                if (schema != null && schema.description() != null && !schema.description().isEmpty()) {
                    var displayName = agentNames.getOrDefault(entry.getKey(), entry.getKey());
                    sb.append("**").append(displayName).append(":** ").append(schema.description()).append("\n");
                    found = true;
                }
            }
        }
    }

    if (!found) return renderRelationalSchemasFromConfig(nearbyAgentIds, agentNames);
    return ObservationSection.text("How You Feel About Those Present", sb.toString().strip());
}

private ObservationSection renderRelationalSchemasFromConfig(Collection<String> nearbyAgentIds, Map<String, String> agentNames) {
    var relevant = socialConfig.relationalSchemas().stream()
        .filter(rs -> nearbyAgentIds.contains(rs.target()))
        .toList();
    if (relevant.isEmpty()) return null;
    var sb = new StringBuilder();
    for (var rs : relevant) {
        var displayName = agentNames.getOrDefault(rs.target(), rs.target());
        sb.append("**").append(displayName).append(":** ");
        if (!rs.description().isEmpty()) sb.append(rs.description());
        sb.append("\n");
    }
    return ObservationSection.text("How You Feel About Those Present", sb.toString().strip());
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/196/examples/wacky-manor/pom.xml test -Dtest="CharacterCognitionTest" -Pfast -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml
```

Expected: PASS — old tests still pass (YAML fallback), new test passes (overlay data).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/src/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "fix: renderRelationalSchemas reads evolved overlay data, YAML as fallback (#175)

RelationalSchemaEvolutionPhase evolves trust/role on overlay nodes, but
renderRelationalSchemas was reading static YAML. Now reads evolved data
from MindMapStore, falling back to YAML when no overlays exist.

Refs #175"
```

---

## References

- `DriveDescriptionGenerationPhase.java` — mindmap-intelligence consolidation, priority 22
- `OtherModelDerivationPhase.java` — mindmap-intelligence consolidation, priority 25 (schema access pattern)
- `CognitionObservationBuilder.java` — cognition prompt observation, renderInternalState + readGeneratedDriveDescriptions
- `AppraisalTickParticipant.java:205-233` — resolveRelationalSchema (overlay lookup pattern)
- `DriveSnapshot.java` — cognition observation model record
- `RelationalSchema.java` — cognitive-index record, fromProperties/toProperties
- `CharacterCognition.java:369-385` — renderRelationalSchemas (static YAML, to be fixed)
- `social-config-generic.yaml` — character/world config with drives, schemas, attachment styles
- `HANDOFF.md` — era 5 eval findings, attention competition analysis
- GitHub #175 — focal issue
- GitHub #156 — related: improve five weak drive expressions
- GitHub #477 — relational schemas (the five layers built in era 5)
