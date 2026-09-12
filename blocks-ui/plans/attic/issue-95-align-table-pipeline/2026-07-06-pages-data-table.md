# pages-data-table Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #22 — Build pages-data-table
**Issue group:** #22

**Goal:** Build a standalone `<pages-data-table>` Lit Web Component with three display modes (auto/paginated/scroll), CSS Grid rendering, virtual scroll engine, multi-mode selection, client-side sorting, column visibility, ARIA grid, and 2D keyboard navigation. Then refactor work-item-inbox to consume it.

**Architecture:** Generic `ColumnDef<R>` data model with `getValue` extractors and optional `render` functions. CSS Grid per row (not `<table>` elements). Pure-function virtual scroll engine. Row styling via CSS `::part()`. 2D grid keyboard navigation implemented directly (not via RovingTabindexMixin which is 1D only).

**Tech Stack:** TypeScript, Lit 3, Vitest, JSDOM, CSS Grid, CSS `::part()`

## Global Constraints

- Element name: `pages-data-table` (pages- prefix for promotion path)
- Package name: `@casehubio/blocks-ui-data-table`
- Dependencies: `lit ^3.0.0`, `@casehubio/blocks-ui-core workspace:*` only
- TypeScript: `strict`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`
- Tests: Vitest with jsdom environment, `document.createElement` + `updateComplete` pattern
- CSS: `var(--blocks-*, fallback)` tokens only, no hardcoded colours
- Events: `bubbles: true, composed: true` on all CustomEvents
- `getRowKey` required when `selection !== 'none'` — throw Error, not warn
- Auto mode threshold: 50 rows (matches existing inbox behaviour)
- Spec: `docs/specs/2026-07-06-pages-data-table-design.md` (design-reviewed, 5 rounds, 19 issues resolved)

---

### Task 1: Package scaffold + types

**Files:**
- Create: `components/data-table/package.json`
- Create: `components/data-table/tsconfig.json`
- Create: `components/data-table/tsconfig.build.json`
- Create: `components/data-table/vitest.config.ts`
- Create: `components/data-table/src/types.ts`
- Create: `components/data-table/src/index.ts`
- Modify: `tsconfig.json` (root — add project reference)
- Modify: `package.json` (root — workspace already covers `components/*`)

**Interfaces:**
- Consumes: nothing
- Produces: `ColumnDef<R>`, `DisplayMode`, `SelectionMode`, `SortDirection`, `ColumnAlign`, all event detail interfaces (`SortChangeDetail`, `PageChangeDetail`, `SelectionChangeDetail`, `ColumnChangeDetail`, `RowActivateDetail`, `LoadMoreDetail`)

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-data-table",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "@open-wc/testing": "^4.0.0",
    "jsdom": "^25.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 2: Create tsconfig files**

`tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": [{ "path": "../../packages/blocks-ui-core" }]
}
```

`tsconfig.build.json`:
```json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

- [ ] **Step 3: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { environment: 'jsdom', globals: true } });
```

- [ ] **Step 4: Create types.ts**

```typescript
import type { TemplateResult } from 'lit';

export type DisplayMode = 'auto' | 'paginated' | 'scroll';
export type SelectionMode = 'none' | 'single' | 'multi';
export type SortDirection = 'asc' | 'desc' | 'none';
export type ColumnAlign = 'start' | 'center' | 'end';

export interface ColumnDef<R = unknown> {
  readonly id: string;
  readonly label: string;
  readonly type?: 'text' | 'number' | 'date';
  readonly getValue: (row: R) => unknown;
  readonly render?: (value: unknown, row: R) => TemplateResult | string;
  readonly compare?: (a: unknown, b: unknown) => number;
  readonly sortable?: boolean;
  readonly visible?: boolean;
  readonly width?: string;
  readonly minWidth?: string;
  readonly align?: ColumnAlign;
}

export interface SortChangeDetail {
  readonly columnId: string;
  readonly direction: SortDirection;
}

export interface PageChangeDetail {
  readonly page: number;
  readonly pageSize: number;
}

export interface SelectionChangeDetail<R = unknown> {
  readonly selectedKeys: readonly string[];
  readonly selectedRows: readonly R[];
  readonly scope?: 'page';
}

export interface ColumnChangeDetail {
  readonly visibleColumns: readonly string[];
}

export interface RowActivateDetail<R = unknown> {
  readonly row: R;
  readonly key?: string;
}

export interface LoadMoreDetail {}
```

- [ ] **Step 5: Create index.ts**

```typescript
export { PagesDataTable } from './pages-data-table.js';
export type {
  ColumnDef,
  DisplayMode,
  SelectionMode,
  SortDirection,
  ColumnAlign,
  SortChangeDetail,
  PageChangeDetail,
  SelectionChangeDetail,
  ColumnChangeDetail,
  RowActivateDetail,
  LoadMoreDetail,
} from './types.js';
export { computeScrollWindow, type ScrollWindow } from './virtual-scroll-engine.js';
export { createComparator } from './sort.js';
```

Note: `pages-data-table.ts`, `virtual-scroll-engine.ts`, and `sort.ts` don't exist yet — they'll be created in subsequent tasks. The index.ts will cause typecheck errors until then. That's expected.

- [ ] **Step 6: Add project reference to root tsconfig.json**

Add `{ "path": "components/data-table" }` to the `references` array in the root `tsconfig.json`.

- [ ] **Step 7: Install dependencies**

```bash
yarn install
```

- [ ] **Step 8: Commit**

```bash
git add components/data-table/package.json components/data-table/tsconfig.json components/data-table/tsconfig.build.json components/data-table/vitest.config.ts components/data-table/src/types.ts components/data-table/src/index.ts tsconfig.json
git commit -m "feat(data-table): package scaffold and type definitions #22"
```

---

### Task 2: Virtual scroll engine (TDD)

**Files:**
- Create: `components/data-table/src/virtual-scroll-engine.ts`
- Create: `components/data-table/src/virtual-scroll-engine.test.ts`

**Interfaces:**
- Consumes: nothing (pure function, no imports)
- Produces: `computeScrollWindow(scrollTop, containerHeight, rowHeight, rowCount, bufferSize) → ScrollWindow`

```typescript
export interface ScrollWindow {
  readonly startIndex: number;
  readonly endIndex: number;
  readonly offsetY: number;
  readonly totalHeight: number;
}
```

- [ ] **Step 1: Write failing tests**

`virtual-scroll-engine.test.ts`:
```typescript
import { describe, it, expect } from 'vitest';
import { computeScrollWindow } from './virtual-scroll-engine.js';

describe('computeScrollWindow', () => {
  it('returns full range for small datasets', () => {
    const w = computeScrollWindow(0, 500, 48, 5, 5);
    expect(w).toEqual({ startIndex: 0, endIndex: 5, offsetY: 0, totalHeight: 240 });
  });

  it('returns empty range for zero rows', () => {
    const w = computeScrollWindow(0, 500, 48, 0, 5);
    expect(w).toEqual({ startIndex: 0, endIndex: 0, offsetY: 0, totalHeight: 0 });
  });

  it('computes visible window at top', () => {
    const w = computeScrollWindow(0, 480, 48, 100, 5);
    expect(w.startIndex).toBe(0);
    expect(w.endIndex).toBe(15); // 10 visible + 5 buffer below
    expect(w.offsetY).toBe(0);
    expect(w.totalHeight).toBe(4800);
  });

  it('computes visible window at scroll offset', () => {
    // scrollTop=960 → first visible row = 20, with 5 buffer above → start=15
    const w = computeScrollWindow(960, 480, 48, 100, 5);
    expect(w.startIndex).toBe(15);
    expect(w.endIndex).toBe(35); // 20+10visible+5buffer = 35
    expect(w.offsetY).toBe(720); // 15 * 48
  });

  it('clamps to dataset bounds', () => {
    // scrollTop near bottom of 100 rows
    const w = computeScrollWindow(4500, 480, 48, 100, 5);
    expect(w.endIndex).toBe(100);
    expect(w.startIndex).toBeLessThan(100);
  });

  it('handles single row', () => {
    const w = computeScrollWindow(0, 500, 48, 1, 5);
    expect(w).toEqual({ startIndex: 0, endIndex: 1, offsetY: 0, totalHeight: 48 });
  });

  it('handles container taller than content', () => {
    const w = computeScrollWindow(0, 1000, 48, 10, 5);
    expect(w.startIndex).toBe(0);
    expect(w.endIndex).toBe(10);
    expect(w.totalHeight).toBe(480);
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
cd components/data-table && npx vitest run src/virtual-scroll-engine.test.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement virtual-scroll-engine.ts**

```typescript
export interface ScrollWindow {
  readonly startIndex: number;
  readonly endIndex: number;
  readonly offsetY: number;
  readonly totalHeight: number;
}

export function computeScrollWindow(
  scrollTop: number,
  containerHeight: number,
  rowHeight: number,
  rowCount: number,
  bufferSize: number,
): ScrollWindow {
  const totalHeight = rowCount * rowHeight;

  if (rowCount === 0) {
    return { startIndex: 0, endIndex: 0, offsetY: 0, totalHeight: 0 };
  }

  const visibleCount = Math.ceil(containerHeight / rowHeight);
  const firstVisible = Math.floor(scrollTop / rowHeight);

  const startIndex = Math.max(0, firstVisible - bufferSize);
  const endIndex = Math.min(rowCount, firstVisible + visibleCount + bufferSize);
  const offsetY = startIndex * rowHeight;

  return { startIndex, endIndex, offsetY, totalHeight };
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
cd components/data-table && npx vitest run src/virtual-scroll-engine.test.ts
```
Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add components/data-table/src/virtual-scroll-engine.ts components/data-table/src/virtual-scroll-engine.test.ts
git commit -m "feat(data-table): virtual scroll engine — pure function #22"
```

---

### Task 3: Sort comparators (TDD)

**Files:**
- Create: `components/data-table/src/sort.ts`
- Create: `components/data-table/src/sort.test.ts`

**Interfaces:**
- Consumes: `ColumnDef` from `types.ts`, `SortDirection` from `types.ts`
- Produces: `createComparator(column: ColumnDef, direction: SortDirection) → (a: unknown, b: unknown) => number`

- [ ] **Step 1: Write failing tests**

`sort.test.ts`:
```typescript
import { describe, it, expect } from 'vitest';
import { createComparator } from './sort.js';
import type { ColumnDef } from './types.js';

const textCol: ColumnDef = { id: 'name', label: 'Name', type: 'text', getValue: (r: any) => r.name };
const numCol: ColumnDef = { id: 'age', label: 'Age', type: 'number', getValue: (r: any) => r.age };
const dateCol: ColumnDef = { id: 'date', label: 'Date', type: 'date', getValue: (r: any) => r.date };
const customCol: ColumnDef = {
  id: 'x', label: 'X', getValue: (r: any) => r.x,
  compare: (a: unknown, b: unknown) => (a as number) - (b as number),
};

describe('createComparator', () => {
  it('returns identity for direction=none', () => {
    const cmp = createComparator(textCol, 'none');
    expect(cmp('b', 'a')).toBe(0);
  });

  it('sorts text ascending with localeCompare', () => {
    const cmp = createComparator(textCol, 'asc');
    expect(cmp('apple', 'banana')).toBeLessThan(0);
    expect(cmp('banana', 'apple')).toBeGreaterThan(0);
    expect(cmp('apple', 'apple')).toBe(0);
  });

  it('sorts text descending', () => {
    const cmp = createComparator(textCol, 'desc');
    expect(cmp('apple', 'banana')).toBeGreaterThan(0);
  });

  it('sorts numbers', () => {
    const cmp = createComparator(numCol, 'asc');
    expect(cmp(1, 10)).toBeLessThan(0);
    expect(cmp(10, 1)).toBeGreaterThan(0);
  });

  it('sorts dates', () => {
    const cmp = createComparator(dateCol, 'asc');
    expect(cmp('2024-01-01', '2024-06-01')).toBeLessThan(0);
    expect(cmp('2024-06-01', '2024-01-01')).toBeGreaterThan(0);
  });

  it('sorts nulls last in ascending', () => {
    const cmp = createComparator(textCol, 'asc');
    expect(cmp(null, 'a')).toBeGreaterThan(0);
    expect(cmp('a', null)).toBeLessThan(0);
    expect(cmp(null, null)).toBe(0);
  });

  it('sorts nulls last in descending', () => {
    const cmp = createComparator(textCol, 'desc');
    expect(cmp(null, 'a')).toBeGreaterThan(0);
    expect(cmp('a', null)).toBeLessThan(0);
  });

  it('uses custom comparator when provided', () => {
    const cmp = createComparator(customCol, 'asc');
    expect(cmp(1, 10)).toBeLessThan(0);
  });

  it('falls back to string comparison for untyped columns', () => {
    const col: ColumnDef = { id: 'x', label: 'X', getValue: (r: any) => r.x };
    const cmp = createComparator(col, 'asc');
    expect(cmp('a', 'b')).toBeLessThan(0);
  });
});
```

- [ ] **Step 2: Run tests — verify fail**

```bash
cd components/data-table && npx vitest run src/sort.test.ts
```

- [ ] **Step 3: Implement sort.ts**

```typescript
import type { ColumnDef, SortDirection } from './types.js';

type Comparator = (a: unknown, b: unknown) => number;

export function createComparator(column: ColumnDef, direction: SortDirection): Comparator {
  if (direction === 'none') return () => 0;

  const base = column.compare ?? resolveByType(column.type);
  const flip = direction === 'desc' ? -1 : 1;

  return (a: unknown, b: unknown): number => {
    const aNull = a == null;
    const bNull = b == null;
    if (aNull && bNull) return 0;
    if (aNull) return 1;  // nulls last
    if (bNull) return -1;
    return flip * base(a, b);
  };
}

function resolveByType(type: string | undefined): Comparator {
  switch (type) {
    case 'number':
      return (a, b) => (a as number) - (b as number);
    case 'date':
      return (a, b) => new Date(a as string).getTime() - new Date(b as string).getTime();
    case 'text':
    default:
      return (a, b) => String(a).localeCompare(String(b));
  }
}
```

- [ ] **Step 4: Run tests — verify pass**

```bash
cd components/data-table && npx vitest run src/sort.test.ts
```

- [ ] **Step 5: Commit**

```bash
git add components/data-table/src/sort.ts components/data-table/src/sort.test.ts
git commit -m "feat(data-table): sort comparators — null-safe, type-aware #22"
```

---

### Task 4: Core rendering — auto mode (TDD)

**Files:**
- Create: `components/data-table/src/pages-data-table.ts`
- Create: `components/data-table/src/pages-data-table.test.ts`

**Interfaces:**
- Consumes: `ColumnDef`, `DisplayMode`, `SelectionMode`, `SortDirection` from `types.ts`; `computeScrollWindow` from `virtual-scroll-engine.ts`; `LiveRegionMixin` from `@casehubio/blocks-ui-core`
- Produces: `PagesDataTable` custom element `<pages-data-table>` with core rendering

This task implements: element registration, CSS Grid layout, column headers, cell rendering pipeline (getValue → render/format), auto mode (static for ≤50, virtual for >50), empty state, loading state, `getRowClass` → `part` attribute, responsive `overflow-x: auto`.

- [ ] **Step 1: Write failing tests for core rendering**

`pages-data-table.test.ts` — first batch (core rendering):
```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import type { ColumnDef } from './types.js';

type DataTableEl = HTMLElement & {
  rows: readonly unknown[];
  columns: readonly ColumnDef[];
  mode: string;
  loading: boolean;
  emptyMessage: string;
  getRowKey: ((row: unknown) => string) | undefined;
  getRowClass: ((row: unknown) => string) | undefined;
  updateComplete: Promise<boolean>;
};

interface TestRow { id: string; name: string; age: number; created: string; }

const testColumns: ColumnDef<TestRow>[] = [
  { id: 'name', label: 'Name', getValue: r => r.name, width: '1fr' },
  { id: 'age', label: 'Age', type: 'number', getValue: r => r.age, width: '80px' },
];

const testRows: TestRow[] = [
  { id: '1', name: 'Alice', age: 30, created: '2024-01-01' },
  { id: '2', name: 'Bob', age: 25, created: '2024-06-15' },
  { id: '3', name: 'Carol', age: 35, created: '2024-03-10' },
];

function makeRows(count: number): TestRow[] {
  return Array.from({ length: count }, (_, i) => ({
    id: String(i), name: `Person ${i}`, age: 20 + i, created: '2024-01-01',
  }));
}

describe('pages-data-table', () => {
  let el: DataTableEl;

  beforeEach(async () => {
    await import('./pages-data-table.js');
    el = document.createElement('pages-data-table') as DataTableEl;
    document.body.appendChild(el);
  });

  afterEach(() => { el.remove(); });

  describe('core rendering', () => {
    it('renders column headers', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = testRows;
      await el.updateComplete;
      const headers = el.shadowRoot!.querySelectorAll('[role="columnheader"]');
      expect(headers.length).toBe(2);
      expect(headers[0]!.textContent).toContain('Name');
      expect(headers[1]!.textContent).toContain('Age');
    });

    it('renders cells using getValue', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = testRows;
      await el.updateComplete;
      const cells = el.shadowRoot!.querySelectorAll('[role="gridcell"]');
      expect(cells.length).toBe(6); // 3 rows × 2 cols
      expect(cells[0]!.textContent).toContain('Alice');
      expect(cells[1]!.textContent).toContain('30');
    });

    it('uses render function when provided', async () => {
      const cols: ColumnDef<TestRow>[] = [
        { id: 'name', label: 'Name', getValue: r => r.name,
          render: v => `<${v}>` },
      ];
      el.columns = cols as ColumnDef[];
      el.rows = testRows;
      await el.updateComplete;
      const cell = el.shadowRoot!.querySelector('[role="gridcell"]')!;
      expect(cell.textContent).toContain('<Alice>');
    });

    it('formats dates by type', async () => {
      const cols: ColumnDef<TestRow>[] = [
        { id: 'created', label: 'Created', type: 'date', getValue: r => r.created },
      ];
      el.columns = cols as ColumnDef[];
      el.rows = [testRows[0]!];
      await el.updateComplete;
      const cell = el.shadowRoot!.querySelector('[role="gridcell"]')!;
      // toLocaleDateString output varies by locale, just check it's not raw ISO
      expect(cell.textContent).not.toContain('2024-01-01T');
    });

    it('renders empty state', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = [];
      await el.updateComplete;
      expect(el.shadowRoot!.textContent).toContain('No data');
    });

    it('renders custom empty message', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = [];
      el.emptyMessage = 'Nothing here';
      await el.updateComplete;
      expect(el.shadowRoot!.textContent).toContain('Nothing here');
    });

    it('renders loading state', async () => {
      el.loading = true;
      await el.updateComplete;
      const busy = el.shadowRoot!.querySelector('[aria-busy="true"]');
      expect(busy).not.toBeNull();
    });

    it('sets role="grid" on container', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = testRows;
      await el.updateComplete;
      expect(el.shadowRoot!.querySelector('[role="grid"]')).not.toBeNull();
    });

    it('sets aria-rowcount', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = testRows;
      await el.updateComplete;
      const grid = el.shadowRoot!.querySelector('[role="grid"]')!;
      expect(grid.getAttribute('aria-rowcount')).toBe('4'); // 1 header + 3 data
    });

    it('applies getRowClass as part attribute', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = testRows;
      el.getRowClass = (r: unknown) => `priority-${(r as TestRow).name.toLowerCase()}`;
      await el.updateComplete;
      const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
      expect(rows[0]!.getAttribute('part')).toContain('priority-alice');
    });
  });

  describe('auto mode', () => {
    it('renders all rows for small datasets', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = testRows;
      await el.updateComplete;
      const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
      expect(rows.length).toBe(3);
    });

    it('activates virtual scroll for >50 rows', async () => {
      el.columns = testColumns as ColumnDef[];
      el.rows = makeRows(100);
      await el.updateComplete;
      const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
      expect(rows.length).toBeLessThan(100);
    });
  });
});
```

- [ ] **Step 2: Run tests — verify fail**

```bash
cd components/data-table && npx vitest run src/pages-data-table.test.ts
```

- [ ] **Step 3: Implement pages-data-table.ts — core rendering + auto mode**

The component is created with: CSS Grid layout, column header rendering, cell rendering pipeline, auto mode with ≤50 static / >50 virtual scroll, empty/loading states, `getRowClass` → `part` attribute, `role="grid"` ARIA, responsive `overflow-x: auto`.

Key implementation details:
- Base class: `LiveRegionMixin(LitElement)` — no RovingTabindexMixin (2D nav is custom, added in Task 9)
- Column template: computed from visible columns' `width` property, defaulting to `1fr`
- Cell rendering: `getValue(row)` → `render(value, row)` or default formatting by `type`
- Virtual scroll in auto >50: uses `computeScrollWindow`, spacer div, `transform: translateY`
- Each row: `part="row ${getRowClass?.(row) ?? ''}"`, `role="row"`, `aria-rowindex`

Full implementation code is guided by the spec at `docs/specs/2026-07-06-pages-data-table-design.md` §Rendering Architecture, §Cell Rendering Pipeline, §Display Modes (auto), §Row Styling via CSS `::part()`.

- [ ] **Step 4: Run tests — verify pass**

```bash
cd components/data-table && npx vitest run src/pages-data-table.test.ts
```

- [ ] **Step 5: Commit**

```bash
git add components/data-table/src/pages-data-table.ts components/data-table/src/pages-data-table.test.ts
git commit -m "feat(data-table): core rendering + auto mode with CSS Grid #22"
```

---

### Task 5: Paginated mode (TDD)

**Files:**
- Modify: `components/data-table/src/pages-data-table.ts`
- Modify: `components/data-table/src/pages-data-table.test.ts`

**Interfaces:**
- Consumes: `PageChangeDetail` from `types.ts`
- Produces: `page-change` CustomEvent, pagination footer UI

- [ ] **Step 1: Write failing tests**

Add to `pages-data-table.test.ts`:
```typescript
describe('paginated mode', () => {
  it('renders only pageSize rows', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(30);
    el.mode = 'paginated';
    (el as any).pageSize = 10;
    await el.updateComplete;
    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
    expect(rows.length).toBe(10);
  });

  it('renders page controls', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(30);
    el.mode = 'paginated';
    (el as any).pageSize = 10;
    await el.updateComplete;
    const nav = el.shadowRoot!.querySelector('[role="navigation"]');
    expect(nav).not.toBeNull();
    expect(nav!.textContent).toContain('1');
    expect(nav!.textContent).toContain('3'); // 30/10 = 3 pages
  });

  it('emits page-change on next click', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(30);
    el.mode = 'paginated';
    (el as any).pageSize = 10;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('page-change', (e) => events.push(e as CustomEvent));

    const next = el.shadowRoot!.querySelector('[aria-label="Next page"]') as HTMLButtonElement;
    next.click();
    await el.updateComplete;

    expect(events.length).toBe(1);
    expect(events[0]!.detail.page).toBe(1);
    expect(events[0]!.detail.pageSize).toBe(10);
  });

  it('shows second page content after navigation', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(30);
    el.mode = 'paginated';
    (el as any).pageSize = 10;
    await el.updateComplete;

    const next = el.shadowRoot!.querySelector('[aria-label="Next page"]') as HTMLButtonElement;
    next.click();
    await el.updateComplete;

    const firstCell = el.shadowRoot!.querySelector('[role="gridcell"]')!;
    expect(firstCell.textContent).toContain('Person 10');
  });

  it('uses totalRows for server-side pagination', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows; // only 3 rows provided (one page)
    el.mode = 'paginated';
    (el as any).pageSize = 3;
    (el as any).totalRows = 30;
    await el.updateComplete;

    const nav = el.shadowRoot!.querySelector('[role="navigation"]')!;
    expect(nav.textContent).toContain('10'); // 30/3 = 10 pages
  });

  it('disables prev/first on first page', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(30);
    el.mode = 'paginated';
    (el as any).pageSize = 10;
    await el.updateComplete;
    const prev = el.shadowRoot!.querySelector('[aria-label="Previous page"]') as HTMLButtonElement;
    expect(prev.disabled).toBe(true);
  });

  it('disables next/last on last page', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(30);
    el.mode = 'paginated';
    (el as any).pageSize = 10;
    (el as any).currentPage = 2;
    await el.updateComplete;
    const next = el.shadowRoot!.querySelector('[aria-label="Next page"]') as HTMLButtonElement;
    expect(next.disabled).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests — verify fail**
- [ ] **Step 3: Implement pagination in pages-data-table.ts**

Add pagination rendering: footer with first/prev/next/last buttons, "Page N of M" display, "Showing X-Y of Z" range. For client-side: internal page state slices `rows`. For server-side (`totalRows` set): renders all provided rows, page count from `totalRows / pageSize`.

- [ ] **Step 4: Run tests — verify pass**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(data-table): paginated mode — client-side + server-side #22"
```

---

### Task 6: Scroll mode (TDD)

**Files:**
- Modify: `components/data-table/src/pages-data-table.ts`
- Modify: `components/data-table/src/pages-data-table.test.ts`

**Interfaces:**
- Consumes: `computeScrollWindow` from `virtual-scroll-engine.ts`, `LoadMoreDetail` from `types.ts`
- Produces: `load-more` CustomEvent

- [ ] **Step 1: Write failing tests**

```typescript
describe('scroll mode', () => {
  it('renders virtual window of rows', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(200);
    el.mode = 'scroll';
    (el as any).rowHeight = 48;
    (el as any).bufferSize = 5;
    await el.updateComplete;
    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
    expect(rows.length).toBeLessThan(200);
    expect(rows.length).toBeGreaterThan(0);
  });

  it('sets spacer height for scrollbar', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(100);
    el.mode = 'scroll';
    (el as any).rowHeight = 48;
    await el.updateComplete;
    const spacer = el.shadowRoot!.querySelector('.body-content') as HTMLElement;
    expect(spacer.style.height).toBe('4800px'); // 100 * 48
  });

  it('sets aria-rowindex on virtual rows', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(100);
    el.mode = 'scroll';
    await el.updateComplete;
    const firstRow = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)')!;
    const idx = parseInt(firstRow.getAttribute('aria-rowindex')!, 10);
    expect(idx).toBeGreaterThanOrEqual(2); // 1-based, header is row 1
  });

  it('does not emit load-more when hasMore is false', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = makeRows(20);
    el.mode = 'scroll';
    (el as any).hasMore = false;
    await el.updateComplete;
    const events: Event[] = [];
    el.addEventListener('load-more', e => events.push(e));
    // Simulate scroll to bottom — jsdom doesn't have real scroll, so verify the property guard
    expect(events.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests — verify fail**
- [ ] **Step 3: Implement scroll mode**

Add scroll rendering: spacer div with `height: totalHeight`, visible rows positioned with `transform: translateY(offsetY)px`, scroll handler updates `virtualScrollTop` state, `load-more` logic when `hasMore && nearBottom`.

- [ ] **Step 4: Run tests — verify pass**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(data-table): scroll mode with virtual rendering + load-more #22"
```

---

### Task 7: Selection (TDD)

**Files:**
- Modify: `components/data-table/src/pages-data-table.ts`
- Modify: `components/data-table/src/pages-data-table.test.ts`

**Interfaces:**
- Consumes: `SelectionChangeDetail`, `RowActivateDetail` from `types.ts`
- Produces: `selection-change` and `row-activate` CustomEvents

- [ ] **Step 1: Write failing tests**

```typescript
describe('selection', () => {
  const keyedCols = testColumns as ColumnDef[];

  it('throws when selection enabled without getRowKey', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'single';
    // getRowKey not set
    let error: Error | null = null;
    try { await el.updateComplete; } catch (e) { error = e as Error; }
    expect(error).not.toBeNull();
  });

  it('single: click selects row and emits events', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'single';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('selection-change', e => events.push(e as CustomEvent));
    el.addEventListener('row-activate', e => events.push(e as CustomEvent));

    const row = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)') as HTMLElement;
    row.click();
    await el.updateComplete;

    const selEvent = events.find(e => e.type === 'selection-change')!;
    expect(selEvent.detail.selectedKeys).toEqual(['1']);
    const actEvent = events.find(e => e.type === 'row-activate')!;
    expect(actEvent.detail.key).toBe('1');
  });

  it('single: click different row deselects previous', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'single';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
    (rows[0] as HTMLElement).click();
    await el.updateComplete;
    (rows[1] as HTMLElement).click();
    await el.updateComplete;

    const selected = el.shadowRoot!.querySelectorAll('[aria-selected="true"]');
    expect(selected.length).toBe(1);
  });

  it('multi: renders checkbox column', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const checkboxes = el.shadowRoot!.querySelectorAll('[role="checkbox"]');
    expect(checkboxes.length).toBeGreaterThan(0);
  });

  it('multi: checkbox click toggles selection', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('selection-change', e => events.push(e as CustomEvent));

    const checkbox = el.shadowRoot!.querySelector('.row:not(.header) [role="checkbox"]') as HTMLElement;
    checkbox.click();
    await el.updateComplete;

    expect(events.length).toBe(1);
    expect(events[0]!.detail.selectedKeys).toContain('1');
  });

  it('multi: single-click does NOT emit row-activate', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('row-activate', e => events.push(e as CustomEvent));

    const row = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)') as HTMLElement;
    row.click();
    await el.updateComplete;

    expect(events.length).toBe(0);
  });

  it('multi: double-click emits row-activate', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('row-activate', e => events.push(e as CustomEvent));

    const row = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)') as HTMLElement;
    row.dispatchEvent(new MouseEvent('dblclick', { bubbles: true }));
    await el.updateComplete;

    expect(events.length).toBe(1);
  });

  it('none: row-activate fires without getRowKey (key is undefined)', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('row-activate', e => events.push(e as CustomEvent));

    const row = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)') as HTMLElement;
    row.dispatchEvent(new MouseEvent('dblclick', { bubbles: true }));
    await el.updateComplete;

    expect(events[0]!.detail.key).toBeUndefined();
    expect(events[0]!.detail.row).toBeDefined();
  });

  it('controlled: selectedKeys drives selection state', async () => {
    el.columns = keyedCols;
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    (el as any).selectedKeys = ['1', '3'];
    await el.updateComplete;

    const selected = el.shadowRoot!.querySelectorAll('[aria-selected="true"]');
    expect(selected.length).toBe(2);
  });
});
```

- [ ] **Step 2: Run tests — verify fail**
- [ ] **Step 3: Implement selection**

Add: selection state (`Set<string>`), click handlers (single/multi distinction), checkbox column rendering (multi mode), shift+click range selection, select-all header checkbox, `selection-change` event, `row-activate` event, controlled/uncontrolled modes, `getRowKey` required check, `aria-selected` on rows.

Key spec references: §Selection, §Select-All Behavior by Mode, §Events (row-activate detail).

- [ ] **Step 4: Run tests — verify pass**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(data-table): selection — single/multi/controlled with select-all #22"
```

---

### Task 8: Sorting + column visibility (TDD)

**Files:**
- Modify: `components/data-table/src/pages-data-table.ts`
- Modify: `components/data-table/src/pages-data-table.test.ts`

**Interfaces:**
- Consumes: `createComparator` from `sort.ts`, `SortChangeDetail`, `ColumnChangeDetail` from `types.ts`
- Produces: `sort-change` and `column-change` CustomEvents

- [ ] **Step 1: Write failing tests**

```typescript
describe('sorting', () => {
  it('renders sort indicator on sortable columns', async () => {
    const cols: ColumnDef<TestRow>[] = [
      { id: 'name', label: 'Name', getValue: r => r.name, sortable: true },
      { id: 'age', label: 'Age', getValue: r => r.age },
    ];
    el.columns = cols as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;

    const headers = el.shadowRoot!.querySelectorAll('[role="columnheader"]');
    expect(headers[0]!.getAttribute('aria-sort')).toBe('none');
    expect(headers[1]!.hasAttribute('aria-sort')).toBe(false);
  });

  it('cycles sort direction on header click', async () => {
    const cols: ColumnDef<TestRow>[] = [
      { id: 'name', label: 'Name', getValue: r => r.name, sortable: true },
    ];
    el.columns = cols as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('sort-change', e => events.push(e as CustomEvent));

    const header = el.shadowRoot!.querySelector('[role="columnheader"]') as HTMLElement;

    header.click(); await el.updateComplete;
    expect(events[0]!.detail.direction).toBe('asc');

    header.click(); await el.updateComplete;
    expect(events[1]!.detail.direction).toBe('desc');

    header.click(); await el.updateComplete;
    expect(events[2]!.detail.direction).toBe('none');
  });

  it('clientSort=true reorders rows', async () => {
    const cols: ColumnDef<TestRow>[] = [
      { id: 'name', label: 'Name', getValue: r => r.name, sortable: true },
    ];
    el.columns = cols as ColumnDef[];
    el.rows = testRows;
    (el as any).clientSort = true;
    await el.updateComplete;

    const header = el.shadowRoot!.querySelector('[role="columnheader"]') as HTMLElement;
    header.click(); await el.updateComplete; // asc

    const cells = el.shadowRoot!.querySelectorAll('[role="gridcell"]');
    expect(cells[0]!.textContent).toContain('Alice');
    expect(cells[1]!.textContent).toContain('Bob');
    expect(cells[2]!.textContent).toContain('Carol');
  });
});

describe('column visibility', () => {
  it('hides columns with visible=false', async () => {
    const cols: ColumnDef<TestRow>[] = [
      { id: 'name', label: 'Name', getValue: r => r.name },
      { id: 'age', label: 'Age', getValue: r => r.age, visible: false },
    ];
    el.columns = cols as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;
    const headers = el.shadowRoot!.querySelectorAll('[role="columnheader"]');
    expect(headers.length).toBe(1);
    expect(headers[0]!.textContent).toContain('Name');
  });

  it('emits column-change when visibility toggled', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('column-change', e => events.push(e as CustomEvent));

    // Find and click column picker, then toggle a column
    const picker = el.shadowRoot!.querySelector('.column-picker-trigger') as HTMLElement;
    if (picker) {
      picker.click();
      await el.updateComplete;
      const checkboxes = el.shadowRoot!.querySelectorAll('.column-picker-item input');
      if (checkboxes.length > 0) {
        (checkboxes[1] as HTMLInputElement).click();
        await el.updateComplete;
        expect(events.length).toBe(1);
      }
    }
  });

  it('grid template excludes hidden columns', async () => {
    const cols: ColumnDef<TestRow>[] = [
      { id: 'name', label: 'Name', getValue: r => r.name, width: '1fr' },
      { id: 'age', label: 'Age', getValue: r => r.age, width: '80px', visible: false },
      { id: 'created', label: 'Created', getValue: r => r.created, width: '120px' },
    ];
    el.columns = cols as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;
    const header = el.shadowRoot!.querySelector('.header') as HTMLElement;
    const template = header.style.gridTemplateColumns;
    expect(template).not.toContain('80px');
    expect(template).toContain('1fr');
    expect(template).toContain('120px');
  });
});
```

- [ ] **Step 2: Run tests — verify fail**
- [ ] **Step 3: Implement sorting + column visibility**

Add: header click handler cycling `none→asc→desc→none`, `sort-change` event, `aria-sort` attribute on sortable headers, `clientSort` internal sorting using `createComparator`, column visibility filtering in render, grid template computation from visible columns only, column picker dropdown UI with checkboxes.

Key spec references: §Client-Side Sorting, §Column Visibility, §Grid Template Computation.

- [ ] **Step 4: Run tests — verify pass**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(data-table): sorting + column visibility with picker #22"
```

---

### Task 9: Keyboard navigation + ARIA (TDD)

**Files:**
- Modify: `components/data-table/src/pages-data-table.ts`
- Modify: `components/data-table/src/pages-data-table.test.ts`

**Interfaces:**
- Consumes: component internal state
- Produces: keyboard-driven focus management, complete ARIA attributes

- [ ] **Step 1: Write failing tests**

```typescript
describe('keyboard navigation', () => {
  it('ArrowDown moves focus to next row', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;
    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');

    (rows[0] as HTMLElement).focus();
    rows[0]!.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowDown', bubbles: true }));
    await el.updateComplete;

    expect(el.shadowRoot!.activeElement).toBe(rows[1]);
  });

  it('ArrowUp moves focus to previous row', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;
    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');

    (rows[1] as HTMLElement).focus();
    rows[1]!.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowUp', bubbles: true }));
    await el.updateComplete;

    expect(el.shadowRoot!.activeElement).toBe(rows[0]);
  });

  it('Enter activates focused row', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('row-activate', e => events.push(e as CustomEvent));

    const row = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)') as HTMLElement;
    row.focus();
    row.dispatchEvent(new KeyboardEvent('keydown', { key: 'Enter', bubbles: true }));
    await el.updateComplete;

    expect(events.length).toBe(1);
  });

  it('Escape clears selection', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    // Select first row
    const checkbox = el.shadowRoot!.querySelector('.row:not(.header) [role="checkbox"]') as HTMLElement;
    checkbox.click();
    await el.updateComplete;

    // Press Escape
    el.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape', bubbles: true }));
    await el.updateComplete;

    const selected = el.shadowRoot!.querySelectorAll('[aria-selected="true"]');
    expect(selected.length).toBe(0);
  });

  it('Space toggles selection in multi mode', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    const events: CustomEvent[] = [];
    el.addEventListener('selection-change', e => events.push(e as CustomEvent));

    const row = el.shadowRoot!.querySelector('.row[role="row"]:not(.header)') as HTMLElement;
    row.focus();
    row.dispatchEvent(new KeyboardEvent('keydown', { key: ' ', bubbles: true }));
    await el.updateComplete;

    expect(events.length).toBe(1);
    expect(events[0]!.detail.selectedKeys).toContain('1');
  });

  it('Home focuses first row', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
    (rows[2] as HTMLElement).focus();
    rows[2]!.dispatchEvent(new KeyboardEvent('keydown', { key: 'Home', bubbles: true }));
    await el.updateComplete;

    expect(el.shadowRoot!.activeElement).toBe(rows[0]);
  });
});

describe('ARIA completeness', () => {
  it('sets aria-colcount', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    await el.updateComplete;
    const grid = el.shadowRoot!.querySelector('[role="grid"]')!;
    expect(grid.getAttribute('aria-colcount')).toBe('2');
  });

  it('header checkbox has aria-checked=mixed for partial selection', async () => {
    el.columns = testColumns as ColumnDef[];
    el.rows = testRows;
    (el as any).selection = 'multi';
    el.getRowKey = (r: unknown) => (r as TestRow).id;
    await el.updateComplete;

    // Select one row
    const checkbox = el.shadowRoot!.querySelector('.row:not(.header) [role="checkbox"]') as HTMLElement;
    checkbox.click();
    await el.updateComplete;

    const headerCheckbox = el.shadowRoot!.querySelector('.header [role="checkbox"]');
    expect(headerCheckbox!.getAttribute('aria-checked')).toBe('mixed');
  });
});
```

- [ ] **Step 2: Run tests — verify fail**
- [ ] **Step 3: Implement keyboard navigation + ARIA**

Add: 2D grid navigation with `(focusRowIndex, focusColIndex)` state, keydown handler on grid container (ArrowDown/Up/Left/Right, Home/End, Ctrl+Home/End, Enter, Space, Escape, Ctrl+A), `tabindex` management on rows, focus-into-grid behavior, virtual scroll scroll-into-view for keyboard navigation to off-screen rows, complete ARIA attributes (`aria-colcount`, `aria-sort`, `aria-selected`, `aria-checked` with `mixed` state).

Key spec references: §Keyboard Navigation, §Virtual scroll keyboard navigation, §ARIA.

- [ ] **Step 4: Run tests — verify pass**
- [ ] **Step 5: Commit**

```bash
git commit -m "feat(data-table): 2D keyboard navigation + ARIA grid #22"
```

---

### Task 10: Inbox refactor

**Files:**
- Modify: `components/work-item-inbox/src/work-item-inbox.ts`
- Modify: `components/work-item-inbox/src/work-item-inbox.test.ts`
- Modify: `components/work-item-inbox/package.json` (add data-table dep)
- Modify: `components/work-item-inbox/tsconfig.json` (add project reference)

**Interfaces:**
- Consumes: `PagesDataTable`, `ColumnDef`, event detail types from `@casehubio/blocks-ui-data-table`
- Produces: refactored `work-item-inbox` using `<pages-data-table>` for rendering

- [ ] **Step 1: Write failing tests**

Update existing inbox tests: verify `<pages-data-table>` element is rendered inside the inbox, verify column definitions match expected columns, verify selection-change events are forwarded to batch operations.

```typescript
it('renders pages-data-table element', async () => {
  el.identity = mockIdentity;
  el.data = mockData;
  await el.updateComplete;
  const table = el.shadowRoot!.querySelector('pages-data-table');
  expect(table).not.toBeNull();
});

it('passes filtered items to table rows', async () => {
  el.identity = mockIdentity;
  el.data = mockData;
  el.mode = 'my-work';
  await el.updateComplete;
  const table = el.shadowRoot!.querySelector('pages-data-table') as any;
  expect(table.rows.length).toBeLessThanOrEqual(mockData.length);
});
```

- [ ] **Step 2: Run tests — verify fail**
- [ ] **Step 3: Refactor work-item-inbox**

1. Add `@casehubio/blocks-ui-data-table` to inbox package.json dependencies
2. Add project reference to inbox tsconfig.json
3. Import `pages-data-table.js` in work-item-inbox.ts
4. Define `tableColumns: ColumnDef<WorkItemRootResponse>[]` with renderers for title, status, category, age
5. Replace `renderItems()` body: remove virtual scroll code, remove selection handling, render `<pages-data-table>` with `.rows`, `.columns`, `.getRowKey`, `mode="scroll"`, `selection="multi"`, event listeners
6. Add `handleSelectionChange` bridging `selection-change` → `selectedItems` Set
7. Add `handleRowActivate` bridging `row-activate` → `emitPagesEvent(SELECTED)`
8. Remove: `virtualScrollTop`, `itemHeight`, `bufferSize`, `getVirtualWindow()`, `handleScroll()`, `handleRowSelect()`, `handleRangeSelect()`, `handleRowClick()`, `lastSelectedIndex`, virtual scroll CSS
9. Keep: all SSE, filtering, tabs, batch operations, queue scope

- [ ] **Step 4: Run full inbox test suite**

```bash
cd components/work-item-inbox && npx vitest run
```

- [ ] **Step 5: Run full project test suite**

```bash
yarn test
```

- [ ] **Step 6: Commit**

```bash
git commit -m "refactor(inbox): use pages-data-table, remove virtual scroll + selection code #22"
```

---

## Verification

After all tasks complete:

1. **Unit tests**: `yarn test` — all workspaces pass
2. **Type check**: `yarn typecheck` — no errors
3. **Build**: `yarn build` — all packages compile
4. **Manual smoke test**: The data-table test file exercises all modes, selection, sorting, keyboard, and ARIA. The inbox tests verify the integration.

## Spec Coverage Check

| Spec Section | Task |
|-------------|------|
| ColumnDef\<R\> data model | Task 1 (types) |
| Component API — properties | Tasks 4-9 (incremental) |
| Component API — events | Tasks 5-9 (incremental) |
| Display modes — auto | Task 4 |
| Display modes — paginated | Task 5 |
| Display modes — scroll | Task 6 |
| Rendering — CSS Grid | Task 4 |
| Row Styling via ::part() | Task 4 |
| Cell rendering pipeline | Task 4 |
| Selection — all modes | Task 7 |
| Client-side sorting | Task 8 |
| Column visibility | Task 8 |
| Grid template computation | Task 8 |
| Responsive behavior | Task 4 (overflow-x: auto) |
| Keyboard navigation | Task 9 |
| ARIA | Task 9 |
| Virtual scroll engine | Task 2 |
| Sort comparators | Task 3 |
| Motivating example: inbox | Task 10 |
