# Handoff — Full Stack Build Fix and Claude Config
2026-05-12

## What changed this session

**Full-stack platform build now passes.** `mvn install -f aggregator.xml` runs clean across all foundation repos. Four root causes fixed:

1. `findBySubjectIdAndTimeRange` missing from `JpaWorkItemLedgerEntryRepository` (work#163), `MessageLedgerEntryRepository` + reactive variant (qhorus#143), and 4× test `NoOpLedgerEntryRepository` in engine (engine#242) — ledger SPI change not propagated
2. `casehub-engine-common` missing serverlessworkflow deps needed by `WorkflowExecutor` interface (engine#242)
3. `ClaudonyCaseChannelProvider` missing `postToChannel(CaseChannel, String, String, MessageType)` 4-arg overload
4. Claudony MCP tool count assertion updated 59→60 (Qhorus added one tool)

All fixes pushed to mdproctor (origin) and casehubio (upstream) for work, qhorus, claudony. Engine fixes are on `main_proposal` branch — see message in conversation history for engine Claude.

**New protocols added:**
- `docs/protocols/ledger-spi-propagation.md` — checklist for propagating LedgerEntryRepository SPI changes across all downstream implementations (work, qhorus, engine test sources)
- `docs/protocols/module-tier-structure.md` — three-tier rule (pure-Java SPI / core library / full extension) with refinement: SPI method signatures must not expose heavy external SDK types

**Claude config:**
- `~/.claude/design-implementation.md` — IntelliJ operation→tool table, config-architecture pointer
- `~/.claude/prompt-snippets.md` — general session snippet (new); casehubio-specific in `docs/prompt-snippets.md`
- `~/.claude/config-architecture.md` — canonical topic ownership map, refreshed daily by `update-claude-md`
- `ide-tooling` skill created (cc-praxis) — single authoritative IntelliJ MCP guide
- `update-claude-md` skill — Step 0 refreshes config-architecture.md daily from GitHub
- All casehubio project CLAUDE.md files — Development Workflow section added (brainstorm/TDD/review hooks + living docs list)
- PLATFORM.md restored (had been emptied twice), expanded with never-dogma language, dev session protocol pointer, protocols/ references

**Workspaces:**
- All 9 casehub repos now have workspaces at `~/claude/public/casehub/<repo>/`
- Bidirectional `proj`/`wksp` symlinks in all repos
- Session histories migrated; casehub-poc blogs moved to engine workspace
- casehub-aml and casehub-clinical bootstrapped (epics, CLAUDE.md, HANDOFF.md, GitHub repos)

## Immediate next actions

1. **Engine session** — merge `main_proposal` to casehubio/engine main (see handover message in this session)
2. **Devtown** — Epic 2 complete (per devtown HANDOFF.md); Epic 3 next (PR review CasePlanModel)
3. **parent#13** — Claude config restructuring (restructuring phase, not started)

## Key references

- Full-stack build: `mvn install -f aggregator.xml` from `~/claude/casehub/parent/`
- Build fix issues: work#163, qhorus#143, engine#242
- New protocols: `docs/protocols/ledger-spi-propagation.md`, `docs/protocols/module-tier-structure.md`
- Config map: `~/.claude/config-architecture.md`
- Prompt snippets: `~/.claude/prompt-snippets.md` (general), `docs/prompt-snippets.md` (casehubio)
- Claude config epic: casehubio/parent#13
