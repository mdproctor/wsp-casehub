# HANDOFF — Slot 112: Cross-Platform Scenario Engine

**Date:** 2026-08-13
**Branch:** `issue-408-scenario-engine`
**Slot:** `/Users/mdproctor/claude/casehub/slots/112`
**Epic:** casehubio/parent#408

---

## Last Session

Designed and partially implemented #412 (Helpdesk UI — blocks-ui components, push, scenario controller). Full brainstorming → spec → reviewed spec → plan → reviewed plan → execution cycle.

**Design decisions** (7 decisions, light decision review, standard spec review — 3 rounds, 22 issues resolved):
- Pages-push WebSocket protocol (not SSE) for real-time push
- CDI event bridge (synchronous `@Observes`) → `EventBroadcaster.broadcast()` per structured topic
- Per-topic `EventStreamController` instances (typed, pool-shared WebSocket)
- `blocks-kpi-metric-row`, `pages-table`, `blocks-timeline` with custom `HelpdeskPipelineStrategy`
- Collapsible scenario controller panel via pages split, step-by-step pacing with push observation
- Production code quality by default — example apps are skeleton starters

**Backend implementation complete** (Tasks 1-3 of 8):
- Task 1: `TicketEvent`/`NotificationEvent` CDI records, `TicketService`/`NotificationService` fire synchronously
- Task 2: `HelpdeskPushEndpoint` (`@WebSocket /push`), `HelpdeskSessionSender`, `ConnectionRegistry`
- Task 3: `TicketPushObserver` bridges CDI events → `EventBroadcaster`, custom `HelpdeskJsonWriter` with JSR310

All 18 backend tests green. 3 commits on examples repo: `697b6ba`, `60cace8`, `b45a29f`.

**Frontend not started** (Tasks 4-8 remain).

## Immediate Next Step

Run `/work` to continue on #412. Start Task 4 (frontend push connection + KPI metrics). Prerequisites:
1. Examples repo is already on `issue-408-scenario-engine` (stashed main changes — `git stash list` to see)
2. The pages slot copy (`/Users/mdproctor/claude/casehub/slots/112/pages`) is behind main — `EventStreamController` was moved to `pages-component` on main but isn't in the slot. Either sync the slot or use the main pages checkout at `/Users/mdproctor/claude/casehub/pages/`
3. Verify blocks-ui package names against each `package.json` `name` field before adding Vite aliases (GE-20260803-17fc03)
4. blocks-ui is at `/Users/mdproctor/claude/casehub/blocks-ui/` (not in slot — session should `add-dir` it)
5. Backend runs on port 8090 (`mvn quarkus:dev`), frontend Vite dev server needs WebSocket proxy for `/push`

## Key Context for Frontend Tasks

**Push connection:** Three `EventStreamController<T>` instances — one per topic (`helpdesk:tickets`, `helpdesk:notifications`, `helpdesk:metrics`). `EventStreamPool` reuses one WebSocket. Import from `@casehubio/pages-component`. API: `latest` (most recent payload), `all` (full history), `status` (connection state). Auto-calls `requestUpdate()`.

**Ticket state derivation:** Computed getter rebuilding from `_ticketPush.all` — no `@state()` field needed. Idempotent, handles reconnection replay. See spec §Ticket Table.

**Timeline strategy:** 4 stages (created/classified/assigned/resolved) mapping 1:1 to `TicketEvent.Type`. `STATUS_TO_STAGE` maps `TicketStatus` enum values to indices. See spec §Pipeline Timeline.

**Scenario controller:** Embedded TS module for steps, temporal correlation for ticket-to-event matching (record `_ticketPush.all.length` before submit, watch for next CREATED). `?standalone` query param for standalone mode.

**UX requirements (user-specified):** Click feedback on all buttons/tabs. Scenario text visible before submission. Explicit Next/Submit pacing. Must work in separate browser window.

## References

- Design spec: `specs/issue-408-scenario-engine/2026-08-13-helpdesk-ui-upgrade-design.md` (in workspace)
- Decisions: `specs/issue-408-scenario-engine/decisions.md` (in workspace)
- Implementation plan: `plans/2026-08-13-helpdesk-ui-upgrade.md` (in workspace)
- Spec review: `/Users/mdproctor/reviews/casehub-slots/issue-412-helpdesk-ui-spec-20260813-153238/`
- Plan review: `/Users/mdproctor/reviews/casehub-slots/issue-412-helpdesk-plan-20260813-162413/`
- `.plan` at `/Users/mdproctor/claude/casehub/slots/112/.plan`

## Design Principles (established prior session, still valid)

1. Only mock what has no real in-memory alternative
2. Demo SPI impls are demand-driven
3. Every external dependency swappable via SPI + CDI profile
4. Verify outcomes, not outputs
5. When you hit a gap, fix the platform
6. Use real capabilities, don't reinvent
7. **Production code quality by default** — example apps are skeleton starters (new this session)
