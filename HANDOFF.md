# HANDOFF — casehub-platform

**Date:** 2026-08-12
**Project:** `/Users/mdproctor/claude/casehub/platform`
**Workspace:** `/Users/mdproctor/claude/public/casehub/platform`

---

## Last Session

Closed slot 117 (`issue-902-worker-rights-followup`). Platform: fixed `AutoApproveWorkerAuthorizationPolicyTest` for record-based `WorkerAction` (1 commit). Engine: migrated to generalized worker rights types, replaced local `WorkerCredentialFilter` with platform `acl-worker` + `CaseScopeExtractor`, fixed `CaseDefinitionYamlMapper` `permissionIntent` parsing gap (4 commits). Both repos pushed to `casehubio/*` main.

## Cross-Module

**Enabled** (we delivered, downstream unblocked):
- engine consumes `casehub-platform-acl-worker` for worker credential filtering
- engine uses record-based `WorkerAction` + `WorkerAuthorizationContext` from platform-api

## What's Left

- #237: long-lived workers with lifecycle scopes (CASE / STAGE / BINDING)
- #238: JavaBeanCaseFile<T> — typed POJO-backed CaseContext
- MongoDB backend for subject view toolkit — not yet filed
