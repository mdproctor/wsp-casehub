# workspace-init: Fix Routing Template and Git Discipline
**Date:** 2026-05-13 (revised 2026-05-14)
**For:** cc-praxis session  
**File to update:** `workspace-init/SKILL.md`

---

## The Model (important context before editing)

Workspace is always the **staging area during an epic**. All skills write artifacts to the workspace while a branch is open. When the epic closes, `epic` reads the routing table and promotes artifacts to their final destinations.

This means:
- The **Artifact Locations table** (where skills write) always points to workspace directories — this is correct and must not change
- The **Routing table** (where artifacts END UP after epic close) is what needs fixing

---

## Problem 1 — Routing Template Defaults Are Wrong

### Current (wrong)

```
| adr        | project     | lands in `docs/adr/` |
| blog       | project     | |
| design     | project     | |
| snapshots  | project     | |
| specs      | project     | lands in `docs/specs/` |
```

### Correct (replace with this)

```
| adr        | project     | lands in `docs/adr/` — promoted at epic close |
| specs      | project     | lands in `docs/specs/` — promoted at epic close |
| blog       | workspace   | staged here; published to mdproctor.github.io via publish-blog at epic close |
| plans      | workspace   | stay in workspace permanently |
| design     | workspace   | epic journal stays in workspace |
| snapshots  | workspace   | stay in workspace permanently |
| handover   | workspace   | |
```

**Key changes:**
- `blog → workspace` (not project — staged in workspace, published via blog-routing.yaml)
- `design → workspace` (not project — epic journal lives in workspace)
- `snapshots → workspace` (not project)
- Add `plans → workspace` (was missing entirely)
- Keep `adr → project` ✓ and `specs → project` ✓

---

## Problem 2 — LLM Cannot Reliably Know Which Repo CWD Belongs To

**The core issue:** A session can be started from either the project directory or the workspace directory. Claude has no reliable way to know which it is. Git operations (`git add`, `git commit`) act on the repo that contains CWD — so writing a plan to the workspace path and then running bare `git add` either silently fails or stages into the wrong repo. The same problem applies in reverse.

The current `git rev-parse --show-toplevel` instruction in Git Discipline is necessary but not sufficient — it tells Claude where it is, but not automatically which path every subsequent operation should use.

### Fix A — Declare Physical Path and Symlink Path as Properties in CLAUDE.md

workspace-init already knows both paths at creation time. It should write them as explicit properties at the top of the workspace CLAUDE.md:

```markdown
**Physical path:** `/Users/mdproctor/claude/public/<project>/CLAUDE.md`
**Symlinked at:** `/Users/mdproctor/claude/<project>/CLAUDE.md`
**Project repo:** `/Users/mdproctor/claude/<project>`
**Workspace:** `/Users/mdproctor/claude/public/<project>`
```

**Why this works:** The workspace CLAUDE.md is always the physical file — it is the same content whether Claude loads it directly (workspace session) or via symlink (project session). By declaring both paths as properties, Claude knows both repos immediately at session load without running any shell command. The properties become a single source of truth that every other section references.

**Parts of CLAUDE.md that benefit from referencing these properties:**

| Section | How it uses the properties |
|---------|---------------------------|
| Session Start | `add-dir {{Project repo}}` — no ambiguity about which dir |
| Git Discipline | `git -C {{Workspace}}` / `git -C {{Project repo}}` — explicit in all examples |
| Blog directory | `{{Workspace}}/blog/` — derived, not hardcoded separately |
| Routing table | Notes column can say "→ `{{Project repo}}/docs/adr/`" for clarity |
| Any skill | Skills can `grep "^\*\*Workspace:\*\*"` from the loaded CLAUDE.md to resolve the path |

**Skills that benefit from reading these properties:**
- `adr` — reads `**Workspace:**` to know where to stage, `**Project repo:**` to know where to promote
- `write-blog` — reads `**Workspace:**` for blog directory instead of separate `Blog directory:` field
- `handover` — reads `**Workspace:**` to write HANDOFF.md to correct repo
- `git-commit` / `java-git-commit` — reads both to use `git -C <path>` correctly
- `epic` — already reads routing; can use these properties to resolve destination paths

