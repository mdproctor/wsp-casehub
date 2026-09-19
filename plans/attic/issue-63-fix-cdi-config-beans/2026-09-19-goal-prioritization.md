# Goal Prioritization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #69 — Goal prioritization from unmet needs
**Issue group:** #63, #66, #67, #68, #69

**Goal:** Wire #68's numeric satisfaction levels into ManorGoalRevisionStrategy so the LLM reprioritizes goals based on unmet needs. Add REPRIORITIZE action to the goal revision pipeline.

**Architecture:** ManorGoalRevisionStrategy reads need-satisfaction nodes from MindMapStore, appends numeric satisfaction levels to the LLM prompt (LLM-as-reasoner), and parses REPRIORITIZE actions from the response. ManorGoalEvaluator applies priority changes.

**Tech Stack:** Java 26, Quarkus 3.32+, JUnit 5, MindMapStore (neocortex)

## Global Constraints

- Engine-api changes (GoalRevisionAction, RevisedGoal) require the engine repo — not in this slot. Check if it can be cloned to this slot, or defer to a separate session.
- Build wacky-manor: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml`
- All commits with `Refs #69`
- Satisfaction levels presented as raw numbers (not qualitative prose) — per GE-20260919-3f610d

---

## Batch 1: Platform — GoalRevisionAction.REPRIORITIZE + RevisedGoal.newPriority

### Task 1: Extend GoalRevisionAction and RevisedGoal in engine-api

**Files:**
- Modify: `engine-api/src/main/java/io/casehub/api/spi/routing/GoalRevisionAction.java`
- Modify: `engine-api/src/main/java/io/casehub/api/spi/routing/GoalRevisionProposal.java` (RevisedGoal record)
- Test: `engine-api/src/test/java/io/casehub/api/spi/routing/GoalRevisionActionTest.java` (new)

**Interfaces:**
- Produces: `GoalRevisionAction.REPRIORITIZE` enum value
- Produces: `GoalRevisionProposal.RevisedGoal` with optional `GoalPriority newPriority` field

