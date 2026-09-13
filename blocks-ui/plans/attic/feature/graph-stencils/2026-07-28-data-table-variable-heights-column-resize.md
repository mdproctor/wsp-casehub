# Variable Row Heights + Column Resizing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #84 — Variable row heights with cell spanning
**Issue group:** #84, #85

**Goal:** Add variable row heights (fixed callback, auto-measured) and column
resizing to pages-data-table, preserving backward compatibility.

**Architecture:** HeightModel abstraction (3 implementations) replaces scalar
rowHeight in the scroll engine. _effectiveRows separates data ordering from
viewport windowing. Column resize uses pointer events on header handles with
_columnWidths override map.

**Tech Stack:** TypeScript, Lit 3, CSS Grid, Vitest

**Implementation repo:** casehub-pages (`/Users/mdproctor/claude/casehub/pages`)
**Package:** `packages/pages-table/src/`
**IntelliJ project_path:** `/Users/mdproctor/claude/casehub/pages`

## Global Constraints

- All existing tests must continue to pass (backward compat via FixedHeightModel)
- `rowHeight` default stays `48` (number)
- `resizable` default stays `false`
- No new npm dependencies
- Tests use Vitest with existing patterns (describe/it/expect, fromRows)

---

### Task 1: HeightModel abstraction + computeScrollWindow refactor

**Files:**
- Modify: `packages/pages-table/src/virtual-scroll-engine.ts`
- Modify: `packages/pages-table/src/virtual-scroll-engine.test.ts`

**Interfaces:**
- Produces: `HeightModel` interface, `FixedHeightModel`, `CallbackHeightModel`,
  `MeasuredHeightModel` classes, refactored `computeScrollWindow(scrollTop,
  containerHeight, heightModel, bufferSize)`

**Steps:**

- [ ] **1.1: Write HeightModel interface + FixedHeightModel tests**

  Add tests for the interface contract via FixedHeightModel. Tests:
  - totalHeight = count * height
  - rowHeight(i) returns fixed height for all i
  - offsetAtIndex(i) = i * height
  - indexAtOffset(y) = floor(y / height)
  - 0 rows → totalHeight 0
  - 1 row → correct values

- [ ] **1.2: Implement HeightModel + FixedHeightModel**

  Export `HeightModel` interface and `FixedHeightModel` class.

- [ ] **1.3: Write CallbackHeightModel tests**

  - Varying heights: prefix sums correct
  - Uniform callback: degenerates to FixedHeightModel
  - Binary search accuracy: exact boundary, mid-row, between rows
  - Rebuild on new rows
  - Clamp ≤0 returns to 1px

- [ ] **1.4: Implement CallbackHeightModel**

  Prefix sum array on construction, binary search for indexAtOffset.

- [ ] **1.5: Write MeasuredHeightModel tests**

  - Initial state: all heights = 48px estimate
  - recordHeight: measured rows return recorded, unmeasured return avg
  - Estimate convergence: measuring rows ~72px → estimate → ~72px
  - Mixed prefix sums: measured + estimated
  - Invalidation: recordHeight invalidates cached sums
  - reset(): clears, re-seeds 48px
  - Re-measurement: second overwrites first
  - remap(keyToNewIndex): preserves at new indices
  - remap without keys: falls back to reset()
  - Load-more (extend): preserves existing, new rows at estimate

- [ ] **1.6: Implement MeasuredHeightModel**

- [ ] **1.7: Refactor computeScrollWindow to use HeightModel**

  Change signature from `(scrollTop, containerHeight, rowHeight, rowCount,
  bufferSize)` to `(scrollTop, containerHeight, heightModel, bufferSize)`.
  Use `heightModel.indexAtOffset(scrollTop)` for start, `heightModel.offsetAtIndex`
  for offset. Update `extendWindowForSpans` — no signature change needed,
  it operates on indices.

- [ ] **1.8: Update existing computeScrollWindow tests**

  Adapt all existing tests to pass a FixedHeightModel instead of scalar
  rowHeight. Verify identical results — no behavioral change.

- [ ] **1.9: Add computeScrollWindow tests with variable heights**

  - CallbackHeightModel with alternating 48/72 heights
  - MeasuredHeightModel with mixed measured/estimated
  - Buffer with tall rows covers correct pixel range

- [ ] **1.10: Run all tests, verify pass**

  `cd /Users/mdproctor/claude/casehub/pages/packages/pages-table && npx vitest run src/virtual-scroll-engine.test.ts`

- [ ] **1.11: Commit**

