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

**Resolution helper:**

```java
boolean isDeliveryTrackingEnabled(Channel ch) {
    if (ch.trackDelivery() != null) return ch.trackDelivery();
    return ch.semantic() == ChannelSemantic.BARRIER
        || ch.semantic() == ChannelSemantic.COLLECT;
}
```

**Three-way diagnostic from the two cursors:**

| Condition | Meaning | Remediation |
|-----------|---------|-------------|
| `latestMessageId > lastDeliveredMessageId` | Not delivered | Retry / check transport |
| `lastDeliveredMessageId > lastReadMessageId` | Delivered, not processed | Wait or escalate |
| `lastReadMessageId >= latestMessageId` | Fully caught up | No action |

**Migrations:**

- **V36:** `ALTER TABLE channel ADD COLUMN track_delivery BOOLEAN`
- **V37:** `ALTER TABLE channel_membership ADD COLUMN last_delivered_message_id BIGINT`

### 2. Store layer

**`ChannelMembershipStore`** — new methods:

```java
void updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId);
void advanceDeliveredCursorForAll(UUID channelId, Long messageId);
```

Both are forward-only — implementation advances only if `messageId > current`.

`updateLastDeliveredMessageId` is for per-participant transports (A2A SSE, WebSocket).

`advanceDeliveredCursorForAll` is for per-room transports (connector-human, slack-bot) where one post reaches all channel members. Single UPDATE with WHERE clause instead of N individual updates.

**`ReactiveChannelMembershipStore`** — reactive parity:

```java
Uni<Void> updateLastDeliveredMessageId(UUID channelId, String memberId, Long messageId);
Uni<Void> advanceDeliveredCursorForAll(UUID channelId, Long messageId);
```

**InMemory implementations** — both blocking and reactive in `persistence-memory`.

**Contract tests** — new test methods in `ChannelMembershipStoreContractTest`:
- Forward-only advancement (second call with lower ID is a no-op)
- `advanceDeliveredCursorForAll` advances all members of a channel
- Null → first value advancement
- Idempotent repeated calls with same ID

### 3. Cursor advancement hooks

Advancement lives **inside each transport's delivery path**, not centralized. Only the transport knows whether delivery succeeded and which participant received it.

**Push backends — via `ChannelGateway.fanOut()`:**

After `backend.post()` returns without exception, the gateway advances the delivery cursor. The backend type (from `BackendEntry.backendType()`) determines which store method:

| Backend type | Advancement method | Rationale |
|-------------|-------------------|-----------|
| `a2a` | `updateLastDeliveredMessageId(channelId, consumerId, messageId)` | Per-participant SSE stream |
| `connector-human` | `advanceDeliveredCursorForAll(channelId, messageId)` | One post reaches all external members |
| `slack-bot` | `advanceDeliveredCursorForAll(channelId, messageId)` | One post reaches all Slack members |
| `qhorus-internal` | N/A | No-op post(); agents pull via MCP |

The gateway already has the channel ID and message from the `OutboundMessage`. It needs to resolve the message ID — `OutboundMessage` carries `messageId`.

For per-participant backends, the gateway needs the participant identity. A2A provides this via the SSE stream's correlation-to-consumer mapping. WebSocket provides it via the connection's user identity.

**Guard:** Before any advancement, the gateway checks `isDeliveryTrackingEnabled(channel)`. If false, no store call.

**Observer transports — `MessageObserver` implementations:**

WebSocket and webhook observers fan out internally to per-connection or per-registration endpoints:

- **WebSocket observer:** After frame sent, advance cursor for the connected user. `WebSocketConnectionRegistry` tracks user identity per connection.
- **Webhook observer:** After HTTP 2xx received, advance cursor. If registration carries a member association, advance per-participant; otherwise advance for all members.

**AT_LEAST_ONCE backends — `DeliveryBatchExecutor`:**

After `deliverBatch()` successfully posts to a backend, advance the delivery cursor. Same per-participant vs per-room logic. The batch executor already iterates messages and knows the backend.

**Pull path — `check_messages`:**

After the query returns messages, advance the delivery cursor for the calling agent. Details in Section 4.

### 4. `check_messages` delivery advancement

`checkMessages()` gains a side effect when delivery tracking is enabled. After querying messages, it advances the delivery cursor for the calling agent.

**Where it hooks in:** Each semantic variant (`checkMessagesAppend`, `checkMessagesBarrier`, `checkMessagesCollect`, `checkMessagesEphemeral`) returns a `CheckResult` with messages and a `lastId`. Before returning, if tracking is enabled, advance the cursor.

**Guard conditions:**
- `isDeliveryTrackingEnabled(ch)` = true
- `readerInstanceId` is non-null (anonymous checks don't advance)
- `lastId > 0` (empty result = nothing to advance)

**Semantic:** "You asked for the messages, that counts as delivery." No opt-out parameter. An agent calling `check_messages` has received the messages — they're in the response. If a "peek without advancing" operation is needed later, adding `mark_delivered=false` is backward-compatible.

**Reactive parity:** Same logic in `ReactiveQhorusMcpTools.checkMessages()`.

**Transaction boundary:** `checkMessages()` is already `@Transactional`. The cursor update happens in the same transaction as the message query.

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

`missingContributors` remains the full list for backward compatibility — it's the union of `notDelivered` and `deliveredNoResponse`.

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

**No other new tools.** Delivery tracking is transparent — a side effect of existing operations (`check_messages`, `fanOut`, observer delivery), not a new API surface.

## Out of Scope

- **`DELIVERY_LAG` watchdog condition** — net-new condition type. Follow-up issue.
- **Platform delivery SPI integration** — qhorus messages are not the right consumer (see rationale above).
- **Per-message delivery status queries** — "which participants received message #42?" Could be derived from cursor comparison but no dedicated tool in this issue.
- **Retry logic for failed deliveries** — the existing `DeliveryService` retry/reconciliation infrastructure handles backend-level retries. Per-participant retry for push failures is future work.

## Testing Strategy

**Unit tests (CDI-free):**
- `isDeliveryTrackingEnabled()` for all semantic × explicit-override combinations
- Forward-only cursor advancement semantics
- `advanceDeliveredCursorForAll` batch advancement

**Store contract tests:**
- `updateLastDeliveredMessageId` forward-only
- `advanceDeliveredCursorForAll` advances all members
- Null → first value
- Idempotent repeated calls

**Integration tests (`@QuarkusTest`):**
- `check_messages` advances delivery cursor when tracking enabled, skips when disabled
- `check_messages` with null `reader_instance_id` does not advance
- `fanOut` advances cursor for push backends on tracked channels
- `BARRIER_STUCK` watchdog produces `notDelivered` / `deliveredNoResponse` split
- `CONVERSATION_STALL` watchdog populates `deliveryConfirmed`
- Channel creation with explicit `track_delivery` override
- `set_delivery_tracking` toggle on existing channel

**Migration test:**
- `FlywayMigrationSchemaTest` extended to verify V36 and V37 produce correct schema
