# Proactive Regulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #819 — feat: proactive regulation — pre-appraisal situation avoidance from emotional memory
**Issue group:** #819 (within epic #851)

**Goal:** Add anticipatory avoidance so characters avoid situations that previously caused emotional distress, before appraisal fires.

**Architecture:** A new `ProactiveRegulationParticipant` in the FOUNDATION phase queries MindMap for past high-intensity regulation episodes, evaluates whether the current context matches a triggering situation, and generates avoidance goals via `appendGoals()`. Context (location, agents, topic) flows through a new `RegulationContext` record on `CognitionTickContext`. Temporal decay prevents permanent avoidance bias.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Quarkus config

## Global Constraints

- All new types in cognition-api must be records or enums (no mutable state)
- All new code in cognition must follow existing participant patterns (constructor injection, no CDI in domain classes)
- Tests use AssertJ assertions, not JUnit assertEquals
- Use `ide_insert_member` for new methods, `ide_create_file` for new files
- Commit after each task with `feat(#819):` or `test(#819):` prefix

---

## Batch 1: API Types — RegulationContext and CognitionTickContext

### Task 1: RegulationContext record and ProactiveAvoidanceMode enum

**Files:**
- Create: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/RegulationContext.java`
- Create: `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveAvoidanceMode.java`
- Test: `cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/RegulationContextTest.java`

**Interfaces:**
- Produces: `RegulationContext(String location, List<String> interactingAgents, String topic)` — consumed by Task 2 (CognitionTickContext), Task 3 (MindMapAdapter write), Task 5 (ProactiveRegulationParticipant)
- Produces: `ProactiveAvoidanceMode { LOCATION_ONLY, LOCATION_AND_AGENTS, FULL }` — consumed by Task 5

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.neocortex.cognition.regulation;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class RegulationContextTest {

    @Test
    void recordFieldsAccessible() {
        var ctx = new RegulationContext("library", List.of("dick-dastardly"), "treasure");
        assertThat(ctx.location()).isEqualTo("library");
        assertThat(ctx.interactingAgents()).containsExactly("dick-dastardly");
        assertThat(ctx.topic()).isEqualTo("treasure");
    }

    @Test
    void nullFieldsAllowed() {
        var ctx = new RegulationContext(null, null, null);
        assertThat(ctx.location()).isNull();
        assertThat(ctx.interactingAgents()).isNull();
        assertThat(ctx.topic()).isNull();
    }

    @Test
    void matchesLocation() {
        var ctx = new RegulationContext("library", null, null);
        assertThat(ctx.matchesDimension("library", ProactiveAvoidanceMode.LOCATION_ONLY)).isTrue();
        assertThat(ctx.matchesDimension("kitchen", ProactiveAvoidanceMode.LOCATION_ONLY)).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=RegulationContextTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — class not found

- [ ] **Step 3: Create RegulationContext record**

Use `ide_create_file` to create `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/RegulationContext.java`:

```java
package io.casehub.neocortex.cognition.regulation;

import org.jspecify.annotations.Nullable;
import java.util.List;

public record RegulationContext(
        @Nullable String location,
        @Nullable List<String> interactingAgents,
        @Nullable String topic
) {
    public boolean matchesDimension(@Nullable String candidateLocation,
                                     ProactiveAvoidanceMode mode) {
        if (location == null || candidateLocation == null) return false;
        if (!location.equalsIgnoreCase(candidateLocation)) return false;
        return true;
    }

    public boolean matchesAgents(@Nullable List<String> candidateAgents) {
        if (interactingAgents == null || candidateAgents == null) return false;
        return interactingAgents.stream().anyMatch(candidateAgents::contains);
    }
}
```

- [ ] **Step 4: Create ProactiveAvoidanceMode enum**

Use `ide_create_file` to create `cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveAvoidanceMode.java`:

```java
package io.casehub.neocortex.cognition.regulation;

