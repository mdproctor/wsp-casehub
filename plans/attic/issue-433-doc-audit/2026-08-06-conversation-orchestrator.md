# ConversationOrchestrator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #91 — ConversationOrchestrator — reactive multi-agent conversation loop
**Issue group:** #91

**Goal:** Compose existing blocks primitives into a reusable
`ConversationOrchestrator` that drives autonomous multi-agent conversations
with pluggable turn policies and termination conditions.

**Architecture:** New `conversation.orchestration` sub-package. Three new SPIs
(`TurnPolicy`, `PromptAssembler`, `ResponseMessageBuilder`), four turn policy
implementations, four termination condition implementations, and one
composition root (`ConversationOrchestrator`) with an iterative queue-based
loop. Reuses `TerminationCondition<T>`, `AgentInvoker<T>`, `AgentRef`,
`PartitionedObservationService`, and `ConversationProjection` without changes.

**Tech Stack:** Java 21, JUnit 5, Mockito, AssertJ, Mutiny (Uni), jspecify

## Global Constraints

- Package: `io.casehub.blocks.conversation.orchestration`
- Test package: `io.casehub.blocks.conversation.orchestration`
- No new external dependencies
- No CDI/Quarkus in tests — plain JUnit 5 with Mockito
- `MessageView` is mocked in tests (interface from qhorus-api)
- All `TerminationCondition` implementations return `Uni` (existing SPI contract)
- Pre-release stage — no backward compatibility constraints

---

### Task 1: Core Records and SPIs

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/TurnContext.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/AgentParticipant.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/ConversationOutcome.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/TurnPolicy.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/PromptAssembler.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/ResponseMessageBuilder.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/AgentParticipantTest.java`

**Interfaces:**
- Consumes: `AgentRef` (from `io.casehub.blocks.agentic`), `AgentResult` (from `io.casehub.blocks.agentic`), `TerminationDecision` (from `io.casehub.blocks.agentic.termination`), `ConversationState` (from `io.casehub.blocks.conversation`), `PartitionedDrain<K>` (from `io.casehub.blocks.summarisation.observation`), `ObservationResult` (from `io.casehub.blocks.summarisation.observation`), `MessageView` (from `io.casehub.qhorus.api.message`)
- Produces: `TurnContext(String senderId, @Nullable String targetId, String entryType, Map<String, String> metadata)`, `AgentParticipant(AgentRef agentRef, String role, String systemPrompt)` with `agentId()`, `ConversationOutcome(ConversationState finalState, TerminationDecision terminationDecision, List<AgentResult> agentResults, int dispatchCount, Duration elapsed)`, `TurnPolicy.nextResponders(ConversationState, TurnContext, List<AgentParticipant>) → List<AgentParticipant>`, `PromptAssembler.assemble(AgentParticipant, PartitionedDrain<String>, ConversationState) → String`, `ResponseMessageBuilder.build(AgentParticipant, AgentResult, ConversationState) → MessageView`

- [ ] **Step 1: Write AgentParticipant test**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AgentParticipantTest {

    @Test
    void agentId_delegatesToAgentRefName() {
        var ref = AgentRef.external("reviewer", ignored -> null);
        var participant = new AgentParticipant(ref, "REV", "You are a reviewer.");
        assertThat(participant.agentId()).isEqualTo("reviewer");
    }

    @Test
    void role_returnsConstructorValue() {
        var ref = AgentRef.external("impl", ignored -> null);
        var participant = new AgentParticipant(ref, "IMP", "You implement.");
        assertThat(participant.role()).isEqualTo("IMP");
    }

    @Test
    void systemPrompt_returnsConstructorValue() {
        var ref = AgentRef.external("impl", ignored -> null);
        var participant = new AgentParticipant(ref, "IMP", "You implement code changes.");
        assertThat(participant.systemPrompt()).isEqualTo("You implement code changes.");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl . -Dtest=AgentParticipantTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `AgentParticipant` not found

- [ ] **Step 3: Create all records and interfaces**

`TurnContext.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import org.jspecify.annotations.Nullable;
import java.util.Map;

public record TurnContext(
        String senderId,
        @Nullable String targetId,
        String entryType,
        Map<String, String> metadata
) {
    public TurnContext { metadata = Map.copyOf(metadata); }
}
```

`AgentParticipant.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;

public record AgentParticipant(
        AgentRef agentRef,
        String role,
        String systemPrompt
) {
    public String agentId() { return agentRef.name(); }
}
```

`ConversationOutcome.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentResult;
import io.casehub.blocks.agentic.termination.TerminationDecision;
import io.casehub.blocks.conversation.ConversationState;

import java.time.Duration;
import java.util.List;

