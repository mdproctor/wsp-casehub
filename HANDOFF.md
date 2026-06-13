# Handoff — 2026-06-13

**Head commit (project):** e47908a — docs(#226,#228,#229,#230,#231): sync repo deep-dives
**Head commit (workspace):** e5398d9 — feat: promote diary entry from issue-226-doc-syncs

---

## What Changed This Session

**Five doc sync issues closed** (#226, #228, #229, #230, #231): clinical Layer 8 ActionRiskClassifier partial status + new tutorial layer row + AdverseEvent.unexpected/suspected fields; qhorus ConnectorQhorusMeshBridge description, QhorusInboundCurrentPrincipal corrected to @DefaultBean (+ test exclude-types note), CommitmentDeclinedEvent in domain model, ChannelService.create() initChannel note; PLATFORM.md Named endpoint registry capability row + HTTP inbound row updated; casehub-platform.md endpoints-memory module added; devtown Layer 4 complete + POST /api/incident-feedback endpoint + new domain types + V2003 migration.

**CLAUDE.md Work Tracking format fixed** — bold-markdown format (`**GitHub repo:** value`) caused ctx.py regex to capture `**` as OWNER_REPO instead of `casehubio/parent`. Root cause revised into existing garden entry GE-20260529-182916.

---

## Immediate Next Step

Read the platform evolution research doc and resolve the 3 P0 layering questions before any desiredstate or ras implementation:

1. `SensoryEvent` placement — casehub-platform-api, casehub-ras-api (integration), or new casehub-streams-api?
2. Platform stream module home — submodules of casehub-platform, or separate `casehub-streams` repo?
3. Deployment YAML compiler placement — casehub-desiredstate runtime, casehub-ops/deployment, or new casehub-deploy?

Research doc: `docs/superpowers/research/2026-06-12-platform-evolution-desiredstate-ras-deployment.md`

---

## What's Left

- #210 — Add casehub-rag-api cross-dependency rows to PLATFORM.md — blocked until engine/eidos actually wire the dep · XS · Low
- casehub-ras `.claude/settings.json` needs `Issue tracking: enabled` format fix for ctx.py (`ISSUES_OK=no`) · XS · Low

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Resolve 3 P0 layering questions (§5 of research doc) | S | High | Must precede desiredstate/ras implementation |
| — | Design `StreamContext` SPI for casehub-platform-api | S | Med | Tenancy in async streams — same pattern as CurrentPrincipal |
| — | Design deployment YAML schema (full spec) | M | High | UX story for CaseHub deployments |
| — | casehub-iot implementation session | M | Med | Spec ready, own session |
| #93 | Extract CaseChannelLayout → casehub-engine-api | S | Med | Duplication confirmed |

---

## Key References

- Platform evolution research: `docs/superpowers/research/2026-06-12-platform-evolution-desiredstate-ras-deployment.md`
- casehub-ras spec: `casehubio/casehub-ras/docs/superpowers/specs/2026-06-12-casehub-ras-design.md`
- casehub-desiredstate spec: `casehubio/casehub-desiredstate/docs/superpowers/specs/` (see repo)
- New repos: casehub-desiredstate, casehub-ops, casehub-ras (all at `/Users/mdproctor/claude/casehub/`)
