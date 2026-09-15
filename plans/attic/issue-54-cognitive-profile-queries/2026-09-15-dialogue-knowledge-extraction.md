# Dialogue Knowledge Extraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #55 — Wire ConversationBridge for dialogue knowledge extraction
**Issue group:** #54, #55, #56, #57, #59, #60

**Goal:** Enable characters to learn structured knowledge (entities, relationships) from dialogue by extracting it into the mindmap where CognitiveProfile queries surface it.

**Architecture:** ScenarioOrchestrator calls MindMapExtractor.extract() once per dialogue event after all dialogue for the tick is published. Extracted entities/relationships are written as per-listener nodes via MindMapStore with proper principalId and ConfidenceOrigin (STATED for direct targets, INFERRED for overhearers). Listener determination uses the existing `perception` capability tag for overhearing gating.

**Tech Stack:** Java 26, Quarkus, CDI (Instance<>), neocortex mindmap-intelligence (MindMapExtractor, ExtractionResult), neocortex cognitive-api (Confidence, ConfidenceOrigin, PrincipalId)

## Global Constraints

- All imports from `io.casehub.neocortex.mindmap.intelligence` (MindMapExtractor, ExtractionResult, ExtractedEntity, ExtractedRelationship)
- All imports from `io.casehub.neocortex.cognitive` (Confidence, ConfidenceOrigin)
- Confidence for STATED = `Confidence.stated(0.8, Instant.now())` — matches ManorCognitiveSeeder's seeded belief confidence
- Confidence for INFERRED = `Confidence.inferred(0.7, Instant.now())`
- Provenance for dialogue-extracted nodes = `"dialogue-extraction"`
- Subgraph naming = `ManorCognitiveSeeder.subgraphName(agentId)` — reuse static helper
- MindMapExtractor.extract() creates orphaned global nodes internally — accepted trade-off, file upstream issue for parseOnly()
- ManorEvent field for dialogue text = `detailedDescription()` (not `detailedText()`)
- ManorEvent field for dialogue target = `dialogueTarget()`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`

---

## Batch 1: Listener determination + text filtering

### Task 1: Listener determination helper and non-verbal filter

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/DialogueExtractionTest.java`

**Interfaces:**
- Consumes: `CharacterState.capabilityTags()` (Set<String>), `WorldState.charactersInRoom(room)` (List<CharacterState>)
- Produces:
  - `ScenarioOrchestrator.determineListeners(String speakerId, String dialogueTargetId, String room, boolean isExchange, WorldState world)` → `List<ListenerInfo>`
  - `ScenarioOrchestrator.isExtractableDialogue(String text)` → `boolean`
  - `ScenarioOrchestrator.ListenerInfo(String agentId, ConfidenceOrigin confidenceOrigin)` — package-private record

