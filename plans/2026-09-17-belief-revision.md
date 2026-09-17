# Belief Revision Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #67 — Belief revision from contradicting evidence
**Issue group:** #63, #283, #66, #70, #67 (current branch covers all)

**Goal:** Add a consolidation phase that uses LLM-assessed contradiction detection to gradually erode belief confidence and supersede stale beliefs with revised ones.

**Architecture:** New `BeliefRevisionPhase` (@Priority 16) in blocks-core scans Belieflike nodes and newly graduated evidence across all cognitive subgraphs, groups by agent, invokes the LLM once per agent to detect contradictions, applies variable-strength confidence decay, and supersedes beliefs when confidence drops below threshold. `CharacterCognition` transitions from static YAML belief rendering to MindMapStore-sourced rendering via `CognitiveObservationSections.beliefsSection()`.

**Tech Stack:** Java 26, Quarkus, CDI, MindMapStore (neocortex), AgentProvider (platform-agent-api), InMemoryMindMapStore (test)

## Global Constraints

- `BeliefRevisionPhase` @Priority(16) — after ExperienceConsolidation@15, before DriveAdaptation@17
- NOT a CDI bean — constructor-instantiated like DriveAdaptationPhase
- Confidence origin transitions STATED → INFERRED on first decay
- `decayReference` reset to `Instant.now()` on each decay (prevents compounding with ConfidenceDecayDecorator)
- One LLM call per agent per consolidation cycle
- Failure handling: catch + log WARNING, phase returns without modifying beliefs
- Cross-subgraph scan: beliefs in `beliefs-{agentId}`, evidence in shared cognitive subgraph
- Beliefs grouped by `node.principalId().id()`, evidence by `node.properties().get("agent-id")`
- `nodesIn()` already filters superseded nodes — active beliefs only
- Revised beliefs: provenance `belief-revision`, confidence INFERRED at 0.6

---

## Batch 1: BeliefRevisionPhase + config (blocks-core)

### Task 1: BeliefRevisionConfig + BeliefRevisionPhase

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/belief/BeliefRevisionConfig.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/belief/BeliefRevisionPhase.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/engine/BeliefRevisionIntegrationTest.java`

**Interfaces:**
- Consumes: `ConsolidationPhase` (neocortex-mindmap-intelligence), `MindMapStore` (neocortex-mindmap-api), `AgentProvider` (platform-agent-api), `AgentSessionConfig.of(String, String)`, `AgentEvent.TextDelta`, `Confidence.inferred(double, Instant)`, `NodeInput`, `NodeUpdate`, `MindMapStore.supersede()`
- Produces: `BeliefRevisionPhase` — ConsolidationPhase @Priority(16). Constructor: `(MindMapStore, AgentProvider, BeliefRevisionConfig)`. `BeliefRevisionConfig` — record with `defaults()`.

- [ ] **Step 1: Write the integration test**

Create test in wacky-manor with a stub AgentProvider:

```java
package io.casehub.examples.manor.engine;

