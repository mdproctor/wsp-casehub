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

**The core issue:** A session can be started from either the project directory or the workspace directory. Git operations (`git add`, `git commit`) act on the repo that contains CWD. If the LLM writes a plan file to the workspace path but CWD is the project, the `git add` either silently fails or stages into the wrong repo. The same applies in reverse — an ADR written to the project path when CWD is the workspace gets committed to the workspace repo.

The current `git rev-parse --show-toplevel` instruction in Git Discipline is necessary but not sufficient — knowing where you are doesn't automatically mean every subsequent git operation uses the right repo.

### Fix: Always use `git -C <absolute-path>` for every git operation

The workspace CLAUDE.md template must establish two named paths at session start, and every git operation must reference them explicitly.

**Add to the workspace CLAUDE.md template — Session Start section:**

```markdown
## Session Start

Run `add-dir <PROJECT_PATH>` before any other work.

At the start of every session, establish:
- `PROJECT_REPO` = `<PROJECT_PATH>` (absolute path to project repo)
- `WORKSPACE` = `<WORKSPACE_PATH>` (absolute path to workspace repo)

All git operations must use explicit paths — never rely on CWD:
- `git -C $PROJECT_REPO add <file>` — stage project repo changes
- `git -C $WORKSPACE add <file>` — stage workspace changes
```

**Add to the Git Discipline section:**

```markdown
## Git Discipline

Two git repositories are active in every session:
- **Workspace** (`<WORKSPACE_PATH>`) — plans, blog (staging), snapshots, handover
- **Project repo** (`<PROJECT_PATH>`) — source code, ADRs, specs

**Never rely on CWD for git operations.** Always use explicit absolute paths:

```bash
# Stage and commit a workspace artifact (e.g. plan, blog entry)
git -C <WORKSPACE_PATH> add <file>
git -C <WORKSPACE_PATH> commit -m "..."

# Stage and commit a project artifact (e.g. ADR, spec, source code)
git -C <PROJECT_PATH> add <file>
git -C <PROJECT_PATH> commit -m "..."
```

Before any git operation, confirm which repo owns the file:
- File path starts with `<WORKSPACE_PATH>` → use `git -C <WORKSPACE_PATH>`
- File path starts with `<PROJECT_PATH>` → use `git -C <PROJECT_PATH>`
- When in doubt: `git -C <path> rev-parse --show-toplevel` to verify
```

### Why `git -C <path>` and not `cd <path> && git ...`

`cd` changes the working directory for the rest of the session. `git -C <path>` is a one-shot flag that runs git against a specific repo without changing CWD. It is safer and avoids state leaking between commands.

---

## Problem 3 — Missing `Blog directory:` Field

`write-blog` reads `Blog directory:` from CLAUDE.md to find where to write blog entries. Without it, write-blog defaults to `blog/` relative to CWD — which is the project repo if Claude was started there.

**Add to workspace CLAUDE.md template** (after the Routing table):

```
**Blog directory:** `<WORKSPACE_PATH>/blog/`
```

---

## Problem 4 — Valid Destinations Missing `mdproctor.github.io`

Update the Valid destinations note:

```
Valid destinations: `project` · `workspace` · `mdproctor.github.io` · `alternative ~/path/to/repo/`

`mdproctor.github.io` — blog publishing destination, resolved via `~/.claude/blog-routing.yaml`.
```

---

## Scope

This affects the **workspace-init template** — so all future workspaces get correct routing and safe git discipline from day one. Existing workspaces were corrected manually in the 2026-05-13 workspace audit session.

The `git -C <path>` pattern should also be reinforced in any skill that performs git commits (e.g. `adr`, `write-blog`, `handover`, `java-git-commit`) — those skills should never use bare `git add/commit` without an explicit path when a workspace is configured.