public enum ProactiveAvoidanceMode {
    LOCATION_ONLY,
    LOCATION_AND_AGENTS,
    FULL
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=RegulationContextTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (3 tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition-api/src/main/java/io/casehub/neocortex/cognition/regulation/ cognition-api/src/test/java/io/casehub/neocortex/cognition/regulation/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#819): add RegulationContext record and ProactiveAvoidanceMode enum"
```

### Task 2: Add RegulationContext to CognitionTickContext

**Files:**
- Modify: `cognition-api/src/main/java/io/casehub/neocortex/cognition/core/CognitionTickContext.java:7-36`
- Modify: `cognition-api/src/test/java/io/casehub/neocortex/cognition/core/CognitionTickContextTest.java`

**Interfaces:**
- Consumes: `RegulationContext` from Task 1
- Produces: `CognitionTickContext.regulationContext()` — consumed by Task 5

- [ ] **Step 1: Write the failing test**

Add to `CognitionTickContextTest.java`:

```java
@Test
void regulationContextAccessibleViaFullConstructor() {
    var regCtx = new RegulationContext("library", List.of("agent-a"), "treasure");
    var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
            "obs", null, null, regCtx);
    assertThat(ctx.regulationContext()).isEqualTo(regCtx);
}

@Test
void existingConstructorsPassNullRegulationContext() {
    var ctx = new CognitionTickContext("agent1", "tenant1", null, null);
    assertThat(ctx.regulationContext()).isNull();
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=CognitionTickContextTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — no 8-arg constructor

- [ ] **Step 3: Add regulationContext field and 8-arg constructor**

Add to the CognitionTickContext record:
- New field: `@Nullable RegulationContext regulationContext`
- New canonical constructor becomes 8-arg (add `regulationContext` as the 8th parameter)
- Update existing 7-arg constructor to chain to 8-arg with `null` for regulationContext
- Update 6-arg constructor to chain to 7-arg (unchanged chain)
- All existing constructors pass `null` for the new field

The record declaration becomes:
```java
public record CognitionTickContext(
        String agentId,
        String tenantId,
        @Nullable AgentDescriptor descriptor,
        SubjectResolver resolver,
        @Nullable String observation,
        @Nullable AttentionBriefing briefing,
        @Nullable String targetAgentId,
        @Nullable RegulationContext regulationContext
) {
    public CognitionTickContext(String agentId, String tenantId,
                                AgentDescriptor descriptor, SubjectResolver resolver) {
        this(agentId, tenantId, descriptor, resolver, null, null, null, null);
    }
    public CognitionTickContext(String agentId, String tenantId,
                                AgentDescriptor descriptor, SubjectResolver resolver,
                                String observation) {
        this(agentId, tenantId, descriptor, resolver, observation, null, null, null);
    }
    public CognitionTickContext(String agentId, String tenantId,
                                AgentDescriptor descriptor, SubjectResolver resolver,
                                String observation, AttentionBriefing briefing) {
        this(agentId, tenantId, descriptor, resolver, observation, briefing, null, null);
    }
    public CognitionTickContext(String agentId, String tenantId,
                                AgentDescriptor descriptor, SubjectResolver resolver,
                                String observation, AttentionBriefing briefing,
                                String targetAgentId) {
        this(agentId, tenantId, descriptor, resolver, observation, briefing, targetAgentId, null);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=CognitionTickContextTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS

- [ ] **Step 5: Verify no downstream compilation breakage**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml compile -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: BUILD SUCCESS (all callers use telescoping constructors)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition-api/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#819): add RegulationContext field to CognitionTickContext"
```

## Batch 2: Infrastructure — Storage Enrichment and Goal Append

### Task 3: Enrich regulation MindMap nodes with context properties

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationMindMapAdapter.java:26-48`
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationMindMapTest.java`

**Interfaces:**
- Consumes: `RegulationContext` from Task 1
- Produces: `write()` overload accepting `RegulationContext` — consumed by existing `EmotionRegulationTickParticipant` (Task 6 wiring)

- [ ] **Step 1: Write the failing test**

Add to `EmotionRegulationMindMapTest.java`:

```java
@Test
void writeStoresRegulationContext() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.COGNITIVE_CHANGE,
            List.of("ANGER"), List.of("COMPETENCE"),
            0.7, 0.3, 0.8, "reframed", DriveAxis.COMPETENCE, null);
    var regCtx = new RegulationContext("library", List.of("dick-dastardly", "hooded-claw"), "treasure");
    EmotionRegulationMindMapAdapter.write(store, "agent1", "tenant1",
            "cog-subgraph", result, "ANGER", regCtx);

    var nodes = store.search(MindMapQuery.of("tenant1", 10).withType(SubgraphTypes.COGNITIVE));
    var regNode = nodes.stream()
            .filter(n -> "emotion-regulation-result".equals(n.properties().get("cognitiveKind")))
            .findFirst().orElseThrow();
    assertThat(regNode.properties().get("regulationLocation")).isEqualTo("library");
    assertThat(regNode.properties().get("regulationAgents")).isEqualTo("dick-dastardly,hooded-claw");
    assertThat(regNode.properties().get("regulationTopic")).isEqualTo("treasure");
}

@Test
void writeHandlesNullRegulationContext() {
    var result = new EmotionRegulationResult(
            EmotionRegulationStrategy.COGNITIVE_CHANGE,
            List.of("ANGER"), List.of("COMPETENCE"),
            0.7, 0.3, 0.8, "reframed", DriveAxis.COMPETENCE, null);
    EmotionRegulationMindMapAdapter.write(store, "agent1", "tenant1",
            "cog-subgraph", result, "ANGER", null);

    var nodes = store.search(MindMapQuery.of("tenant1", 10).withType(SubgraphTypes.COGNITIVE));
    var regNode = nodes.stream()
            .filter(n -> "emotion-regulation-result".equals(n.properties().get("cognitiveKind")))
            .findFirst().orElseThrow();
    assertThat(regNode.properties().get("regulationLocation")).isNull();
    assertThat(regNode.properties().get("regulationAgents")).isNull();
    assertThat(regNode.properties().get("regulationTopic")).isNull();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=EmotionRegulationMindMapTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — no matching write() overload

- [ ] **Step 3: Add write() overload accepting RegulationContext**

Add a new `write()` overload to `EmotionRegulationMindMapAdapter` that accepts `@Nullable RegulationContext regulationContext` as the last parameter. After the existing property population, add:

```java
if (regulationContext != null) {
    if (regulationContext.location() != null) {
        props.put("regulationLocation", regulationContext.location());
    }
    if (regulationContext.interactingAgents() != null && !regulationContext.interactingAgents().isEmpty()) {
        props.put("regulationAgents", String.join(",", regulationContext.interactingAgents()));
    }
    if (regulationContext.topic() != null) {
        props.put("regulationTopic", regulationContext.topic());
    }
}
```

Chain the existing `write()` overloads to pass `null` for the new parameter so they continue working.

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=EmotionRegulationMindMapTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (all tests including existing ones)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationMindMapAdapter.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationMindMapTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#819): store regulationLocation/Agents/Topic on regulation MindMap nodes"
```

### Task 4: Add appendGoals() to GoalProposalOrchestrator

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestrator.java:116-137`
- Modify: `cognition/src/test/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestratorTest.java`

**Interfaces:**
- Produces: `appendGoals(String agentId, String tenantId, List<DriveGoalProposal> goals)` — consumed by Task 5

- [ ] **Step 1: Write the failing test**

Add to `GoalProposalOrchestratorTest.java`:

```java
@Test
void appendGoalsAddsToExistingRegisteredGoals() {
    var goal1 = new DriveGoalProposal(DriveAxis.AUTONOMY, "goal-1", "desc1", "reason1", 0.5, null, null);
    var goal2 = new DriveGoalProposal(DriveAxis.AUTONOMY, "goal-2", "desc2", "reason2", 0.6, null, null);
    var goal3 = new DriveGoalProposal(DriveAxis.AUTONOMY, "goal-3", "desc3", "reason3", 0.7, null, null);

    orchestrator.registerGoals("agent1", "tenant1", List.of(goal1));
    orchestrator.appendGoals("agent1", "tenant1", List.of(goal2, goal3));

    var registered = orchestrator.registeredGoals("agent1", "tenant1");
    assertThat(registered).hasSize(3);
    assertThat(registered).extracting(DriveGoalProposal::goalName)
            .containsExactly("goal-1", "goal-2", "goal-3");
}

@Test
void registerGoalsReplacesAppendedGoals() {
    var goal1 = new DriveGoalProposal(DriveAxis.AUTONOMY, "appended", "desc", "reason", 0.5, null, null);
    var goal2 = new DriveGoalProposal(DriveAxis.AUTONOMY, "replaced", "desc", "reason", 0.6, null, null);

    orchestrator.appendGoals("agent1", "tenant1", List.of(goal1));
    orchestrator.registerGoals("agent1", "tenant1", List.of(goal2));

    var registered = orchestrator.registeredGoals("agent1", "tenant1");
    assertThat(registered).hasSize(1);
    assertThat(registered.get(0).goalName()).isEqualTo("replaced");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=GoalProposalOrchestratorTest#appendGoals* -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — method not found

- [ ] **Step 3: Implement appendGoals()**

Add to `GoalProposalOrchestrator` after `registerGoals()` (line ~121):

```java
public void appendGoals(String agentId, String tenantId, List<DriveGoalProposal> goals) {
    var state = stateFor(agentId, tenantId);
    var merged = new java.util.ArrayList<>(state.registeredGoals);
    merged.addAll(goals);
    state.registeredGoals = List.copyOf(merged);
}
```

The `stateFor()` method already exists (creates or retrieves per-agent state).

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=GoalProposalOrchestratorTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (all tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestrator.java cognition/src/test/java/io/casehub/neocortex/cognition/goal/GoalProposalOrchestratorTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#819): add appendGoals() with additive semantics to GoalProposalOrchestrator"
```

## Batch 3: Participant — ProactiveRegulationParticipant and Wiring

### Task 5: ProactiveRegulationParticipant implementation

**Files:**
- Create: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipant.java`
- Create: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipantTest.java`

**Interfaces:**
- Consumes: `RegulationContext` from Task 1, `EmotionRegulationMindMapAdapter.loadHistory()` (existing), `GoalProposalOrchestrator.appendGoals()` from Task 4, `RegulationDefaults.redirectTendency()` (existing)
- Produces: `ProactiveRegulationParticipant` implementing `CognitionTickParticipant` — consumed by Task 6 (CognitionCore wiring)

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.core.CognitionTickContext;
import io.casehub.neocortex.cognition.drive.DriveAxis;
import io.casehub.neocortex.cognition.goal.DriveGoalProposal;
import io.casehub.neocortex.cognition.goal.GoalProposalOrchestrator;
import io.casehub.neocortex.cognitive.index.CognitiveDefaults;
import io.casehub.neocortex.cognitive.index.CognitiveDefaultsRegistry;
import io.casehub.neocortex.cognitive.index.RegulationDefaults;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ProactiveRegulationParticipantTest {

    private MindMapStore mindMapStore;
    private GoalProposalOrchestrator goalOrchestrator;
    private CognitiveDefaultsRegistry defaultsRegistry;
    private ProactiveRegulationParticipant participant;

    @BeforeEach
    void setUp() {
        mindMapStore = mock(MindMapStore.class);
        goalOrchestrator = mock(GoalProposalOrchestrator.class);
        defaultsRegistry = mock(CognitiveDefaultsRegistry.class);
        participant = new ProactiveRegulationParticipant(
                mindMapStore, goalOrchestrator, defaultsRegistry,
                "cog-subgraph", ProactiveAvoidanceMode.LOCATION_ONLY, 50, 10.0);
    }

    @Test
    void skipsWhenNoRegulationContext() {
        var ctx = new CognitionTickContext("agent1", "tenant1", null, null);
        participant.tick(ctx);
        verifyNoInteractions(goalOrchestrator);
    }

    @Test
    void skipsWhenNoPersonalityDefaults() {
        var regCtx = new RegulationContext("library", null, null);
        var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
                null, null, null, regCtx);
        when(defaultsRegistry.forAgent("agent1")).thenReturn(Optional.empty());
        participant.tick(ctx);
        verifyNoInteractions(goalOrchestrator);
    }

    @Test
    void generatesAvoidanceGoalForMatchingLocation() {
        var regCtx = new RegulationContext("library", null, null);
        var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
                null, null, null, regCtx);

        var rd = new RegulationDefaults(0.5, 0.5, 0.8);
        var defaults = mock(CognitiveDefaults.class);
        when(defaults.regulationDefaults()).thenReturn(rd);
        when(defaultsRegistry.forAgent("agent1")).thenReturn(Optional.of(defaults));

        var node = mockRegulationNode("library", "ANGER",
                0.2, Instant.now().minusSeconds(30));
        when(mindMapStore.search(any())).thenReturn(List.of(node));

        participant.tick(ctx);

        verify(goalOrchestrator).appendGoals(eq("agent1"), eq("tenant1"), argThat(goals -> {
            assertThat(goals).hasSize(1);
            assertThat(goals.get(0).goalName()).contains("proactive-avoidance");
            assertThat(goals.get(0).goalName()).contains("library");
            return true;
        }));
    }

    @Test
    void noGoalWhenLocationDoesNotMatch() {
        var regCtx = new RegulationContext("kitchen", null, null);
        var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
                null, null, null, regCtx);

        var rd = new RegulationDefaults(0.5, 0.5, 0.8);
        var defaults = mock(CognitiveDefaults.class);
        when(defaults.regulationDefaults()).thenReturn(rd);
        when(defaultsRegistry.forAgent("agent1")).thenReturn(Optional.of(defaults));

        var node = mockRegulationNode("library", "ANGER",
                0.2, Instant.now().minusSeconds(30));
        when(mindMapStore.search(any())).thenReturn(List.of(node));

        participant.tick(ctx);
        verify(goalOrchestrator, never()).appendGoals(any(), any(), any());
    }

    @Test
    void noGoalWhenEpisodeFullyDecayed() {
        var regCtx = new RegulationContext("library", null, null);
        var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
                null, null, null, regCtx);

        var rd = new RegulationDefaults(0.5, 0.5, 0.8);
        var defaults = mock(CognitiveDefaults.class);
        when(defaults.regulationDefaults()).thenReturn(rd);
        when(defaultsRegistry.forAgent("agent1")).thenReturn(Optional.of(defaults));

        // 600 seconds ago / 10s per tick = 60 ticks > 50 decay horizon
        var node = mockRegulationNode("library", "ANGER",
                0.2, Instant.now().minusSeconds(600));
        when(mindMapStore.search(any())).thenReturn(List.of(node));

        participant.tick(ctx);
        verify(goalOrchestrator, never()).appendGoals(any(), any(), any());
    }

    @Test
    void lowRedirectTendencyBlocksAvoidance() {
        var regCtx = new RegulationContext("library", null, null);
        var ctx = new CognitionTickContext("agent1", "tenant1", null, null,
                null, null, null, regCtx);

        // Low redirect = high threshold = hard to trigger
        var rd = new RegulationDefaults(0.5, 0.5, 0.1);
        var defaults = mock(CognitiveDefaults.class);
        when(defaults.regulationDefaults()).thenReturn(rd);
        when(defaultsRegistry.forAgent("agent1")).thenReturn(Optional.of(defaults));

        // Moderate effectiveness (0.5) → avoidance score = (1-0.5)*decay ≈ 0.45
        // Threshold = 1.0 - 0.1 = 0.9 → score < threshold → no avoidance
        var node = mockRegulationNode("library", "ANGER",
                0.5, Instant.now().minusSeconds(30));
        when(mindMapStore.search(any())).thenReturn(List.of(node));

        participant.tick(ctx);
        verify(goalOrchestrator, never()).appendGoals(any(), any(), any());
    }

    private MindMapNode mockRegulationNode(String location, String emotionType,
                                            double effectiveness, Instant timestamp) {
        var node = mock(MindMapNode.class);
        var props = new HashMap<String, String>();
        props.put("cognitiveKind", "emotion-regulation-result");
        props.put("agent-id", "agent1");
        props.put("regulationLocation", location);
        props.put("dominantEmotionType", emotionType);
        props.put("effectiveness", String.valueOf(effectiveness));
        props.put("timestamp", timestamp.toString());
        props.put("regulationAxis", "AUTONOMY");
        when(node.properties()).thenReturn(props);
        return node;
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=ProactiveRegulationParticipantTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: FAIL — class not found

- [ ] **Step 3: Implement ProactiveRegulationParticipant**

Use `ide_create_file` to create `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipant.java`:

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.core.CognitionTickContext;
import io.casehub.neocortex.cognition.core.CognitionTickParticipant;
import io.casehub.neocortex.cognition.drive.DriveAxis;
import io.casehub.neocortex.cognition.goal.DriveGoalProposal;
import io.casehub.neocortex.cognition.goal.GoalProposalOrchestrator;
import io.casehub.neocortex.cognitive.index.CognitiveDefaultsRegistry;
import io.casehub.neocortex.cognitive.index.RegulationDefaults;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapQuery;
import io.casehub.neocortex.mindmap.MindMapStore;
import io.casehub.neocortex.mindmap.SubgraphTypes;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

public final class ProactiveRegulationParticipant implements CognitionTickParticipant {

    private final MindMapStore mindMapStore;
    private final GoalProposalOrchestrator goalOrchestrator;
    private final CognitiveDefaultsRegistry defaultsRegistry;
    private final String cognitiveSubgraphId;
    private final ProactiveAvoidanceMode mode;
    private final int decayHorizonTicks;
    private final double tickIntervalSeconds;

    public ProactiveRegulationParticipant(MindMapStore mindMapStore,
                                          GoalProposalOrchestrator goalOrchestrator,
                                          CognitiveDefaultsRegistry defaultsRegistry,
                                          String cognitiveSubgraphId,
                                          ProactiveAvoidanceMode mode,
                                          int decayHorizonTicks,
                                          double tickIntervalSeconds) {
        this.mindMapStore = mindMapStore;
        this.goalOrchestrator = goalOrchestrator;
        this.defaultsRegistry = defaultsRegistry;
        this.cognitiveSubgraphId = cognitiveSubgraphId;
        this.mode = mode;
        this.decayHorizonTicks = decayHorizonTicks;
        this.tickIntervalSeconds = tickIntervalSeconds;
    }

    @Override
    public void tick(CognitionTickContext context) {
        var regCtx = context.regulationContext();
        if (regCtx == null || regCtx.location() == null) return;

        var defaults = defaultsRegistry.forAgent(context.agentId());
        if (defaults.isEmpty()) return;

        var rd = defaults.get().regulationDefaults();
        if (rd == null) return;

        double threshold = 1.0 - rd.redirectTendency();
        var now = Instant.now();

        var nodes = mindMapStore.search(
                MindMapQuery.of(context.tenantId(), 200)
                            .withType(SubgraphTypes.COGNITIVE));

        List<DriveGoalProposal> avoidanceGoals = new ArrayList<>();

        for (var node : nodes) {
            var props = node.properties();
            if (!EmotionRegulationMindMapAdapter.COGNITIVE_KIND.equals(props.get("cognitiveKind"))) continue;
            if (!context.agentId().equals(props.get("agent-id"))) continue;

            String episodeLocation = props.get("regulationLocation");
            if (!regCtx.location().equalsIgnoreCase(episodeLocation)) continue;

            if (mode == ProactiveAvoidanceMode.LOCATION_AND_AGENTS || mode == ProactiveAvoidanceMode.FULL) {
                String episodeAgents = props.get("regulationAgents");
                if (regCtx.interactingAgents() != null && episodeAgents != null) {
                    var episodeAgentList = List.of(episodeAgents.split(","));
                    if (regCtx.interactingAgents().stream().noneMatch(episodeAgentList::contains)) continue;
                }
            }

            String timestampStr = props.get("timestamp");
            if (timestampStr == null) continue;
            Instant episodeTime = Instant.parse(timestampStr);
            double elapsedTicks = Duration.between(episodeTime, now).toSeconds() / tickIntervalSeconds;
            double decayWeight = 1.0 - (elapsedTicks / decayHorizonTicks);
            if (decayWeight <= 0) continue;

            double effectiveness = parseDouble(props.get("effectiveness"));
            double avoidanceScore = (1.0 - effectiveness) * decayWeight;
            if (avoidanceScore <= threshold) continue;

            String emotionType = props.getOrDefault("dominantEmotionType", "UNKNOWN");
            String axisStr = props.get("regulationAxis");
            DriveAxis axis = DriveAxis.AUTONOMY;
            if (axisStr != null) {
                try { axis = DriveAxis.valueOf(axisStr); }
                catch (IllegalArgumentException ignored) {}
            }

            avoidanceGoals.add(new DriveGoalProposal(
                    axis,
                    "proactive-avoidance-" + regCtx.location().toLowerCase() + "-" + emotionType.toLowerCase(),
                    "Avoid " + regCtx.location() + " — previous emotional distress (" + emotionType + ")",
                    "Proactive regulation: memory-based anticipatory avoidance",
                    avoidanceScore,
                    null, null));
        }

        if (!avoidanceGoals.isEmpty()) {
            goalOrchestrator.appendGoals(context.agentId(), context.tenantId(), avoidanceGoals);
        }
    }

    private static double parseDouble(String value) {
        if (value == null) return 0.0;
        try { return Double.parseDouble(value); }
        catch (NumberFormatException e) { return 0.0; }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=ProactiveRegulationParticipantTest -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipant.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationParticipantTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#819): implement ProactiveRegulationParticipant with decay and personality modulation"
```

### Task 6: Wire into CognitionCore and pass RegulationContext from tick participant

**Files:**
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:758-800`
- Modify: `cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java:172-176`
- Create: `cognition/src/test/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationPipelineTest.java`

**Interfaces:**
- Consumes: `ProactiveRegulationParticipant` from Task 5, `RegulationContext` from Task 1
- Produces: Wired pipeline — FOUNDATION phase includes proactive regulation participant

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.neocortex.cognition.regulation;

import io.casehub.neocortex.cognition.core.CognitionCore;
import io.casehub.neocortex.cognition.core.CognitionPhase;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class ProactiveRegulationPipelineTest {

    @Test
    void proactiveParticipantRegisteredInFoundationPhase() {
        // Verify CognitionCore.configureProactiveRegulation() registers
        // the participant in FOUNDATION phase
        // This test will use CognitionCore's participant list inspection
        // to verify ordering
    }
}
```

The actual test will verify that after `configureProactiveRegulation()` is called, the participant list for FOUNDATION contains the ProactiveRegulationParticipant after SubThoughtTickParticipant.

- [ ] **Step 2: Add configureProactiveRegulation() to CognitionCore**

Add a new configuration method to `CognitionCore` following the pattern of `configureRegulation()`:

```java
public void configureProactiveRegulation(GoalProposalOrchestrator goalOrchestrator,
                                          MindMapStore mindMapStore,
                                          CognitiveDefaultsRegistry defaultsRegistry,
                                          String cognitiveSubgraphId,
                                          ProactiveAvoidanceMode mode,
                                          int decayHorizonTicks,
                                          double tickIntervalSeconds) {
    this.proactiveRegulationParticipant = new ProactiveRegulationParticipant(
            mindMapStore, goalOrchestrator, defaultsRegistry,
            cognitiveSubgraphId, mode, decayHorizonTicks, tickIntervalSeconds);
    addParticipant(CognitionPhase.FOUNDATION, this.proactiveRegulationParticipant);
}
```

Add the field: `private ProactiveRegulationParticipant proactiveRegulationParticipant;`

- [ ] **Step 3: Pass RegulationContext through to EmotionRegulationTickParticipant write call**

In `EmotionRegulationTickParticipant.tick()` at line ~172-176, update the `write()` call to pass the `RegulationContext` from the tick context:

```java
if (mindMapStore != null && cognitiveSubgraphId != null) {
    EmotionRegulationMindMapAdapter.write(
            mindMapStore, context.agentId(), context.tenantId(),
            cognitiveSubgraphId, result,
            dominantType != null ? dominantType.name() : null,
            context.regulationContext());
}
```

- [ ] **Step 4: Write pipeline ordering test**

Verify that FOUNDATION participants are ordered: SubThought before Proactive.

- [ ] **Step 5: Run all regulation tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest="*Regulation*,*Proactive*" -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -pl .`
Expected: PASS (all tests)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java cognition/src/main/java/io/casehub/neocortex/cognition/regulation/EmotionRegulationTickParticipant.java cognition/src/test/java/io/casehub/neocortex/cognition/regulation/ProactiveRegulationPipelineTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#819): wire ProactiveRegulationParticipant into CognitionCore FOUNDATION phase"
```

## References

- [2026-11-08-proactive-regulation-design.md] — design spec this plan implements
- [EmotionRegulationMindMapAdapter.java:29-48] — existing regulation node storage
- [EmotionRegulationTickParticipant.java:172-176] — existing write() call site
- [GoalProposalOrchestrator.java:116-121] — registerGoals() API
- [CognitionTickContext.java:7-36] — telescoping constructors
- [CognitionCore.java:758,793] — configureRegulation() and configureSubThoughts() patterns
- [EmotionRegulationGoalParticipant.java:22-37] — reactive goal bridge pattern
- GitHub casehubio/neocortex#819 — focal issue
- GitHub casehubio/neocortex#851 — parent epic