---

### Task 2: _effectiveRows + rowHeight property + HeightModel lifecycle

**Files:**
- Modify: `packages/pages-table/src/pages-data-table.ts`
- Modify: `packages/pages-table/src/types.ts` (export ColumnResizeDetail)

**Interfaces:**
- Consumes: `HeightModel`, `FixedHeightModel`, `CallbackHeightModel`,
  `MeasuredHeightModel` from virtual-scroll-engine.ts
- Produces: `_effectiveRows` getter, polymorphic `rowHeight` property,
  `_heightModel` instance state, HeightModel lifecycle in willUpdate

**Steps:**

- [ ] **2.1: Add _effectiveRows computation**

  Extract filter/sort/tree logic from `_visibleRows` into `_effectiveRows`
  (computed in willUpdate, stored as instance state). `_visibleRows` becomes
  a window into `_effectiveRows` (paginate/scroll/all).

- [ ] **2.2: Migrate _dataRows → _effectiveRows call sites**

  Per the spec's migration inventory:
  - Keyboard nav bounds: `_dataRows.length` → `_effectiveRows.length`
  - Keyboard shift-select: `_dataRows[rovingIndex]` → `_effectiveRows[rovingIndex]`
  - Span covered-row selection: `_dataRows[actualIndex + j]` → `_effectiveRows[actualIndex + j]`
  - SpanMap computation: `computeSpanMap(this._dataRows, ...)` → `computeSpanMap(this._effectiveRows, ...)`
  - Virtual scroll threshold: `_dataRows.length > AUTO_THRESHOLD` → `_effectiveRows.length > AUTO_THRESHOLD`

- [ ] **2.3: Add rowHeight polymorphic property with custom Lit converter**

  Replace `@property({ type: Number })` with custom converter per spec
  §Property decorator. Handle number, 'auto', function.

- [ ] **2.4: Add _heightModel + lifecycle in willUpdate**

  - Construct FixedHeightModel when rowHeight is number
  - Construct CallbackHeightModel when rowHeight is function
  - Construct MeasuredHeightModel when rowHeight is 'auto'
  - Update/reconstruct on dataSet change, sort/filter, type transition
  - Load-more append detection via _loadingMore flag

- [ ] **2.5: Update computeScrollWindow call sites**

  Replace `computeScrollWindow(scrollTop, containerH, this.rowHeight, rows.length, bufferSize)`
  with `computeScrollWindow(scrollTop, containerH, this._heightModel, this.bufferSize)`.
  Update `_scrollWindow`, `_visibleRows`, `_onScroll` (load-more detection),
  `_scrollToRowIfNeeded`.

- [ ] **2.6: Run all existing tests, verify pass**

  `cd /Users/mdproctor/claude/casehub/pages/packages/pages-table && npx vitest run`

  All existing tests must pass — FixedHeightModel with default rowHeight=48
  produces identical behavior.

- [ ] **2.7: Commit**

---

### Task 3: Variable height rendering + measurement lifecycle

**Files:**
- Modify: `packages/pages-table/src/pages-data-table.ts`

**Interfaces:**
- Consumes: `_heightModel`, `_effectiveRows` from Task 2
- Produces: `_gridTemplateRows` getter, measurement cycle in `updated()`,
  scroll correction with `_isCorrectingScroll` guard

**Steps:**

- [ ] **3.1: Add _gridTemplateRows computed property**

  - FixedHeightModel: `repeat(${count}, ${height}px)` (current behavior)
  - CallbackHeightModel: explicit per-row `${h0}px ${h1}px ...`
  - MeasuredHeightModel (virtual scroll): split template — explicit px
    for unrendered rows, `auto` for rendered viewport rows
  - Non-virtual modes: literal `auto` tracks (no HeightModel consultation)

- [ ] **3.2: Update render() to use _gridTemplateRows**

  Replace inline `grid-template-rows: repeat(...)` with `this._gridTemplateRows`
  in the virtual scroll rendering path. Non-virtual paths use `auto` tracks
  when rowHeight is 'auto' or function.

- [ ] **3.3: Add measurement cycle in updated()**

  When _heightModel is MeasuredHeightModel and virtual scroll active:
  - Read `getComputedStyle(bodyContent).gridTemplateRows`
  - Parse resolved pixel values for rendered viewport range
  - Call `recordHeight(index, height, key)` for each
  - If any changed, recompute _gridTemplateRows, requestUpdate()

