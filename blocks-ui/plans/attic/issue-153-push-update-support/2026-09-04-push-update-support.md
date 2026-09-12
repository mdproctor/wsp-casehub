# Push Unification via EventStreamController — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #153 — Push update support on blocks-list-pane and blocks-work-item-inbox via EventConnection
**Issue group:** #153

**Goal:** Unify all push updates on EventStreamController, migrate 8 SSEManager consumers, add push to list-pane and kpi-metric-row, remove SSEManager from blocks-ui.

**Architecture:** Components use EventStreamController (Lit ReactiveController wrapping EventStream/EventConnection) for push events. Simple consumers use reactive pull (`controller.latest` in render). Complex consumers (inbox) use EventStream directly with `onChange` callback for imperative domain logic. Connection sharing via default EventStreamPool (automatic, no explicit pool property needed). DataSourceMixin stays unchanged — push is orthogonal to fetch.

**Tech Stack:** EventStreamController + EventStream from `@casehubio/pages-component` and `@casehubio/pages-data`, Lit 3, TypeScript, vitest

## Global Constraints

- EventStreamController import: `import { EventStreamController } from '@casehubio/pages-component';`
- EventStream import: `import { EventStream } from '@casehubio/pages-data';`
- SSEManager import to remove: `import { SSEManager } from '@casehubio/pages-data/dist/sse/sse-manager.js';`
- EventStreamController constructor: `new EventStreamController<T>(host, url, topics, options?)`
- EventStream uses `shared: true` by default — pool sharing is automatic
- `controller.latest` returns the most recent event payload (typed via generic)
- `controller.status` returns `'connected' | 'reconnecting' | 'disconnected'`
- Deletion convention: `{ _deleted: true }` in event payload
- ARIA: announce push-driven state changes via `this.announce()` (LiveRegionMixin)

---

## Batch 1: list-pane push support (Pattern A — new feature)

### Task 1: Add push event handling to list-pane

**Files:**
- Modify: `components/list-pane/src/list-pane.ts`
- Modify: `components/list-pane/src/list-pane.test.ts` (or create if absent)

**Interfaces:**
- Consumes: `EventStreamController` from `@casehubio/pages-component`, `fromRows` from `@casehubio/pages-data`
- Produces: `pushUrl`, `pushTopics` properties on ListPane; push events update the displayed dataset

- [ ] **Step 1: Write failing test — push event appends new row**

Add to list-pane test file:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import './list-pane.js';

