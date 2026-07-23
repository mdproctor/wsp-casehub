# Opt-in Delivery Tracking for BARRIER and COLLECT Channels

**Issue:** casehubio/qhorus#376
**Date:** 2026-07-23
**Status:** Approved

## Problem

Qhorus has three tracking layers for message lifecycle, but a gap between backend delivery and participant acknowledgement:

| Question | Answered by | Exists? |
|----------|-------------|---------|
| Did the backend accept the message? | `DeliveryCursor` (per backend, per channel) | Yes |
| Did participant X receive the message? | **Nothing** | **No** |
| Has participant X read up to message N? | `lastReadMessageId` (per member) | Yes |
| Did participant X acknowledge the obligation? | `CommitmentStore` (OPEN→ACKNOWLEDGED) | Yes |

`DeliveryCursor` is per-backend, not per-participant. If a message reaches the Slack backend but one participant's agent process has crashed, the cursor shows success while one participant never received it.

The commitment lifecycle covers obligations (COMMANDs), but non-COMMAND messages — BARRIER contributions, HANDOFF context, informational updates — have no commitment. Whether a participant received those matters for coordination, but nothing tracks it today.

The watchdog's `BARRIER_STUCK` condition can say "contributor X hasn't written" but cannot distinguish "never received the COMMAND" from "received it but hasn't responded." Same gap in `CONVERSATION_STALL`.

## Approach

Extend `channel_membership` with a `lastDeliveredMessageId` cursor — same shape as the existing `lastReadMessageId`. Forward-only, per-member, per-channel. Advanced when the transport layer confirms delivery to a specific participant.

Opt-in by channel semantic: BARRIER and COLLECT enable tracking automatically. All others default off. Explicit override via `channel.trackDelivery`.

### Why not platform's delivery SPI?

The platform's `DeliveryAttempt` / `EngagementEvent` infrastructure (post platform#192) is designed for notification delivery — multi-channel, multi-attempt, retry with backoff, per-attempt failure reasons. That's the right shape for "did this alert reach this person via email AND Slack AND push."

Qhorus message delivery is cursor-shaped: "transport confirmed, advance the high-water mark." Using the platform SPI would create:

1. **Shape mismatch** — cursor vs multi-attempt retry store
2. **Storage duplication** — `EngagementEvent.OPENED` duplicates `lastReadMessageId`
3. **Cross-store queries** — watchdog would query platform's store alongside qhorus's
4. **Retention overhead** — platform delivery tracking needs scheduled sweeps; a cursor column has none

Platform's delivery SPI serves notification delivery (#375). Qhorus messages are a different consumer.

## Design

### 1. Data model

**`ChannelMembership` record** — add `lastDeliveredMessageId`:

```java
public record ChannelMembership(
        Long id, UUID channelId, String memberId, MemberRole role,
        String tenancyId, Instant joinedAt,
        Long lastReadMessageId, Long lastDeliveredMessageId) {

    // Backward-compatible constructor — existing callers pass 7 args
    public ChannelMembership(Long id, UUID channelId, String memberId, MemberRole role,
                             String tenancyId, Instant joinedAt, Long lastReadMessageId) {
        this(id, channelId, memberId, role, tenancyId, joinedAt, lastReadMessageId, null);
    }
}
```

**`ChannelMembershipEntity`** — new column:

```sql
ALTER TABLE channel_membership ADD COLUMN last_delivered_message_id BIGINT;
```

Nullable, no FK. Same pattern as `lastReadMessageId`. Existing rows get null — tracking starts from next message after feature is enabled.

**`Channel` record** — add `trackDelivery` (Boolean, nullable):

- `null` = use semantic default (true for BARRIER/COLLECT, false for others)
- `true` = explicit opt-in regardless of semantic
- `false` = explicit opt-out regardless of semantic

**`ChannelEntity`** — new column:

```sql
ALTER TABLE channel ADD COLUMN track_delivery BOOLEAN;
```

Nullable. Null means "use semantic default."

**`ChannelCreateRequest`** — add `trackDelivery(Boolean)` to builder. Null default.

**`OutboundMessage` record** — add `Long sequenceId`:

```java
public record OutboundMessage(
        UUID messageId,
        Long sequenceId,        // database sequence ID for cursor advancement
        String sender, MessageType type, String content,
        String correlationId, Long inReplyTo, ActorType senderActorType,
        List<ArtefactRef> artefactRefs, String target) {}
```

The existing `messageId` (UUID) serves as a correlation/idempotency key. The new `sequenceId` carries the `Message.id()` Long value needed for cursor advancement. All `OutboundMessage` construction sites must pass the sequence ID: `MessageService.dispatch()` has it from `messageStore.put(message)`, `DeliveryBatchExecutor.toOutbound()` has it from the `Message` record, and `ChannelGateway.deliverRemote()` receives it as a parameter.

**Resolution helper:**

```java
boolean isDeliveryTrackingEnabled(Channel ch) {
    if (ch.trackDelivery() != null) return ch.trackDelivery();
    return ch.semantic() == ChannelSemantic.BARRIER
        || ch.semantic() == ChannelSemantic.COLLECT;
}
```

All five semantics have defined defaults:

| Semantic | Default | Rationale |
|----------|---------|-----------|
| BARRIER | On | Coordination depends on knowing who received the gate command |
| COLLECT | On | Fan-in aggregation depends on knowing contributors received the prompt |
| APPEND | Off | General conversation; delivery tracking is overhead without coordination benefit |
| EPHEMERAL | Off | Fire-and-forget by design |
| LAST_WRITE | Off | Single-value blackboard; no coordination semantics |

Any semantic can be overridden via explicit `channel.trackDelivery`.

**Three-way diagnostic from the two cursors:**

| Condition | Meaning | Remediation |
|-----------|---------|-------------|
| `latestMessageId > lastDeliveredMessageId` | Not delivered | Retry / check transport |
| `lastDeliveredMessageId > lastReadMessageId` | Delivered, not processed | Wait or escalate |
| `lastReadMessageId >= latestMessageId` | Fully caught up | No action |

For BARRIER and COLLECT channels where messages are deleted after delivery: the diagnostic is meaningful during the in-progress period (between message dispatch and agent pull). After a successful delivery cycle, messages are deleted and the channel is empty until the next round. `latestMessageId` refers to the highest message ID currently in the channel — after deletion this may be null or an EVENT message ID. The "fully caught up" state is implicitly satisfied when no undelivered messages remain.

Cursor advancement in `check_messages` happens **before** message deletion in the BARRIER/COLLECT paths, ensuring the delivery record is captured before messages are removed.

**Migrations:**

- **V40:** `ALTER TABLE channel ADD COLUMN track_delivery BOOLEAN`
- **V41:** `ALTER TABLE channel_membership ADD COLUMN last_delivered_message_id BIGINT`

### 2. Store layer

**`ChannelMembershipStore`** — new methods:

```java
void updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId);
void advanceDeliveredCursorForMembers(UUID channelId, Set<String> memberIds, Long messageId);
```

Both are forward-only — implementation advances only if `messageId > current`.

`updateLastDeliveredMessageId` is for per-participant transports (A2A SSE, WebSocket, `check_messages` pull).

`advanceDeliveredCursorForMembers` is for transports where one post reaches a known set of participants (e.g., external platform backends). The caller provides the specific member IDs to advance — not all channel members. Single UPDATE with WHERE clause instead of N individual updates.

**Reactive parity:** Deferred. The `issue-384-retire-reactive` branch is in progress locally. Adding new reactive API surface while the reactive stack is being evaluated for retirement is premature. If the reactive retirement does not proceed, reactive parity methods can be added as a follow-up.

**InMemory implementations** — blocking in `persistence-memory`.

