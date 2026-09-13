# Review Findings — UI Primitives Batch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #23 — Minor review findings from UI primitives batch
**Issue group:** #23

**Goal:** Fix sla-indicator invalid date handling, add aria-valuemin to approval-gate, and add missing test coverage for approval-gate error paths and kpi-metric-row endpoint mode.

**Architecture:** Three independent components, each with targeted fixes. No new files — all changes are to existing source and test files. Production changes in sla-indicator (validity tracking) and approval-gate (aria attribute). Test-only changes in approval-gate and kpi-metric-row.

**Tech Stack:** TypeScript, Lit 3, vitest, Web Components

## Global Constraints

- All imports use `.js` extensions
- CSS uses `var(--blocks-*)` design tokens — no hardcoded values
- Fetch mocks use `vi.stubGlobal('fetch', ...)` with `vi.restoreAllMocks()` in `afterEach`
- Tests run via `yarn test`

---

### Task 1: sla-indicator invalid date guard

**Files:**
- Modify: `components/sla-indicator/src/sla-indicator.ts`
- Modify: `components/sla-indicator/src/sla-indicator.test.ts`

**Interfaces:**
- Consumes: `subscribe`/`unsubscribe` from `@casehubio/blocks-ui-core` (unchanged)
- Produces: `SlaIndicator` component now renders `nothing` for invalid deadlines and warns once per transition via `console.warn`

- [ ] **Step 1: Write three failing tests**

Add `vi.restoreAllMocks()` to the `afterEach` block, then add three tests at the end of the describe block in `components/sla-indicator/src/sla-indicator.test.ts`:

```typescript
// In afterEach, add after vi.useRealTimers():
vi.restoreAllMocks();
```

```typescript
it('renders nothing for an invalid deadline', async () => {
  el.deadline = 'not-a-date';
  await el.updateComplete;
  const indicator = el.shadowRoot!.querySelector('.sla-indicator');
  expect(indicator).toBeFalsy();
});

it('does not emit state-changed for an invalid deadline', async () => {
  const handler = vi.fn();
  document.addEventListener('pages-event', handler);
  el.deadline = 'not-a-date';
  await el.updateComplete;
  const stateEvent = handler.mock.calls.find(
    (c: any) => c[0].detail.topic === 'sla.state-changed'
  );
  expect(stateEvent).toBeFalsy();
  document.removeEventListener('pages-event', handler);
});

it('transitions from valid to invalid deadline without emitting', async () => {
  el.deadline = new Date(Date.now() + 86400000).toISOString();
  await el.updateComplete;
  expect(el.shadowRoot!.querySelector('.sla-indicator')).toBeTruthy();

  const handler = vi.fn();
  document.addEventListener('pages-event', handler);
  el.deadline = 'now-invalid';
  await el.updateComplete;
  expect(el.shadowRoot!.querySelector('.sla-indicator')).toBeFalsy();
  const stateEvent = handler.mock.calls.find(
    (c: any) => c[0].detail.topic === 'sla.state-changed'
  );
  expect(stateEvent).toBeFalsy();
  document.removeEventListener('pages-event', handler);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn test components/sla-indicator`
Expected: 3 failures — invalid deadline currently renders `"Breached NaNm ago"` instead of nothing, and emits a spurious `state-changed` event.

- [ ] **Step 3: Add `_deadlineValid` state field**

In `components/sla-indicator/src/sla-indicator.ts`, add the state field after the existing `_state` field (line 67):

```typescript
@state() private _deadlineValid = true;
```

- [ ] **Step 4: Add validity guard in `_update()`**

Replace the body of `_update()` in `components/sla-indicator/src/sla-indicator.ts`:

```typescript
private _update(): void {
  if (!this.deadline) return;
  const deadlineMs = new Date(this.deadline).getTime();
  if (isNaN(deadlineMs)) {
    if (this._deadlineValid) {
      console.warn(`sla-indicator: invalid deadline value "${this.deadline}"`);
    }
    this._deadlineValid = false;
    this._remaining = 0;
    this._state = 'normal';
    return;
  }
  this._deadlineValid = true;
  this._remaining = deadlineMs - Date.now();
  this._state = this._computeState();

  if (this._state !== this._lastEmittedState) {
    this._lastEmittedState = this._state;
    emitPagesEvent(this, SlaIndicatorTopics.STATE_CHANGED, {
      state: this._state,
      deadline: this.deadline,
    });
  }
}
```

- [ ] **Step 5: Add validity guard in `render()`**

In `components/sla-indicator/src/sla-indicator.ts`, change the first line of `render()`:

From:
```typescript
if (!this.deadline) return nothing;
```

To:
```typescript
if (!this.deadline || !this._deadlineValid) return nothing;
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn test components/sla-indicator`
Expected: All tests pass (existing 13 + 3 new = 16 total).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/sla-indicator/src/sla-indicator.ts components/sla-indicator/src/sla-indicator.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "fix(sla-indicator): guard against invalid deadline values #23

Add @state _deadlineValid flag — computed once in _update(), checked
in render(). Invalid deadlines render nothing, warn once per transition,
and do not emit spurious state-changed events.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: approval-gate aria-valuemin and error path tests

