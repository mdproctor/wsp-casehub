# HANDOFF — casehub

**Date:** 2026-06-23
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**Docs/build batch + triage: #301, #303, #305, #306, #277 closed; casehub-worker registered**

- **#277** (CloudEvent rollout epic): all 5 sub-issues closed → epic closed. Found missing RAS consumer issue → filed casehub-ras#11 before closing. Auth retrofit issues filed for openclaw (openclaw#41) and clinical (clinical#88).
- **#301–#306** (docs/build batch): casehub-worker registered in modules-core.csv, dashboards, README, index.html; new casehub-worker deep-dive created; PLATFORM.md stale type names fixed (WorkerContext/WorkerSpec/WorkerCapability → Worker/Capability/WorkerFunction, 5 occurrences); openclaw.md deferred note resolved; neural-text JVM-mode-by-design documented.
- **#304** closed as duplicate of #288 (already done).
- **idle eviction**: launchd daemon + Stop hook → `~/.claude_idle/<pid>.touch` timestamps; kills claude sessions idle >1h. Initial scan freed ~4.5 GB (13 processes killed).

## Immediate Next Step

Switch to **casehub-engine session** → engine#543 (Worker primitive migration, 60+ files). This is the critical path for the entire Worker Foundation Extraction chain.

## Cross-Module: Worker Foundation Extraction

Tracking: `casehub-desiredstate#40`

| Step | Repo | Issue | Status |
|------|------|-------|--------|
| 0a | platform | platform#104 | ✅ Done |
| 0b | casehub-worker | — | ✅ Done |
| 1 | parent | parent#288 | ✅ Done |
| **2** | **engine** | **engine#543** | **Next — 60+ file migration** |
| 3 | desiredstate | desiredstate#41 | Blocked on engine#543 |
| 4–7 | claudony, workers, openclaw, ops | 4 issues | Blocked on engine#543 |

## What's Left

- `qhorus#294` — bug: QhorusCloudEventAdapter wrong timestamp (affects Drools CEP) · XS · Low
- `platform#108` — null-guard tenancyId in poll/camel modules · XS · Low
- `work#273` — WorkCloudEventAdapter · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#543 | Worker primitive migration | L | High | Critical path; do in casehub-engine session |
| #251 | Auth retrofit epic — devtown#90, openclaw#41, clinical#88 | S | Low | Each in own session |
| #293 | Formalise channel taxonomy | S | Low | No blockers |
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon |
