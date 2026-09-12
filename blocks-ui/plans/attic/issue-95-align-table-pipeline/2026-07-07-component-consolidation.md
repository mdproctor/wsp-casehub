# Component Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #29 — pages-data-table: text filter support
**Issue group:** #29, #24, #25, #18, #9, #10, #11

**Goal:** Implement 7 features: DataEndpointMixin infrastructure, data-table text filter, KPI density + reactive endpoint, work-item relations, and 3 new components (audit-trail-viewer, case-timeline, trust-score-panel).

**Architecture:** DataEndpointMixin provides shared endpoint/fetch/SSE lifecycle for new components. Existing components keep their current patterns. All new components follow TDD, use `--pages-*` tokens, emit `pages-event` for cross-panel communication.

**Tech Stack:** TypeScript, Lit 3, Web Components, Vitest, CSS Grid, inline SVG

## Global Constraints

- All CSS custom properties use `--pages-*` prefix (migrated from `--blocks-*`)
- All components use `@casehubio/blocks-ui-core` mixins (LiveRegionMixin, KeyboardShortcutMixin)
- Events use `emitPagesEvent(target, topic, payload)` from `@casehubio/blocks-ui-core`
- Test environment: jsdom via vitest
- No external chart libraries in blocks-ui — use `pages-viz` for charts, inline SVG for gauges/sparklines
- Each task commits independently with issue reference
- Spec: `docs/specs/2026-07-07-component-consolidation-design.md` — consult for design decisions

---

### Task 1: DataEndpointMixin

**Files:**
- Create: `packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts`
- Create: `packages/blocks-ui-core/src/data-endpoint/index.ts`
- Create: `packages/blocks-ui-core/src/data-endpoint/data-endpoint.test.ts`
- Modify: `packages/blocks-ui-core/src/index.ts` (add export)

**Interfaces:**
- Consumes: `SSEManager` from `@casehubio/pages-data`, `WorkIdentity` from `../types/work-item.js`
- Produces: `DataEndpointMixin` function, used by Tasks 6, 7, 8

- [ ] **Step 1: Write the failing test**

```typescript
// data-endpoint.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { LitElement, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import { DataEndpointMixin } from './data-endpoint.js';

@customElement('test-data-endpoint')
class TestComponent extends DataEndpointMixin(LitElement) {
  @property() subjectId?: string;
  fetchCount = 0;

  async fetchData(): Promise<void> {
    this.fetchCount++;
    const res = await this.fetchFn(`${this.endpoint}/items`);
    if (!res.ok) throw new Error('fetch failed');
  }

  override render() {
    if (this.loading) return html`<div class="loading">Loading</div>`;
    if (this.error) return html`<div class="error">${this.error}<button @click=${() => this.fetchData()}>Retry</button></div>`;
    return html`<div class="content">OK</div>`;
  }
}

describe('DataEndpointMixin', () => {
  let el: TestComponent;
  let mockFetch: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    mockFetch = vi.fn().mockResolvedValue({ ok: true, json: async () => ({}) });
  });
  afterEach(() => el?.remove());

  it('does not fetch without endpoint', async () => {
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    expect(el.fetchCount).toBe(0);
  });

  it('fetches when endpoint is set via configure()', async () => {
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.configure({ endpoint: 'http://localhost:8080' });
    await new Promise(r => setTimeout(r, 10));
    expect(el.fetchCount).toBe(1);
  });

  it('fetches exactly once via configure() — no double-fetch from willUpdate', async () => {
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.configure({ endpoint: 'http://localhost:8080' });
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 50));
    expect(el.fetchCount).toBe(1);
  });

  it('re-fetches when endpoint changes via property', async () => {
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.endpoint = 'http://localhost:8080';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    expect(el.fetchCount).toBe(1);
    el.endpoint = 'http://localhost:9090';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    expect(el.fetchCount).toBe(2);
  });

  it('sets loading during fetch', async () => {
    let resolvePromise: () => void;
    mockFetch.mockReturnValue(new Promise(r => { resolvePromise = () => r({ ok: true }); }));
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.endpoint = 'http://localhost:8080';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    expect(el.loading).toBe(true);
    resolvePromise!();
    await new Promise(r => setTimeout(r, 10));
    expect(el.loading).toBe(false);
  });

  it('sets error on fetch failure', async () => {
    mockFetch.mockRejectedValue(new Error('network down'));
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.endpoint = 'http://localhost:8080';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    expect(el.error).toContain('network down');
  });

  it('aborts in-flight fetch when endpoint changes', async () => {
    let abortSignals: AbortSignal[] = [];
    mockFetch.mockImplementation((_url: string, init?: RequestInit) => {
      if (init?.signal) abortSignals.push(init.signal);
      return new Promise(() => {});
    });
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.endpoint = 'http://localhost:8080';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    el.endpoint = 'http://localhost:9090';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 10));
    expect(abortSignals[0]?.aborted).toBe(true);
  });

  it('configure() sets identity', async () => {
    el = document.createElement('test-data-endpoint') as TestComponent;
    el.fetchFn = mockFetch;
    document.body.appendChild(el);
    el.configure({ endpoint: 'http://x', identity: { userId: 'u1', displayName: 'User', groups: [] } });
    expect(el.identity?.userId).toBe('u1');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy npx vitest run --config packages/blocks-ui-core/vitest.config.ts --root packages/blocks-ui-core src/data-endpoint/data-endpoint.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement DataEndpointMixin**

Create `packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts`:

```typescript
import { type LitElement, type PropertyValues } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { WorkIdentity } from '../types/work-item.js';
import type { SSEManager, SSEEvent } from '@casehubio/pages-data/dist/sse/sse-manager.js';
import { SSEManager as SSEManagerImpl } from '@casehubio/pages-data/dist/sse/sse-manager.js';

type Constructor<T = {}> = new (...args: any[]) => T;

export function DataEndpointMixin<T extends Constructor<LitElement>>(Base: T) {
  abstract class DataEndpointHost extends Base {
    @property({ type: String }) endpoint?: string;
    @property({ type: Object }) identity?: WorkIdentity;
    @state() loading = false;
    @state() error: string | null = null;

    fetchFn: typeof fetch = fetch;
    sseManager: SSEManager = new SSEManagerImpl();

    private _abortController: AbortController | null = null;
    private _configurePending = false;
    private _sseUrl: string | null = null;
    private _sseHandler: ((event: SSEEvent) => void) | null = null;

    abstract fetchData(): Promise<void>;
    sseUrl?(): string;
    handleSSEEvent?(event: SSEEvent): void;

    get abortSignal(): AbortSignal | undefined {
      return this._abortController?.signal;
    }

    configure(props: Record<string, unknown>): void {
      if (props.endpoint !== undefined) this.endpoint = props.endpoint as string;
      if (props.identity !== undefined) this.identity = props.identity as WorkIdentity;
      this._configurePending = true;
      queueMicrotask(() => {
        this._configurePending = false;
        this._doFetch();
        this._resubscribeSSE();
      });
    }

    override willUpdate(changed: PropertyValues): void {
      super.willUpdate(changed);
      if (!this._configurePending && changed.has('endpoint') && this.endpoint) {
        this._doFetch();
        this._resubscribeSSE();
      }
    }

    override disconnectedCallback(): void {
      super.disconnectedCallback();
      this._abortController?.abort();
      this._abortController = null;
      this._unsubscribeSSE();
    }

    private async _doFetch(): Promise<void> {
      if (!this.endpoint) return;
      this._abortController?.abort();
      this._abortController = new AbortController();
      this.loading = true;
      this.error = null;
      try {
        await this.fetchData();
      } catch (e) {
        if (e instanceof DOMException && e.name === 'AbortError') return;
        this.error = e instanceof Error ? e.message : String(e);
      } finally {
        this.loading = false;
      }
    }

    private _resubscribeSSE(): void {
      this._unsubscribeSSE();
      const url = this.sseUrl?.();
      if (!url || !this.handleSSEEvent) return;
      this._sseUrl = url;
      this._sseHandler = (event: SSEEvent) => this.handleSSEEvent!(event);
      this.sseManager.subscribe(url, this._sseHandler);
    }

    private _unsubscribeSSE(): void {
      if (this._sseUrl && this._sseHandler) {
        this.sseManager.unsubscribe(this._sseUrl, this._sseHandler);
        this._sseUrl = null;
        this._sseHandler = null;
      }
    }
  }

  return DataEndpointHost as unknown as Constructor<{
    endpoint?: string;
    identity?: WorkIdentity;
    loading: boolean;
    error: string | null;
    fetchFn: typeof fetch;
    sseManager: SSEManager;
    abortSignal: AbortSignal | undefined;
    fetchData(): Promise<void>;
    configure(props: Record<string, unknown>): void;
    sseUrl?(): string;
    handleSSEEvent?(event: SSEEvent): void;
  }> & T;
}
```

Create `packages/blocks-ui-core/src/data-endpoint/index.ts`:
```typescript
export { DataEndpointMixin } from './data-endpoint.js';
```

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './data-endpoint/index.js';
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `GH_PACKAGES_TOKEN=dummy npx vitest run --config packages/blocks-ui-core/vitest.config.ts --root packages/blocks-ui-core`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```
git add packages/blocks-ui-core/src/data-endpoint/ packages/blocks-ui-core/src/index.ts
git commit -m "feat(core): DataEndpointMixin — shared endpoint/fetch/SSE lifecycle #29"
```

---

### Task 2: KPI Reactive Endpoint (#25)

**Files:**
- Modify: `components/kpi-metric-row/src/kpi-metric-row.ts`
- Modify: `components/kpi-metric-row/src/kpi-metric-row.test.ts` (add test)

**Interfaces:**
- Consumes: nothing new
- Produces: reactive `endpoint` property on `kpi-metric-row`

- [ ] **Step 1: Write the failing test**

Add to `kpi-metric-row.test.ts`:
```typescript
it('re-fetches when endpoint changes after mount', async () => {
  const mockFetch = vi.fn().mockResolvedValue({
    ok: true,
    json: async () => [{ key: 'k1', value: 1, label: 'L1' }],
  });
  window.fetch = mockFetch as typeof fetch;

  const el = document.createElement('kpi-metric-row') as any;
  el.endpoint = '/api/v1/metrics';
  document.body.appendChild(el);
  await el.updateComplete;
  await new Promise(r => setTimeout(r, 50));
  const firstCount = mockFetch.mock.calls.length;

  el.endpoint = '/api/v2/metrics';
  await el.updateComplete;
  await new Promise(r => setTimeout(r, 50));

  expect(mockFetch.mock.calls.length).toBeGreaterThan(firstCount);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN=dummy npx vitest run --config components/kpi-metric-row/vitest.config.ts --root components/kpi-metric-row`
Expected: FAIL — endpoint change doesn't trigger re-fetch

- [ ] **Step 3: Implement willUpdate reactivity**

In `kpi-metric-row.ts`, add `willUpdate` and remove the fetch from `connectedCallback`:

```typescript
override willUpdate(changed: PropertyValues): void {
  if (changed.has('endpoint') && this.endpoint) {
    this._fetchMetrics();
  }
}
```

Remove the `if (this.endpoint) this._fetchMetrics();` from `connectedCallback`.

- [ ] **Step 4: Run tests**

Run: `GH_PACKAGES_TOKEN=dummy npx vitest run --config components/kpi-metric-row/vitest.config.ts --root components/kpi-metric-row`
Expected: All PASS

- [ ] **Step 5: Commit**

```
git commit -m "fix(kpi-metric-row): reactive endpoint — re-fetch on property change #25"
```

---

### Task 3: KPI Density Property (#24)

**Files:**
- Modify: `components/kpi-metric-row/src/kpi-metric-row.ts`
- Modify: `components/kpi-metric-row/src/kpi-metric-row.test.ts`

**Interfaces:**
- Consumes: nothing new
- Produces: `density` property on `kpi-metric-row`

- [ ] **Step 1: Write the failing tests**

```typescript
it('reflects density attribute to the host element', async () => {
  const el = document.createElement('kpi-metric-row') as any;
  el.metrics = [{ key: 'k1', value: 42, label: 'Test' }];
  el.density = 'compact';
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.getAttribute('density')).toBe('compact');
  el.remove();
});

it('defaults to comfortable density', async () => {
  const el = document.createElement('kpi-metric-row') as any;
  el.metrics = [{ key: 'k1', value: 42, label: 'Test' }];
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.density).toBe('comfortable');
  el.remove();
});

it('uses 120px minmax in compact density', async () => {
  const el = document.createElement('kpi-metric-row') as any;
  el.metrics = [{ key: 'k1', value: 42, label: 'Test' }];
  el.density = 'compact';
  document.body.appendChild(el);
  await el.updateComplete;
  const grid = el.shadowRoot!.querySelector('.grid') as HTMLElement;
  expect(grid.style.gridTemplateColumns).toContain('120px');
  el.remove();
});
```

- [ ] **Step 2: Run to verify failure**

- [ ] **Step 3: Implement density property and CSS**

Add property to `kpi-metric-row.ts`:
```typescript
@property({ type: String, reflect: true }) density: 'comfortable' | 'compact' | 'dense' = 'comfortable';
```

Update the grid template in `render()` to use density-dependent minmax:
```typescript
private get _gridMinmax(): string {
  switch (this.density) {
    case 'dense': return '90px';
    case 'compact': return '120px';
    default: return '160px';
  }
}
```

Add CSS host selectors:
```css
:host([density="compact"]) .card { padding: var(--pages-space-3, 12px); }
:host([density="compact"]) .value { font-size: var(--pages-font-size-xl, 20px); }
:host([density="dense"]) .card { padding: var(--pages-space-2, 8px); }
:host([density="dense"]) .value { font-size: var(--pages-font-size-lg, 16px); }
```

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

```
git commit -m "feat(kpi-metric-row): density property — comfortable/compact/dense grid layouts #24"
```

---

### Task 4: Data-Table Text Filter (#29)

**Files:**
- Modify: `components/data-table/src/types.ts` (add `filterable`, `filterValue`, `FilterChangeDetail`)
- Modify: `components/data-table/src/pages-data-table.ts` (add filter properties, UI, pipeline)
- Modify: `components/data-table/src/pages-data-table.test.ts` (add filter tests)

**Interfaces:**
- Consumes: `ColumnDef` (existing)
- Produces: `clientFilter`, `filterText` properties; `filter-change` event; `FilterChangeDetail` type

- [ ] **Step 1: Write the failing tests**

```typescript
describe('client filter', () => {
  it('filters rows by text match across columns', async () => {
    const el = createTable([
      { name: 'Alice', role: 'Engineer' },
      { name: 'Bob', role: 'Designer' },
      { name: 'Charlie', role: 'Engineer' },
    ], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
      { id: 'role', label: 'Role', getValue: (r: any) => r.role },
    ]);
    el.clientFilter = true;
    el.filterText = 'engineer';
    document.body.appendChild(el);
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('[role="row"]:not(.header-row)');
    expect(rows.length).toBe(2); // Alice and Charlie
  });

  it('matches any column — row appears if ANY column matches', async () => {
    const el = createTable([
      { name: 'Alice', role: 'Engineer' },
      { name: 'Bob', role: 'Designer' },
    ], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
      { id: 'role', label: 'Role', getValue: (r: any) => r.role },
    ]);
    el.clientFilter = true;
    el.filterText = 'bob';
    document.body.appendChild(el);
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('[role="row"]:not(.header-row)');
    expect(rows.length).toBe(1);
  });

  it('respects filterable: false on columns', async () => {
    const el = createTable([
      { name: 'Alice', code: '123' },
      { name: 'Bob', code: '456' },
    ], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
      { id: 'code', label: 'Code', getValue: (r: any) => r.code, filterable: false },
    ]);
    el.clientFilter = true;
    el.filterText = '123';
    document.body.appendChild(el);
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('[role="row"]:not(.header-row)');
    expect(rows.length).toBe(0); // code column not filterable
  });

  it('uses filterValue when provided', async () => {
    const el = createTable([
      { name: 'Alice', tags: ['eng', 'lead'] },
      { name: 'Bob', tags: ['design'] },
    ], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
      { id: 'tags', label: 'Tags', getValue: (r: any) => r.tags, filterValue: (r: any) => r.tags.join(' ') },
    ]);
    el.clientFilter = true;
    el.filterText = 'lead';
    document.body.appendChild(el);
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('[role="row"]:not(.header-row)');
    expect(rows.length).toBe(1);
  });

  it('resets currentPage to 0 when filter changes', async () => {
    const el = createTable(
      Array.from({ length: 100 }, (_, i) => ({ name: `Item ${i}` })),
      [{ id: 'name', label: 'Name', getValue: (r: any) => r.name }],
    );
    el.mode = 'paginated';
    el.pageSize = 10;
    el.currentPage = 5;
    el.clientFilter = true;
    document.body.appendChild(el);
    await el.updateComplete;

    el.filterText = 'Item 1';
    await el.updateComplete;
    expect(el.currentPage).toBe(0);
  });

  it('is ignored when totalRows is set (server pagination)', async () => {
    const el = createTable([
      { name: 'Alice' },
      { name: 'Bob' },
    ], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
    ]);
    el.clientFilter = true;
    el.filterText = 'alice';
    el.totalRows = 100;
    document.body.appendChild(el);
    await el.updateComplete;

    const rows = el.shadowRoot!.querySelectorAll('[role="row"]:not(.header-row)');
    expect(rows.length).toBe(2); // no filtering — server-paginated
  });

  it('emits filter-change event', async () => {
    const el = createTable([
      { name: 'Alice' },
      { name: 'Bob' },
    ], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
    ]);
    el.clientFilter = true;
    document.body.appendChild(el);
    await el.updateComplete;

    const events: any[] = [];
    el.addEventListener('filter-change', (e: any) => events.push(e.detail));
    el.filterText = 'alice';
    await el.updateComplete;
    await new Promise(r => setTimeout(r, 200)); // debounce

    expect(events.length).toBeGreaterThan(0);
    expect(events[0].text).toBe('alice');
    expect(events[0].matchCount).toBe(1);
  });

  it('renders filter input when clientFilter is true', async () => {
    const el = createTable([{ name: 'A' }], [
      { id: 'name', label: 'Name', getValue: (r: any) => r.name },
    ]);
    el.clientFilter = true;
    document.body.appendChild(el);
    await el.updateComplete;

    const input = el.shadowRoot!.querySelector('.filter-input');
    expect(input).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run to verify failure**

- [ ] **Step 3: Add types**

In `types.ts`, add to `ColumnDef`:
```typescript
readonly filterable?: boolean;
readonly filterValue?: (row: R) => string;
```

Add `FilterChangeDetail`:
```typescript
export interface FilterChangeDetail {
  readonly text: string;
  readonly matchCount: number;
}
```

- [ ] **Step 4: Implement filter in pages-data-table.ts**

Add properties:
```typescript
@property({ type: Boolean, attribute: 'client-filter' }) clientFilter = false;
@property({ type: String, attribute: 'filter-text' }) filterText = '';
```

Add filter step to `_visibleRows` getter (BEFORE sort):
```typescript
private get _visibleRows(): readonly unknown[] {
  let rows = this.rows;

  // Apply client-side filtering
  if (this.clientFilter && this.filterText && this.totalRows === undefined) {
    const text = this.filterText.toLowerCase();
    rows = [...rows].filter(row =>
      this._visibleColumns.some(col => {
        if (col.filterable === false) return false;
        const val = col.filterValue
          ? col.filterValue(row)
          : String(col.getValue(row));
        return val.toLowerCase().includes(text);
      })
    );
  }

  // Apply client-side sorting (existing)
  if (this.clientSort && this.sortColumnId && this.sortDirection !== 'none') {
    // ... existing sort code
  }
  // ... existing pagination code
}
```

Reset page on filter change (in `willUpdate`):
```typescript
if (changed.has('filterText') && this.clientFilter) {
  this.currentPage = 0;
  this._emitFilterChange();
}
```

Add filter input to toolbar (alongside column picker).

Add debounced event emission.

- [ ] **Step 5: Run tests**

- [ ] **Step 6: Commit**

```
git commit -m "feat(data-table): client-side text filter with filterable/filterValue on ColumnDef #29"
```

---

### Task 5: Work-Item-Detail Relations (#18)

**Files:**
- Modify: `components/work-item-detail/src/detail-relations-tab.ts` (new type, new rendering)
- Modify: `components/work-item-detail/src/work-item-detail.ts` (fetch relations)
- Modify: `components/work-item-detail/src/work-item-detail.test.ts` (add relation tests)

**Interfaces:**
- Consumes: `GET /workitems/{id}/relations`, `GET /workitems/{id}/relations/incoming`
- Produces: `WorkItemRelation` type, populated relations tab

- [ ] **Step 1: Write the failing test**

```typescript
it('fetches and displays relations when work item loads', async () => {
  mockFetch.mockImplementation((url: string) => {
    if (url.includes('/relations/incoming'))
      return Promise.resolve({ ok: true, json: async () => [
        { id: 'r2', sourceId: 'wi-other', targetId: 'wi-1', relationType: 'BLOCKS', createdBy: 'user1', createdAt: '2026-07-07T00:00:00Z' }
      ]});
    if (url.includes('/relations'))
      return Promise.resolve({ ok: true, json: async () => [
        { id: 'r1', sourceId: 'wi-1', targetId: 'wi-2', relationType: 'RELATES_TO', createdBy: 'user1', createdAt: '2026-07-07T00:00:00Z' }
      ]});
    if (url.includes('/workitems/wi-2'))
      return Promise.resolve({ ok: true, json: async () => ({ item: { id: 'wi-2', title: 'Related Item', status: 'ASSIGNED' } }) });
    if (url.includes('/workitems/wi-other'))
      return Promise.resolve({ ok: true, json: async () => ({ item: { id: 'wi-other', title: 'Blocking Item', status: 'IN_PROGRESS' } }) });
    return Promise.resolve({ ok: true, json: async () => ({ item: { id: 'wi-1', title: 'Test', status: 'PENDING' }, events: [] }) });
  });

  el.endpoint = 'http://localhost';
  el.workItemId = 'wi-1';
  await el.updateComplete;
  await new Promise(r => setTimeout(r, 100));

  // Relations should be fetched and passed to the tab
  expect(mockFetch).toHaveBeenCalledWith(expect.stringContaining('/relations'));
  expect(mockFetch).toHaveBeenCalledWith(expect.stringContaining('/relations/incoming'));
});
```

- [ ] **Step 2: Run to verify failure**

- [ ] **Step 3: Implement**

Add `WorkItemRelation` interface and `RELATION_INVERSES` constant to `detail-relations-tab.ts`.

In `work-item-detail.ts`, add to `_loadWorkItem()`:
```typescript
const [outgoing, incoming] = await Promise.all([
  fetch(`${this.endpoint}/workitems/${this.workItemId}/relations`).then(r => r.json()),
  fetch(`${this.endpoint}/workitems/${this.workItemId}/relations/incoming`).then(r => r.json()),
]);
const relations = [
  ...outgoing.map((r: any) => ({ ...r, direction: 'outgoing' as const })),
  ...incoming.map((r: any) => ({
    ...r, direction: 'incoming' as const,
    relationType: RELATION_INVERSES[r.relationType] ?? `${r.relationType} (incoming)`,
  })),
];
this._relations = relations;
```

Pass to tab: `<detail-relations-tab .relations=${this._relations}>`

Fetch related item titles in batch: `Promise.all(relations.map(...))`

- [ ] **Step 4: Run tests**

- [ ] **Step 5: Commit**

```
git commit -m "feat(work-item-detail): fetch and display work item relations #18"
```

---

### Task 6: Audit Trail Viewer (#9)

**Files:**
- Create: `components/audit-trail-viewer/package.json`
- Create: `components/audit-trail-viewer/tsconfig.json`
- Create: `components/audit-trail-viewer/tsconfig.build.json`
- Create: `components/audit-trail-viewer/vitest.config.ts`
- Create: `components/audit-trail-viewer/src/index.ts`
- Create: `components/audit-trail-viewer/src/types.ts`
- Create: `components/audit-trail-viewer/src/audit-trail-viewer.ts`
- Create: `components/audit-trail-viewer/src/audit-trail-viewer.test.ts`
- Modify: `examples/src/shell.ts` (add nav item)
- Create: `examples/src/pages/audit-trail-page.ts`
- Create: `examples/mock-data/ledger-entries.json`

**Interfaces:**
- Consumes: `DataEndpointMixin` (Task 1), `pages-data-table` (existing), ledger REST API
- Produces: `<audit-trail-viewer>` custom element

**Implementation approach:**
1. Define types (`LedgerEntry`, `Attestation`, `VerificationResult`)
2. Build component extending `DataEndpointMixin(LiveRegionMixin(LitElement))`
3. `fetchData()` calls ledger entries endpoint + verification endpoint
4. Entry list via `pages-data-table` with `clientFilter` enabled
5. Expandable detail via row-activate handler + detail panel
6. Filter controls: actor dropdown, entry type chips, date range
7. Verification banner at top
8. GDPR: null payload → "Content redacted" placeholder
9. Example page with mock data

Each sub-step follows TDD. See spec §9 for full design including ARIA table, filter controls, and GDPR handling.

- [ ] **Step 1: Create package scaffold** — package.json, tsconfig files, vitest.config.ts
- [ ] **Step 2: Write failing tests for types and fetchData**
- [ ] **Step 3: Implement types and core component with DataEndpointMixin**
- [ ] **Step 4: Write failing tests for entry list rendering**
- [ ] **Step 5: Implement entry list with pages-data-table**
- [ ] **Step 6: Write failing tests for expandable detail**
- [ ] **Step 7: Implement expandable detail with attestation display**
- [ ] **Step 8: Write failing tests for verification banner**
- [ ] **Step 9: Implement verification banner**
- [ ] **Step 10: Write failing tests for filter controls**
- [ ] **Step 11: Implement filter controls (actor, type, date range)**
- [ ] **Step 12: Add ARIA and LiveRegionMixin announcements**
- [ ] **Step 13: Create example page with mock data**
- [ ] **Step 14: Run full test suite**
- [ ] **Step 15: Commit**

```
git commit -m "feat(audit-trail-viewer): ledger entry viewer with verification and attestations #9"
```

---

### Task 7: Case Timeline (#10)

**Files:**
- Modify: `components/case-timeline/package.json` (add deps)
- Create: `components/case-timeline/vitest.config.ts`
- Modify: `components/case-timeline/src/index.ts` (replace stub)
- Create: `components/case-timeline/src/types.ts`
- Create: `components/case-timeline/src/case-timeline.ts`
- Create: `components/case-timeline/src/case-timeline.test.ts`
- Modify: `examples/src/shell.ts` (add nav item)
- Create: `examples/src/pages/case-timeline-page.ts`
- Create: `examples/mock-data/case-events.json`

**Interfaces:**
- Consumes: `DataEndpointMixin` (Task 1), scaffold EventLog API
- Produces: `<case-timeline>` custom element with full and compact modes

**Implementation approach:**
1. Define types (`CaseEvent`, `CaseHubEventType`, `EventStreamType`)
2. Build component extending `DataEndpointMixin(LiveRegionMixin(LitElement))`
3. `fetchData()` calls paginated case events endpoint
4. Full mode: vertical CSS timeline with node type mapping (see spec §10 for 6 node categories)
5. Compact mode: horizontal dot strip with truncation (>7 nodes → first 3 + last 2 + ellipsis)
6. Filter bar: stream type chips
7. Events: `timeline.event-selected`, `work-item.selected`, `timeline.expand-requested`
8. Example page with mock event data

Each sub-step follows TDD. See spec §10 for full design including compact mode specs, ARIA, and node type mapping.

- [ ] **Step 1: Update package.json with dependencies, create vitest.config.ts**
- [ ] **Step 2: Write failing tests for types and fetchData**
- [ ] **Step 3: Implement types and core component**
- [ ] **Step 4: Write failing tests for full mode timeline rendering**
- [ ] **Step 5: Implement full mode — vertical CSS timeline with node type mapping**
- [ ] **Step 6: Write failing tests for compact mode**
- [ ] **Step 7: Implement compact mode — horizontal dot strip**
- [ ] **Step 8: Write failing tests for filter bar**
- [ ] **Step 9: Implement stream type filter chips**
- [ ] **Step 10: Write failing tests for events (timeline.event-selected, work-item.selected)**
- [ ] **Step 11: Implement event emission**
- [ ] **Step 12: Add ARIA and LiveRegionMixin announcements**
- [ ] **Step 13: Create example page with mock data**
- [ ] **Step 14: Run full test suite**
- [ ] **Step 15: Commit**

```
git commit -m "feat(case-timeline): case lifecycle visualization with full and compact modes #10"
```

---

### Task 8: Trust Score Panel (#11)

**Files:**
- Modify: `components/trust-score-panel/package.json` (add deps)
- Create: `components/trust-score-panel/vitest.config.ts`
- Modify: `components/trust-score-panel/src/index.ts` (replace stub)
- Create: `components/trust-score-panel/src/types.ts`
- Create: `components/trust-score-panel/src/trust-score-panel.ts`
- Create: `components/trust-score-panel/src/score-gauge.ts` (inline SVG arc)
- Create: `components/trust-score-panel/src/trust-score-panel.test.ts`
- Modify: `examples/src/shell.ts` (add nav item)
- Create: `examples/src/pages/trust-score-page.ts`
- Create: `examples/mock-data/trust-scores.json`

**Interfaces:**
- Consumes: `DataEndpointMixin` (Task 1), `pages-data-table` (existing), `pages-viz` (line chart), ledger trust API
- Produces: `<trust-score-panel>` custom element with full and compact modes

**Implementation approach:**
1. Define types (`TrustScoreResponse`, `CapabilityScoreResponse`, `TrustLevel`, `MaturityPhase`)
2. Build component extending `DataEndpointMixin(LiveRegionMixin(LitElement))`
3. `fetchData()` calls trust score endpoint
4. Full mode section 1: SVG arc gauge (extracted to `score-gauge.ts`)
5. Full mode section 2: per-capability `pages-data-table` with maturity badges
6. Full mode section 3: trend line via `pages-line-chart` (or placeholder if pages-viz not available)
7. Compact mode: coloured badge with pre-fetched data path (no fetch when `score` prop is set)
8. Events: `trust.capability-selected`
9. Example page with mock data

Each sub-step follows TDD. See spec §11 for full design including compact mode data paths, ARIA, and maturity phase thresholds.

- [ ] **Step 1: Update package.json with dependencies, create vitest.config.ts**
- [ ] **Step 2: Write failing tests for types and fetchData**
- [ ] **Step 3: Implement types and core component**
- [ ] **Step 4: Write failing tests for SVG gauge rendering**
- [ ] **Step 5: Implement score-gauge.ts — inline SVG arc with colour coding**
- [ ] **Step 6: Write failing tests for per-capability table**
- [ ] **Step 7: Implement capability breakdown with pages-data-table**
- [ ] **Step 8: Write failing tests for compact mode (pre-fetched path)**
- [ ] **Step 9: Implement compact mode — coloured badge, dual data path**
- [ ] **Step 10: Write failing tests for events**
- [ ] **Step 11: Implement event emission**
- [ ] **Step 12: Add ARIA and LiveRegionMixin announcements**
- [ ] **Step 13: Create example page with mock data**
- [ ] **Step 14: Run full test suite**
- [ ] **Step 15: Commit**

```
git commit -m "feat(trust-score-panel): trust score visualization with gauge, capabilities, and compact mode #11"
```

---

## File Structure Summary

```
packages/blocks-ui-core/
  src/data-endpoint/
    data-endpoint.ts          ← NEW: DataEndpointMixin
    data-endpoint.test.ts     ← NEW: mixin tests
    index.ts                  ← NEW: barrel export

components/data-table/
  src/types.ts                ← MODIFY: add filterable, filterValue, FilterChangeDetail
  src/pages-data-table.ts     ← MODIFY: add filter properties, UI, pipeline

components/kpi-metric-row/
  src/kpi-metric-row.ts       ← MODIFY: add willUpdate, density property, CSS

components/work-item-detail/
  src/detail-relations-tab.ts ← MODIFY: new type, grouped rendering
  src/work-item-detail.ts     ← MODIFY: fetch relations in _loadWorkItem

components/audit-trail-viewer/ ← NEW package
  src/types.ts
  src/audit-trail-viewer.ts
  src/audit-trail-viewer.test.ts

components/case-timeline/      ← REPLACE stub
  src/types.ts
  src/case-timeline.ts
  src/case-timeline.test.ts

components/trust-score-panel/  ← REPLACE stub
  src/types.ts
  src/trust-score-panel.ts
  src/score-gauge.ts
  src/trust-score-panel.test.ts

examples/
  src/pages/audit-trail-page.ts
  src/pages/case-timeline-page.ts
  src/pages/trust-score-page.ts
  mock-data/ledger-entries.json
  mock-data/case-events.json
  mock-data/trust-scores.json
```

## Dependency Graph

```
Task 1 (DataEndpointMixin) ─┬─→ Task 6 (Audit Trail)
                             ├─→ Task 7 (Case Timeline)
                             └─→ Task 8 (Trust Score)

Task 2 (#25 KPI reactive) ──→ independent
Task 3 (#24 KPI density)  ──→ independent
Task 4 (#29 text filter)  ──→ independent (Task 6 uses it)
Task 5 (#18 relations)    ──→ independent
```

Tasks 2, 3, 4, 5 are independent of each other and of Task 1. Tasks 6, 7, 8 depend on Task 1 only. Task 6 benefits from Task 4 (uses clientFilter) but can work without it.