**Files:**
- Modify: `components/approval-gate/src/approval-gate.ts`
- Modify: `components/approval-gate/src/approval-gate.test.ts`

**Interfaces:**
- Consumes: `LiveRegionMixin.announce()` appends `div[aria-live]` to `document.body` with the message as `textContent`
- Produces: No interface changes — tests verify existing behavior, `aria-valuemin` is an attribute addition

- [ ] **Step 1: Fix existing fetch mock and afterEach**

In `components/approval-gate/src/approval-gate.test.ts`:

Add `vi.restoreAllMocks()` to the `afterEach` block after `vi.useRealTimers()`:

```typescript
afterEach(() => {
  el.remove();
  vi.useRealTimers();
  vi.restoreAllMocks();
});
```

Fix the existing test at line 136 — replace `globalThis.fetch = vi.fn()...` with `vi.stubGlobal`:

From:
```typescript
globalThis.fetch = vi.fn().mockResolvedValue({ ok: true, json: () => ({}) });
```

To:
```typescript
vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: true, json: () => ({}) }));
```

- [ ] **Step 2: Write failing aria-valuemin test**

Add to the existing quorum test or add a new test in `components/approval-gate/src/approval-gate.test.ts`:

```typescript
it('quorum progressbar has aria-valuemin', async () => {
  el.quorum = {
    required: 3,
    total: 5,
    voters: [
      { id: 'user-2', name: 'Bob', status: 'voted', outcome: 'approve' },
      { id: 'user-3', name: 'Charlie', status: 'pending' },
      { id: 'user-1', name: 'Alice', status: 'pending' },
    ],
  };
  await el.updateComplete;
  const progressbar = el.shadowRoot!.querySelector('[role="progressbar"]');
  expect(progressbar!.getAttribute('aria-valuemin')).toBe('0');
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn test components/approval-gate`
Expected: FAIL — `aria-valuemin` is null (attribute not present).

- [ ] **Step 4: Add aria-valuemin attribute**

In `components/approval-gate/src/approval-gate.ts`, in `_renderQuorum()`, add `aria-valuemin="0"` to the progressbar element:

From:
```html
<div class="quorum-bar" role="progressbar"
     aria-valuenow="${votedCount}"
     aria-valuemax="${q.required}"
```

To:
```html
<div class="quorum-bar" role="progressbar"
     aria-valuemin="0"
     aria-valuenow="${votedCount}"
     aria-valuemax="${q.required}"
```

- [ ] **Step 5: Run test to verify aria-valuemin passes**

Run: `yarn test components/approval-gate`
Expected: All existing tests pass + new aria-valuemin test passes.

- [ ] **Step 6: Write error path tests**

Add two tests to `components/approval-gate/src/approval-gate.test.ts`:

```typescript
it('renders error and re-enables buttons on HTTP error', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: false, status: 500 }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error).toBeTruthy();
  expect(error!.textContent).toContain('HTTP 500');

  const buttons = Array.from(el.shadowRoot!.querySelectorAll<HTMLButtonElement>('.action-btn'));
  for (const btn of buttons) {
    expect(btn.disabled).toBe(false);
  }

  const liveRegion = document.body.querySelector('[aria-live]');
  expect(liveRegion).toBeTruthy();
  expect(liveRegion!.textContent).toBe('Decision failed: HTTP 500');
  expect(liveRegion!.getAttribute('aria-live')).toBe('assertive');

  // Verify error clears on successful retry
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: true, json: () => ({}) }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;
  expect(el.shadowRoot!.querySelector('.error')).toBeFalsy();
});

it('renders error on network failure', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockRejectedValue(new Error('Network error')));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error).toBeTruthy();
  expect(error!.textContent).toContain('Network error');

  const buttons = Array.from(el.shadowRoot!.querySelectorAll<HTMLButtonElement>('.action-btn'));
  for (const btn of buttons) {
    expect(btn.disabled).toBe(false);
  }

  const liveRegion = document.body.querySelector('[aria-live]');
  expect(liveRegion!.textContent).toBe('Decision failed: Network error');
  expect(liveRegion!.getAttribute('aria-live')).toBe('assertive');
});
```

- [ ] **Step 7: Run tests to verify error path tests pass**

Run: `yarn test components/approval-gate`
Expected: All tests pass (existing 15 + 3 new = 18 total). The error path tests verify existing production code — they should pass immediately.

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/approval-gate/src/approval-gate.ts components/approval-gate/src/approval-gate.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "fix(approval-gate): add aria-valuemin and error path tests #23

Add explicit aria-valuemin=\"0\" on quorum progressbar.
Add tests for HTTP error and network failure paths including
error-clear-on-retry cycle and announce() assertions.
Standardise fetch mocks to vi.stubGlobal with vi.restoreAllMocks.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: kpi-metric-row endpoint mode tests

**Files:**
- Modify: `components/kpi-metric-row/src/kpi-metric-row.test.ts`

**Interfaces:**
- Consumes: `KpiMetricRow.endpoint`, `KpiMetricRow.refresh()`, `MetricDefinition` type
- Produces: No interface changes — test-only task