- [ ] **3.4: Add scroll position correction**

  - Compute delta: sum of (measured - previous) for rows above viewport
  - Set `_isCorrectingScroll = true`
  - Adjust scrollTop
  - Clear flag after scrollTop assignment
  - `_onScroll` checks flag and skips state update when set

- [ ] **3.5: Run full test suite**

  `cd /Users/mdproctor/claude/casehub/pages/packages/pages-table && npx vitest run`

- [ ] **3.6: Commit**

---

### Task 4: Column resizing

**Files:**
- Modify: `packages/pages-table/src/types.ts`
- Modify: `packages/pages-table/src/pages-data-table.ts`

**Interfaces:**
- Produces: `ColumnResizeDetail` type, `resizable` property,
  `_columnWidths` map, resize handles in header, `column-resize` event,
  header-body horizontal scroll sync

**Steps:**

- [ ] **4.1: Add ColumnResizeDetail to types.ts**

  ```typescript
  export interface ColumnResizeDetail {
    readonly columnId: string;
    readonly width: number;
  }
  ```

- [ ] **4.2: Add resizable property + _columnWidths state**

  - `@property({ type: Boolean }) resizable = false;`
  - `private _columnWidths: Map<string, string> = new Map();`
  - `private _resizing: { columnId: string; startX: number; startWidth: number } | null = null;`

- [ ] **4.3: Update _gridTemplateColumns for resize overrides**

  Modify getter: check `_columnWidths.get(id)` before falling back to
  config width.

- [ ] **4.4: Add resize handles to header rendering**

  In `_renderHeaderCell`, when `this.resizable`: append a resize handle
  div with `cursor: col-resize`, positioned at right edge.

- [ ] **4.5: Add pointer event handlers**

  - `_handleResizeStart(e, columnId)`: capture pointer, record startX
    and starting width (resolve from computed style), set _resizing
  - `_handleResizeMove(e)`: compute delta, clamp to minWidth (parse
    from config, default 50px), update _columnWidths
  - `_handleResizeEnd(e)`: release capture, emit column-resize event,
    clear _resizing
  - `_handleResizeCancel()`: revert, no event

- [ ] **4.6: Add resize CSS to static styles**

  Resize handle styles: position, width, cursor, hover highlight.

- [ ] **4.7: Add header-body horizontal scroll sync**

  In `_onScroll`: read `scrollLeft`, apply `translateX(-${scrollLeft}px)`
  to `.header`. Add `overflow: hidden` to `.header-container`.

- [ ] **4.8: Add double-click auto-fit**

  On dblclick on resize handle: measure widest rendered content in column
  via `scrollWidth`, set to `max(contentWidth + padding, minWidth)`,
  emit column-resize.

- [ ] **4.9: Add pipeline integration**

  In `set props()`: consume `p.resizable`, `p.rowHeight`.

- [ ] **4.10: Add resize override lifecycle**

  Clear _columnWidths when columnConfig changes. Preserve across
  dataSet/sort/filter changes.

- [ ] **4.11: Run full test suite**

- [ ] **4.12: Commit**

---

### Task 5: Tests + integration verification

**Files:**
- Modify: `packages/pages-table/src/virtual-scroll-engine.test.ts`
- Modify: `packages/pages-table/src/pages-table.test.ts`
- Modify: `packages/pages-table/src/span-map.test.ts`

**Steps:**

- [ ] **5.1: Add HeightModel edge case tests**

  - 0 rows all models
  - 1 row all models
  - Very tall row (500px) among short (32px)
  - CallbackHeightModel uniform → same as Fixed
  - MeasuredHeightModel all measured → no estimates

- [ ] **5.2: Add _effectiveRows regression tests**

  - Client sort doesn't break keyboard navigation bounds
  - Client filter reduces effective row count
  - Span selection uses correct rows after sort

- [ ] **5.3: Add column resize unit tests**

  - Template with override
  - Min-width clamping
  - Override persistence/reset lifecycle
  - Event detail correctness

- [ ] **5.4: Add cross-feature interaction tests**

  - Variable heights + sort (remap)
  - Variable heights + filter (model adjusts)
  - Load-more preserves measurements
  - Auto-height + sort without getRowKey (reset fallback)

- [ ] **5.5: Run full test suite, verify all pass**

  `cd /Users/mdproctor/claude/casehub/pages/packages/pages-table && npx vitest run`

- [ ] **5.6: Sync to blocks-ui .casehub-packages**

  Copy changed files to blocks-ui's .casehub-packages for local testing.

- [ ] **5.7: Commit final**
