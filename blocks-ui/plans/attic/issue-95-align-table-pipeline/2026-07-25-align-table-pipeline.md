# Align Table Components with Pages Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #95 — refactor: align blocks-ui table components with pages pipeline
**Issue group:** #95

**Goal:** Align all 10 table-rendering components with the pages pipeline (DataSource → SourceConnector → DataSourceController) so no component manually fetches, builds datasets at render time, or manages its own loading/error state.

**Architecture:** Every component uses DataSourceMixin (or DataSourceAdapter for SSE/special cases) as the Lit integration layer for the pages pipeline. Tables use standalone mode with `client-sort`/`client-filter` for local interaction. Data flows through the pipeline for both endpoint and property data paths.

**Tech Stack:** TypeScript, Lit, `@casehubio/pages-data`, `@casehubio/pages-component`, `@casehubio/pages-table`, Vitest

## Global Constraints

- Tables stay in standalone mode — never use `.props` setter / pipeline mode (that's for the pages framework runtime)
- Column renderers use inline styles (protocol PP-20260713-8ea1af — render callbacks cross shadow DOM boundaries)
- Test framework: Vitest with jsdom, `globalThis.fetch` mocking
- Build: `yarn build` / `yarn typecheck` / per-component `yarn test`

---

### Task 1: compliance-summary — wire client-sort

**Files:**
- Modify: `components/compliance-summary/src/compliance-summary.ts`
- Test: `components/compliance-summary/src/compliance-summary.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin (already used), `pages-table` element
- Produces: no interface changes — internal refactor only

The table has four sortable columns in `TABLE_CONFIG` but the `<pages-table>` element lacks `client-sort`, so clicking headers does nothing.

- [ ] **Step 1: Write failing test — sort headers respond to clicks**

Add a test that verifies the table has `client-sort` enabled:

```ts
it('renders table with client-sort enabled', async () => {
  const el = document.createElement('compliance-summary') as ComplianceSummaryEl;
  el.requirements = TEST_REQUIREMENTS;
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();

  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table).toBeTruthy();
  expect(table.clientSort).toBe(true);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-compliance-summary test
```

Expected: FAIL — `table.clientSort` is `false`

- [ ] **Step 3: Add client-sort attribute to the table**

In `compliance-summary.ts` render method, add `client-sort` to the `<pages-table>`:

```html
<pages-table
  .dataSet=${this.dataSet}
  .columnConfig=${TABLE_CONFIG}
  .columnRenderers=${this._columnRenderers}
  client-sort
  @row-activate=${this._handleRowActivate}
></pages-table>
```

- [ ] **Step 4: Run test to verify it passes**

```bash
yarn workspace @casehubio/blocks-ui-compliance-summary test
```

Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/compliance-summary/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): compliance-summary — wire client-sort"
```

---

### Task 2: similarity-panel — wire client-sort

**Files:**
- Modify: `components/similarity-panel/src/similarity-panel.ts`
- Test: `components/similarity-panel/src/similarity-panel.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin (already used), `pages-table` element
- Produces: no interface changes

Same pattern as Task 1 — sortable columns in config but no `client-sort` on the element.

- [ ] **Step 1: Write failing test — sort headers respond to clicks**

```ts
it('renders table with client-sort enabled', async () => {
  const el = document.createElement('similarity-panel') as SimilarityPanelEl;
  el.data = TEST_PRECEDENTS;
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();

  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table).toBeTruthy();
  expect(table.clientSort).toBe(true);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-similarity-panel test
```

- [ ] **Step 3: Add client-sort attribute**

In `similarity-panel.ts` render method:

```html
<pages-table
  .dataSet=${this.dataSet}
  .columnConfig=${TABLE_CONFIG}
  .columnRenderers=${this._columnRenderers}
  client-sort
  @row-activate=${this._handleRowActivate}
></pages-table>
```

- [ ] **Step 4: Run test to verify it passes**

```bash
yarn workspace @casehubio/blocks-ui-similarity-panel test
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/similarity-panel/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): similarity-panel — wire client-sort"
```

---

### Task 3: routing-rationale — fix endpoint-path renderers, wire client-sort

**Files:**
- Modify: `components/routing-rationale/src/routing-rationale.ts`
- Test: `components/routing-rationale/src/routing-rationale.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin, `createTypedFetchSource`, `pages-table`
- Produces: no interface changes

Two bugs: (1) endpoint path stores `_rawData` and pushes dataset through the pipeline but never builds `_renderers` — table renders with no custom column renderers. (2) six sortable columns but no `client-sort`.

- [ ] **Step 1: Write failing test — endpoint path builds renderers**

```ts
it('builds column renderers when data arrives via endpoint', async () => {
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    headers: new Headers({ 'content-type': 'application/json' }),
    json: () => Promise.resolve(TEST_ROUTING_DATA),
  });

  const el = document.createElement('routing-rationale') as RoutingRationaleEl;
  el.endpoint = '/api/routing';
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();
  await el.updateComplete;

  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table).toBeTruthy();
  expect(table.columnRenderers).toBeDefined();
  expect(table.columnRenderers!.size).toBeGreaterThan(0);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-routing-rationale test
```

Expected: FAIL — `columnRenderers` is an empty Map in the endpoint path

- [ ] **Step 3: Build renderers when _rawData changes**

In `routing-rationale.ts`, update the sourceFactory callback to also build renderers, and add `client-sort` to the table. Replace the `createSourceFactory` method:

```ts
override createSourceFactory(): SourceFactory {
  return (url) => createTypedFetchSource<RoutingRationaleData>(url, (data, sink) => {
    this._rawData = data;
    this._renderers = buildColumnRenderers(data.policy, data.selected.workerId);
    const allCandidates = [data.selected, ...data.alternatives];
    const dataset = fromRows(allCandidates, CANDIDATE_COLUMNS(data.selected.workerId));
    sink.apply({ type: 'snapshot', dataset });
  });
}
```

And add `client-sort` to the table in the render method:

```html
<pages-table
  .dataSet=${this.dataSet}
  .columnConfig=${TABLE_CONFIG}
  .columnRenderers=${this._renderers}
  client-sort
  @row-activate=${this._handleRowActivate}
></pages-table>
```

- [ ] **Step 4: Run tests to verify all pass**

```bash
yarn workspace @casehubio/blocks-ui-routing-rationale test
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/routing-rationale/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): routing-rationale — fix endpoint-path renderers, wire client-sort"
```

---

### Task 4: trust-score-panel — remove duplicate dataset build, use pipeline

**Files:**
- Modify: `components/trust-score-panel/src/trust-score-panel.ts`
- Test: `components/trust-score-panel/src/trust-score-panel.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin, TrendSourceMixin, `pages-table`
- Produces: no interface changes

`_renderCapabilityTable()` (line 276) builds a brand new `fromRows()` dataset, creates fresh renderers and config on every render — duplicating work the sourceFactory already did. Fix: use `this.dataSet` from the pipeline, hoist renderers and config to stable references.

- [ ] **Step 1: Write test asserting table uses pipeline dataSet**

```ts
it('capability table uses dataSet from pipeline, not a local rebuild', async () => {
  globalThis.fetch = vi.fn().mockResolvedValue({
    ok: true,
    headers: new Headers({ 'content-type': 'application/json' }),
    json: () => Promise.resolve(TEST_TRUST_RESPONSE),
  });

  const el = document.createElement('trust-score-panel') as TrustScorePanelEl;
  el.endpoint = '/api';
  el.actorId = 'worker-1';
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();
  await el.updateComplete;

  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table).toBeTruthy();
  expect(table.dataSet).toBe(el.dataSet);
  expect(table.clientSort).toBe(true);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-trust-score-panel test
```

Expected: FAIL — table.dataSet is a locally-built dataset, not el.dataSet

- [ ] **Step 3: Refactor _renderCapabilityTable to use pipeline dataSet**

Hoist renderers and config as stable class members. Rewrite `_renderCapabilityTable()`:

```ts
private _capabilityRenderers: ReadonlyMap<ColumnId, ColumnRenderer> = new Map([
  [SCORE_COL, (cell: CellValue) => {
    const numValue = cell.type === 'NULL' ? 0 : (cell as { value: number }).value;
    const level = trustLevelFromScore(numValue);
    return html`
      <div class="score-bar">
        <div
          class="score-bar-fill ${level}"
          style="width: ${numValue * 100}%"
        ></div>
      </div>
      <span style="margin-left: 8px">${numValue.toFixed(2)}</span>
    `;
  }],
]);

private _capabilityConfig: readonly TableColumnConfig[] = [
  { id: TAG_COL, sortable: true },
  { id: SCORE_COL, sortable: true },
];
```

Replace `_renderCapabilityTable()`:

```ts
private _renderCapabilityTable() {
  if (!this.dataSet || this.dataSet.rows.length === 0) {
    return html`<p>No capability scores available</p>`;
  }

  return html`
    <pages-table
      .dataSet=${this.dataSet}
      .columnConfig=${this._capabilityConfig}
      .columnRenderers=${this._capabilityRenderers}
      client-sort
      @row-activate=${this._handleCapabilityClick}
    ></pages-table>
  `;
}
```

- [ ] **Step 4: Run all tests**

```bash
yarn workspace @casehubio/blocks-ui-trust-score-panel test
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/trust-score-panel/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): trust-score-panel — remove duplicate fromRows, use pipeline dataSet"
```

---

### Task 5: trust-workbench — pass data to list-pane through pipeline

**Files:**
- Modify: `components/trust-workbench/src/trust-workbench.ts`
- Test: `components/trust-workbench/src/trust-workbench.test.ts`

**Interfaces:**
- Consumes: list-pane (DataSourceMixin host), routing-rationale, trust-score-panel
- Produces: no interface changes

`_syncListPane()` imperatively queries `list-pane` from the shadow DOM and sets `.dataSet` / `.endpoint` directly. This works but couples trust-workbench to list-pane's internals. Refactor: bind list-pane properties declaratively in the template. Move filtering and fromRows into a reactive getter.

- [ ] **Step 1: Write test verifying list-pane receives correct data via properties**

```ts
it('passes routing history to list-pane via template binding', async () => {
  const el = document.createElement('trust-workbench') as TrustWorkbenchEl;
  el.endpoint = '/api';
  el.actorId = 'worker-1';
  el.routingHistory = TEST_ROUTING_HISTORY;
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();
  await el.updateComplete;

  const listPane = el.shadowRoot!.querySelector('list-pane') as any;
  expect(listPane).toBeTruthy();
  expect(listPane.dataSet).toBeDefined();
  expect(listPane.dataSet.rows.length).toBe(TEST_ROUTING_HISTORY.length);
  el.remove();
});
```

- [ ] **Step 2: Run test — verify current behavior passes or identify gaps**

```bash
yarn workspace @casehubio/blocks-ui-trust-workbench test
```

- [ ] **Step 3: Refactor to declarative template binding**

Add a reactive getter for the list dataset and bind properties in the template instead of `_syncListPane()`:

```ts
private get _listDataSet(): TypedDataSet | undefined {
  if (!this.routingHistory) return undefined;
  const source = this._selectedCapability
    ? this.routingHistory.filter(s => s.capabilityTag === this._selectedCapability)
    : this.routingHistory;
  return fromRows([...source], ROUTING_HISTORY_COLUMNS);
}

private get _listEndpoint(): string | undefined {
  if (this.routingHistory) return undefined;
  return this._routingEndpoint;
}
```

Update the template to bind these properties declaratively:

```html
<list-pane
  selection-topic="trust-routing"
  .endpoint=${this._listEndpoint}
  .dataSet=${this._listDataSet}
  .columnConfig=${this.routingColumns ?? ROUTING_HISTORY_TABLE_CONFIG}
  .columnRenderers=${this.routingColumnRenderers ?? DEFAULT_ROUTING_RENDERERS}
  .getRowKey=${(row: TypedRow) => row.text(ID_COL)}
  empty-message="No routing decisions"
></list-pane>
```

Remove `_syncListPane()` method and the `updated()` override that called it. Remove `willUpdate` if only used for `_syncListPane` triggering (keep the `actorId` reset).

- [ ] **Step 4: Run all tests**

```bash
yarn workspace @casehubio/blocks-ui-trust-workbench test
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/trust-workbench/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): trust-workbench — declarative list-pane binding, remove _syncListPane"
```

---

### Task 6: grouped-data-view — verify pipeline alignment

**Files:**
- Modify: `components/grouped-data-view/src/grouped-data-view.ts` (if needed)
- Test: `components/grouped-data-view/src/grouped-data-view.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin (already used), `pages-grouped-view` element
- Produces: no interface changes

Already uses DataSourceMixin correctly. The `updated()` lifecycle passes `this.dataSet` from the pipeline to `pages-grouped-view`. The pre-sort in `_prepareDataSet()` is legitimate domain logic (group ordering). Verify alignment and add `client-sort` if missing.

- [ ] **Step 1: Review and verify alignment**

Read the component and confirm:
- DataSourceMixin delivers `this.dataSet` ✓
- `_prepareDataSet()` transforms (sorts) the pipeline's dataset before passing to grouped-view ✓
- `gv.dataSet = preparedDataSet` goes to the grouped-view element ✓
- No manual fetch, no manual fromRows bypass

- [ ] **Step 2: Run existing tests to confirm baseline**

```bash
yarn workspace @casehubio/blocks-ui-grouped-data-view test
```

Expected: all tests PASS

- [ ] **Step 3: Commit if any changes made, otherwise skip**

---

### Task 7: audit-trail-viewer — adopt DataSourceMixin

**Files:**
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.ts`
- Test: `components/audit-trail-viewer/src/audit-trail-viewer.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin (to be adopted), `pages-table`, `createTypedFetchSource`
- Produces: no public interface changes — `endpoint`, `loading`, `error`, `dataSet` stay the same

Currently creates two `DataSourceAdapter` instances manually. The `entries` adapter fetches ledger entries, stores them in `this._entries` (component state), and pushes an **empty** dataset to the sink — using the adapter only for loading/error state tracking. The `verify` adapter handles Merkle verification (separate concern, stays as-is).

Refactor the entries adapter to DataSourceMixin. The sourceFactory fetches entries, transforms them into a real dataset, and pushes through the pipeline. Domain-specific filtering (actor/type/date) applies to the pipeline's dataset before rendering.

- [ ] **Step 1: Write test — pipeline delivers real dataset**

```ts
it('delivers entries dataset through the pipeline', async () => {
  globalThis.fetch = mockFetch(TEST_LEDGER_ENTRIES);

  const el = document.createElement('audit-trail-viewer') as AuditTrailViewerEl;
  el.endpoint = '/api/ledger/entries';
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();
  await el.updateComplete;

  expect(el.dataSet).toBeDefined();
  expect(el.dataSet!.rows.length).toBe(TEST_LEDGER_ENTRIES.length);

  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table).toBeTruthy();
  expect(table.dataSet).toBeDefined();
  expect(table.dataSet.rows.length).toBe(TEST_LEDGER_ENTRIES.length);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-audit-trail-viewer test
```

Expected: FAIL — current pipeline pushes empty dataset

- [ ] **Step 3: Refactor to DataSourceMixin**

Change the class declaration to extend DataSourceMixin:

```ts
@customElement('audit-trail-viewer')
export class AuditTrailViewer extends DataSourceMixin(LitElement) {
```

Override `createSourceFactory` to fetch and build the real dataset:

```ts
override createSourceFactory(): SourceFactory {
  return (url) => createTypedFetchSource<LedgerEntry[]>(url, (data, sink) => {
    this._allEntries = data;
    const dataset = fromRows(data, ENTRY_COL_DEFS);
    sink.apply({ type: 'snapshot', dataset });
  });
}
```

Store `_allEntries` for domain filtering. The `_filteredEntries` getter filters `_allEntries` by actor/type/date. In the render method, when filters are active, compute a filtered dataset:

```ts
private get _filteredDataSet(): TypedDataSet | undefined {
  if (!this._allEntries) return this.dataSet;
  if (!this._hasActiveFilters()) return this.dataSet;
  const filtered = this._applyFilters(this._allEntries);
  return fromRows(filtered, ENTRY_COL_DEFS);
}
```

Pass `_filteredDataSet` to the table instead of rebuilding on every render.

Remove the manual `entries` DataSourceAdapter instance. Keep the `verify` adapter (separate concern — Merkle verification is not table data).

Add `client-sort` and `client-filter` to the table element.

- [ ] **Step 4: Update remaining tests for new data flow**

Existing tests that check `_entries` or `_filteredEntries` may need to check `dataSet` and `_allEntries` instead. Update test setup to work with DataSourceMixin's fetch pattern.

- [ ] **Step 5: Run all tests**

```bash
yarn workspace @casehubio/blocks-ui-audit-trail-viewer test
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/audit-trail-viewer/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): audit-trail-viewer — adopt DataSourceMixin, pipeline delivers real dataset"
```

---

### Task 8: preferences-editor — adopt DataSourceAdapter

**Files:**
- Modify: `components/preferences-editor/src/preferences-editor.ts`
- Test: `components/preferences-editor/src/preferences-editor.test.ts`

**Interfaces:**
- Consumes: DataSourceAdapter (to be adopted), `pages-data-table`, `PreferencesApi`
- Produces: no public interface changes

Currently extends plain `LitElement` with manual `PreferencesApi` fetch and `_buildDataSet()` / `_buildRows()` for tree construction. Adopt DataSourceAdapter for loading/error/dataSet state management. The tree-building logic moves into a sourceFactory transform.

Note: this component uses `<pages-data-table>` (not `<pages-table>`) and the `.props` setter for expandable tree config. This is correct and stays — the expandable config is a props feature, not pipeline mode activation for event handling purposes.

- [ ] **Step 1: Write test — adapter manages loading/error state**

```ts
it('manages loading state through the adapter', async () => {
  const el = document.createElement('preferences-editor') as PreferencesEditorEl;
  el.endpoint = '/api/preferences';
  el.fetchFn = vi.fn().mockImplementation(() => new Promise(() => {})); // never resolves
  document.body.appendChild(el);
  await el.updateComplete;

  expect(el.loading).toBe(true);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-preferences-editor test
```

- [ ] **Step 3: Adopt DataSourceAdapter**

Add a `DataSourceAdapter` field and wire loading/error/dataSet through it. The sourceFactory wraps `PreferencesApi`:

```ts
private _dataSource = new DataSourceAdapter(this, {
  sourceFactory: (url) => {
    let abort: AbortController | undefined;
    return {
      connect: (sink) => {
        abort = new AbortController();
        this._loadPreferences(url, sink, abort.signal);
      },
      disconnect: () => { abort?.abort(); },
    };
  },
});

private async _loadPreferences(url: string, sink: DataSink, signal: AbortSignal): Promise<void> {
  try {
    const api = new PreferencesApi(url, this.fetchFn);
    const [schema, values] = await Promise.all([api.getSchema(), api.getValues()]);
    if (signal.aborted) return;
    this._schema = schema;
    this._scopedValues = values;
    const rows = this._buildRows(schema, values);
    const dataset = fromRows(rows, PREFERENCE_COLUMNS);
    sink.apply({ type: 'snapshot', dataset });
  } catch (err) {
    if (signal.aborted) return;
    sink.error({ message: err instanceof Error ? err.message : String(err), permanent: true });
  }
}
```

Add loading/error/dataSet getters that delegate to the adapter:

```ts
get loading(): boolean { return this._dataSource.loading; }
get error(): string { return this._dataSource.error; }
get dataSet(): TypedDataSet | undefined { return this._dataSource.dataSet; }
```

Wire `endpoint` changes to trigger the adapter:

```ts
override willUpdate(changed: PropertyValues): void {
  super.willUpdate(changed);
  if (changed.has('endpoint') && this.endpoint) {
    this._dataSource.endpoint = this.endpoint;
  }
}
```

Remove manual loading/error state management from the existing `_loadData()` method.

- [ ] **Step 4: Update tests for adapter-managed state**

Tests that mock PreferencesApi or check loading/error transitions may need adjustment to work with the adapter's lifecycle.

- [ ] **Step 5: Run all tests**

```bash
yarn workspace @casehubio/blocks-ui-preferences-editor test
```

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/preferences-editor/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): preferences-editor — adopt DataSourceAdapter for pipeline lifecycle"
```

---

### Task 9: work-item-inbox — adopt DataSourceAdapter with SSE

**Files:**
- Modify: `components/work-item-inbox/src/work-item-inbox.ts`
- Test: `components/work-item-inbox/src/work-item-inbox.test.ts`

**Interfaces:**
- Consumes: DataSourceAdapter (to be adopted), `pages-table`, SSEManager
- Produces: no public interface changes

The most complex refactor. Currently: manual `fetch()` in `fetchItems()`, manual `fromRows()` on every render in the template, SSE updates via `SSEManager` mutating `this.items` directly, manual filtering.

Adopt DataSourceAdapter for the initial fetch. SSE updates modify the dataset through the adapter. Domain filtering (mode/status/priority/overdue/breach) stays as component logic — it filters `_allItems` and pushes a filtered dataset through the adapter.

- [ ] **Step 1: Write test — adapter delivers initial dataset**

```ts
it('delivers initial items through the adapter pipeline', async () => {
  globalThis.fetch = mockFetch({ items: TEST_ITEMS });

  const el = document.createElement('work-item-inbox') as WorkItemInboxEl;
  el.endpoint = '/api/work-items';
  document.body.appendChild(el);
  await el.updateComplete;
  await flush();
  await el.updateComplete;

  expect(el.loading).toBe(false);
  const table = el.shadowRoot!.querySelector('pages-table') as any;
  expect(table).toBeTruthy();
  expect(table.dataSet).toBeDefined();
  expect(table.dataSet.rows.length).toBe(TEST_ITEMS.length);
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn workspace @casehubio/blocks-ui-work-item-inbox test
```

- [ ] **Step 3: Adopt DataSourceAdapter**

Add a DataSourceAdapter with a sourceFactory that does the initial fetch:

```ts
private _dataSource = new DataSourceAdapter(this, {
  sourceFactory: (url) => {
    let abort: AbortController | undefined;
    return {
      connect: (sink) => {
        abort = new AbortController();
        this._fetchInitial(url, sink, abort.signal);
      },
      disconnect: () => { abort?.abort(); },
    };
  },
});
```

The `_fetchInitial` method replaces `fetchItems()`:

```ts
private async _fetchInitial(url: string, sink: DataSink, signal: AbortSignal): Promise<void> {
  try {
    const response = await fetch(url, { signal });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const data = await response.json();
    if (signal.aborted) return;
    this._allItems = data.items;
    this._pushFilteredDataset(sink);
    this._setupSSE(url);
  } catch (err) {
    if (signal.aborted || (err as Error).name === 'AbortError') return;
    sink.error({ message: err instanceof Error ? err.message : String(err), permanent: true });
  }
}
```

SSE handlers update `_allItems` and rebuild the filtered dataset:

```ts
private _rebuildDataset(): void {
  const filtered = this._applyFilters(this._allItems);
  this._dataSource.dataSet = fromRows(filtered, INBOX_COL_DEFS);
}
```

Remove `fromRows()` from the render template. The table binds to the adapter's `dataSet`:

```html
<pages-table
  .dataSet=${this._dataSource.dataSet}
  .columnConfig=${INBOX_COL_CONFIG}
  .columnRenderers=${this._columnRenderers}
  .getRowKey=${this._getRowKey}
  .getRowClass=${this._getRowClass}
  .loading=${this._dataSource.loading}
  mode="auto"
  selection="multi"
  client-sort
  @selection-change=${this._handleSelectionChange}
></pages-table>
```

- [ ] **Step 4: Update SSE handlers to push through adapter**

Each SSE event handler (`_handleItemAdded`, `_handleItemRemoved`, `_handleItemUpdated`) modifies `_allItems` and calls `_rebuildDataset()` instead of relying on Lit reactivity on `this.items`.

- [ ] **Step 5: Update filter handlers to rebuild dataset**

When filter state changes (mode, status, priority, overdue, breach), call `_rebuildDataset()` to push a new filtered dataset through the adapter.

- [ ] **Step 6: Update tests for adapter-managed flow**

Tests that check `this.items` or the render-time `fromRows()` call need to check the adapter's `dataSet` instead. Mock fetch setup stays the same.

- [ ] **Step 7: Run all tests**

```bash
yarn workspace @casehubio/blocks-ui-work-item-inbox test
```

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/work-item-inbox/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#95): work-item-inbox — adopt DataSourceAdapter, SSE through pipeline"
```

---

### Task 10: list-pane — verify alignment, add any missing features

**Files:**
- Modify: `components/list-pane/src/list-pane.ts` (if needed)
- Test: `components/list-pane/src/list-pane.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin (already correct)
- Produces: no changes

list-pane is the reference implementation — already uses DataSourceMixin correctly with `client-sort`, `client-filter`, paginated mode, and `totalRows` from the controller. Verify nothing is missing.

- [ ] **Step 1: Verify alignment**

Confirm:
- DataSourceMixin provides `this.dataSet` ✓
- Table has `client-sort` and `client-filter` ✓
- `mode="paginated"` with `pageSize` ✓
- `totalRows` from `this.dataSource.controller.totalRows` ✓
- No manual `fromRows()` ✓

- [ ] **Step 2: Run existing tests**

```bash
yarn workspace @casehubio/blocks-ui-list-pane test
```

Expected: all PASS

- [ ] **Step 3: Skip commit if no changes needed**

---

### Task 11: Full build verification

**Files:** none (verification only), plus tag standardisation if needed

- [ ] **Step 0: Verify element tag consistency**

Check which tag the current pages-table dist registers. If `pages-table` and `pages-data-table` are both registered, pick the source-of-truth tag (`pages-data-table` per source) and align preferences-editor's template tag if it differs from the other 9 components. If only one tag is registered, align all components to that tag.

- [ ] **Step 1: Run full build**

```bash
yarn build
```

- [ ] **Step 2: Run full typecheck**

```bash
yarn typecheck
```

- [ ] **Step 3: Run all tests**

```bash
yarn test
```

- [ ] **Step 4: Commit any remaining fixes**

If typecheck or tests surface cross-component issues (e.g., type changes in shared exports), fix and commit.
