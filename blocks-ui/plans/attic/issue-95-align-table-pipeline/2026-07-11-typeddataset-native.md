# TypedDataSet Native Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #49 — refactor: make Lit components consume TypedDataSet natively
**Issue group:** #49

**Goal:** Unify all data sources (fetch, push, SSE, inline, simulated) through a single extraction pipeline that produces TypedDataSet, then make pages-table and all blocks-ui consumers speak TypedDataSet natively.

**Architecture:** Fix the type contract at `DataReceiver.dataSet` from `unknown` to `TypedDataSet | undefined`. Extend `SourceFactory` to carry column declarations. Rewrite `fetchSource` to use the pages extraction pipeline instead of raw `as never` casts. Redesign `pages-data-table` → `pages-table` to consume TypedDataSet directly with `columnRenderers` and `TableColumnConfig`. Propagate types through blocks-ui and migrate all consumers.

**Tech Stack:** TypeScript 5.6+, Lit 3, vitest, pages-data extraction pipeline

## Global Constraints

- Pre-release platform: breaking changes cost nothing. Fix the design, don't protect callers.
- All source-file edits via IntelliJ MCP (`ide_edit_member`, `ide_replace_member`, `ide_insert_member`, `ide_refactor_rename`, `ide_move_file`). Never bash Edit/Write on existing `.ts` files.
- pages is a peer repo. Changes to pages source files are made locally (vite aliases resolve them), but commits to pages happen in a separate session.
- TDD: write failing test → verify fail → implement → verify pass → commit.

## Cross-Repo Note

This plan makes changes to both repos. blocks-ui's vite/vitest configs alias `@casehubio/pages-*` to pages source paths, so local changes are immediately visible for builds and tests. The plan is ordered so pages foundation changes come first (Tasks 1-4), then blocks-ui changes build on them (Tasks 5-8).

Pages changes will be committed in a separate pages session after this plan completes. blocks-ui changes commit on the `issue-49-typeddataset-native` branch.

---

### Task 1: ExtractionDef type and extractDataSet signature (pages-data)

**Files:**
- Modify: `pages/packages/pages-data/src/dataset/external/types.ts`
- Modify: `pages/packages/pages-data/src/dataset/external/extraction.ts`
- Test: `pages/packages/pages-data/src/dataset/external/extraction.test.ts`

**Interfaces:**
- Produces: `ExtractionDef` interface (used by Tasks 3, 5); `extractDataSet(result: FetchResult, def: ExtractionDef, presetRegistry: PresetRegistry)` signature

- [ ] **Step 1: Define ExtractionDef interface**

Add to `types.ts`:

```typescript
export interface ExtractionDef {
  readonly url?: string;
  readonly content?: string;
  readonly dataPath?: string;
  readonly type?: string;
  readonly expression?: string;
  readonly columns?: readonly ExternalColumnDef[];
  readonly accumulate?: boolean;
}
```

Make `ExternalDataSetDef` extend it:

```typescript
export interface ExternalDataSetDef extends ExtractionDef {
  readonly uuid: DataSetId;
  readonly name?: string;
  // ... existing request fields (method, headers, query, form, body)
  // ... existing config fields (cacheEnabled, cacheMaxRows, refreshTime, keyColumn, serverQuery, join)
}
```

Use `ide_edit_member` on `ExternalDataSetDef` to change its declaration, extracting the shared fields into `ExtractionDef`. Use `ide_insert_member` to add `ExtractionDef` before `ExternalDataSetDef`.

- [ ] **Step 2: Change extractDataSet parameter type**

In `extraction.ts`, change `extractDataSet` signature from `ExternalDataSetDef` to `ExtractionDef`:

```typescript
export async function extractDataSet(
  result: FetchResult,
  def: ExtractionDef,
  presetRegistry: PresetRegistry,
): Promise<ExtractionResult> {
```

Use `ide_edit_member` on `extractDataSet`. Update the import to bring in `ExtractionDef` instead of (or in addition to) `ExternalDataSetDef`.

- [ ] **Step 3: Verify existing tests still pass**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-data test -- --run`

Expected: All existing extraction tests pass. `ExternalDataSetDef extends ExtractionDef` is a structural subtype, so all existing callers passing `ExternalDataSetDef` still compile.

- [ ] **Step 4: Verify resolver still compiles**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-data typecheck`

Check `ide_diagnostics` on `resolver.ts` — it passes `ExternalDataSetDef` to `extractDataSet`, which is now `ExtractionDef`. Since `ExternalDataSetDef extends ExtractionDef`, this compiles.

- [ ] **Step 5: Write test for ExtractionDef usage**

Add a test in `extraction.test.ts` that calls `extractDataSet` with a plain `ExtractionDef` (no `uuid`):

```typescript
it("accepts ExtractionDef without uuid", async () => {
  const result: FetchResult = {
    data: [{ name: "Alice", score: 42 }],
    contentType: "application/json",
  };
  const def: ExtractionDef = {
    columns: [
      { id: columnId("name"), type: ColumnType.TEXT },
      { id: columnId("score"), type: ColumnType.NUMBER },
    ],
  };
  const { dataset } = await extractDataSet(result, def, emptyRegistry);
  expect(dataset.columns).toHaveLength(2);
  expect(dataset.rows).toHaveLength(1);
  expect(dataset.rows[0]!.text(columnId("name"))).toBe("Alice");
  expect(dataset.rows[0]!.number(columnId("score"))).toBe(42);
});
```