**Pre-requisite:** The engine repo must be available in this slot. If not:
```bash
git clone git@github.com:casehubio/engine.git /Users/mdproctor/claude/casehub/slots/196/engine
```
Check the current branch structure and create a feature branch if needed.

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.api.spi.routing;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class GoalRevisionActionTest {

    @Test
    void reprioritizeActionExists() {
        assertNotNull(GoalRevisionAction.REPRIORITIZE);
        assertEquals(GoalRevisionAction.REPRIORITIZE,
                     GoalRevisionAction.valueOf("REPRIORITIZE"));
    }

    @Test
    void revisedGoalAcceptsNewPriority() {
        var revised = new GoalRevisionProposal.RevisedGoal(
            "test-goal", GoalRevisionAction.REPRIORITIZE,
            null, "Needs neglected",
            io.casehub.eidos.api.GoalPriority.PRIMARY);
        assertEquals(io.casehub.eidos.api.GoalPriority.PRIMARY, revised.newPriority());
        assertNull(revised.revisedDescription());
    }

    @Test
    void revisedGoalDefaultsNewPriorityToNull() {
        var revised = new GoalRevisionProposal.RevisedGoal(
            "test-goal", GoalRevisionAction.REVISE,
            "Updated desc", "Refinement", null);
        assertNull(revised.newPriority());
    }
}
```

- [ ] **Step 2: Add REPRIORITIZE to GoalRevisionAction enum**

```java
public enum GoalRevisionAction {
    REVISE, ABANDON, COMPLETE, REPRIORITIZE
}
```

- [ ] **Step 3: Add newPriority to RevisedGoal record**

Extend the existing `RevisedGoal` record in `GoalRevisionProposal`:

```java
public record RevisedGoal(
    String goalName,
    GoalRevisionAction action,
    String revisedDescription,
    String revisionReason,
    GoalPriority newPriority  // nullable — used with REPRIORITIZE
) {}
```

Add import: `import io.casehub.eidos.api.GoalPriority;`

**Backward compatibility:** Existing callers construct RevisedGoal with 4 args. Either add a 4-arg constructor that defaults newPriority to null, or update all call sites. Since this is pre-release with ~5 call sites, updating call sites is simpler.

- [ ] **Step 4: Update all RevisedGoal construction sites**

Find all `new GoalRevisionProposal.RevisedGoal(` and add `null` as the 5th argument where REPRIORITIZE is not intended.

Key locations:
- `ManorGoalRevisionStrategy.parseResponse()` — already in wacky-manor
- Any test files constructing RevisedGoal

- [ ] **Step 5: Run tests and install**

```bash
# Run engine-api tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-api -f /Users/mdproctor/claude/casehub/slots/196/engine/pom.xml

# Install updated jar
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl engine-api -f /Users/mdproctor/claude/casehub/slots/196/engine/pom.xml -DskipTests -q
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/engine add engine-api/
git -C /Users/mdproctor/claude/casehub/slots/196/engine commit -m "feat(#69): GoalRevisionAction.REPRIORITIZE + RevisedGoal.newPriority

Refs casehubio/examples#69

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Wiring — ManorGoalRevisionStrategy + ManorGoalEvaluator

### Task 2: ManorGoalRevisionStrategy needs-aware prompt

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalRevisionStrategy.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorGoalRevisionStrategyTest.java` (extend)

**Interfaces:**
- Consumes: `MindMapStore` (reads `cognitiveKind: "need-satisfaction"` nodes)
- Consumes: `GoalRevisionContext` (provides `agentId()`, `tenancyId()`)
- Produces: Enhanced LLM prompt with satisfaction levels; REPRIORITIZE parsing in response

- [ ] **Step 1: Write the failing test — satisfaction in prompt**

Add to `ManorGoalRevisionStrategyTest.java`:

```java
@Test
void revise_prompt_includes_satisfaction_levels() {
    String[] capturedPrompt = {null};
    AgentProvider capturingProvider = new AgentProvider() {
        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            capturedPrompt[0] = config.userPrompt();
            return Multi.createFrom().item(new AgentEvent.TextDelta(
                "{\"revisions\": [], \"rationale\": \"\"}"));
        }
        @Override
        public AgentSession openSession(AgentSessionInit init) {
            throw new UnsupportedOperationException();
        }
    };

    var store = new InMemoryMindMapStore();
    var subgraphId = store.createSubgraph(
        new SubgraphInput("beliefs-hc", "cognitive", null), "wacky-manor");
    store.addNode(NodeInput.of("need-safety", subgraphId)
        .withProvenance("need-satisfaction")
        .withProperties(Map.of(
            "cognitiveKind", "need-satisfaction",
            "agent-id", "hc",
            "tier", "SAFETY",
            "satisfaction", "0.62")),
        "wacky-manor");
    store.addNode(NodeInput.of("need-tasks", subgraphId)
        .withProvenance("need-satisfaction")
        .withProperties(Map.of(
            "cognitiveKind", "need-satisfaction",
            "agent-id", "hc",
            "tier", "TASKS",
            "satisfaction", "0.28")),
        "wacky-manor");

    var strategy = new ManorGoalRevisionStrategy(capturingProvider, store);
    strategy.revise(buildContext());

    assertThat(capturedPrompt[0]).contains("SAFETY=0.62");
    assertThat(capturedPrompt[0]).contains("TASKS=0.28");
    assertThat(capturedPrompt[0]).contains("Need satisfaction");
}
```

Additional imports needed:
```java
import io.casehub.neocortex.mindmap.NodeInput;
import io.casehub.neocortex.mindmap.SubgraphInput;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import java.util.Map;
```

- [ ] **Step 2: Write the failing test — REPRIORITIZE parsing**

```java
@Test
void revise_parses_reprioritize_action() {
    String response = """
        {"revisions": [{"goalName": "protect-tea", "action": "REPRIORITIZE",
          "revisedDescription": null, "newPriority": "PRIMARY",
          "revisionReason": "Tasks neglected"}],
         "rationale": "Needs-based reprioritization"}
        """;
    var strategy = new ManorGoalRevisionStrategy(mockProvider(response), null);
    var context = buildContext();
    GoalRevisionProposal proposal = strategy.revise(context);
    assertThat(proposal.revisions()).hasSize(1);
    assertThat(proposal.revisions().get(0).action()).isEqualTo(GoalRevisionAction.REPRIORITIZE);
    assertThat(proposal.revisions().get(0).newPriority()).isEqualTo(GoalPriority.PRIMARY);
}
```

- [ ] **Step 3: Write the failing test — graceful degradation (no satisfaction nodes)**

```java
@Test
void revise_prompt_omits_satisfaction_when_no_nodes() {
    String[] capturedPrompt = {null};
    AgentProvider capturingProvider = new AgentProvider() {
        @Override
        public Multi<AgentEvent> invoke(AgentSessionConfig config) {
            capturedPrompt[0] = config.userPrompt();
            return Multi.createFrom().item(new AgentEvent.TextDelta(
                "{\"revisions\": [], \"rationale\": \"\"}"));
        }
        @Override
        public AgentSession openSession(AgentSessionInit init) {
            throw new UnsupportedOperationException();
        }
    };

    var strategy = new ManorGoalRevisionStrategy(capturingProvider, null);
    strategy.revise(buildContext());

    assertThat(capturedPrompt[0]).doesNotContain("Need satisfaction");
}
```

- [ ] **Step 4: Run tests to verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ManorGoalRevisionStrategyTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: compilation failure — constructor doesn't accept MindMapStore.

- [ ] **Step 5: Modify ManorGoalRevisionStrategy constructor**

Add `MindMapStore mindMapStore` parameter (nullable for backward compat):

```java
private final AgentProvider agentProvider;
private final MindMapStore mindMapStore;

