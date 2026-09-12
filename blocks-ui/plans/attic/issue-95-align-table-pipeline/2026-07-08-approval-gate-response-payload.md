# Approval Gate Response Payload Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #40 — approval-gate gate.decided event should include HTTP response payload
**Issue group:** #40

**Goal:** Parse the HTTP response body from the gate completion endpoint and include it in the `gate.decided` event payload, plus extract server error messages for better error display.

**Architecture:** Two changes in `_submitDecision`: (1) parse `response.json()` on success and include as `serverData` in the event, (2) parse error response bodies to show server error messages instead of bare HTTP status codes. A new `GateDecidedPayload` interface types the event payload.

**Tech Stack:** TypeScript, Lit, Vitest

## Global Constraints

- Pre-release project — no backward compat shims needed
- `emitPagesEvent(element, topic, payload)` is the event emission API
- `vi.stubGlobal('fetch', ...)` is the established fetch mock pattern
- Tests use `requireConfirmation = false` to skip confirmation dialog in fetch tests

---

### Task 1: Add `GateDecidedPayload` interface and `serverData` to the event

**Files:**
- Modify: `components/approval-gate/src/approval-gate.ts:7-10` (add interface near `ApprovalGateTopics`)
- Modify: `components/approval-gate/src/approval-gate.ts:506-538` (`_submitDecision` method)
- Modify: `components/approval-gate/src/index.ts` (export new type)
- Test: `components/approval-gate/src/approval-gate.test.ts`

**Interfaces:**
- Produces: `GateDecidedPayload { readonly gateId: string; readonly outcome: string; readonly resolution?: string; readonly serverData: unknown; }` — exported from `index.ts`

- [ ] **Step 1: Write failing test — success with JSON body includes serverData**

```typescript
it('includes serverData in gate.decided event on success', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  const serverResponse = { trustScore: { before: 72, after: 85 }, status: 'completed' };
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: true, json: () => Promise.resolve(serverResponse) }));
  const handler = vi.fn();
  document.addEventListener('pages-event', handler);
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  const event = handler.mock.calls.find((c: any) => c[0].detail.topic === 'gate.decided');
  expect(event).toBeTruthy();
  expect(event![0].detail.payload.serverData).toEqual(serverResponse);
  document.removeEventListener('pages-event', handler);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run components/approval-gate/src/approval-gate.test.ts -t "includes serverData"`
Expected: FAIL — `serverData` is `undefined` (not in current payload)

- [ ] **Step 3: Write failing test — success with empty body sets serverData to null**

```typescript
it('sets serverData to null when response body cannot be parsed', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: true, json: () => Promise.reject(new SyntaxError('Unexpected end of JSON input')) }));
  const handler = vi.fn();
  document.addEventListener('pages-event', handler);
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  const event = handler.mock.calls.find((c: any) => c[0].detail.topic === 'gate.decided');
  expect(event).toBeTruthy();
  expect(event![0].detail.payload.serverData).toBeNull();
  document.removeEventListener('pages-event', handler);
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `yarn vitest run components/approval-gate/src/approval-gate.test.ts -t "sets serverData to null"`
Expected: FAIL — `serverData` is `undefined`

- [ ] **Step 5: Implement — add GateDecidedPayload interface**

Use `ide_insert_member` to add the interface after `ApprovalGateTopics` (after line 10, before `OutcomeDefinition`):

```typescript
export interface GateDecidedPayload {
  readonly gateId: string;
  readonly outcome: string;
  readonly resolution?: string;
  readonly serverData: unknown;
}
```

- [ ] **Step 6: Implement — modify _submitDecision to parse response and include serverData**

Use `ide_replace_member` on `_submitDecision` to replace the body with:

```typescript
this._submitting = true;
this._error = null;