Where `emptyRegistry` is `{ get: () => undefined, has: () => false }`.

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-data test -- --run -t "accepts ExtractionDef"`

Expected: PASS

---

### Task 2: fromRows() factory and SnapshotEvent.totalRows (pages-data)

**Files:**
- Modify: `pages/packages/pages-data/src/dataset/conversion.ts`
- Modify: `pages/packages/pages-data/src/dataset/events.ts`
- Test: `pages/packages/pages-data/src/dataset/conversion.test.ts`

**Interfaces:**
- Consumes: `createTypedRow()`, `CellValue`, `Column`, `ColumnType`, `ColumnId` from types.ts/conversion.ts
- Produces: `fromRows<R>(rows, columns)` → `TypedDataSet` (used by Tasks 7, 8); `SnapshotEvent.totalRows?: number` (used by Tasks 3, 4, 5)

- [ ] **Step 1: Write failing test for fromRows()**

Add to `conversion.test.ts`:

```typescript
describe("fromRows", () => {
  it("converts domain objects to TypedDataSet with typed accessors", () => {
    interface Capability { tag: string; score: number; }
    const rows: Capability[] = [
      { tag: "authentication", score: 0.95 },
      { tag: "authorization", score: 0.72 },
    ];
    const dataset = fromRows(rows, [
      { id: columnId("tag"), name: "Capability", type: ColumnType.TEXT, getValue: (c: Capability) => c.tag },
      { id: columnId("score"), name: "Score", type: ColumnType.NUMBER, getValue: (c: Capability) => c.score },
    ]);
    expect(dataset.columns).toHaveLength(2);
    expect(dataset.columns[0]!.id).toBe(columnId("tag"));
    expect(dataset.columns[0]!.type).toBe(ColumnType.TEXT);
    expect(dataset.rows).toHaveLength(2);
    expect(dataset.rows[0]!.text(columnId("tag"))).toBe("authentication");
    expect(dataset.rows[0]!.number(columnId("score"))).toBe(0.95);
    expect(dataset.rows[1]!.text(columnId("tag"))).toBe("authorization");
  });

  it("handles null/undefined values as NULL cells", () => {
    const rows = [{ name: null as string | null }];
    const dataset = fromRows(rows, [
      { id: columnId("name"), type: ColumnType.TEXT, getValue: (r: { name: string | null }) => r.name },
    ]);
    const cell = dataset.rows[0]!.cell(columnId("name"));
    expect(cell.type).toBe("NULL");
  });

  it("handles Date values", () => {
    const now = new Date("2026-07-11T10:00:00Z");
    const rows = [{ created: now }];
    const dataset = fromRows(rows, [
      { id: columnId("created"), type: ColumnType.DATE, getValue: (r: { created: Date }) => r.created },
    ]);
    expect(dataset.rows[0]!.date(columnId("created")).getTime()).toBe(now.getTime());
  });

  it("produces empty dataset for empty input", () => {
    const dataset = fromRows([], [
      { id: columnId("x"), type: ColumnType.TEXT, getValue: () => "" },
    ]);
    expect(dataset.columns).toHaveLength(1);
    expect(dataset.rows).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-data test -- --run -t "fromRows"`

Expected: FAIL — `fromRows` is not defined.

- [ ] **Step 3: Implement fromRows()**

Add to `conversion.ts` using `ide_insert_member`:

```typescript
export function fromRows<R>(
  rows: readonly R[],
  columns: readonly {
    readonly id: ColumnId;
    readonly name?: string;
    readonly type: ColumnType;
    readonly getValue: (row: R) => unknown;
  }[],
): TypedDataSet {
  const cols: Column[] = columns.map(c => ({
    id: c.id,
    name: c.name ?? String(c.id),
    type: c.type,
  }));

  const typedRows: TypedRow[] = rows.map((row, rowIdx) => {
    const cells: CellValue[] = columns.map((col) => {
      const raw = col.getValue(row);
      if (raw === null || raw === undefined) {
        return { type: "NULL" as const };
      }
      switch (col.type) {
        case ColumnType.NUMBER:
          return { type: ColumnType.NUMBER, value: typeof raw === "number" ? raw : parseFloat(String(raw)) };
        case ColumnType.DATE:
          return { type: ColumnType.DATE, value: raw instanceof Date ? raw : new Date(String(raw)) };
        case ColumnType.LABEL:
          return { type: ColumnType.LABEL, value: String(raw) };
        case ColumnType.TEXT:
        default:
          return { type: ColumnType.TEXT, value: String(raw) };
      }
    });
    return createTypedRow(cells, cols);
  });

  return { columns: cols, rows: typedRows };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-data test -- --run -t "fromRows"`

Expected: PASS

- [ ] **Step 5: Add totalRows to SnapshotEvent**

In `events.ts`, use `ide_edit_member` on `SnapshotEvent`:

```typescript
export interface SnapshotEvent {
  readonly type: "snapshot";
  readonly dataset: TypedDataSet;
  readonly totalRows?: number;
}
```

- [ ] **Step 6: Verify all pages-data tests pass**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-data test -- --run`

Expected: All pass. `totalRows` is optional — existing snapshot emitters don't set it.

---

### Task 3: DataReceiver and SourceFactory contract changes (pages-component)

**Files:**
- Modify: `pages/packages/pages-component/src/model/hosting.ts`
- Modify: `pages/packages/pages-component/src/controller/data-source-controller.ts`
- Test: `pages/packages/pages-component/src/controller/data-source-controller.test.ts`

**Interfaces:**
- Consumes: `TypedDataSet` from pages-data; `SnapshotEvent.totalRows` from Task 2
- Produces: `DataReceiver.dataSet: TypedDataSet | undefined`; `SourceFactory` with `SourceFactoryOptions`; `DataSourceControllerOptions` with `columns`, `dataPath`, `totalPath` (used by Tasks 5, 6)

- [ ] **Step 1: Change DataReceiver.dataSet type**

In `hosting.ts`, use `ide_edit_member` on `DataReceiver`:

```typescript
export interface DataReceiver {
  loading: boolean;
  dataSet: TypedDataSet | undefined;
  error: string;
}
```

Add import: `import type { TypedDataSet } from "@casehubio/pages-data/dist/dataset/types.js";`

- [ ] **Step 2: Update DataSourceController field and accessors**

In `data-source-controller.ts`:

Change `_dataSet` field from `unknown` to `TypedDataSet | undefined`:
```typescript
private _dataSet: TypedDataSet | undefined = undefined;
```

Change getter/setter:
```typescript
get dataSet(): TypedDataSet | undefined { return this._dataSet; }
set dataSet(v: TypedDataSet | undefined) {
  this._loading = false;
  this._error = "";
  this._dataSet = v;
  this.onChange?.();
}
```

Use `ide_edit_member` for each.

- [ ] **Step 3: Add SourceFactoryOptions and extend SourceFactory**

In `data-source-controller.ts`, add `SourceFactoryOptions` interface and update `SourceFactory` type:

```typescript
export interface SourceFactoryOptions {
  readonly columns?: readonly ExternalColumnDef[];
  readonly dataPath?: string;
  readonly totalPath?: string;
}

export type SourceFactory = (url: string, id: DataSetId, options?: SourceFactoryOptions) => DataSource;
```

Add `ExternalColumnDef` import from pages-data.

- [ ] **Step 4: Extend DataSourceControllerOptions**

Use `ide_edit_member` on `DataSourceControllerOptions`:

```typescript
export interface DataSourceControllerOptions {
  onChange?: () => void;
  onRefresh?: () => void;
  dataSetId?: DataSetId;
  sourceFactory?: SourceFactory;
  columns?: readonly ExternalColumnDef[];
  dataPath?: string;
  totalPath?: string;
}
```

- [ ] **Step 5: Update createSourceFromUrl to pass options**

Use `ide_replace_member` on `createSourceFromUrl`:

```typescript
private createSourceFromUrl(url: string): DataSource {
  if (this._sourceFactory) {
    return this._sourceFactory(url, this._dataSetId, {
      columns: this._columns,
      dataPath: this._dataPath,
      totalPath: this._totalPath,
    });
  }
  return {
    connect() {},
    disconnect() {},
  };
}
```

Store the options in the constructor:
```typescript
private readonly _columns: readonly ExternalColumnDef[] | undefined;
private readonly _dataPath: string | undefined;
private readonly _totalPath: string | undefined;
```

And in the constructor body:
```typescript
this._columns = options?.columns;
this._dataPath = options?.dataPath;
this._totalPath = options?.totalPath;
```

- [ ] **Step 6: Propagate totalRows from SnapshotEvent**

In `handleEvent`, update the snapshot case:

```typescript
case "snapshot":
  this.dataSet = event.dataset;
  if (event.totalRows !== undefined) this.totalRows = event.totalRows;
  break;
```

- [ ] **Step 7: Write test for totalRows propagation**

Add to `data-source-controller.test.ts`:

```typescript
it("propagates totalRows from snapshot event", () => {
  const ctrl = new DataSourceController({ onChange: () => {} });
  const dataset = toTypedDataSet({ columns: [{ id: columnId("a"), name: "a", type: ColumnType.TEXT }], data: [["x"]] });
  // Simulate a snapshot with totalRows
  ctrl.dataSet = dataset;
  expect(ctrl.totalRows).toBe(-1); // default
  // Use source that emits snapshot with totalRows
  const source: DataSource = {
    connect(sink) {
      sink.apply({ type: "snapshot", dataset, totalRows: 100 });
    },
    disconnect() {},
  };
  ctrl.source = source;
  ctrl.connect();
  expect(ctrl.dataSet).toBe(dataset);
  expect(ctrl.totalRows).toBe(100);
});
```

- [ ] **Step 8: Run all pages-component tests**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-component test -- --run`

Expected: PASS

- [ ] **Step 9: Typecheck both packages**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-component typecheck && yarn workspace @casehubio/pages-data typecheck`

Fix any compile errors from the `DataReceiver.dataSet` type change cascading through pages-viz or pages-runtime. These should be type-only fixes — the runtime already handles TypedDataSet.

---

### Task 4: Table redesign — pages-data-table → pages-table (pages)

**Files:**
- Modify: `pages/packages/pages-data-table/src/types.ts`
- Modify: `pages/packages/pages-data-table/src/sort.ts`
- Modify: `pages/packages/pages-data-table/src/tree.ts`
- Modify: `pages/packages/pages-data-table/src/pages-data-table.ts`
- Modify: `pages/packages/pages-data-table/src/index.ts`
- Rewrite: `pages/packages/pages-data-table/src/pages-data-table.test.ts`
- Rename: package `@casehubio/pages-data-table` → `@casehubio/pages-table`, element `pages-data-table` → `pages-table`, class `PagesDataTable` → `PagesTable`

**Interfaces:**
- Consumes: `TypedDataSet`, `TypedRow`, `CellValue`, `Column`, `ColumnId`, `ColumnType` from pages-data
- Produces: `<pages-table>` element with `dataSet: TypedDataSet`, `columnRenderers`, `columnConfig: TableColumnConfig[]` (used by Tasks 7, 8)

This is the largest task. It replaces the `ColumnDef<R>` + `rows: unknown[]` API with `dataSet: TypedDataSet` + `columnRenderers` + `columnConfig`.

- [ ] **Step 1: Replace ColumnDef with TableColumnConfig in types.ts**

Use `ide_edit_member` to replace `ColumnDef` with `TableColumnConfig`:

```typescript
export interface TableColumnConfig {
  readonly id: ColumnId;
  readonly label?: string;
  readonly sortable?: boolean;
  readonly visible?: boolean;
  readonly width?: string;
  readonly minWidth?: string;
  readonly align?: ColumnAlign;
  readonly filterable?: boolean;
  readonly compare?: (a: CellValue, b: CellValue) => number;
}
```

Add imports for `ColumnId`, `CellValue` from pages-data. Keep all existing event detail types (`SortChangeDetail`, `PageChangeDetail`, `SelectionChangeDetail`, `RowActivateDetail`, etc.) but update their generics from `<R = unknown>` to use `TypedRow`:

```typescript
export interface SelectionChangeDetail {
  readonly selectedKeys: readonly string[];
  readonly selectedRows: readonly TypedRow[];
  readonly scope?: 'page';
}

export interface RowActivateDetail {
  readonly row: TypedRow;
  readonly key?: string;
}
```

- [ ] **Step 2: Update sort.ts for TypedDataSet**

Replace sort module to work with `TypedRow` and `CellValue`:

```typescript
import type { CellValue, ColumnId, TypedRow } from '@casehubio/pages-data/dist/dataset/types.js';
import { ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { TableColumnConfig, SortDirection, SortEntry } from './types.js';

type Comparator = (a: TypedRow, b: TypedRow) => number;

function resolveByType(type: string): (a: CellValue, b: CellValue) => number {
  switch (type) {
    case ColumnType.NUMBER:
      return (a, b) => {
        if (a.type === "NULL" && b.type === "NULL") return 0;
        if (a.type === "NULL") return 1;
        if (b.type === "NULL") return -1;
        return (a as { value: number }).value - (b as { value: number }).value;
      };
    case ColumnType.DATE:
      return (a, b) => {
        if (a.type === "NULL" && b.type === "NULL") return 0;
        if (a.type === "NULL") return 1;
        if (b.type === "NULL") return -1;
        return (a as { value: Date }).value.getTime() - (b as { value: Date }).value.getTime();
      };
    default:
      return (a, b) => {
        if (a.type === "NULL" && b.type === "NULL") return 0;
        if (a.type === "NULL") return 1;
        if (b.type === "NULL") return -1;
        return String((a as { value: unknown }).value).localeCompare(String((b as { value: unknown }).value));
      };
  }
}

export function createComparator(
  columnId: ColumnId,
  columnType: string,
  direction: SortDirection,
  config?: TableColumnConfig,
): Comparator {
  if (direction === 'none') return () => 0;
  const base = config?.compare ?? resolveByType(columnType);
  const flip = direction === 'desc' ? -1 : 1;
  return (a: TypedRow, b: TypedRow): number => {
    return flip * base(a.cell(columnId), b.cell(columnId));
  };
}

export function createMultiComparator(
  sortStack: readonly SortEntry[],
  columns: readonly import('@casehubio/pages-data/dist/dataset/types.js').Column[],
  configs: readonly TableColumnConfig[],
): Comparator {
  const configMap = new Map(configs.map(c => [c.id, c]));
  const comparators = sortStack
    .filter(entry => entry.direction !== 'none')
    .map(entry => {
      const col = columns.find(c => c.id === entry.columnId);
      if (!col) return null;
      return createComparator(entry.columnId, col.type, entry.direction, configMap.get(entry.columnId));
    })
    .filter((c): c is Comparator => c !== null);

  return (a: TypedRow, b: TypedRow): number => {
    for (const cmp of comparators) {
      const result = cmp(a, b);
      if (result !== 0) return result;
    }
    return 0;
  };
}
```

- [ ] **Step 3: Update tree.ts**

Change the generic default from `unknown` to `TypedRow`:

```typescript
import type { TypedRow } from '@casehubio/pages-data/dist/dataset/types.js';

export interface TreeRow {
  readonly row: TypedRow;
  readonly depth: number;
  readonly hasChildren: boolean;
  readonly expanded: boolean;
}

export function flattenTree(
  rows: readonly TypedRow[],
  getChildren: (row: TypedRow) => readonly TypedRow[],
  expandedIds: ReadonlySet<string>,
  getRowId: (row: TypedRow) => string,
  depth = 0,
): TreeRow[] {
  // ... same logic, just typed
}
```

- [ ] **Step 4: Rewrite PagesDataTable → PagesTable**

This is the core change. The component's public API changes from `rows`/`columns` to `dataSet`/`columnRenderers`/`columnConfig`.

Key property changes:
```typescript
// REMOVED:
//   rows: readonly unknown[]
//   columns: readonly ColumnDef[]

// ADDED:
@property({ attribute: false }) dataSet?: TypedDataSet;
@property({ attribute: false }) columnRenderers?: ReadonlyMap<ColumnId, (cell: CellValue, row: TypedRow, column: Column) => TemplateResult | string>;
@property({ attribute: false }) columnConfig?: readonly TableColumnConfig[];
```

Internal changes:
- `_visibleColumns()` reads from `this.dataSet?.columns` filtered by `columnConfig`
- `_renderCell` uses `row.cell(column.id)` + `columnRenderers.get(column.id)` or `_formatValue`
- `_formatValue` dispatches on `CellValue.type` instead of `column.type`
- `_visibleRows()` returns `TypedRow[]`, sorts via updated `createMultiComparator`
- `getRowKey` typed as `(row: TypedRow) => string`
- Selection, tree, and keyboard navigation work with `TypedRow`

Use `ide_edit_member` for each changed member. The CSS styles are unchanged.

Rename class: `ide_refactor_rename` on `PagesDataTable` → `PagesTable`
Rename element: change `@customElement('pages-data-table')` → `@customElement('pages-table')`

- [ ] **Step 5: Update index.ts exports**

```typescript
export { PagesTable } from './pages-table.js';
export type {
  TableColumnConfig,
  DisplayMode,
  SelectionMode,
  SortDirection,
  SortEntry,
  ColumnAlign,
  SortChangeDetail,
  PageChangeDetail,
  SelectionChangeDetail,
  ColumnChangeDetail,
  RowActivateDetail,
  FilterChangeDetail,
  LoadMoreDetail,
} from './types.js';
export { computeScrollWindow, type ScrollWindow } from './virtual-scroll-engine.js';
export { createComparator, createMultiComparator } from './sort.js';
export { tableToCsv, downloadCsv, copyToClipboard } from './csv-export.js';
export { flattenTree, type TreeRow } from './tree.js';
```

- [ ] **Step 6: Rename package**

In `package.json`, change `"name"` from `"@casehubio/pages-data-table"` to `"@casehubio/pages-table"`.

Add `@casehubio/pages-data` as a dependency (for TypedDataSet types):
```json
"dependencies": {
  "lit": "^3.0.0",
  "@casehubio/pages-data": "workspace:*"
}
```

Rename the source file: `ide_move_file` from `pages-data-table.ts` → `pages-table.ts`.

- [ ] **Step 7: Rewrite tests for TypedDataSet API**

Rewrite `pages-data-table.test.ts` (rename to `pages-table.test.ts`) using `toTypedDataSet` and `fromRows` to create test data:

```typescript
import { toTypedDataSet } from '@casehubio/pages-data/dist/dataset/conversion.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { TypedDataSet } from '@casehubio/pages-data/dist/dataset/types.js';

function makeTestDataSet(count = 3): TypedDataSet {
  const columns = [
    { id: columnId("name"), name: "Name", type: ColumnType.TEXT },
    { id: columnId("age"), name: "Age", type: ColumnType.NUMBER },
  ];
  const data = Array.from({ length: count }, (_, i) => [`Person ${i}`, String(20 + i)]);
  return toTypedDataSet({ columns, data });
}
```

Cover: rendering columns from dataSet, cell value display, columnRenderers, columnConfig (visibility, width, sortable), client-sort with TypedRow, client-filter, selection, pagination, keyboard navigation, tree-table.

- [ ] **Step 8: Run all table tests**

Run: `cd /Users/mdproctor/claude/casehub/pages && yarn workspace @casehubio/pages-table test -- --run`

Expected: PASS

- [ ] **Step 9: Update vitest configs in blocks-ui**

All blocks-ui vitest configs that alias `@casehubio/pages-data-table` need updating to `@casehubio/pages-table` and pointing to the renamed source. These are in every component's `vitest.config.ts`:

```typescript
{ find: '@casehubio/pages-table', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data-table/src') },
```

(The directory on disk stays `pages-data-table` until the pages session renames it — the alias just maps the new package name.)

---

### Task 5: fetchSource pipeline integration (blocks-ui-core)

**Files:**
- Modify: `blocks-ui/packages/blocks-ui-core/src/data-source/fetch-source.ts`
- Rewrite: `blocks-ui/packages/blocks-ui-core/src/data-source/fetch-source.test.ts`

**Interfaces:**
- Consumes: `extractDataSet` from pages-data (Task 1); `ExtractionDef`, `ExternalColumnDef` from pages-data; `SnapshotEvent.totalRows` from Task 2
- Produces: `fetchSource(url, options)` where options includes `columns`, `dataPath`, `totalPath` (used by Tasks 6, 7, 8)

- [ ] **Step 1: Write failing test for extraction pipeline integration**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { fetchSource } from './fetch-source.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { DataSink } from '@casehubio/pages-data/dist/datasource/types.js';

describe('fetchSource with extraction pipeline', () => {
  it('produces TypedDataSet with working accessors from JSON array', async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      headers: new Headers({ 'content-type': 'application/json' }),
      json: () => Promise.resolve([
        { name: 'Alice', score: 42 },
        { name: 'Bob', score: 88 },
      ]),
    });

    const source = fetchSource('http://test/api', {
      fetchFn: mockFetch,
      columns: [
        { id: columnId('name'), type: ColumnType.TEXT },
        { id: columnId('score'), type: ColumnType.NUMBER },
      ],
    });

    const events: any[] = [];
    const sink: DataSink = {
      apply: (e) => events.push(e),
      error: (e) => events.push(e),
    };

    source.connect(sink);
    await vi.waitFor(() => expect(events).toHaveLength(1));

    const event = events[0]!;
    expect(event.type).toBe('snapshot');
    expect(event.dataset.columns).toHaveLength(2);
    expect(event.dataset.rows).toHaveLength(2);
    // Verify TypedRow accessors work
    expect(event.dataset.rows[0]!.text(columnId('name'))).toBe('Alice');
    expect(event.dataset.rows[0]!.number(columnId('score'))).toBe(42);
  });

  it('extracts totalRows via totalPath', async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      headers: new Headers({ 'content-type': 'application/json' }),
      json: () => Promise.resolve({
        items: [{ name: 'Alice' }],
        total: 100,
      }),
    });

    const source = fetchSource('http://test/api', {
      fetchFn: mockFetch,
      columns: [{ id: columnId('name'), type: ColumnType.TEXT }],
      dataPath: 'items',
      totalPath: 'total',
    });

    const events: any[] = [];
    source.connect({
      apply: (e) => events.push(e),
      error: (e) => events.push(e),
    });

    await vi.waitFor(() => expect(events).toHaveLength(1));
    expect(events[0]!.totalRows).toBe(100);
  });

  it('routes extraction errors to sink.error', async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      headers: new Headers({ 'content-type': 'application/json' }),
      json: () => Promise.resolve('not-an-array-or-object'),
    });

    const source = fetchSource('http://test/api', {
      fetchFn: mockFetch,
      columns: [{ id: columnId('x'), type: ColumnType.TEXT }],
    });

    const errors: any[] = [];
    source.connect({
      apply: () => {},
      error: (e) => errors.push(e),
    });

    await vi.waitFor(() => expect(errors).toHaveLength(1));
    expect(errors[0]!.permanent).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test -- --run -t "extraction pipeline"`

Expected: FAIL — `fetchSource` doesn't accept `columns`/`dataPath`/`totalPath` and doesn't produce TypedDataSet.

- [ ] **Step 3: Implement fetchSource with extraction pipeline**

Rewrite `fetch-source.ts`:

```typescript
import type { DataSource, DataSink } from "@casehubio/pages-data/dist/datasource/types.js";
import type { ExternalColumnDef } from "@casehubio/pages-data/dist/dataset/external/types.js";
import type { ExtractionDef } from "@casehubio/pages-data/dist/dataset/external/types.js";
import type { FetchResult, PresetRegistry } from "@casehubio/pages-data/dist/dataset/external/types.js";
import { extractDataSet } from "@casehubio/pages-data/dist/dataset/external/extraction.js";

