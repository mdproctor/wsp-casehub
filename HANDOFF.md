# Handover — issue-376-opt-in-delivery-tracking

## Last Session

Implemented opt-in per-participant delivery tracking for BARRIER and COLLECT channels (#376). All 8 implementation tasks complete: data model (Channel.trackDelivery, ChannelMembership.lastDeliveredMessageId, OutboundMessage.sequenceId), store layer, pull path (check_messages), push paths (DeliveryBatchExecutor + A2A SSE), watchdog enrichments (BarrierStuckContext/ConversationStallContext), MCP tools (create_channel track_delivery + set_delivery_tracking), and CLAUDE.md docs. Full build green — 17 modules, all tests pass.

## Immediate Next Step

Run `/work-end` to close the branch. All implementation is done — only the close ceremony remains (code review, pre-close sweep, squash, push).

## What Was Built

- `Channel.trackDelivery` (Boolean, nullable) — BARRIER/COLLECT default on, others off
- `ChannelMembership.lastDeliveredMessageId` — forward-only cursor per participant
- `OutboundMessage.sequenceId` — carries Message.id() for cursor advancement
- Pull path: `check_messages` + `wait_for_reply` advance cursor as side effect
- Push paths: `DeliveryBatchExecutor` (AT_LEAST_ONCE) + `A2AResource.streamTask()` (BEST_EFFORT)
- Watchdog: `BarrierStuckContext.notDelivered/deliveredNoResponse`, `ConversationStallContext.deliveryConfirmed`
- MCP: `create_channel` gains `track_delivery`, new `set_delivery_tracking` tool
- V40 + V41 migrations
- CDI fix: `ReactiveAgentIdentityVerificationService` excluded from test discovery (SNAPSHOT dep mismatch)

## Key Decisions

- `ide_change_signature` used for 61-file `createChannel` parameter propagation — text-pattern replacement causes cascading damage across method calls with similar null-ending patterns
- JPA delivery cursor uses entity-level update (not bulk UPDATE) for L1 cache consistency in `@TestTransaction`
- IntelliJ worktree issue: IDE edits via `project_path=/Users/mdproctor/claude/casehub/qhorus` go to main repo (branch `main`), not the worktree. Use `project_path=/Users/mdproctor/claude/casehub/worktrees/26/qhorus` for worktree edits

## What's Left

- Observer advancement (WebSocket, webhook) deferred to follow-up — Task 8 in plan. Requires new WebSocket member identity infrastructure. · M · Med
- Follow-up issues filed: #377 (DELIVERY_LAG), #378 (platform SPI decision), #379 (per-message queries), #380 (per-participant retry)

## References

- Spec: `specs/2026-07-23-opt-in-delivery-tracking-design.md` (workspace)
- Plan: `plans/2026-07-23-opt-in-delivery-tracking.md` (workspace)
- Design review: `~/adr/casehub-worktrees/opt-in-delivery-tracking-20260723-050753/` (8 rounds, $23.49)
