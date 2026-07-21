# HANDOFF — engine-rest extraction (#762)

**Date:** 2026-07-21
**Branch:** issue-762-engine-rest (slot 7)
**Status:** Engine PR open, scaffold follow-up filed

## What was done

**casehub-engine-rest module** extracted from scaffold — first module built on
the virtual thread direction.

### Engine changes (landed on local main, PR #766 open — CI green)

1. **SPI pagination layer** — `CaseDefinitionQuery`, `CaseInstanceQuery`,
   `EventLogQuery` following work's `AuditQuery` pattern. Default `query()`/`count()`
   methods on `CaseMetaModelRepository`, `CaseInstanceRepository`, `EventLogRepository`.
   Implemented in both `persistence-memory` and `persistence-hibernate`. 20 contract tests.

2. **casehub-engine-rest module** — opt-in JAX-RS library:
   - 5 resources: CaseDefinition, CaseInstance, CaseControl, Signal, EventLog
   - 9 DTOs with Bean Validation + OpenAPI annotations
   - `CaseService` for multi-step flows (startCase, requireCase)
   - 5 exception mappers producing RFC 7807 ProblemDetail
   - All `@RunOnVirtualThread`, blocking SPIs, no Uni, no Panache, no ACL
   - 5 `@QuarkusTest` integration tests with test stubs

3. **BOM fix** — added `casehub-connectors-core` to engine `dependencyManagement`,
   removed duplicate `casehub-platform-api` entry.

### Design decisions made during brainstorming

- **Virtual threads over reactive** for REST — `@RunOnVirtualThread` + blocking SPIs.
  Platform direction confirmed (parent ADR already in flight, Phase 1 pilot running).
- **No Panache in REST** — all data access through SPIs. SPIs abstract the database
  (PostgreSQL, MongoDB, H2, in-memory).
- **No ACL in REST library** — domain authorization belongs at API level, not HTTP layer.
  Scaffold retains ACL as deployment config via `ContainerRequestFilter`.
- **Pagination at SPI level** — follows work's `AuditQuery` pattern. No in-memory
  pagination over unbounded lists.

### Design spec

Adversarially reviewed (4 rounds, 24 issues, all resolved):
`docs/specs/2026-07-20-engine-rest-design.md`

## What's next

### Scaffold follow-up (casehubio/scaffold#35)

Replace scaffold's inline REST with `casehub-engine-rest` dependency:
- Delete `io.casehub.flow.rest.*`, `io.casehub.flow.service.*`, `io.casehub.flow.exception.*`
- Add `ContainerRequestFilter` for ACL
- Update test imports to `io.casehub.engine.rest.dto.*`
- Depends on engine PR #766 being merged upstream

### Engine PR #766

- CI green, PR open at https://github.com/casehubio/engine/pull/766
- Needs merge to upstream `casehubio/engine`

## Slot state

- Worktree slot 7: branch `issue-762-engine-rest` stamped as closed
- Engine main: commit `12229f81` + BOM fix `cfd8a71a`
- Engine-rest jar installed to `~/.m2` — other repos can resolve it
- Scaffold branch `issue-762-scaffold-engine-rest` created but empty (no changes yet)
