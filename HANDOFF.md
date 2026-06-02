# Handoff — 2026-06-02

**Head commit (project):** 92d6fce — feat(parent#56): merge issue-56-actor-state-view
**Head commit (workspace):** 410299f — chore(issue-56-actor-state-view): remove promoted spec

---

## What Changed This Session (2026-06-02)

**parent#56 implemented and closed** — actor state view across 5 repos:
- `casehub-platform-api`: `ActorStateContributor` + `ActorStateAccumulator` SPI (stdlib types only; ADR-0001 filed)
- `casehub-engine-actor-state`: new module — 4 contributors, blocking + reactive aggregators, `GET /actors/{actorId}/state`
- `casehub-qhorus`: `CommitmentStore.findOpenByObligor(obligor)` + `ChannelStore.findByIds(ids)` + contract tests
- `casehub-ledger`: `TrustGateService.allCapabilityScores(actorId)`
- `casehub-work`: `WorkItemCallerRef.parseCaseId(callerRef)` utility
- `casehub-engine`: `WorkerExecutionManager.getActiveCaseIds(workerId)` + `CaseChannel.parseCaseId(channelName)`
- 3 new platform protocols: PP-20260601-c60b28 (cross-backend aggregation), PP-20260601-81b9e5 (SPI default methods), reactive consumer contract extension
- PLATFORM.md, docs/repos/*.md, docs/adr/INDEX.md updated

**Also closed:** parent#115 (AML trust routing → PreferenceKey), parent#47 (workspace path cleanup), parent#5 (declared already resolved).

**Deferred:**
- devtown pom addition for actor-state — branch issue-249 active; add when it lands on main
- `casehubio/qhorus#229` — index on commitment.obligor (performance)
- `casehubio/engine#413` — test coverage gaps (partial-write + deleted-channel)

---

## Immediate Next Step

Add `casehub-engine-actor-state` to devtown `app/pom.xml` once `issue-249-arc42stories-sync` lands on devtown main.

Until then: `clinical` blog recovery — recover stranded blog from `epic-3-multi-site-sub-case`, merge to clinical workspace main, publish.

---

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Left

- `devtown` — add `casehub-engine-actor-state` dep to `app/pom.xml` (waiting on issue-249) · XS · Low
- `clinical` — recover stranded blog from `epic-3-multi-site-sub-case` · XS · Low
- CLAUDE.md mass update (ARC42STORIES.MD references, LAYER-LOG retirement) — cross-repo, each repo's own session
- `parent#123` — per-repo hygiene items (engine, claudony, ledger, platform, eidos, connectors, clinical, work, flow)

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #93 | Extract normative channel layout from claudony → engine-api | M | Med | Blocked by claudony#142 |
| #106 | Update spec §7.1 oversight channel allowedTypes | XS | Med | Blocked by claudony#142 |
| #56 | Unified actor state view | L | High | ✅ Done this session |
| #3 | Automate linked PR chain across ecosystem repos | L | High | — |
| #111 | Migrate actorId format to DID across all repos | XL | High | Needs ledger#108/110 first |
| #7 | Platform foundation roadmap tracker | XL | High | Tracker — closes last |
| #13 | Cohesive Claude config design | XL | High | Needs dedicated design session |

---

## Key References

- parent#56 merged: 92d6fce (project main), blog: `2026-06-02-mdp01-four-backends-one-question.md`
- ADR-0001: actor state SPI in platform-api — `docs/adr/0001-actor-state-spi-in-platform-api.md`
- Garden: 6 new entries (GE-20260602-6cfbdb through GE-20260602-c4a68a)
