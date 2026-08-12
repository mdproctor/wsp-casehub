# HANDOFF — casehub-platform (slot 117)

**Date:** 2026-08-12
**Slot:** 117 — issue-902-worker-rights-followup
**Branch:** `issue-902-worker-rights-followup`
**Repos:** platform (primary), engine

---

## Last Session

Migrated the engine to the generalized worker rights SPI types from platform commit `cafb326`. Created `EngineWorkerActions` constants class and `EngineAuthorizationContext` record, updated `WorkerGrantOrchestrator` to use `ResourceId` and `WorkerAuthorizationContext`, replaced the engine's local `WorkerCredentialFilter` with platform's `acl-worker` module + `CaseScopeExtractor`. Fixed a pre-existing gap where `CaseDefinitionYamlMapper` never parsed `permissionIntent` from YAML — every worker dispatch silently defaulted to read-only. Also fixed platform's `AutoApproveWorkerAuthorizationPolicyTest` missed in `cafb326`.

## Immediate Next Step

Run `/work` to continue on this branch. #902 is complete — advance to #237 (long-lived workers with lifecycle scopes) or #238 (JavaBeanCaseFile typed POJO-backed CaseContext).

## Cross-Module

**Enabled** (we delivered, downstream unblocked):
- engine now consumes `casehub-platform-acl-worker` — engine#902 changes need to land on engine main before other engine work that touches worker rights
