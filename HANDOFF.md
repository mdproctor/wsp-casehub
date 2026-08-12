# HANDOFF — Slot 112: Cross-Platform Scenario Engine

**Date:** 2026-08-12
**Branch:** `issue-408-scenario-engine`
**Slot:** `/Users/mdproctor/claude/casehub/slots/112`
**Epic:** casehubio/parent#408

---

## Last Session

Major design pivot: replaced standalone demo SPI implementations (#93) with progressive example applications in casehub-examples. Built the first example (IT help desk) end-to-end, then refined the approach based on platform design principles.

### Key decisions made in conversation

1. **Demo SPI impls are demand-driven, not supply-driven.** The demo-spi-convention.md describes a pattern — you apply it when an app needs a demo alternative, not as a build list. #93 was wrong; closed it.

2. **Use real platform capabilities, only mock what you must.** Refactored the helpdesk to use `RefChatPlatform` + `InMemoryChatBackend` from `chat-ref` instead of a mock `DemoChatPlatform`. The only mock remaining is `DemoTicketClassifier` — the genuine case where no real in-memory alternative exists (LLM classification).

3. **Every LLM integration point must sit behind an SPI.** Use existing SPIs (e.g., `AgentProvider`) where they exist — add demo-profile implementations, don't create parallel boundaries.

4. **Scenario verification asserts outcomes, not outputs.** Live-mode verification checks downstream state changes (ticket categorized, work item assigned), not non-deterministic LLM text.

5. **Progressive example applications form a capability coverage matrix.** Each slice introduces new platform capabilities. When all capabilities are covered, the scenario tool is also fully exercised.

### What was built

**casehub-examples/helpdesk/** — standalone Quarkus app on branch `issue-408-scenario-engine`:

| Component | What it does |
|-----------|-------------|
| `TicketService` | In-memory CRUD with status lifecycle (OPEN → TRIAGED → ASSIGNED → RESOLVED) |
| `TicketClassifier` SPI | App-local interface for classification |
| `DemoTicketClassifier` | `@Alternative @Priority(300) @IfBuildProfile("demo")` — lookup from scenario data |
| `ChatInjectionResource` | `POST /scenario/inject/chat` — fires `InboundMessage` CDI events via `chat-ref` pipeline |
| `TicketCreationHandler` | `@ObservesAsync ReceivedMessage` → creates + classifies + assigns ticket |
| `NotificationService` | Sends resolution notifications via `ChatPlatform.messaging()` |
| `ScenarioBootstrapResource` | `POST /scenario/bootstrap/helpdesk` — loads classification lookup |
| `VerificationResource` | `GET /scenario/verify/{tickets,notifications}` |
| `help-desk-basic.yaml` | Scenario file for the full pipeline |

5 commits, 12 tests (unit + integration), all green. Uses `chat-ref` (real impl), no unnecessary mocks.

**Workspace specs and plans:**
- `specs/issue-408-scenario-engine/2026-08-12-example-applications-design.md` — design spec
- `specs/issue-408-scenario-engine/decisions.md` — 7 decisions captured
- `plans/2026-08-12-helpdesk-example.md` — implementation plan (completed)

### Issue changes

- **#93 closed** (Demo SPI alternatives — premise wrong)
- **#95 created** (Slice 1: IT help desk example application) — replaces #93 in the queue
- **#311 completed** (Scenario executor backend) — advanced at session start

## Queue State

```
[x] #409 — Scenario format specification (M / Med) [parent]
[x] #410 — Platform protocol — demo SPI convention (S / Low) [parent]
[x] #311 — Scenario executor backend (XL / High) [pages]
[ ] #95  — Slice 1: IT help desk example app (M / Med) [connectors] ← active
[ ] #94  — New SPIs — BankFeedPlatform + EmailPlatform (L / High) [connectors]
[ ] #109 — Life household scenario files (L / High) [life]
[ ] #149 — Migrate DemoDataSeeder to scenario format (M / Med) [clinical]
```

Position: 4/7 (3 done, 4 remaining). Active: #95.

## Immediate Next Step — #95 continuation: Example Showcase UI

The helpdesk backend is complete. What's missing is the **frontend showcase** — a UI built on blocks-ui + pages that:

1. **Gallery framework** — parent app in casehub-examples that lists all examples, each tagged with capability labels. Auto-generates a capability coverage matrix. Filter by capability. Each example's README renders as the lesson content — some examples may also have multi-page guides depending on depth. The gallery aggregates all READMEs/guides into a navigable curriculum.

2. **Per-example UI** — each example is its own web app composing blocks-ui components (execution-monitor, case-explorer, work-item-inbox, channel-activity, etc.) + pages components (charts, tables, metrics). Shows runtime state as the scenario runs — educational, not just functional.

3. **Scenario integration** — the scenario runner drives both UI (`delivery: ui-form`) and REST (`delivery: rest`) paths. The showcase shows scenario progress alongside the runtime state.

### Key components available in blocks-ui

`execution-monitor`, `case-explorer`, `work-item-inbox`, `channel-activity`, `audit-trail-viewer`, `trust-score-panel`, `blocks-timeline`, `blocks-dag-viewer`, `blocks-plan-item-tree`, `routing-rationale`, `sla-indicator`, `commitment-viz`, `kpi-metric-row`

### Architecture questions for next session

- Gallery framework: how do examples register? YAML metadata per example? Auto-discovery?
- Per-example UI: served as `src/main/webui/` inside each example's Quarkus app? Or standalone?
- Content format: README-only for simple examples, multi-page guide for complex ones. Gallery aggregates both into a readable curriculum.
- blocks-ui + pages composition: how do they wire together in a single app?

### Repos needed

- casehub-examples (helpdesk backend — done)
- blocks-ui (UI components — needs adding to slot or working from main checkout)
- pages (pages components — already in slot)

## Slot Repos

| Repo | Role in this epic |
|------|-------------------|
| pages (primary) | Scenario executor backend, pages components |
| parent | Protocol docs (done), capability-ownership |
| connectors | chat-ref (used by helpdesk), issue tracking for #95 |
| life | Household scenario files (#109) |
| clinical | DemoDataSeeder migration (#149) |
| **casehub-examples** | Helpdesk app (new — not in slot, working from main checkout) |
| **blocks-ui** | UI components (new — needs adding for showcase work) |

## Design Principles Established

These emerged from conversation and should carry forward:

1. **Only mock what has no real in-memory alternative.** If the platform has a ref/in-memory implementation, use it.
2. **Demo SPI impls are demand-driven.** Build them inside the apps that need them, extract to shared modules when a second consumer appears.
3. **Every external dependency (connectors AND LLM) swappable via SPI + CDI profile.** Use existing SPIs where they exist.
4. **Verify outcomes, not outputs.** Scenario assertions check state changes, not LLM text.
5. **When you hit a gap, fix the platform.** Example apps are a forcing function for clean SPI design and automatable infrastructure.

## References

- Examples commits: `7e31864..69aa99a` on `issue-408-scenario-engine` branch
- Workspace commits: `616e457`, `045eb77` on `issue-408-scenario-engine` branch
- #93 closed on GitHub (2026-08-12)
- #95 created on GitHub (2026-08-12)
- Decision review: `/Users/mdproctor/reviews/casehub-slots/issue-408-decision-20260812-112939/`
- `.plan` at `/Users/mdproctor/claude/casehub/slots/112/.plan`
