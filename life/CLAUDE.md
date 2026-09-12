# casehub-life Workspace

**Name:** casehub-life

**Physical path:** (use proj/ symlink)
**Symlinked at:** (use wksp/ symlink)
**Project repo:** proj/
**Workspace:** wksp/
**Workspace type:** public

## Session Start


## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md
- `design/` — epic journal (created by `epic` at branch start)

## Git Discipline

Two git repositories are active in every session:
- **Workspace** (wksp/) — methodology artifacts: handover, blog, specs, plans, ADRs
- **Project repo** (proj/) — source code

Before any git operation, run `git rev-parse --show-toplevel` to confirm which repo is currently active. Do not assume — the session may have opened in either. cd to the correct repo before staging:
- Source code commits → project repo
- Methodology artifacts → workspace


## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Peer Repos — Hard Boundary

**This session owns exactly two repos: the workspace and the project repo.**
Every other casehubio repo is a peer repo with its own Claude session.

Peer repos (never commit or push to these from this session):
- `proj/` and all paths under it
- `proj/`
- `proj/`
- `proj/`
- `proj/`
- `proj/`
- `proj/`
- `proj/`
- `proj/`
- `proj/` (casehub-openclaw)
- Any other sibling directory under the casehub family

**When a cross-repo doc change is needed** (e.g. `docs/PLATFORM.md`,
`docs/repos/casehub-life.md` in the parent): file a GitHub issue on
`casehubio/parent` describing the change — never edit or commit directly.

Skills that check this (implementation-doc-sync, work-end, handover) must
read this section before deciding where to commit doc changes.

## Routing

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` — promoted at epic close |
| specs      | project     | lands in `docs/specs/` — promoted at epic close |
| blog       | project     | lands in `docs/blog/` — promoted at work end |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | epic journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |
