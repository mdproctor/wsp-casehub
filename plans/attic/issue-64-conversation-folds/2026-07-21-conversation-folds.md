# Conversation Folds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #65 — common ground projection
**Issue group:** #65, #66

**Goal:** Add post-fold derivation layer over `ConversationState` —
epistemic fact classification via pluggable `EpistemicRule` and
convergence signal detection via pluggable `ConvergencePolicy`.

**Architecture:** Enrich `ThreadEntry` with `sender` and `createdAt`
(data preservation). Common ground and convergence are pure functions
from folded state, not fold extensions. `CommonGroundAnalyser` derives
`CommonGroundState` via a strategy `EpistemicRule`.
`ConvergenceAnalyser` derives `ConvergenceSignal` via a strategy
`ConvergencePolicy`, consuming `CommonGroundState` as input.
`ConvergenceTermination<T>` bridges to the platform's
`TerminationCondition` SPI.

**Tech Stack:** Java 21, JUnit 5, AssertJ, Mockito. No CDI, no Quarkus
runtime. Pure Java matching the existing conversation package pattern.

## Global Constraints

- All code in `io.casehub.blocks.conversation` except `ConvergenceTermination` (in `io.casehub.blocks.agentic.termination`)
- No new dependencies — `casehub-qhorus-api`, `java.time.Instant` already available
- Records are immutable — defensive copies in compact constructors for collections
- `@FunctionalInterface` on strategy interfaces (`EpistemicRule`, `ConvergencePolicy`)
- IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing — never bash grep/Edit on .java files
- `project_path` for all IntelliJ calls: `/Users/mdproctor/claude/casehub/worktrees/24/blocks`
- Tests: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test` (main checkout shares branch content)
- Build verification: `ide_build_project` or `ide_diagnostics` after edits

**Spec:** `docs/specs/2026-07-21-conversation-folds-design.md`

---

### Task 1: ThreadEntry Enrichment

Add `sender` and `createdAt` to `ThreadEntry`, propagate through
`ConversationFold` and `ConversationProjection`, update all existing tests.

**Files:**
- Modify: `src/main/java/io/casehub/blocks/conversation/ThreadEntry.java`
- Modify: `src/main/java/io/casehub/blocks/conversation/ConversationFold.java`
- Modify: `src/main/java/io/casehub/blocks/conversation/ConversationProjection.java`
- Modify: `src/test/java/io/casehub/blocks/conversation/ConversationFoldTest.java`
- Modify: `src/test/java/io/casehub/blocks/conversation/ConversationProjectionTest.java`
- Modify: `src/test/java/io/casehub/blocks/conversation/ConversationRendererTest.java`
- Modify: `src/test/java/io/casehub/blocks/conversation/TopicAwareConversationIntegrationTest.java`

**Interfaces:**
- Produces: `ThreadEntry(String entryId, Long messageId, MessageType messageType, String sender, Instant createdAt, String role, int round, String entryType, String content)` — used by Tasks 2, 3, 5, 6

- [ ] **Step 1: Modify ThreadEntry record**

Add `sender` and `createdAt` fields after `messageType`:

```java
public record ThreadEntry(
        String entryId,
        Long messageId,
        io.casehub.qhorus.api.message.MessageType messageType,
        String sender,
        java.time.Instant createdAt,
        String role,
        int round,
        String entryType,
        String content) {}
```

Use `ide_edit_member` with `member=ThreadEntry` (class declaration).

- [ ] **Step 2: Update ConversationFold — all seven methods**

Every method that constructs `ThreadEntry` gains `sender` and `createdAt`
parameters. Update each method signature and its `new ThreadEntry(...)` call.

**createPoint** — add `String sender, Instant createdAt` after `MessageType messageType`:

```java
public static ConversationState createPoint(ConversationState state,
                                            String pointId,
                                            String topic,
                                            Long messageId,
                                            io.casehub.qhorus.api.message.MessageType messageType,
                                            String sender,
                                            java.time.Instant createdAt,
                                            PointClassification classification,
                                            String role, int round,
                                            String entryType, String content) {
    var thread = new ArrayList<ThreadEntry>();
    thread.add(new ThreadEntry(pointId, messageId, messageType, sender, createdAt, role, round, entryType, content));
    var point = new ConversationPoint(pointId, topic, classification, thread,
                                      ConversationProtocol.STATUS_OPEN);
    var points = new LinkedHashMap<>(state.points());
    points.put(pointId, point);
    return new ConversationState(points, new ArrayList<>(state.humanFlags()),
                                 new ArrayList<>(state.memos()), new LinkedHashMap<>(state.subTaskFindings()));
}
```

**respondToPoint** — add `String sender, Instant createdAt` after `MessageType messageType`:

```java
public static ConversationState respondToPoint(ConversationState state,
                                               String targetId,
                                               Long messageId,
                                               io.casehub.qhorus.api.message.MessageType messageType,
                                               String sender,
                                               java.time.Instant createdAt,
                                               String role, int round,
                                               String entryType, String content,
                                               String newStatus) {
    if (!state.points().containsKey(targetId)) {return state;}
    ConversationPoint existing = state.points().get(targetId);
    var thread = new ArrayList<>(existing.thread());
    thread.add(new ThreadEntry(null, messageId, messageType, sender, createdAt, role, round, entryType, content));
    String resolvedStatus = newStatus != null ? newStatus : existing.status();
    var updated = new ConversationPoint(existing.id(), existing.topic(), existing.classification(),
                                        thread, resolvedStatus);
    var points = new LinkedHashMap<>(state.points());
    points.put(targetId, updated);
    return new ConversationState(points, new ArrayList<>(state.humanFlags()),
                                 new ArrayList<>(state.memos()), new LinkedHashMap<>(state.subTaskFindings()));
}
```

**flagHuman** — add `String sender, Instant createdAt` after `Long messageId`:

```java
public static ConversationState flagHuman(ConversationState state,
                                          String targetId,
                                          Long messageId,
                                          String sender,
                                          java.time.Instant createdAt,
                                          String role, int round,
                                          String content) {
    var points = new LinkedHashMap<>(state.points());
    if (targetId != null && points.containsKey(targetId)) {
        ConversationPoint p = points.get(targetId);
        var thread = new ArrayList<>(p.thread());
        thread.add(new ThreadEntry(null, messageId, null, sender, createdAt, role, round, ConversationProtocol.FLAG_HUMAN, content));
        points.put(targetId, new ConversationPoint(p.id(), p.topic(), p.classification(),
                                                   thread, ConversationProtocol.STATUS_ESCALATED));
    }
    var flags = new ArrayList<>(state.humanFlags());
    flags.add(new FlagEntry(null, round, role, content));
    return new ConversationState(points, flags,
                                 new ArrayList<>(state.memos()), new LinkedHashMap<>(state.subTaskFindings()));
}
```

**addMemo, requestSubTask, completeSubTask, errorSubTask** — these do not
create `ThreadEntry` instances, so their signatures stay unchanged.

Use `ide_edit_member` for each method.

- [ ] **Step 3: Update ConversationProjection — pass sender and createdAt**

In `handlePointInitiation`, add `message.sender()` and `message.createdAt()`
to the `ConversationFold.createPoint()` call:

```java
return ConversationFold.createPoint(state, pointId, message.topic(), message.id(), message.type(),
                                    message.sender(), message.createdAt(),
                                    classification, role, round, entryType, body);
```

In `handlePointResponse`, add to the `ConversationFold.respondToPoint()` call:

```java
return ConversationFold.respondToPoint(state, targetId, message.id(), message.type(),
                                       message.sender(), message.createdAt(),
                                       role, round, entryType, body, statusAfter(entryType));
```

In `handleFlagHuman`, add to the `ConversationFold.flagHuman()` call:

```java
return ConversationFold.flagHuman(state, message.correlationId(), message.id(),
                                  message.sender(), message.createdAt(),
                                  role, round, content);