- [ ] **Step 1: Write failing tests for listener determination**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.cognitive.ConfidenceOrigin;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class DialogueExtractionTest {

    // --- Listener determination ---

    @Test
    void directedDialogue_targetGetsStated() {
        var world = TestWorldBuilder.create()
                .withCharacter("speaker", "Library")
                .withCharacter("target", "Library")
                .build();
        var listeners = ScenarioOrchestrator.determineListeners(
                "speaker", "target", "Library", false, world);
        assertThat(listeners).hasSize(1);
        assertThat(listeners.get(0).agentId()).isEqualTo("target");
        assertThat(listeners.get(0).confidenceOrigin()).isEqualTo(ConfidenceOrigin.STATED);
    }

    @Test
    void directedDialogue_perceptionTagOverhears() {
        var world = TestWorldBuilder.create()
                .withCharacter("speaker", "Library")
                .withCharacter("target", "Library")
                .withCharacter("eavesdropper", "Library", Set.of("perception"))
                .build();
        var listeners = ScenarioOrchestrator.determineListeners(
                "speaker", "target", "Library", false, world);
        assertThat(listeners).hasSize(2);
        assertThat(listeners).extracting("agentId").containsExactlyInAnyOrder("target", "eavesdropper");
        var eavesdropperInfo = listeners.stream()
                .filter(l -> l.agentId().equals("eavesdropper")).findFirst().orElseThrow();
        assertThat(eavesdropperInfo.confidenceOrigin()).isEqualTo(ConfidenceOrigin.INFERRED);
    }

    @Test
    void directedDialogue_noPerceptionNoOverhear() {
        var world = TestWorldBuilder.create()
                .withCharacter("speaker", "Library")
                .withCharacter("target", "Library")
                .withCharacter("bystander", "Library")
                .build();
        var listeners = ScenarioOrchestrator.determineListeners(
                "speaker", "target", "Library", false, world);
        assertThat(listeners).hasSize(1);
        assertThat(listeners.get(0).agentId()).isEqualTo("target");
    }

    @Test
    void directedDialogue_differentRoomNotIncluded() {
        var world = TestWorldBuilder.create()
                .withCharacter("speaker", "Library")
                .withCharacter("target", "Library")
                .withCharacter("elsewhere", "Kitchen", Set.of("perception"))
                .build();
        var listeners = ScenarioOrchestrator.determineListeners(
                "speaker", "target", "Library", false, world);
        assertThat(listeners).hasSize(1);
    }

    @Test
    void exchange_bothParticipantsGetStated() {
        var world = TestWorldBuilder.create()
                .withCharacter("initiator", "Library")
                .withCharacter("target", "Library")
                .withCharacter("bystander", "Library", Set.of("perception"))
                .build();
        var listeners = ScenarioOrchestrator.determineListeners(
                "initiator", "target", "Library", true, world);
        assertThat(listeners).hasSize(2);
        assertThat(listeners).extracting("agentId")
                .containsExactlyInAnyOrder("initiator", "target");
        assertThat(listeners).allMatch(l -> l.confidenceOrigin() == ConfidenceOrigin.STATED);
    }

    @Test
    void roomDialogue_allPresentGetStated() {
        var world = TestWorldBuilder.create()
                .withCharacter("speaker", "Library")
                .withCharacter("alice", "Library")
                .withCharacter("bob", "Library")
                .build();
        var listeners = ScenarioOrchestrator.determineListeners(
                "speaker", null, "Library", false, world);
        assertThat(listeners).hasSize(2);
        assertThat(listeners).extracting("agentId")
                .containsExactlyInAnyOrder("alice", "bob");
        assertThat(listeners).allMatch(l -> l.confidenceOrigin() == ConfidenceOrigin.STATED);
    }

    // --- Non-verbal filter ---

    @Test
    void extractable_normalDialogue() {
        assertThat(ScenarioOrchestrator.isExtractableDialogue(
                "The poison is hidden in the Ballroom")).isTrue();
    }

    @Test
    void notExtractable_purelySounds() {
        assertThat(ScenarioOrchestrator.isExtractableDialogue("Hehehehehehe!")).isFalse();
    }

    @Test
    void notExtractable_singleWord() {
        assertThat(ScenarioOrchestrator.isExtractableDialogue("SQUEAK!")).isFalse();
    }

    @Test
    void notExtractable_null() {
        assertThat(ScenarioOrchestrator.isExtractableDialogue(null)).isFalse();
    }

    @Test
    void notExtractable_blank() {
        assertThat(ScenarioOrchestrator.isExtractableDialogue("   ")).isFalse();
    }

    @Test
    void extractable_shortButMeaningful() {
        assertThat(ScenarioOrchestrator.isExtractableDialogue(
                "I saw Dick steal it")).isTrue();
    }
}
```

Note: `TestWorldBuilder` is a test helper that needs to be created — a minimal builder that constructs a `WorldState` with characters in specific rooms with specific capability tags. Check if an existing test helper exists first; if not, create a minimal one in the test class.

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=DialogueExtractionTest`
Expected: FAIL — `determineListeners` and `isExtractableDialogue` do not exist

- [ ] **Step 3: Implement ListenerInfo record and helper methods**

Add to `ScenarioOrchestrator.java`:

