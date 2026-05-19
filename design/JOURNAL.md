# Design Journal — issue-94-work-lifecycle

### 2026-05-19 — Initial implementation · §Workspace Model

Implemented all four new lifecycle commands (work-start, work-end, work-pause, work-resume)
and deprecated epic as user-facing. The key design decisions confirmed during implementation:

**Detection state ordering** — orphaned `.meta` on main (state 3) must be checked before
misaligned branches (state 4). Both states satisfy "branches misaligned" so without the
explicit ordering, the Branch Switch Helper would attempt to switch to a deleted branch.

**`$DESIGN_REPO` stored in `.meta`** — Not re-derived from routing config at work-end time
because the routing config may change between sessions. `.meta` is the stable source of truth.

**Journal merge branch context** — When `design → workspace` routing is in effect, the
journal merge must happen during the 8a main-visit (while workspace is on main), not on
the workspace epic branch. Commits to the epic branch are discarded at close; the merge
must land on main to survive.

**java-update-design workspace mode detection** — Changed from `epic-*` branch prefix
(fragile, broke for `issue-NNN-*` branches) to `.meta` + `JOURNAL.md` + not-on-main
(correct for all branch naming conventions). Silent failure was the danger — every commit
on a non-epic-prefixed branch would write directly to DESIGN.md with no error.
