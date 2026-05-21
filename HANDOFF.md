# Handoff — 2026-05-21

**Head commit (parent):** 11955dd — fix(protocol): Rule 4 path must be outside db/migration/
**Head commit (workspace):** 54bafbb — docs: add blog entry 2026-05-21 — Cleaning the Board

## What Changed This Session

Closed parent#30, #11, #6 and #34 (path detection unification).

- **parent#30** — protocol PP-20260520-650742: dedicated `@ApplicationScoped` writer bean for shared `LedgerEntry.sequenceNumber` ownership. `docs/protocols/casehub/harness-ledger-writer.md`.
- **parent#11** — protocol PP-20260520-98b57d: strategy SPIs must pass `caseId` through all method signatures. Verified all listed SPIs already comply. `docs/protocols/casehub/spi-case-id-parameter.md`.
- **parent#6** — SLA propagation: WorkItem `expiresAt` bounded by `PropagationContext.deadline` in `casehub-engine-work-adapter`. Engine commit `da3c41a` on main. Claudony side deferred — claudony never passes `correlationId` to Qhorus so no Commitments are opened. Issues filed: engine#300, claudony#122.
- **parent#34** — Path detection unified across all workspace-aware cc-praxis skills. `readlink -f proj` + `git rev-parse --show-toplevel` replaces CLAUDE.md grepping. Canonical block documented in cc-praxis CLAUDE.md. All 13 repo symlinks verified complete.

**Open workspace branches:**
- `issue-6-sla-propagation` — work-end not run; engine changes landed separately via direct commit. Needs cleanup.
- `epic-atomic-human-task` — engine branch exists (issue #273), 3 days old.
- `epic-io-thread-safety` — no issue number, 3 days old. Investigate.

## Immediate Next Step

Clean up `issue-6-sla-propagation` workspace branch (run work-end or discard). Then tackle #33 or the protocol cluster.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #33 | Update PLATFORM.md: engine-work-adapter + blackboard in devtown deps | XS | Low | Doc only |
| #22 | Update casehub-clinical.md layer table (L1/2/4 done, L3 in progress) | XS | Low | Doc only |
| #23 | Add casehub-qhorus direct dep to clinical cross-repo table | XS | Low | Doc only |
| #29 | Protocol: InboundNormaliser channel scope rule | S | Low | Protocol file only |
| #28 | Protocol: per-entity governance channel naming | S | Low | Protocol file only |
| #32 | Protocol: reactive vs blocking execution model selection | S | Med | Needs design thought |
| #19 | Protocol: casehub-work Hibernate scan packages | XS | Low | Protocol file only |
| #26 | Design: zero-dep platform-api module | L | High | Multi-repo |
| engine#300 | Add deadline to COMMAND content JSON in dispatchCommand() | XS | Low | Prerequisite for claudony#122 |
| claudony#122 | Extract correlationId + deadline from COMMAND content | S | Med | Depends on engine#300 |

## Key References

- Blog: `blog/2026-05-21-mdp01-cleaning-the-board.md`
- Garden: GE-20260521-50acf0 (grep-c double output), GE-20260521-53dae7 (git stash exits 0), GE-20260521-523b94 (A&&B||C not if/else)