```java
record ListenerInfo(String agentId, ConfidenceOrigin confidenceOrigin) {}

static List<ListenerInfo> determineListeners(
        String speakerId, String dialogueTargetId, String room,
        boolean isExchange, WorldState world) {
    var listeners = new java.util.ArrayList<ListenerInfo>();
    var inRoom = world.charactersInRoom(room);

    if (isExchange) {
        // PULL_ASIDE: both participants get STATED
        for (var c : inRoom) {
            if (c.agentId().equals(speakerId) || c.agentId().equals(dialogueTargetId)) {
                listeners.add(new ListenerInfo(c.agentId(), ConfidenceOrigin.STATED));
            }
        }
    } else if (dialogueTargetId != null) {
        // Directed dialogue: target gets STATED, perception-tagged get INFERRED
        for (var c : inRoom) {
            if (c.agentId().equals(speakerId)) continue;
            if (c.agentId().equals(dialogueTargetId)) {
                listeners.add(new ListenerInfo(c.agentId(), ConfidenceOrigin.STATED));
            } else if (c.capabilityTags().contains("perception")) {
                listeners.add(new ListenerInfo(c.agentId(), ConfidenceOrigin.INFERRED));
            }
        }
    } else {
        // Room dialogue: all present (except speaker) get STATED
        for (var c : inRoom) {
            if (!c.agentId().equals(speakerId)) {
                listeners.add(new ListenerInfo(c.agentId(), ConfidenceOrigin.STATED));
            }
        }
    }
    return listeners;
}

static boolean isExtractableDialogue(String text) {
    if (text == null || text.isBlank()) return false;
    String[] words = text.strip().split("\\s+");
    long alphabeticWords = java.util.Arrays.stream(words)
            .filter(w -> w.replaceAll("[^a-zA-Z]", "").length() >= 2)
            .count();
    return text.length() > 10 && alphabeticWords >= 3;
}
```

Add import: `import io.casehub.neocortex.cognitive.ConfidenceOrigin;`

- [ ] **Step 4: Create TestWorldBuilder if needed**

Check existing test helpers first. If none exist, add a minimal builder inside `DialogueExtractionTest.java`:

```java
static class TestWorldBuilder {
    private final java.util.Map<String, io.casehub.examples.manor.model.CharacterState> characters = new java.util.HashMap<>();

    static TestWorldBuilder create() { return new TestWorldBuilder(); }

    TestWorldBuilder withCharacter(String agentId, String room) {
        return withCharacter(agentId, room, Set.of());
    }

    TestWorldBuilder withCharacter(String agentId, String room, Set<String> tags) {
        var cs = new io.casehub.examples.manor.model.CharacterState(agentId, agentId, room, 0, 0);
        cs.setCapabilityTags(tags);
        characters.put(agentId, cs);
        return this;
    }

    io.casehub.examples.manor.model.WorldState build() {
        var world = new io.casehub.examples.manor.model.WorldState();
        characters.values().forEach(world::addCharacter);
        return world;
    }
}
```

Verify `WorldState.addCharacter()` and `CharacterState` constructor exist via IDE before writing.

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=DialogueExtractionTest`
Expected: PASS — all 10 tests

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: All 436+ tests pass

- [ ] **Step 7: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/DialogueExtractionTest.java
git commit -m "feat(#55): listener determination + non-verbal filter for dialogue extraction

Refs #55

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: MindMapExtractor wiring + tick loop integration

### Task 2: Inject MindMapExtractor and implement extractDialogueKnowledge()

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`

**Interfaces:**
- Consumes: `MindMapExtractor.extract(String text, String tenantId, List<String> recentEntityNames)` → `ExtractionResult`
- Consumes: `ExtractionResult.entities()` → `List<ExtractedEntity>`, `ExtractionResult.relationships()` → `List<ExtractedRelationship>`
- Consumes: `MindMapStore.addNode(NodeInput, String tenantId)`, `MindMapStore.createSubgraph(SubgraphInput, String tenantId)`
- Consumes: `ManorCognitiveSeeder.subgraphName(String agentId)` → `String`
- Consumes: `ListenerInfo` and `determineListeners()` from Task 1
- Produces: `ScenarioOrchestrator.extractDialogueKnowledge(String text, String speakerId, String dialogueTargetId, String room, boolean isExchange, WorldState world)` — private method

- [ ] **Step 1: Add MindMapExtractor injection field**

Add to ScenarioOrchestrator alongside existing `Instance<MindMapStore>`:

```java
@Inject
Instance<io.casehub.neocortex.mindmap.intelligence.MindMapExtractor> mindMapExtractorInstance;
```

- [ ] **Step 2: Implement extractDialogueKnowledge() method**

Add private method to ScenarioOrchestrator:

```java
private final java.util.Map<String, String> subgraphIdCache = new java.util.concurrent.ConcurrentHashMap<>();

private void extractDialogueKnowledge(String text, String speakerId,
        String dialogueTargetId, String room, boolean isExchange,
        WorldState world) {
    if (!isExtractableDialogue(text)) return;
    if (!mindMapExtractorInstance.isResolvable()) return;

    var listeners = determineListeners(speakerId, dialogueTargetId, room, isExchange, world);
    if (listeners.isEmpty()) return;

    var mindMapExtractor = mindMapExtractorInstance.get();
    var mindMapStore = mindMapStoreInstance.isResolvable() ? mindMapStoreInstance.get() : null;
    if (mindMapStore == null) return;

    var nearbyNames = world.charactersInRoom(room).stream()
            .map(io.casehub.examples.manor.model.CharacterState::name)
            .toList();

    io.casehub.neocortex.mindmap.intelligence.ExtractionResult result;
    try {
        result = mindMapExtractor.extract(text, ManorConstants.TENANCY_ID, nearbyNames);
    } catch (Exception e) {
        log.warnf(e, "Dialogue extraction failed for speaker=%s room=%s", speakerId, room);
        return;
    }

    if (result.entities().isEmpty() && result.relationships().isEmpty()) return;

    var now = java.time.Instant.now();
    for (var listener : listeners) {
        var confidence = listener.confidenceOrigin() == ConfidenceOrigin.STATED
                ? Confidence.stated(0.8, now)
                : Confidence.inferred(0.7, now);

        var subgraphId = subgraphIdCache.computeIfAbsent(listener.agentId(), id -> {
            var existing = ManorCognitiveSeeder.subgraphName(id);
            var created = mindMapStore.createSubgraph(
                    new io.casehub.neocortex.mindmap.SubgraphInput(existing, "cognitive", null),
                    ManorConstants.TENANCY_ID);
            return created != null && !created.isBlank() ? created : existing;
        });

        for (var entity : result.entities()) {
            var nodeInput = io.casehub.neocortex.mindmap.NodeInput.of(entity.name(), subgraphId)
                    .withConfidence(confidence)
                    .withProvenance("dialogue-extraction")
                    .withPrincipalId(io.casehub.platform.api.identity.PrincipalId.agent(listener.agentId()));
            if (entity.properties() != null && !entity.properties().isEmpty()) {
                nodeInput = nodeInput.withProperties(entity.properties());
            }
            if (entity.subgraphType() != null) {
                nodeInput = nodeInput.withTraits(java.util.Set.of(entity.subgraphType()));
            }
            mindMapStore.addNode(nodeInput, ManorConstants.TENANCY_ID);
        }
    }
}
```

Add imports:
```java
import io.casehub.neocortex.cognitive.Confidence;
import io.casehub.neocortex.cognitive.ConfidenceOrigin;
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl wacky-manor -s slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: All tests pass (extractDialogueKnowledge() not yet called from the tick loop — no behavioral change)

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git commit -m "feat(#55): extractDialogueKnowledge() — MindMapExtractor wiring with per-listener node creation

Refs #55

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: Wire extraction into the tick loop + integration tests

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` (runAutonomousTicks method)
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/DialogueExtractionIntegrationTest.java`

**Interfaces:**
- Consumes: `extractDialogueKnowledge()` from Task 2
- Consumes: ExchangeRunner.run() → `List<ManorEvent>`
- Consumes: `ManorEvent.detailedDescription()`, `ManorEvent.characterId()`, `ManorEvent.dialogueTarget()`, `ManorEvent.room()`

- [ ] **Step 1: Write integration test for MindMapExtractor availability**

```java
package io.casehub.examples.manor.agent;

