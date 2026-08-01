# HANDOFF — 2026-08-01

## Last Session

Phase 2.6 — observation accumulator wiring with casehub-blocks. Built central `ObservationService` with per-character per-room `ObservationAccumulator` instances, two-tier compaction (mechanical supersession + LLM summarisation), cross-room memory retention via drain-once caching, and aside privacy filtering. Enriched `ManorEvent` with structured action metadata. Then promoted the generic `PartitionedObservationService` to casehub-blocks (blocks#76) and refactored wacky-manor to consume it. Added self-contained multi-agent monitoring example to blocks.

Adversarial design review (4 dimensions, 48 issues, $55.60) enriched the spec significantly — drove `ConcurrentHashMap` for thread safety, `volatile` on `currentRoom`, departure-room routing from event payload, and LLM failure fallback.

## Immediate Next Step

Phase 2.7 — live LLM narrator consuming the accumulated event stream. The observation pipeline is the narrator's input source. `NarratorAgent` class exists but is not wired.

## What's Left

- blocks#76 spec + blog entry — deferred from this session (the promotion was done, but no spec/blog written for blocks itself) · S · Low
- blocks issue-47 rebased but not pushed — owning session should verify and push · XS · Low
- Push garden entry GE-20260728-f7ad43 — committed locally, pre-push hook blocked · XS · Low
- Wacky-manor not in parent pom modules list — `mvn` requires `-f wacky-manor/pom.xml` · XS · Low

## What's Next

| Phase | Description | Scale | Complexity | Notes |
|---|---|---|---|---|
| 2.7 | Live LLM narrator — wire NarratorAgent to accumulated event stream | S | Low | NarratorAgent class exists, not wired |
| 2.8 | NPC system — Tier 2/3 scripted fixtures, player/NPC split | M | Med | RPG framing |
| 2.9 | Scale to 6 rooms — Library, Laboratory, Tower | L | Med | Re-evaluate buffer growth (examples#6) |
| 3.0 | Platform integration — memory, trust, HiL, replay | XL | High | Full casehub exercise |

## Cross-repo Changes This Session

| Repo | What | Branch |
|------|------|--------|
| casehubio/examples | Phase 2.6 + blocks consumer refactor | feat/wacky-manor-poc (pushed fork + upstream) |
| casehubio/blocks | PartitionedObservationService + example | main (pushed) |
| casehubio/blocks | issue-47 rebase conflict resolution | issue-47-llm-htn-heuristics (local only) |
| mdproctor.github.io | Diary entry published | main (pushed) |
| casehubio.github.io | Diary entry published | main (pushed) |

## Key Decisions

- Central `ObservationService` over per-character — visibility is cross-cutting
- Two-tier compaction: mechanical (deterministic) then LLM (conditional on budget threshold)
- Per-room accumulators with drain-once caching for remembered partitions
- `VisibilityPolicy<E, K>` SPI — applications define routing, blocks owns the machinery
- EventStreamBus deliberately avoided — direct method calls for dynamic room-based routing (GE-20260629-e8b16d)
- `ManorEvent` enriched with structured fields — compaction operates on metadata, never parses narrative text

## References

- Spec: `specs/issue-5-observation-accumulator-blocks/2026-08-01-observation-accumulator-design.md`
- Blog: `~/claude/mdproctor.github.io/_notes/2026-08-01-mdp01-when-characters-need-to-remember.md`
- Design review: `~/reviews/casehub-worktrees/observation-accumulator-*` (4 dimensions)
- Plan: `plans/2026-08-01-observation-accumulator.md`
- Phase 2.5 spec: `wacky-manor/docs/specs/2026-07-27-phase-2.5-autonomous-characters-design.md`
- blocks issue: casehubio/blocks#76
- Deferred issues: casehubio/examples#6 (buffer growth), #7 (per-type EventLevels), #8 (full blocks pipeline)