public record ConversationOutcome(
        ConversationState finalState,
        TerminationDecision terminationDecision,
        List<AgentResult> agentResults,
        int dispatchCount,
        Duration elapsed
) {
    public ConversationOutcome { agentResults = List.copyOf(agentResults); }
}
```

`TurnPolicy.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.conversation.ConversationState;
import java.util.List;

public interface TurnPolicy {
    List<AgentParticipant> nextResponders(
            ConversationState state,
            TurnContext context,
            List<AgentParticipant> participants);
}
```

`PromptAssembler.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.summarisation.observation.PartitionedDrain;

@FunctionalInterface
public interface PromptAssembler {
    String assemble(AgentParticipant agent,
                    PartitionedDrain<String> drain,
                    ConversationState state);
}
```

`ResponseMessageBuilder.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentResult;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.qhorus.api.message.MessageView;

@FunctionalInterface
public interface ResponseMessageBuilder {
    MessageView build(AgentParticipant agent,
                      AgentResult result,
                      ConversationState currentState);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl . -Dtest=AgentParticipantTest`
Expected: PASS (3 tests)

- [ ] **Step 5: Commit**

```
feat(#91): add core records and SPIs for conversation orchestration

TurnContext, AgentParticipant, ConversationOutcome records.
TurnPolicy, PromptAssembler, ResponseMessageBuilder interfaces.

Refs casehubio/blocks#91
```

---

### Task 2: Turn Policy Implementations

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/RoundRobinTurnPolicy.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/AddressedTurnPolicy.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/PointAddressedTurnPolicy.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/FreeTurnPolicy.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/RoundRobinTurnPolicyTest.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/AddressedTurnPolicyTest.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/PointAddressedTurnPolicyTest.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/FreeTurnPolicyTest.java`

**Interfaces:**
- Consumes: `TurnPolicy`, `TurnContext`, `AgentParticipant`, `ConversationState`, `ConversationPoint`, `ThreadEntry`
- Produces: `RoundRobinTurnPolicy implements TurnPolicy`, `AddressedTurnPolicy implements TurnPolicy`, `PointAddressedTurnPolicy implements TurnPolicy`, `FreeTurnPolicy implements TurnPolicy`

- [ ] **Step 1: Write RoundRobinTurnPolicy test**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;
import io.casehub.blocks.conversation.ConversationFold;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.conversation.PointClassification;
import io.casehub.qhorus.api.model.MessageType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RoundRobinTurnPolicyTest {

    private final AgentParticipant alice = new AgentParticipant(
            AgentRef.external("alice", i -> null), "REV", "");
    private final AgentParticipant bob = new AgentParticipant(
            AgentRef.external("bob", i -> null), "IMP", "");
    private final List<AgentParticipant> participants = List.of(alice, bob);
    private final TurnPolicy policy = new RoundRobinTurnPolicy();

    @Test
    void emptyState_returnsFirstNonSender() {
        var state = emptyState();
        var ctx = new TurnContext("alice", null, "RAISE", Map.of());
        var result = policy.nextResponders(state, ctx, participants);
        assertThat(result).containsExactly(bob);
    }

    @Test
    void afterAliceSpeaks_bobResponds() {
        var state = stateWithLastSender("alice");
        var ctx = new TurnContext("alice", null, "RAISE", Map.of());
        var result = policy.nextResponders(state, ctx, participants);
        assertThat(result).containsExactly(bob);
    }

    @Test
    void afterBobSpeaks_aliceResponds() {
        var state = stateWithLastSender("bob");
        var ctx = new TurnContext("bob", null, "COUNTER", Map.of());
        var result = policy.nextResponders(state, ctx, participants);
        assertThat(result).containsExactly(alice);
    }

    @Test
    void senderNotInParticipants_returnsFirstParticipant() {
        var state = emptyState();
        var ctx = new TurnContext("human", null, "RAISE", Map.of());
        var result = policy.nextResponders(state, ctx, participants);
        assertThat(result).containsExactly(alice);
    }

    private ConversationState emptyState() {
        return new ConversationState(Map.of(), List.of(), List.of(), Map.of());
    }

    private ConversationState stateWithLastSender(String sender) {
        var state = emptyState();
        return ConversationFold.createPoint(state, "p1", "topic",
                1L, null, sender, Instant.now(),
                PointClassification.of("ISSUE"), "REV", 1, "RAISE", "content");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl . -Dtest=RoundRobinTurnPolicyTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure — `RoundRobinTurnPolicy` not found

- [ ] **Step 3: Implement RoundRobinTurnPolicy**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.conversation.ConversationState;

import java.util.List;

public class RoundRobinTurnPolicy implements TurnPolicy {

    @Override
    public List<AgentParticipant> nextResponders(
            ConversationState state,
            TurnContext context,
            List<AgentParticipant> participants) {
        if (participants.isEmpty()) return List.of();

        int senderIndex = -1;
        for (int i = 0; i < participants.size(); i++) {
            if (participants.get(i).agentId().equals(context.senderId())) {
                senderIndex = i;
                break;
            }
        }

        if (senderIndex == -1) {
            return List.of(participants.getFirst());
        }

        int nextIndex = (senderIndex + 1) % participants.size();
        return List.of(participants.get(nextIndex));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl . -Dtest=RoundRobinTurnPolicyTest`
Expected: PASS (4 tests)

- [ ] **Step 5: Write AddressedTurnPolicy test and implementation**

Test — `AddressedTurnPolicyTest.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;
import io.casehub.blocks.conversation.ConversationState;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class AddressedTurnPolicyTest {

    private final AgentParticipant rev = new AgentParticipant(
            AgentRef.external("reviewer", i -> null), "REV", "");
    private final AgentParticipant imp = new AgentParticipant(
            AgentRef.external("implementor", i -> null), "IMP", "");
    private final List<AgentParticipant> participants = List.of(rev, imp);
    private final TurnPolicy policy = new AddressedTurnPolicy();
    private final ConversationState empty = new ConversationState(
            Map.of(), List.of(), List.of(), Map.of());

    @Test
    void targetMatchesRole_returnsMatchingParticipant() {
        var ctx = new TurnContext("human", "IMP", "RAISE", Map.of());
        assertThat(policy.nextResponders(empty, ctx, participants))
                .containsExactly(imp);
    }

    @Test
    void nullTarget_returnsEmptyList() {
        var ctx = new TurnContext("human", null, "RAISE", Map.of());
        assertThat(policy.nextResponders(empty, ctx, participants)).isEmpty();
    }

    @Test
    void unknownTarget_returnsEmptyList() {
        var ctx = new TurnContext("human", "UNKNOWN", "RAISE", Map.of());
        assertThat(policy.nextResponders(empty, ctx, participants)).isEmpty();
    }
}
```

Implementation — `AddressedTurnPolicy.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.conversation.ConversationState;

import java.util.List;

public class AddressedTurnPolicy implements TurnPolicy {

    @Override
    public List<AgentParticipant> nextResponders(
            ConversationState state,
            TurnContext context,
            List<AgentParticipant> participants) {
        if (context.targetId() == null) return List.of();
        return participants.stream()
                .filter(p -> p.role().equals(context.targetId()))
                .toList();
    }
}
```

- [ ] **Step 6: Run AddressedTurnPolicy tests**

Run: `mvn --batch-mode test -pl . -Dtest=AddressedTurnPolicyTest`
Expected: PASS (3 tests)

- [ ] **Step 7: Write FreeTurnPolicy test and implementation**

Test — `FreeTurnPolicyTest.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;
import io.casehub.blocks.conversation.ConversationState;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class FreeTurnPolicyTest {

    private final AgentParticipant alice = new AgentParticipant(
            AgentRef.external("alice", i -> null), "REV", "");
    private final AgentParticipant bob = new AgentParticipant(
            AgentRef.external("bob", i -> null), "IMP", "");
    private final AgentParticipant carol = new AgentParticipant(
            AgentRef.external("carol", i -> null), "SUP", "");
    private final TurnPolicy policy = new FreeTurnPolicy();
    private final ConversationState empty = new ConversationState(
            Map.of(), List.of(), List.of(), Map.of());

    @Test
    void returnsAllExceptSender() {
        var ctx = new TurnContext("alice", null, "RAISE", Map.of());
        assertThat(policy.nextResponders(empty, ctx, List.of(alice, bob, carol)))
                .containsExactly(bob, carol);
    }

    @Test
    void singleParticipant_senderExcluded_returnsEmpty() {
        var ctx = new TurnContext("alice", null, "RAISE", Map.of());
        assertThat(policy.nextResponders(empty, ctx, List.of(alice))).isEmpty();
    }

    @Test
    void unknownSender_returnsAll() {
        var ctx = new TurnContext("human", null, "RAISE", Map.of());
        assertThat(policy.nextResponders(empty, ctx, List.of(alice, bob)))
                .containsExactly(alice, bob);
    }
}
```

Implementation — `FreeTurnPolicy.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.conversation.ConversationState;

import java.util.List;

public class FreeTurnPolicy implements TurnPolicy {

    @Override
    public List<AgentParticipant> nextResponders(
            ConversationState state,
            TurnContext context,
            List<AgentParticipant> participants) {
        return participants.stream()
                .filter(p -> !p.agentId().equals(context.senderId()))
                .toList();
    }
}
```

- [ ] **Step 8: Run FreeTurnPolicy tests**

Run: `mvn --batch-mode test -pl . -Dtest=FreeTurnPolicyTest`
Expected: PASS (3 tests)

- [ ] **Step 9: Write PointAddressedTurnPolicy test and implementation**

Test — `PointAddressedTurnPolicyTest.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;
import io.casehub.blocks.conversation.ConversationFold;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.conversation.PointClassification;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PointAddressedTurnPolicyTest {

    private final AgentParticipant rev = new AgentParticipant(
            AgentRef.external("reviewer", i -> null), "REV", "");
    private final AgentParticipant imp = new AgentParticipant(
            AgentRef.external("implementor", i -> null), "IMP", "");
    private final List<AgentParticipant> participants = List.of(rev, imp);
    private final TurnPolicy policy = new PointAddressedTurnPolicy();

    @Test
    void openPoint_raisedByRev_impShouldRespond() {
        var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
        state = ConversationFold.createPoint(state, "p1", "design",
                1L, null, "reviewer", Instant.now(),
                PointClassification.of("ISSUE"), "REV", 1, "RAISE", "This needs fixing");
        var ctx = new TurnContext("reviewer", null, "RAISE", Map.of());
        var result = policy.nextResponders(state, ctx, participants);
        assertThat(result).containsExactly(imp);
    }

    @Test
    void pointAlreadyResponded_notReturned() {
        var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
        state = ConversationFold.createPoint(state, "p1", "design",
                1L, null, "reviewer", Instant.now(),
                PointClassification.of("ISSUE"), "REV", 1, "RAISE", "Fix this");
        state = ConversationFold.respondToPoint(state, "p1",
                2L, null, "implementor", Instant.now(),
                "IMP", 1, "AGREE", "Will do", "AGREED");
        var ctx = new TurnContext("implementor", null, "AGREE", Map.of());
        var result = policy.nextResponders(state, ctx, participants);
        assertThat(result).isEmpty();
    }

    @Test
    void emptyState_returnsEmpty() {
        var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
        var ctx = new TurnContext("reviewer", null, "RAISE", Map.of());
        assertThat(policy.nextResponders(state, ctx, participants)).isEmpty();
    }
}
```

Implementation — `PointAddressedTurnPolicy.java`:
```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.conversation.ConversationState;

import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;

public class PointAddressedTurnPolicy implements TurnPolicy {

    private static final Set<String> OPEN_STATUSES = Set.of("OPEN", "ACTIVE");

    @Override
    public List<AgentParticipant> nextResponders(
            ConversationState state,
            TurnContext context,
            List<AgentParticipant> participants) {
        var needed = new LinkedHashSet<AgentParticipant>();
        for (var point : state.points().values()) {
            if (!OPEN_STATUSES.contains(point.status())) continue;

            Set<String> respondedRoles = new java.util.HashSet<>();
            for (var entry : point.thread()) {
                respondedRoles.add(entry.role());
            }

            for (var participant : participants) {
                if (participant.agentId().equals(context.senderId())) continue;
                if (respondedRoles.contains(participant.role())) continue;
                needed.add(participant);
            }
        }
        return List.copyOf(needed);
    }
}
```

- [ ] **Step 10: Run PointAddressedTurnPolicy tests**

Run: `mvn --batch-mode test -pl . -Dtest=PointAddressedTurnPolicyTest`
Expected: PASS (3 tests)

- [ ] **Step 11: Run all turn policy tests**

Run: `mvn --batch-mode test -pl . -Dtest="RoundRobinTurnPolicyTest,AddressedTurnPolicyTest,FreeTurnPolicyTest,PointAddressedTurnPolicyTest"`
Expected: PASS (13 tests)

- [ ] **Step 12: Commit**

```
feat(#91): add turn policy SPI and four standard implementations

RoundRobinTurnPolicy, AddressedTurnPolicy, FreeTurnPolicy,
PointAddressedTurnPolicy — all stateless, pure functions of
ConversationState.

Refs casehubio/blocks#91
```

---

### Task 3: Termination Condition Implementations

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/AllAgreedTermination.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/SupervisorTermination.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/ContestedEscalation.java`
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/CompositeTermination.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/AllAgreedTerminationTest.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/SupervisorTerminationTest.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/ContestedEscalationTest.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/CompositeTerminationTest.java`

**Interfaces:**
- Consumes: `TerminationCondition<ConversationState>` (from `agentic.termination`), `TerminationContext<ConversationState>`, `TerminationDecision`, `ConversationState`, `ConversationPoint`
- Produces: `AllAgreedTermination implements TerminationCondition<ConversationState>`, `SupervisorTermination implements TerminationCondition<ConversationState>`, `ContestedEscalation implements TerminationCondition<ConversationState>`, `CompositeTermination implements TerminationCondition<ConversationState>`

- [ ] **Step 1: Write AllAgreedTermination test**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.termination.TerminationContext;
import io.casehub.blocks.agentic.termination.TerminationDecision;
import io.casehub.blocks.conversation.ConversationFold;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.conversation.PointClassification;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class AllAgreedTerminationTest {

    private final AllAgreedTermination termination =
            new AllAgreedTermination(Set.of("AGREED", "VERIFIED"));

    @Test
    void noPoints_continues() {
        var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
        var decision = evaluate(state);
        assertThat(decision).isInstanceOf(TerminationDecision.Continue.class);
    }

    @Test
    void allPointsResolved_completes() {
        var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
        state = ConversationFold.createPoint(state, "p1", "topic",
                1L, null, "alice", Instant.now(),
                PointClassification.of("ISSUE"), "REV", 1, "RAISE", "content");
        state = ConversationFold.respondToPoint(state, "p1",
                2L, null, "bob", Instant.now(),
                "IMP", 1, "AGREE", "ok", "AGREED");
        var decision = evaluate(state);
        assertThat(decision).isInstanceOf(TerminationDecision.Complete.class);
    }

    @Test
    void somePointsUnresolved_continues() {
        var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
        state = ConversationFold.createPoint(state, "p1", "t1",
                1L, null, "alice", Instant.now(),
                PointClassification.of("ISSUE"), "REV", 1, "RAISE", "c1");
        state = ConversationFold.respondToPoint(state, "p1",
                2L, null, "bob", Instant.now(),
                "IMP", 1, "AGREE", "ok", "AGREED");
        state = ConversationFold.createPoint(state, "p2", "t2",
                3L, null, "alice", Instant.now(),
                PointClassification.of("ISSUE"), "REV", 1, "RAISE", "c2");
        var decision = evaluate(state);
        assertThat(decision).isInstanceOf(TerminationDecision.Continue.class);
    }

    private TerminationDecision evaluate(ConversationState state) {
        var ctx = new TerminationContext<>(state, 1, Duration.ZERO, List.of());
        return termination.evaluate(ctx).await().indefinitely();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl . -Dtest=AllAgreedTerminationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Implement AllAgreedTermination**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.termination.TerminationCondition;
import io.casehub.blocks.agentic.termination.TerminationContext;
import io.casehub.blocks.agentic.termination.TerminationDecision;
import io.casehub.blocks.conversation.ConversationState;
import io.smallrye.mutiny.Uni;

import java.util.Set;

public class AllAgreedTermination implements TerminationCondition<ConversationState> {

    private final Set<String> resolvedStatuses;

    public AllAgreedTermination(Set<String> resolvedStatuses) {
        this.resolvedStatuses = Set.copyOf(resolvedStatuses);
    }

    @Override
    public Uni<TerminationDecision> evaluate(TerminationContext<ConversationState> context) {
        var points = context.state().points();
        if (points.isEmpty()) {
            return Uni.createFrom().item(TerminationDecision.Continue.INSTANCE);
        }
        boolean allResolved = points.values().stream()
                .allMatch(p -> resolvedStatuses.contains(p.status()));
        if (allResolved) {
            return Uni.createFrom().item(
                    new TerminationDecision.Complete("All points resolved"));
        }
        return Uni.createFrom().item(TerminationDecision.Continue.INSTANCE);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl . -Dtest=AllAgreedTerminationTest`
Expected: PASS (3 tests)

- [ ] **Step 5: Write and implement SupervisorTermination, ContestedEscalation, CompositeTermination**

Follow the same TDD cycle for each. Key implementation details:

`SupervisorTermination`: Scans `state.points()` for a point where the last thread entry's role matches `supervisorRole` and entry type signals end (e.g., "TERMINATE" or a configurable signal entry type). Returns `Complete` with the supervisor's content.

`ContestedEscalation`: Counts how many thread entries have `DISPUTED` status for each point. If any point has been disputed more than `maxDisputeRounds` times, returns `Escalate`.

`CompositeTermination`: Iterates conditions in order, evaluates each (awaiting the `Uni`). First non-Continue result wins. If all return Continue, returns Continue.

```java
// CompositeTermination.java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.termination.TerminationCondition;
import io.casehub.blocks.agentic.termination.TerminationContext;
import io.casehub.blocks.agentic.termination.TerminationDecision;
import io.casehub.blocks.conversation.ConversationState;
import io.smallrye.mutiny.Uni;

import java.util.List;

public class CompositeTermination implements TerminationCondition<ConversationState> {

    private final List<TerminationCondition<ConversationState>> conditions;

    public CompositeTermination(
            List<TerminationCondition<ConversationState>> conditions) {
        this.conditions = List.copyOf(conditions);
    }

    @Override
    public Uni<TerminationDecision> evaluate(
            TerminationContext<ConversationState> context) {
        Uni<TerminationDecision> chain = Uni.createFrom()
                .item(TerminationDecision.Continue.INSTANCE);
        for (var condition : conditions) {
            chain = chain.flatMap(prev -> {
                if (prev instanceof TerminationDecision.Continue) {
                    return condition.evaluate(context);
                }
                return Uni.createFrom().item(prev);
            });
        }
        return chain;
    }
}
```

- [ ] **Step 6: Run all termination tests**

Run: `mvn --batch-mode test -pl . -Dtest="AllAgreedTerminationTest,SupervisorTerminationTest,ContestedEscalationTest,CompositeTerminationTest"`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#91): add conversation termination conditions

AllAgreedTermination, SupervisorTermination, ContestedEscalation,
CompositeTermination — all implement TerminationCondition<ConversationState>.

Refs casehubio/blocks#91
```

---

### Task 4: ConversationOrchestrator

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/orchestration/ConversationOrchestrator.java`
- Test: `src/test/java/io/casehub/blocks/conversation/orchestration/ConversationOrchestratorTest.java`

**Interfaces:**
- Consumes: All types from Tasks 1-3, plus `ConversationProjection`, `PartitionedObservationService<MessageView, String>`, `AgentInvoker<String>`, `PartitionedDrain<String>`, `TerminationContext<ConversationState>`, `MessageView`
- Produces: `ConversationOrchestrator(projection, observationService, turnPolicy, terminationCondition, agentInvoker, promptAssembler, responseBuilder, responseDispatcher, participants)`, `converse(MessageView) → Uni<ConversationOutcome>`, `terminate()`

- [ ] **Step 1: Write test — two-agent round-robin, max iterations stops**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentRef;
import io.casehub.blocks.agentic.AgentResult;
import io.casehub.blocks.agentic.model.AgentInvoker;
import io.casehub.blocks.agentic.termination.MaxIterationsTermination;
import io.casehub.blocks.agentic.termination.TerminationDecision;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.summarisation.EventLevel;
import io.casehub.blocks.summarisation.observation.ObservationResult;
import io.casehub.blocks.summarisation.observation.PartitionedDrain;
import io.casehub.blocks.summarisation.observation.PartitionedObservationService;
import io.casehub.blocks.summarisation.observation.TieredObservationRenderer;
import io.casehub.blocks.summarisation.observation.VisibilityPolicy;
import io.casehub.qhorus.api.message.MessageView;
import io.smallrye.mutiny.Uni;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.atomic.AtomicInteger;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class ConversationOrchestratorTest {

    @Test
    void twoAgentDebate_roundRobin_maxIterationsStops() {
        var projection = new TestProjection();
        var observationService = createObservationService();
        var dispatched = new ArrayList<MessageView>();
        var callCount = new AtomicInteger();

        AgentInvoker<String> invoker = (agent, prompt) -> {
            int n = callCount.incrementAndGet();
            return Uni.createFrom().item(
                    AgentResult.success(agent, "Response " + n));
        };

        var alice = new AgentParticipant(
                AgentRef.external("alice", i -> null), "REV", "Review prompt");
        var bob = new AgentParticipant(
                AgentRef.external("bob", i -> null), "IMP", "Implement prompt");

        ResponseMessageBuilder responseBuilder = (agent, result, state) -> {
            var msg = mock(MessageView.class);
            when(msg.content()).thenReturn("TEST:entry_type=COMMENT\nrole=" 
                    + agent.role() + "\n" + result.output());
            when(msg.sender()).thenReturn(agent.agentId());
            when(msg.correlationId()).thenReturn("p-resp-" + callCount.get());
            when(msg.id()).thenReturn((long) callCount.get());
            when(msg.type()).thenReturn(null);
            when(msg.createdAt()).thenReturn(java.time.Instant.now());
            return msg;
        };

        var orchestrator = new ConversationOrchestrator(
                projection,
                observationService,
                new RoundRobinTurnPolicy(),
                new MaxIterationsTermination<>(4),
                invoker,
                (agent, drain, state) -> agent.systemPrompt()
                        + "\n" + drain.currentPartition().renderedText(),
                responseBuilder,
                dispatched::add,
                List.of(alice, bob)
        );

        var trigger = mock(MessageView.class);
        when(trigger.content()).thenReturn("Start the debate");
        when(trigger.sender()).thenReturn("human");
        when(trigger.correlationId()).thenReturn("trigger");
        when(trigger.id()).thenReturn(0L);
        when(trigger.type()).thenReturn(null);
        when(trigger.createdAt()).thenReturn(java.time.Instant.now());

        var outcome = orchestrator.converse(trigger).await().indefinitely();

        assertThat(outcome.dispatchCount()).isEqualTo(4);
        assertThat(outcome.terminationDecision())
                .isInstanceOf(TerminationDecision.Complete.class);
        assertThat(dispatched).hasSize(4);
    }

    private PartitionedObservationService<MessageView, String> createObservationService() {
        return new PartitionedObservationService<>(
                (events, ctx) -> CompletableFuture.completedFuture(
                        new ObservationResult("rendered", List.of(),
                                events.size(), null, 0)),
                event -> Map.of(),
                msg -> msg.createdAt() != null ? msg.createdAt().toEpochMilli() : 0L,
                new EventLevel("conversation", 0));
    }
}
```

Note: `TestProjection` is a minimal `ConversationProjection` subclass for testing — it does not fold structured metadata (no sentinel), just passes state through. Create it as an inner class or package-private test class.

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl . -Dtest=ConversationOrchestratorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation failure

- [ ] **Step 3: Implement ConversationOrchestrator**

```java
package io.casehub.blocks.conversation.orchestration;

import io.casehub.blocks.agentic.AgentResult;
import io.casehub.blocks.agentic.model.AgentInvoker;
import io.casehub.blocks.agentic.termination.TerminationCondition;
import io.casehub.blocks.agentic.termination.TerminationContext;
import io.casehub.blocks.agentic.termination.TerminationDecision;
import io.casehub.blocks.conversation.ConversationProjection;
import io.casehub.blocks.conversation.ConversationState;
import io.casehub.blocks.summarisation.observation.PartitionedObservationService;
import io.casehub.qhorus.api.message.MessageView;
import io.smallrye.mutiny.Uni;

import java.time.Duration;
import java.time.Instant;
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

public class ConversationOrchestrator {

    private static final System.Logger LOG =
            System.getLogger(ConversationOrchestrator.class.getName());

    private final ConversationProjection projection;
    private final PartitionedObservationService<MessageView, String> observationService;
    private final TurnPolicy turnPolicy;
    private final TerminationCondition<ConversationState> terminationCondition;
    private final AgentInvoker<String> agentInvoker;
    private final PromptAssembler promptAssembler;
    private final ResponseMessageBuilder responseBuilder;
    private final Consumer<MessageView> responseDispatcher;
    private final List<AgentParticipant> participants;
    private volatile boolean terminated = false;

    public ConversationOrchestrator(
            ConversationProjection projection,
            PartitionedObservationService<MessageView, String> observationService,
            TurnPolicy turnPolicy,
            TerminationCondition<ConversationState> terminationCondition,
            AgentInvoker<String> agentInvoker,
            PromptAssembler promptAssembler,
            ResponseMessageBuilder responseBuilder,
            Consumer<MessageView> responseDispatcher,
            List<AgentParticipant> participants) {
        this.projection = projection;
        this.observationService = observationService;
        this.turnPolicy = turnPolicy;
        this.terminationCondition = terminationCondition;
        this.agentInvoker = agentInvoker;
        this.promptAssembler = promptAssembler;
        this.responseBuilder = responseBuilder;
        this.responseDispatcher = responseDispatcher;
        this.participants = List.copyOf(participants);

        for (var p : this.participants) {
            observationService.addObserver(p.agentId(), p.agentId());
        }
    }

    public Uni<ConversationOutcome> converse(MessageView triggeringMessage) {
        return Uni.createFrom().item(() -> {
            var start = Instant.now();
            var state = projection.identity();
            var allResults = new ArrayList<AgentResult>();
            var queue = new ArrayDeque<MessageView>();
            int dispatchCount = 0;

            queue.add(triggeringMessage);

            TerminationDecision finalDecision = TerminationDecision.Continue.INSTANCE;

            while (!queue.isEmpty() && !terminated) {
                var message = queue.poll();

                state = projection.apply(state, message);
                observationService.publishEvent(message);

                var turnContext = extractContext(message);
                var responders = turnPolicy.nextResponders(
                        state, turnContext, participants);

                for (var agent : responders) {
                    if (terminated) break;

                    AgentResult result;
                    try {
                        var drain = observationService.drain(
                                agent.agentId(), agent.agentId(),
                                System.currentTimeMillis());
                        var prompt = promptAssembler.assemble(agent, drain, state);
                        result = agentInvoker.invoke(agent.agentRef(), prompt)
                                .await().indefinitely();
                    } catch (Exception e) {
                        LOG.log(System.Logger.Level.WARNING,
                                "Agent invocation failed: " + agent.agentId(), e);
                        result = AgentResult.failure(agent.agentRef(), e.getMessage());
                    }

                    allResults.add(result);
                    dispatchCount++;

                    if (result.status() == AgentResult.AgentResultStatus.FAILURE
                            || result.status() == AgentResult.AgentResultStatus.TIMEOUT) {
                        continue;
                    }

                    var responseMessage = responseBuilder.build(agent, result, state);
                    state = projection.apply(state, responseMessage);
                    observationService.publishEvent(responseMessage);
                    responseDispatcher.accept(responseMessage);
                    queue.add(responseMessage);

                    var termCtx = new TerminationContext<>(
                            state, dispatchCount,
                            Duration.between(start, Instant.now()),
                            List.copyOf(allResults));
                    finalDecision = terminationCondition.evaluate(termCtx)
                            .await().indefinitely();

                    if (!(finalDecision instanceof TerminationDecision.Continue)) {
                        break;
                    }
                }

                if (!(finalDecision instanceof TerminationDecision.Continue)) {
                    break;
                }
            }

            if (finalDecision instanceof TerminationDecision.Continue) {
                finalDecision = new TerminationDecision.Complete("Queue drained");
            }

            return new ConversationOutcome(
                    state, finalDecision, allResults, dispatchCount,
                    Duration.between(start, Instant.now()));
        });
    }

    public void terminate() {
        terminated = true;
    }

    private TurnContext extractContext(MessageView message) {
        var content = message.content() != null ? message.content() : "";
        var sender = message.sender() != null ? message.sender() : "";
        return new TurnContext(sender, null, "", java.util.Map.of());
    }
}
```

- [ ] **Step 4: Create TestProjection helper**

```java
// Inner class in test file or package-private test helper
static class TestProjection extends io.casehub.blocks.conversation.ConversationProjection {
    @Override protected String sentinel() { return "TEST:"; }
    @Override protected boolean isPointInitiator(String entryType) {
        return "RAISE".equals(entryType);
    }
    @Override protected String statusAfter(String entryType) {
        return switch (entryType) {
            case "AGREE" -> "AGREED";
            case "COUNTER" -> "ACTIVE";
            case "DISPUTE" -> "DISPUTED";
            default -> null;
        };
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn --batch-mode test -pl . -Dtest=ConversationOrchestratorTest`
Expected: PASS

- [ ] **Step 6: Add integration tests**

Add these test methods to `ConversationOrchestratorTest`:

- `twoAgentDebate_convergence`: Alice raises a point, Bob agrees → `AllAgreedTermination` completes
- `agentFailure_skipsAndContinues`: One agent's invoker throws, other agent continues
- `emptyResponders_queueDrains`: Turn policy returns empty, conversation settles
- `responseDispatcher_calledPerResponse`: Verify sink receives each response in order
- `promptAssembly_perAgentContext`: Verify each agent gets different observation context via drain

Each test follows the same pattern: construct orchestrator with specific policies, mock invoker with scripted responses, assert on `ConversationOutcome`.

- [ ] **Step 7: Run all orchestrator tests**

Run: `mvn --batch-mode test -pl . -Dtest=ConversationOrchestratorTest`
Expected: PASS (all integration tests)

- [ ] **Step 8: Commit**

```
feat(#91): add ConversationOrchestrator composition root

Iterative queue-based loop composing ConversationProjection,
PartitionedObservationService, TurnPolicy, TerminationCondition,
AgentInvoker, PromptAssembler, and ResponseMessageBuilder.

Refs casehubio/blocks#91
```

---

### Task 5: Full Suite Verification and Consumer Guide

**Files:**
- Modify: `docs/guides/consumer-guide.md`
- Modify: `CLAUDE.md` (key directories section)

**Interfaces:**
- Consumes: All types from Tasks 1-4
- Produces: Documentation updates

- [ ] **Step 1: Run full test suite**

Run: `mvn --batch-mode test`
Expected: All existing + new tests PASS. No regressions.

- [ ] **Step 2: Update consumer guide**

Add a new section to `docs/guides/consumer-guide.md` documenting:
- The `conversation.orchestration` package and its purpose
- How to wire `ConversationOrchestrator` into a `ChannelBackend`
- Available turn policies and when to use each
- Available termination conditions and composition pattern
- The `PromptAssembler` and `ResponseMessageBuilder` extension points
- Example wiring (adapted from the spec's drafthouse example)

- [ ] **Step 3: Update CLAUDE.md key directories**

Add entries for the new package:
```
| `src/main/java/io/casehub/blocks/conversation/orchestration/` | Conversation orchestrator — TurnPolicy SPI, TerminationCondition impls, PromptAssembler, composition root |
| `src/test/java/io/casehub/blocks/conversation/orchestration/` | Tests for conversation orchestrator |
```

Add a new package table section documenting all types.

- [ ] **Step 4: Commit**

```
docs(#91): add conversation orchestrator to consumer guide and CLAUDE.md

Refs casehubio/blocks#91
```

- [ ] **Step 5: Run full test suite one final time**

Run: `mvn --batch-mode test`
Expected: PASS — all tests green, no regressions.
