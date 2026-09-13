---
layout: post
title: "The Table That Hid Its Own Data"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [work-item-inbox, pages-table, column-renderers]
---

The work-item-inbox had 30+ fields on its model and showed four of them. Priority was defined but set `visible: false`. The mock data was richer than the actual UI.

This is the kind of gap that accumulates invisibly — each column was deferred because the four that shipped were enough for the demo, and nobody circled back. The `WorkItemResponse` interface kept growing (labels, percentComplete, statusNote, expiresAt, assigneeId) while the table stayed frozen at title, status, category, and created date.

The fix was straightforward. Priority became visible — arguably more important than category in a triage view. Five more columns got added as toggleable: progress, note, deadline, assignee, labels. All hidden by default, all available via the pages-table column picker that was already wired up and doing nothing useful.

The interesting part was the renderers. A progress column needs a progress bar, not a number. A labels column needs tag pills, not a comma-separated string. A deadline column needs to turn red when it's overdue. Each renderer follows the same pattern as the existing status pill — a `ColumnRenderer` function that receives a `CellValue` and returns a Lit template — but the visual treatment varies enough that copy-paste would have been wrong.

```typescript
[PERCENT_COL, (cell: CellValue) => {
  if (cell.type === 'NULL') return html`<span>—</span>`;
  const pct = (cell as { value: number }).value;
  return html`<span role="img" aria-label="${pct}% complete"
    style="display: inline-flex; align-items: center; gap: 6px;">
    <span style="width: 60px; height: 6px; border-radius: 3px;
      background: var(--pages-neutral-4); overflow: hidden;">
      <span style="height: 100%; width: ${pct}%;
        background: var(--pages-accent-9);"></span>
    </span>
    <span style="font-size: 11px;">${pct}%</span>
  </span>`;
}],
```

The progress bar renderer is the one worth showing. A 60px bar with a percentage label, ARIA markup for accessibility, and null handling that renders a dash. Compact enough for a table cell, informative enough to scan at a glance.

One thing that tripped us up: `columnId()` from pages-data returns a branded string, not an object. The TypeScript type makes it look opaque, but at runtime it's the raw string. Test assertions like `c.id.key === 'priority'` fail silently — `undefined === 'priority'` is always false. The fix is `c.id === 'priority'`. Small thing, but it burned twenty minutes before we spotted it.

The column picker infrastructure was the unsung hero here. All this work was possible because pages-table already has a built-in kebab menu with checkbox toggles. The inbox just hadn't caught up with its own model. Now it has — and the next time someone adds a field to `WorkItemResponse`, the pattern for exposing it is a three-line addition to `INBOX_COL_DEFS`, a one-line config entry, and optionally a renderer.