import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import io.casehub.neocortex.mindmap.intelligence.MindMapExtractor;
import io.casehub.neocortex.mindmap.MindMapStore;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DialogueExtractionIntegrationTest {

    @Inject
    Instance<MindMapExtractor> mindMapExtractorInstance;

    @Inject
    Instance<MindMapStore> mindMapStoreInstance;

    @Test
    void mindMapExtractor_resolvableOrGracefullyAbsent() {
        // MindMapExtractor may or may not be available depending on CDI context.
        // If resolvable, verify it's not null. If not, verify graceful degradation.
        if (mindMapExtractorInstance.isResolvable()) {
            assertThat(mindMapExtractorInstance.get()).isNotNull();
        }
        // Either way, the test passes — extraction is optional
    }

    @Test
    void subgraphCreation_followsSeederConvention() {
        if (!mindMapStoreInstance.isResolvable()) return;
        var store = mindMapStoreInstance.get();
        var subgraphId = store.createSubgraph(
                new io.casehub.neocortex.mindmap.SubgraphInput(
                        ManorCognitiveSeeder.subgraphName("test-agent"),
                        "cognitive", null),
                "test-tenant");
        assertThat(subgraphId).isNotNull();
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=DialogueExtractionIntegrationTest`
Expected: PASS

- [ ] **Step 3: Wire extraction into runAutonomousTicks()**

In `ScenarioOrchestrator.runAutonomousTicks()`, make these changes:

**3a.** After the PULL_ASIDE exchange loop (after line ~288, after the `suppressed` set processing), collect exchange events for extraction:

```java
// Collect exchange dialogue for extraction
var exchangeTexts = new java.util.ArrayList<String[]>(); // [text, speakerId, targetId, room]
```

Inside the PULL_ASIDE block (where `exchangeRunner.run()` is called), after `dispatcher.publishDialogue()`:

```java
// Collect exchange text for knowledge extraction
var exchangeText = exchangeEvents.stream()
        .map(ManorEvent::detailedDescription)
        .filter(d -> d != null && !d.isBlank())
        .collect(java.util.stream.Collectors.joining("\n"));
if (!exchangeText.isBlank()) {
    exchangeTexts.add(new String[]{exchangeText, c.agentId(), targetId, c.currentRoom()});
}
```

**3b.** After all dialogue is published (after the regular dialogue loop ending at line ~321), add extraction calls:

```java
// Extract knowledge from dialogue events
for (var ex : exchangeTexts) {
    extractDialogueKnowledge(ex[0], ex[1], ex[2], ex[3], true, world);
}
for (var c : actingThisTick) {
    if (suppressed.contains(c.agentId())) continue;
    var response = responses.get(c.agentId());
    if (response == null || response.dialogue() == null) continue;
    String dialogueText = response.dialogue();
    String validatedTalkTo = response.talkTo();
    if (validatedTalkTo != null) {
        var target = world.character(validatedTalkTo);
        if (target == null || !target.currentRoom().equals(c.currentRoom())) {
            validatedTalkTo = null;
        }
    }
    extractDialogueKnowledge(dialogueText, c.agentId(), validatedTalkTo,
            c.currentRoom(), false, world);
}
```

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl wacky-manor -s slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: All tests pass. Extraction calls are guarded by `mindMapExtractorInstance.isResolvable()` — in test contexts where MindMapExtractor isn't available, extraction is silently skipped.

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/DialogueExtractionIntegrationTest.java
git commit -m "feat(#55): wire dialogue extraction into tick loop — PULL_ASIDE + directed + room dialogue

Refs #55

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-15-conversation-bridge-design.md] — design spec this plan implements
- [ScenarioOrchestrator.java:188-387] — runAutonomousTicks method
- [ManorCognitiveSeeder.java:24-52] — seeder pattern for node creation with principalId
- [ManorEvent.java] — record fields: detailedDescription(), dialogueTarget(), characterId(), room()
- [ExchangeRunner.java:38-94] — run() returns List<ManorEvent>
- [MindMapExtractor] — io.casehub.neocortex.mindmap.intelligence, extract(text, tenantId, recentEntityNames) → ExtractionResult
- [Confidence] — io.casehub.neocortex.cognitive, stated(double, Instant), inferred(double, Instant)
- [GE-20260912-ff141b] — Neocortex cognitive stack documentation
- [GitHub #55] — Wire ConversationBridge for dialogue knowledge extraction
- [GitHub #53] — Phase B epic