Note: kpi-metric-row tests do NOT use `vi.useFakeTimers()`. The component has no timer dependency. Async fetch settling uses `await new Promise(r => setTimeout(r, 0))` to flush the microtask queue, then `await el.updateComplete` for the Lit render.

- [ ] **Step 1: Add afterEach cleanup**

In `components/kpi-metric-row/src/kpi-metric-row.test.ts`, update the `afterEach` block:

From:
```typescript
afterEach(() => el.remove());
```

To:
```typescript
afterEach(() => {
  el.remove();
  vi.restoreAllMocks();
});
```

- [ ] **Step 2: Write five endpoint mode tests**

Add a `describe('endpoint mode', ...)` block at the end of the outer describe in `components/kpi-metric-row/src/kpi-metric-row.test.ts`. Each test creates a fresh element with `endpoint` set before appending to DOM (because `_fetchMetrics` runs in `connectedCallback` only on mount):

```typescript
describe('endpoint mode', () => {
  it('renders loading skeleton when fetching', async () => {
    vi.stubGlobal('fetch', vi.fn(() => new Promise(() => {})));
    const endpointEl = document.createElement('kpi-metric-row') as KpiMetricRowEl;
    endpointEl.endpoint = '/api/metrics';
    document.body.appendChild(endpointEl);
    await endpointEl.updateComplete;
    const skeletons = endpointEl.shadowRoot!.querySelectorAll('.skeleton-card');
    expect(skeletons.length).toBeGreaterThan(0);
    endpointEl.remove();
  });

  it('renders metric cards on successful fetch', async () => {
    const mockData: MetricDefinition[] = [
      { key: 'test', value: 42, label: 'Test Metric' },
    ];
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve(mockData),
    }));
    const endpointEl = document.createElement('kpi-metric-row') as KpiMetricRowEl;
    endpointEl.endpoint = '/api/metrics';
    document.body.appendChild(endpointEl);
    await new Promise(r => setTimeout(r, 0));
    await endpointEl.updateComplete;
    const cards = endpointEl.shadowRoot!.querySelectorAll('[role="listitem"]');
    expect(cards.length).toBe(1);
    expect(endpointEl.shadowRoot!.textContent).toContain('42');
    endpointEl.remove();
  });

  it('renders error state on fetch failure', async () => {
    vi.stubGlobal('fetch', vi.fn().mockRejectedValue(new Error('Network error')));
    const endpointEl = document.createElement('kpi-metric-row') as KpiMetricRowEl;
    endpointEl.endpoint = '/api/metrics';
    document.body.appendChild(endpointEl);
    await new Promise(r => setTimeout(r, 0));
    await endpointEl.updateComplete;
    const error = endpointEl.shadowRoot!.querySelector('.error');
    expect(error).toBeTruthy();
    expect(error!.textContent).toContain('Network error');
    endpointEl.remove();
  });

  it('refresh() re-fetches metrics', async () => {
    const initialData: MetricDefinition[] = [
      { key: 'a', value: 1, label: 'Initial' },
    ];
    const updatedData: MetricDefinition[] = [
      { key: 'b', value: 2, label: 'Updated' },
    ];
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve(initialData),
    }));
    const endpointEl = document.createElement('kpi-metric-row') as KpiMetricRowEl;
    endpointEl.endpoint = '/api/metrics';
    document.body.appendChild(endpointEl);
    await new Promise(r => setTimeout(r, 0));
    await endpointEl.updateComplete;
    expect(endpointEl.shadowRoot!.textContent).toContain('Initial');

    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve(updatedData),
    }));
    await endpointEl.refresh();
    await endpointEl.updateComplete;
    expect(endpointEl.shadowRoot!.textContent).toContain('Updated');
    endpointEl.remove();
  });

  it('does not re-fetch when endpoint changes after mount', async () => {
    const mockFn = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve([{ key: 'a', value: 1, label: 'A' }]),
    });
    vi.stubGlobal('fetch', mockFn);
    const endpointEl = document.createElement('kpi-metric-row') as KpiMetricRowEl;
    endpointEl.endpoint = '/api/metrics';
    document.body.appendChild(endpointEl);
    await new Promise(r => setTimeout(r, 0));
    await endpointEl.updateComplete;
    expect(mockFn).toHaveBeenCalledTimes(1);

    endpointEl.endpoint = '/api/other-metrics';
    await new Promise(r => setTimeout(r, 0));
    await endpointEl.updateComplete;
    expect(mockFn).toHaveBeenCalledTimes(1);
    endpointEl.remove();
  });
});
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `yarn test components/kpi-metric-row`
Expected: All tests pass (existing 12 + 5 new = 17 total). These tests verify existing production code — they should pass immediately.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/kpi-metric-row/src/kpi-metric-row.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "test(kpi-metric-row): add endpoint mode coverage #23

Add tests for loading skeleton, successful fetch, fetch error,
refresh(), and endpoint-change-after-mount behavior.
Standardise fetch mocks to vi.stubGlobal with vi.restoreAllMocks.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```