```

Use `ide_replace_member` for each handler method.

- [ ] **Step 4: Update test helpers**

In `ConversationRendererTest`, update the `entry()` helper to pass
`null` for sender and createdAt:

```java
static ThreadEntry entry(String role, String entryType, String content) {
    return new ThreadEntry(null, null, null, null, null, role, 1, entryType, content);
}
```

In `ConversationFoldTest`, all `ConversationFold.createPoint()` calls
gain `null, null` for sender, createdAt after the `messageType` arg.
All `ConversationFold.respondToPoint()` calls gain `null, null` after
the `messageType` arg. All `ConversationFold.flagHuman()` calls gain
`null, null` after the `messageId` arg.

In `ConversationProjectionTest`, the message mock builder may need
`sender()` and `createdAt()` stubs. Add:

```java
when(msg.sender()).thenReturn("test-agent");
when(msg.createdAt()).thenReturn(java.time.Instant.now());
```

In `TopicAwareConversationIntegrationTest`, add the same stubs to the
`message()` helper.

- [ ] **Step 5: Verify existing tests pass**

Run: `ide_build_project` to check compilation.
Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all existing tests pass.

- [ ] **Step 6: Add enrichment-specific tests**

In `ConversationFoldTest`, add test verifying sender and createdAt
propagate:

```java
@Test
void createPoint_preservesSenderAndCreatedAt() {
    var now = java.time.Instant.now();
    var classification = new PointClassification(Priority.HIGH, "global", null);
    var result = ConversationFold.createPoint(empty,
            "p1", "general", 42L, MessageType.COMMAND, "agent-a", now,
            classification, "REVIEWER", 1, "RAISE", "content");

    ThreadEntry entry = result.points().get("p1").thread().get(0);
    assertThat(entry.sender()).isEqualTo("agent-a");
    assertThat(entry.createdAt()).isEqualTo(now);
}
```

Add similar test for `respondToPoint` and `flagHuman`.

In `ConversationProjectionTest`, add test verifying the projection passes
sender and createdAt through from the `MessageView`:

```java
@Test
void apply_preservesSenderAndCreatedAt() {
    var now = java.time.Instant.now();
    var msg = mock(MessageView.class);
    when(msg.content()).thenReturn("TEST:entryType=OPEN_TOPIC\nrole=REV\nround=1\npriority=HIGH\nscope=test\n---\nPoint content");
    when(msg.correlationId()).thenReturn("p1");
    when(msg.topic()).thenReturn("general");
    when(msg.id()).thenReturn(1L);
    when(msg.type()).thenReturn(MessageType.COMMAND);
    when(msg.sender()).thenReturn("agent-reviewer");
    when(msg.createdAt()).thenReturn(now);

    var state = projection.apply(projection.identity(), msg);
    ThreadEntry entry = state.points().get("p1").thread().get(0);
    assertThat(entry.sender()).isEqualTo("agent-reviewer");
    assertThat(entry.createdAt()).isEqualTo(now);
}
```

- [ ] **Step 7: Run all tests, verify pass**

Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all tests pass including new enrichment tests.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add -A
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "feat(#65): enrich ThreadEntry with sender and createdAt

Preserve MessageView.sender() and MessageView.createdAt() through the
conversation fold — needed for common ground participant tracking and
convergence temporal analysis.

Refs casehubio/blocks#65, casehubio/blocks#66"
```

---

### Task 2: Common Ground — Types, Analyser, and Epistemic Rules

Create the common ground derivation layer: `EpistemicStatus`,
`ParticipantContext`, `EpistemicRule`, `EpistemicRules`,
`GroundedFact`, `CommonGroundState`, `CommonGroundAnalyser`.

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/EpistemicStatus.java`
- Create: `src/main/java/io/casehub/blocks/conversation/ParticipantContext.java`
- Create: `src/main/java/io/casehub/blocks/conversation/EpistemicRule.java`
- Create: `src/main/java/io/casehub/blocks/conversation/EpistemicRules.java`
- Create: `src/main/java/io/casehub/blocks/conversation/GroundedFact.java`
- Create: `src/main/java/io/casehub/blocks/conversation/CommonGroundState.java`
- Create: `src/main/java/io/casehub/blocks/conversation/CommonGroundAnalyser.java`
- Create: `src/test/java/io/casehub/blocks/conversation/CommonGroundAnalyserTest.java`

**Interfaces:**
- Consumes: `ThreadEntry.sender()`, `ThreadEntry.createdAt()` from Task 1; `ConversationState`, `ConversationPoint` (existing)
- Produces: `CommonGroundAnalyser.analyse(ConversationState, EpistemicRule) → CommonGroundState` — used by Tasks 3, 4, 5, 6

- [ ] **Step 1: Create the type records**

**EpistemicStatus.java:**

```java
package io.casehub.blocks.conversation;

public enum EpistemicStatus {
    ESTABLISHED,
    PENDING,
    DISPUTED
}
```

**ParticipantContext.java:**

```java
package io.casehub.blocks.conversation;

import java.util.Set;

public record ParticipantContext(
        Set<String> allParticipants,
        Set<String> respondedBy,
        Set<String> acknowledgedBy,
        Set<String> completedBy,
        Set<String> disputedBy,
        Set<String> failedBy,
        int roundsSinceLastActivity) {
    public ParticipantContext {
        allParticipants = Set.copyOf(allParticipants);
        respondedBy = Set.copyOf(respondedBy);
        acknowledgedBy = Set.copyOf(acknowledgedBy);
        completedBy = Set.copyOf(completedBy);
        disputedBy = Set.copyOf(disputedBy);
        failedBy = Set.copyOf(failedBy);
    }
}
```

**GroundedFact.java:**

```java
package io.casehub.blocks.conversation;

import java.util.Set;

public record GroundedFact(
        String pointId,
        String topic,
        EpistemicStatus status,
        String content,
        Set<String> acknowledgedBy,
        Set<String> disputedBy,
        int round) {
    public GroundedFact {
        acknowledgedBy = Set.copyOf(acknowledgedBy);
        disputedBy = Set.copyOf(disputedBy);
    }
}
```

**CommonGroundState.java:**

```java
package io.casehub.blocks.conversation;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

public record CommonGroundState(
        Map<String, GroundedFact> establishedFacts,
        Map<String, GroundedFact> pendingClaims,
        Map<String, GroundedFact> disputedPoints) {
    public CommonGroundState {
        establishedFacts = Collections.unmodifiableMap(new LinkedHashMap<>(establishedFacts));
        pendingClaims = Collections.unmodifiableMap(new LinkedHashMap<>(pendingClaims));
        disputedPoints = Collections.unmodifiableMap(new LinkedHashMap<>(disputedPoints));
    }
}
```

Use `ide_create_file` for each.

- [ ] **Step 2: Create EpistemicRule interface**

```java
package io.casehub.blocks.conversation;

@FunctionalInterface
public interface EpistemicRule {
    EpistemicStatus classify(ConversationPoint point, ParticipantContext context);

    default EpistemicRule and(EpistemicRule other) {
        return (point, context) -> {
            EpistemicStatus a = this.classify(point, context);
            EpistemicStatus b = other.classify(point, context);
            // Most conservative wins: DISPUTED > PENDING > ESTABLISHED
            if (a == EpistemicStatus.DISPUTED || b == EpistemicStatus.DISPUTED) {
                return EpistemicStatus.DISPUTED;
            }
            if (a == EpistemicStatus.PENDING || b == EpistemicStatus.PENDING) {
                return EpistemicStatus.PENDING;
            }
            return EpistemicStatus.ESTABLISHED;
        };
    }

    default EpistemicRule or(EpistemicRule other) {
        return (point, context) -> {
            EpistemicStatus a = this.classify(point, context);
            EpistemicStatus b = other.classify(point, context);
            // Most permissive wins: ESTABLISHED > PENDING > DISPUTED
            if (a == EpistemicStatus.ESTABLISHED || b == EpistemicStatus.ESTABLISHED) {
                return EpistemicStatus.ESTABLISHED;
            }
            if (a == EpistemicStatus.PENDING || b == EpistemicStatus.PENDING) {
                return EpistemicStatus.PENDING;
            }
            return EpistemicStatus.DISPUTED;
        };
    }
}
```

- [ ] **Step 3: Create EpistemicRules — three provided rules**

```java
package io.casehub.blocks.conversation;

public final class EpistemicRules {

    private EpistemicRules() {}

    public static EpistemicRule explicitAcknowledgement(int minParticipants) {
        return (point, context) -> {
            if (!context.disputedBy().isEmpty()) {
                return EpistemicStatus.DISPUTED;
            }
            if (context.acknowledgedBy().size() >= minParticipants) {
                return EpistemicStatus.ESTABLISHED;
            }
            return EpistemicStatus.PENDING;
        };
    }

