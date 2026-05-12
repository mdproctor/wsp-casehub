# Handoff — Naming Alignment and Protocol Work
2026-05-12

## What changed this session

**Full naming alignment across all repos — complete and pushed.**

All nine casehub repos now follow the platform convention (PP-20260508-5c0e4b):
- groupId: `io.casehub` everywhere (claudony was `dev.claudony`)
- Root artifactId: `casehub-{repo}-parent`
- Module artifactIds: `casehub-{repo}-{function}`
- Folder names: short, no repo prefix (`api/`, `runtime/`, `queues/` etc.)
- Versions: all on `0.2-SNAPSHOT` (aml and clinical were on `0.1-SNAPSHOT`)

Repos changed: devtown, qhorus (stale submodule removed), work, engine, connectors, claudony, aml, clinical. Ledger and qhorus module folders were already correct.

**Two consumers missed in connectors rename — caught and fixed:**
- `casehub-work-notifications` and `casehub-devtown-app` referenced `casehub-connectors` (now `casehub-connectors-core`)
- Fixed before push

**New protocols added to `docs/protocols/`:**
- `maven-coordinate-standard.md` — full coordinate rules with verification checklist
- `artifact-rename-propagation.md` — find all consumers before renaming, update same session

**PLATFORM.md — Cross-Repo Dependency Map added:**
- 25-row table mapping every cross-repo artifact dependency
- Primary tool for impact analysis on future renames

**cc-praxis skills added:**
- `work-start` — mandatory pre-work checks (PLATFORM.md + Coherence Protocol, protocols, issue, IntelliJ)
- `implementation-doc-sync` — session-scoped doc sweep after implementation

**Prompt snippet finalised** (`docs/prompt-snippets.md`):
```
invoke work-start first. superpowers:brainstorming before designing. superpowers:test-driven-development before implementing. java-dev for all Java (loads testing-principles + ide-tooling). superpowers:requesting-code-review before committing. implementation-doc-sync after.
[describe the issue]
```

**CLAUDE.md Development Workflow updated** — now lists java-dev, implementation-doc-sync, and prompt snippet reference.

## Immediate next actions

1. **Devtown** — Epic 3: PR review CasePlanModel (was next before this session)
2. **parent#13** — Claude config restructuring (partially addressed this session via prompt snippet + skills; issue may need updating or closing)
3. **Clinical** — on `epic-adverse-event-escalation`; surefire LogManager fix is uncommitted there; leave to clinical Claude session
4. **engine** — `main_proposal` branch can be cleaned up once treblereel has acknowledged the reconstruction

## Key references

- Naming convention protocol: `docs/protocols/maven-submodule-folder-naming.md`
- Coordinate standard: `docs/protocols/maven-coordinate-standard.md`
- Artifact rename propagation: `docs/protocols/artifact-rename-propagation.md`
- Cross-repo dependency map: `docs/PLATFORM.md` (Cross-Repo Dependency Map section)
- Work-item prompt snippet: `docs/prompt-snippets.md`
- New skills: `~/.claude/skills/work-start/`, `~/.claude/skills/implementation-doc-sync/`
- Blog: `blog/2026-05-12-mdp01-naming-alignment-and-propagation.md`