import io.casehub.blocks.agentic.social.belief.BeliefRevisionConfig;
import io.casehub.blocks.agentic.social.belief.BeliefRevisionPhase;
import io.casehub.examples.manor.agent.ManorCognitiveSeeder;
import io.casehub.examples.manor.agent.ManorSocialConfigLoader;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class BeliefRevisionIntegrationTest {

    private static final String TENANT = "revision-test";
    private static final String AGENT = "hooded-claw";

    @Test
    void fullLifecycle_seedContradictSupersede() {
        var store = new InMemoryMindMapStore();
        var seeder = new ManorCognitiveSeeder(store);
        var allConfigs = ManorSocialConfigLoader.load();

        var seedResult = seeder.seed(AGENT, allConfigs.get(AGENT), TENANT);

        // Add graduated evidence that contradicts "Penelope is naive"
        for (int i = 0; i < 5; i++) {
            store.addNode(
                NodeInput.of("Penelope outsmarts Hooded Claw attempt " + i, seedResult.subgraphId())
                    .withProvenance("experience-consolidation")
                    .withProperties(Map.of(
                        "cognitiveKind", "experience",
                        "agent-id", AGENT,
                        "event-type", "social_interaction"))
                    .withConfidence(io.casehub.neocortex.cognitive.Confidence.inferred(0.7, Instant.now())),
                TENANT);
        }

        // Each call returns strong contradiction for "penelope-awareness" belief
        var stubProvider = new StubAgentProvider("""
            {"contradictions": [{
                "beliefNodeId": "PLACEHOLDER",
                "beliefText": "Penelope is naive and trusts too easily",
                "contradictingEvidence": "Penelope outsmarts Hooded Claw",
                "reasoning": "Penelope demonstrated perceptiveness",
                "contradictionStrength": 0.9,
                "revisedBelief": "Penelope is more perceptive than she appears"
            }]}""");

        var config = new BeliefRevisionConfig(0.15, 0.3, 0.6);
        var phase = new BeliefRevisionPhase(store, stubProvider, config);

        // Run multiple times to accumulate decay past threshold
        for (int i = 0; i < 5; i++) {
            phase.run(TENANT, List.of());
        }

        // Check that the original belief was superseded
        var subgraphs = store.listSubgraphs(TENANT);
        var beliefSg = subgraphs.stream()
            .filter(sg -> sg.name().equals("beliefs-" + AGENT))
            .findFirst().orElseThrow();
        var activeNodes = store.nodesIn(beliefSg.id(), TENANT);

        // Should have the revised belief
        var revisedBelief = activeNodes.stream()
            .filter(n -> n.traits().contains("Belieflike"))
            .filter(n -> "belief-revision".equals(n.provenance()))
            .findFirst();
        assertThat(revisedBelief).isPresent();
        assertThat(revisedBelief.get().name()).contains("perceptive");

        // Original belief should be superseded (not in nodesIn results)
        var originalBelief = activeNodes.stream()
            .filter(n -> n.name().contains("naive"))
            .findFirst();
        assertThat(originalBelief).isEmpty();
    }

    @Test
    void noNewEvidence_beliefsUnchanged() {
        var store = new InMemoryMindMapStore();
        var seeder = new ManorCognitiveSeeder(store);
        var allConfigs = ManorSocialConfigLoader.load();

        seeder.seed(AGENT, allConfigs.get(AGENT), TENANT);

        // No graduated nodes added — no evidence
        var stubProvider = new StubAgentProvider("""
            {"contradictions": []}""");

        var phase = new BeliefRevisionPhase(store, stubProvider, BeliefRevisionConfig.defaults());
        phase.run(TENANT, List.of());

        var subgraphs = store.listSubgraphs(TENANT);
        var beliefSg = subgraphs.stream()
            .filter(sg -> sg.name().equals("beliefs-" + AGENT))
            .findFirst().orElseThrow();
        var nodes = store.nodesIn(beliefSg.id(), TENANT);
        var beliefs = nodes.stream()
            .filter(n -> n.traits().contains("Belieflike"))
            .toList();

        // Original beliefs unchanged
        assertThat(beliefs).hasSize(2); // hooded-claw has 2 initial beliefs
        beliefs.forEach(b ->
            assertThat(b.confidence().value()).isEqualTo(0.8));
    }

    @Test
    void weakContradiction_decaysButDoesNotSupersede() {
        var store = new InMemoryMindMapStore();
        var seeder = new ManorCognitiveSeeder(store);
        var allConfigs = ManorSocialConfigLoader.load();

        var seedResult = seeder.seed(AGENT, allConfigs.get(AGENT), TENANT);

        store.addNode(
            NodeInput.of("Penelope seemed slightly less naive", seedResult.subgraphId())
                .withProvenance("experience-consolidation")
                .withProperties(Map.of(
                    "cognitiveKind", "experience",
                    "agent-id", AGENT,
                    "event-type", "social_interaction"))
                .withConfidence(io.casehub.neocortex.cognitive.Confidence.inferred(0.5, Instant.now())),
            TENANT);

        // Weak contradiction
        var stubProvider = new StubAgentProvider("""
            {"contradictions": [{
                "beliefNodeId": "PLACEHOLDER",
                "beliefText": "Penelope is naive and trusts too easily",
                "contradictingEvidence": "Penelope seemed slightly less naive",
                "reasoning": "Minor evidence of awareness",
                "contradictionStrength": 0.3,
                "revisedBelief": "Penelope may be less naive than assumed"
            }]}""");

        var config = new BeliefRevisionConfig(0.15, 0.3, 0.6);
        var phase = new BeliefRevisionPhase(store, stubProvider, config);
        phase.run(TENANT, List.of());

        var subgraphs = store.listSubgraphs(TENANT);
        var beliefSg = subgraphs.stream()
            .filter(sg -> sg.name().equals("beliefs-" + AGENT))
            .findFirst().orElseThrow();
        var nodes = store.nodesIn(beliefSg.id(), TENANT);

        // Belief should still be active but with reduced confidence
        var penelopeBelief = nodes.stream()
            .filter(n -> n.traits().contains("Belieflike"))
            .filter(n -> n.name().contains("naive"))
            .findFirst().orElseThrow();

        // 0.8 - (0.15 * 0.3) = 0.755
        assertThat(penelopeBelief.confidence().value()).isLessThan(0.8);
        assertThat(penelopeBelief.confidence().value()).isGreaterThan(0.3);

        // No revised belief created
        var revised = nodes.stream()
            .filter(n -> "belief-revision".equals(n.provenance()))
            .findFirst();
        assertThat(revised).isEmpty();
    }

    static class StubAgentProvider implements AgentProvider {
        private final String response;

        StubAgentProvider(String response) { this.response = response; }

        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            return Multi.createFrom().item(new AgentEvent.TextDelta(response));
        }

        @Override
        public io.casehub.platform.agent.AgentSession openSession(
                io.casehub.platform.agent.AgentSessionInit init) {
            throw new UnsupportedOperationException();
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s .mvn/slot-settings.xml -Dtest=BeliefRevisionIntegrationTest`
Expected: FAIL — `BeliefRevisionConfig` and `BeliefRevisionPhase` do not exist

- [ ] **Step 3: Create BeliefRevisionConfig**

Use `ide_create_file` in blocks-core:

```java
package io.casehub.blocks.agentic.social.belief;

public record BeliefRevisionConfig(
    double beliefDecayPerContradiction,
    double beliefSupersessionThreshold,
    double revisedBeliefInitialConfidence
) {
    public static BeliefRevisionConfig defaults() {
        return new BeliefRevisionConfig(0.15, 0.3, 0.6);
    }
}
```

- [ ] **Step 4: Create BeliefRevisionPhase**

Use `ide_create_file` in blocks-core:

```java
package io.casehub.blocks.agentic.social.belief;

import io.casehub.neocortex.cognitive.Confidence;
import io.casehub.neocortex.cognitive.ConfidenceOrigin;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.NodeUpdate;
import io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase;
import io.casehub.platform.agent.AgentEvent;
import io.casehub.platform.agent.AgentProvider;
import io.casehub.platform.agent.AgentSessionConfig;
import io.casehub.platform.api.identity.PrincipalId;
import jakarta.annotation.Priority;

import java.time.Instant;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.logging.Level;
import java.util.logging.Logger;

@Priority(16)
public class BeliefRevisionPhase implements ConsolidationPhase {

    private static final Logger LOG = Logger.getLogger(BeliefRevisionPhase.class.getName());
    private static final String CURSOR_NODE_NAME = "belief-revision-cursor";
    private static final String BELIEF_TRAIT = "Belieflike";
    private static final String EXPERIENCE_PROVENANCE = "experience-consolidation";
    private static final String REVISION_PROVENANCE = "belief-revision";

    private static final String SYSTEM_PROMPT = """
            You are analyzing a character's beliefs against recent evidence from their experiences.
            For each belief that is contradicted by the evidence, identify the contradiction.
            Respond with JSON only. If no contradictions are found, return {"contradictions": []}.
            """;

    private final MindMapStore mindMapStore;
    private final AgentProvider agentProvider;
    private final BeliefRevisionConfig config;

    public BeliefRevisionPhase(MindMapStore mindMapStore, AgentProvider agentProvider,
                               BeliefRevisionConfig config) {
        this.mindMapStore = mindMapStore;
        this.agentProvider = agentProvider;
        this.config = config;
    }

    @Override
    public String name() {
        return "belief-revision";
    }

    @Override
    public void run(String tenantId, List<String> subgraphPriority) {
        var subgraphs = mindMapStore.listSubgraphs(tenantId);
        var cognitiveSubgraphs = subgraphs.stream()
            .filter(sg -> "cognitive".equals(sg.type()))
            .toList();
        if (cognitiveSubgraphs.isEmpty()) return;

        var allBeliefs = new HashMap<String, List<MindMapNode>>();
        var allEvidence = new HashMap<String, List<MindMapNode>>();
        String cursorSubgraphId = null;
        String cursorNodeId = null;
        String lastProcessedId = null;

        for (var sg : cognitiveSubgraphs) {
            var nodes = mindMapStore.nodesIn(sg.id(), tenantId);
            for (var node : nodes) {
                if (CURSOR_NODE_NAME.equals(node.name())) {
                    cursorSubgraphId = sg.id();
                    cursorNodeId = node.id();
                    lastProcessedId = node.properties().get("last-processed-node-id");
                    continue;
                }
                if (node.traits().contains(BELIEF_TRAIT)) {
                    var agentId = node.principalId() != null ? node.principalId().id() : null;
                    if (agentId != null) {
                        allBeliefs.computeIfAbsent(agentId, k -> new ArrayList<>()).add(node);
                    }
                } else if (EXPERIENCE_PROVENANCE.equals(node.provenance())) {
                    var agentId = node.properties().get("agent-id");
                    if (agentId != null) {
                        allEvidence.computeIfAbsent(agentId, k -> new ArrayList<>()).add(node);
                    }
                }
            }
        }

        if (allBeliefs.isEmpty()) return;

        String latestProcessedId = lastProcessedId;
        for (var agentId : allBeliefs.keySet()) {
            var beliefs = allBeliefs.get(agentId);
            var evidence = allEvidence.getOrDefault(agentId, List.of());
            var newEvidence = filterNewEvidence(evidence, lastProcessedId);
            if (newEvidence.isEmpty()) continue;

            processAgent(agentId, beliefs, newEvidence, tenantId);

            var lastEvId = newEvidence.get(newEvidence.size() - 1).id();
            if (latestProcessedId == null || lastEvId.compareTo(latestProcessedId) > 0) {
                latestProcessedId = lastEvId;
            }
        }

        if (latestProcessedId != null && !latestProcessedId.equals(lastProcessedId)) {
            saveCursor(cognitiveSubgraphs.get(0).id(), cursorSubgraphId,
                       cursorNodeId, latestProcessedId, tenantId);
        }
    }

    private List<MindMapNode> filterNewEvidence(List<MindMapNode> evidence, String lastProcessedId) {
        if (lastProcessedId == null) return evidence;
        return evidence.stream()
            .filter(n -> n.id().compareTo(lastProcessedId) > 0)
            .toList();
    }

    private void processAgent(String agentId, List<MindMapNode> beliefs,
                              List<MindMapNode> evidence, String tenantId) {
        try {
            var contradictions = detectContradictions(agentId, beliefs, evidence);
            if (contradictions.isEmpty()) return;

            for (var c : contradictions) {
                var beliefNode = beliefs.stream()
                    .filter(b -> b.id().equals(c.beliefNodeId) || b.name().equals(c.beliefText))
                    .findFirst().orElse(null);
                if (beliefNode == null) continue;

                double currentConfidence = beliefNode.confidence().value();
                double effectiveDecay = config.beliefDecayPerContradiction() * c.contradictionStrength;
                double newConfidence = Math.max(0.0, currentConfidence - effectiveDecay);

                if (newConfidence < config.beliefSupersessionThreshold()) {
                    var subjectKey = beliefNode.property("subject").orElse(beliefNode.name());
                    var newNodeId = mindMapStore.addNode(
                        NodeInput.of(c.revisedBelief, beliefNode.subgraphId())
                            .withConfidence(Confidence.inferred(
                                config.revisedBeliefInitialConfidence(), Instant.now()))
                            .withProvenance(REVISION_PROVENANCE)
                            .withTraits(Set.of(BELIEF_TRAIT))
                            .withProperties(Map.of("subject", subjectKey))
                            .withPrincipalId(PrincipalId.agent(agentId)),
                        tenantId);
                    mindMapStore.supersede(beliefNode.id(), newNodeId, c.reasoning, tenantId);
                } else {
                    mindMapStore.updateNode(beliefNode.id(),
                        NodeUpdate.empty().withConfidence(
                            new Confidence(ConfidenceOrigin.INFERRED, newConfidence, Instant.now())),
                        tenantId);
                }
            }
        } catch (Exception e) {
            LOG.log(Level.WARNING, agentId + ": belief revision failed (non-fatal)", e);
        }
    }

    record Contradiction(String beliefNodeId, String beliefText,
                         String contradictingEvidence, String reasoning,
                         double contradictionStrength, String revisedBelief) {}

    private List<Contradiction> detectContradictions(String agentId,
            List<MindMapNode> beliefs, List<MindMapNode> evidence) {
        var beliefsText = new StringBuilder();
        for (int i = 0; i < beliefs.size(); i++) {
            var b = beliefs.get(i);
            beliefsText.append(String.format("%d. [%s] \"%s\" (confidence: %.2f)\n",
                i + 1, b.id(), b.name(), b.confidence().value()));
        }

        var evidenceText = new StringBuilder();
        for (var e : evidence) {
            var eventType = e.properties().getOrDefault("event-type", "unknown");
            evidenceText.append(String.format("- \"%s\" (event-type: %s)\n", e.name(), eventType));
        }

        var userPrompt = String.format("""
                Character: %s

                Current beliefs:
                %s
                Recent evidence:
                %s
                For each belief contradicted by this evidence, provide JSON:
                - beliefNodeId: the ID in brackets above
                - beliefText: the belief text
                - contradictingEvidence: which evidence contradicts it
                - reasoning: why it's a contradiction
                - contradictionStrength: 0.0 (barely relevant) to 1.0 (directly disproven)
                - revisedBelief: what the character should now believe (one sentence, their perspective)""",
                agentId, beliefsText, evidenceText);

        var sessionConfig = AgentSessionConfig.of(SYSTEM_PROMPT, userPrompt);
        var responseText = new StringBuilder();
        agentProvider.invoke(sessionConfig).subscribe().asStream()
            .filter(e -> e instanceof AgentEvent.TextDelta)
            .map(e -> ((AgentEvent.TextDelta) e).text())
            .forEach(responseText::append);

        return parseContradictions(responseText.toString());
    }

    private List<Contradiction> parseContradictions(String json) {
        json = json.strip();
        if (json.startsWith("```")) {
            json = json.replaceFirst("```[a-z]*\\n?", "").replaceFirst("\\n?```$", "").strip();
        }
        var results = new ArrayList<Contradiction>();
        int searchFrom = 0;
        while (true) {
            int objStart = json.indexOf('{', searchFrom);
            if (objStart < 0) break;
            int objEnd = json.indexOf('}', objStart);
            if (objEnd < 0) break;
            String obj = json.substring(objStart, objEnd + 1);
            searchFrom = objEnd + 1;

            if (!obj.contains("beliefNodeId")) continue;

            var nodeId = extractField(obj, "beliefNodeId");
            var text = extractField(obj, "beliefText");
            var evidence = extractField(obj, "contradictingEvidence");
            var reasoning = extractField(obj, "reasoning");
            var strength = extractDoubleField(obj, "contradictionStrength");
            var revised = extractField(obj, "revisedBelief");

            if (nodeId != null && text != null && strength > 0) {
                results.add(new Contradiction(nodeId, text, evidence,
                    reasoning != null ? reasoning : "LLM-detected contradiction",
                    strength, revised != null ? revised : text + " (revised)"));
            }
        }
        return results;
    }

    private static String extractField(String json, String field) {
        int start = json.indexOf("\"" + field + "\"");
        if (start < 0) return null;
        int colon = json.indexOf(':', start);
        if (colon < 0) return null;
        int quote = json.indexOf('"', colon + 1);
        if (quote < 0) return null;
        quote++;
        var sb = new StringBuilder();
        for (int i = quote; i < json.length(); i++) {
            char c = json.charAt(i);
            if (c == '\\' && i + 1 < json.length()) {
                sb.append(json.charAt(++i));
            } else if (c == '"') {
                break;
            } else {
                sb.append(c);
            }
        }
        return sb.toString();
    }

    private static double extractDoubleField(String json, String field) {
        int start = json.indexOf("\"" + field + "\"");
        if (start < 0) return 0.0;
        int colon = json.indexOf(':', start);
        if (colon < 0) return 0.0;
        var numStr = new StringBuilder();
        for (int i = colon + 1; i < json.length(); i++) {
            char c = json.charAt(i);
            if (c == ',' || c == '}' || c == ']') break;
            if (!Character.isWhitespace(c)) numStr.append(c);
        }
        try {
            return Double.parseDouble(numStr.toString());
        } catch (NumberFormatException e) {
            return 0.0;
        }
    }

    private void saveCursor(String defaultSubgraphId, String cursorSubgraphId,
                            String cursorNodeId, String lastId, String tenantId) {
        String sgId = cursorSubgraphId != null ? cursorSubgraphId : defaultSubgraphId;
        if (cursorNodeId != null) {
            mindMapStore.updateNode(cursorNodeId,
                NodeUpdate.empty().withPropertiesToSet(
                    Map.of("last-processed-node-id", lastId)),
                tenantId);
        } else {
            mindMapStore.addNode(
                NodeInput.of(CURSOR_NODE_NAME, sgId)
                    .withProvenance(REVISION_PROVENANCE)
                    .withProperties(Map.of(
                        "cognitiveKind", "cursor",
                        "last-processed-node-id", lastId)),
                tenantId);
        }
    }
}
```

- [ ] **Step 5: Install blocks-core**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl blocks-core -s .mvn/slot-settings.xml -DskipTests -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s .mvn/slot-settings.xml -Dtest=BeliefRevisionIntegrationTest`
Expected: PASS — all 3 tests green

If the `fullLifecycle` test fails because the cursor prevents re-processing across multiple `run()` calls, adjust: either add new graduated nodes before each run, or reset the cursor between runs by clearing cursor node property. The stub returns the same JSON each time, but the cursor filtering means only the first run processes evidence. Fix by adding new evidence nodes with incrementing IDs before each subsequent run.

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s .mvn/slot-settings.xml`
Expected: ALL PASS (ignoring pre-existing ManorResourceProfileTest failure)

- [ ] **Step 8: Commit both repos**

blocks repo:
```bash
git -C /path/to/blocks add blocks-core/src/main/java/io/casehub/blocks/agentic/social/belief/
git -C /path/to/blocks commit -m "feat(#67): BeliefRevisionPhase consolidation @Priority(16)

Refs casehubio/examples#67

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

examples repo:
```bash
git add wacky-manor/src/test/java/io/casehub/examples/manor/engine/BeliefRevisionIntegrationTest.java
git commit -m "test(#67): BeliefRevisionPhase integration tests

Refs #67

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Rendering transition (wacky-manor)

### Task 2: CharacterCognition belief rendering from MindMapStore

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:100-111`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: `MindMapStore.nodesIn()`, `MindMapStore.getSupersessionStatus()`, `ManorCognitiveSeeder.subgraphName(String)`, `Belief.of(String, String, int)`, `CognitiveObservationSections.beliefsSection(List<? extends Belief<?>>, Set<String>)`
- Produces: Modified `renderCognitiveSections()` that reads beliefs from MindMapStore instead of static `socialConfig.initialBeliefs()`

- [ ] **Step 1: Write the failing test**

Add to `CharacterCognitionTest`:

```java
@Test
void beliefRenderingFromMindMapStore() {
    var store = new InMemoryMindMapStore();
    var seeder = new ManorCognitiveSeeder(store);
    var allConfigs = ManorSocialConfigLoader.load();
    var agent = "hooded-claw";
    var tenant = "rendering-test";

    seeder.seed(agent, allConfigs.get(agent), tenant);

    var cognition = new CharacterCognition(
        agent, null, null, allConfigs.get(agent), List.of(),
        null, new ManorContextStrategy(), null, null,
        tenant, store, null);

    var sections = cognition.renderCognitiveSections(
        new CharacterState(agent, "library", List.of(), List.of()),
        List.of(), Map.of());

    var beliefSection = sections.stream()
        .filter(s -> "Your Beliefs".equals(s.heading()))
        .findFirst().orElseThrow();

    assertThat(beliefSection.items())
        .anyMatch(item -> item.contains("naive"));
}

@Test
void revisedBeliefShowsRevisedMarker() {
    var store = new InMemoryMindMapStore();
    var seeder = new ManorCognitiveSeeder(store);
    var allConfigs = ManorSocialConfigLoader.load();
    var agent = "hooded-claw";
    var tenant = "revised-test";

    var seedResult = seeder.seed(agent, allConfigs.get(agent), tenant);

    // Create a revised belief and supersede the original
    var revisedNodeId = store.addNode(
        io.casehub.neocortex.mindmap.NodeInput.of(
            "Penelope is more perceptive than she appears", seedResult.subgraphId())
            .withConfidence(io.casehub.neocortex.cognitive.Confidence.inferred(0.6, java.time.Instant.now()))
            .withProvenance("belief-revision")
            .withTraits(java.util.Set.of("Belieflike"))
            .withProperties(java.util.Map.of("subject", "penelope-awareness"))
            .withPrincipalId(io.casehub.platform.api.identity.PrincipalId.agent(agent)),
        tenant);

    // Find the original penelope-awareness belief and supersede it
    var subgraphs = store.listSubgraphs(tenant);
    var beliefSg = subgraphs.stream()
        .filter(sg -> sg.name().equals("beliefs-" + agent))
        .findFirst().orElseThrow();
    // We need to find the original node ID before supersession
    // nodesIn still returns it until we supersede
    var originalId = store.nodesIn(beliefSg.id(), tenant).stream()
        .filter(n -> n.traits().contains("Belieflike"))
        .filter(n -> "penelope-awareness".equals(n.property("subject").orElse(null)))
        .filter(n -> "manor-seed".equals(n.provenance()))
        .map(io.casehub.neocortex.mindmap.MindMapNode::id)
        .findFirst().orElseThrow();

    store.supersede(originalId, revisedNodeId, "Penelope demonstrated perceptiveness", tenant);

    var cognition = new CharacterCognition(
        agent, null, null, allConfigs.get(agent), List.of(),
        null, new ManorContextStrategy(), null, null,
        tenant, store, null);

    var sections = cognition.renderCognitiveSections(
        new CharacterState(agent, "library", List.of(), List.of()),
        List.of(), Map.of());

    var beliefSection = sections.stream()
        .filter(s -> "Your Beliefs".equals(s.heading()))
        .findFirst().orElseThrow();

    // Revised belief should be present with [REVISED] marker
    assertThat(beliefSection.items())
        .anyMatch(item -> item.contains("[REVISED]") && item.contains("penelope-awareness"));
    // Original "naive" text should NOT appear
    assertThat(beliefSection.items())
        .noneMatch(item -> item.contains("naive"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s .mvn/slot-settings.xml -Dtest=CharacterCognitionTest#beliefRenderingFromMindMapStore`
Expected: FAIL — current code reads from static config, doesn't query MindMapStore

- [ ] **Step 3: Replace static belief rendering with MindMapStore query**

Use `ide_replace_member` or Edit to replace the belief rendering block in `CharacterCognition.renderCognitiveSections()` (lines 106-111):

Replace:
```java
if (!socialConfig.initialBeliefs().isEmpty()) {
    var items = socialConfig.initialBeliefs().stream()
                            .map(SocialConfig.InitialBelief::value)
                            .toList();
    sections.add(ObservationSection.items("Your Beliefs", null, items));
}
```

With:
```java
if (mindMapStore != null && tenantId != null) {
    var beliefSubgraphName = ManorCognitiveSeeder.subgraphName(agentId);
    var subgraphs = mindMapStore.listSubgraphs(tenantId);
    var beliefSg = subgraphs.stream()
        .filter(sg -> beliefSubgraphName.equals(sg.name()))
        .findFirst();
    if (beliefSg.isPresent()) {
        var nodes = mindMapStore.nodesIn(beliefSg.get().id(), tenantId);
        var beliefNodes = nodes.stream()
            .filter(n -> n.traits().contains("Belieflike"))
            .toList();
        if (!beliefNodes.isEmpty()) {
            var revisedKeys = new java.util.HashSet<String>();
            var beliefs = new java.util.ArrayList<io.casehub.blocks.agentic.belief.Belief<String>>();
            for (var node : beliefNodes) {
                var key = node.property("subject").orElse(node.name());
                var value = node.name();
                int entrenchment = (int) (node.confidence().value() * 10);
                beliefs.add(io.casehub.blocks.agentic.belief.Belief.of(key, value, entrenchment));
                if ("belief-revision".equals(node.provenance())) {
                    revisedKeys.add(key);
                }
            }
            sections.add(io.casehub.blocks.summarisation.observation.affordance
                .CognitiveObservationSections.beliefsSection(beliefs, revisedKeys));
        }
    }
} else if (!socialConfig.initialBeliefs().isEmpty()) {
    var items = socialConfig.initialBeliefs().stream()
                            .map(SocialConfig.InitialBelief::value)
                            .toList();
    sections.add(ObservationSection.items("Your Beliefs", null, items));
}
```

The fallback to static config ensures rendering works when `mindMapStore` is null (tests that don't wire MindMapStore).

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s .mvn/slot-settings.xml -Dtest=CharacterCognitionTest`
Expected: ALL PASS

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s .mvn/slot-settings.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java
git commit -m "feat(#67): CharacterCognition renders beliefs from MindMapStore

Transitions from Phase A (static config) to Phase B (MindMapStore-sourced).
Revised beliefs marked with [REVISED] via CognitiveObservationSections.beliefsSection().

Refs #67

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-17-belief-revision-design.md] — design spec this plan implements
- [DriveAdaptationPhase.java] (blocks-core) — consolidation phase + cursor pattern
- [UserModelOrchestrator.invokeLlmSynthesis()] (blocks-core:231-282) — AgentProvider invocation pattern
- [UserModelOrchestrator.extractJsonField()] (blocks-core:308-339) — JSON parsing pattern
- [Belief.java] (blocks-core) — `Belief.of(key, value, entrenchment)` factory
- [CognitiveObservationSections.beliefsSection()] (blocks-core:103) — belief rendering with revised marking
- [ManorCognitiveSeeder.seed()] (wacky-manor:85-128) — belief seeding with Belieflike trait
- [CharacterCognition.renderCognitiveSections()] (wacky-manor:100-129) — current static belief rendering
- [InMemoryMindMapStore.nodesIn()] — filters superseded nodes (line 183)
- [MindMapStore.supersede()] — supersession API
- [Confidence] — `inferred()`, `withValue()`, `withDecayReference()`
- GitHub #67, #52
