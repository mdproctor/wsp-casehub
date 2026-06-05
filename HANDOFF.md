# Handoff — 2026-06-05

**Head commit (project):** facb861 — chore: remove quarkus-langchain4j from CI/CD
**Head commit (workspace):** 66c4ab1 — docs: correct #158 description in parent handover

---

## What Changed This Session

**26-issue triage.** Reviewed every open issue in casehubio/parent for validity, scoping, and blocker status. #106 and #111 were newly unblocked (claudony#142 and ledger#81 both closed since last session). #163 was already committed but not auto-closed by GitHub.

**14 issues closed.** Batch doc-sync commit (6680ad3) closed #153, #159–#162, #165–#169, #171. Then #106 (oversight channel type), #148 (trust threshold split code/YAML), #163 (qhorus sync already done).

**PLATFORM.md gap analysis.** Used Explore subagent to scan all docs/repos/*.md deep-dives against capability ownership. Added 8 missing rows: LedgerEnricherPipeline, ActorDIDProvider, HumanParticipatingChannelBackend, Qhorus MCP surface, fleet management, claudony MCP server, OversightGateService, ActionRiskClassifier (both flagged with intended home: engine-api). Added Known Placement Violations section and application-tier notification SPI pattern to Step 4.

**Worker(Workflow) promoted.** Added casehub-engine-flow capability row and Step 4 guidance calling it the preferred pattern for durable multi-step workers.

**arc42stories-readme.md written.** General-purpose (not casehub-specific). Covers origins, what it adds, C4 diagrams rationale, epic mapping. Lives at docs/arc42stories-readme.md.

**langchain4j retired.** All fixes merged upstream. Removed from CI/CD workflows, README badge, CLAUDE.md peer repos list.

---

## Immediate Next Step

File the `OversightGateService` tracking issue in casehubio/engine (listed in PLATFORM.md Known Placement Violations as "untracked — file issue before implementing"). Then work on parent#93 (CaseChannelLayout extraction to engine-api) — openclaw#6 closing confirmed the duplication has materialized.

---

## What's Left

- `clinical`, `devtown`, `drafthouse` — upstream delivery outstanding · XS · Low
- Workspace branch cleanup — `issue-12` through `issue-19` deletion (overdue) · XS · Low
- `engine`, `work`, `qhorus`, `ledger`, `flow` — diverged repos, each needs own session · varies · High
- `platform`, `quarkmind` — add upstream remote · XS · Low
- cc-praxis `issue-110-router-skills` — README form branch needs PR + merge · XS · Low
- #170 — casehub-work.md ARC42STORIES.MD sync — BLOCKED on casehubio/work#246 · XS · Low

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #93 | Extract CaseChannelLayout to engine-api | S | Med | Urgent — openclaw#6 closed, duplication confirmed |
| #111 | DID migration across all repos | XL | High | Unblocked (ledger#81 closed) — own session |
| #3 | Automate linked PR chain | M | Med | No blockers |
| #13 | Cohesive Claude config design | L | Med | No blockers |
| #154–#157 | AI modules (observability, policy, eval, artifacts) | M–L | High | Each needs own session |

---

## Key References

- PLATFORM.md: `docs/PLATFORM.md` — Known Placement Violations section added
- arc42stories README: `docs/arc42stories-readme.md`
- Ideas: `~/claude/public/casehub/IDEAS.md` — ARC42STORIES planning loop, build badges
