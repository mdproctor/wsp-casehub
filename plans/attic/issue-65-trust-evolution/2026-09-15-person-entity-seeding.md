# Person-Entity Node Seeding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #61 — Person-entity node seeding — SocialComparison data path
**Issue group:** #61

**Goal:** Seed the mindmap with shared person nodes and perspectival overlays so CognitiveProfile.compare() produces meaningful social comparison output from the start.

**Architecture:** Extend SocialConfig with relationship PAD data, extend ManorSocialConfigLoader to parse it, extend ManorCognitiveSeeder to create shared "people" subgraph with per-agent overlay nodes.

**Tech Stack:** Java 26, Quarkus, Maven multi-module

## Global Constraints

- PAD values use [-1, 1] range (standard PAD model)
- All changes in examples/wacky-manor — no upstream changes
- Overlay nodes use `"overlay"` trait, `OverlayRef.of(sharedNodeId)` ref, `OverlayRef.AGENT_ID` property
- Shared subgraph name: `"people"`
- Idempotent seeding — second call creates no duplicates

---

## Batch 1: SocialConfig + YAML extension

### Task 1: SocialConfig.Relationship + ManorSocialConfigLoader + YAML data

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java`
- Modify: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/SocialConfigTest.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java`

**Interfaces:**
- Consumes: existing SocialConfig record, ManorSocialConfigLoader
- Produces: `SocialConfig.Relationship(String targetAgentId, double pleasure, double arousal, double dominance)`, `SocialConfig.relationships()` returns `List<Relationship>`

- [ ] **Step 1: Write Relationship record tests**

Add to `SocialConfigTest.java`:

```java
@Test
void relationship_validConstruction() {
    var r = new SocialConfig.Relationship("peter-perfect", 0.6, 0.3, 0.5);
    assertThat(r.targetAgentId()).isEqualTo("peter-perfect");
    assertThat(r.pleasure()).isEqualTo(0.6);
    assertThat(r.arousal()).isEqualTo(0.3);
    assertThat(r.dominance()).isEqualTo(0.5);
}

@Test
void relationship_nullTargetThrows() {
    assertThatNullPointerException()
        .isThrownBy(() -> new SocialConfig.Relationship(null, 0.0, 0.0, 0.0));
}

@Test
void relationship_pleasureOutOfRangeThrows() {
    assertThatIllegalArgumentException()
        .isThrownBy(() -> new SocialConfig.Relationship("x", 1.5, 0.0, 0.0));
}

@Test
void relationship_negativePadAllowed() {
    var r = new SocialConfig.Relationship("x", -0.5, -0.3, -0.7);
    assertThat(r.pleasure()).isEqualTo(-0.5);
}

@Test
void emptyConfig_hasEmptyRelationships() {
    assertThat(SocialConfig.empty().relationships()).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=SocialConfigTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: FAIL — Relationship record doesn't exist

- [ ] **Step 3: Add Relationship record to SocialConfig**

Add inside `SocialConfig`:

```java
public record Relationship(String targetAgentId, double pleasure, double arousal, double dominance) {
    public Relationship {
        Objects.requireNonNull(targetAgentId, "targetAgentId required");
        if (pleasure < -1.0 || pleasure > 1.0)
            throw new IllegalArgumentException("pleasure must be in [-1,1], got " + pleasure);
        if (arousal < -1.0 || arousal > 1.0)
            throw new IllegalArgumentException("arousal must be in [-1,1], got " + arousal);
        if (dominance < -1.0 || dominance > 1.0)
            throw new IllegalArgumentException("dominance must be in [-1,1], got " + dominance);
    }
}
```

Add `relationships` field to SocialConfig record:

```java
public record SocialConfig(
        List<Drive> drives,
        List<NormEntry> norms,
        List<InitialBelief> initialBeliefs,
        List<Relationship> relationships
) {
    public SocialConfig {
        drives = drives != null ? List.copyOf(drives) : List.of();
        norms = norms != null ? List.copyOf(norms) : List.of();
        initialBeliefs = initialBeliefs != null ? List.copyOf(initialBeliefs) : List.of();
        relationships = relationships != null ? List.copyOf(relationships) : List.of();
    }

    public static SocialConfig empty() {
        return new SocialConfig(List.of(), List.of(), List.of(), List.of());
    }
    // ... existing inner records unchanged
}
```

- [ ] **Step 4: Fix compilation — update all SocialConfig constructor calls**

The record now has 4 components. Find and fix all existing call sites that construct SocialConfig with 3 args. Use `ide_find_references` to locate them.

- [ ] **Step 5: Run SocialConfigTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=SocialConfigTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: PASS

- [ ] **Step 6: Write ManorSocialConfigLoader relationship parsing test**

Add to `ManorSocialConfigLoaderTest.java`:

```java
@Test
void hoodedClaw_relationshipsLoaded() {
    var hc = ManorSocialConfigLoader.load().get("hooded-claw");
    assertThat(hc.relationships()).isNotEmpty();
    var penRel = hc.relationships().stream()
            .filter(r -> r.targetAgentId().equals("penelope-pitstop")).findFirst().orElseThrow();
    assertThat(penRel.pleasure()).isBetween(-1.0, 1.0);
    assertThat(penRel.arousal()).isBetween(-1.0, 1.0);
    assertThat(penRel.dominance()).isBetween(-1.0, 1.0);
}