### Fix B — Always use `git -C <absolute-path>` for every git operation

Every git operation in Git Discipline examples and skill instructions must use `git -C <path>` with the value from the declared properties. Never bare `git add/commit`.

**Git Discipline section template:**

```markdown
## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`{{Workspace}}`) — plans, blog (staging), snapshots, handover
- **Project repo** (`{{Project repo}}`) — source code, ADRs, specs

**Never rely on CWD for git operations.** Use `git -C` with the absolute path from the properties above:

```bash
# Workspace artifact (plan, blog entry, handover)
git -C {{Workspace}} add <file>
git -C {{Workspace}} commit -m "..."

# Project artifact (source code, ADR, spec)
git -C {{Project repo}} add <file>
git -C {{Project repo}} commit -m "..."
```

Rule: if the file path starts with `{{Workspace}}`, use workspace. If it starts with `{{Project repo}}`, use project. The file path determines the repo — CWD does not.
```

### Why `git -C <path>` and not `cd <path> && git ...`

`cd` changes CWD for the rest of the session, creating hidden state. `git -C <path>` is a one-shot flag scoped to a single command. It is safer, auditable, and avoids state leaking between commands.

---

## Problem 3 — Missing `Blog directory:` Field

`write-blog` reads `Blog directory:` from CLAUDE.md to find where to write blog entries. Without it, write-blog defaults to `blog/` relative to CWD — which is the project repo if Claude was started there.

Once `**Workspace:**` is a declared property (Fix A above), `Blog directory:` becomes derivable. Until write-blog is updated to read `**Workspace:**` directly, keep the explicit field in the template — but derive it from the declared property rather than hardcoding it separately:

```
**Blog directory:** `{{Workspace}}/blog/`
```

---

## Problem 4 — Valid Destinations Missing `mdproctor.github.io`

Update the Valid destinations note:

```
Valid destinations: `project` · `workspace` · `mdproctor.github.io` · `alternative ~/path/to/repo/`

`mdproctor.github.io` — blog publishing destination, resolved via `~/.claude/blog-routing.yaml`.
```

---

## Problem 5 — Symlink Direction: Project Is Always Authoritative

**CLAUDE.md always lives in the project repo.** The workspace symlinks to it. This applies to all projects you own, regardless of whether they are personal or org projects.

- Authoritative file: **project repo** (`~/claude/<project>/CLAUDE.md`)
- Symlink: workspace `CLAUDE.md` → project `CLAUDE.md`
- `Physical path` property = project path; `Symlinked at` = workspace path

**The only exception:** projects where governance or policy prohibits adding CLAUDE.md to the repository — e.g. Apache projects (you may have commit rights but Apache contribution rules do not permit personal tool config files in the repo). In that case a third pattern applies:

**Pattern 3 — Policy-restricted project (e.g. Drools/Apache):**
- CLAUDE.md lives in the workspace only — cannot go in the project repo
- The workspace contains symlinks TO the project, not the reverse
- No symlink on the project side at all
- Example: `~/claude/drools/` is the workspace; `~/claude/drools/droolsoct2025` and `~/claude/drools/proj` are symlinks to `/Users/mdproctor/dev/droolsoct2025`

This is already correctly set up for Drools. workspace-init must ask at Step 1 whether the project permits CLAUDE.md in the repo, and choose the right pattern.

**Three valid patterns — no others:**
1. **Normal** (your project, policy allows): CLAUDE.md in project, workspace symlinks to it
2. **Policy-restricted** (Apache etc.): CLAUDE.md in workspace, workspace contains symlink(s) to project
3. ~~Two separate CLAUDE.md files with @include~~ — **never valid**

---

## Scope

This affects the **workspace-init template** — so all future workspaces get correct routing and safe git discipline from day one. Existing workspaces were corrected manually in the 2026-05-13/14 workspace audit session.

The `git -C <path>` pattern should also be reinforced in any skill that performs git commits (e.g. `adr`, `write-blog`, `handover`, `java-git-commit`) — those skills should never use bare `git add/commit` without an explicit path when a workspace is configured.
