# HANDOFF — Slot 112: Cross-Platform Scenario Engine

**Date:** 2026-08-12
**Branch:** `issue-408-scenario-engine`
**Slot:** `/Users/mdproctor/claude/casehub/slots/112`
**Epic:** casehubio/parent#408

---

## Last Session

Closed #410 (Platform protocol — demo SPI convention). The previous session had already committed both protocol documents; this session verified completeness, advanced the plan, and closed the GitHub issue.

**Completed issues (committed to parent repo, branch `issue-408-scenario-engine`):**

1. **#409 — Scenario format specification** (`f9580ce0`)
   - `parent/docs/platform/scenario-format.md` — 775 lines covering YAML schema, delivery modes (rest/ui-form/simulated), trigger types (time/after/data), data shapes (single/bulk/stepped/stream), UI action primitives, speed control with fast-fallback, await/verification model, error policy, and TypeScript vocabulary mapping to existing `ScenarioController` in pages-data.
   - Complete examples: REST bulk seed, UI form fill, simulated event injection, continuous stream, mixed multi-delivery scenario.

2. **#410 — Platform protocol — demo SPI convention** (`999beec6`)
   - `parent/docs/platform/demo-spi-convention.md` — 347 lines covering profile convention (demo/dev/prod), CDI annotation pattern (`@Alternative @Priority(300) @IfBuildProfile("demo")`), module placement, pull mode (bootstrap endpoint + pre-loaded data), push mode (injection endpoints firing CDI events), shared `DemoCurrentPrincipal` in platform-api, priority allocation table, and new-connector checklist.
   - `parent/docs/platform/protocols.md` updated — line 43 references demo SPI convention.
   - `parent/docs/platform/capability-ownership.md` updated — line 71 has scenario engine entry.

## Queue State

```
[x] #409 — Scenario format specification (M / Med) [parent]
[x] #410 — Platform protocol — demo SPI convention (S / Low) [parent]
[ ] #311 — Scenario executor backend (XL / High) [pages] ← active
[ ] #93  — Demo SPI alternatives — ChatPlatform + CalendarPlatform (M / Med) [connectors]
[ ] #94  — New SPIs — BankFeedPlatform + EmailPlatform (L / High) [connectors]
[ ] #109 — Life household scenario files + conversational intake (L / High) [life]
[ ] #149 — Migrate DemoDataSeeder to scenario format (M / Med) [clinical]
```

Position: 2/7 (2 done, 5 remaining). Next: #311.

## Immediate Next Step — #311: Scenario Executor Backend

**Repo:** casehub-pages (Java backend at `backend/`)
**Scale:** XL / **Complexity:** High
**GitHub:** casehubio/casehub-pages#311

Build the `ScenarioExecutor` — the Quarkus backend in casehub-pages that:
1. Parses scenario YAML files into a `TriggerGraph` (DAG of steps)
2. Schedules step execution based on trigger dependencies (time, after, data-poll)
3. Delivers steps via three modes:
   - `rest` — HTTP calls to target service APIs
   - `simulated` — POST to `/scenario/inject/{connector}` on target services
   - `ui-form` — dispatches UIAction sequences to frontend via ControlChannel
4. Manages speed control (0.5x demo to 100x verification)
5. Implements the bootstrap sequence (health check → POST /scenario/bootstrap → playback)
6. Produces JUnit XML in verify mode

### Key design inputs (all committed files — read these first)

| File | What it covers |
|------|---------------|
| `parent/docs/platform/scenario-format.md` | YAML schema, delivery modes, triggers, data shapes, UI actions, error model, verification mode, TypeScript type mapping |
| `parent/docs/platform/demo-spi-convention.md` | Profile convention, CDI annotations, bootstrap/injection endpoints, DemoCurrentPrincipal, priority allocation |
| `parent/docs/platform/capability-ownership.md` (line 71) | Executor placement, repo responsibilities, cross-cutting concerns |

### Existing code to integrate with

- `pages-data/src/datasource/controller.ts` — `ScenarioController` (client-side virtual-time queue, speed/pause control) from pages#140
- `backend/push/` — `casehub-pages-push` module (typed wire protocol SDK, TopicRegistry, EventStore)
- `backend/push-runtime/` — CDI producers for EventBroadcaster, TopicRegistry

### Architecture decisions already made

- Executor lives in pages backend, not a separate repo
- Build profile `demo` is a compile-time gate (`@IfBuildProfile`), not a runtime flag
- Trigger evaluation is server-side — no client-side DataSet involvement for triggers
- `DemoCurrentPrincipal` is shared in `casehub-platform-api`, not per-app
- Speed control: `fast` mode uses `fast-fallback` for ui-form steps; steps without fallback are skipped
- Error model: three policies (continue/stop/pause) with dependent-step skipping

### Design spec

The epic (#408) references a design spec at:
`specs/2026-08-11-cross-platform-scenario-engine-design.md` in the life workspace.
Check `/Users/mdproctor/claude/casehub/slots/112/life/docs/specs/` or the life workspace for it.

## Slot Repos

| Repo | Role in this epic |
|------|-------------------|
| pages (primary) | Scenario executor backend, YAML parser, ControlChannel integration |
| parent | Protocol docs (done), capability-ownership updates |
| connectors | Demo SPI implementations (#93, #94) |
| life | Household scenario files (#109) |
| clinical | DemoDataSeeder migration (#149) |

## Blog

Written: `docs/blog/2026-08-12-mdp01-the-demo-that-tests-itself.md` — covers the dual-purpose nature of scenario files (demo + verification), three delivery modes, trigger graphs, and the build-profile convention.

## References

- Parent commits: `999beec6` (#410), `f9580ce0` (#409)
- #410 closed on GitHub (2026-08-12)
- #409 closed on GitHub (previous session)
- Epic #408 open — 5 issues remaining
- `.plan` at `/Users/mdproctor/claude/casehub/slots/112/.plan`
