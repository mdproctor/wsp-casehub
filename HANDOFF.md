# HANDOFF — casehub

**Date:** 2026-06-22
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**parent#288, #296, #297, #298, #300 closed — BOM + doc syncs**

Three issues completed in sequence:

- **#288** (`build(bom)`): `casehub-worker-api`, `casehub-worker`, `casehub-worker-testing`, `casehub-platform-governance` added to BOM. PLATFORM.md updated (repo map, build order, capability ownership, cross-dep map, Step 4 coherence note). Unblocks `engine#543`.
- **#296–#300** (`docs(repos)`): four deep-dives synced — `engine.md` (ExpressionEngine.extractString SPI + HumanTaskTarget.expiresAtExpression, engine#549), `clinical.md` (Layer 10 IND deadline enforcement, clinical#83), `aml.md` (casehub-engine-flow dep, aml#46), `PLATFORM.md` + `life.md` (auth wired in casehub-life, life#40/#26 closed).

## Immediate Next Step

Next issue: pick up **engine#543** — Worker migration (60+ files). Branch in `casehub-engine` session. Parent BOM now has the artifacts it needs.

## Cross-Module: Worker Foundation Extraction

Tracking: `casehub-desiredstate#40`

| Step | Repo | Issue | Status |
|------|------|-------|--------|
| 0a | casehub-platform | platform#104 | Done (branch needs merge) |
| 0b | casehub-worker | — | Done |
| 1 | **casehub-parent** | **#288** | ✅ Done |
| 2 | casehub-engine | engine#543 | **Next — 60+ file migration** |
| 3 | casehub-desiredstate | desiredstate#41 | Blocked on engine#543 |
| 4–7 | downstream | various | Blocked on engine#543 |

## What's Left

- platform#107 — remove cloudevents version pins from platform root pom · XS · Low
- platform#104 — governance types branch needs merge + publish · XS · Low
- #210 — casehub-rag-api cross-dep rows — blocked until engine/eidos wire the dep · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #277 | CloudEvent infrastructure rollout epic | L | Med | Adapters done; track consumer adoption |
| #293 | Formalise channel taxonomy (CHANNELS.md) | S | Low | Partially done in #93 session |
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon design |
| engine#556 | PerCaseDynamicStrategy design | S | Med | Design only — caseId classification |
| openclaw#40 | Config-driven layout (align with claudony) | S | Low | Follow-up from openclaw#38 |
