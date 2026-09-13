---
title: "The Missing UX Pattern"
date: 2026-07-13
projects:
  - casehub-blocks-ui
  - casehub-pages
tags: [design, ux, web-components, accessibility]
---

The audit-trail-viewer's row click was broken. Clicking a row set `_expandedEntryId` and called `_renderExpandedDetail`, but the detail panels rendered *after* the entire `<pages-table>` element — not between rows where they belong. The table had no mechanism to inject content between rows. Tree expansion renders child rows (same column structure). Grouped view handles section-level collapse. Neither supports arbitrary detail panels below a single row.

We went looking for prior art. Carbon, MUI, Ant Design, Spectrum, Cloudscape — all have this pattern, and they converge on the same set of decisions. Dedicated expand column on the left (not injected into the first data cell). Chevron rotation as the primary indicator. Disclosure ARIA pattern, not treegrid. The reasoning is sound: treegrid has poor assistive technology support, and injecting the toggle into the first cell breaks down the moment you combine it with tree expansion or selection checkboxes. Two chevrons in one cell is ambiguous.

The first-principles question was where the toggle belongs. Tree expansion says "this entity contains other entities" — a hierarchical relationship that belongs with the entity name in the first cell. Row-detail expansion says "this row has supplementary information" — a property of the row itself. Different semantics, different placement. A dedicated column composes cleanly with selection and tree features without any ambiguity. The 40px cost is negligible for the clarity gained.

The API is a single callback: `getRowDetail: (row: TypedRow) => TemplateResult | undefined`. Return a template and the row gets an expand toggle. Return `undefined` and it doesn't. Virtual scrolling is disabled when the callback is set — detail panels have variable height, and the fixed-`rowHeight` scroll engine can't position them. That's by design, not a limitation. Row-detail expansion targets curated datasets where virtual scroll isn't needed.

Single-expand is the default (accordion — one panel open at a time). Multi-expand is opt-in for power users who need to compare entries side by side. Controlled mode via `expandedDetailKeys` follows the existing `selectedKeys` pattern.

The spec lives in pages now, with the [issue](https://github.com/casehubio/casehub-pages/issues/172). When pages implements it, the blocks-ui fix is a one-line change — pass a callback instead of rendering detail panels outside the table.