@Test
void characterWithNoRelationships_emptyList() {
    var configs = ManorSocialConfigLoader.load();
    for (var config : configs.values()) {
        assertThat(config.relationships()).isNotNull();
    }
}
```

- [ ] **Step 7: Add relationship parsing to ManorSocialConfigLoader**

In `parseCharacterConfig()`, add after the beliefs parsing:

```java
var relationships = raw.containsKey("relationships")
        ? ((List<Map<String, Object>>) raw.get("relationships")).stream()
            .map(m -> new SocialConfig.Relationship(
                    (String) m.get("target"),
                    ((Number) m.get("pleasure")).doubleValue(),
                    ((Number) m.get("arousal")).doubleValue(),
                    ((Number) m.get("dominance")).doubleValue()))
            .toList()
        : List.<SocialConfig.Relationship>of();

return new SocialConfig(drives, norms, beliefs, relationships);
```

- [ ] **Step 8: Add relationships to social-config.yaml**

Add `relationships:` section to each character. Example for hooded-claw:

```yaml
hooded-claw:
  # ... existing drives, norms, initial-beliefs ...
  relationships:
    - target: penelope-pitstop
      pleasure: -0.3
      arousal: 0.8
      dominance: 0.9
    - target: peter-perfect
      pleasure: -0.5
      arousal: 0.6
      dominance: 0.4
    - target: dick-dastardly
      pleasure: 0.3
      arousal: 0.4
      dominance: 0.7
    - target: ant-hill-mob
      pleasure: -0.4
      arousal: 0.5
      dominance: 0.6
```

Add relationships for all 5 characters (each has relationships to the other 4). PAD values should reflect the character dynamics from the show:
- hooded-claw → penelope: negative pleasure (adversary), high arousal (obsessed), high dominance (controlling)
- penelope → hooded-claw (as sneekly): positive pleasure (trusts), low arousal, moderate dominance
- peter → penelope: high pleasure (romantic), moderate arousal, moderate dominance
- etc.

- [ ] **Step 9: Run ManorSocialConfigLoaderTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ManorSocialConfigLoaderTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: PASS — all existing + 2 new tests green

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#61): SocialConfig.Relationship + YAML relationship data

Extends SocialConfig with Relationship(targetAgentId, pleasure, arousal,
dominance) for initial PAD values. All 5 characters have relationships
to each other 4.

Refs #61"
```

---

## Batch 2: ManorCognitiveSeeder — shared people subgraph + overlay nodes

### Task 2: seedPeople() — shared person nodes + perspectival overlays

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorCognitiveSeeder.java`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorCognitiveSeederTest.java`

**Interfaces:**
- Consumes: `SocialConfig.relationships()` (from Task 1), `MindMapStore.addNode(NodeInput, tenantId)`, `MindMapStore.createSubgraph(SubgraphInput, tenantId)`, `MindMapStore.resolveNode(name, subgraphId, tenantId)`, `OverlayRef.of(sharedNodeId)`, `NodeInput.withPad()`, `NodeInput.withTraits()`, `NodeInput.withRefs()`
- Produces: `ManorCognitiveSeeder.seedPeople(Map<String, SocialConfig> allConfigs, String tenantId)` returning `PeopleSeedResult(String subgraphId, Set<String> seededAgentIds, int overlayCount)`

- [ ] **Step 1: Write seedPeople() tests**

Replace `ManorCognitiveSeederTest.java` with comprehensive tests using a mock MindMapStore:

```java
@Test
void seedPeople_createsSharedSubgraph() {
    var store = mock(MindMapStore.class);
    when(store.createSubgraph(any(), eq("t1"))).thenReturn("people-sg");
    when(store.addNode(any(), eq("t1"))).thenReturn("node-1");

    var seeder = new ManorCognitiveSeeder(store);
    var configs = Map.of(
        "agent-a", new SocialConfig(List.of(), List.of(), List.of(),
            List.of(new SocialConfig.Relationship("agent-b", 0.5, 0.3, 0.2))),
        "agent-b", new SocialConfig(List.of(), List.of(), List.of(), List.of())
    );
    var result = seeder.seedPeople(configs, "t1");

    assertThat(result.subgraphId()).isEqualTo("people-sg");
    verify(store).createSubgraph(argThat(s -> s.name().equals("people")), eq("t1"));
}

@Test
void seedPeople_createsSharedPersonNodePerCharacter() {
    var store = mock(MindMapStore.class);
    when(store.createSubgraph(any(), any())).thenReturn("sg-1");
    when(store.addNode(any(), any())).thenReturn("node-id");

    var seeder = new ManorCognitiveSeeder(store);
    var configs = Map.of(
        "agent-a", new SocialConfig(List.of(), List.of(), List.of(),
            List.of(new SocialConfig.Relationship("agent-b", 0.5, 0.3, 0.2))),
        "agent-b", new SocialConfig(List.of(), List.of(), List.of(),
            List.of(new SocialConfig.Relationship("agent-a", -0.1, 0.4, 0.6)))
    );
    seeder.seedPeople(configs, "t1");

    // 2 shared person nodes + 2 overlay nodes = 4 addNode calls
    verify(store, times(4)).addNode(any(), eq("t1"));
}

@Test
void seedPeople_overlayHasCorrectTraitsAndRefs() {
    var store = mock(MindMapStore.class);
    when(store.createSubgraph(any(), any())).thenReturn("sg-1");
    when(store.addNode(any(), any())).thenReturn("shared-node-id");

    var seeder = new ManorCognitiveSeeder(store);
    var configs = Map.of(
        "observer", new SocialConfig(List.of(), List.of(), List.of(),
            List.of(new SocialConfig.Relationship("target", 0.6, 0.3, 0.5)))
    );
    seeder.seedPeople(configs, "t1");

    var captor = ArgumentCaptor.forClass(NodeInput.class);
    verify(store, atLeast(1)).addNode(captor.capture(), eq("t1"));

    var overlayInputs = captor.getAllValues().stream()
        .filter(n -> n.traits().contains("overlay")).toList();
    assertThat(overlayInputs).hasSize(1);

    var overlay = overlayInputs.get(0);
    assertThat(overlay.pleasure()).isEqualTo(0.6);
    assertThat(overlay.arousal()).isEqualTo(0.3);
    assertThat(overlay.dominance()).isEqualTo(0.5);
    assertThat(overlay.properties()).containsEntry("agentId", "observer");
    assertThat(overlay.refs()).anyMatch(r -> "overlay".equals(r.scheme()));
}

@Test
void seedPeople_sharedNodeHasEntitylikeTrait() {
    var store = mock(MindMapStore.class);
    when(store.createSubgraph(any(), any())).thenReturn("sg-1");
    when(store.addNode(any(), any())).thenReturn("node-id");

    var seeder = new ManorCognitiveSeeder(store);
    var configs = Map.of(
        "a", new SocialConfig(List.of(), List.of(), List.of(),
            List.of(new SocialConfig.Relationship("b", 0.0, 0.0, 0.0)))
    );
    seeder.seedPeople(configs, "t1");

    var captor = ArgumentCaptor.forClass(NodeInput.class);
    verify(store, atLeast(1)).addNode(captor.capture(), eq("t1"));

    var entityNodes = captor.getAllValues().stream()
        .filter(n -> n.traits().contains("Entitylike")).toList();
    assertThat(entityNodes).isNotEmpty();
    assertThat(entityNodes.get(0).properties()).containsEntry("agentId", "a");
}

@Test
void seedPeople_emptyConfigs_noNodes() {
    var store = mock(MindMapStore.class);
    var seeder = new ManorCognitiveSeeder(store);
    var result = seeder.seedPeople(Map.of(), "t1");
    assertThat(result.overlayCount()).isZero();
    verify(store, never()).addNode(any(), any());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ManorCognitiveSeederTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: FAIL — seedPeople method doesn't exist

- [ ] **Step 3: Implement seedPeople()**

Add `PeopleSeedResult` record and `seedPeople()` method to ManorCognitiveSeeder:

```java
public record PeopleSeedResult(String subgraphId, Set<String> seededAgentIds, int overlayCount) {}

