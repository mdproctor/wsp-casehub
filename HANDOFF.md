# HANDOFF — Slot 112: Cross-Platform Scenario Engine

**Date:** 2026-08-12
**Branch:** `issue-408-scenario-engine`
**Slot:** `/Users/mdproctor/claude/casehub/slots/112`
**Epic:** casehubio/parent#408

---

## Last Session

Major design pivot + full implementation of the first example application.

### Design pivot
Replaced standalone demo SPI implementations (#93, closed) with progressive example applications in casehub-examples. Demo SPI impls are demand-driven — built inside the apps that need them. Only mock what has no real in-memory alternative (used `chat-ref` RefChatPlatform instead of a mock).

### What was built

**casehub-examples/helpdesk/** on branch `issue-408-scenario-engine` (6 commits):

**Backend (Quarkus, 12 tests, all green):**
- `TicketService` — in-memory CRUD with status lifecycle
- `TicketClassifier` SPI + `DemoTicketClassifier` — lookup from scenario data (the only mock)
- `ChatInjectionResource` — fires `InboundMessage` CDI events via `chat-ref` pipeline
- `TicketCreationHandler` — `@ObservesAsync ReceivedMessage` → create + classify + assign
- `NotificationService` — sends via `ChatPlatform.messaging()`
- `ScenarioBootstrapResource`, `VerificationResource` — scenario infrastructure
- `help-desk-basic.yaml` — scenario file

**Frontend (Vite + Lit, first version):**
- Live ops dashboard: metrics, ticket table, submit form, event log, notifications
- Auto-polls REST endpoints every 2s via Vite proxy
- Demonstrated end-to-end in Playwright: bootstrap → submit → classify → assign → resolve → notify

### Issue changes
- **#93 closed** — premise wrong (demo SPI convention is a pattern, not a build list)
- **#95 created and completed** — helpdesk example app
- **#94 removed from queue** — independent of scenario engine epic
- **#411 created** — example showcase gallery (capability matrix, README rendering)
- **#412 created** — helpdesk UI enhancement (blocks-ui components, SSE push)
- **#413 created** — future example slices (umbrella)

## Queue State

```
[x] #409 — Scenario format specification (M / Med) [parent]
[x] #410 — Platform protocol — demo SPI convention (S / Low) [parent]
[x] #311 — Scenario executor backend (XL / High) [pages]
[x] #95  — Slice 1: IT help desk example app (M / Med) [connectors]
[ ] #109 — Life household scenario files (L / High) [life]
[ ] #149 — Migrate DemoDataSeeder to scenario format (M / Med) [clinical]
[ ] #411 — Example showcase gallery (M / Med) [parent] ← active
[ ] #412 — Helpdesk UI — blocks-ui + SSE push (M / Med) [parent]
[ ] #413 — Slice 2+ — additional example apps (L / High) [parent]
```

Position: 4/9 (4 done, 5 remaining). Next: #411.

## Immediate Next Step — #411: Example Showcase Gallery

Build a parent gallery app in casehub-examples using blocks-ui + pages:
- Lists all examples with capability labels and descriptions
- Auto-generates capability coverage matrix from per-example YAML metadata
- Renders each example's README as lesson content
- Filter/search by capability
- Navigate into each example's own web app

### Key context
- blocks-ui at `/Users/mdproctor/claude/casehub/blocks-ui/` (not in slot — add it)
- blocks-ui-examples pattern: Vite aliases resolve `@casehubio/*` from source trees
- blocks-ui has 40+ pre-built components: execution-monitor, case-explorer, work-item-inbox, channel-activity, blocks-timeline, kpi-metric-row, etc.
- pages packages already built in slot at `/Users/mdproctor/claude/casehub/slots/112/pages/`

### Repos needed
- casehub-examples (`/Users/mdproctor/claude/casehub/examples/` — branch `issue-408-scenario-engine`)
- blocks-ui (`/Users/mdproctor/claude/casehub/blocks-ui/` — needs adding to slot or using main checkout)
- pages (in slot at `/Users/mdproctor/claude/casehub/slots/112/pages/`)

## Design Principles (established this session)

1. **Only mock what has no real in-memory alternative.** RefChatPlatform, not DemoChatPlatform.
2. **Demo SPI impls are demand-driven.** Build inside the app, extract when a second consumer appears.
3. **Every external dependency swappable via SPI + CDI profile.** Use existing SPIs where they exist.
4. **Verify outcomes, not outputs.** Scenario assertions check state changes, not LLM text.
5. **When you hit a gap, fix the platform.** Example apps are a forcing function.
6. **Use real capabilities, don't reinvent.** blocks-ui components, pages components, chat-ref — use what exists.

## References

- Examples commits: `7e31864..0c0bd18` on branch `issue-408-scenario-engine`
- Workspace commits: `616e457`, `045eb77`, `82566e5` on branch `issue-408-scenario-engine`
- Design spec: `specs/issue-408-scenario-engine/2026-08-12-example-applications-design.md`
- Plan: `plans/2026-08-12-helpdesk-example.md`
- `.plan` at `/Users/mdproctor/claude/casehub/slots/112/.plan`
