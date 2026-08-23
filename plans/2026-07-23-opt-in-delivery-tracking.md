# Opt-in Delivery Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #376 — feat: opt-in delivery/read tracking for BARRIER and COLLECT channels
**Issue group:** #376

**Goal:** Add per-participant delivery cursor (`lastDeliveredMessageId`) to channel membership, advanced by each transport when delivery is confirmed, with watchdog enrichments for BARRIER_STUCK and CONVERSATION_STALL.

**Architecture:** Extends the existing `channel_membership` table with a forward-only `lastDeliveredMessageId` cursor alongside `lastReadMessageId`. Each transport (MCP pull, AT_LEAST_ONCE push, BEST_EFFORT push, observers) advances the cursor at the point delivery is confirmed. Opt-in by channel semantic: BARRIER/COLLECT default on, others off. `channel.trackDelivery` (Boolean, nullable) overrides the default.

**Tech Stack:** Java 21, Quarkus 3.32.2, H2 (tests), Flyway (migrations)

## Global Constraints

- **Reactive parity deferred:** `issue-384-retire-reactive` is in progress. No new reactive store methods or reactive MCP tool paths. Blocking stack only.
- **Migrations:** V40 (`channel.track_delivery`), V41 (`channel_membership.last_delivered_message_id`). V36-V39 are occupied on main.
- **OutboundMessage.sequenceId:** New `Long sequenceId` field on the `OutboundMessage` record. ~150 construction sites across the project. Added with a backward-compatible constructor (old 9-arg → delegates to new 10-arg with `sequenceId=null`). New code passes the sequence ID; old code passes null (delivery tracking skips null sequence IDs).
- **Breaking record changes:** `BarrierStuckContext` and `ConversationStallContext` gain new fields. All call sites must be updated. Pre-release — breaking changes cost nothing.
- **Forward-only cursor:** `updateLastDeliveredMessageId` and `advanceDeliveredCursorForMembers` only advance if `messageId > current`. Never regress.
- **Guard pattern:** Before any cursor advancement, check `isDeliveryTrackingEnabled(channel)`. If false, no store call.

---

### Task 1: Data Model Foundations

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/Channel.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelCreateRequest.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelMembership.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/channel/ChannelDetail.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/gateway/OutboundMessage.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelEntity.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelMembershipEntity.java`
- Create: `runtime/src/main/resources/db/qhorus/migration/V40__channel_track_delivery.sql`
- Create: `runtime/src/main/resources/db/qhorus/migration/V41__membership_last_delivered_message_id.sql`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/QhorusEntityMapper.java` (toChannelDetail gains trackDelivery)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` (isDeliveryTrackingEnabled helper)
- Test: `api/src/test/java/io/casehub/qhorus/api/DomainRecordTest.java` (update record construction)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/channel/DeliveryTrackingResolutionTest.java` (new — CDI-free)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/conversion/SmallEntityConversionTest.java` (update entity round-trip)

**Interfaces:**
- Consumes: Nothing — foundational task
- Produces:
  - `Channel.trackDelivery()` → `Boolean` (nullable)
  - `ChannelMembership.lastDeliveredMessageId()` → `Long` (nullable)
  - `OutboundMessage.sequenceId()` → `Long` (nullable)
  - `ChannelService.isDeliveryTrackingEnabled(Channel)` → `boolean`
  - `ChannelDetail.trackDelivery()` → `Boolean`
  - Backward-compatible constructors on `ChannelMembership` (7-arg → 8-arg) and `OutboundMessage` (9-arg → 10-arg)

- [ ] **Step 1: Write failing test — delivery tracking resolution**

Create `DeliveryTrackingResolutionTest.java` in `runtime/src/test/java/.../runtime/channel/`:

```java
package io.casehub.qhorus.runtime.channel;

import io.casehub.qhorus.api.channel.Channel;
import io.casehub.qhorus.api.channel.ChannelSemantic;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.EnumSource;

import static org.junit.jupiter.api.Assertions.*;

class DeliveryTrackingResolutionTest {

    @Test
    void barrier_defaultsToEnabled() {
        Channel ch = Channel.builder("barrier-ch").semantic(ChannelSemantic.BARRIER).build();
        assertTrue(ChannelService.isDeliveryTrackingEnabled(ch));
    }

    @Test
    void collect_defaultsToEnabled() {
        Channel ch = Channel.builder("collect-ch").semantic(ChannelSemantic.COLLECT).build();
        assertTrue(ChannelService.isDeliveryTrackingEnabled(ch));
    }

    @ParameterizedTest
    @EnumSource(value = ChannelSemantic.class, names = {"APPEND", "EPHEMERAL", "LAST_WRITE"})
    void nonCoordination_defaultsToDisabled(ChannelSemantic semantic) {
        Channel ch = Channel.builder("ch").semantic(semantic).build();
        assertFalse(ChannelService.isDeliveryTrackingEnabled(ch));
    }

    @Test
    void explicitTrue_overridesDefault() {
        Channel ch = Channel.builder("ch").semantic(ChannelSemantic.APPEND).trackDelivery(true).build();
        assertTrue(ChannelService.isDeliveryTrackingEnabled(ch));
    }

    @Test
    void explicitFalse_overridesDefault() {
        Channel ch = Channel.builder("ch").semantic(ChannelSemantic.BARRIER).trackDelivery(false).build();
        assertFalse(ChannelService.isDeliveryTrackingEnabled(ch));
    }

    @Test
    void nullTrackDelivery_usesSemanticDefault() {
        Channel ch = Channel.builder("ch").semantic(ChannelSemantic.BARRIER).trackDelivery(null).build();
        assertTrue(ChannelService.isDeliveryTrackingEnabled(ch));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryTrackingResolutionTest -pl runtime -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: FAIL — `Channel` has no `trackDelivery` field, `ChannelService.isDeliveryTrackingEnabled` does not exist.

- [ ] **Step 3: Add `trackDelivery` to Channel record**

Add `Boolean trackDelivery` field to `Channel` record (after `protocolParticipants`, before `tenancyId`). Add to the canonical constructor's compact constructor (no normalization — null is preserved). Add to `Builder` with setter. Add backward-compatible constructors that pass `null` for `trackDelivery`. Update `Channel.fromRequest()` to pass `req.trackDelivery()`. Update `Channel.toBuilder()`.

- [ ] **Step 4: Add `trackDelivery` to ChannelCreateRequest**

Add `Boolean trackDelivery` field to `ChannelCreateRequest` record. Add to compact constructor (no normalization). Add `trackDelivery(Boolean)` to Builder. Add backward-compatible constructors. Update `build()`.

- [ ] **Step 5: Add `lastDeliveredMessageId` to ChannelMembership**

Add `Long lastDeliveredMessageId` field to `ChannelMembership` record. Add backward-compatible 7-arg constructor that delegates with `null`.

- [ ] **Step 6: Add `sequenceId` to OutboundMessage**

Add `Long sequenceId` field to `OutboundMessage` record (after `messageId`). Add backward-compatible 9-arg constructor that delegates with `sequenceId=null`. This avoids updating all ~150 existing construction sites immediately — they continue to compile with null sequenceId.

- [ ] **Step 7: Add `trackDelivery` to ChannelDetail**

Add `Boolean trackDelivery` field to `ChannelDetail` record (after `protocolParticipants`, before `connectorBinding`). Add backward-compatible constructor without it. Update `QhorusEntityMapper.toChannelDetail()` to pass the resolved (effective) tracking state — `ChannelService.isDeliveryTrackingEnabled(ch)`.

- [ ] **Step 8: Update ChannelEntity**

Add `public Boolean trackDelivery` field with `@Column(name = "track_delivery")`. Update `fromDomain()` and `toDomain()`.

- [ ] **Step 9: Update ChannelMembershipEntity**

Add `public Long lastDeliveredMessageId` field with `@Column(name = "last_delivered_message_id")`. Update `fromDomain()` and `toDomain()`.

- [ ] **Step 10: Add isDeliveryTrackingEnabled to ChannelService**

Add static method:

```java
public static boolean isDeliveryTrackingEnabled(Channel ch) {
    if (ch.trackDelivery() != null) return ch.trackDelivery();
    return ch.semantic() == ChannelSemantic.BARRIER
        || ch.semantic() == ChannelSemantic.COLLECT;
}
```

- [ ] **Step 11: Create migrations**

V40:
```sql
ALTER TABLE channel ADD COLUMN track_delivery BOOLEAN;
```

V41:
```sql
ALTER TABLE channel_membership ADD COLUMN last_delivered_message_id BIGINT;
```

- [ ] **Step 12: Fix all compilation errors**

Update `DomainRecordTest`, `SmallEntityConversionTest`, and any other tests that construct `Channel`, `ChannelMembership`, `ChannelDetail`, or `OutboundMessage` records directly. Use `ide_diagnostics` to find all compilation errors. Fix each with `ide_edit_member` or `ide_replace_member`.

- [ ] **Step 13: Run tests to verify resolution helper passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryTrackingResolutionTest -pl runtime -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 14: Run full build to verify no compilation errors**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: BUILD SUCCESS — all modules compile, all tests pass.

- [ ] **Step 15: Commit**

```
feat(#376): data model — trackDelivery, lastDeliveredMessageId, OutboundMessage.sequenceId

Add Channel.trackDelivery (Boolean, nullable) with semantic defaults:
BARRIER/COLLECT → on, APPEND/EPHEMERAL/LAST_WRITE → off.
Add ChannelMembership.lastDeliveredMessageId cursor.
Add OutboundMessage.sequenceId for cursor advancement.
V40 + V41 migrations. Backward-compatible constructors.

Refs #376
```

---

### Task 2: Store Layer

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/store/ChannelMembershipStore.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/store/jpa/JpaChannelMembershipStore.java`
- Modify: `persistence-memory/src/main/java/io/casehub/qhorus/persistence/memory/InMemoryChannelMembershipStore.java`
- Modify: `persistence-memory/src/test/java/io/casehub/qhorus/persistence/memory/contract/ChannelMembershipStoreContractTest.java`

**Interfaces:**
- Consumes: `ChannelMembership.lastDeliveredMessageId()` from Task 1
- Produces:
  - `ChannelMembershipStore.updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId)` → `void`
  - `ChannelMembershipStore.advanceDeliveredCursorForMembers(UUID channelId, Set<String> memberIds, Long messageId)` → `void`

- [ ] **Step 1: Write failing contract tests**

Add to `ChannelMembershipStoreContractTest`:

```java
@Test
void updateLastDeliveredMessageId_advancesForward() {
    var m = store().put(membership(channelId, "agent-a", MemberRole.PARTICIPANT));
    store().updateLastDeliveredMessageId(channelId, "agent-a", 10L);
    assertEquals(10L, store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
}

@Test
void updateLastDeliveredMessageId_forwardOnly_lowerIdIsNoOp() {
    store().put(membership(channelId, "agent-a", MemberRole.PARTICIPANT));
    store().updateLastDeliveredMessageId(channelId, "agent-a", 10L);
    store().updateLastDeliveredMessageId(channelId, "agent-a", 5L);
    assertEquals(10L, store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
}

@Test
void updateLastDeliveredMessageId_nullToFirstValue() {
    store().put(membership(channelId, "agent-a", MemberRole.PARTICIPANT));
    assertNull(store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
    store().updateLastDeliveredMessageId(channelId, "agent-a", 1L);
    assertEquals(1L, store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
}

@Test
void updateLastDeliveredMessageId_idempotent() {
    store().put(membership(channelId, "agent-a", MemberRole.PARTICIPANT));
    store().updateLastDeliveredMessageId(channelId, "agent-a", 10L);
    store().updateLastDeliveredMessageId(channelId, "agent-a", 10L);
    assertEquals(10L, store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
}

@Test
void advanceDeliveredCursorForMembers_advancesSpecifiedMembers() {
    store().put(membership(channelId, "agent-a", MemberRole.PARTICIPANT));
    store().put(membership(channelId, "agent-b", MemberRole.PARTICIPANT));
    store().put(membership(channelId, "agent-c", MemberRole.PARTICIPANT));
    store().advanceDeliveredCursorForMembers(channelId, Set.of("agent-a", "agent-b"), 10L);
    assertEquals(10L, store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
    assertEquals(10L, store().find(channelId, "agent-b").orElseThrow().lastDeliveredMessageId());
    assertNull(store().find(channelId, "agent-c").orElseThrow().lastDeliveredMessageId());
}

@Test
void advanceDeliveredCursorForMembers_forwardOnly() {
    store().put(membership(channelId, "agent-a", MemberRole.PARTICIPANT));
    store().advanceDeliveredCursorForMembers(channelId, Set.of("agent-a"), 10L);
    store().advanceDeliveredCursorForMembers(channelId, Set.of("agent-a"), 5L);
    assertEquals(10L, store().find(channelId, "agent-a").orElseThrow().lastDeliveredMessageId());
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryChannelMembershipStoreTest -pl persistence-memory -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: FAIL — methods don't exist yet.

- [ ] **Step 3: Add methods to ChannelMembershipStore interface**

```java
void updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId);
void advanceDeliveredCursorForMembers(UUID channelId, Set<String> memberIds, Long messageId);
```

- [ ] **Step 4: Implement in InMemoryChannelMembershipStore**

```java
@Override
public void updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId) {
    find(channelId, memberId).ifPresent(m -> {
        if (m.lastDeliveredMessageId() == null || messageId > m.lastDeliveredMessageId()) {
            var updated = new ChannelMembership(m.id(), m.channelId(), m.memberId(), m.role(),
                    m.tenancyId(), m.joinedAt(), m.lastReadMessageId(), messageId);
            members.put(key(channelId, memberId), updated);
        }
    });
}

@Override
public void advanceDeliveredCursorForMembers(UUID channelId, Set<String> memberIds, Long messageId) {
    for (String memberId : memberIds) {
        updateLastDeliveredMessageId(channelId, memberId, messageId);
    }
}
```

- [ ] **Step 5: Implement in JpaChannelMembershipStore**

```java
@Override
@Transactional
public void updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId) {
    repo.update("lastDeliveredMessageId = ?1 where channelId = ?2 and memberId = ?3 "
            + "and (lastDeliveredMessageId is null or lastDeliveredMessageId < ?1)",
            messageId, channelId, memberId);
}

@Override
@Transactional
public void advanceDeliveredCursorForMembers(UUID channelId, Set<String> memberIds, Long messageId) {
    if (memberIds.isEmpty()) return;
    repo.update("lastDeliveredMessageId = ?1 where channelId = ?2 and memberId in ?3 "
            + "and (lastDeliveredMessageId is null or lastDeliveredMessageId < ?1)",
            messageId, channelId, memberIds);
}
```

- [ ] **Step 6: Run contract tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=InMemoryChannelMembershipStoreTest -pl persistence-memory -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```
feat(#376): store layer — delivery cursor advancement methods

Add updateLastDeliveredMessageId and advanceDeliveredCursorForMembers
to ChannelMembershipStore with forward-only semantics. JPA and InMemory
implementations. Contract tests for forward-only, null-to-first,
idempotent, and batch advancement.

Refs #376
```

---

### Task 3: Pull Path — check_messages and wait_for_reply Delivery Advancement

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` (checkMessages variants + waitForReply)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` (isDeliveryTrackingEnabled access)
- Test: `runtime/src/test/java/io/casehub/qhorus/mcp/DeliveryTrackingPullTest.java` (new)

**Interfaces:**
- Consumes: `ChannelService.isDeliveryTrackingEnabled(Channel)`, `ChannelMembershipStore.updateLastDeliveredMessageId(UUID, String, Long)` from Tasks 1-2
- Produces: Side effect — `check_messages` and `wait_for_reply` advance delivery cursor when tracking is enabled

- [ ] **Step 1: Write failing integration test**

Create `DeliveryTrackingPullTest.java`:

```java
@QuarkusTest
class DeliveryTrackingPullTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelMembershipStore membershipStore;
    @Inject ChannelService channelService;

    @Test
    @TestTransaction
    void checkMessages_barrierChannel_advancesDeliveryCursor() {
        // Create a BARRIER channel (tracking enabled by default)
        var ch = channelService.create(ChannelCreateRequest.builder("barrier-dt-test")
                .semantic(ChannelSemantic.BARRIER)
                .barrierContributors(List.of("agent-a", "agent-b"))
                .build());
        // Register reader and join
        tools.register("reader-1", "test reader", null, null, null, null, null, null);
        membershipStore.put(new ChannelMembership(null, ch.id(), "reader-1",
                MemberRole.PARTICIPANT, null, Instant.now(), null));
        // Send a message
        tools.sendMessage("barrier-dt-test", "agent-a", "STATUS", "contribution", null, null, null, null, null, null, null, null, null);
        // check_messages with reader_instance_id
        tools.checkMessages("barrier-dt-test", 0L, null, null, "reader-1", null);
        // Verify cursor advanced
        var membership = membershipStore.find(ch.id(), "reader-1").orElseThrow();
        assertNotNull(membership.lastDeliveredMessageId());
        assertTrue(membership.lastDeliveredMessageId() > 0);
    }

    @Test
    @TestTransaction
    void checkMessages_appendChannel_doesNotAdvanceCursor() {
        var ch = channelService.create(ChannelCreateRequest.builder("append-dt-test")
                .semantic(ChannelSemantic.APPEND).build());
        tools.register("reader-2", "test reader", null, null, null, null, null, null);
        membershipStore.put(new ChannelMembership(null, ch.id(), "reader-2",
                MemberRole.PARTICIPANT, null, Instant.now(), null));
        tools.sendMessage("append-dt-test", "agent-a", "STATUS", "hello", null, null, null, null, null, null, null, null, null);
        tools.checkMessages("append-dt-test", 0L, null, null, "reader-2", null);
        var membership = membershipStore.find(ch.id(), "reader-2").orElseThrow();
        assertNull(membership.lastDeliveredMessageId());
    }

    @Test
    @TestTransaction
    void checkMessages_noReaderInstanceId_doesNotAdvance() {
        var ch = channelService.create(ChannelCreateRequest.builder("barrier-no-reader")
                .semantic(ChannelSemantic.BARRIER)
                .barrierContributors(List.of("agent-a"))
                .build());
        tools.sendMessage("barrier-no-reader", "agent-a", "STATUS", "data", null, null, null, null, null, null, null, null, null);
        tools.checkMessages("barrier-no-reader", 0L, null, null, null, null);
        // No membership to check — no reader instance joined
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryTrackingPullTest -pl runtime -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: FAIL — cursor not advanced.

- [ ] **Step 3: Implement cursor advancement in checkMessages**

Add a private helper method to `QhorusMcpTools`:

```java
private void advanceDeliveryCursorIfTracked(Channel ch, String readerInstanceId, Long lastId) {
    if (lastId == null || lastId <= 0) return;
    if (readerInstanceId == null || readerInstanceId.isBlank()) return;
    if (!ChannelService.isDeliveryTrackingEnabled(ch)) return;
    membershipStore.updateLastDeliveredMessageId(ch.id(), readerInstanceId, lastId);
}
```

Inject `ChannelMembershipStore` into `QhorusMcpTools`. Call `advanceDeliveryCursorIfTracked(ch, readerInstanceId, lastId)` before the return statement in each `checkMessages*` variant.

For BARRIER and COLLECT: call BEFORE `messageStore.deleteNonEvent()`.

- [ ] **Step 4: Implement cursor advancement in waitForReply**

In `waitForReply`, after finding a matching RESPONSE/DONE message and before returning, call:

```java
advanceDeliveryCursorIfTracked(ch, callerInstanceId, matchedMessage.id());
```

The caller's instance ID is available from the method parameters.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryTrackingPullTest -pl runtime -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(#376): pull path — check_messages and wait_for_reply advance delivery cursor

check_messages advances lastDeliveredMessageId for the calling agent
when delivery tracking is enabled. Same for wait_for_reply. BARRIER/
COLLECT cursor advancement happens before message deletion.

Refs #376
```

---

### Task 4: Push Path — DeliveryBatchExecutor (AT_LEAST_ONCE)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutor.java` (`toOutbound` gains sequenceId)
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/gateway/DeliveryBatchExecutorDeliveryTrackingTest.java` (new)

**Interfaces:**
- Consumes: `ChannelService.isDeliveryTrackingEnabled(Channel)`, `ChannelMembershipStore.updateLastDeliveredMessageId/advanceDeliveredCursorForMembers`, `OutboundMessage.sequenceId()` from Tasks 1-2
- Produces: Side effect — `deliverBatch()` advances delivery cursor after successful `post()` for AT_LEAST_ONCE backends

- [ ] **Step 1: Write failing test**

Create `DeliveryBatchExecutorDeliveryTrackingTest.java` (CDI-free unit test):

```java
class DeliveryBatchExecutorDeliveryTrackingTest {

    // Test that deliverBatch advances the delivery cursor after successful post
    // for a channel with delivery tracking enabled.
    // Uses InMemory stores, a recording backend, and a Channel with BARRIER semantic.

    @Test
    void deliverBatch_trackedChannel_advancesCursor() {
        // Setup: BARRIER channel, one member, one message, AT_LEAST_ONCE backend
        // After deliverBatch succeeds, verify membershipStore cursor advanced
    }

    @Test
    void deliverBatch_untrackedChannel_doesNotAdvanceCursor() {
        // Setup: APPEND channel, AT_LEAST_ONCE backend
        // After deliverBatch succeeds, verify cursor NOT advanced
    }
}
```

Full test setup uses `InMemoryChannelMembershipStore`, `InMemoryMessageStore`, `InMemoryCrossTenantMessageStore`, `InMemoryChannelStore`, `InMemoryDeliveryCursorStore`, and a `RecordingChannelBackend(backendId, ActorType.HUMAN, DeliveryGuarantee.AT_LEAST_ONCE)`.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryBatchExecutorDeliveryTrackingTest -pl runtime -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`

- [ ] **Step 3: Update toOutbound to pass sequenceId**

In `DeliveryBatchExecutor.toOutbound(Message m)`:

```java
static OutboundMessage toOutbound(Message m) {
    return new OutboundMessage(
            m.messageId(), m.id(),  // sequenceId = Message.id() (Long)
            m.sender(), m.messageType(), m.content(),
            m.correlationId(), m.inReplyTo(),
            ActorTypeResolver.resolve(m.sender()),
            m.artefactRefs(), m.target());
}
```

- [ ] **Step 4: Add delivery cursor advancement to deliverBatch**

After `backend.post(ref, outbound)` succeeds in the delivery loop, add:

```java
if (outbound.sequenceId() != null && channel != null
        && ChannelService.isDeliveryTrackingEnabled(channel)) {
    Set<String> memberIds = channelMembershipStore.findByChannel(channelId).stream()
            .filter(m -> ActorTypeResolver.resolve(m.memberId()) == backend.actorType())
            .map(ChannelMembership::memberId)
            .collect(Collectors.toSet());
    if (!memberIds.isEmpty()) {
        channelMembershipStore.advanceDeliveredCursorForMembers(channelId, memberIds, outbound.sequenceId());
    }
}
```

Inject `ChannelMembershipStore` and `CrossTenantChannelStore` (for channel lookup) into `DeliveryBatchExecutor`.

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=DeliveryBatchExecutorDeliveryTrackingTest -pl runtime -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: PASS

- [ ] **Step 6: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```
feat(#376): push path — DeliveryBatchExecutor advances delivery cursor for AT_LEAST_ONCE backends

After successful post() in deliverBatch(), advances
lastDeliveredMessageId for members whose actorType matches the
backend. toOutbound() now passes Message.id() as sequenceId.

Refs #376
```

---

### Task 5: Push Path — A2A SSE (BEST_EFFORT)

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/api/A2AResource.java` (`streamTask` gains cursor advancement)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/message/MessageService.java` (pass sequenceId in OutboundMessage construction)
- Test: `runtime/src/test/java/io/casehub/qhorus/api/A2ADeliveryTrackingTest.java` (new)

**Interfaces:**
- Consumes: `ChannelService.isDeliveryTrackingEnabled(Channel)`, `ChannelMembershipStore.updateLastDeliveredMessageId`, `OutboundMessage.sequenceId()` from Tasks 1-2
- Produces: Side effect — A2A SSE stream advances delivery cursor after each `sink.send()` of a non-keepalive message

- [ ] **Step 1: Write failing test**

Create `A2ADeliveryTrackingTest.java`. This is an integration test that verifies the SSE stream advances the delivery cursor. Uses the A2A send/stream test pattern from existing `A2ASendMessageTest`.

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Pass sequenceId in MessageService.dispatch()**

In `MessageService.dispatch()`, update the `OutboundMessage` construction to pass `savedMessage.id()` as `sequenceId`:

```java
new OutboundMessage(savedMessage.messageId(), savedMessage.id(), ...)
```

There are two `OutboundMessage` constructions in `dispatch()` — update both.

- [ ] **Step 4: Add cursor advancement to A2AResource.streamTask()**

After each successful `sink.send()` of a non-keepalive message in `streamTask()`:

```java
if (msg.sequenceId() != null && trackingEnabled) {
    QuarkusTransaction.requiringNew().run(() ->
        channelMembershipStore.updateLastDeliveredMessageId(
            channelId, consumerMemberId, msg.sequenceId()));
}
```

Inject `ChannelMembershipStore` and `ChannelService` into `A2AResource`. Resolve `consumerMemberId` from the task context (the sender of the initial message — already available from the message history read).

- [ ] **Step 5: Run tests**

- [ ] **Step 6: Run full build**

- [ ] **Step 7: Commit**

```
feat(#376): push path — A2A SSE advances delivery cursor after sink.send()

streamTask() advances lastDeliveredMessageId for the SSE consumer
after each non-keepalive message. Uses QuarkusTransaction.requiringNew()
on @RunOnVirtualThread (proven pattern). MessageService.dispatch()
now passes Message.id() as OutboundMessage.sequenceId.

Refs #376
```

---

### Task 6: Watchdog Enrichments

**Files:**
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/BarrierStuckContext.java`
- Modify: `api/src/main/java/io/casehub/qhorus/api/watchdog/ConversationStallContext.java`
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/watchdog/WatchdogEvaluationService.java`
- Test: `runtime/src/test/java/io/casehub/qhorus/runtime/watchdog/WatchdogDeliveryTrackingTest.java` (new)
- Modify: `runtime/src/test/java/io/casehub/qhorus/mcp/WatchdogEnabledTest.java` (update existing BarrierStuckContext constructions)

**Interfaces:**
- Consumes: `ChannelMembershipStore.find(UUID, String)`, `ChannelService.isDeliveryTrackingEnabled(Channel)` from Tasks 1-2
- Produces:
  - `BarrierStuckContext.notDelivered()` → `List<String>`
  - `BarrierStuckContext.deliveredNoResponse()` → `List<String>`
  - `ConversationStallContext.deliveryConfirmed()` → `Boolean` (nullable)

- [ ] **Step 1: Write failing test**

Create `WatchdogDeliveryTrackingTest.java`:

```java
class WatchdogDeliveryTrackingTest {
    // CDI-free unit test

    @Test
    void barrierStuck_splitsNotDeliveredFromDeliveredNoResponse() {
        // Setup: BARRIER channel, 3 contributors, 2 have membership records
        // contributor-a: lastDeliveredMessageId = 10 (covers the command), no contribution → deliveredNoResponse
        // contributor-b: lastDeliveredMessageId = null → notDelivered
        // contributor-c: no membership record → notDelivered
    }

    @Test
    void conversationStall_deliveryConfirmedTrue_whenCursorPastCommand() {
        // Stalled commitment where obligor's lastDeliveredMessageId > command's message ID
    }

    @Test
    void conversationStall_deliveryConfirmedFalse_whenCursorBehindCommand() {
        // Stalled commitment where obligor's lastDeliveredMessageId < command's message ID
    }

    @Test
    void conversationStall_deliveryConfirmedNull_whenTrackingDisabled() {
        // APPEND channel (tracking disabled) — deliveryConfirmed should be null
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Update BarrierStuckContext**

```java
public record BarrierStuckContext(
        UUID channelId, String channelName,
        List<String> missingContributors,
        List<String> notDelivered,
        List<String> deliveredNoResponse,
        long elapsedSeconds) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.BARRIER_STUCK; }
}
```

- [ ] **Step 4: Update ConversationStallContext**

```java
public record ConversationStallContext(
        UUID channelId, String channelName,
        int stalledCount, List<String> correlationIds,
        long stalledSeconds,
        Boolean deliveryConfirmed) implements AlertContext {
    @Override
    public WatchdogConditionType conditionType() { return WatchdogConditionType.CONVERSATION_STALL; }
}
```

- [ ] **Step 5: Update evaluateBarrierStuck**

Inject `ChannelMembershipStore` into `WatchdogEvaluationService` (uses `CrossTenant` variant per protocol). After identifying missing contributors, split them:

```java
List<String> notDelivered = new ArrayList<>();
List<String> deliveredNoResponse = new ArrayList<>();
if (ChannelService.isDeliveryTrackingEnabled(ch)) {
    Long latestId = crossTenantMessageStore.findLastMessage(ch.id()).map(Message::id).orElse(null);
    for (String contributor : missing) {
        var membership = crossTenantMembershipStore.find(ch.id(), contributor);
        if (membership.isPresent() && membership.get().lastDeliveredMessageId() != null
                && latestId != null && membership.get().lastDeliveredMessageId() >= latestId) {
            deliveredNoResponse.add(contributor);
        } else {
            notDelivered.add(contributor);
        }
    }
} else {
    notDelivered.addAll(missing);
}
```

Pass both lists to `BarrierStuckContext`.

- [ ] **Step 6: Update evaluateConversationStall**

After identifying stalled commitments, resolve delivery status for the obligor:

```java
Boolean deliveryConfirmed = null;
if (ChannelService.isDeliveryTrackingEnabled(ch)) {
    // Find the COMMAND message for the stalled correlation
    // Check obligor's lastDeliveredMessageId against it
    // deliveryConfirmed = true if cursor past, false if behind
}
```

Pass to `ConversationStallContext`.

- [ ] **Step 7: Fix all call sites for updated records**

Use `ide_find_references` on `BarrierStuckContext` and `ConversationStallContext` to find all construction sites. Update each with the new fields.

- [ ] **Step 8: Run tests**

- [ ] **Step 9: Run full build**

- [ ] **Step 10: Commit**

```
feat(#376): watchdog — delivery-aware BARRIER_STUCK and CONVERSATION_STALL

BarrierStuckContext splits missing contributors into notDelivered
(transport issue) and deliveredNoResponse (agent issue).
ConversationStallContext adds deliveryConfirmed (Boolean, three-state)
for stalled obligations.

Refs #376
```

---

### Task 7: MCP Tool Surface

**Files:**
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpTools.java` (create_channel gains track_delivery; new set_delivery_tracking tool)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/mcp/QhorusMcpToolsBase.java` (toChannelDetail gains trackDelivery)
- Modify: `runtime/src/main/java/io/casehub/qhorus/runtime/channel/ChannelService.java` (setTrackDelivery method)
- Test: `runtime/src/test/java/io/casehub/qhorus/mcp/DeliveryTrackingToolTest.java` (new)

**Interfaces:**
- Consumes: `ChannelService.isDeliveryTrackingEnabled`, `ChannelMembershipStore.advanceDeliveredCursorForMembers` from Tasks 1-2
- Produces:
  - `create_channel` MCP tool gains `track_delivery` parameter
  - `set_delivery_tracking` new MCP tool
  - `ChannelDetail.trackDelivery` in get_channel/list_channels responses

- [ ] **Step 1: Write failing test**

Create `DeliveryTrackingToolTest.java`:

```java
@QuarkusTest
class DeliveryTrackingToolTest {

    @Inject QhorusMcpTools tools;
    @Inject ChannelMembershipStore membershipStore;

    @Test
    @TestTransaction
    void createChannel_withTrackDelivery_setsFlag() {
        tools.createChannel("track-test", null, "BARRIER", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, true);
        var detail = tools.getChannel("track-test");
        assertTrue(detail.trackDelivery());
    }

    @Test
    @TestTransaction
    void setDeliveryTracking_enablesOnExistingChannel() {
        tools.createChannel("append-track", null, "APPEND", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null);
        var detailBefore = tools.getChannel("append-track");
        assertFalse(detailBefore.trackDelivery());
        tools.setDeliveryTracking("append-track", true);
        var detailAfter = tools.getChannel("append-track");
        assertTrue(detailAfter.trackDelivery());
    }

    @Test
    @TestTransaction
    void setDeliveryTracking_null_revertsToSemanticDefault() {
        tools.createChannel("barrier-revert", null, "BARRIER", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, false);
        assertFalse(tools.getChannel("barrier-revert").trackDelivery());
        tools.setDeliveryTracking("barrier-revert", null);
        assertTrue(tools.getChannel("barrier-revert").trackDelivery());
    }

    @Test
    @TestTransaction
    void setDeliveryTracking_enableOnExisting_initializesMemberCursors() {
        tools.createChannel("init-test", null, "APPEND", null, null, null, null, null, null, null, null, null, null, null, null, null, null, null, null);
        // Add a member and a message
        // Enable tracking
        // Verify member's lastDeliveredMessageId initialized to latest message ID
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Add track_delivery to create_channel**

Add `@ToolArg(name = "track_delivery", ...) Boolean trackDelivery` parameter to `createChannel()`. Pass through to `ChannelCreateRequest.builder().trackDelivery(trackDelivery)`.

- [ ] **Step 4: Add setTrackDelivery to ChannelService**

```java
@Transactional
public void setTrackDelivery(UUID channelId, Boolean trackDelivery) {
    Channel ch = channelStore.findById(channelId).orElseThrow(
            () -> new IllegalArgumentException("Channel not found: " + channelId));
    boolean wasEnabled = isDeliveryTrackingEnabled(ch);
    channelStore.updateTrackDelivery(channelId, trackDelivery);
    Channel updated = ch.toBuilder().trackDelivery(trackDelivery).build();
    boolean nowEnabled = isDeliveryTrackingEnabled(updated);
    if (!wasEnabled && nowEnabled) {
        Long latestId = messageStore.findLastMessage(channelId).map(Message::id).orElse(null);
        if (latestId != null) {
            Set<String> memberIds = membershipStore.findByChannel(channelId).stream()
                    .map(ChannelMembership::memberId).collect(Collectors.toSet());
            if (!memberIds.isEmpty()) {
                membershipStore.advanceDeliveredCursorForMembers(channelId, memberIds, latestId);
            }
        }
    }
}
```

Add `updateTrackDelivery(UUID channelId, Boolean trackDelivery)` to `ChannelStore` interface and implementations.

- [ ] **Step 5: Add set_delivery_tracking MCP tool**

```java
@Tool(name = "set_delivery_tracking",
      description = "Enable, disable, or reset delivery tracking on a channel. "
              + "true = explicit opt-in, false = explicit opt-out, null/omit = revert to semantic default.")
@Transactional
public String setDeliveryTracking(
        @ToolArg(name = "channel", description = "Channel name or UUID") String channel,
        @ToolArg(name = "tracking", description = "true/false/null", required = false) Boolean tracking) {
    Channel ch = resolveChannel(channel);
    channelService.setTrackDelivery(ch.id(), tracking);
    boolean effective = ChannelService.isDeliveryTrackingEnabled(
            ch.toBuilder().trackDelivery(tracking).build());
    return "Delivery tracking " + (effective ? "enabled" : "disabled")
           + " on channel '" + ch.name() + "'";
}
```

- [ ] **Step 6: Run tests**

- [ ] **Step 7: Run full build**

- [ ] **Step 8: Commit**

```
feat(#376): MCP tools — create_channel track_delivery + set_delivery_tracking

create_channel gains track_delivery parameter. New set_delivery_tracking
tool with three-state (true/false/null). Enabling on existing channel
initializes member cursors to latest message ID. ChannelDetail shows
effective (resolved) tracking state.

Refs #376
```

---

### Task 8: Observer Advancement (WebSocket, Webhook) — Deferred Scope

**Note:** This task covers observer-based cursor advancement. The spec identifies that WebSocket requires **new infrastructure** (member identity tracking in `WebSocketConnectionRegistry`), and webhook requires member-to-registration association. These are non-trivial additions.

**Recommendation:** Implement the WebSocket and webhook observer advancement as a follow-up issue. Tasks 1-7 deliver the complete delivery tracking infrastructure — data model, store, pull path (MCP agents), push path (AT_LEAST_ONCE backends and A2A SSE), watchdog enrichments, and MCP tools. Observer advancement adds incremental value for browser clients and webhook consumers but requires architectural changes to the observer registry.

If the decision is to include it now, the implementation involves:
- Add `memberId` parameter to `WebSocketConnectionRegistry.subscribe()`
- Maintain `connection → memberId` mapping in the registry
- Use `TransactionSynchronization.afterCompletion()` in `WebSocketMessageObserver` to advance cursor post-commit
- Similar pattern for `WebhookMessageObserver` with registration-to-member mapping

**Files (if included):**
- Modify: `websocket-observer/src/main/java/.../WebSocketConnectionRegistry.java`
- Modify: `websocket-observer/src/main/java/.../WebSocketMessageObserver.java`
- Modify: `webhook-observer/src/main/java/.../WebhookMessageObserver.java`

---

### Task 9: Documentation and CLAUDE.md Update

**Files:**
- Modify: `CLAUDE.md` (add delivery tracking conventions to testing/project structure sections)

**Interfaces:**
- Consumes: All prior tasks
- Produces: Updated project documentation

- [ ] **Step 1: Update CLAUDE.md**

Add to the testing conventions section:
- `ChannelMembershipStore.updateLastDeliveredMessageId` and `advanceDeliveredCursorForMembers` are forward-only — implementation advances only if `messageId > current`.
- `ChannelService.isDeliveryTrackingEnabled(Channel)` is the single resolution point for delivery tracking defaults. BARRIER/COLLECT default on; APPEND/EPHEMERAL/LAST_WRITE default off. `Channel.trackDelivery()` (Boolean, nullable) overrides.
- `check_messages` advances the delivery cursor as a side effect when tracking is enabled and `reader_instance_id` is non-null. Tests asserting cursor state should use `@TestTransaction` (the advancement is in the same transaction as the query).
- `set_delivery_tracking` initializes member cursors to `latestMessageId` when enabling tracking on a channel with existing messages.
- `BarrierStuckContext` and `ConversationStallContext` are breaking record changes (new fields).

- [ ] **Step 2: Commit**

```
docs(#376): add delivery tracking conventions to CLAUDE.md

Refs #376
```

- [ ] **Step 3: Run final full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install -f /Users/mdproctor/claude/casehub/worktrees/26/qhorus/pom.xml`
Expected: BUILD SUCCESS — all modules compile, all tests pass.
