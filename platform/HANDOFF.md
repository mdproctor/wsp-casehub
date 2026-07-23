# HANDOFF — Slot 30: Retire Reactive Tiers (#384)

**Issue:** casehubio/parent#384
**Slot:** `/Users/mdproctor/claude/casehub/worktrees/30/`
**Branch:** `issue-384-retire-reactive` (all repos)
**Cookbook:** `engine/docs/guides/virtual-thread-migration.md`

## What's Done

| Repo | Status | PR |
|------|--------|----|
| **platform** | Merged | casehubio/platform#194 |
| **ras** | PR open | casehubio/casehub-ras#54 |
| **ops** | PR open | casehubio/casehub-ops#63 |
| **desiredstate** | PR open | casehubio/casehub-desiredstate#88 |
| **iot** | PR open | casehubio/iot#70 |

### Platform (merged)
- Deleted `ReactiveNotificationStore`, `ReactiveSubscriptionStore` SPIs + all impls (NoOp, InMemory, JPA reactive)
- Deleted `ReactiveAgentIdentityVerificationService` identity bridge
- Rewrote `JpaNotificationStore` and `JpaSubscriptionStore` from Hibernate Reactive Panache to standard JPA (`EntityManager` + `@Transactional`)
- Converted `NotificationResource` + `SubscriptionResource` to `@RunOnVirtualThread`
- Converted `NotificationSseResource` — blocking store + `Executors.newVirtualThreadPerTaskExecutor()` offload (per `sse-endpoint-no-virtual-thread` protocol)
- POMs: `quarkus-hibernate-reactive-panache` → `quarkus-hibernate-orm`

### ras, ops, desiredstate, iot (PRs open)
- Done in a prior session (Phase A complete before this session started)
- Phase B completed this session: rebased, ff-merged, pushed forks, opened PRs
- Check CI status before merging — all should be green

## What's Left

| Repo | Scope | Complexity |
|------|-------|------------|
| **qhorus** | ~15 dual-stack pairs + services + `@IfBuildProperty` gating + `QhorusBuildTimeConfig.reactive()` + 42 `@Blocking` annotations | **Heavy** — full session |
| **neocortex** | ~10 dual-stack pairs + full decorator chain (9 reactive decorators) + bridges | **Heavy** — full session |
| **ledger** | Consumes engine SPIs — reactive usage retires when engine SPIs change | Medium |
| **eidos** | Consumes engine SPIs — has `ReactiveRenderedPromptCache` (garden protocol `reactive-rendered-prompt-cache-canonical-spi` becomes moot) | Medium |
| **connectors** | Unknown scope — audit needed | Unknown |
| **claudony** | Implements engine Reactive* SPIs — switch to blocking | Small–Medium |
| **openclaw** | Implements engine Reactive* SPIs — switch to blocking | Small |
| **blocks** | Implements engine Reactive* SPIs — switch to blocking | Small |

### Recommended order
1. Merge the 4 open PRs (ras, ops, desiredstate, iot) — check CI first
2. Small app repos (claudony, openclaw, blocks) — mechanical, follow cookbook
3. Medium repos (ledger, eidos, connectors) — depend on engine #381 landing first
4. Heavy repos (qhorus, neocortex) — dedicated sessions each

### Dependency: engine #381
The issue says ledger, eidos, and work repos' reactive usage retires when engine SPIs change (#381). Check whether #381 has landed before touching those repos. If not, they may need to wait.

## Artifacts Created This Session

- **Protocol:** `sse-endpoint-no-virtual-thread` in garden — SSE endpoints must not use `@RunOnVirtualThread`
- **Migration guide update:** `engine/docs/guides/virtual-thread-migration.md` §7 + §9 — SSE warning + common mistake
- **Garden entry:** GE-20260723-fbbdb6 — IntelliJ MCP worktree mismatch gotcha
- **Plan:** `plans/2026-07-23-retire-reactive-platform.md` (workspace)
- **Diary entries:** `2026-07-23-mdp01-reactive-layer-already-dead.md`, `2026-07-23-mdp02-thirteen-repos-one-branch.md`

## Key Decisions

- `@RunOnVirtualThread` on regular REST endpoints, NOT on SSE endpoints (protocol)
- SSE blocking calls offloaded via `Executors.newVirtualThreadPerTaskExecutor()` static field
- `@ObservesAsync` CDI handlers can block directly (managed executor threads)
- JPA stores use named parameters (`:param`) not positional (`?1`) — matches platform convention
- Entity classes use plain `@Entity` not `PanacheEntityBase` after migration (notifications-jpa, subscriptions-jpa)
- `findAllEnabled()` uses eager `getResultList().stream()` not lazy `getResultStream()` — code review catch

## IntelliJ Worktree Warning

When working in git worktrees, IntelliJ MCP operates on whichever project is open — not necessarily the worktree. Must call `ide_open_project` with the worktree's absolute path first, then verify with `ide_project_status` that module paths point to the worktree. See GE-20260723-fbbdb6.