    public static EpistemicRule tacitAcceptance(int windowRounds) {
        return (point, context) -> {
            if (!context.disputedBy().isEmpty() || !context.failedBy().isEmpty()) {
                return EpistemicStatus.DISPUTED;
            }
            if (context.respondedBy().size() >= 1
                    && context.roundsSinceLastActivity() >= windowRounds) {
                return EpistemicStatus.ESTABLISHED;
            }
            return EpistemicStatus.PENDING;
        };
    }

    public static EpistemicRule commitmentResolution() {
        return (point, context) -> {
            if (!context.disputedBy().isEmpty() || !context.failedBy().isEmpty()) {
                return EpistemicStatus.DISPUTED;
            }
            if (!context.completedBy().isEmpty()) {
                return EpistemicStatus.ESTABLISHED;
            }
            return EpistemicStatus.PENDING;
        };
    }
}
```

- [ ] **Step 4: Create CommonGroundAnalyser**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.MessageType;

import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;

public final class CommonGroundAnalyser {

    private CommonGroundAnalyser() {}

    public static CommonGroundState analyse(ConversationState state, EpistemicRule rule) {
        int maxRound = maxRound(state);

        var established = new LinkedHashMap<String, GroundedFact>();
        var pending = new LinkedHashMap<String, GroundedFact>();
        var disputed = new LinkedHashMap<String, GroundedFact>();

        for (var entry : state.points().entrySet()) {
            ConversationPoint point = entry.getValue();
            ParticipantContext context = buildContext(point, maxRound);
            EpistemicStatus status = rule.classify(point, context);

            String content = point.thread().isEmpty() ? "" : point.thread().get(0).content();
            int round = point.thread().isEmpty() ? 0 : point.thread().get(0).round();

            Set<String> ackBy = context.acknowledgedBy();
            Set<String> dispBy = new HashSet<>(context.disputedBy());
            dispBy.addAll(context.failedBy());

            var fact = new GroundedFact(point.id(), point.topic(), status,
                                        content, ackBy, dispBy, round);

            switch (status) {
                case ESTABLISHED -> established.put(point.id(), fact);
                case PENDING -> pending.put(point.id(), fact);
                case DISPUTED -> disputed.put(point.id(), fact);
            }
        }

        return new CommonGroundState(established, pending, disputed);
    }

    static ParticipantContext buildContext(ConversationPoint point, int maxRound) {
        var allParticipants = new HashSet<String>();
        var respondedBy = new HashSet<String>();
        var acknowledgedBy = new HashSet<String>();
        var completedBy = new HashSet<String>();
        var disputedBy = new HashSet<String>();
        var failedBy = new HashSet<String>();
        int lastRound = 0;

        for (ThreadEntry te : point.thread()) {
            if (te.sender() != null) {
                allParticipants.add(te.sender());
            }
            if (te.messageType() != null && te.sender() != null) {
                if (te.messageType().isAgentVisible()
                        && te.messageType() != MessageType.EVENT
                        && te.messageType() != MessageType.QUERY
                        && te.messageType() != MessageType.COMMAND) {
                    respondedBy.add(te.sender());
                }
                if (te.messageType() == MessageType.RESPONSE
                        || te.messageType() == MessageType.DONE) {
                    acknowledgedBy.add(te.sender());
                }
                if (te.messageType() == MessageType.DONE) {
                    completedBy.add(te.sender());
                }
                if (te.messageType() == MessageType.DECLINE) {
                    disputedBy.add(te.sender());
                }
                if (te.messageType() == MessageType.FAILURE) {
                    failedBy.add(te.sender());
                }
            }
            if (te.round() > lastRound) {
                lastRound = te.round();
            }
        }

        return new ParticipantContext(allParticipants, respondedBy,
                acknowledgedBy, completedBy, disputedBy, failedBy,
                maxRound - lastRound);
    }

    private static int maxRound(ConversationState state) {
        int max = 0;
        for (ConversationPoint point : state.points().values()) {
            for (ThreadEntry te : point.thread()) {
                if (te.round() > max) {max = te.round();}
            }
        }
        return max;
    }
}
```

- [ ] **Step 5: Write CommonGroundAnalyserTest — core rule tests**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.MessageType;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class CommonGroundAnalyserTest {

    private static final Instant T0 = Instant.parse("2026-07-21T10:00:00Z");

    static ConversationState emptyState() {
        return new ConversationState(Map.of(), List.of(), List.of(), Map.of());
    }

    static ThreadEntry entry(String sender, MessageType type, String role, int round, String entryType, String content) {
        return new ThreadEntry(null, null, type, sender, T0.plusSeconds(round * 60L), role, round, entryType, content);
    }

    static ConversationPoint point(String id, String topic, List<ThreadEntry> thread) {
        return new ConversationPoint(id, topic,
                new PointClassification(Priority.MEDIUM, null, null),
                thread, ConversationProtocol.STATUS_OPEN);
    }

    static ConversationState stateWith(ConversationPoint... points) {
        var map = new LinkedHashMap<String, ConversationPoint>();
        for (var p : points) map.put(p.id(), p);
        return new ConversationState(map, List.of(), List.of(), Map.of());
    }

    @Nested
    class ExplicitAcknowledgement {

        @Test
        void noResponses_isPending() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            assertThat(result.pendingClaims()).containsKey("p1");
        }

        @Test
        void oneAck_meetsThreshold_isEstablished() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "agreed")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            assertThat(result.establishedFacts()).containsKey("p1");
        }

        @Test
        void belowThreshold_isPending() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "agreed")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(2));
            assertThat(result.pendingClaims()).containsKey("p1");
        }

        @Test
        void decline_isDisputed() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.DECLINE, "IMP", 2, "REJECT", "disagree")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            assertThat(result.disputedPoints()).containsKey("p1");
        }

        @Test
        void done_countsAsAcknowledgement() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "do this"),
                    entry("bob", MessageType.DONE, "IMP", 2, "ACCEPT", "done")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            assertThat(result.establishedFacts()).containsKey("p1");
        }

        @Test
        void status_doesNotCountAsAcknowledgement() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.STATUS, "IMP", 2, "ACCEPT", "working on it")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            assertThat(result.pendingClaims()).containsKey("p1");
        }
    }

    @Nested
    class TacitAcceptance {

        @Test
        void zeroResponses_neverEstablished() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim")));
            var state = stateWith(p);
            // Simulate 10 rounds of inactivity by adding another point at round 11
            var p2 = point("p2", "t", List.of(
                    entry("carol", MessageType.COMMAND, "REV", 11, "RAISE", "other")));
            state = stateWith(p, p2);
            var result = CommonGroundAnalyser.analyse(state,
                    EpistemicRules.tacitAcceptance(3));
            assertThat(result.pendingClaims()).containsKey("p1");
        }

        @Test
        void respondedAndStale_isEstablished() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.STATUS, "IMP", 2, "ACCEPT", "noted")));
            var p2 = point("p2", "t", List.of(
                    entry("carol", MessageType.COMMAND, "REV", 6, "RAISE", "other")));
            var result = CommonGroundAnalyser.analyse(stateWith(p, p2),
                    EpistemicRules.tacitAcceptance(3));
            assertThat(result.establishedFacts()).containsKey("p1");
        }

        @Test
        void decline_blocksEstablishment() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.DECLINE, "IMP", 2, "REJECT", "no")));
            var p2 = point("p2", "t", List.of(
                    entry("carol", MessageType.COMMAND, "REV", 6, "RAISE", "other")));
            var result = CommonGroundAnalyser.analyse(stateWith(p, p2),
                    EpistemicRules.tacitAcceptance(3));
            assertThat(result.disputedPoints()).containsKey("p1");
        }

        @Test
        void failure_blocksEstablishment() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "do this"),
                    entry("bob", MessageType.FAILURE, "IMP", 2, "ACCEPT", "failed")));
            var p2 = point("p2", "t", List.of(
                    entry("carol", MessageType.COMMAND, "REV", 6, "RAISE", "other")));
            var result = CommonGroundAnalyser.analyse(stateWith(p, p2),
                    EpistemicRules.tacitAcceptance(3));
            assertThat(result.disputedPoints()).containsKey("p1");
        }
    }

    @Nested
    class CommitmentResolution {

        @Test
        void done_isEstablished() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "do X"),
                    entry("bob", MessageType.DONE, "IMP", 2, "ACCEPT", "done")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.commitmentResolution());
            assertThat(result.establishedFacts()).containsKey("p1");
        }

        @Test
        void responseOnly_isPending() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "do X"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "acknowledged")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.commitmentResolution());
            assertThat(result.pendingClaims()).containsKey("p1");
        }

        @Test
        void failure_isDisputed() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "do X"),
                    entry("bob", MessageType.FAILURE, "IMP", 2, "ACCEPT", "failed")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.commitmentResolution());
            assertThat(result.disputedPoints()).containsKey("p1");
        }
    }

    @Nested
    class Composition {

        @Test
        void and_returnsConservative() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "ok")));
            // explicitAck(1) → ESTABLISHED, commitmentResolution → PENDING
            var rule = EpistemicRules.explicitAcknowledgement(1)
                    .and(EpistemicRules.commitmentResolution());
            var result = CommonGroundAnalyser.analyse(stateWith(p), rule);
            assertThat(result.pendingClaims()).containsKey("p1");
        }

        @Test
        void or_returnsPermissive() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "ok")));
            // explicitAck(1) → ESTABLISHED, commitmentResolution → PENDING
            var rule = EpistemicRules.explicitAcknowledgement(1)
                    .or(EpistemicRules.commitmentResolution());
            var result = CommonGroundAnalyser.analyse(stateWith(p), rule);
            assertThat(result.establishedFacts()).containsKey("p1");
        }
    }

    @Nested
    class EdgeCases {

        @Test
        void emptyState_producesEmptyCommonGround() {
            var result = CommonGroundAnalyser.analyse(emptyState(),
                    EpistemicRules.explicitAcknowledgement(1));
            assertThat(result.establishedFacts()).isEmpty();
            assertThat(result.pendingClaims()).isEmpty();
            assertThat(result.disputedPoints()).isEmpty();
        }

        @Test
        void groundedFact_containsFirstThreadContent() {
            var p = point("p1", "review", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "the claim"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "agreed")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            var fact = result.establishedFacts().get("p1");
            assertThat(fact.content()).isEqualTo("the claim");
            assertThat(fact.topic()).isEqualTo("review");
            assertThat(fact.round()).isEqualTo(1);
        }

        @Test
        void groundedFact_acknowledgedByContainsCorrectSenders() {
            var p = point("p1", "t", List.of(
                    entry("alice", MessageType.COMMAND, "REV", 1, "RAISE", "claim"),
                    entry("bob", MessageType.RESPONSE, "IMP", 2, "ACCEPT", "ok"),
                    entry("carol", MessageType.DONE, "IMP2", 3, "ACCEPT", "done")));
            var result = CommonGroundAnalyser.analyse(stateWith(p),
                    EpistemicRules.explicitAcknowledgement(1));
            var fact = result.establishedFacts().get("p1");
            assertThat(fact.acknowledgedBy()).containsExactlyInAnyOrder("bob", "carol");
        }
    }
}
```

- [ ] **Step 6: Run tests, verify all pass**

Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add -A
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "feat(#65): common ground derivation — EpistemicRule + CommonGroundAnalyser

Stateless post-fold derivation classifying conversation points as
ESTABLISHED, PENDING, or DISPUTED. Three provided rules:
explicitAcknowledgement, tacitAcceptance, commitmentResolution.
Rules compose via and() (conservative) / or() (permissive).

Refs casehubio/blocks#65"
```

