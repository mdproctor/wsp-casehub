# Handoff — 2026-06-19

*Updated by casehub-desiredstate session — cross-repo handoff for Worker Foundation Extraction.*

**Head commit (project):** 44741da — docs(#236–#250): sync repo deep-dives — batch doc sync
**Head commit (workspace):** *see below*

---

## What Changed (cross-repo, from desiredstate session 2026-06-18/19)

**Worker Foundation Extraction** (desiredstate#40): Worker primitives extracted from casehub-engine-api to a new foundation-tier `casehub-worker` repo. Execution governance types added to casehub-platform-api with PolicyEnforcer in a new `governance/` submodule.

**New artifacts available in local Maven repo (0.2-SNAPSHOT):**
- `casehub-platform-api` — now includes `io.casehub.platform.api.governance.{ExecutionPolicy, RetryPolicy, BackoffStrategy}`
- `casehub-platform-governance` — new submodule with `PolicyEnforcer` + `DefaultPolicyEnforcer`
- `casehub-worker-api` — `Worker`, `WorkerFunction`, `Capability`, `WorkerResult`, `WorkerOutcome`
- `casehub-worker` (runtime) — `WorkerExecutor` (interface) + `DefaultWorkerExecutor`
- `casehub-worker-testing` — `MockWorkerExecutor` (@DefaultBean) + `TestWorkerBuilder`

**Platform changes on branch `issue-104-governance-types`** — needs merge to main + publish.

---

## Immediate Next Step

**parent#288** — Add casehub-worker and casehub-platform-governance artifacts to the BOM (`pom.xml` dependencyManagement) and update `docs/PLATFORM.md`:
1. BOM: `casehub-worker-api`, `casehub-worker`, `casehub-worker-testing`, `casehub-platform-governance`
2. PLATFORM.md: Repository Map (casehub-worker, Foundation tier), Build Order (after platform, before ledger), Capability Ownership (Worker primitives), Cross-Repo Dependency Map (worker-api consumed by engine, desiredstate)
3. Add `casehub-worker` to peer repos list in this CLAUDE.md

This unblocks engine#543 (Worker migration — 60+ files).

---

## What's Left

- parent#288 — BOM + PLATFORM.md update for casehub-worker · S · Low
- #210 — Add casehub-rag-api cross-dependency rows to PLATFORM.md — blocked until engine/eidos wire the dep · XS · Low

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Resolve 3 P0 layering questions (§5 of research doc) | S | High | Must precede desiredstate/ras implementation |
| — | Design `StreamContext` SPI for casehub-platform-api | S | Med | Tenancy in async streams |
| — | Design deployment YAML schema (full spec) | M | High | UX story for CaseHub deployments |
| — | casehub-iot implementation session | M | Med | Spec ready, own session |
| #93 | Extract CaseChannelLayout → casehub-engine-api | S | Med | Duplication confirmed |

---

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

---

## Key References

- Design spec: `casehub-desiredstate/docs/superpowers/specs/2026-06-18-worker-foundation-extraction-design.md`
- Implementation plan: `casehub-desiredstate/docs/superpowers/plans/2026-06-18-worker-foundation-extraction.md`
- Platform evolution research: `docs/superpowers/research/2026-06-12-platform-evolution-desiredstate-ras-deployment.md`