try {
  const body: { outcome: string; resolution?: string } = { outcome: outcomeKey };
  if (resolution) body.resolution = resolution;

  const response = await fetch(`${this.endpoint}/workitems/${this.gateId}/complete`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  const serverData: unknown = await response.json().catch(() => null);

  const outcomeLabel = this.outcomes.find(o => o.key === outcomeKey)?.label ?? outcomeKey;
  emitPagesEvent(this, ApprovalGateTopics.DECIDED, {
    gateId: this.gateId,
    outcome: outcomeKey,
    resolution,
    serverData,
  } satisfies GateDecidedPayload);
  this.announce(`Decision submitted: ${outcomeLabel}`);
} catch (err) {
  const message = err instanceof Error ? err.message : String(err);
  this._error = message;
  this.announce(`Decision failed: ${message}`, 'assertive');
} finally {
  this._submitting = false;
}
```

- [ ] **Step 7: Export GateDecidedPayload from index.ts**

Modify `components/approval-gate/src/index.ts` to add the type export:

```typescript
export { ApprovalGate, ApprovalGateTopics, type OutcomeDefinition, type QuorumConfig, type VoterStatus, type GateDecision, type GateDecidedPayload } from './approval-gate.js';
```

- [ ] **Step 8: Run all tests to verify both new tests pass and existing tests still pass**

Run: `yarn vitest run components/approval-gate/src/approval-gate.test.ts`
Expected: ALL PASS (including the two new tests and all existing tests)

- [ ] **Step 9: Typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 10: Commit**

```
feat(approval-gate): include serverData in gate.decided event (#40)

Parse response.json() after successful gate completion and include
the result as serverData in the gate.decided event payload.
Consumers no longer need a separate fetch for post-decision server data.
```

---

### Task 2: Parse error response bodies for better error messages

**Files:**
- Modify: `components/approval-gate/src/approval-gate.ts:506-538` (`_submitDecision` method — error path)
- Test: `components/approval-gate/src/approval-gate.test.ts`

**Interfaces:**
- Consumes: Nothing new — modifies the error path of `_submitDecision`
- Produces: Nothing new — changes visible error message text only

- [ ] **Step 1: Write failing test — error with JSON `error` field**

```typescript
it('shows server error message from response body error field', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: false,
    status: 409,
    json: () => Promise.resolve({ error: 'Gate already completed by another voter' }),
  }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error!.textContent).toContain('Gate already completed by another voter');
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run components/approval-gate/src/approval-gate.test.ts -t "shows server error message from response body error field"`
Expected: FAIL — error text contains "HTTP 409" instead of the server message

- [ ] **Step 3: Write failing test — error with JSON `message` field**

```typescript
it('shows server error message from response body message field', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: false,
    status: 422,
    json: () => Promise.resolve({ message: 'Outcome not allowed for this gate type' }),
  }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error!.textContent).toContain('Outcome not allowed for this gate type');
});
```

- [ ] **Step 4: Write failing test — error with no parseable body falls back**

```typescript
it('falls back to HTTP status when error body is not parseable', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: false,
    status: 500,
    json: () => Promise.reject(new SyntaxError('Unexpected token')),
  }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error!.textContent).toContain('HTTP 500');
});
```

- [ ] **Step 5: Write failing test — error with JSON array body falls back**

```typescript
it('falls back to HTTP status when error body is a JSON array', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: false,
    status: 400,
    json: () => Promise.resolve(['validation error 1', 'validation error 2']),
  }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error!.textContent).toContain('HTTP 400');
});
```

- [ ] **Step 6: Write failing test — error with empty string error field falls back**

```typescript
it('falls back to HTTP status when error field is empty string', async () => {
  el.requireConfirmation = false;
  await el.updateComplete;
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
    ok: false,
    status: 403,
    json: () => Promise.resolve({ error: '' }),
  }));
  el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
  await el.updateComplete;
  await vi.runAllTimersAsync();
  await el.updateComplete;

  const error = el.shadowRoot!.querySelector('.error');
  expect(error!.textContent).toContain('HTTP 403');
});
```

- [ ] **Step 7: Run all new tests to verify they fail**

Run: `yarn vitest run components/approval-gate/src/approval-gate.test.ts -t "shows server error message|falls back to HTTP status"`
Expected: FAIL — all show "HTTP {status}" instead of server messages (tests 1-2), or pass trivially because they already show HTTP status (tests 3-6 may already pass with current code since the error mock now provides `json`)

Note: tests 3-6 may pass already since the current code throws `HTTP ${status}` and the mock provides a `json` that is never called. That's fine — they serve as regression tests for the new code path.

- [ ] **Step 8: Implement — modify error path in _submitDecision**

Replace the `if (!response.ok)` block in `_submitDecision`. The current code:

```typescript
if (!response.ok) {
  throw new Error(`HTTP ${response.status}`);
}
```

Replace with:

```typescript
if (!response.ok) {
  const errorBody: unknown = await response.json().catch(() => null);
  const serverMessage =
    errorBody !== null && typeof errorBody === 'object' && !Array.isArray(errorBody)
      ? (errorBody as Record<string, unknown>).error ?? (errorBody as Record<string, unknown>).message
      : null;
  throw new Error(
    typeof serverMessage === 'string' && serverMessage
      ? serverMessage
      : `HTTP ${response.status}`
  );
}
```

- [ ] **Step 9: Run all tests**

Run: `yarn vitest run components/approval-gate/src/approval-gate.test.ts`
Expected: ALL PASS

- [ ] **Step 10: Verify the existing HTTP 500 error test still passes**

The existing test at line 253 mocks `{ ok: false, status: 500 }` without a `json` method. The new code calls `response.json()` which will throw (no method) — the `.catch(() => null)` handles this, so it falls back to `HTTP 500`. Confirm this is the case in the test output.

- [ ] **Step 11: Typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 12: Commit**

```
feat(approval-gate): use server error messages in error display (#40)

Parse error response body for human-readable error messages.
Precedence: body.error → body.message → HTTP ${status} fallback.
Empty strings and non-object bodies fall back to HTTP status.
```
