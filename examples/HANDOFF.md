# casehub-examples Handover — 2026-08-20

## Last Session

Implemented per-type EventLevels for observation filtering (#7). Added `Function<E, EventLevel>` level-resolver constructor overload to `PartitionedObservationService` in blocks (branch `feat/per-event-level-resolver`). Three levels: DIALOGUE(30) > ACTION(20) > MOVEMENT(10). Also fixed pre-existing compile errors in CharacterProfileDTOTest (migrated to AgentDescriptor.builder()). 389 tests pass.

Moved 3 misplaced issues to their correct repos: #35 → Hortora/trellis#52, #36 → casehubio/docs#1, #37 → casehubio/platform#242. All closed in examples.

## Cross-Repo Changes

- **blocks** branch `feat/per-event-level-resolver` (commit 8677e2a) — needs review and merge. Also fixes pre-existing ChannelObserver compile error. Test suite has pre-existing compile errors in channel/negotiation tests (MessageReceivedEvent constructor changed in qhorus-api).

## What's Next

| Item | Scale | Complexity | Notes |
|------|-------|------------|-------|
| #48 Observation SPI extraction | M | Med | Gates eidos-api SPI definition |
| Hortora/trellis#52 — migrate ActionService rate limiter | S | Low | Depends on platform agent-gate |
| casehubio/docs#1 — document @Decorator CDI pattern | S | Low | Depends on #30 |
| casehubio/platform#242 — GatedAgentSession leak detection | S | Med | Depends on #30 |