---

### Task 3: Convergence — Types, Analyser, and Policies

Create the convergence derivation layer: `ConvergenceState`,
`ConvergenceSignal`, `ConvergenceContext`, `ConvergencePolicy`,
`ConvergencePolicies`, `ConvergenceAnalyser`.

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/ConvergenceState.java`
- Create: `src/main/java/io/casehub/blocks/conversation/ConvergenceSignal.java`
- Create: `src/main/java/io/casehub/blocks/conversation/ConvergenceContext.java`
- Create: `src/main/java/io/casehub/blocks/conversation/ConvergencePolicy.java`
- Create: `src/main/java/io/casehub/blocks/conversation/ConvergencePolicies.java`
- Create: `src/main/java/io/casehub/blocks/conversation/ConvergenceAnalyser.java`
- Create: `src/test/java/io/casehub/blocks/conversation/ConvergenceAnalyserTest.java`

**Interfaces:**
- Consumes: `CommonGroundAnalyser.analyse()`, `CommonGroundState` from Task 2; `ConversationState`, `ThreadEntry` (existing + Task 1)
- Produces: `ConvergenceAnalyser.analyse(ConversationState, CommonGroundState, ConvergencePolicy, int) → ConvergenceSignal` — used by Tasks 4, 5, 6

- [ ] **Step 1: Create type records and enum**

**ConvergenceState.java:**

```java
package io.casehub.blocks.conversation;

public enum ConvergenceState {
    PROGRESSING,
    CONVERGING,
    CONSENSUS,
    DEADLOCK,
    DIMINISHING_RETURNS
}
```

**ConvergenceSignal.java:**

```java
package io.casehub.blocks.conversation;

public record ConvergenceSignal(
        ConvergenceState state,
        double confidence,
        String reason) {}
```

**ConvergenceContext.java:**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.MessageType;

import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.Map;

public record ConvergenceContext(
        int totalPoints,
        int establishedCount,
        int pendingCount,
        int disputedCount,
        double recentSimilarity,
        double messageLengthTrend,
        int roundsSinceNewPoint,
        int roundsSinceStatusChange,
        Map<MessageType, Integer> recentMessageTypeCounts) {
    public ConvergenceContext {
        recentMessageTypeCounts = Collections.unmodifiableMap(
                new LinkedHashMap<>(recentMessageTypeCounts));
    }
}
```

**ConvergencePolicy.java:**

```java
package io.casehub.blocks.conversation;

@FunctionalInterface
public interface ConvergencePolicy {
    ConvergenceSignal evaluate(ConversationState state,
                               CommonGroundState commonGround,
                               ConvergenceContext context);
}
```

Use `ide_create_file` for each.

- [ ] **Step 2: Create ConvergenceAnalyser**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.MessageType;

import java.util.*;

public final class ConvergenceAnalyser {

    private ConvergenceAnalyser() {}

    public static ConvergenceSignal analyse(ConversationState state,
                                            CommonGroundState commonGround,
                                            ConvergencePolicy policy,
                                            int recentWindow) {
        ConvergenceContext context = buildContext(state, commonGround, recentWindow);
        return policy.evaluate(state, commonGround, context);
    }

    static ConvergenceContext buildContext(ConversationState state,
                                           CommonGroundState commonGround,
                                           int recentWindow) {
        int totalPoints = commonGround.establishedFacts().size()
                        + commonGround.pendingClaims().size()
                        + commonGround.disputedPoints().size();

        List<ThreadEntry> allEntries = flattenAndSort(state);
        int recentStart = Math.max(0, allEntries.size() - recentWindow);
        List<ThreadEntry> recent = allEntries.subList(recentStart, allEntries.size());

        double recentSimilarity = computeSimilarity(recent);
        double messageLengthTrend = computeLengthTrend(allEntries, recent);

        int maxRound = allEntries.isEmpty() ? 0
                : allEntries.stream().mapToInt(ThreadEntry::round).max().orElse(0);
        int lastNewPointRound = lastNewPointRound(state);
        int lastStatusChangeRound = lastStatusChangeRound(state, commonGround);

        var recentTypeCounts = new LinkedHashMap<MessageType, Integer>();
        for (ThreadEntry te : recent) {
            if (te.messageType() != null) {
                recentTypeCounts.merge(te.messageType(), 1, Integer::sum);
            }
        }

        return new ConvergenceContext(
                totalPoints,
                commonGround.establishedFacts().size(),
                commonGround.pendingClaims().size(),
                commonGround.disputedPoints().size(),
                recentSimilarity,
                messageLengthTrend,
                maxRound - lastNewPointRound,
                maxRound - lastStatusChangeRound,
                recentTypeCounts);
    }

    private static List<ThreadEntry> flattenAndSort(ConversationState state) {
        var entries = new ArrayList<ThreadEntry>();
        for (ConversationPoint point : state.points().values()) {
            entries.addAll(point.thread());
        }
        entries.sort(Comparator.comparingInt(ThreadEntry::round)
                .thenComparing(e -> e.createdAt() != null ? e.createdAt() : java.time.Instant.EPOCH));
        return entries;
    }

    private static double computeSimilarity(List<ThreadEntry> recent) {
        if (recent.size() < 2) {return 0.0;}
        double sum = 0;
        int count = 0;
        for (int i = 1; i < recent.size(); i++) {
            sum += jaccardSimilarity(
                    tokenize(recent.get(i - 1).content()),
                    tokenize(recent.get(i).content()));
            count++;
        }
        return count > 0 ? sum / count : 0.0;
    }

    private static double computeLengthTrend(List<ThreadEntry> all, List<ThreadEntry> recent) {
        if (all.isEmpty() || recent.isEmpty()) {return 1.0;}
        double overallAvg = all.stream()
                .mapToInt(e -> e.content() != null ? e.content().length() : 0)
                .average().orElse(1.0);
        double recentAvg = recent.stream()
                .mapToInt(e -> e.content() != null ? e.content().length() : 0)
                .average().orElse(1.0);
        return overallAvg > 0 ? recentAvg / overallAvg : 1.0;
    }

    private static int lastNewPointRound(ConversationState state) {
        int last = 0;
        for (ConversationPoint point : state.points().values()) {
            if (!point.thread().isEmpty()) {
                int firstRound = point.thread().get(0).round();
                if (firstRound > last) {last = firstRound;}
            }
        }
        return last;
    }

    private static int lastStatusChangeRound(ConversationState state,
                                              CommonGroundState commonGround) {
        // Approximate: the latest round of any established or disputed point's last entry
        int last = 0;
        for (ConversationPoint point : state.points().values()) {
            if (commonGround.establishedFacts().containsKey(point.id())
                    || commonGround.disputedPoints().containsKey(point.id())) {
                for (ThreadEntry te : point.thread()) {
                    if (te.round() > last) {last = te.round();}
                }
            }
        }
        return last;
    }

    static double jaccardSimilarity(Set<String> a, Set<String> b) {
        if (a.isEmpty() && b.isEmpty()) {return 1.0;}
        if (a.isEmpty() || b.isEmpty()) {return 0.0;}
        long intersection = a.stream().filter(b::contains).count();
        var union = new HashSet<>(a);
        union.addAll(b);
        return (double) intersection / union.size();
    }

    static Set<String> tokenize(String text) {
        if (text == null || text.isBlank()) {return Set.of();}
        var tokens = new HashSet<String>();
        for (String token : text.split("\\s+")) {
            String cleaned = token.toLowerCase().replaceAll("[^a-z0-9]", "");
            if (!cleaned.isEmpty()) {tokens.add(cleaned);}
        }
        return tokens;
    }
}
```

- [ ] **Step 3: Create ConvergencePolicies**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.MessageType;

import java.util.Comparator;
import java.util.Set;

public final class ConvergencePolicies {

    private ConvergencePolicies() {}

    private static final Comparator<ConvergenceSignal> TIEBREAKER =
            Comparator.comparingDouble(ConvergenceSignal::confidence)
                    .thenComparingInt(s -> severity(s.state()));

    public static ConvergencePolicy structural(double similarityThreshold, int staleRounds) {
        return (state, commonGround, context) -> {
            if (context.totalPoints() == 0) {
                return new ConvergenceSignal(ConvergenceState.PROGRESSING, 0.0, "no points yet");
            }

            double establishedRatio = (double) context.establishedCount() / context.totalPoints();
            double disputedRatio = (double) context.disputedCount() / context.totalPoints();

            // Deadlock: high content similarity
            if (context.recentSimilarity() >= similarityThreshold) {
                return new ConvergenceSignal(ConvergenceState.DEADLOCK,
                        context.recentSimilarity(),
                        "content similarity " + String.format("%.2f", context.recentSimilarity())
                                + " exceeds threshold");
            }

            // Diminishing returns: stale rounds
            if (context.roundsSinceNewPoint() >= staleRounds) {
                return new ConvergenceSignal(ConvergenceState.DIMINISHING_RETURNS,
                        Math.min(1.0, (double) context.roundsSinceNewPoint() / (staleRounds * 2)),
                        "no new points for " + context.roundsSinceNewPoint() + " rounds");
            }

            // Diminishing returns: STATUS-heavy
            int statusCount = context.recentMessageTypeCounts().getOrDefault(MessageType.STATUS, 0);
            int substantiveCount = context.recentMessageTypeCounts().getOrDefault(MessageType.RESPONSE, 0)
                    + context.recentMessageTypeCounts().getOrDefault(MessageType.COMMAND, 0)
                    + context.recentMessageTypeCounts().getOrDefault(MessageType.QUERY, 0);
            if (substantiveCount > 0 && statusCount > substantiveCount * 2) {
                return new ConvergenceSignal(ConvergenceState.DIMINISHING_RETURNS,
                        0.6, "STATUS messages dominate — ratio " + statusCount + ":" + substantiveCount);
            }

            // Consensus: near-complete agreement
            if (establishedRatio >= 0.9) {
                return new ConvergenceSignal(ConvergenceState.CONSENSUS,
                        establishedRatio,
                        context.establishedCount() + "/" + context.totalPoints() + " points established");
            }

            // Converging: more than half established and still active
            if (establishedRatio >= 0.5 && context.roundsSinceStatusChange() <= 2) {
                return new ConvergenceSignal(ConvergenceState.CONVERGING,
                        establishedRatio,
                        "common ground growing — " + context.pendingCount() + " pending, "
                                + context.disputedCount() + " disputed");
            }

            return new ConvergenceSignal(ConvergenceState.PROGRESSING, 0.0, "conversation active");
        };
    }

    public static ConvergencePolicy commonGroundRatio(double consensusThreshold,
                                                       double deadlockDisputeRatio) {
        return (state, commonGround, context) -> {
            if (context.totalPoints() == 0) {
                return new ConvergenceSignal(ConvergenceState.PROGRESSING, 0.0, "no points yet");
            }
            double establishedRatio = (double) context.establishedCount() / context.totalPoints();
            double disputedRatio = (double) context.disputedCount() / context.totalPoints();

            if (establishedRatio >= consensusThreshold) {
                return new ConvergenceSignal(ConvergenceState.CONSENSUS,
                        establishedRatio,
                        context.establishedCount() + "/" + context.totalPoints() + " established");
            }
            if (disputedRatio >= deadlockDisputeRatio) {
                return new ConvergenceSignal(ConvergenceState.DEADLOCK,
                        disputedRatio,
                        context.disputedCount() + "/" + context.totalPoints() + " disputed");
            }
            return new ConvergenceSignal(ConvergenceState.PROGRESSING, 0.0, "conversation active");
        };
    }

    public static ConvergencePolicy composite(ConvergencePolicy... policies) {
        return (state, commonGround, context) -> {
            ConvergenceSignal best = null;
            for (ConvergencePolicy policy : policies) {
                ConvergenceSignal signal = policy.evaluate(state, commonGround, context);
                if (best == null || TIEBREAKER.compare(signal, best) > 0) {
                    best = signal;
                }
            }
            return best != null ? best
                    : new ConvergenceSignal(ConvergenceState.PROGRESSING, 0.0, "no policies");
        };
    }

    private static int severity(ConvergenceState state) {
        return switch (state) {
            case DEADLOCK -> 5;
            case DIMINISHING_RETURNS -> 4;
            case CONVERGING -> 3;
            case PROGRESSING -> 2;
            case CONSENSUS -> 1;
        };
    }
}
```