public static final String PEOPLE_SUBGRAPH = "people";

public PeopleSeedResult seedPeople(Map<String, SocialConfig> allConfigs, String tenantId) {
    Set<String> allAgentIds = allConfigs.keySet();
    if (allAgentIds.isEmpty()) {
        return new PeopleSeedResult(PEOPLE_SUBGRAPH, Set.of(), 0);
    }

    var subgraphId = mindMapStore.createSubgraph(
        new SubgraphInput(PEOPLE_SUBGRAPH, "social", null), tenantId);
    if (subgraphId == null || subgraphId.isBlank()) {
        return new PeopleSeedResult(PEOPLE_SUBGRAPH, Set.of(), 0);
    }

    var now = Instant.now();
    Map<String, String> personNodeIds = new HashMap<>();

    for (String agentId : allAgentIds) {
        String nodeId = mindMapStore.addNode(
            NodeInput.of(agentId, subgraphId)
                .withConfidence(Confidence.stated(0.9, now))
                .withProvenance("manor-seed")
                .withTraits(Set.of("Entitylike"))
                .withProperties(Map.of("agentId", agentId))
                .withPad(0.0, 0.0, 0.0),
            tenantId);
        personNodeIds.put(agentId, nodeId);
    }

    int overlayCount = 0;
    for (var entry : allConfigs.entrySet()) {
        String observerId = entry.getKey();
        for (var rel : entry.getValue().relationships()) {
            String targetNodeId = personNodeIds.get(rel.targetAgentId());
            if (targetNodeId == null) continue;

            mindMapStore.addNode(
                NodeInput.of(observerId + "-sees-" + rel.targetAgentId(), subgraphId)
                    .withConfidence(Confidence.stated(0.7, now))
                    .withProvenance("manor-seed")
                    .withTraits(Set.of("overlay"))
                    .withRefs(Set.of(OverlayRef.of(targetNodeId)))
                    .withProperties(Map.of(OverlayRef.AGENT_ID, observerId))
                    .withPad(rel.pleasure(), rel.arousal(), rel.dominance())
                    .withPrincipalId(PrincipalId.agent(observerId)),
                tenantId);
            overlayCount++;
        }
    }

    return new PeopleSeedResult(subgraphId, Set.copyOf(allAgentIds), overlayCount);
}
```

Add required imports: `OverlayRef`, `PrincipalId`, `NodeRef`.

- [ ] **Step 4: Run ManorCognitiveSeederTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ManorCognitiveSeederTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: PASS — all tests green

- [ ] **Step 5: Run full wacky-manor test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest='!ManorResourceProfileTest' -Dtest=ActionImportanceScorerTest,ManorGraduationScorerTest,ActionResolverTest,WorldStateTest,SceneDirectorTest,TriggerEvaluatorTest,ManorCognitiveSeederTest,ManorSocialConfigLoaderTest,SocialConfigTest,PerceptionTranslatorTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
Expected: PASS — no regressions

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#61): ManorCognitiveSeeder.seedPeople() — shared person nodes + perspectival overlays

Creates shared 'people' subgraph with Entitylike nodes per character and
overlay nodes per observer→target with PAD from SocialConfig.Relationship.
CognitiveProfile.compare() can now resolve and merge perspectival data.

Refs #61"
```

---

## References

- [2026-09-15-person-entity-seeding-design.md] — design spec this plan implements
- `io.casehub.examples.manor.agent.ManorCognitiveSeeder` — extended with seedPeople()
- `io.casehub.examples.manor.agent.SocialConfig` — extended with Relationship record
- `io.casehub.examples.manor.agent.ManorSocialConfigLoader` — extended to parse relationships
- `io.casehub.neocortex.mindmap.OverlayRef` — overlay node linking pattern
- `io.casehub.neocortex.mindmap.NodeInput` — withPad(), withTraits(), withRefs()
- `io.casehub.neocortex.cognitive.index.CognitiveProfile#compare` — downstream consumer
- `io.casehub.examples.manor.agent.PerceptionTranslator` — downstream consumer
- [GitHub #61] — Person-entity node seeding
- [GitHub #53] — Phase B: mindmap-native cognitive integration (parent)
- [GitHub #60] — SocialComparison integration (design review R1-02 identified this gap)
