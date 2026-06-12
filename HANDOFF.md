# Handoff — 2026-06-12

**Head commit (project):** 0f70c54 — docs: platform evolution research
**Head commit (workspace):** aecaa02 — docs: promote casehub-ras idea

---

## What Changed This Session

**Massive doc sync batch** — closed 25 issues across three days (#170, #192–#225). All were doc-only updates to PLATFORM.md, per-repo deep dives, and garden protocols.

**Three new repos bootstrapped** — casehub-desiredstate, casehub-ops, casehub-ras. All have: GitHub repos, Maven POM structure, CLAUDE.md, local clones, workspace repos, bidirectional symlinks, `.claude/settings.json` (no env.PATH), and initial issues/epics.

**casehub-ras** — new repo for Reticular Activating System: situational awareness and reactive case creation. Ganglia (pluggable detection strategies) feed into a RAS engine that triggers cases. Service lifecycle management pattern documented: long-lived service case + RAS monitoring + child incident/upgrade/decommission cases.

**Platform evolution research doc** — comprehensive cross-cutting research at `docs/superpowers/research/2026-06-12-platform-evolution-desiredstate-ras-deployment.md` covering: desiredstate + ras architecture, deployment YAML UX concept, self-governance property, stream/data first-class citizenship, and layering analysis.

**README badges** — CI + PR badges added to all 19 casehub repos via gh api.

---

## Immediate Next Step

Read the platform evolution research doc. Three P0 layering questions must be answered before any implementation of desiredstate or ras begins:

1. **`SensoryEvent` placement** — casehub-platform-api, casehub-ras-api (integration), or new casehub-streams-api? Foundation modules can't depend on integration tier — this blocks all platform stream modules.
2. **Platform stream module home** — submodules of casehub-platform, or separate `casehub-streams` repo?
3. **Deployment YAML compiler placement** — in casehub-desiredstate runtime, casehub-ops/deployment, or new casehub-deploy?

Research doc: `docs/superpowers/research/2026-06-12-platform-evolution-desiredstate-ras-deployment.md`

---

## What's Left

- #210 — Add casehub-rag-api cross-dependency rows to PLATFORM.md — blocked until engine/eidos actually wire the dep · XS · Low
- #202, #204 — protocol updates blocked on life#27 (now closed — recheck next session) · XS · Low
- casehub-ras `.claude/settings.json` needs `Issue tracking: enabled` format fix for ctx.py (currently `ISSUES_OK=no`) · XS · Low

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Resolve 3 P0 layering questions (see research doc §5) | S | High | Must precede any desiredstate/ras implementation |
| — | Design `StreamContext` SPI for casehub-platform-api | S | Med | Tenancy in async stream processing — same pattern as CurrentPrincipal |
| — | Design deployment YAML schema (full spec) | M | High | The UX story for CasehHub deployments |
| — | casehub-iot implementation session | M | Med | Spec ready, own session |
| #93 | Extract CaseChannelLayout → casehub-engine-api | S | Med | Urgent — duplication confirmed |

---

## Key References

- Platform evolution research: `docs/superpowers/research/2026-06-12-platform-evolution-desiredstate-ras-deployment.md`
- casehub-ras spec: `casehubio/casehub-ras/docs/superpowers/specs/2026-06-12-casehub-ras-design.md`
- casehub-desiredstate spec: `casehubio/casehub-desiredstate/docs/superpowers/specs/` (see repo)
- New repos: casehub-desiredstate, casehub-ops, casehub-ras (all at `/Users/mdproctor/claude/casehub/`)