- [ ] **Step 4: Write ConvergenceAnalyserTest**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.MessageType;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class ConvergenceAnalyserTest {

    private static final Instant T0 = Instant.parse("2026-07-21T10:00:00Z");

    // reuse helpers from CommonGroundAnalyserTest pattern
    static ThreadEntry entry(String sender, MessageType type, int round, String content) {
        return new ThreadEntry(null, null, type, sender, T0.plusSeconds(round * 60L),
                sender, round, "ENTRY", content);
    }

    static ConversationPoint point(String id, List<ThreadEntry> thread) {
        return new ConversationPoint(id, "general",
                new PointClassification(Priority.MEDIUM, null, null),
                thread, ConversationProtocol.STATUS_OPEN);
    }

    static ConversationState stateWith(ConversationPoint... points) {
        var map = new LinkedHashMap<String, ConversationPoint>();
        for (var p : points) map.put(p.id(), p);
        return new ConversationState(map, List.of(), List.of(), Map.of());
    }

    @Nested
    class StructuralPolicy {

        @Test
        void highSimilarity_isDeadlock() {
            var p = point("p1", List.of(
                    entry("a", MessageType.COMMAND, 1, "we should use approach X"),
                    entry("b", MessageType.RESPONSE, 2, "we should use approach X"),
                    entry("a", MessageType.RESPONSE, 3, "yes use approach X")));
            var state = stateWith(p);
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.structural(0.5, 5), 10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.DEADLOCK);
        }

        @Test
        void staleRounds_isDiminishingReturns() {
            // All points initiated in round 1, max round driven by another point at round 10
            var p1 = point("p1", List.of(
                    entry("a", MessageType.COMMAND, 1, "point one")));
            var p2 = point("p2", List.of(
                    entry("b", MessageType.COMMAND, 1, "point two"),
                    entry("a", MessageType.STATUS, 10, "still here")));
            var state = stateWith(p1, p2);
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.structural(0.95, 3), 10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.DIMINISHING_RETURNS);
        }

        @Test
        void allEstablished_isConsensus() {
            var p1 = point("p1", List.of(
                    entry("a", MessageType.COMMAND, 1, "claim one"),
                    entry("b", MessageType.RESPONSE, 2, "agreed")));
            var p2 = point("p2", List.of(
                    entry("a", MessageType.COMMAND, 1, "claim two"),
                    entry("b", MessageType.RESPONSE, 2, "agreed too")));
            var state = stateWith(p1, p2);
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.structural(0.95, 5), 10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.CONSENSUS);
        }

        @Test
        void emptyState_isProgressing() {
            var state = new ConversationState(Map.of(), List.of(), List.of(), Map.of());
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.structural(0.8, 3), 10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.PROGRESSING);
        }
    }

    @Nested
    class CommonGroundRatioPolicy {

        @Test
        void aboveConsensusThreshold_isConsensus() {
            var p1 = point("p1", List.of(
                    entry("a", MessageType.COMMAND, 1, "A"),
                    entry("b", MessageType.DONE, 2, "done")));
            var state = stateWith(p1);
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.commitmentResolution());
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.commonGroundRatio(0.8, 0.5), 10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.CONSENSUS);
        }

        @Test
        void aboveDeadlockThreshold_isDeadlock() {
            var p1 = point("p1", List.of(
                    entry("a", MessageType.COMMAND, 1, "A"),
                    entry("b", MessageType.DECLINE, 2, "no")));
            var state = stateWith(p1);
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.commitmentResolution());
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.commonGroundRatio(0.8, 0.5), 10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.DEADLOCK);
        }
    }

    @Nested
    class CompositePolicy {

        @Test
        void highestConfidence_wins() {
            // structural → PROGRESSING(0.0), commonGroundRatio → CONSENSUS(1.0)
            var p1 = point("p1", List.of(
                    entry("a", MessageType.COMMAND, 1, "claim"),
                    entry("b", MessageType.DONE, 2, "done")));
            var state = stateWith(p1);
            var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.commitmentResolution());
            var signal = ConvergenceAnalyser.analyse(state, cg,
                    ConvergencePolicies.composite(
                            ConvergencePolicies.structural(0.95, 10),
                            ConvergencePolicies.commonGroundRatio(0.8, 0.5)),
                    10);
            assertThat(signal.state()).isEqualTo(ConvergenceState.CONSENSUS);
        }
    }

    @Nested
    class JaccardSimilarity {

        @Test
        void identicalTokens_returnsOne() {
            assertThat(ConvergenceAnalyser.jaccardSimilarity(
                    ConvergenceAnalyser.tokenize("hello world"),
                    ConvergenceAnalyser.tokenize("hello world")))
                    .isEqualTo(1.0);
        }

        @Test
        void disjointTokens_returnsZero() {
            assertThat(ConvergenceAnalyser.jaccardSimilarity(
                    ConvergenceAnalyser.tokenize("hello world"),
                    ConvergenceAnalyser.tokenize("foo bar")))
                    .isEqualTo(0.0);
        }

        @Test
        void partialOverlap_returnsFraction() {
            double sim = ConvergenceAnalyser.jaccardSimilarity(
                    ConvergenceAnalyser.tokenize("hello world foo"),
                    ConvergenceAnalyser.tokenize("hello world bar"));
            // intersection=2 (hello, world), union=4 (hello, world, foo, bar)
            assertThat(sim).isEqualTo(0.5);
        }
    }
}
```

- [ ] **Step 5: Run tests, verify all pass**

Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add -A
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "feat(#66): convergence detection — ConvergencePolicy + ConvergenceAnalyser

Post-fold derivation producing ConvergenceSignal (PROGRESSING, CONVERGING,
CONSENSUS, DEADLOCK, DIMINISHING_RETURNS) from ConversationState and
CommonGroundState. Three policies: structural (Jaccard + staleness),
commonGroundRatio, composite. Configurable recentWindow for context
building.

Refs casehubio/blocks#66"
```

---

### Task 4: ConvergenceTermination Bridge

Bridge `ConvergencePolicy` to the platform's `TerminationCondition<T>` SPI.

**Files:**
- Create: `src/main/java/io/casehub/blocks/agentic/termination/ConvergenceTermination.java`
- Create: `src/test/java/io/casehub/blocks/agentic/termination/ConvergenceTerminationTest.java`

**Interfaces:**
- Consumes: `CommonGroundAnalyser.analyse()` from Task 2; `ConvergenceAnalyser.analyse()` from Task 3; `TerminationCondition<T>`, `TerminationContext<T>`, `TerminationDecision` (existing)
- Produces: `ConvergenceTermination<T> implements TerminationCondition<T>` — consumable by `DebateBuilder.convergence()`, `OrchestratedDriver`, `ChoreographedDriver`

- [ ] **Step 1: Create ConvergenceTermination**

```java
package io.casehub.blocks.agentic.termination;

import io.casehub.blocks.conversation.*;
import io.smallrye.mutiny.Uni;

import java.util.Set;
import java.util.function.Function;

public class ConvergenceTermination<T> implements TerminationCondition<T> {

    private final Function<T, ConversationState> stateExtractor;
    private final EpistemicRule epistemicRule;
    private final ConvergencePolicy policy;
    private final int recentWindow;
    private final double confidenceThreshold;
    private final Set<ConvergenceState> terminateOn;

    public ConvergenceTermination(Function<T, ConversationState> stateExtractor,
                                  EpistemicRule epistemicRule,
                                  ConvergencePolicy policy,
                                  int recentWindow,
                                  double confidenceThreshold,
                                  Set<ConvergenceState> terminateOn) {
        this.stateExtractor = stateExtractor;
        this.epistemicRule = epistemicRule;
        this.policy = policy;
        this.recentWindow = recentWindow;
        this.confidenceThreshold = confidenceThreshold;
        this.terminateOn = Set.copyOf(terminateOn);
    }

    @Override
    public Uni<TerminationDecision> evaluate(TerminationContext<T> context) {
        ConversationState state = stateExtractor.apply(context.state());
        CommonGroundState cg = CommonGroundAnalyser.analyse(state, epistemicRule);
        ConvergenceSignal signal = ConvergenceAnalyser.analyse(state, cg, policy, recentWindow);

        if (terminateOn.contains(signal.state())
                && signal.confidence() >= confidenceThreshold) {
            return Uni.createFrom().item(toDecision(signal));
        }
        return Uni.createFrom().item(TerminationDecision.Continue.INSTANCE);
    }

    private TerminationDecision toDecision(ConvergenceSignal signal) {
        return switch (signal.state()) {
            case CONSENSUS -> new TerminationDecision.Complete(signal.reason());
            case DEADLOCK -> new TerminationDecision.Escalate(signal.reason());
            case DIMINISHING_RETURNS -> new TerminationDecision.Complete(signal.reason());
            default -> TerminationDecision.Continue.INSTANCE;
        };
    }
}
```

