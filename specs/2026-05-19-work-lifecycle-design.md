# Work Lifecycle — Unified Design Spec
**Date:** 2026-05-19  
**Issue:** cc-praxis#94  
**Status:** Design approved — ready for implementation

---

## Context and Decisions

Replaces the fragmented `work-start` + `epic begin/close` + proposed pause/resume with four coherent commands. The `epic` skill is deprecated as user-facing; its close logic migrates into `work-end`. `/epic begin` continues to work during migration (it will be retired once `work-start` absorbs branch creation fully, tracked in cc-praxis#94).

**Confirmed decisions:**
- Branches (not worktrees) for all issue-scoped work
- Every branch gets `.meta` + `JOURNAL.md` — no lightweight vs epic distinction
- Branch name derived from issue: `issue-NNN-<slug>`, user confirms before creation
- Full pre-checks on every session including resume (revisit if friction emerges)
- Epic skill: parallel during migration, retired once stable
- GitHub rebase merge on PRs — linear history throughout

---

## The Four Commands

| Command | Replaces | Purpose |
|---------|----------|---------|
| `work-start` | `work-start` + `epic begin` | Entry point: detect state, create branch, run pre-checks |
| `work-end` | `epic close` | Close branch: promote artifacts, merge journal, mark closed |
| `work-pause` | (new) | Save context, switch both repos to main |
| `work-resume` | (new) | Restore context, switch both repos back to branch |

---

## Shared: Branch Switch Helper

Referenced by all four commands. Never switch one repo alone.

```bash
PROJECT=$(grep "add-dir" CLAUDE.md | head -1 | sed 's/.*add-dir //')
WORKSPACE=$(grep "^\*\*Workspace:\*\*" CLAUDE.md | head -1 | sed 's/.*`\(.*\)`.*/\1/')

git -C "$PROJECT" checkout <branch>
git -C "$WORKSPACE" checkout <branch>

# Check remote ahead — PROMPT before pulling
PROJECT_BEHIND=$(git -C "$PROJECT" rev-list HEAD..origin/<branch> --count 2>/dev/null || echo 0)
WORKSPACE_BEHIND=$(git -C "$WORKSPACE" rev-list HEAD..origin/<branch> --count 2>/dev/null || echo 0)
if [ "$PROJECT_BEHIND" -gt 0 ] || [ "$WORKSPACE_BEHIND" -gt 0 ]; then
  echo "Remote has new commits (+${PROJECT_BEHIND} project, +${WORKSPACE_BEHIND} workspace)."
  echo "Incorporate now with pull --rebase? (y/n)"
  # Wait — user may not be ready for upstream changes
fi

# Verify alignment after switch
PROJECT_BRANCH=$(git -C "$PROJECT" branch --show-current)
WORKSPACE_BRANCH=$(git -C "$WORKSPACE" branch --show-current)
[ "$PROJECT_BRANCH" = "$WORKSPACE_BRANCH" ] \
  || { echo "⚠️ Mismatch after switch. Manual alignment required."; exit 1; }
echo "✅ Both repos on: $PROJECT_BRANCH"
```

If helper fails (branch absent in one repo, network error): hard stop with instructions. Do not loop.

---

## work-start

### Detection — checked in order, first match wins

```
1. design/.paused on workspace main
   → Hard stop. "Paused branch detected. Invoke work-resume first."

2. design/.meta exists, both repos on matching branch
   → Resume path.

3. design/.meta exists, branches misaligned
   → Invoke Branch Switch Helper inline.
     If helper fails → hard stop with manual instructions (no loop).

4. design/.meta on main (orphaned)
   → Hard stop. "Invoke work-end to complete or discard the abandoned branch."

5. On main, no .meta, no .paused
   → New branch path.

6. On non-epic, non-main branch, no .meta
   → "You are on <branch> with no branch scaffold.
      Continue here (y) or switch to main (n)?"
      y → run pre-checks only, no scaffold.
      n → Branch Switch Helper to main, re-run detection.
```

### New branch path

**Step 1 — Work description**  
Use invocation argument if provided. Otherwise prompt: "Describe the work in one sentence."

**Step 2 — Platform coherence**  
Read platform doc. Run five coherence questions against the work description. Resolve any concerns before proceeding.

**Step 3 — Relevant protocols**  
Scan `docs/protocols/`. Read applicable rules. Resolve violations before proceeding.

**Step 4 — Issue**  
If tracking enabled:
- Search for existing open issue
- If found: confirm (`Found #N: <title>. Use this? y/n`)
- If not found: offer to create; wait for result
- Record issue number and title for Step 5

If tracking disabled: skip silently.

**Step 5 — Branch name**  
Derive: `issue-NNN-<slug>` (title lowercased, special chars stripped, max 30 chars).  
Show to user, allow override. Guards: reject `main`, `HEAD`, or any existing branch name.  
Branch number (NNN) is the stable key — title slug is a convenience only.

**Step 6 — Flyway V scan**  
```bash
git -C <project-path> fetch --all 2>/dev/null || echo "⚠️ No network — scan incomplete"
```
If network available: scan main + all remote branches for claimed V numbers. Compute `next-safe-v = max + 1`. If conflict: warn, show `Next safe V: <N>`, block until acknowledged.  
If no network: warn, record `flyway-next-v: unknown`. User must verify before adding migrations.

Default: `flyway-next-v: unknown` — the user may not know yet at branch creation time.  
Only ask if the user has already described migration work: "Will this branch include database migrations? (y/n)"  
→ y: `flyway-next-v: <N>`  
→ explicit n: `flyway-next-v: none`  
→ no answer / unsure: `flyway-next-v: unknown` (safe default; user verifies before adding migrations)

**Step 7 — Create branches (atomic)**  
```bash
git -C <project-path> checkout -b <branch-name>
# If fails → abort (nothing to clean up)
git checkout -b <branch-name>
# If fails → git -C <project-path> branch -D <branch-name>, abort, report error
```

**Step 8 — Resolve routing-aware SHA baseline**  
Read routing config (3-layer cascade) for `design` artifact:
- `design → workspace`: baseline = `git -C <workspace-path> rev-parse main`
- `design → project` (default): baseline = `git -C <project-path> rev-parse HEAD`

**Step 9 — Scaffold**  
```bash
mkdir -p design
```
`design/JOURNAL.md`:
```markdown
# Design Journal — <branch-name>
```
`design/.meta`:
```
branch: <branch-name>
project-sha: <from Step 8>
date: <YYYY-MM-DD>
issue: <N or blank>
flyway-next-v: <N | none | unknown>
design-section-hashes: <H2 hashes from <design-repo>/DESIGN.md, or blank>
```

**Step 10 — Commit and push scaffold**  
```bash
git -C <workspace-path> add design/JOURNAL.md design/.meta
git -C <workspace-path> commit -m "init(<branch-name>): scaffold workspace branch"
git -C <workspace-path> push  # non-fatal if fails; warn and continue
```

**Step 11 — IntelliJ MCPs**  
Call `mcp__intellij-index__ide_index_status` and `mcp__intellij__get_project_modules`. Hard stop if unavailable.

**Step 12 — Offer brainstorming**  
"Start a brainstorm? (y/n)" — if yes, specs write to `specs/<branch-name>/`.

**Specs routing:** `specs` is always routed to `project` (`docs/specs/` at close). Add `specs → project` as a non-configurable fixed route in work-end's routing resolution. Do not add `specs` to the CLAUDE.md routing table — it is not user-configurable. The three-layer cascade applies to blog/adr/snapshots/plans/design; specs are always promoted to project.

### Resume path (Detection state 2)

Surface `.meta`:
```
⚡ Resuming: <branch-name>  Issue: #<N>  Started: <date>
   Flyway V: <N | none | unknown>
   Project: <branch>  Workspace: <branch>
```
Run Steps 2, 3, 11 only. Skip all branch creation steps.

---

## work-end

### Pre-conditions

Checked in order:

1. **If `design/.paused` exists on workspace main** → hard stop. "You have a paused branch. Invoke `work-resume` first, then `work-end`."
2. **Must be on a branch with `.meta`** → proceed.
3. **If invoked from main with orphaned `.meta`** → hard stop. Offer to switch to surviving epic branch and close from there, or discard.

`.paused` check always comes first — it takes priority over orphaned `.meta`.

### Step 1 — Read context
```bash
cat design/.meta          # branch, project-sha, issue, flyway-next-v
grep "add-dir" CLAUDE.md  # project path
grep "GitHub repo:" CLAUDE.md  # owner/repo
```

### Step 2 — Flyway V re-scan

Re-scan at close time — another branch may have claimed the same V numbers since this branch started.

```bash
git -C <project-path> fetch --all 2>/dev/null || echo "⚠️ No network — scan skipped"
```

If conflict: `[R]` renumber affected migration files, `[A]` abort. Block close until resolved.

### Step 3 — Resolve routing

Three-layer cascade for each artifact type. Warn on deprecated vocabulary (`base`, `project repo`, `design-journal`). Show resolved table, detect capability (`remote-git`, `local-git`, `filesystem`), user confirms before proceeding.

### Step 4 — Inventory artifacts
```bash
ls adr/ | grep -v INDEX.md
ls blog/ | grep -v INDEX.md
ls snapshots/ | grep -v INDEX.md
ls specs/<branch-name>/
ls plans/ | grep -v "^attic$"
cat design/JOURNAL.md
```

### Step 5 — Journal validation

**5a — DESIGN.md existence**  
If missing: `[C]` create from journal entries, `[S]` skip merge.

**5b — Section heading drift**  
Re-hash H2 headings. Compare against `design-section-hashes` in `.meta`. For each §Section anchor in JOURNAL.md, verify its heading still exists unchanged.  
If drift: `[U]` update journal anchors, `[S]` skip drifted sections, `[A]` abort.

**5c — Anchor validation**  
Count `^### .*·.*§` lines vs total `^### ` lines.  
If any entries lack anchors: `[F]` fix via java-update-design, `[S]` skip merge, `[C]` continue accepting loss.

**5d — Empty journal**  
If no entries at all: `[W]` write retrospective via java-update-design, `[S]` skip and accept permanent loss (stated explicitly — this is a permanent loss of design narrative).

### Step 6 — Select specs for GitHub posting

If tracking enabled, list `specs/<branch-name>/`, ask which to post. Skip silently if disabled.

### Step 7 — Present close plan

```
work-end close plan — <branch-name>

  Flyway V check     ✅ no conflicts
  Artifact routing
  ├── adr/<N>        → project      [remote-git]
  ├── blog/<N>       → workspace    [remote-git]
  ├── specs/<N>      → project      [remote-git]
  └── snapshots/<N>  → workspace    [remote-git]
  Plan archiving     → plans/attic/<branch-name>/  [workspace main]
  Journal merge      → DESIGN.md  (<N> sections)
  Spec posting       → #<N>  (<filenames>)
  Issue              → close #<N>
  Publish blog       → offer after (N entries staged)

Approve all, or step by step? (all / step)
```

### Step 8 — Execute

Failures are reported but do not stop remaining steps, **except**: journal merge failure prompts before continuing to issue close.

**8a — Batch workspace-main operations** (single main-visit covers both artifact promotion and plan archiving):
```bash
git stash  # only if uncommitted workspace changes exist
git checkout main
git -C <workspace-path> pull --rebase origin main

# Promote workspace-routed artifacts (blog, snapshots)
for each workspace-routed artifact:
  mkdir -p <workspace-dest>
  git checkout <branch-name> -- <artifact-files>
  git -C <workspace-path> add <workspace-dest>/
  git -C <workspace-path> commit -m "feat: promote <type> from <branch-name>"

# Archive plans to attic
if plans exist:
  git checkout <branch-name> -- plans/<files>
  mkdir -p plans/attic/<branch-name>
  mv plans/<files> plans/attic/<branch-name>/
  git -C <workspace-path> add -A
  git -C <workspace-path> commit -m "archive(<branch-name>): move plans to attic"

git -C <workspace-path> push  # single push for all workspace-main commits
git checkout <branch-name>
git stash pop  # only if stashed above
```

**8b — Project-routed artifact promotion** (ADRs, specs):
```bash
for each project-routed artifact:
  mkdir -p <project-dest>
  cp <files> <project-dest>/
  git -C <project-path> add <project-dest>/
  git -C <project-path> commit -m "feat: promote <type> from <branch-name>"
  git -C <project-path> push  # non-fatal if fails; report
```

**8c — Spec cleanup** (only if 8b push exit code was 0 — confirmed successful):
If 8b push failed, spec cleanup is skipped entirely. The workspace copy is the only remaining copy — deleting it would cause permanent data loss.
```bash
rm -rf specs/<branch-name>/
git -C <workspace-path> add -A
git -C <workspace-path> commit -m "chore(<branch-name>): remove promoted specs from staging"
git -C <workspace-path> push
```

**8d — Journal merge**:
1. Read baseline: `git -C <design-repo> show <project-sha>:DESIGN.md`
2. Read current `<design-repo>/DESIGN.md`
3. Apply journal narrative per §Section, preserving independent main-branch changes
4. Write merged result
5. Post-merge verification: re-read each §Section; if anything looks wrong, present to user (`[A]` accept, `[R]` redo, `[X]` abort) before committing
6. Commit and push

If journal merge fails: prompt user before continuing to issue close. Do not silently skip.

**8e — Spec posting**: post selected specs as collapsible comments on GitHub issue.

**8f — Issue close**: `gh issue close <N> --repo <owner>/<repo>`

**8g — Offer publish-blog**: if blog entries were staged to workspace.

**8h — Final report**:
```
✅ ADRs → project
✅ Specs → project  
✅ Blog → workspace
✅ Plans → attic
✅ Journal merged → DESIGN.md (N sections)
✅ Specs posted to #N, issue closed
❌ Push failed — <path>. Run: git -C <path> push
```

### Step 8 (step path)

Phase 1: Artifact routing — confirm, execute, report → "Continue to journal merge? (y/n)"  
Phase 2: Journal merge — show each §Section before/after, confirm → "Continue to GitHub posting? (y/n)"  
Phase 3: Spec posting, issue close, publish-blog offer → "Continue to branch cleanup? (y/n)"  
Phase 4: EPIC-CLOSED.md and return to main.

### Step 9 — Mark closed

Write `EPIC-CLOSED.md` on workspace epic branch:
```markdown
# Branch Closed — <branch-name>
**Date:** <YYYY-MM-DD>
**Issue:** #<N>
**Scheduled for deletion:** <date + 14 days>
```
```bash
git -C <workspace-path> add EPIC-CLOSED.md
git -C <workspace-path> commit -m "docs(<branch-name>): mark closed, deletion due <date>"
git -C <workspace-path> push
```

Branches are **not deleted**. `EPIC-CLOSED.md` is the signal for hygiene scan cleanup.

### Step 10 — Return to main

```
Return both repos to main? (y/n)
```
If y:
```bash
git -C <project-path> checkout main
git checkout main
```
Check remote ahead; prompt before `pull --rebase`. Not automatic.

---

## work-pause

### Step 1 — Validate state
Must be on a branch with `.meta`. If `.paused` already on workspace main: hard stop — only one paused branch at a time.

### Step 2 — Handle uncommitted changes
Check both repos. If changes exist:
```
Uncommitted changes found. Stash them? (y/n)
  y → git stash in both repos; record stash SHAs in .meta
  n → abort (commit or discard first)
```
Record in `.meta`: `stash-project: <stash@{N} | none>`, `stash-workspace: <stash@{N} | none>` — use the stash stack reference, not the SHA. The stash position `stash@{0}` is recorded at pause time. **Note:** if the user manually runs `git stash` or `git stash pop` on either repo between pause and resume, the recorded reference will shift and restoration will fail silently. This is a known limitation — document it to the user at pause time.

### Step 3 — Record pause in .meta (atomic with Step 4)
Add to `.meta`:
```
paused: true
paused-at: <ISO timestamp>
paused-issue: <N if working on different issue, else same as issue>
```
Commit to workspace epic branch and push. **If push fails: abort before Step 4.** Do not write `.paused` to main if `.meta` push failed.

### Step 4 — Write .paused to workspace main
```bash
# If workspace changes were stashed in Step 2, they are already on the stash stack
# Do NOT stash again — go directly to checkout main
git checkout main
git -C <workspace-path> pull --rebase origin main

cat > design/.paused << EOF
branch: <branch-name>
paused-at: <ISO timestamp>
EOF

git -C <workspace-path> add design/.paused
git -C <workspace-path> commit -m "chore: pause marker for <branch-name>"
git -C <workspace-path> push
```

### Step 5 — Switch project repo to main
```bash
git -C <project-path> checkout main
```
Prompt before remote pull (may not be ready for upstream changes).

### Step 6 — Confirm
```
⏸  Paused: <branch-name>  Both repos on main.
   Resume with: work-resume
```

---

## work-resume

### Step 1 — Check .paused
```bash
cat design/.paused 2>/dev/null || { echo "Nothing to resume."; exit 1; }
```
Read branch name from `.paused`.

### Step 2 — Stale check
Verify branch exists in both repos:
```bash
git branch -a | grep <branch-name>
git -C <project-path> branch -a | grep <branch-name>
```
If missing from either: `[D]` discard `.paused` and clean up, `[A]` abort.

### Step 3 — Switch both repos to epic branch
Use Branch Switch Helper. Prompt before remote pull.

### Step 4 — Remove .paused from workspace main
```bash
git stash  # workspace is now on epic branch; stash if any changes
git checkout main
git -C <workspace-path> pull --rebase origin main
rm design/.paused
git -C <workspace-path> add -A
git -C <workspace-path> commit -m "chore: resume <branch-name>, remove pause marker"
git -C <workspace-path> push
git checkout <branch-name>
git stash pop  # only if stashed above
```

### Step 5 — Restore stashed changes
Read `stash-project` and `stash-workspace` from `.meta`.
```bash
[ "$stash-project" != "none" ] && git -C <project-path> stash pop
[ "$stash-workspace" != "none" ] && git stash pop
```
If stash pop fails (conflict): report, do not abort — user resolves manually.

### Step 6 — Clear pause flags from .meta
Remove `paused:`, `paused-at:`, `paused-issue:`, `stash-project:`, `stash-workspace:`.  
Commit and push.

### Step 7 — Surface context
```
▶  Resumed: <branch-name>  Issue: #<N>  Paused <duration> ago
   Stash restored: <yes | no | conflict — resolve manually>
```

### Step 8 — Run pre-checks
Steps 2, 3, 11 from work-start (platform coherence, protocols, IntelliJ).

---

## Epic skill — Deprecation

`epic/SKILL.md` gains a deprecation header:
```
⚠️  This skill is deprecated. Use:
    work-start  — for /epic begin
    work-end    — for /epic close
    work-pause  — new
    work-resume — new

The workflows below are preserved for reference during migration.
```

Skill remains functional during migration. Retired once all sessions use the new commands and no issues are found.

**Gate text migration:** The current work-start gate says "Option 1 (branch work) → invoke `/epic begin`". Once the new skills are deployed, this gate text must change to "branch creation is handled by work-start itself — proceed." Update the gate text in work-start when deploying, not before.

---

## Required Skill Updates (non-negotiable for correctness)

These existing skills must be updated as part of the migration. Without them, the new branch naming convention silently breaks core functionality.

### java-update-design — workspace mode detection

**Current (broken for `issue-NNN-*` branches):**
```bash
git branch --show-current   # must match epic-* pattern
ls design/.meta 2>/dev/null
ls design/JOURNAL.md 2>/dev/null
```

**Required update — detect on `.meta` presence, not branch prefix:**
```bash
# All three must be true; branch name prefix is NOT checked
ls design/.meta 2>/dev/null        # .meta must exist
ls design/JOURNAL.md 2>/dev/null   # JOURNAL.md must exist
git branch --show-current | grep -v "^main$"  # not on main
```

Without this fix, every commit on an `issue-NNN-*` branch silently writes directly to `DESIGN.md` instead of `JOURNAL.md`. The journal mechanism is entirely bypassed with no error.

### Branch Hygiene Scan — detection by .meta, not branch name

**Current (broken for `issue-NNN-*` branches):**
```bash
git branch | grep 'epic-'
```

**Required update — detect all work branches by `.meta` presence:**
```bash
# Find all branches that have a .meta file on the workspace side
for branch in $(git branch | sed 's/^[* ]*//' | grep -v '^main$'); do
  git show "$branch:design/.meta" 2>/dev/null && echo "$branch"
done
```

Without this fix, the hygiene scan returns empty for all new branches — Flyway conflicts and unmerged code go undetected.

---

## Branch Hygiene Scan

Moved to the handover wrap checklist (epic hygiene item). Also offered by `work-end` after close.

**Detection:** Find branches by `.meta` presence, not by `epic-*` name prefix. See Required Skill Updates above.

Checks per branch: Flyway V conflicts across all open branches, unmerged code on project branch, PR status, EPIC-CLOSED.md existence and deletion date, artifact promotion status (specs, plans).

Blocks deletion of any branch with unmerged code. Reports only — does not auto-delete.

---

## .meta Schema

```
branch: <branch-name>
project-sha: <SHA — routing-aware baseline>
date: <YYYY-MM-DD>
issue: <N or blank>
flyway-next-v: <N | none | unknown>
design-section-hashes: <H2 hashes, one per line>
paused: true                         # only present when paused
paused-at: <ISO timestamp>           # only present when paused
paused-issue: <N>                    # only present when paused on different issue
stash-project: <stash@{N} | none>    # only present when paused; stack position at pause time
stash-workspace: <stash@{N} | none>  # only present when paused; stack position at pause time
```

---

## Known Limitations

- Only one paused branch at a time
- Stash reference uses `stash@{N}` position — if the user manually stashes or pops between pause and resume, the recorded position shifts and restoration fails. User is warned at pause time.
- If project repo stash is accidentally popped during interruption, it cannot be recovered
- `paused-issue` is informational only — not enforced by work-start
- Flyway V scan requires network; `unknown` V is a manual verification burden
- **Flyway scan staleness in concurrent sessions:** The family workspace (casehub) runs multiple Claude sessions simultaneously across repos. The Flyway V scan at `work-end` may be stale if another session merged a branch between `work-start` and `work-end`. The re-scan at close time mitigates this but does not eliminate the race. If two sessions close branches simultaneously, the second session's re-scan will catch any conflict introduced by the first. Treat this as best-effort conflict detection, not a guarantee.
- `work-start` gate text currently says "invoke `/epic begin`" — update this text when deploying the new skills (see Epic skill — Deprecation section) 