@Inject
public ManorGoalRevisionStrategy(AgentProvider agentProvider,
                                  @Nullable MindMapStore mindMapStore) {
    this.agentProvider = agentProvider;
    this.mindMapStore = mindMapStore;
}
```

Add imports:
```java
import io.casehub.neocortex.mindmap.MindMapStore;
import org.jspecify.annotations.Nullable;
```

- [ ] **Step 6: Add satisfaction reading to buildPrompt()**

Add a method to read satisfaction and append to prompt:

```java
private String readSatisfaction(String agentId, String tenantId) {
    if (mindMapStore == null) return "";
    var subgraphs = mindMapStore.listSubgraphs(tenantId);
    var sb = new StringBuilder();
    for (var sg : subgraphs) {
        if (!"cognitive".equals(sg.type())) continue;
        for (var node : mindMapStore.nodesIn(sg.id(), tenantId)) {
            if (!"need-satisfaction".equals(node.properties().get("cognitiveKind"))) continue;
            if (!agentId.equals(node.properties().get("agent-id"))) continue;
            var tier = node.properties().get("tier");
            var sat = node.properties().get("satisfaction");
            if (tier != null && sat != null) {
                if (sb.isEmpty()) sb.append("\nNeed satisfaction levels:\n  ");
                else sb.append(", ");
                sb.append(tier).append("=").append(sat);
            }
        }
    }
    return sb.toString();
}
```

Call from `buildPrompt()` after the goals section:

```java
String satisfaction = readSatisfaction(context.agentId(), context.tenancyId());
if (!satisfaction.isEmpty()) {
    sb.append(satisfaction);
    sb.append("\n\nConsider these satisfaction levels when evaluating goals. ");
    sb.append("Goals addressing neglected needs (low satisfaction) should be elevated. ");
    sb.append("Goals addressing satisfied needs may be deprioritized.\n");
}
```

- [ ] **Step 7: Update SYSTEM_PROMPT**

Add REPRIORITIZE to the system prompt instruction list:

```java
private static final String SYSTEM_PROMPT = """
    You are a goal effectiveness analyst for an autonomous agent. Given the \
    agent's goals, their performance metrics, and their current need \
    satisfaction levels, evaluate each goal and recommend an action:
    - REVISE: refine the goal description to better capture what the agent \
      should accomplish (provide revisedDescription)
    - ABANDON: drop the goal — it is unachievable or no longer relevant
    - COMPLETE: the goal has been achieved
    - REPRIORITIZE: change goal priority to address unmet needs \
      (provide newPriority: "PRIMARY" or "SECONDARY")
    Only act on goals with clear signals. If a goal is fine as-is, omit it \
    from revisions.
    Return ONLY a JSON object: {"revisions": [{"goalName": "...", \
    "action": "REVISE"|"ABANDON"|"COMPLETE"|"REPRIORITIZE", \
    "revisedDescription": "..."|null, "newPriority": "PRIMARY"|"SECONDARY"|null, \
    "revisionReason": "..."}], \
    "rationale": "..."}""";