- [ ] **Step 2: Write ConvergenceTerminationTest**

```java
package io.casehub.blocks.agentic.termination;

import io.casehub.blocks.conversation.*;
import io.casehub.qhorus.api.message.MessageType;
import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class ConvergenceTerminationTest {

    private static final Instant T0 = Instant.parse("2026-07-21T10:00:00Z");

    static ThreadEntry entry(String sender, MessageType type, int round, String content) {
        return new ThreadEntry(null, null, type, sender, T0.plusSeconds(round * 60L),
                sender, round, "ENTRY", content);
    }

    static ConversationPoint point(String id, List<ThreadEntry> thread) {
        return new ConversationPoint(id, "general",
                new PointClassification(Priority.MEDIUM, null, null),
                thread, ConversationProtocol.STATUS_OPEN);
    }

    static ConversationState stateWith(ConversationPoint... points) {
        var map = new LinkedHashMap<String, ConversationPoint>();
        for (var p : points) map.put(p.id(), p);
        return new ConversationState(map, List.of(), List.of(), Map.of());
    }

    @Test
    void consensus_completesConversation() {
        var p1 = point("p1", List.of(
                entry("a", MessageType.COMMAND, 1, "claim"),
                entry("b", MessageType.RESPONSE, 2, "agreed")));
        var state = stateWith(p1);

        var termination = new ConvergenceTermination<>(
                s -> s, EpistemicRules.explicitAcknowledgement(1),
                ConvergencePolicies.structural(0.95, 5),
                10, 0.5, Set.of(ConvergenceState.CONSENSUS));

        var ctx = new TerminationContext<>(state, 1, Duration.ZERO, List.of());
        var decision = termination.evaluate(ctx).await().indefinitely();

        assertThat(decision).isInstanceOf(TerminationDecision.Complete.class);
    }

    @Test
    void deadlock_escalates() {
        var p1 = point("p1", List.of(
                entry("a", MessageType.COMMAND, 1, "claim"),
                entry("b", MessageType.DECLINE, 2, "no")));
        var state = stateWith(p1);

        var termination = new ConvergenceTermination<>(
                s -> s, EpistemicRules.commitmentResolution(),
                ConvergencePolicies.commonGroundRatio(0.8, 0.5),
                10, 0.5, Set.of(ConvergenceState.DEADLOCK));

        var ctx = new TerminationContext<>(state, 1, Duration.ZERO, List.of());
        var decision = termination.evaluate(ctx).await().indefinitely();

        assertThat(decision).isInstanceOf(TerminationDecision.Escalate.class);
    }

    @Test
    void belowConfidenceThreshold_continues() {
        var p1 = point("p1", List.of(
                entry("a", MessageType.COMMAND, 1, "claim"),
                entry("b", MessageType.RESPONSE, 2, "agreed")));
        var state = stateWith(p1);

        var termination = new ConvergenceTermination<>(
                s -> s, EpistemicRules.explicitAcknowledgement(1),
                ConvergencePolicies.structural(0.95, 5),
                10, 0.99, Set.of(ConvergenceState.CONSENSUS));

        var ctx = new TerminationContext<>(state, 1, Duration.ZERO, List.of());
        var decision = termination.evaluate(ctx).await().indefinitely();

        assertThat(decision).isInstanceOf(TerminationDecision.Continue.class);
    }

    @Test
    void stateNotInTerminateSet_continues() {
        var p1 = point("p1", List.of(
                entry("a", MessageType.COMMAND, 1, "claim"),
                entry("b", MessageType.RESPONSE, 2, "agreed")));
        var state = stateWith(p1);

        // Only terminate on DEADLOCK — consensus won't trigger
        var termination = new ConvergenceTermination<>(
                s -> s, EpistemicRules.explicitAcknowledgement(1),
                ConvergencePolicies.structural(0.95, 5),
                10, 0.5, Set.of(ConvergenceState.DEADLOCK));

        var ctx = new TerminationContext<>(state, 1, Duration.ZERO, List.of());
        var decision = termination.evaluate(ctx).await().indefinitely();

        assertThat(decision).isInstanceOf(TerminationDecision.Continue.class);
    }
}
```

- [ ] **Step 3: Run tests, verify all pass**

Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all tests pass.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add -A
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "feat(#66): ConvergenceTermination — bridge to TerminationCondition SPI

Adapter connecting ConvergencePolicy to the platform's TerminationCondition
interface. Configurable: stateExtractor, epistemicRule, policy, recentWindow,
confidenceThreshold, terminateOn states. CONSENSUS→Complete,
DEADLOCK→Escalate, DIMINISHING_RETURNS→Complete.

Refs casehubio/blocks#66"
```

---

### Task 5: RenderContext and Renderer Updates

Replace renderer overloads with `RenderContext` record. Add common ground
badges and convergence header rendering.

**Files:**
- Create: `src/main/java/io/casehub/blocks/conversation/RenderContext.java`
- Modify: `src/main/java/io/casehub/blocks/conversation/ConversationRendererConfig.java`
- Modify: `src/main/java/io/casehub/blocks/conversation/ConversationRenderer.java`
- Modify: `src/test/java/io/casehub/blocks/conversation/ConversationRendererTest.java`
- Modify: `src/test/java/io/casehub/blocks/conversation/TopicAwareConversationIntegrationTest.java`

**Interfaces:**
- Consumes: `CommonGroundState`, `GroundedFact` from Task 2; `ConvergenceSignal`, `ConvergenceState` from Task 3
- Produces: `ConversationRenderer.render(ConversationState, RenderContext)` — used by Task 6

- [ ] **Step 1: Create RenderContext**

```java
package io.casehub.blocks.conversation;

import io.casehub.qhorus.api.message.ReactionGroup;

import java.util.List;
import java.util.Map;

public record RenderContext(
        Map<Long, List<ReactionGroup>> reactions,
        CommonGroundState commonGround,
        ConvergenceSignal convergence) {

    public static final RenderContext EMPTY =
            new RenderContext(Map.of(), null, null);

    public static RenderContext withReactions(Map<Long, List<ReactionGroup>> reactions) {
        return new RenderContext(reactions, null, null);
    }
}
```

- [ ] **Step 2: Add config fields**

Add `showEpistemicStatus` and `showConvergenceSignal` to
`ConversationRendererConfig` — both default `false`. Update the record,
compact constructor, builder, and `toBuilder()`.

- [ ] **Step 3: Update ConversationRenderer**

Replace `render(ConversationState, Map<Long, List<ReactionGroup>>)` with
`render(ConversationState, RenderContext)`. The `render(ConversationState)`
convenience stays, delegating to `render(state, RenderContext.EMPTY)`.

Add convergence header rendering at the top (before topic/flat sections)
when `config.showConvergenceSignal()` and `ctx.convergence() != null`.

Add epistemic badge rendering per point when
`config.showEpistemicStatus()` and `ctx.commonGround() != null`. Look up
each point in the common ground maps and append the badge after the
header.

Add topic-level common ground summary when `groupByTopic` and
`showEpistemicStatus` are both true.

- [ ] **Step 4: Update existing test calls**

In `ConversationRendererTest`, update `reactions_renderedInlineAfterThreadEntries`
and `reactions_noReactionsForMessage_noDecoration` to use
`RenderContext.withReactions(reactions)` instead of passing the map directly.

In `TopicAwareConversationIntegrationTest`, update the
`fullChoreography_multiTopicWithObligationChainsAndReactions` test to use
`RenderContext.withReactions(reactions)`.

- [ ] **Step 5: Add epistemic rendering tests**

In `ConversationRendererTest`:

```java
@Test
void epistemicBadges_renderedPerPoint() {
    var thread = List.of(entry("REV", "RAISE", "a claim"));
    var points = Map.of("p1", point("p1", Priority.HIGH, "test", null, "OPEN", thread));
    var config = reviewConfig().toBuilder()
            .showEpistemicStatus(true).build();
    var renderer = new ConversationRenderer(config);
    var cg = new CommonGroundState(
            Map.of("p1", new GroundedFact("p1", "general", EpistemicStatus.ESTABLISHED,
                    "a claim", java.util.Set.of("agent-a"), java.util.Set.of(), 1)),
            Map.of(), Map.of());
    var ctx = new RenderContext(Map.of(), cg, null);
    var result = renderer.render(state(points, List.of(), List.of(), Map.of()), ctx);
    assertThat(result).contains("[established by agent-a]");
}

