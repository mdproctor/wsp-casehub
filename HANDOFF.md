# Handoff — 2026-06-07

**Head commit (project):** ccb4847 — chore: allowedTools from stash pop
**Head commit (workspace):** b196163 — feat: promote blog entry from issue-179-doc-sync-batch

---

## What Changed This Session

**15 doc sync issues closed** (two batches: #172–178, #179–191 minus #170). PLATFORM.md, casehub-engine, casehub-ledger, casehub-eidos, casehub-work, casehub-clinical, claudony, casehub-connectors, plus three garden protocols (eidos vocabulary rules) and auth-retrofit-readiness protocol updated.

**Protocol migration completed.** `docs/protocols/` still existed in parent after migration was declared complete — sessions had been silently writing there for weeks. Deleted the directory, updated 27 PLATFORM.md references, swept 8 other docs files. Garden is now the only destination.

**casehub-iot bootstrapped.** New foundation repo created (`casehubio/casehub-iot`), workspace set up (`mdproctor/wsp-casehub-iot`) with bidirectional symlinks, Maven module skeletons (iot-api, iot-homeassistant, iot-openhab, iot-testing, iot-bridge), pre-push hook activated. Spec at `docs/superpowers/specs/2026-06-05-iot-foundation-design.md` in the iot repo. Three ADRs written (parent 0002–0004).

**Home automation brainstorm completed.** Research doc at `docs/superpowers/research/2026-06-05-home-automation-research.md`. Life Layer 9 spec at `docs/superpowers/specs/2026-06-05-life-layer9-home-automation.md` (in parent, for life session to pick up after casehub-iot ships).

**RBAC documented as implemented.** Platform docs updated — infrastructure is ready, adoption pending. Role name convention section added to PLATFORM.md.

**All repos made public** (casehubio/garden, workspace repos).

---

## Immediate Next Step

Start `casehub-iot` implementation session. HANDOFF.md in workspace at `~/claude/public/casehub-iot/` points to the spec. Begin with `iot-api` — the public SPI module.

---

## What's Left

- #170 — casehub-work.md ARC42STORIES.MD sync — BLOCKED on casehubio/work#246 · XS · Low
- cc-praxis `issue-110-router-skills` — README form branch needs PR + merge · XS · Low
- `engine`, `work`, `qhorus`, `ledger` — diverged repos, each needs own session · varies · High

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | casehub-iot implementation (iot-api first) | M | Med | Start here — own session, spec ready |
| — | casehub-life Layer 9 (home automation) | L | High | After casehub-iot ships to GitHub Packages |
| #93 | Extract CaseChannelLayout to engine-api | S | Med | Urgent — duplication confirmed |
| #111 | DID migration across all repos | XL | High | Own session |
| #3 | Automate linked PR chain | M | High | No blockers |

---

## Key References

- casehub-iot spec: `casehubio/casehub-iot` — `docs/superpowers/specs/2026-06-05-iot-foundation-design.md`
- casehub-life Layer 9 spec: `docs/superpowers/specs/2026-06-05-life-layer9-home-automation.md` (this repo)
- Home automation research: `docs/superpowers/research/2026-06-05-home-automation-research.md`
- ADRs 0002–0004: `docs/adr/` (casehub-iot foundation, life as app, three deployment modes)
