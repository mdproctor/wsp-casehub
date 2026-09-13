# HANDOFF — casehub-blocks-ui

## Current Session

**Branch:** `issue-157-rich-org-diagram` (active, not closed)
**Issue:** casehubio/blocks-ui#157 — Rich org diagram
**Queue:** 1/1 — all plan tasks complete, NOT merged to main yet

### What Was Done

**Session 7 — layout rule engine + cross-repo refactoring:**

- Designed and implemented a forward-chaining layout rule engine
- Refactored generic engine into casehub-pages (graph-renderer), org-specific layer stays in blocks-ui
- Added supervision panel and chip bar UI for panel toggling

**Layout Rule Engine (7 commits → then refactored to pages):**
1. Types + FactBase — Phase, Fact, FactBase, LayoutRule, ClassificationRule, HardConstraint, CompositionReport
2. LayoutEngine — phase execution, group resolution (mutual exclusion), composition validation, explain mode
3. Classification rules — 5 generic structural detectors + archetype selector (replaces archetype-detection.ts)
4. Layout rules — horizontal-internal, vertical-stacking, position-aware-handles (parameterized by node type)
5. Hard constraints — no-container-overlap, child-containment, no-sibling-overlap
6. Integration — wired engine into blocks-org-diagram.ts, deleted 8 old scattered files
7. Explain mode + archetype composition tests for 6 archetypes

**Cross-Repo Refactoring to Pages:**
- Generic layout engine moved to `pages/packages/graph-renderer/src/layout/` (5 new files)
- blocks-ui now re-exports from `@casehubio/graph-renderer` with thin wrappers
- blocks-ui retains only: `sizingClassifier` (org-specific agent card heights), node type bindings (`'org-unit'`, `'org-agent'`)
- Pages SNAPSHOT built and installed, `.casehub-packages` updated (both `src/` and `dist/`)

**UI Enhancements:**
- Supervision hierarchy panel (blue) — aggregated supervisor → targets view
- Chip bar (ESC / SUP / ATT / LGD) — unified toggle for all panels, replaces toolbar Legend button
- Derived data enriched with `supervisionSummary` in `DerivedOrgData`

**Test results:** 151 graph-stencil-org tests, 13 org-diagram tests, 11 schema tests — all pass.

### CRITICAL: Examples Shell Breakage

Multiple examples are broken in the dev shell. **Not clear if this session caused it or pre-existing.** User has started a separate session to fix all broken examples before continuing #157 work.

Known from session 6: `.casehub-packages/packages/graph-renderer/dist/mapping.js` had unstripped `import type {}` — a casehub-pages build issue. A local workaround was applied (in gitignored `.casehub-packages`).

### What's Left To Do

1. **Fix broken examples** (separate session in progress)
2. **Edge hover tooltips** — hovering over edge labels/lines should show full label text (labels are often clipped). Needs `graph:edge:mouseenter`/`graph:edge:mouseleave` events added to pages graph-canvas (ReactFlowApp bridge). Cross-repo change.
3. **Visual QA** — verify org diagram renders correctly with all 4 archetypes after examples are fixed
4. **Run `work end`** to close the branch (code review, squash, push)

### Deferred from Issue #157

- Draggable agent repositioning within units
- Double-click inline editing
- Soft constraint scoring (bounded search over alternative rule selections)
- User overrides / pins (YAML-stored layout pinning)

### Key Files Changed (this session)

| Repo | Package | Files | What |
|------|---------|-------|------|
| pages | graph-renderer/src/layout/ | 5 new + index.ts | Generic LayoutEngine, types, FactBase, classifiers, layout-rules |
| blocks-ui | graph-stencil-org/src/layout/ | 7 modified | Thin re-exports from graph-renderer, org-specific sizing classifier |
| blocks-ui | org-diagram/src/ | 3 modified + 1 new | Engine wiring, chip bar, supervision panel |
| blocks-ui | org-diagram/src/panels/ | 1 new | supervision-chain-panel.ts |

### Commits on Branch (this session — blocks-ui)

```
1a2bde5 refactor(graph-stencil-org): delegate generic layout engine to pages graph-renderer
1d43676 feat(org-diagram): move legend to chip bar, remove toolbar legend button
7f63aa9 feat(org-diagram): add supervision panel and chip bar for panel toggles
b189ac0 test(graph-stencil-org): add archetype composition tests for layout engine
c471e2b feat(graph-stencil-org): add explain mode to layout engine
b5b418c refactor(graph-stencil-org): wire layout engine, delete old scattered layout files
d4d23de feat(graph-stencil-org): add layout transform and hard constraint rules
eb2730a feat(graph-stencil-org): add classification rules replacing archetype detection
9eed42f feat(graph-stencil-org): add OrgLayoutEngine with phase execution and composition validation
f01cbbe feat(graph-stencil-org): add layout rule engine types and FactBase
```

### Commits on Branch (this session — pages)

```
38148fa8 feat(graph-renderer): add generic layout rule engine
```
