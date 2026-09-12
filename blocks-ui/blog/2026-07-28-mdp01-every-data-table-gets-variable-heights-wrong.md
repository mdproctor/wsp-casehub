---
layout: post
title: "Every Data Table Gets Variable Heights Wrong"
date: 2026-07-28
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [casehub-pages, blocks-ui, css-grid, virtual-scroll, data-table, column-resize]
---

## The table that needed to grow up

`pages-data-table` started as a CSS Grid table with virtual scrolling, sorting, selection, and keyboard navigation. A few weeks ago we added cell spanning — the single-grid model where `.body-content` is one CSS Grid container, row wrappers use `display: contents`, and cells are placed with explicit `grid-row` / `grid-column` styles. That gave us native `grid-row: span 3` for rowspan instead of the position hacks every other library uses.

But two things were explicitly deferred from the spanning work: variable row heights and column resizing. Both interact with the single-grid model in ways that needed their own design.

## Why variable heights is harder than it looks

The scroll engine uses a scalar `rowHeight` to compute everything. Position of row N is `N * height`. Index from a scroll offset is `floor(scrollTop / height)`. O(1), simple, fast. Variable heights break that — you need prefix sums for offset lookup and binary search for index-from-offset. O(N) construction, O(log N) lookups.

But the real question is: where do heights come from?

Three models, each with different compatibility:

```typescript
rowHeight: number | 'auto' | ((row: TypedRow, index: number) => number)
```

A fixed number is the current behaviour. A callback lets the data determine height — section headers at 32px, data rows at 48px, summary rows at 64px. Heights are known without rendering, so virtual scroll works via prefix sums.

The interesting case is `'auto'` — content-determined heights where rows size to their rendered content. Heights are unknown until after paint. AG Grid handles this with measure-and-extrapolate: render with an estimated height, measure after paint, update the model, correct the scroll position. We do the same, but the single-grid model gives us something AG Grid doesn't have.

## The split template trick

With a single CSS Grid, we can use *different track types* in the same template. Unrendered rows above and below the viewport get explicit pixel tracks (the current estimate). Rendered viewport rows get CSS `auto` tracks — the browser sizes them to content natively. After paint, `getComputedStyle(bodyContent).gridTemplateRows` returns the resolved pixel values for every track, including the `auto` ones that are now content-measured.

```
grid-template-rows: repeat(100, 52px) repeat(15, auto) repeat(885, 52px)
                    ↑ above viewport    ↑ rendered       ↑ below viewport
```

The measured heights feed back into the model. The estimate converges within two or three viewports of scrolling. Scroll correction — adjusting `scrollTop` when above-viewport measurements change — prevents visible jumps. A `_isCorrectingScroll` flag breaks the feedback loop where the programmatic `scrollTop` change fires the scroll handler.

## HeightModel — three strategies, one interface

```typescript
interface HeightModel {
  readonly totalHeight: number;
  readonly rowCount: number;
  rowHeight(index: number): number;
  offsetAtIndex(index: number): number;
  indexAtOffset(scrollTop: number): number;
}
```

`FixedHeightModel` wraps the scalar — identical behaviour to the old code. `CallbackHeightModel` builds prefix sums from the callback. `MeasuredHeightModel` tracks measured heights alongside a running estimate, with lazy prefix sums that invalidate on new measurements.

`computeScrollWindow` takes a `HeightModel` instead of a scalar. Every call site migrates — `_scrollToRowIfNeeded`, `_onScroll` load-more detection, the grid template generation. The key refactoring was separating `_effectiveRows` (the filtered, sorted, tree-expanded row set) from `_visibleRows` (the viewport window). The HeightModel operates on `_effectiveRows`, not the raw dataset — so filtering 100 rows to 5 means the scroll engine sees 5 rows, not 100.

## Column resizing — simpler than expected

The single-grid model makes this straightforward. One `grid-template-columns` change on `.body-content` resizes every row atomically. No per-row sync. The header is a separate grid container using the same computed template.

Resize handles are 6px hit zones on the right edge of header cells, pointer-captured during drag. Resized columns convert to `px`; untouched columns keep their original unit (`fr`, `minmax()`, whatever the consumer specified). The grid engine redistributes space naturally.

The only subtlety is header-body horizontal scroll sync — when total column width exceeds the container, `_onScroll` reads `scrollLeft` and applies `translateX` to the header. Column resizing makes this first-class because resized columns are likely to create overflow.

## What the review caught

I ran a full adversarial design review — six rounds, two independent sessions debating the spec. The biggest finding: the original measurement approach was fundamentally wrong. Explicit pixel tracks (`grid-template-rows: 48px 48px ...`) don't auto-size to content — `getComputedStyle` returns the declared values, not measured ones. The split template (px for unrendered, `auto` for rendered) was the fix.

The review also caught that the `_dataRows` → `_effectiveRows` migration had blind spots in the keyboard handler (shift-select after client sort would select the wrong row) and span selection (spanned cells would check unsorted row indices). Pre-existing bugs, but the refactoring was the natural fix point.

The HeightModel operates on effective rows, SpanMap computes from effective rows, keyboard navigation indexes into effective rows. One abstraction, consistently applied.
