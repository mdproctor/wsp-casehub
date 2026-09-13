---
layout: post
title: "The Property That Does Nothing"
date: 2026-07-29
type: phase-update
entry_type: note
subtype: diary
projects: [blocks-ui]
tags: [pages-table, lit, ci, silent-failure]
---

## The Property That Does Nothing

We spent a session chasing a visual bug in the session-workbench example page — clicking rows fired selection events (visible in the event log), the detail pane loaded content, but the table rows themselves showed no highlight. The `selectedKeys` property was correctly set on `pages-table`, the array contained the right ID, and Lit's reactivity system dutifully stored it. Nothing was visually wrong. Nothing was visually right either.

The root cause: `pages-table` defaults its `selection` attribute to `'none'`. The internal `_isRowSelected()` method gates on `selection !== 'none'` before it ever looks at `selectedKeys`. The property is accepted, stored, and silently ignored. No warning, no console message, no TypeScript error — the type system is happy with an array of strings regardless of whether anyone reads it.

The fix is one attribute: `selection="single"`. But the discovery cost was disproportionate because the failure is completely silent. A developer who sets `.selectedKeys` and sees it reflected in the component's state has no reason to suspect a second attribute is required to activate it. The property and the attribute don't reference each other in their names, types, or documentation paths.

This was one of four CI fixes that landed this session. The others were mechanical: `entity-tree` guarding `this.nodes.map()` against non-array values during test teardown, `channel-feed` using optional chaining on `scrollIntoView` (jsdom doesn't implement it), and `channel-topic-bar` having its `active` CSS class inside the `size` attribute instead of the `class` attribute — a Lit template typo where `size="sm ${condition ? 'active' : ''}"` looks syntactically fine but puts `active` in the wrong place.

We also cleaned house on three stale test files: `themes.test.ts` imported a function that was never re-exported from blocks-ui-core, and two data-source test suites tested an API contract (raw `DataSource.connect()`) that the mixin never implemented — it delegates to `DataSourceAdapter` instead. These had been silently failing or flaking depending on which CI workspace ran first.

The broader session milestone: closing the three migration tracking epics (#56, #35, #36). Every component promotion from all five consuming apps — AML, Clinical, DevTown, Claudony, OpenClaw — has landed. The openclaw issues that #36 was tracking turned out to be already closed; we just hadn't cross-checked. The blocks-ui component library has zero open issues for the first time.
