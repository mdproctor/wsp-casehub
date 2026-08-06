# HANDOFF — casehub-blocks-ui

**Branch:** main (no active branch)
**Date:** 2026-08-05

## What landed

Generic `<status-badge>` component with a 10-domain status registry (#109). Replaces 4 ad-hoc status badge implementations (work-item-inbox, session-list, commitment-state-pill, badge-mappings) with one component and one source of truth. Registry uses cross-domain defaults so new domains get sensible rendering for shared state names (COMPLETED, PENDING, RUNNING, etc.) without explicit registration. `toDecoration()` in graph-stencil-case converts the same descriptors to graph node decorations via a separate `BADGE_COLORS` hex palette. Case-level status badge added to diagram toolbar. Consumer and contributor guides updated.

Filed 6 new epics (#106–#111) for the remaining blocks-ui modelling gaps: SWF diagram, HTN/DAG visualiser, worker function drill-down, runtime state expansion (done), conversation protocol viewer, orchestration monitor. Slot 85 created for #106 (SWF diagram).

## What's left

- pages-table pagination buttons still use light backgrounds (upstream pages fix) · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #106 | SWF diagram — complete graph-stencil-swf | L | Med | Slot 85 ready, `@openworkflowspec/sdk` available |
| #107 | HTN decomposition tree and DAG plan visualiser | L | High | Needs design |
| #108 | Worker function drill-down — agent/flow/a2a/mcp config | M | Med | Partially independent of #106 |
| #110 | Conversation protocol viewer — convergence, epistemic status | L | High | Needs design |
| #111 | Orchestration monitor — execution lifecycle, audit chain | L | High | Needs design |