**Contract tests** — new test methods in `ChannelMembershipStoreContractTest`:
- Forward-only advancement (second call with lower ID is a no-op)
- `advanceDeliveredCursorForMembers` advances specified members of a channel
- Null → first value advancement
- Idempotent repeated calls with same ID

### 3. Cursor advancement hooks

Advancement lives **inside each transport's delivery path**, not centralized. Only the transport knows whether delivery succeeded and which participant received it.

The hook location depends on the backend's `DeliveryGuarantee`:

**AT_LEAST_ONCE backends — via `DeliveryBatchExecutor.deliverBatch()`:**

Backends declaring `DeliveryGuarantee.AT_LEAST_ONCE` are skipped by `fanOut()` and delivered through the delivery pump (`DeliveryService` → `DeliveryBatchExecutor`). `deliverBatch()` is already `@Transactional`, reads messages from the `DeliveryCursor`, and has access to the full `Message` record (including `Long id`). This is the natural hook point for cursor advancement.

After `backend.post(ref, outbound)` succeeds in `deliverBatch()`, advance the delivery cursor:

| Backend | Guarantee | Advancement method | Rationale |
|---------|-----------|-------------------|-----------|
| `connector-human` | AT_LEAST_ONCE | `advanceDeliveredCursorForMembers(channelId, platformMemberIds, sequenceId)` | One post reaches all external platform members |
| `slack-bot` | AT_LEAST_ONCE | `advanceDeliveredCursorForMembers(channelId, platformMemberIds, sequenceId)` | One post reaches all Slack members |
| `openclaw` | AT_LEAST_ONCE | `updateLastDeliveredMessageId(channelId, consumerId, sequenceId)` | Per-consumer webhook delivery |

For `connector-human` and `slack-bot`, the caller must resolve `platformMemberIds` — the set of channel members whose delivery path is the external platform. This excludes agent members (whose delivery path is `check_messages`) and members on other backends (A2A, WebSocket). The resolution queries `channelMembershipStore.findByChannel()` and filters to members served by this backend. The exact filtering mechanism (by actor type lookup or backend consumer registry) is an implementation detail.

**BEST_EFFORT push backends — via `ChannelGateway.fanOut()`:**

BEST_EFFORT backends are dispatched in fire-and-forget virtual threads via `Thread.ofVirtual().start()`. Three constraints apply:

1. No completion signal back to the calling thread
2. CDI `@Transactional` does not propagate to virtual threads started with `Thread.ofVirtual().start()`
3. The virtual thread closure captures only the variables explicitly referenced

For cursor advancement in BEST_EFFORT virtual threads, use programmatic transaction management: `QuarkusTransaction.requiringNew()` opens a new transaction within the virtual thread after `backend.post()` succeeds. The virtual thread closure must capture: (a) the `ChannelMembershipStore` reference (injected into `ChannelGateway`), (b) the channel ID, (c) the `OutboundMessage` (which now carries `sequenceId`), and (d) the delivery tracking enabled flag.

| Backend | Guarantee | Advancement method | Rationale |
|---------|-----------|-------------------|-----------|
| `a2a` | BEST_EFFORT | `updateLastDeliveredMessageId(channelId, consumerId, sequenceId)` | Per-participant SSE stream |
| `qhorus-internal` | N/A (skipped in fanOut) | N/A | Agents pull via MCP; cursor advances in `check_messages` |

For A2A, the consumer ID comes from the SSE stream's correlation-to-consumer mapping.

**Guard:** Before any advancement, the delivery code checks `isDeliveryTrackingEnabled(channel)`. If false, no store call.

**Observer transports — `MessageObserver` implementations:**

Observers fire **before the enclosing transaction commits** (documented in `MessageObserver` Javadoc). Cursor advancement inside an observer's `onMessage()` path would run pre-commit — if the transaction rolls back, the cursor may be advanced for a message that was never persisted.

