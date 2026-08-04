# Skill Routing Improvements — Instructions for cc-praxis
**Date:** 2026-05-13  
**For:** cc-praxis session  
**Priority:** High — these gaps cause silent artifact misplacement

---

## Background

A workspace audit found that most skills ignore the `## Routing` table in workspace CLAUDE.md. Only the `epic` skill reads and applies it. The other skills use hardcoded paths or simpler heuristics, causing artifacts to land in the wrong location (project repo vs workspace) without any error.

**Root cause summary (from skill audit):**

| Skill | How it finds output dir | Gap |
|-------|------------------------|-----|
| `adr` | Hardcoded `docs/adr/` | Ignores routing entirely |
| `write-blog` | Reads `Blog directory:` field in CLAUDE.md | Ignores Routing table; only works if `Blog directory:` is set |
| `handover` | Assumes CWD is workspace | No routing logic — works by convention |
| `epic` | Reads Routing table (3-layer cascade) | ✅ Correct — this is the reference implementation |
| `brainstorming`/`writing-plans` | Unknown (source not accessible) | Unknown |

---

## Fix 1 — `adr` skill: read Routing table before writing

**Current behaviour:** Always writes to `docs/adr/` relative to CWD regardless of routing config.

**Desired behaviour:** Before writing, resolve the destination using the same 3-layer cascade that `epic` uses:
1. Check `## Routing` table in loaded CLAUDE.md for `adr` row
2. Fall back to global `~/.claude/CLAUDE.md` `**Default destination:**` 
3. Fall back to built-in default: `project` (i.e. `docs/adr/`)

If destination is `workspace`, write to `adr/` in the workspace directory (find workspace from CLAUDE.md `**Workspace:**` or `**Project repo:**` field). If destination is `project`, keep existing `docs/adr/` behaviour.

**Reference:** Look at how `epic`'s Step B5 reads the Routing table — replicate that logic in `adr`.

---

## Fix 2 — `write-blog` skill: read Routing table, not just `Blog directory:`

**Current behaviour:** Reads `Blog directory:` field from CLAUDE.md. If absent, defaults to `blog/` in CWD.

**Problem:** `Blog directory:` is a workaround that requires each project CLAUDE.md to manually specify the path. The Routing table (`blog → workspace`) should be sufficient.

**Desired behaviour:** Resolution order:
1. Check `## Routing` table for `blog` row — if `workspace`, write to `blog/` in workspace; if `project`, write to `blog/` in project  
2. Fall back to `Blog directory:` field (backwards compatibility)
3. Fall back to `blog/` in CWD

**Note:** `Blog directory:` should remain supported for projects that explicitly set a non-standard path (e.g. `_posts/` for Jekyll sites). It overrides the Routing table.

---

## Fix 3 — `workspace-init` skill: add `Blog directory:` to workspace CLAUDE.md automatically

**Current behaviour:** workspace-init creates the workspace CLAUDE.md with an Artifact Locations table and Routing table, but does NOT add a `Blog directory:` field. This means `write-blog` (which reads `Blog directory:` but not the Routing table) has no way to find the workspace blog.

**Desired behaviour:** In Step 5, when writing the workspace CLAUDE.md, also add:
```
**Blog directory:** `<workspace-path>/blog/`
```
alongside the Writing Style Guide section (or in the Artifact Locations section).

**Why:** Until Fix 2 is implemented, `write-blog` needs `Blog directory:` to find the workspace. This is a bridging fix.

---

## Fix 4 — `workspace-init` skill: ensure project CLAUDE.md is workspace-aware

**Current behaviour:** workspace-init adds a `## Project Artifacts` section to the project CLAUDE.md, but does not make the project CLAUDE.md load the workspace routing config.

**Two patterns currently in use:**
- **Symlink** (cccli): project CLAUDE.md is a symlink to workspace CLAUDE.md — workspace CLAUDE.md is always loaded regardless of start dir ✅
- **Separate files + @include** (quarkmind): workspace CLAUDE.md @includes project CLAUDE.md — but starting from the project dir only loads the project CLAUDE.md, not the workspace routing table ⚠️

**Desired behaviour:** workspace-init should use the symlink pattern consistently OR add the workspace CLAUDE.md as an @include in the project CLAUDE.md (accepting the one-way dependency, no circular reference if the workspace does NOT @include the project in return).

**Recommendation:** Prefer symlink. If symlink, the workspace CLAUDE.md should NOT use `@<project>/CLAUDE.md` — instead the project content is inlined into the workspace CLAUDE.md directly. workspace-init should run `/init` on the project CLAUDE.md first, then incorporate the content directly.

---

## Fix 5 — `brainstorming` / `writing-plans` skills: verify routing support

These skills write specs and plans to `specs/` and `plans/` respectively. Their source was not accessible during the audit. Verify whether they:
- Read the Routing table
- Read a `Specs directory:` / `Plans directory:` field
- Or hardcode to `specs/` / `plans/` in CWD

If they use CWD-relative paths and the session starts in the project repo (not workspace), specs and plans land in the project repo. This is the same bug found in quarkmind (32 spec files stranded in `docs/superpowers/specs/`).

Apply the same routing cascade fix as `adr` if they don't already read routing.

---

## Additional Finding — cccli workspace has no GitHub remote

`~/claude/public/cccli` has no git remote configured. The CLAUDE.md doesn't specify a GitHub workspace repo. This workspace exists only locally. Either:
1. Create a GitHub repo (`gh repo create mdproctor/wsp-cccli --public`) and set it as remote, or
2. Decide this workspace stays local-only

This is separate from the skill routing fixes but was found during the audit.

---

## What Was Already Fixed (this session, not needed in cc-praxis)

- cccli workspace CLAUDE.md: added Routing table and `Blog directory:` field
- quarkmind project: 32 specs + 2 plans removed (were stranded in `docs/superpowers/`)
- cccli project: 9 blog entries + 1 plan removed (were stranded in `blog/` and `docs/blog/`)
- cccli workspace: duplicate `design/DESIGN.md` removed