describe('list-pane push support', () => {
  let el: any;

  afterEach(() => {
    el?.remove();
  });

  it('has pushUrl and pushTopics properties', async () => {
    el = document.createElement('blocks-list-pane') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.pushUrl).toBe('');
    expect(el.pushTopics).toEqual([]);
  });

  it('creates EventStreamController when pushUrl and pushTopics set', async () => {
    el = document.createElement('blocks-list-pane') as any;
    el.pushUrl = '/api/push';
    el.pushTopics = ['items:*'];
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el._pushStream).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd components/list-pane && npx vitest run --reporter=verbose`
Expected: FAIL — pushUrl property not defined

- [ ] **Step 3: Implement push support on list-pane**

Add to `list-pane.ts`:

```typescript
import { EventStreamController } from '@casehubio/pages-component';

// New properties
@property({ attribute: 'push-url' }) pushUrl = '';
@property({ attribute: false }) pushTopics: string[] = [];

// Internal state
private _pushStream: EventStreamController | null = null;
@state() private _lastPushEvent: unknown = undefined;
```

In `willUpdate`:
```typescript
if (changed.has('pushUrl') || changed.has('pushTopics')) {
  this._teardownPush();
  if (this.pushUrl && this.pushTopics.length) {
    this._setupPush();
  }
}

// Detect push event change and apply to dataset
const latest = this._pushStream?.latest;
if (latest !== undefined && latest !== this._lastPushEvent) {
  this._lastPushEvent = latest;
  this._applyPushEvent(latest as Record<string, unknown>);
}
```

Add methods:
```typescript
private _setupPush(): void {
  this._pushStream = new EventStreamController(this, this.pushUrl, this.pushTopics);
}

private _teardownPush(): void {
  // EventStreamController auto-disconnects via hostDisconnected
  this._pushStream = null;
}

private _applyPushEvent(payload: Record<string, unknown>): void {
  if (!this.dataSet) return;
  const key = this._resolveKeyFromPayload(payload);
  if (!key) return;

  if (payload._deleted) {
    this._removeRow(key);
    return;
  }

  const existingIndex = this._findRowIndexByKey(key);
  if (existingIndex >= 0) {
    this._replaceRow(existingIndex, payload);
  } else {
    this._appendRow(payload);
  }
}

private _resolveKeyFromPayload(payload: Record<string, unknown>): string | null {
  // Use column definitions to convert payload to a temporary TypedRow, then extract key
  if (!this._colDefs || !this.getRowKey) return null;
  try {
    const tempDs = fromRows([payload], this._colDefs);
    if (tempDs.rows.length === 0) return null;
    return this.getRowKey(tempDs.rows[0]);
  } catch { return null; }
}

private _findRowIndexByKey(key: string): number {
  if (!this.dataSet || !this.getRowKey) return -1;
  return this.dataSet.rows.findIndex(row => this.getRowKey!(row) === key);
}

private _removeRow(key: string): void {
  if (!this.dataSet) return;
  const rows = this.dataSet.rows.filter(row => this.getRowKey!(row) !== key);
  this.dataSet = fromRows(rows.map(r => this._rowToRecord(r)), this._colDefs!);
  this.announce('Item removed');
}

private _replaceRow(index: number, payload: Record<string, unknown>): void {
  if (!this.dataSet) return;
  const records = this.dataSet.rows.map((r, i) =>
    i === index ? payload : this._rowToRecord(r)
  );
  this.dataSet = fromRows(records, this._colDefs!);
}

private _appendRow(payload: Record<string, unknown>): void {
  if (!this.dataSet) return;
  const records = [payload, ...this.dataSet.rows.map(r => this._rowToRecord(r))];
  this.dataSet = fromRows(records, this._colDefs!);
  this.announce('New item received');
}

private _rowToRecord(row: TypedRow): Record<string, unknown> {
  const record: Record<string, unknown> = {};
  for (const col of this._colDefs ?? []) {
    record[col.id as string] = row.value(col.id);
  }
  return record;
}
```

Note: `_colDefs` must be stored internally — extract from the existing column setup. If list-pane doesn't currently store its raw column definitions, add a `@state() private _colDefs` that's set alongside `columnConfig`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd components/list-pane && npx vitest run --reporter=verbose`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add components/list-pane/
git commit -m "feat(list-pane): add push update support via EventStreamController Refs #153"
```

---

## Batch 2: Simple SSE migrations (Pattern B — reactive pull)

### Task 2: execution-monitor — SSEManager → EventStreamController

**Files:**
- Modify: `components/execution-monitor/src/execution-monitor.ts`
- Modify: `components/execution-monitor/src/execution-monitor.test.ts` (or create)

**Interfaces:**
- Consumes: `EventStreamController` from `@casehubio/pages-component`
- Produces: `pushUrl`, `pushTopics` properties replacing SSEManager; `controller.latest` provides `ExecutionSnapshot`

- [ ] **Step 1: Write failing test**

```typescript
it('has pushUrl and pushTopics properties', async () => {
  el = document.createElement('blocks-execution-monitor') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect(el.pushTopics).toBeDefined();
});

it('does not import SSEManager', async () => {
  // Verify by checking the component has no _sseManager property
  el = document.createElement('blocks-execution-monitor') as any;
  document.body.appendChild(el);
  expect((el as any)._sseManager).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd components/execution-monitor && npx vitest run --reporter=verbose`
Expected: FAIL — pushUrl not defined, _sseManager still exists

- [ ] **Step 3: Implement migration**

Remove from imports:
```typescript
// REMOVE:
import { SSEManager } from '@casehubio/pages-data/dist/sse/sse-manager.js';
import type { SSEEvent } from '@casehubio/pages-data/dist/sse/sse-manager.js';
```

Add:
```typescript
import { EventStreamController } from '@casehubio/pages-component';
```

Remove fields:
```typescript
// REMOVE:
private _sseManager = new SSEManager();
private _sseHandler = (event: SSEEvent) => { ... };
private _sseUrl = '';
```

Add properties and controller:
```typescript
@property({ attribute: 'push-url' }) pushUrl = '';
@property({ attribute: false }) pushTopics: string[] = [];

private _pushStream: EventStreamController<ExecutionSnapshot> | null = null;
@state() private _lastPushEvent: ExecutionSnapshot | undefined = undefined;
```

Replace `_reconnectSSE` / `_teardownSSE` with push lifecycle in `willUpdate`:
```typescript
// In willUpdate — when endpoint or executionId changes:
if (changed.has('pushUrl') || changed.has('pushTopics') || changed.has('executionId')) {
  this._pushStream = null; // old controller auto-disconnects
  if (this.pushUrl && this.executionId) {
    const topics = this.pushTopics.length ? this.pushTopics : [`execution:${this.executionId}:*`];
    this._pushStream = new EventStreamController<ExecutionSnapshot>(this, this.pushUrl, topics);
  }
}

// Detect push event and apply
const latest = this._pushStream?.latest;
if (latest !== undefined && latest !== this._lastPushEvent) {
  this._lastPushEvent = latest;
  this._snapshot = latest;
  this._stale = false;
  this._lastUpdateTime = Date.now();
}

// Connection status
if (this._pushStream) {
  this._connected = this._pushStream.status === 'connected';
}
```

Remove `_reconnectSSE()`, `_teardownSSE()` methods entirely.

Remove SSE cleanup from `disconnectedCallback` (EventStreamController handles it via hostDisconnected).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd components/execution-monitor && npx vitest run --reporter=verbose`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add components/execution-monitor/
git commit -m "refactor(execution-monitor): migrate SSEManager to EventStreamController Refs #153"
```

### Task 3: topology-viewer — SSEManager → EventStreamController

**Files:**
- Modify: `components/topology-viewer/src/topology-viewer.ts`

**Interfaces:**
- Same pattern as Task 2

- [ ] **Step 1: Write failing test**

```typescript
it('has pushUrl and pushTopics properties', async () => {
  el = document.createElement('blocks-topology-viewer') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect(el.pushTopics).toBeDefined();
});

it('no SSEManager reference', async () => {
  el = document.createElement('blocks-topology-viewer') as any;
  document.body.appendChild(el);
  expect((el as any)._sseManager).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement migration**

Same pattern as Task 2:
1. Remove `SSEManager` import, `_sseManager` field, `_sseHandler`, subscribe/unsubscribe calls
2. Add `import { EventStreamController } from '@casehubio/pages-component'`
3. Add `pushUrl`, `pushTopics` properties
4. Create `EventStreamController` in `willUpdate` when pushUrl/topics change
5. Read `controller.latest` for topology state updates
6. Derive `_connected` from `controller.status`

Default topics: `['topology:*']`

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git add components/topology-viewer/
git commit -m "refactor(topology-viewer): migrate SSEManager to EventStreamController Refs #153"
```

### Task 4: reconciliation-status — SSEManager → EventStreamController

**Files:**
- Modify: `components/reconciliation-status/src/reconciliation-status.ts`

**Interfaces:**
- Same pattern as Task 2

- [ ] **Step 1: Write failing test**

```typescript
it('has pushUrl and pushTopics properties and no SSEManager', async () => {
  el = document.createElement('blocks-reconciliation-status') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect((el as any)._sseManager).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement migration**

Same pattern as Task 2. Default topics: `['reconciliation:*']`

1. Remove SSEManager import and fields
2. Add EventStreamController with `pushUrl`, `pushTopics`
3. Read `controller.latest` for reconciliation state
4. Derive connection status from `controller.status`

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git add components/reconciliation-status/
git commit -m "refactor(reconciliation-status): migrate SSEManager to EventStreamController Refs #153"
```

### Task 5: session-detail — SSEManager → EventStreamController

**Files:**
- Modify: `components/session-detail/src/session-detail.ts`

**Interfaces:**
- Same pattern as Task 2

- [ ] **Step 1: Write failing test**

```typescript
it('has pushUrl and pushTopics properties and no SSEManager', async () => {
  el = document.createElement('blocks-session-detail') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect((el as any)._sseManager).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement migration**

Same pattern as Task 2. Session-detail manages SSE per tab — the EventStreamController lifecycle should be tied to tab activation:
1. Create controller when a tab with SSE requirements is activated
2. Nullify controller when tab is deactivated (auto-disconnects)
3. Default topics: `['session:{sessionId}:*']`

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git add components/session-detail/
git commit -m "refactor(session-detail): migrate SSEManager to EventStreamController Refs #153"
```

---

## Batch 3: Complex SSE migrations (Pattern B — imperative domain handlers)

### Task 6: work-item-inbox — SSEManager → EventStream with domain handlers

**Files:**
- Modify: `components/work-item-inbox/src/work-item-inbox.ts`
- Modify: `components/work-item-inbox/src/work-item-inbox.test.ts`

**Interfaces:**
- Consumes: `EventStream` from `@casehubio/pages-data` (not EventStreamController — inbox needs imperative onChange callback for fetch-on-event)
- Produces: `pushUrl`, `pushTopics` properties; existing domain handlers (`handleItemAppears`, `handleItemDisappears`, `handleItemUpdated`) called via EventStream onChange

- [ ] **Step 1: Write failing tests**

```typescript
it('has pushUrl and pushTopics properties', async () => {
  el = createElement();
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect(el.pushTopics).toBeDefined();
});

it('no SSEManager reference', async () => {
  el = createElement();
  await el.updateComplete;
  expect((el as any).sseManager).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd components/work-item-inbox && npx vitest run --reporter=verbose`
Expected: FAIL — pushUrl not defined, sseManager still exists

- [ ] **Step 3: Implement migration**

Remove:
```typescript
// REMOVE these imports:
import { SSEManager } from '@casehubio/pages-data/dist/sse/sse-manager.js';
import type { SSEEvent } from '@casehubio/pages-data/dist/sse/sse-manager.js';

// REMOVE these fields:
private sseManager = new SSEManager();
private sseHandler = (event: SSEEvent) => { ... };

// REMOVE these methods:
private subscribeSSE() { ... }
private unsubscribeSSE() { ... }
```

Add:
```typescript
import { EventStream } from '@casehubio/pages-data';

@property({ attribute: 'push-url' }) pushUrl = '';
@property({ attribute: false }) pushTopics: string[] = [];

private _pushStream: EventStream<WorkItemLifecycleEvent> | null = null;
```

Add push lifecycle methods:
```typescript
private _setupPush(): void {
  if (!this.pushUrl) return;
  const topics = this.pushTopics.length ? this.pushTopics : ['work-items:lifecycle:*'];
  this._pushStream = new EventStream<WorkItemLifecycleEvent>(this.pushUrl, topics, {
    onChange: () => {
      const event = this._pushStream?.latest;
      if (event) this._handlePushEvent(event);
    },
  });
  this._pushStream.connect();
}

private _teardownPush(): void {
  this._pushStream?.disconnect();
  this._pushStream = null;
}

private _handlePushEvent(data: WorkItemLifecycleEvent): void {
  // Same routing as the old handleSSEEvent, without the SSEEvent wrapper
  switch (data.type) {
    case 'CREATED': case 'ASSIGNED': case 'SLA_REASSIGNED':
      this.handleItemAppears(data.workItemId); break;
    case 'COMPLETED': case 'REJECTED': case 'FAULTED':
    case 'CANCELLED': case 'OBSOLETE': case 'EXPIRED': case 'ESCALATED':
      this.handleItemDisappears(data.workItemId); break;
    case 'SPAWNED': case 'SIGNAL_RECEIVED': case 'CLAIM_EXPIRED':
      this.fetchSummary(); break;
    default:
      this.handleItemUpdated(data.workItemId); break;
  }
}
```

Update `connectedCallback`:
```typescript
// Replace: this.subscribeSSE();
// With:
if (this.pushUrl) this._setupPush();
```

Update `disconnectedCallback`:
```typescript
// Replace: this.unsubscribeSSE();
// With:
this._teardownPush();
```

Queue-scoped push — replace `_subscribeQueueSSE` / `_unsubscribeQueueSSE`:
```typescript
private _queuePushStream: EventStream | null = null;

private _subscribeQueuePush(queueId: string): void {
  this._unsubscribeQueuePush();
  if (!this.pushUrl) return;
  this._queuePushStream = new EventStream(this.pushUrl, [`work-items:queue:${queueId}:*`], {
    onChange: () => {
      const event = this._queuePushStream?.latest;
      if (event) this._handleQueueSSEEvent(event as any);
    },
  });
  this._queuePushStream.connect();
}

private _unsubscribeQueuePush(): void {
  this._queuePushStream?.disconnect();
  this._queuePushStream = null;
}
```

Update `_handleQueueScopeChanged` to use `_subscribeQueuePush` / `_unsubscribeQueuePush` instead of SSE versions.

Domain handlers (`handleItemAppears`, `handleItemDisappears`, `handleItemUpdated`, `_handleQueueItemAdded`, etc.) are **unchanged** — they receive the same data, only the transport wrapper is different.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd components/work-item-inbox && npx vitest run --reporter=verbose`
Expected: All existing tests PASS + new push property tests PASS

- [ ] **Step 5: Commit**

```bash
git add components/work-item-inbox/
git commit -m "refactor(work-item-inbox): migrate SSEManager to EventStream with domain handlers Refs #153"
```

### Task 7: notification-inbox + notification-bell — SSEManager → EventStreamController

**Files:**
- Modify: `components/notification-inbox/src/notification-inbox.ts`
- Modify: `components/notification-inbox/src/notification-bell.ts`

**Interfaces:**
- Consumes: `EventStreamController` from `@casehubio/pages-component`
- Produces: `pushUrl`, `pushTopics` properties on both components

- [ ] **Step 1: Write failing tests**

```typescript
// notification-inbox
it('has pushUrl property and no SSEManager', async () => {
  el = document.createElement('blocks-notification-inbox') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect((el as any)._sseManager).toBeUndefined();
});

// notification-bell
it('has pushUrl property and no SSEManager', async () => {
  el = document.createElement('blocks-notification-bell') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect((el as any)._sseManager).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement migration — notification-inbox**

Same pattern as Task 2 (reactive pull for most event types). If the inbox has imperative handlers like work-item-inbox, use EventStream with `onChange` instead.

1. Remove SSEManager import and fields
2. Add `pushUrl`, `pushTopics` properties
3. Create EventStreamController in willUpdate
4. Read `controller.latest` or use `onChange` for imperative logic
5. Default topics: `['notifications:*']`

- [ ] **Step 4: Implement migration — notification-bell**

1. Remove SSEManager import and fields
2. Add `pushUrl`, `pushTopics` properties
3. Create EventStreamController for unread count events
4. Default topics: `['notifications:unread-count']`

- [ ] **Step 5: Run tests to verify they pass**

- [ ] **Step 6: Commit**

```bash
git add components/notification-inbox/
git commit -m "refactor(notification-inbox): migrate SSEManager to EventStreamController Refs #153"
```

---

## Batch 4: kpi-metric-row push + cleanup

### Task 8: kpi-metric-row — replace polling with EventStreamController

**Files:**
- Modify: `components/kpi-metric-row/src/kpi-metric-row.ts`
- Modify: `components/kpi-metric-row/src/kpi-metric-row.test.ts` (or create)

**Interfaces:**
- Consumes: `EventStreamController` from `@casehubio/pages-component`
- Produces: `pushUrl`, `pushTopics` properties; polling timer disabled when push active

- [ ] **Step 1: Write failing tests**

```typescript
it('has pushUrl and pushTopics properties', async () => {
  el = document.createElement('blocks-kpi-metric-row') as any;
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.pushUrl).toBeDefined();
  expect(el.pushTopics).toBeDefined();
});

it('disables polling timer when pushUrl is set', async () => {
  el = document.createElement('blocks-kpi-metric-row') as any;
  el.refreshInterval = 5000;
  el.pushUrl = '/api/push';
  el.pushTopics = ['kpi:metrics:*'];
  document.body.appendChild(el);
  await el.updateComplete;
  // Timer should not be running
  expect((el as any)._refreshTimer).toBeNull();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement push support**

Add:
```typescript
import { EventStreamController } from '@casehubio/pages-component';

@property({ attribute: 'push-url' }) pushUrl = '';
@property({ attribute: false }) pushTopics: string[] = [];

private _pushStream: EventStreamController<MetricDefinition> | null = null;
@state() private _lastPushMetric: MetricDefinition | undefined = undefined;
```

In `willUpdate`:
```typescript
if (changed.has('pushUrl') || changed.has('pushTopics')) {
  this._pushStream = null;
  if (this.pushUrl && this.pushTopics.length) {
    this._pushStream = new EventStreamController<MetricDefinition>(this, this.pushUrl, this.pushTopics);
    this._stopRefreshTimer(); // disable polling when push active
  } else if (this.refreshInterval) {
    this._startRefreshTimer(); // re-enable polling when push removed
  }
}

// Apply push event
const latest = this._pushStream?.latest;
if (latest !== undefined && latest !== this._lastPushMetric) {
  this._lastPushMetric = latest;
  this._applyMetricUpdate(latest);
}
```

Add method:
```typescript
private _applyMetricUpdate(update: MetricDefinition): void {
  const index = this.metrics.findIndex(m => m.key === update.key);
  if (index >= 0) {
    this.metrics = [...this.metrics.slice(0, index), update, ...this.metrics.slice(index + 1)];
    this.announce(`${update.label ?? update.key} updated`);
  } else {
    this.metrics = [...this.metrics, update];
  }
}
```

In `_startRefreshTimer`, guard against starting timer when push is active:
```typescript
private _startRefreshTimer(): void {
  if (this.pushUrl) return; // don't poll when push is active
  // ... existing timer logic
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git add components/kpi-metric-row/
git commit -m "feat(kpi-metric-row): add push support via EventStreamController, replace polling Refs #153"
```

### Task 9: Remove SSEManager imports and update docs

**Files:**
- Verify: all component files — grep for SSEManager
- Modify: `CLAUDE.md` — update component descriptions
- Modify: `ARC42STORIES.MD` — update §5 if needed

**Interfaces:**
- Consumes: all completed tasks (1–8)
- Produces: clean codebase with zero SSEManager references

- [ ] **Step 1: Verify no SSEManager references remain**

```bash
grep -r "SSEManager\|sse-manager\|sseManager\|sseHandler" components/ --include="*.ts" -l
```

Expected: zero results. If any remain, fix them.

- [ ] **Step 2: Update CLAUDE.md component descriptions**

Update the Key Directories table entries for:
- `work-item-inbox` — mention EventStream push instead of SSE lifecycle
- `execution-monitor` — mention EventStreamController instead of SSE
- Any other component descriptions that reference SSE

- [ ] **Step 3: Update ARC42STORIES.MD §6**

Verify §6 ("SSE and DataSource are orthogonal") is accurate — it should now reference EventStreamController adoption, not just aspiration.

- [ ] **Step 4: Run full test suite**

```bash
yarn test
```

Expected: all tests pass across all components

- [ ] **Step 5: Commit**

```bash
git add CLAUDE.md ARC42STORIES.MD
git commit -m "docs: update component inventory for EventStreamController migration Refs #153"
```

## References

- [2026-09-04-push-update-support-design.md](/Users/mdproctor/claude/public/casehub/blocks-ui/specs/issue-153-push-update-support/2026-09-04-push-update-support-design.md) — design spec
- [decisions.md](/Users/mdproctor/claude/public/casehub/blocks-ui/specs/issue-153-push-update-support/decisions.md) — D1–D6
- [EventStreamController](/Users/mdproctor/claude/casehub/blocks-ui/.casehub-packages/packages/pages-component/src/controller/event-stream-controller.ts) — Lit ReactiveController, constructor: `(host, url, topics, options?)`
- [EventStream](/Users/mdproctor/claude/casehub/blocks-ui/.casehub-packages/packages/pages-data/src/event-stream/event-stream.ts) — lower-level, `onChange` callback, `connect()`/`disconnect()`
- [EventStreamPool](/Users/mdproctor/claude/casehub/blocks-ui/.casehub-packages/packages/pages-data/src/event-stream/event-stream-pool.ts) — ref-counted connection sharing (automatic via `shared: true`)
- [execution-monitor SSEManager usage](/Users/mdproctor/claude/casehub/blocks-ui/components/execution-monitor/src/execution-monitor.ts) — reference Pattern B consumer
- [work-item-inbox SSE handlers](/Users/mdproctor/claude/casehub/blocks-ui/components/work-item-inbox/src/work-item-inbox.ts:472-623) — domain logic preserved
- [Spec review R1](/Users/mdproctor/reviews/casehub-blocks-ui/issue-153-push-update-support-spec-20260904-151403/responses/reviewer-1.md) — 13 findings, corrected architecture
- [GitHub #153](https://github.com/casehubio/blocks-ui/issues/153)