@Test
void convergenceHeader_renderedAtTop() {
    var config = reviewConfig().toBuilder()
            .showConvergenceSignal(true).build();
    var renderer = new ConversationRenderer(config);
    var signal = new ConvergenceSignal(ConvergenceState.CONVERGING, 0.78,
            "common ground growing, 1 disputed point remaining");
    var ctx = new RenderContext(Map.of(), null, signal);
    var result = renderer.render(
            state(Map.of(), List.of(), List.of(), Map.of()), ctx);
    assertThat(result).contains("**Convergence:** CONVERGING (0.78)");
}
```

- [ ] **Step 6: Run tests, verify all pass**

Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add -A
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "feat(#65,#66): RenderContext + epistemic badges + convergence header

Replace renderer overloads with RenderContext record. Add per-point
epistemic status badges and conversation-level convergence signal header.
Both opt-in via ConversationRendererConfig flags (default false).

Refs casehubio/blocks#65, casehubio/blocks#66"
```

---

### Task 6: Integration Test

Full pipeline: fold messages → derive common ground → detect convergence
→ render. Three scenarios testing the end-to-end chain.

**Files:**
- Create: `src/test/java/io/casehub/blocks/conversation/CommonGroundConvergenceIntegrationTest.java`

**Interfaces:**
- Consumes: All from Tasks 1–5

- [ ] **Step 1: Write integration test**

```java
package io.casehub.blocks.conversation;

import io.casehub.blocks.channel.ChannelMessageMeta;
import io.casehub.qhorus.api.message.MessageType;
import io.casehub.qhorus.api.message.MessageView;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.when;

class CommonGroundConvergenceIntegrationTest {

    private final AtomicLong messageIdSeq = new AtomicLong(1);
    private final Instant base = Instant.parse("2026-07-21T10:00:00Z");
    private final ConversationProjectionTest.TestConversationProjection projection =
            new ConversationProjectionTest.TestConversationProjection();

    private MessageView message(String content, String correlationId, String topic,
                                 MessageType type, String sender, int minuteOffset) {
        var msg = mock(MessageView.class);
        when(msg.content()).thenReturn(content);
        when(msg.correlationId()).thenReturn(correlationId);
        when(msg.topic()).thenReturn(topic);
        when(msg.id()).thenReturn(messageIdSeq.getAndIncrement());
        when(msg.type()).thenReturn(type);
        when(msg.sender()).thenReturn(sender);
        when(msg.createdAt()).thenReturn(base.plusSeconds(minuteOffset * 60L));
        return msg;
    }

    private String encode(Map<String, String> meta, String body) {
        return ChannelMessageMeta.encode("TEST:", meta, body);
    }

    private Map<String, String> meta(String... pairs) {
        var map = new LinkedHashMap<String, String>();
        for (int i = 0; i < pairs.length; i += 2) map.put(pairs[i], pairs[i + 1]);
        return map;
    }

    @Test
    void consensusScenario_debateReachesAgreement() {
        var state = projection.identity();

        // Round 1: two agents raise points
        state = projection.apply(state, message(encode(meta(
                "entryType", "OPEN_TOPIC", "role", "AGENT_A", "round", "1",
                "priority", "HIGH", "scope", "design"),
                "Use event sourcing"), "p1", "architecture", MessageType.COMMAND, "agent-a", 1));
        state = projection.apply(state, message(encode(meta(
                "entryType", "OPEN_TOPIC", "role", "AGENT_B", "round", "1",
                "priority", "MEDIUM", "scope", "design"),
                "Use CQRS with projections"), "p2", "architecture", MessageType.COMMAND, "agent-b", 2));

        // Round 2: both acknowledge each other
        state = projection.apply(state, message(encode(meta(
                "entryType", "ACCEPT", "role", "AGENT_B", "round", "2"),
                "Event sourcing works"), "p1", "architecture", MessageType.RESPONSE, "agent-b", 3));
        state = projection.apply(state, message(encode(meta(
                "entryType", "ACCEPT", "role", "AGENT_A", "round", "2"),
                "Projections are the way"), "p2", "architecture", MessageType.RESPONSE, "agent-a", 4));

        // Derive common ground
        var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
        assertThat(cg.establishedFacts()).hasSize(2);
        assertThat(cg.pendingClaims()).isEmpty();

        // Detect convergence
        var signal = ConvergenceAnalyser.analyse(state, cg,
                ConvergencePolicies.structural(0.8, 3), 10);
        assertThat(signal.state()).isEqualTo(ConvergenceState.CONSENSUS);

        // Render
        var config = ConversationRendererConfig.builder()
                .groupByTopic(true)
                .showEpistemicStatus(true)
                .showConvergenceSignal(true)
                .build();
        var renderer = new ConversationRenderer(config);
        var ctx = new RenderContext(Map.of(), cg, signal);
        var result = renderer.render(state, ctx);

        assertThat(result).contains("**Convergence:** CONSENSUS");
        assertThat(result).contains("[established by agent-b]");
    }

    @Test
    void deadlockScenario_positionsHarden() {
        var state = projection.identity();

        // Agent A raises, Agent B declines
        state = projection.apply(state, message(encode(meta(
                "entryType", "OPEN_TOPIC", "role", "AGENT_A", "round", "1",
                "priority", "HIGH", "scope", "design"),
                "Use microservices"), "p1", "architecture", MessageType.COMMAND, "agent-a", 1));
        state = projection.apply(state, message(encode(meta(
                "entryType", "REJECT", "role", "AGENT_B", "round", "2"),
                "Monolith is better"), "p1", "architecture", MessageType.DECLINE, "agent-b", 2));

        // Derive
        var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
        assertThat(cg.disputedPoints()).hasSize(1);

        var signal = ConvergenceAnalyser.analyse(state, cg,
                ConvergencePolicies.commonGroundRatio(0.8, 0.5), 10);
        assertThat(signal.state()).isEqualTo(ConvergenceState.DEADLOCK);
    }

    @Test
    void diminishingReturnsScenario_conversationGoesThin() {
        var state = projection.identity();

        // Early: substantive points
        state = projection.apply(state, message(encode(meta(
                "entryType", "OPEN_TOPIC", "role", "AGENT_A", "round", "1",
                "priority", "HIGH", "scope", "design"),
                "This is a detailed analysis of the system architecture with full context"),
                "p1", "design", MessageType.COMMAND, "agent-a", 1));

        // Late: thin status messages, no new points, high round
        state = projection.apply(state, message(encode(meta(
                "entryType", "ACCEPT", "role", "AGENT_B", "round", "8"),
                "ok"), "p1", "design", MessageType.STATUS, "agent-b", 10));

        var cg = CommonGroundAnalyser.analyse(state, EpistemicRules.explicitAcknowledgement(1));
        var signal = ConvergenceAnalyser.analyse(state, cg,
                ConvergencePolicies.structural(0.8, 3), 10);
        assertThat(signal.state()).isEqualTo(ConvergenceState.DIMINISHING_RETURNS);
    }
}
```

- [ ] **Step 2: Run all tests**

Run: `mvn --batch-mode -f /Users/mdproctor/claude/casehub/blocks/pom.xml test`
Expected: all tests pass including integration tests.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add -A
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "test(#65,#66): integration test — fold → common ground → convergence → render

Three end-to-end scenarios: consensus (debate reaches agreement),
deadlock (positions harden), diminishing returns (conversation goes thin).
Full pipeline through ConversationProjection, CommonGroundAnalyser,
ConvergenceAnalyser, and ConversationRenderer.

Refs casehubio/blocks#65, casehubio/blocks#66"
```

---

### Task 7: CLAUDE.md and Spec Updates

Update CLAUDE.md to document the new types and update the spec with any
implementation-time decisions.

**Files:**
- Modify: `CLAUDE.md` (Package tables)

- [ ] **Step 1: Update CLAUDE.md package documentation**

Add the new types to the `## Package: io.casehub.blocks.conversation`
table.

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/worktrees/24/blocks commit -m "docs(#65,#66): update CLAUDE.md with common ground and convergence types

Refs casehubio/blocks#65, casehubio/blocks#66"
```
