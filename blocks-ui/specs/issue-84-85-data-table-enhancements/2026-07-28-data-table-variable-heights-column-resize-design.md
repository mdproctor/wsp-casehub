# Data Table: Variable Row Heights + Column Resizing

**Issues:** casehubio/blocks-ui#84, casehubio/blocks-ui#85
**Date:** 2026-07-28
**Status:** Draft
**Implementation target:** `@casehubio/pages-table` in casehub-pages
**Depends on:** #26 (cell spanning — implemented)

## Context

Both features were deferred from the cell-spanning design (#26). The single-grid
model introduced for spanning (`display: grid` on `.body-content`, `display: contents`
on row wrappers) creates the foundation for both:

- **Variable row heights (#84):** The virtual scroll engine uses
  `computeScrollWindow(scrollTop, containerHeight, rowHeight, rowCount, bufferSize)`
  with a fixed scalar `rowHeight`. Position math is O(1): `position = index * height`.
  Variable heights require a different data structure and algorithm.

- **Column resizing (#85):** The single-grid model means one `grid-template-columns`
  change on `.body-content` resizes all rows atomically. No per-row sync needed. The
  header is a separate grid container that shares the same computed template string.

### Prior art

| Library | Variable heights | Column resizing |
|---------|-----------------|-----------------|
| AG Grid | `autoHeight` — measure after render, estimate unrendered rows, correct scroll position | Drag handles, `colDef.resizable`, real-time update |
| Ant Design | Fixed heights only (virtual scroll) or auto-height (non-virtual) | `column.ellipsis` + resize via `react-resizable` |
| TanStack Table | Virtualizer accepts `estimateSize` + `measureElement` | Header-only — consumer implements via CSS |

### Garden entries considered

- **GE-20260719-4db710:** CSS Grid single-container virtual scroll technique. Documents
  the constraint: "Requires fixed row heights for scroll position math." This spec
  removes that constraint.
- **GE-20260727-e642b2:** `row-activate` event carries `{ row, key }`, not `{ index }`.
  Not directly affected by these changes — event emission is index-independent.

## Architecture: HeightModel Abstraction

The core change. The scroll engine currently takes a scalar `rowHeight`. It must work
against an interface that abstracts three height strategies:

```typescript
interface HeightModel {
  readonly totalHeight: number;
  rowHeight(index: number): number;
  offsetAtIndex(index: number): number;
  indexAtOffset(scrollTop: number): number;
}
```

### FixedHeightModel

Wraps the current scalar. O(1) everything.

```
totalHeight    = count * height
rowHeight(i)   = height
offsetAtIndex  = index * height
indexAtOffset  = floor(scrollTop / height)
```

This is the existing behavior behind an interface. Zero behavioral change for
`rowHeight: number`.

### CallbackHeightModel

Wraps a `(row: TypedRow, index: number) => number` callback. Builds a prefix sum
array on construction.

```
heights[i]     = callback(rows[i], i)
prefixSums[0]  = 0
prefixSums[i]  = prefixSums[i-1] + heights[i-1]
totalHeight    = prefixSums[N]
offsetAtIndex  = prefixSums[index]           — O(1)
indexAtOffset  = binarySearch(prefixSums, y)  — O(log N)
```

Recomputes when data changes (new `dataSet`, sort, filter). O(N) construction,
O(1) / O(log N) lookups.

Validation: callback results clamped to minimum 1px (zero or negative heights
produce invisible rows).

### MeasuredHeightModel

The auto-height engine. Tracks measured heights and a running estimate.

```typescript
class MeasuredHeightModel implements HeightModel {
  private measured: Map<number, number>;
  private estimate: number;          // average of measured, seeded at 48px
  private count: number;
  private _prefixSums: number[] | null; // lazily computed, invalidated on change

  recordHeight(index: number, height: number, key?: string): boolean; // key for remap
  remap(keyToNewIndex: Map<string, number>): void;        // preserve measurements across sort
  reset(): void;                                          // clear all, re-seed estimate
}
```

- `rowHeight(i)` — returns `measured.get(i) ?? estimate`
- `offsetAtIndex(i)` — prefix sum using measured where known, estimate elsewhere
- `indexAtOffset(y)` — binary search over prefix sums
- `recordHeight(i, h, key?)` — stores measurement (indexed by `i`, keyed by `key`
  when provided for remap support), updates running estimate (average of all
  measurements), invalidates prefix sums. The `key` comes from `getRowKey(row)`.
- `remap(keyToNewIndex)` — on sort/filter, rebuilds the index→height map by
  looking up each key's new index. Measurements for rows whose key is absent
  (filtered out) are retained but inactive. Requires `getRowKey` — without it,
  falls back to `reset()`.
- `reset()` — on `dataSet` change (or sort/filter without `getRowKey`), clears
  everything, re-seeds estimate to 48px

Lazy prefix sums: `_prefixSums` is computed on first access after invalidation.
Multiple `recordHeight` calls batch into a single recomputation.

## `rowHeight` API

The `rowHeight` property becomes polymorphic:

```typescript
rowHeight: number | 'auto' | ((row: TypedRow, index: number) => number)
```

Default: `48` (backward compatible).

### Property decorator

The current `@property({ type: Number, attribute: 'row-height' })` uses Lit's
`type: Number` converter, which converts `"auto"` to `NaN`. The decorator must
use a custom converter to handle the polymorphic type:

```typescript
@property({
  attribute: 'row-height',
  converter: {
    fromAttribute(value: string | null): number | 'auto' {
      if (value === 'auto') return 'auto';
      const n = Number(value);
      return Number.isNaN(n) || n <= 0 ? 48 : n;
    },
    toAttribute(value: number | 'auto' | Function): string | null {
      if (typeof value === 'function') return null;
      return String(value);
    },
  },
}) rowHeight: number | 'auto' | ((row: TypedRow, index: number) => number) = 48;
```

- `row-height="48"` — backward compatible, parsed as number
- `row-height="auto"` — parsed as `'auto'` string
- Functions — property-only (`toAttribute` returns `null`, no HTML attribute)

### Effective row computation

Currently, `_visibleRows` mixes two concerns: computing the effective row set
(filter, sort, tree expansion) and selecting the visible window (pagination,
virtual scroll). These are separated:

`willUpdate` computes `_effectiveRows` — the complete ordered, filtered row set:

1. Start from `_dataRows` (the raw data set)
2. Apply tree flattening / expand-collapse (if `getChildren` or expandable config)
3. Apply client-side sort (if `_sortStack` is set)
4. Apply client-side filter (if filter text is set)

`_effectiveRows` is the input to both the HeightModel and the virtual scroll
threshold. `_visibleRows` becomes a simple window into `_effectiveRows`:
- Paginated → slice by page
- Virtual scroll → slice by scroll window (from HeightModel)
- Auto ≤ threshold → return all effective rows

This separation resolves three issues: the HeightModel sees the correct row count,
sort/filter order is available for remap before render, and the virtual scroll
threshold reflects the actual displayable row count.

### HeightModel lifecycle

The component stores `_heightModel: HeightModel` as persistent instance state.
Construction and update rules in `willUpdate`, keyed on `_effectiveRows`:

| Trigger | FixedHeightModel | CallbackHeightModel | MeasuredHeightModel |
|---------|-----------------|--------------------|--------------------|
| `rowHeight` type changed | Construct new | Construct new | Construct new |
| `dataSet` changed (replacement) | Reconstruct with new count | Reconstruct (new prefix sums) | `reset()` — clear measurements, re-seed estimate |
| `dataSet` changed (load-more append) | Reconstruct with new count | Reconstruct (extend prefix sums) | Preserve existing measurements, extend count for new rows at estimate |
| `_effectiveRows` changed (sort/filter/tree) | Reconstruct with new count | Reconstruct prefix sums from reordered rows | `remap(keyToNewIndex)` if `getRowKey` available; `reset()` otherwise |

**Load-more append detection:** When `dataSet` changes while `_loadingMore` is
`true` (set by `_onScroll` when it emits the `load-more` event), the new data is
an append — existing rows are unchanged, new rows are added at the end. The
component already tracks this state. For MeasuredHeightModel, existing
measurements are preserved and new rows start at the current estimate. This
avoids scroll-position jumps and estimate regression during infinite scroll.

The model is **not** reconstructed on every render. `willUpdate` checks whether
the `rowHeight` type has changed (number ↔ `'auto'` ↔ function) and only
constructs a new model on type transitions. Within the same type, the existing
model is reused and updated in place.

Because `_effectiveRows` is computed in `willUpdate` (not lazily in render),
the sorted/filtered row order is available for `remap()`. The key-to-new-index
mapping is built from `_effectiveRows` using `getRowKey`.

**`getRowKey` fallback for MeasuredHeightModel:** `getRowKey` is not required
for `rowHeight: 'auto'` — but without it, sort/filter operations trigger
`reset()` instead of `remap()`, wiping all accumulated measurements and causing
full re-measurement on the next render. For best performance with interactive
sort/filter, provide `getRowKey` when using `rowHeight: 'auto'`.

### Mode interaction

Every combination of `rowHeight` and `mode` is valid. No errors, no fallbacks.

The `_useVirtualScroll` threshold checks `_effectiveRows.length > AUTO_THRESHOLD`
(not `_dataRows.length`). If 100 data rows are filtered to 5, the component uses
non-virtual mode and CSS `auto` tracks — no measurement cycle for 5 rows.

| `rowHeight` | `mode='auto'` ≤50 | `mode='auto'` >50 | `mode='paginated'` | `mode='scroll'` |
|-------------|-------------------|-------------------|---------------------|------------------|
| `number` | all rows, fixed tracks | virtual scroll | paginated, fixed tracks | virtual scroll |
| `'auto'` | all rows, CSS `auto` tracks | virtual scroll + measurement | paginated, CSS `auto` tracks | virtual scroll + measurement |
| `function` | all rows, CSS `auto` tracks | virtual scroll + prefix sum | paginated, CSS `auto` tracks | virtual scroll + prefix sum |

The threshold (≤50 / >50) is evaluated against `_effectiveRows.length`. In
non-virtual modes (all rows rendered), `grid-template-rows` uses CSS `auto`
regardless of `rowHeight` type — the browser sizes everything correctly from
content. The HeightModel is not consulted.

In virtual scroll modes, `grid-template-rows` uses explicit pixel values from the
HeightModel — the browser needs known track sizes for scroll height computation.

### `computeScrollWindow` signature change

```typescript
// Before
computeScrollWindow(scrollTop, containerHeight, rowHeight, rowCount, bufferSize)

// After
computeScrollWindow(scrollTop, containerHeight, heightModel, bufferSize)
```

Row count comes from `heightModel.totalHeight / heightModel.rowHeight(0)` — but
actually the model encapsulates this. The function uses `heightModel.indexAtOffset`
for start index and `heightModel.offsetAtIndex` for offset calculations.

The function remains pure — HeightModel is a read-only data structure.

### Call-site migration inventory

All existing `this.rowHeight` scalar usages and their replacements:

| Location | Current usage | After |
|----------|--------------|-------|
| `_onScroll` (load-more detection) | `this.bufferSize * this.rowHeight` | `this.bufferSize * this._heightModel.rowHeight(0)` — estimate-based buffer; exact per-row precision is unnecessary for near-bottom detection |
| `_scrollToRowIfNeeded` | `rowIndex * this.rowHeight` / `rowTop + this.rowHeight` | `this._heightModel.offsetAtIndex(rowIndex)` / `offset + this._heightModel.rowHeight(rowIndex)` |
| `_visibleRows` getter | `computeScrollWindow(..., this.rowHeight, rows.length, ...)` | `computeScrollWindow(..., this._heightModel, ...)` |
| `_scrollWindow` getter | `computeScrollWindow(..., this.rowHeight, this._dataRows.length, ...)` | `computeScrollWindow(..., this._heightModel, ...)` |
| Render template (`grid-template-rows`) | `repeat(${count}, ${this.rowHeight}px)` | `this._gridTemplateRows` — cached string from HeightModel |
| `willUpdate` error message | `"virtual scrolling requires fixed row heights"` | `"getRowDetail is incompatible with mode='scroll'"` — message was about detail panels requiring non-virtual mode, not about row heights |

### `_dataRows` → `_effectiveRows` migration inventory

All `_dataRows` usages that index by position must migrate to `_effectiveRows`
after the refactoring. The `_effectiveRows` array is the sorted/filtered/tree-
expanded row set — the same order the render loop uses.

| Location | Current code | After | Why |
|----------|-------------|-------|-----|
| Keyboard navigation bounds (line ~1740) | `const totalRows = this._dataRows.length` | `this._effectiveRows.length` | With client filter active (100 rows filtered to 10), `totalRows` stays 100, allowing `rovingIndex` to exceed the visible row count |
| Keyboard shift-select (lines ~1756, ~1780) | `const row = this._dataRows[this.rovingIndex]` | `this._effectiveRows[this.rovingIndex]` | With client sort, `rovingIndex` 5 maps to sorted position 5 — `_dataRows[5]` is the wrong row (original order) |
| Span covered-row selection (line ~2586) | `const coveredRow = this._dataRows[actualIndex + j]` | `this._effectiveRows[actualIndex + j]` | With client sort + `mergeRows`, `actualIndex` is an index into the sorted row set — `_dataRows` reads unsorted data |
| SpanMap computation (line ~1256) | `computeSpanMap(this._dataRows, ...)` | `computeSpanMap(this._effectiveRows, ...)` | Covered by R3-02 — mergeRows spans must reflect sorted adjacency |
| Virtual scroll threshold | `this._dataRows.length > AUTO_THRESHOLD` | `this._effectiveRows.length > AUTO_THRESHOLD` | Covered by R2-05 — filtered count determines virtual scroll activation |

The first three are pre-existing bugs with client sort — they produce wrong
behaviour today when sort is active. The `_effectiveRows` refactoring is the
natural fix point.

### Pipeline integration

The `set props()` handler gains clauses for new and changed properties:

```typescript
if (p.resizable === true) this.resizable = true;

if (p.rowHeight !== undefined) {
  this.rowHeight = p.rowHeight as number | 'auto' | ((row: TypedRow, index: number) => number);
}
```

`DataTableProps` already declares `resizable?: boolean` — the component just
needs to consume it. `rowHeight` is not currently in `DataTableProps`; it will
be added there as part of this work.

### Grid template generation

| HeightModel | `grid-template-rows` value |
|-------------|--------------------------|
| Fixed | `repeat(${count}, ${height}px)` — current behavior |
| Callback | Explicit per-row: `${heights[0]}px ${heights[1]}px ...` |
| Measured (virtual scroll) | Split: explicit px for unrendered rows, `auto` for rendered viewport rows (see §Measurement Lifecycle) |

For callback and measured models, the explicit template is a string of N height
values. For 100K rows this is ~70KB. Browser track sizing stores this as an array
of floats — O(N) memory, efficiently represented. The garden entry (GE-20260719-4db710)
confirms browsers handle 100K+ tracks without degradation.

The `_gridTemplateRows` string is cached and invalidated only when the HeightModel
changes (new measurements, data change, or model reconstruction). Reactive state
changes that don't affect heights (e.g., column resize for Fixed/Callback models)
do not trigger recomputation.

## Measurement Lifecycle (Auto-Height)

When `rowHeight: 'auto'` and virtual scroll is active, the component runs a
measure-correct cycle:

### Frame 1 — initial render

- MeasuredHeightModel seeded with 48px estimate for all rows
- `grid-template-rows` uses a **split template**: explicit `48px` for unrendered
  rows (above and below viewport), `auto` for rendered viewport rows
- First viewport of cells rendered into the grid
- Rendered cells are placed via `grid-row` into the `auto` tracks, which size
  to fit content naturally

### Frame 2 — measure and correct

- `updated()` lifecycle fires after paint
- Read resolved track sizes: `getComputedStyle(bodyContent).gridTemplateRows`
  returns pixel values for all tracks — the `auto` tracks are now resolved to
  their content-determined sizes (e.g., `"48px 48px 72px 56px 64px 48px ..."`)
- Extract resolved heights for the rendered viewport range (indices
  `startIndex` to `endIndex`)
- Call `recordHeight(index, measuredHeight, key)` for each rendered row
- Running estimate updates to average of all measurements
- Recompute `grid-template-rows` with measured heights for all known rows,
  updated estimate for unknown rows, and `auto` for currently-rendered rows
- Scrollbar thumb may adjust slightly (total height changed) — one-time,
  barely perceptible

### Subsequent scrolls

- Each scroll renders new rows → `updated()` measures them → model refines
- After ~2-3 viewports of scrolling, estimate converges to true average
- Scroll position correction runs when measurements above viewport differ
  from estimates

### Scroll position correction

When measurements for rows above the current viewport arrive (user scrolled up
into previously-estimated territory), the total height of above-viewport content
changes. Without correction, the visible content shifts.

```
delta = sum(measuredHeight[i] - previousHeight[i])
        for all i where i < firstVisibleRow and measurement is new
scrollTop += delta
```

Applied before the next paint. The user sees no jump — the viewport content
stays in place, only the scrollbar position adjusts.

**Feedback loop guard:** Setting `scrollTop` programmatically fires the `scroll`
event handler. A `_isCorrectingScroll` flag is set before the adjustment and
cleared immediately after. `_onScroll` checks this flag and skips `_scrollTop`
state update when set. This breaks the render → measure → correct → scroll →
render cycle.

### Data change handling

- New `dataSet` (replacement) → `MeasuredHeightModel.reset()` → clears
  measurements, re-seeds estimate to 48px, full re-measurement on next render
- New `dataSet` (load-more append, detected via `_loadingMore` flag) → preserve
  existing measurements, extend model count for appended rows at current estimate.
  No scroll position disruption, no estimate regression.
- Sort/filter → `MeasuredHeightModel.remap(keyToNewIndex)` → preserves measured
  heights at their new indices via row key mapping
- Column visibility change → no effect on row heights (column axis is independent)

### Detail panel interaction

Detail panels use `auto`-height tracks in the paired pattern (`varHeight auto`).
With variable data-row heights, the pattern becomes per-row heights interleaved
with auto tracks. The detail track stays `auto`. No conflict.

The existing `_gridRowFor(actualIndex)` method handles the track-doubling pattern
for detail panels — data rows on odd tracks (`2 * actualIndex + 1`), detail panels
on even tracks. With variable heights, only the data track portion changes; the
`auto` detail tracks are unaffected. This mechanism is unchanged.

### Non-virtual modes

When all rows are rendered (paginated, auto ≤50), `grid-template-rows` uses CSS
`auto` directly. No HeightModel, no measurement cycle. The browser handles
everything natively. This is simpler and correct — there's no need to estimate
when every row is in the DOM.

## Column Resizing

### Table-level property

```typescript
resizable: boolean  // default: false
```

When `true`, all data columns get resize handles. Prefix columns (checkbox at
40px, expand at 40px) are component-generated and never resizable.

### Resize handles

6px-wide hit zones on the right edge of each header cell. Implemented as
positioned elements within the header cell (pseudo-elements or divs). CSS
`cursor: col-resize` on hover. No extra DOM in the body — column widths are
controlled by the grid template.

### Drag behavior

1. `pointerdown` on handle — capture pointer, record start X and starting
   column width (resolved to px from computed style)
2. `pointermove` — compute delta from start X, compute new width, clamp to
   resolved `minWidth` (see §Constraints), update internal column width state
3. `pointerup` — release capture, emit `column-resize` event
4. `pointercancel` — release capture, revert to pre-drag width, no event

During drag, the component maintains `_columnWidths: Map<string, string>` that
overrides `ColumnDef.width` for resized columns. The overridden width is always
`px`. Untouched columns keep their original unit (`fr`, `minmax()`, etc.).

`_gridTemplateColumns` reads from this map, falling back to column config:
```typescript
get _gridTemplateColumns(): string {
  const columns = this._visibleColumns.map(c => {
    const id = String(c.id);
    const override = this._columnWidths.get(id);
    if (override) return override;
    const config = this._configFor(c);
    return config?.width ?? '1fr';
  }).join(' ');
  // ... prefix columns as before
}
```

Real-time update: each `pointermove` updates reactive state, triggering re-render
of both header and body `grid-template-columns`. Lit batches within a frame.

### Width representation

Resized columns become `px`. Untouched columns keep their original unit. The
grid engine handles redistribution — `fr` columns absorb the space change
automatically. This preserves the consumer's original layout intent for
non-resized columns.

### Event

```typescript
interface ColumnResizeDetail {
  readonly columnId: string;
  readonly width: number;  // new width in px
}
```

Event: `column-resize`, `bubbles: true`, `composed: true`. Emitted on `pointerup`
(resize end), not during drag. The component does not persist widths — the consumer
receives the event, updates their column config, and provides new widths on the
next render cycle. Same pattern as `column-change` for visibility.

### Double-click auto-fit

Double-click on a resize handle measures the widest rendered content in that
column (visible rows only — no off-screen rendering) and sets the column width
to `max(contentWidth + padding, minWidth)`. Emits `column-resize`.

Measurement: iterate visible cells in the column, read
`cell.scrollWidth + padding`. For virtual scroll, only the rendered viewport
is measured — the user sees these rows and can judge the result.

### Drag performance

During column resize drag, only `grid-template-columns` changes. For Fixed and
Callback HeightModels, row heights are data-determined and unaffected by column
widths — `_gridTemplateRows` is unchanged and not recomputed.

For MeasuredHeightModel, column resize triggers content reflow which may change
row heights. Re-measurement happens in `updated()` after the current frame, not
during the `pointermove` handler. The `_gridTemplateRows` update is deferred to
the next render cycle after measurement completes — the drag frame only pays the
cost of `_gridTemplateColumns` recomputation.

### Constraints

- `minWidth` from `TableColumnConfig` — enforced during drag and auto-fit.
  `TableColumnConfig.minWidth` is typed as `string` (a CSS value like `"50px"`,
  `"5em"`). At `pointerdown` (drag start), the string is resolved to pixels:
  parse numeric `px` values directly; for other units, read the column cell's
  computed min-width via `getComputedStyle`. The resolved pixel value is cached
  for the duration of the drag. Default 50px when `minWidth` is not specified.
- No `maxWidth` enforcement — columns can grow freely. The table's horizontal
  overflow handles excess width.

### Header-body horizontal scroll synchronization

The header (`.header-container`) and body (`.body`) are separate containers. The
body has `overflow-x: auto`. When total column width exceeds the container, the
body scrolls horizontally but the header stays fixed — columns appear misaligned.

Fix: the `_onScroll` handler reads `scrollLeft` from the body element and applies
`transform: translateX(-${scrollLeft}px)` to the `.header` element. The
`.header-container` gains `overflow: hidden` to clip the translated header at the
container edge. This is compositor-driven (no layout thrashing) and keeps header
columns aligned with body columns during horizontal scroll.

This is a pre-existing gap that column resizing makes first-class — resized columns
are likely to create horizontal overflow.

### Resize override lifecycle

| Event | Effect on `_columnWidths` |
|-------|--------------------------|
| Column drag completes | Override set for dragged column |
| New `columnConfig` provided | All overrides cleared (consumer provides new widths via config) |
| Column hidden | Override preserved in map, excluded from template |
| Column shown | Override restored from map |
| `dataSet` changes | Overrides preserved (column widths are independent of data) |
| Sort/filter | Overrides preserved |

## Feature Interaction

### Variable heights + column resizing

Independent axes — `grid-template-rows` and `grid-template-columns` don't
interact in fixed/callback height modes.

In `'auto'` mode, column resize changes content wrapping, which changes row
heights. The MeasuredHeightModel handles this naturally: resize → re-render →
`updated()` re-measures → model updates → `grid-template-rows` recomputed.
No special case needed — the measurement cycle absorbs any height changes.

### Variable heights + cell spanning

A spanned cell's visual height is the sum of its covered rows' heights:
- FixedHeightModel: `rowSpan * height`
- CallbackHeightModel: `prefixSums[origin + rowSpan] - prefixSums[origin]`
- MeasuredHeightModel: same prefix sum math using measured/estimated values

CSS Grid handles the visual rendering natively — `grid-row: span 3` spans
three tracks regardless of their individual sizes.

`extendWindowForSpans` is unchanged — it operates on row indices, not pixel
offsets. The HeightModel converts indices to offsets when needed.

**SpanMap consistency:** `computeSpanMap` must operate on `_effectiveRows`, not
`_dataRows`. After client-side sort, the row order in `_effectiveRows` differs
from `_dataRows` — a SpanMap built from the unsorted order produces incorrect
`mergeRows` spans (adjacent rows in the sorted order may have the same value
and should merge, but the SpanMap wouldn't see them as adjacent). The SpanMap
is recomputed in `willUpdate` whenever `_effectiveRows` changes (sort, filter,
tree, or data change), not just on `dataSet`/`columnConfig`/`hiddenColumns`.

### Column resizing + cell spanning

A spanned cell's width spans N column tracks. Resizing one of those columns
changes the cell's total width proportionally. CSS Grid handles this natively.

### Variable heights + detail panels

Detail panels use `auto`-height tracks. Variable data-row heights change the
data track portion of the paired pattern. No conflict — detail tracks remain
`auto`. The existing `_gridRowFor(actualIndex)` method handles the track-doubling
pattern — data rows on odd tracks, detail panels on even tracks. With variable
heights, only the data track portion changes.

Detail panels remain incompatible with virtual scroll (existing constraint,
preserved).

### All three combined

Auto-height + column resize + spanning: column resize triggers content reflow →
row heights change → measurement cycle fires → SpanMap uses updated heights for
offset calculations. The pipeline is self-correcting.

## Backward Compatibility

Fully backward compatible. The default `rowHeight: 48` produces identical
behavior through FixedHeightModel. `resizable` defaults to `false`. No consumer
changes required.

| Existing usage | Effect |
|---------------|--------|
| `rowHeight: 48` (explicit or default) | FixedHeightModel — identical behavior |
| `mode: 'auto'` | Same threshold, same rendering |
| `mode: 'scroll'` | Same virtual scroll |
| `mode: 'paginated'` | Same pagination |
| All selection modes | Unchanged |
| All span configurations | Unchanged |
| All tree/detail/groupBy features | Unchanged |

## Testing Strategy

### Layer 1 — Regression (existing behavior unchanged)

Every feature that flows through the scroll engine or render pipeline must produce
identical results when `rowHeight` is a number.

| Area | Tests |
|------|-------|
| Virtual scroll | Fixed height render, scroll to middle, scroll to end, scroll to top |
| Pagination | Client-side and server-side page navigation |
| Selection | Single, multi, checkbox, shift-click, select-all, controlled mode |
| Selection + virtual scroll | Selection survives scroll away and back |
| Client sort/filter | Sort cycle, filter match/no-match, combined |
| Cell spanning | mergeRows, cellSpan, suppressed cells, overlap validation |
| Span + virtual scroll | Boundary scan finds origins above viewport |
| Tree rows | Expand/collapse, nesting, tree + pagination |
| Detail panels | Expand/collapse, animation, auto-height |
| Column visibility | Toggle, picker, column-change event, grid template update |
| Keyboard navigation | All keys (Arrow, Home/End, Ctrl+Home/End, Enter, Escape, Space) |
| Keyboard + virtual scroll | ArrowDown past viewport triggers scroll-into-view |
| groupBy | Group headers, per-group rows, boundaries |
| Row styling | getRowClass, part attributes, getRowAccent |
| Loading/error/empty states | Correct rendering, no grid involvement |
| ARIA | All roles, aria-rowcount, aria-rowindex, aria-sort, aria-selected |
| Horizontal scroll | Overflow, header-body sync |
| Hover | Row hover, span hover, mouseleave reset |
| Events | row-activate, sort-change, page-change, selection-change, load-more |
| Resize observer | Container resize updates _containerHeight |

### Layer 2 — Variable row heights

**Unit tests — HeightModel:**

| Test | What it verifies |
|------|-----------------|
| FixedHeightModel — all methods | Matches scalar arithmetic |
| CallbackHeightModel — uniform callback | Degenerates to FixedHeightModel results |
| CallbackHeightModel — varying heights | Prefix sums correct |
| CallbackHeightModel — binary search | Correct index at exact boundary, mid-row, between rows |
| CallbackHeightModel — recompute on data change | New rows → new prefix sums |
| MeasuredHeightModel — initial state | All rows return 48px estimate |
| MeasuredHeightModel — after measurements | Measured rows return recorded, unmeasured return updated average |
| MeasuredHeightModel — estimate convergence | Measuring rows of ~72px → estimate converges to ~72px |
| MeasuredHeightModel — mixed prefix sums | Sums from heterogeneous measured/estimated |
| MeasuredHeightModel — invalidation | recordHeight invalidates, next access recomputes |
| MeasuredHeightModel — data reset | New dataset clears measurements, re-seeds estimate |
| MeasuredHeightModel — re-measurement | Same row, different height: second overwrites |
| MeasuredHeightModel — remap on sort | Measurements preserved at new indices by key |
| MeasuredHeightModel — remap without getRowKey | Sort triggers reset(), all rows return estimate |
| MeasuredHeightModel — load-more append | Existing measurements preserved, new rows at estimate |
| computeScrollWindow — CallbackHeightModel | Correct window for non-uniform heights |
| computeScrollWindow — MeasuredHeightModel | Valid window with mixed measured/estimated |
| computeScrollWindow — buffer with tall rows | Buffer covers correct pixel range |
| extendWindowForSpans — variable heights | Span origin above viewport found correctly |
| Scroll correction — delta above viewport | Correct scrollTop adjustment |
| Scroll correction — no delta below viewport | No adjustment for below-viewport measurements |
| Scroll correction — no delta at viewport start | Visible row measurements don't shift |
| Grid template — callback heights | Correct per-row px values |
| Grid template — measured heights | Measured rows use real, unmeasured use estimate |
| Validation — callback returns 0 | Clamped to 1px minimum |
| Validation — callback returns negative | Clamped to 1px minimum |
| Validation — callback throws | Error propagated with row context |

**Visual tests:**

| Test | Setup | Action | Assert |
|------|-------|--------|--------|
| Auto-height non-virtual | 20 rows, `rowHeight: 'auto'`, varying text | None | Rows sized to content |
| Auto-height virtual scroll | 500 rows, `rowHeight: 'auto'` | Scroll to middle | Correct sizes, no gaps |
| Auto-height scroll up | Same, at row 100 | Scroll to top | No content jump |
| Auto-height fast scroll | Same | Rapid scroll top to bottom | Clean layout at rest |
| Auto-height scroll to end | Same | Scroll to bottom | Last row visible, no blank |
| Auto-height wrapping text | 200 rows, paragraph column | Scroll through | Full text visible per row |
| Auto-height convergence | 1000 rows, ~64px actual | Scroll 3 viewports | Scrollbar stabilises |
| Callback heights | 200 rows, alternating 72/48 | Scroll through | Alternating heights visible |
| Callback + spanning | 50 rows, callback, mergeRows | None | Span height = sum of covered |
| Auto-height + spanning | 30 rows, auto, mergeRows | None | Correct cumulative height |
| Auto-height + span + virtual | 500 rows, auto, mergeRows | Scroll into mid-span | Origin rendered correctly |
| Auto-height paginated | 100 rows, auto, paginated | Navigate pages | Auto-height per page |
| Variable heights + keyboard | 200 rows, callback | ArrowDown | Focus moves, tall rows scroll fully |
| Fixed height regression | 50 rows, `rowHeight: 48` | Render | Pixel-identical to pre-change |

### Layer 3 — Column resizing

**Unit tests:**

| Test | What it verifies |
|------|-----------------|
| Template with resize override | Resized → px, others → original unit |
| Override + fr columns | Grid redistributes correctly |
| Override + minmax columns | Coexistence with px override |
| Min-width clamping | Drag below minWidth → clamped |
| Min-width default | No minWidth → 50px default |
| Override persistence | Data change doesn't clear overrides |
| Override reset on config change | New columnConfig clears overrides |
| Override + hidden column | Hide/show preserves override |
| Override + column reorder | ID-based mapping, not positional |
| Event detail | Correct columnId and px width |
| Event timing | Fires on pointerup only |
| Checkbox prefix | Checkbox column not resizable |
| Expand prefix | Expand column not resizable |
| Pointer capture/release | Drag outside component works |
| Pointer cancel | Reverts, no event |
| `resizable: false` | No handles, no handlers |

**Visual tests:**

| Test | Setup | Action | Assert |
|------|-------|--------|--------|
| Handles visible | 6-col, resizable | Hover border | Cursor change, handle visible |
| Drag right | Same | Drag col 2 right 100px | Col 2 wider, body matches header |
| Drag left | Same | Drag col 2 left 100px | Col 2 narrower, above minWidth |
| Min-width stop | minWidth 80px | Drag past 80px | Stops at 80px |
| Resize last column | Same | Drag last border | Correct resize |
| Resize first column | Same | Drag first border | Correct resize |
| Resize + horizontal scroll | Wide table | Drag while scrolled | Works, header/body synced |
| Resize + virtual scroll | 500 rows, resizable | Resize, scroll | All rows use new width |
| Resize + spanning | colSpan 3, resizable | Resize spanned column | Span adjusts naturally |
| Double-click auto-fit | Varying content widths | Double-click border | Width matches widest visible |
| No handles when disabled | `resizable: false` | Hover borders | No cursor change |

### Layer 4 — Cross-feature interactions

| Test | Setup | Action | Assert |
|------|-------|--------|--------|
| Auto-height + resize → reflow | Wrapping text, auto, resizable | Narrow column | Heights increase, model re-measures |
| Auto-height + resize → widen | Same, after narrowing | Widen back | Heights decrease, model re-measures |
| Callback heights + resize | Callback, resizable | Resize | Heights unchanged (data-determined) |
| Variable heights + selection + scroll | Auto, multi, 500 rows | Select 5-10, scroll away, back | Selection preserved |
| Variable heights + sort | Auto, 200 measured rows | Sort | Heights remap by key |
| Variable heights + filter | Auto, client filter | Filter text | Model adjusts to filtered set |
| Resize + sort | Resizable, resized col 3 | Sort by col 2 | Override preserved (ID-based) |
| Resize + visibility | Resizable, resized col 3 | Hide col 3, show col 3 | Override restored |
| Resize + selection multi | Resizable, multi | Resize next to checkbox | Checkbox unaffected |
| Variable heights + tree | Auto, getChildren | Expand node | New rows measured |
| Variable heights + detail | Auto, getRowDetail, paginated | Expand detail | Both tracks auto-height |
| Variable heights + load-more | Auto, scroll, hasMore | Scroll near bottom | load-more at correct offset |
| Load-more preserves heights | Auto, scroll, hasMore, 200 measured rows | Trigger load-more (50 new rows) | Previously-measured rows retain heights, no scrollbar jump |
| Auto-height + sort without getRowKey | Auto, no getRowKey | Sort triggered | All rows re-measured from estimate (reset fallback) |
| Span + client sort + mergeRows | mergeRows on col A, unsorted rows non-adjacent | Sort by col A | Spans reflect sorted adjacency — adjacent equal values merge |
| Keyboard shift-select + client sort | Multi-select, client sort | Sort, then Shift+ArrowDown | Correct row selected (sorted order, not data order) |
| Span selection + client sort + mergeRows | mergeRows, client sort, multi-select | Sort, select spanned row | Selection highlight covers correct rows in sorted order |
| All three combined | Auto, resizable, mergeRows | Resize, scroll | Self-correcting pipeline |

### Layer 5 — Edge cases and boundaries

| Test | What it verifies |
|------|-----------------|
| 0 rows + variable height | No HeightModel errors |
| 1 row + variable height | Correct layout |
| 1 column + resizable | Handle present, drag works |
| Exact threshold (50→51) + auto | Non-virtual → virtual + measurement transition |
| Container height = 0 then revealed | ResizeObserver triggers measurement |
| Container shorter than one row | At least one row visible |
| All rows same height via callback | Degenerates to FixedHeightModel behavior |
| rowHeight callback changes | Model rebuilds |
| rowHeight switches number → auto | Model type changes, measurements begin |
| rowHeight switches auto → number | Measurements discarded, fixed model |
| Very tall row (500px) among short (32px) | Fully visible, no clipping |
| Rapid programmatic scrollTop jump | Measurement catches up in 1-2 frames |
| Data change during measurement cycle | Stale measurements discarded |
| Resize during keyboard navigation | Focus preserved |
| Browser zoom (125%, 150%) | Heights and handles work correctly |

### Layer 6 — Performance

| Test | Threshold |
|------|-----------|
| CallbackHeightModel build, 10K rows | < 5ms |
| CallbackHeightModel build, 100K rows | < 50ms |
| MeasuredHeightModel.recordHeight, 1000 calls | < 1ms |
| Grid template string, 10K rows | < 2ms |
| Grid template string, 100K rows | < 20ms |
| Column resize drag, 10K virtual rows | 60fps |
| Auto-height measurement cycle | < 16ms (one frame) |
| MeasuredHeightModel memory, 100K rows | < 10MB |

### Layer 7 — Accessibility

| Test | What it verifies |
|------|-----------------|
| aria-rowcount with variable heights | Equals total row count |
| aria-rowindex with variable heights | Correct 1-based index |
| Keyboard ArrowDown with tall rows | Scrolls to reveal full row |
| Keyboard ArrowUp with variable heights | Correct position |
| Resize handle not keyboard-focusable | Tab skips handles (v1 — pointer-only) |
| Resizable column announcement | aria-description on header cell |
| Focus preservation after measurement | Focus survives re-render |
| Focus preservation after resize | Focus survives re-render |

## API Summary

### Changed properties

| Property | Before | After | Default |
|----------|--------|-------|---------|
| `rowHeight` | `number` | `number \| 'auto' \| ((row: TypedRow, index: number) => number)` | `48` |

### New properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `resizable` | `boolean` | `false` | Enable column resize handles |

### New events

| Event | Detail | When |
|-------|--------|------|
| `column-resize` | `{ columnId: string, width: number }` | Pointer released after column drag |

### New types

| Type | File | Description |
|------|------|-------------|
| `HeightModel` | `virtual-scroll-engine.ts` | Height strategy interface |
| `FixedHeightModel` | `virtual-scroll-engine.ts` | Uniform fixed heights — current behavior |
| `CallbackHeightModel` | `virtual-scroll-engine.ts` | Data-determined heights with prefix sums |
| `MeasuredHeightModel` | `virtual-scroll-engine.ts` | Auto-height with measurement lifecycle |
| `ColumnResizeDetail` | `types.ts` | Column resize event payload |

## Scope Exclusions

Each exclusion will be filed as a GitHub issue on casehubio/blocks-ui, linking
back to this spec for context.

- **Keyboard-accessible resize (v1)** — resize is pointer-only. Keyboard resize
  (e.g., Shift+Arrow on focused header) is a future enhancement.
- **Resize persistence** — the component does not persist widths. The consumer
  stores widths from `column-resize` events and provides them via column config.
- **Horizontal auto-scroll during resize** — when the table has horizontal
  overflow, dragging near the edge does not auto-scroll. Future enhancement.
- **Auto-fit all columns** — double-click auto-fits one column. A "fit all" action
  is a future enhancement.
- **Content-measured heights for off-screen rows** — MeasuredHeightModel estimates
  unrendered rows from the average of measured rows. It does not render off-screen
  rows temporarily to measure them.