To handle this: cursor advancement for observers uses `TransactionSynchronization.afterCompletion()` — the observer records the delivery event (connection ID → member ID mapping, message sequence ID), and a post-commit callback advances the cursor only if the transaction committed. This pattern already exists in `MessageService.dispatch()` for delivery signalling.

- **WebSocket observer:** After frame sent, record the delivery for post-commit cursor advancement. **Requires adding member identity tracking to `WebSocketConnectionRegistry`** — the current registry maps `channelId → Set<WebSocketConnection>` with no member identity. `subscribe()` must gain a `memberId` parameter, and the registry must maintain a `connection → memberId` mapping. This is new infrastructure.
- **Webhook observer:** After HTTP 2xx received, record the delivery. If registration carries a member association, advance per-participant; otherwise skip (no member identity to advance).
- **Kafka observer:** Not applicable. `KafkaMessageObserver` is a LOCAL-scoped observer that publishes message events to a Kafka topic for external consumers (system integration). Kafka consumers are not channel members with membership records — delivery tracking does not apply.

**Pull path — `check_messages` and `wait_for_reply`:**

After the query returns messages, advance the delivery cursor for the calling agent. Details in Section 4.

### 4. `check_messages` and `wait_for_reply` delivery advancement

Both MCP pull paths gain a side effect when delivery tracking is enabled.

**`check_messages`:**

Each semantic variant (`checkMessagesAppend`, `checkMessagesBarrier`, `checkMessagesCollect`, `checkMessagesEphemeral`) returns a `CheckResult` with messages and a `lastId`. Before returning (and before any message deletion for BARRIER/COLLECT), if tracking is enabled, advance the cursor.

