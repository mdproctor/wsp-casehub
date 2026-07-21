# HANDOFF — casehub-blocks-ui

**Branch:** issue-89-trust-workbench (closed)
**Date:** 2026-07-21
**Issue:** casehubio/blocks-ui#89

## What landed

`<trust-workbench>` — composite trust visibility component composing trust-score-panel, routing-rationale, and trust-feedback-display in a pre-wired split-workbench layout. Capability drill-down filters routing history. Three consumption tiers (drop-in, custom renderers, direct composition). 29 tests.

**Prerequisites landed alongside:**
- trust-score-panel: event topic migrated from dot to colon separator (`trust:capability-selected`), switched from raw CustomEvent to `emitPagesEvent` (payload convention), SVG gauge replaced with compact score header, trend section hidden when no data
- routing-rationale: `PHASE_STYLES` exported for downstream reuse

**Showcase:** `examples/src/pages/trust-workbench-page.ts` with full mock data layer (routing-history endpoints in mock-fetch.ts). Vite alias fix: pages-component source alias removed (source uses refactored SourceConnector API incompatible with DataSourceAdapter).

## Known issues

- **pages-component source/dist mismatch:** The pages repo source has a refactored SourceConnector API that breaks DataSourceAdapter.connect(). The examples vite config works around this by NOT aliasing pages-component to source. This needs resolving when pages-component is next released.
- **Backend routing-history endpoints:** Don't exist yet. trust-workbench works via inline data mode. Engine team needs to build `GET /trust/{actorId}/routing-history` and `GET /trust/{actorId}/routing-history/{decisionId}`.
- **Worktree + IntelliJ MCP:** IntelliJ MCP edits target the main repo, not the worktree. File syncing via shutil.copy2 was needed throughout. Garden entry GE-20260720-3573ac documents the root cause.

## Garden entries

- GE-20260720-ebe1cd — onPagesEvent callback receives payload directly, not CustomEvent
- GE-20260720-3573ac — git worktree + yarn workspace symlink resolution crosses working trees