export interface FetchSourceOptions {
  readonly method?: string;
  readonly headers?: Record<string, string> | (() => Record<string, string>);
  readonly body?: string;
  readonly fetchFn?: typeof globalThis.fetch;
  readonly columns?: readonly ExternalColumnDef[];
  readonly dataPath?: string;
  readonly totalPath?: string;
}

const emptyPresetRegistry: PresetRegistry = {
  get: () => undefined,
  has: () => false,
};

function navigatePath(data: unknown, path: string): number | undefined {
  let current: unknown = data;
  for (const segment of path.split(".")) {
    if (current === null || current === undefined || typeof current !== "object") return undefined;
    current = (current as Record<string, unknown>)[segment];
  }
  return typeof current === "number" ? current : undefined;
}

export function fetchSource(url: string, options?: FetchSourceOptions): DataSource {
  let abort: AbortController | undefined;
  return {
    connect(sink: DataSink) {
      abort = new AbortController();
      const signal = abort.signal;
      const doFetch = options?.fetchFn ?? globalThis.fetch.bind(globalThis);
      const headers = typeof options?.headers === "function"
        ? options.headers()
        : options?.headers;
      const init: RequestInit = { signal };
      if (options?.method) init.method = options.method;
      if (headers) init.headers = headers;
      if (options?.body) init.body = options.body;
      doFetch(url, init)
        .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
        .then(async (data) => {
          if (signal.aborted) return;

          let totalRows: number | undefined;
          if (options?.totalPath) {
            totalRows = navigatePath(data, options.totalPath);
          }

          const contentType = undefined; // already parsed as JSON
          const result: FetchResult = { data, contentType };
          const def: ExtractionDef = {
            columns: options?.columns,
            dataPath: options?.dataPath,
          };

          try {
            const { dataset } = await extractDataSet(result, def, emptyPresetRegistry);
            if (!signal.aborted) {
              sink.apply({ type: "snapshot", dataset, totalRows });
            }
          } catch (err) {
            if (!signal.aborted) {
              sink.error({
                message: err instanceof Error ? err.message : String(err),
                permanent: true,
              });
            }
          }
        })
        .catch(err => {
          if (!signal.aborted && err.name !== "AbortError") {
            sink.error({ message: err.message, permanent: true });
          }
        });
    },
    disconnect() {
      abort?.abort();
      abort = undefined;
    },
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test -- --run -t "fetchSource"`

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/data-source/fetch-source.ts packages/blocks-ui-core/src/data-source/fetch-source.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: fetchSource uses extraction pipeline — produces real TypedDataSet (#49)"
```

---

### Task 6: DataSourceAdapter and DataSourceMixin type propagation (blocks-ui-core)

**Files:**
- Modify: `blocks-ui/packages/blocks-ui-core/src/data-source/data-source-adapter.ts`
- Modify: `blocks-ui/packages/blocks-ui-core/src/data-source/data-source-mixin.ts`
- Modify: `blocks-ui/packages/blocks-ui-core/src/data-source/data-source-adapter.test.ts`
- Modify: `blocks-ui/packages/blocks-ui-core/src/data-source/data-source-mixin.test.ts`
- Modify: `blocks-ui/packages/blocks-ui-core/src/data-source/index.ts`

**Interfaces:**
- Consumes: `DataSourceController` with `TypedDataSet | undefined` from Task 3; `SourceFactory` with `SourceFactoryOptions` from Task 3
- Produces: `DataSourceAdapter.dataSet: TypedDataSet | undefined`; `DataSourceMixin.dataSet: TypedDataSet | undefined`; `createSourceFactory()` passing columns/dataPath/totalPath (used by Tasks 7, 8)

- [ ] **Step 1: Update DataSourceAdapter**

Change `dataSet` accessors from `unknown` to `TypedDataSet | undefined`:

```typescript
get dataSet(): TypedDataSet | undefined { return this.controller.dataSet; }
set dataSet(v: TypedDataSet | undefined) { this.controller.dataSet = v; }
```

Add import for `TypedDataSet`. Use `ide_edit_member` for each accessor.

- [ ] **Step 2: Update DataSourceMixin**

Change `dataSet` accessors and return type in the mixin:

```typescript
get dataSet(): TypedDataSet | undefined { return this.dataSource.dataSet; }
set dataSet(v: TypedDataSet | undefined) { this.dataSource.dataSet = v; }
```

Update `createSourceFactory()` to pass options through:

```typescript
createSourceFactory(): SourceFactory {
  return (url, _id, options) => fetchSource(url, {
    columns: options?.columns,
    dataPath: options?.dataPath,
    totalPath: options?.totalPath,
  });
}
```

Update the mixin's return type declaration to include `TypedDataSet | undefined` for `dataSet`.

- [ ] **Step 3: Update index.ts exports**

Ensure `SourceFactoryOptions` is re-exported if consumers need it. Update any type re-exports.

- [ ] **Step 4: Update existing tests**

Fix any test that sets `adapter.dataSet = someUnknownValue` — it must now provide `TypedDataSet | undefined`. Use `toTypedDataSet()` or `fromRows()` to create test data.

- [ ] **Step 5: Run all blocks-ui-core tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test -- --run`

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/data-source/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: DataSourceAdapter/Mixin typed as TypedDataSet (#49)"
```

---

### Task 7: Migrate list-pane (blocks-ui)

**Files:**
- Modify: `blocks-ui/components/list-pane/src/list-pane.ts`
- Rewrite: `blocks-ui/components/list-pane/src/list-pane.test.ts`

**Interfaces:**
- Consumes: `TypedDataSet`, `TypedRow` from pages-data; `PagesTable` with `dataSet`, `columnRenderers`, `columnConfig` from Task 4; `DataSourceMixin.dataSet: TypedDataSet | undefined` from Task 6
- Produces: `<list-pane>` with `columnConfig`, `columnRenderers` properties (replacing old `columns: ColumnDef[]`)

- [ ] **Step 1: Rewrite list-pane to pass TypedDataSet directly**

The key change: remove `_rows`, `_totalRows`, and the `willUpdate` extraction logic. Pass `this.dataSet` directly to `<pages-table>`.

Remove: `columns: ColumnDef<any>[]` property, `_rows` state, `_totalRows` state, `_lastDataSet` field, the `willUpdate` data extraction block.

Add: `columnConfig` and `columnRenderers` properties that pass through to the table.

```typescript
@property({ attribute: false }) columnConfig?: readonly TableColumnConfig[];
@property({ attribute: false }) columnRenderers?: ReadonlyMap<ColumnId, (cell: CellValue, row: TypedRow, column: Column) => TemplateResult | string>;
```

Render method:
```typescript
override render() {
  if (!this.loading && !this.dataSet?.rows.length && !this.error) {
    return html`<div class="empty" role="status">${this.emptyMessage}</div>`;
  }

  return html`
    <pages-table
      .dataSet=${this.dataSet}
      .columnConfig=${this.columnConfig}
      .columnRenderers=${this.columnRenderers}
      .getRowKey=${this.getRowKey}
      .getRowClass=${this.getRowClass}
      .totalRows=${this.dataSource.controller.totalRows > 0 ? this.dataSource.controller.totalRows : undefined}
      .pageSize=${this.pageSize}
      .loading=${this.loading}
      .emptyMessage=${this.emptyMessage}
      selection="single"
      mode="paginated"
      client-sort
      client-filter
      @row-activate=${this._handleRowActivate}
    ></pages-table>
  `;
}
```

Update import from `@casehubio/pages-table` instead of `@casehubio/pages-data-table`.

- [ ] **Step 2: Update tests**

Rewrite tests to provide TypedDataSet via `toTypedDataSet()` or `fromRows()` instead of setting `columns` + mock data.

- [ ] **Step 3: Run list-pane tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/list-pane test -- --run`

Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/list-pane/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: list-pane consumes TypedDataSet natively (#49)"
```

---

### Task 8: Migrate remaining consumers (blocks-ui)

**Files:**
- Modify: `blocks-ui/components/trust-score-panel/src/trust-score-panel.ts`
- Modify: `blocks-ui/components/audit-trail-viewer/src/audit-trail-viewer.ts`
- Modify: `blocks-ui/components/work-item-inbox/src/work-item-inbox.ts`
- Modify: `blocks-ui/components/notification-inbox/src/notification-inbox.ts`
- Modify: `blocks-ui/components/notification-inbox/src/subscription-list.ts`
- Update tests for each component
- Modify: all vitest.config.ts files with pages-data-table alias
- Modify: `blocks-ui/examples/vite.config.ts` and `blocks-ui/examples/src/pages/data-table-page.ts`

**Interfaces:**
- Consumes: All from Tasks 1-7

This task migrates all remaining consumers from `ColumnDef` + `rows: unknown[]` + `<pages-data-table>` to `TypedDataSet` + `columnRenderers` + `<pages-table>`.

- [ ] **Step 1: Migrate trust-score-panel**

Replace the broken ColumnDef columns (using wrong `key`/`header` properties) with `fromRows()` + `columnRenderers`:

```typescript
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';

// In _renderCapabilityTable():
const dataset = fromRows(capabilities, [
  { id: columnId("tag"), name: "Capability", type: ColumnType.TEXT, getValue: (c: { tag: string; score: number }) => c.tag },
  { id: columnId("score"), name: "Score", type: ColumnType.NUMBER, getValue: (c: { tag: string; score: number }) => c.score },
]);

const renderers = new Map([
  [columnId("score"), (cell: CellValue) => {
    const numValue = cell.type === "NULL" ? 0 : (cell as { value: number }).value;
    const level = trustLevelFromScore(numValue);
    return html`
      <div class="score-bar">
        <div class="score-bar-fill ${level}" style="width: ${numValue * 100}%"></div>
      </div>
      <span style="margin-left: 8px">${numValue.toFixed(2)}</span>
    `;
  }],
]);

return html`
  <pages-table
    .dataSet=${dataset}
    .columnRenderers=${renderers}
    .columnConfig=${[
      { id: columnId("tag"), sortable: true },
      { id: columnId("score"), sortable: true },
    ]}
  ></pages-table>
`;
```

- [ ] **Step 2: Migrate audit-trail-viewer**

Fix the broken `.data=` property (should be `.rows=`, now becomes `.dataSet=`). Declare columns in DataSourceAdapter options. Use `columnRenderers` for timestamp/actor/digest formatting.

The component currently casts `entries.dataSet as LedgerEntry[]`. After this change, `entries.dataSet` is `TypedDataSet` — the pipeline converts the JSON response. The component's `_filteredEntries()` method filters `TypedRow[]` instead of `LedgerEntry[]`.

- [ ] **Step 3: Migrate work-item-inbox**

Declare columns in the source factory or use `fromRows()` for the items fetched directly. Replace `ColumnDef<WorkItemRootResponse>` columns with `columnConfig` + `columnRenderers`. All custom renderers (status pills, relative time, etc.) move to `columnRenderers` operating on `CellValue`.

- [ ] **Step 4: Migrate notification-inbox and subscription-list**

Same pattern — replace `ColumnDef` columns with `columnConfig` + `columnRenderers`. Update import from `@casehubio/pages-table`.

- [ ] **Step 5: Update all vitest.config.ts aliases**

Every component's vitest.config.ts that has:
```typescript
{ find: '@casehubio/pages-data-table', replacement: ... }
```
Changes to:
```typescript
{ find: '@casehubio/pages-table', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data-table/src') }
```

Components: list-pane, trust-score-panel, audit-trail-viewer, work-item-inbox, work-item-workbench, notification-inbox, split-workbench, detail-pane, approval-gate, case-timeline, kpi-metric-row.

Also update `examples/vite.config.ts` and `examples/vitest.config.ts`.

- [ ] **Step 6: Update data-table-page example**

Rewrite `examples/src/pages/data-table-page.ts` to use `<pages-table>` with `TypedDataSet`.

- [ ] **Step 7: Run all tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn test`

Expected: All pass.

- [ ] **Step 8: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`

Expected: Clean.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add .
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: migrate all consumers to TypedDataSet + pages-table (#49)"
```
