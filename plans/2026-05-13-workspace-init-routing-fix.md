# workspace-init: Fix Routing Template
**Date:** 2026-05-13  
**For:** cc-praxis session  
**File to update:** `workspace-init/SKILL.md`

---

## The Model (important context before editing)

Workspace is always the **staging area during an epic**. All skills write artifacts to the workspace while a branch is open. When the epic closes, `epic` reads the routing table and promotes artifacts to their final destinations.

This means:
- The **Artifact Locations table** (where skills write) always points to workspace directories — this is correct and must not change
- The **Routing table** (where artifacts END UP after epic close) is what needs fixing

---

## Current Routing Template in workspace-init (wrong)

```
| adr        | project     | lands in `docs/adr/` |
| blog       | project     | |
| design     | project     | |
| snapshots  | project     | |
| specs      | project     | lands in `docs/specs/` — design specs are project knowledge |
```

---

## Correct Routing Template (replace with this)

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
- `blog → workspace` (not project — blog is staged in workspace then published via blog-routing.yaml)
- `design → workspace` (not project — epic journal lives in workspace)
- `snapshots → workspace` (not project — snapshots are workspace artifacts)
- Add `plans → workspace` (was missing entirely)
- Keep `adr → project` ✓
- Keep `specs → project` ✓

---

## Also Add: `Blog directory:` Field to Workspace CLAUDE.md Template

In Step 5, after the Routing table, add this line to the workspace CLAUDE.md template:

```
**Blog directory:** `<WORKSPACE_PATH>/blog/`
```

Where `<WORKSPACE_PATH>` is the resolved workspace path (e.g. `~/claude/public/quarkmind`).

**Why:** `write-blog` reads `Blog directory:` to find where to write blog entries. Without this field, write-blog defaults to `blog/` relative to CWD — which is the project repo if Claude was started there, causing blog entries to land in the wrong place. This was the root cause of contamination found in the workspace audit.

---

## Also Fix: Valid Destinations Documentation

Update the Valid destinations note to include `mdproctor.github.io`:

```
Valid destinations: `project` · `workspace` · `mdproctor.github.io` · `alternative ~/path/to/repo/`
```

And add a note:
```
`mdproctor.github.io` is shorthand for the blog publishing destination — resolved via `~/.claude/blog-routing.yaml`.
```

---

## Scope

This affects the **template** in workspace-init — so all future workspace creations get the correct routing. Existing workspaces were corrected manually in the 2026-05-13 workspace audit session.