**Guard conditions:**
- `isDeliveryTrackingEnabled(ch)` = true
- `readerInstanceId` is non-null (anonymous checks don't advance)
- `lastId > 0` (empty result = nothing to advance)

**Semantic:** "You asked for the messages, that counts as delivery." No opt-out parameter. An agent calling `check_messages` has received the messages — they're in the response. If a "peek without advancing" operation is needed later, adding `mark_delivered=false` is backward-compatible.

**Transaction boundary:** `checkMessages()` is already `@Transactional`. The cursor update happens in the same transaction as the message query.

**Ordering for BARRIER/COLLECT:** Cursor advancement happens before `messageStore.deleteNonEvent()`. The sequence is: (1) query messages, (2) advance delivery cursor to `lastId`, (3) delete messages. This ensures the delivery record is captured before messages are removed.

**`wait_for_reply`:**

`wait_for_reply` is a separate code path from `check_messages` — it polls `CommitmentStore.findByCorrelationId()` and `MessageService.findResponseByCorrelationId()` / `findDoneByCorrelationId()` in a loop. It does not delegate to `check_messages`.

When `wait_for_reply` finds a matching RESPONSE or DONE message and returns it, advance the delivery cursor for the calling agent to that message's sequence ID. Guard conditions:
- `isDeliveryTrackingEnabled(ch)` = true
- Instance ID is available from the calling context
- The matched message has a valid sequence ID

**Transaction boundary:** `wait_for_reply` polls outside a long-running transaction. The cursor advancement should use a dedicated transaction for the update (same pattern as the poll's message lookups).

### 5. Watchdog enrichments

**`BARRIER_STUCK` — delivery-aware diagnostics:**

For each missing contributor, query `ChannelMembershipStore.find(channelId, contributorId)` and compare `lastDeliveredMessageId` against the channel's latest message ID:

- `lastDeliveredMessageId` is null or less than the latest COMMAND → contributor marked as "not delivered"
- `lastDeliveredMessageId` covers the COMMAND but no contribution → contributor marked as "delivered, not responded"

`BarrierStuckContext` gains two new fields:

```java
public record BarrierStuckContext(
        UUID channelId, String channelName,
        List<String> missingContributors,  // existing — union of both
        List<String> notDelivered,         // new — transport issue
        List<String> deliveredNoResponse,  // new — agent issue
        long elapsedSeconds) implements AlertContext { ... }
```

This is a **breaking change** to the record's canonical constructor — all call sites constructing `BarrierStuckContext` must be updated to pass the two new fields. `missingContributors` remains as the union of both lists for callers that don't need the split.

**`CONVERSATION_STALL` — delivery context:**

For each stalled correlation, check whether the obligor's `lastDeliveredMessageId` covers the COMMAND message. Add to `ConversationStallContext`:

```java
public record ConversationStallContext(
        UUID channelId, String channelName,
        int stalledCount, List<String> correlationIds,
        long stalledSeconds,
        boolean deliveryConfirmed) implements AlertContext { ... }
```

`deliveryConfirmed` = true when the obligor received the COMMAND (delivery cursor past the COMMAND's message ID). False or null when delivery status is unknown (tracking not enabled, or no membership record).

**`DELIVERY_LAG` — out of scope.** Separate follow-up issue depending on #376. New condition type, threshold semantics, evaluation cadence — distinct from enriching existing conditions.

### 6. MCP tool surface

**`create_channel`** — gains optional `track_delivery` parameter (Boolean). Null = semantic default. Passed through to `ChannelCreateRequest`.

**`get_channel` / `list_channels`** — `ChannelDetail` gains `trackDelivery` field showing effective state (resolved, not raw — so callers see `true` for BARRIER channels even when the column is null).

**`set_delivery_tracking`** — new tool to toggle tracking on an existing channel:

```
set_delivery_tracking(channel, enabled)
```

Takes UUID-or-name channel reference and boolean. Updates `channel.trackDelivery`. Follows the UUID-first service pattern per `mcp-tool-channel-resolution-boundary` protocol.

**No other new tools.** Delivery tracking is transparent — a side effect of existing operations (`check_messages`, `wait_for_reply`, delivery pump, observer delivery), not a new API surface.

## Out of Scope

- **`DELIVERY_LAG` watchdog condition** — net-new condition type. GitHub issue to be filed before implementation begins.
- **Platform delivery SPI integration** — qhorus messages are not the right consumer (see rationale above). GitHub issue to be filed for tracking purposes.
- **Per-message delivery status queries** — "which participants received message #42?" Could be derived from cursor comparison but no dedicated tool in this issue. GitHub issue to be filed.
- **Retry logic for failed deliveries** — the existing `DeliveryService` retry/reconciliation infrastructure handles backend-level retries. Per-participant retry for push failures is future work. GitHub issue to be filed.

## Testing Strategy

**Unit tests (CDI-free):**
- `isDeliveryTrackingEnabled()` for all five semantics × explicit-override combinations
- Forward-only cursor advancement semantics
- `advanceDeliveredCursorForMembers` batch advancement

**Store contract tests:**
- `updateLastDeliveredMessageId` forward-only
- `advanceDeliveredCursorForMembers` advances specified members
- Null → first value
- Idempotent repeated calls

**Integration tests (`@QuarkusTest`):**
- `check_messages` advances delivery cursor when tracking enabled, skips when disabled
- `check_messages` with null `reader_instance_id` does not advance
- `wait_for_reply` advances delivery cursor when returning a matched message
- `DeliveryBatchExecutor.deliverBatch()` advances cursor for AT_LEAST_ONCE backends on tracked channels
- `fanOut` advances cursor for BEST_EFFORT push backends on tracked channels (A2A)
- `BARRIER_STUCK` watchdog produces `notDelivered` / `deliveredNoResponse` split
- `CONVERSATION_STALL` watchdog populates `deliveryConfirmed`
- Channel creation with explicit `track_delivery` override
- `set_delivery_tracking` toggle on existing channel
- Cursor advancement before message deletion in BARRIER/COLLECT paths

**Migration test:**
- `FlywayMigrationSchemaTest` extended to verify V40 and V41 produce correct schema