```

- [ ] **Step 8: Update parseResponse() for REPRIORITIZE**

In `parseResponse()`, add newPriority parsing:

```java
GoalPriority priority = null;
if (node.has("newPriority") && !node.get("newPriority").isNull()) {
    try {
        priority = GoalPriority.valueOf(node.get("newPriority").asText());
    } catch (IllegalArgumentException ignored) {}
}
revisions.add(new GoalRevisionProposal.RevisedGoal(
    goalName, action, desc, reason, priority));
```

- [ ] **Step 9: Update existing constructor calls**

Update the existing `mockProvider` test helper and any other places that construct `ManorGoalRevisionStrategy` with the old 1-arg constructor. Add `null` for `mindMapStore` where satisfaction is not relevant:

```java
var strategy = new ManorGoalRevisionStrategy(mockProvider(response), null);
```

- [ ] **Step 10: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -Dtest=ManorGoalRevisionStrategyTest -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml -Dsurefire.failIfNoSpecifiedTests=false
```
Expected: PASS (all tests including 3 new ones).

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#69): ManorGoalRevisionStrategy — needs-aware prompt + REPRIORITIZE

Reads need-satisfaction levels from MindMapStore, includes as numeric
values in LLM prompt. Parses REPRIORITIZE action with GoalPriority.
Gracefully degrades when no satisfaction nodes exist.

Refs #69

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: ManorGoalEvaluator REPRIORITIZE handling

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorGoalEvaluator.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorGoalEvaluatorTest.java` (extend or create)

**Interfaces:**
- Consumes: `GoalRevisionProposal` with REPRIORITIZE actions
- Produces: Updated `AgentGoal` with new priority via `goal.toBuilder().priority(newPriority).build()`

- [ ] **Step 1: Write the failing test**

```java
@Test
void evaluate_handles_reprioritize_action() {
    // Set up a mock revision strategy that returns REPRIORITIZE
    String response = """
        {"revisions": [{"goalName": "protect-tea", "action": "REPRIORITIZE",
          "revisedDescription": null, "newPriority": "PRIMARY",
          "revisionReason": "Tasks neglected"}],
         "rationale": "Needs-based"}
        """;
    // ... set up ManorGoalEvaluator with mock strategy returning above
    // ... call evaluate()
    // ... verify the goal's priority changed from SECONDARY to PRIMARY
}
```

(Exact test setup depends on how ManorGoalEvaluator is constructed in existing tests — check ManorGoalEvaluatorTest if it exists, or create one following the existing test patterns.)

- [ ] **Step 2: Add REPRIORITIZE case to ManorGoalEvaluator.evaluate()**

After the `COMPLETE` case in the switch statement (line ~137), add:

```java
case REPRIORITIZE -> {
    if (revision.newPriority() != null) {
        for (int i = 0; i < finalGoals.size(); i++) {
            if (finalGoals.get(i).name().equals(revision.goalName())) {
                finalGoals.set(i, finalGoals.get(i).toBuilder()
                    .priority(revision.newPriority()).build());
                changed = true;
                break;
            }
        }
    }
}
```

- [ ] **Step 3: Run tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s /Users/mdproctor/claude/casehub/slots/196/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/examples/pom.xml
```
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#69): ManorGoalEvaluator — REPRIORITIZE action handling

Updates goal priority when GoalRevisionStrategy returns REPRIORITIZE.
Uses goal.toBuilder().priority(newPriority).build().

Refs #69

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-19-goal-prioritization-design.md] — design spec this plan implements
- [ManorGoalRevisionStrategy.java] — wacky-manor, enhanced in Task 2
- [ManorGoalEvaluator.java] — wacky-manor, REPRIORITIZE case in Task 3
- [ManorGoalRevisionStrategyTest.java] — wacky-manor, test pattern reference
- [GoalRevisionAction] — engine-api jar, extended in Task 1
- [GoalRevisionProposal.RevisedGoal] — engine-api jar, newPriority added in Task 1
- [GoalPriority] — eidos-api jar (PRIMARY, SECONDARY)
- [NeedsPyramidPromptSection] — blocks-core (#68), renders qualitative prose
- GE-20260919-3f610d — numeric for reasoning, qualitative for embodiment
- Decisions D43–D45
- GitHub #69 — Goal prioritization from unmet needs
- GitHub #68 — Needs pyramid (upstream dependency)
