# HANDOFF — casehub

**Date:** 2026-06-21
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**#276 closed:** Added `cloudevents-core:4.0.1`, `cloudevents-api:4.0.1`, and `cloudevents-json-jackson:4.0.1` to the BOM under a new third-party version pins section. Triggered by `iot#19`, `qhorus#279`, `connectors#20` all shipping their CloudEvent adapters. Also cleaned up tracked `.claude/` system files (`scheduled_tasks.lock`, `settings.local.json`) from both the workspace and parent repos — added to `.gitignore` and untracked.

Separate session (casehub-platform): comprehensive ARC42STORIES.MD audit — added C19 (AgentSession + LangChain4j), C20 (Stream Ingestion), C21 (ACL Phase 1), L10/L11 layer entries, 15 glossary terms, stale §12 fixes. Pushed to both remotes.

## Immediate Next Step

**parent#288** — Add casehub-worker and casehub-platform-governance artifacts to the BOM + update `docs/PLATFORM.md`. This unblocks `engine#543` (Worker migration, 60+ files).

BOM entries needed: `casehub-worker-api`, `casehub-worker`, `casehub-worker-testing`, `casehub-platform-governance`.
PLATFORM.md: add casehub-worker to Repository Map (Foundation tier), Build Order, Capability Ownership, Cross-Repo Dependency Map.

## Cross-Module: Worker Foundation Extraction

**Tracking issue:** casehub-desiredstate#40

| Step | Repo | Issue | Status |
|------|------|-------|--------|
| 0a | casehub-platform | platform#104 | Done (branch `issue-104-governance-types`, needs merge) |
| 0b | casehub-worker | — | Done (repo created, published) |
| 1 | **casehub-parent** | **parent#288** | **Next — BOM + PLATFORM.md** |
| 2 | casehub-engine | engine#543 | Blocked on parent#288 |
| 3 | casehub-desiredstate | desiredstate#41 | Blocked on engine#543 |
| 4 | claudony | claudony#157 | Blocked on engine#543 |
| 5 | casehub-workers | workers#14 | Blocked on engine#543 |
| 6 | casehub-openclaw | openclaw#37 | Blocked on engine#543 |
| 7 | casehub-ops | ops#8 | Blocked on engine#543 |

## What's Left

- platform#104 — governance types branch needs merge to main + publish · S · Low
- #210 — casehub-rag-api cross-dep rows in PLATFORM.md — blocked until engine/eidos wire the dep · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #288 | casehub-worker + governance BOM + PLATFORM.md | S | Low | Immediate — unblocks engine#543 |
| #277 | CloudEvent infrastructure rollout epic | L | Med | #276 done; track consumer adoption |
| #293 | Formalise channel taxonomy (CHANNELS.md) | S | Low | Recent docs commit landed |
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon design |

## Key References

- Design spec: `casehub-desiredstate/docs/superpowers/specs/2026-06-18-worker-foundation-extraction-design.md`
- Implementation plan: `casehub-desiredstate/docs/superpowers/plans/2026-06-18-worker-foundation-extraction.md`
