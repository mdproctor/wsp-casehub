# DataSourceMixin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #44 — feat: migrate DataEndpointMixin to DataSourceMixin
**Issue group:** #44

**Goal:** Replace DataEndpointMixin with a two-layer Lit binding
(DataSourceAdapter + DataSourceMixin) wrapping pages' DataSourceController,
then migrate all four consumer components.

**Architecture:** `fetchSource` provides raw-JSON DataSource for domain
components. `DataSourceAdapter` is a Lit ReactiveController wrapping
DataSourceController with lifecycle hooks. `DataSourceMixin` adds
convenience properties (endpoint as HTML attribute, loading/error/dataSet
getters/setters, configure()). Components set endpoint and receive data —
the controller handles all fetch lifecycle.

**Tech Stack:** TypeScript, Lit 3, vitest + jsdom,
`@casehubio/pages-component` (DataSourceController),
`@casehubio/pages-data` (DataSource/DataSink types)

## Global Constraints

- Import `DataSourceController` from `@casehubio/pages-component`
  (barrel export — not `dist/` paths)
- Import `DataSource`, `DataSink` from `@casehubio/pages-data`
  (barrel export)
- `as never` for type assertions (not `as any`)
- `fetchSource` headers accept `Record<string, string> | (() => Record<string, string>)`
  for refresh-safe dynamic state
- All test files use vitest + jsdom, matching existing patterns in blocks-ui-core
- `useDefineForClassFields: false` in vitest esbuild config (already set) —
  required for decorator-based class fields
- Pre-release: delete DataEndpointMixin, do not deprecate

---

### Task 1: fetchSource — raw JSON DataSource

**Files:**
- Create: `packages/blocks-ui-core/src/data-source/fetch-source.ts`
- Test: `packages/blocks-ui-core/src/data-source/fetch-source.test.ts`

**Interfaces:**
- Consumes: `DataSource`, `DataSink` types from `@casehubio/pages-data`
- Produces: `fetchSource(url: string, options?: FetchSourceOptions): DataSource`,
  `FetchSourceOptions` interface

- [ ] **Step 1: Write failing tests for fetchSource**

Create `packages/blocks-ui-core/src/data-source/fetch-source.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { fetchSource, type FetchSourceOptions } from './fetch-source.js';
import type { DataSink } from '@casehubio/pages-data';

function createMockSink(): DataSink & { applyCalls: any[]; errorCalls: any[] } {
  const sink = {
    applyCalls: [] as any[],
    errorCalls: [] as any[],
    apply(event: any) { sink.applyCalls.push(event); },
    error(err: any) { sink.errorCalls.push(err); },
  };
  return sink;
}

function mockFetchOk(data: unknown): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve(data),
  }) as unknown as typeof globalThis.fetch;
}

function mockFetchFail(status: number): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue({
    ok: false,
    status,
  }) as unknown as typeof globalThis.fetch;
}

function mockFetchReject(message: string): typeof globalThis.fetch {
  return vi.fn().mockRejectedValue(new Error(message)) as unknown as typeof globalThis.fetch;
}

describe('fetchSource', () => {
  it('delivers JSON response as snapshot dataset', async () => {
    const data = [{ id: 1, name: 'Alice' }];
    const source = fetchSource('http://api/items', { fetchFn: mockFetchOk(data) });
    const sink = createMockSink();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));

    expect(sink.applyCalls).toHaveLength(1);
    expect(sink.applyCalls[0].type).toBe('snapshot');
    expect(sink.applyCalls[0].dataset).toEqual(data);
  });

  it('calls sink.error on HTTP failure', async () => {
    const source = fetchSource('http://api/items', { fetchFn: mockFetchFail(500) });
    const sink = createMockSink();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));

    expect(sink.errorCalls).toHaveLength(1);
    expect(sink.errorCalls[0].message).toContain('500');
    expect(sink.errorCalls[0].permanent).toBe(true);
  });

  it('calls sink.error on network failure', async () => {
    const source = fetchSource('http://api/items', { fetchFn: mockFetchReject('network down') });
    const sink = createMockSink();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));

    expect(sink.errorCalls).toHaveLength(1);
    expect(sink.errorCalls[0].message).toContain('network down');
  });

  it('does not call sink after disconnect (abort)', async () => {
    let resolvePromise!: (v: any) => void;
    const fetchFn = vi.fn().mockReturnValue(
      new Promise(r => { resolvePromise = r; })
    ) as unknown as typeof globalThis.fetch;

    const source = fetchSource('http://api/items', { fetchFn });
    const sink = createMockSink();
    source.connect(sink);
    source.disconnect();

    resolvePromise({ ok: true, json: () => Promise.resolve([]) });
    await new Promise(r => setTimeout(r, 10));

    expect(sink.applyCalls).toHaveLength(0);
    expect(sink.errorCalls).toHaveLength(0);
  });

  it('passes static headers', async () => {
    const fetchFn = mockFetchOk([]);
    const source = fetchSource('http://api/items', {
      fetchFn,
      headers: { 'X-Custom': 'value' },
    });
    const sink = createMockSink();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));

    expect(fetchFn).toHaveBeenCalledWith('http://api/items', expect.objectContaining({
      headers: { 'X-Custom': 'value' },
    }));
  });

  it('evaluates dynamic headers function on each connect', async () => {
    let callCount = 0;
    const fetchFn = mockFetchOk([]);
    const source = fetchSource('http://api/items', {
      fetchFn,
      headers: () => {
        callCount++;
        return { 'X-Call': String(callCount) };
      },
    });
    const sink = createMockSink();

    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));
    expect(fetchFn).toHaveBeenCalledWith('http://api/items', expect.objectContaining({
      headers: { 'X-Call': '1' },
    }));

    source.disconnect();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));
    expect(fetchFn).toHaveBeenLastCalledWith('http://api/items', expect.objectContaining({
      headers: { 'X-Call': '2' },
    }));
  });

  it('uses custom method', async () => {
    const fetchFn = mockFetchOk([]);
    const source = fetchSource('http://api/items', { fetchFn, method: 'POST' });
    const sink = createMockSink();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));

    expect(fetchFn).toHaveBeenCalledWith('http://api/items', expect.objectContaining({
      method: 'POST',
    }));
  });

  it('passes body', async () => {
    const fetchFn = mockFetchOk([]);
    const source = fetchSource('http://api/items', { fetchFn, body: '{"q":"x"}' });
    const sink = createMockSink();
    source.connect(sink);
    await new Promise(r => setTimeout(r, 10));

    expect(fetchFn).toHaveBeenCalledWith('http://api/items', expect.objectContaining({
      body: '{"q":"x"}',
    }));
  });

  it('uses globalThis.fetch when no fetchFn provided', async () => {
    const originalFetch = globalThis.fetch;
    globalThis.fetch = mockFetchOk({ result: true });
    try {
      const source = fetchSource('http://api/items');
      const sink = createMockSink();
      source.connect(sink);
      await new Promise(r => setTimeout(r, 10));

      expect(sink.applyCalls).toHaveLength(1);
      expect(sink.applyCalls[0].dataset).toEqual({ result: true });
    } finally {
      globalThis.fetch = originalFetch;
    }
  });
});
```

- [ ] **Step 2: Run tests — verify all fail**

Run: `yarn vitest run packages/blocks-ui-core/src/data-source/fetch-source.test.ts`
Expected: All tests fail — module not found.

- [ ] **Step 3: Implement fetchSource**

Create `packages/blocks-ui-core/src/data-source/fetch-source.ts`:

```typescript
import type { DataSource, DataSink } from "@casehubio/pages-data";

export interface FetchSourceOptions {
  readonly method?: string;
  readonly headers?: Record<string, string> | (() => Record<string, string>);
  readonly body?: string;
  readonly fetchFn?: typeof globalThis.fetch;
}

export function fetchSource(url: string, options?: FetchSourceOptions): DataSource {
  let abort: AbortController | undefined;
  return {
    connect(sink: DataSink) {
      abort = new AbortController();
      const doFetch = options?.fetchFn ?? globalThis.fetch.bind(globalThis);
      const headers = typeof options?.headers === "function"
        ? options.headers()
        : options?.headers;
      doFetch(url, {
        method: options?.method,
        headers,
        body: options?.body,
        signal: abort.signal,
      })
        .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
        .then(data => sink.apply({ type: "snapshot", dataset: data as never }))
        .catch(err => {
          if (err.name !== "AbortError") {
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

- [ ] **Step 4: Run tests — verify all pass**

Run: `yarn vitest run packages/blocks-ui-core/src/data-source/fetch-source.test.ts`
Expected: All 9 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/data-source/fetch-source.ts packages/blocks-ui-core/src/data-source/fetch-source.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(core): fetchSource — raw JSON DataSource for domain components #44"
```

---

### Task 2: DataSourceAdapter — Lit ReactiveController

**Files:**
- Create: `packages/blocks-ui-core/src/data-source/data-source-adapter.ts`
- Test: `packages/blocks-ui-core/src/data-source/data-source-adapter.test.ts`

**Interfaces:**
- Consumes: `DataSourceController`, `DataSourceControllerOptions`, `SourceFactory`
  from `@casehubio/pages-component`;
  `ReactiveController`, `ReactiveControllerHost` from `lit`
- Produces: `DataSourceAdapter` class — `endpoint`, `loading`, `error`, `dataSet`,
  `source` (get/set), `refresh()`, `dispose()`, `hostConnected()`, `hostDisconnected()`,
  `controller` (readonly escape hatch)

- [ ] **Step 1: Write failing tests for DataSourceAdapter**

Create `packages/blocks-ui-core/src/data-source/data-source-adapter.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import { LitElement, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import { DataSourceAdapter } from './data-source-adapter.js';
import { fetchSource } from './fetch-source.js';

function mockFetchOk(data: unknown): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve(data),
  }) as unknown as typeof globalThis.fetch;
}

@customElement('test-adapter-host')
class TestHost extends LitElement {
  readonly dataSource = new DataSourceAdapter(this, {
    sourceFactory: (url, _id) => fetchSource(url, {
      fetchFn: (this as any)._mockFetch ?? mockFetchOk([]),
    }),
  });

  _mockFetch?: typeof globalThis.fetch;
}

@customElement('test-dual-adapter')
class DualHost extends LitElement {
  readonly primary = new DataSourceAdapter(this);
  readonly secondary = new DataSourceAdapter(this);
}

async function flush(): Promise<void> {
  await new Promise(r => setTimeout(r, 20));
}

describe('DataSourceAdapter', () => {
  let el: TestHost;

  afterEach(() => {
    el?.remove();
    document.querySelectorAll('test-adapter-host, test-dual-adapter').forEach(e => e.remove());
  });

  it('connects controller on hostConnected', async () => {
    el = document.createElement('test-adapter-host') as TestHost;
    el._mockFetch = mockFetchOk([{ id: 1 }]);
    document.body.appendChild(el);
    await el.updateComplete;

    el.dataSource.endpoint = '/api/items';
    await flush();
    expect(el.dataSource.dataSet).toEqual([{ id: 1 }]);
  });

  it('disconnects on removal from DOM', async () => {
    el = document.createElement('test-adapter-host') as TestHost;
    el._mockFetch = mockFetchOk([]);
    document.body.appendChild(el);
    await el.updateComplete;

    el.dataSource.endpoint = '/api/items';
    el.remove();
    await flush();
    // No error — disconnect handled gracefully
  });

  it('triggers host requestUpdate on data arrival', async () => {
    el = document.createElement('test-adapter-host') as TestHost;
    el._mockFetch = mockFetchOk([{ id: 1 }]);
    const spy = vi.spyOn(el, 'requestUpdate');
    document.body.appendChild(el);
    await el.updateComplete;

    el.dataSource.endpoint = '/api/items';
    await flush();
    expect(spy).toHaveBeenCalled();
  });

  it('proxies loading/error/dataSet to controller', async () => {
    el = document.createElement('test-adapter-host') as TestHost;
    document.body.appendChild(el);
    await el.updateComplete;

    el.dataSource.loading = true;
    expect(el.dataSource.loading).toBe(true);
    expect(el.dataSource.controller.loading).toBe(true);

    el.dataSource.error = 'fail';
    expect(el.dataSource.error).toBe('fail');
    expect(el.dataSource.loading).toBe(false); // mutual-clearing

    el.dataSource.dataSet = { items: [] };
    expect(el.dataSource.dataSet).toEqual({ items: [] });
    expect(el.dataSource.error).toBe(''); // mutual-clearing
  });

  it('refresh delegates to controller', async () => {
    el = document.createElement('test-adapter-host') as TestHost;
    const fetchFn = mockFetchOk([]);
    el._mockFetch = fetchFn;
    document.body.appendChild(el);
    await el.updateComplete;

    el.dataSource.endpoint = '/api/items';
    await flush();

    (fetchFn as ReturnType<typeof vi.fn>).mockClear();
    el.dataSource.refresh();
    await flush();
    // refresh disconnects + reconnects source — new fetch
    expect(fetchFn).toHaveBeenCalled();
  });

  it('multiple adapters on one host both get lifecycle', async () => {
    const dual = document.createElement('test-dual-adapter') as DualHost;
    document.body.appendChild(dual);
    await dual.updateComplete;

    // Both adapters connected — setting dataSet works on each independently
    dual.primary.dataSet = 'primary-data';
    dual.secondary.dataSet = 'secondary-data';
    expect(dual.primary.dataSet).toBe('primary-data');
    expect(dual.secondary.dataSet).toBe('secondary-data');

    dual.remove();
    // Both disconnected — no errors
  });

  it('exposes controller for escape hatch', () => {
    el = document.createElement('test-adapter-host') as TestHost;
    expect(el.dataSource.controller).toBeDefined();
    expect(el.dataSource.controller.constructor.name).toBe('DataSourceController');
  });
});
```

- [ ] **Step 2: Run tests — verify all fail**

Run: `yarn vitest run packages/blocks-ui-core/src/data-source/data-source-adapter.test.ts`
Expected: All fail — module not found.

- [ ] **Step 3: Implement DataSourceAdapter**

Create `packages/blocks-ui-core/src/data-source/data-source-adapter.ts`:

```typescript
import type { ReactiveController, ReactiveControllerHost } from "lit";
import {
  DataSourceController,
  type DataSourceControllerOptions,
} from "@casehubio/pages-component";

export class DataSourceAdapter implements ReactiveController {
  readonly controller: DataSourceController;

  constructor(
    private readonly host: ReactiveControllerHost,
    options?: DataSourceControllerOptions,
  ) {
    this.controller = new DataSourceController({
      ...options,
      onChange: () => {
        options?.onChange?.();
        host.requestUpdate();
      },
    });
    host.addController(this);
  }

  get endpoint(): string | undefined { return this.controller.endpoint; }
  set endpoint(v: string | undefined) { this.controller.endpoint = v; }

  get loading(): boolean { return this.controller.loading; }
  set loading(v: boolean) { this.controller.loading = v; }
  get error(): string { return this.controller.error; }
  set error(v: string) { this.controller.error = v; }
  get dataSet(): unknown { return this.controller.dataSet; }
  set dataSet(v: unknown) { this.controller.dataSet = v; }

  get source() { return this.controller.source; }
  set source(s) { this.controller.source = s; }

  hostConnected(): void { this.controller.connect(); }
  hostDisconnected(): void { this.controller.disconnect(); }

  refresh(): void { this.controller.refresh(); }
  dispose(): void { this.controller.dispose(); }
}
```

- [ ] **Step 4: Run tests — verify all pass**

Run: `yarn vitest run packages/blocks-ui-core/src/data-source/data-source-adapter.test.ts`
Expected: All 7 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/data-source/data-source-adapter.ts packages/blocks-ui-core/src/data-source/data-source-adapter.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(core): DataSourceAdapter — Lit ReactiveController for DataSourceController #44"
```

---

### Task 3: DataSourceMixin — convenience mixin

**Files:**
- Create: `packages/blocks-ui-core/src/data-source/data-source-mixin.ts`
- Create: `packages/blocks-ui-core/src/data-source/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts` — swap `data-endpoint` → `data-source`
- Test: `packages/blocks-ui-core/src/data-source/data-source-mixin.test.ts`

**Interfaces:**
- Consumes: `DataSourceAdapter` (Task 2), `fetchSource` (Task 1),
  `SourceFactory` from `@casehubio/pages-component`
- Produces: `DataSourceMixin(Base)` function — `endpoint` (@property),
  `loading`/`error`/`dataSet` (get/set), `dataSource` (DataSourceAdapter),
  `resolveEndpoint()`, `syncEndpoint()`, `configure(props)`

- [ ] **Step 1: Write failing tests for DataSourceMixin**

Create `packages/blocks-ui-core/src/data-source/data-source-mixin.test.ts`:

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import { LitElement, html } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { DataSourceMixin } from './data-source-mixin.js';
import { fetchSource } from './fetch-source.js';
import type { SourceFactory } from '@casehubio/pages-component';

function mockFetchOk(data: unknown): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve(data),
  }) as unknown as typeof globalThis.fetch;
}

function mockFetchFail(status: number): typeof globalThis.fetch {
  return vi.fn().mockResolvedValue({
    ok: false,
    status,
  }) as unknown as typeof globalThis.fetch;
}

let testFetchFn: typeof globalThis.fetch = mockFetchOk([]);

@customElement('test-mixin-simple')
class SimpleMixinHost extends DataSourceMixin(LitElement) {
  protected override createSourceFactory(): SourceFactory {
    return (url, _id) => fetchSource(url, { fetchFn: testFetchFn });
  }
}

@customElement('test-mixin-derived')
class DerivedMixinHost extends DataSourceMixin(LitElement) {
  @property({ type: String }) itemId?: string;

  protected override createSourceFactory(): SourceFactory {
    return (url, _id) => fetchSource(url, { fetchFn: testFetchFn });
  }

  protected override resolveEndpoint(): string | undefined {
    if (!this.endpoint || !this.itemId) return undefined;
    return `${this.endpoint}/items/${this.itemId}`;
  }

  override willUpdate(changed: Map<string, unknown>): void {
    super.willUpdate(changed);
    if (changed.has('itemId')) this.syncEndpoint();
  }

  override configure(props: Record<string, unknown>): void {
    if (props.itemId !== undefined) this.itemId = props.itemId as string;
    super.configure(props);
  }
}

async function flush(): Promise<void> {
  await new Promise(r => setTimeout(r, 20));
}

describe('DataSourceMixin', () => {
  afterEach(() => {
    document.querySelectorAll('test-mixin-simple, test-mixin-derived').forEach(e => e.remove());
    testFetchFn = mockFetchOk([]);
  });

  describe('simple endpoint', () => {
    it('fetches when endpoint is set as property', async () => {
      const fetchFn = mockFetchOk([{ id: 1 }]);
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      el.endpoint = '/api/items';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();
      expect(el.dataSet).toEqual([{ id: 1 }]);
    });

    it('does not fetch when endpoint is undefined', async () => {
      const fetchFn = mockFetchOk([]);
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();
      expect(fetchFn).not.toHaveBeenCalled();
    });

    it('re-fetches when endpoint changes', async () => {
      const fetchFn = mockFetchOk([]);
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      el.endpoint = '/api/a';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();

      (fetchFn as ReturnType<typeof vi.fn>).mockClear();
      el.endpoint = '/api/b';
      await el.updateComplete;
      await flush();
      expect(fetchFn).toHaveBeenCalledWith('/api/b', expect.anything());
    });

    it('sets loading during fetch', async () => {
      let resolveJson!: (v: any) => void;
      testFetchFn = vi.fn().mockResolvedValue({
        ok: true,
        json: () => new Promise(r => { resolveJson = r; }),
      }) as unknown as typeof globalThis.fetch;

      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      el.endpoint = '/api/items';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();

      expect(el.loading).toBe(true);
      resolveJson([]);
      await flush();
      expect(el.loading).toBe(false);
    });

    it('sets error on fetch failure', async () => {
      testFetchFn = mockFetchFail(500);
      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      el.endpoint = '/api/items';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();
      expect(el.error).toContain('500');
    });

    it('mutual-clearing: dataSet clears error', async () => {
      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      document.body.appendChild(el);
      await el.updateComplete;

      el.error = 'something broke';
      expect(el.error).toBe('something broke');

      el.dataSet = { items: [] };
      expect(el.error).toBe('');
      expect(el.dataSet).toEqual({ items: [] });
    });

    it('DataReceiver setters delegate to controller', async () => {
      const el = document.createElement('test-mixin-simple') as SimpleMixinHost;
      document.body.appendChild(el);
      await el.updateComplete;

      el.loading = true;
      expect(el.dataSource.controller.loading).toBe(true);

      el.dataSet = 'pushed-data';
      expect(el.dataSource.controller.dataSet).toBe('pushed-data');
      expect(el.loading).toBe(false); // mutual-clearing

      el.error = 'push-error';
      expect(el.dataSource.controller.error).toBe('push-error');
    });
  });

  describe('resolveEndpoint', () => {
    it('derives URL from endpoint + itemId', async () => {
      const fetchFn = mockFetchOk({ name: 'thing' });
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-derived') as DerivedMixinHost;
      el.endpoint = '/api';
      el.itemId = '42';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();

      expect(fetchFn).toHaveBeenCalledWith('/api/items/42', expect.anything());
    });

    it('does not fetch when itemId is missing', async () => {
      const fetchFn = mockFetchOk({});
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-derived') as DerivedMixinHost;
      el.endpoint = '/api';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();

      expect(fetchFn).not.toHaveBeenCalled();
    });

    it('re-fetches when itemId changes', async () => {
      const fetchFn = mockFetchOk({});
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-derived') as DerivedMixinHost;
      el.endpoint = '/api';
      el.itemId = '1';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();

      (fetchFn as ReturnType<typeof vi.fn>).mockClear();
      el.itemId = '2';
      await el.updateComplete;
      await flush();
      expect(fetchFn).toHaveBeenCalledWith('/api/items/2', expect.anything());
    });
  });

  describe('configure()', () => {
    it('batches endpoint + custom props — single fetch', async () => {
      const fetchFn = mockFetchOk({ name: 'configured' });
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-derived') as DerivedMixinHost;
      document.body.appendChild(el);
      await el.updateComplete;

      el.configure({ endpoint: '/api', itemId: '99' });
      await flush();
      await el.updateComplete;
      await flush();

      // Only one fetch — not two (one for endpoint, one for itemId)
      const calls = (fetchFn as ReturnType<typeof vi.fn>).mock.calls;
      const itemCalls = calls.filter((c: any) => c[0] === '/api/items/99');
      expect(itemCalls.length).toBeGreaterThanOrEqual(1);
    });

    it('syncEndpoint suppressed during configure', async () => {
      const fetchFn = mockFetchOk({});
      testFetchFn = fetchFn;
      const el = document.createElement('test-mixin-derived') as DerivedMixinHost;
      el.endpoint = '/api';
      el.itemId = '1';
      document.body.appendChild(el);
      await el.updateComplete;
      await flush();

      (fetchFn as ReturnType<typeof vi.fn>).mockClear();
      el.configure({ endpoint: '/api2', itemId: '2' });

      // syncEndpoint from willUpdate should be suppressed
      await el.updateComplete;
      await flush();

      // Verify we got the right derived URL, not partial
      const lastCall = (fetchFn as ReturnType<typeof vi.fn>).mock.calls.slice(-1)[0];
      expect(lastCall[0]).toBe('/api2/items/2');
    });
  });
});
```

- [ ] **Step 2: Run tests — verify all fail**

Run: `yarn vitest run packages/blocks-ui-core/src/data-source/data-source-mixin.test.ts`
Expected: All fail — module not found.

- [ ] **Step 3: Implement DataSourceMixin**

Create `packages/blocks-ui-core/src/data-source/data-source-mixin.ts`:

```typescript
import type { LitElement, PropertyValues } from "lit";
import { property } from "lit/decorators.js";
import { DataSourceAdapter } from "./data-source-adapter.js";
import { fetchSource } from "./fetch-source.js";
import type { SourceFactory } from "@casehubio/pages-component";

type Constructor<T = {}> = new (...args: any[]) => T;

export function DataSourceMixin<T extends Constructor<LitElement>>(Base: T) {
  class DataSourceHost extends Base {
    protected createSourceFactory(): SourceFactory {
      return (url, _id) => fetchSource(url);
    }

    readonly dataSource: DataSourceAdapter = new DataSourceAdapter(this, {
      sourceFactory: this.createSourceFactory(),
    });

    @property({ type: String }) endpoint?: string;

    get loading(): boolean { return this.dataSource.loading; }
    set loading(v: boolean) { this.dataSource.loading = v; }
    get error(): string { return this.dataSource.error; }
    set error(v: string) { this.dataSource.error = v; }
    get dataSet(): unknown { return this.dataSource.dataSet; }
    set dataSet(v: unknown) { this.dataSource.dataSet = v; }

    protected resolveEndpoint(): string | undefined {
      return this.endpoint;
    }

    private _configuring = false;

    protected syncEndpoint(): void {
      if (this._configuring) return;
      this.dataSource.endpoint = this.resolveEndpoint();
    }

    override willUpdate(changed: PropertyValues): void {
      super.willUpdate(changed);
      if (changed.has("endpoint")) {
        this.syncEndpoint();
      }
    }

    configure(props: Record<string, unknown>): void {
      this._configuring = true;
      if (props.endpoint !== undefined) this.endpoint = props.endpoint as string;
      queueMicrotask(() => {
        this._configuring = false;
        this.syncEndpoint();
        this.dataSource.refresh();
      });
    }
  }

  return DataSourceHost as unknown as Constructor<{
    endpoint?: string;
    loading: boolean;
    error: string;
    dataSet: unknown;
    dataSource: DataSourceAdapter;
    resolveEndpoint(): string | undefined;
    syncEndpoint(): void;
    configure(props: Record<string, unknown>): void;
  }> & T;
}
```

- [ ] **Step 4: Create barrel export and update main index**

Create `packages/blocks-ui-core/src/data-source/index.ts`:

```typescript
export { fetchSource, type FetchSourceOptions } from "./fetch-source.js";
export { DataSourceAdapter } from "./data-source-adapter.js";
export { DataSourceMixin } from "./data-source-mixin.js";
```

Update `packages/blocks-ui-core/src/index.ts` — replace the data-endpoint
export with data-source:

```
// Change: export * from './data-endpoint/index.js';
// To:     export * from './data-source/index.js';
```

- [ ] **Step 5: Run tests — verify all pass**

Run: `yarn vitest run packages/blocks-ui-core/src/data-source/data-source-mixin.test.ts`
Expected: All 11 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/data-source/data-source-mixin.ts packages/blocks-ui-core/src/data-source/data-source-mixin.test.ts packages/blocks-ui-core/src/data-source/index.ts packages/blocks-ui-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(core): DataSourceMixin — convenience mixin with endpoint resolution and configure batching #44"
```

---

### Task 4: Migrate consumers + delete DataEndpointMixin

**Files:**
- Modify: `components/list-pane/src/list-pane.ts`
- Modify: `components/list-pane/src/list-pane.test.ts`
- Modify: `components/trust-score-panel/src/trust-score-panel.ts`
- Modify: `components/trust-score-panel/src/trust-score-panel.test.ts`
- Modify: `components/case-timeline/src/case-timeline.ts`
- Modify: `components/case-timeline/src/case-timeline.test.ts`
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.ts`
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.test.ts`
- Delete: `packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts`
  (use `ide_refactor_safe_delete`)
- Delete: `packages/blocks-ui-core/src/data-endpoint/data-endpoint.test.ts`
  (use `ide_refactor_safe_delete`)
- Delete: `packages/blocks-ui-core/src/data-endpoint/index.ts`
  (use `ide_refactor_safe_delete`)

**Interfaces:**
- Consumes: `DataSourceMixin` (Task 3), `DataSourceAdapter` (Task 2),
  `fetchSource` (Task 1)
- Produces: Migrated components with identical public API (endpoint, loading,
  error, configure) but backed by DataSourceController

This task is large but atomic — all four consumers must migrate together
because DataEndpointMixin is deleted. Splitting would leave broken imports.

- [ ] **Step 1: Migrate list-pane**

Rewrite `components/list-pane/src/list-pane.ts`:
- Change import: `DataEndpointMixin` → `DataSourceMixin`
- Change class: `extends DataEndpointMixin(LitElement)` → `extends DataSourceMixin(LitElement)`
- Override `createSourceFactory()` to inject test fetchFn
- Delete `fetchData()` method
- Add `_lastDataSet` tracking + `_rows`/`_totalRows` derivation in `willUpdate`
- Change `refresh()` to call `this.dataSource.refresh()` then re-configure
  via `this.configure({ endpoint: this.endpoint })`
- Remove `fetchFn`, `abortSignal` usage from render

Use `ide_edit_member` / `ide_replace_member` / `ide_insert_member` for
all class member changes.

- [ ] **Step 2: Update list-pane tests**

Update `components/list-pane/src/list-pane.test.ts`:
- Update `ListPaneEl` type — remove `fetchFn`, `error: string | null` → `error: string`
- Change `createElement()` helper: instead of `el.fetchFn = mockFetch()`,
  override the module-level fetchFn that the sourceFactory closure captures.
  Pattern: set a module-level `testFetchFn` variable, then use it in
  `createSourceFactory()` inside the component — OR mock at the adapter level.
  Simplest: set `globalThis.fetch` in tests (already done in fetchSource tests).
- Verify existing test assertions still pass against the new component shape.

- [ ] **Step 3: Run list-pane tests**

Run: `yarn vitest run components/list-pane/src/list-pane.test.ts`
Expected: All tests PASS.

- [ ] **Step 4: Migrate trust-score-panel**

Rewrite `components/trust-score-panel/src/trust-score-panel.ts`:
- Change import: `DataEndpointMixin` → `DataSourceMixin`
- Change class: `extends LiveRegionMixin(DataEndpointMixin(LitElement))` →
  `extends DataSourceMixin(LiveRegionMixin(LitElement))`
- Override `resolveEndpoint()` — returns `undefined` when `_hasPreFetchedData()`,
  otherwise derives `${this.endpoint}/trust/${this.actorId}`
- Override `willUpdate` — call `this.syncEndpoint()` when `actorId` changes
- Delete `fetchData()` method
- Read `this.dataSet as TrustScoreResponse` instead of `this._trustData` in
  `_getDisplayScore()` and `_getDisplayTrustLevel()`.
  Keep `_hasPreFetchedData()` — when true, render from props.

- [ ] **Step 5: Update trust-score-panel tests + run**

Update tests to use the new data flow (dataSet instead of _trustData).
Run: `yarn vitest run components/trust-score-panel/src/trust-score-panel.test.ts`
Expected: All tests PASS.

- [ ] **Step 6: Migrate case-timeline**

Rewrite `components/case-timeline/src/case-timeline.ts`:
- Change import: `DataEndpointMixin` → `DataSourceMixin`
- Change class: `extends LiveRegionMixin(DataEndpointMixin(LitElement))` →
  `extends DataSourceMixin(LiveRegionMixin(LitElement))`
- Declare `@property({ type: Object }) identity?: WorkIdentity` on the component
- Override `createSourceFactory()` — returns fetchSource with dynamic tenancy headers
- Override `resolveEndpoint()` — derives `${this.endpoint}/cases/${this.caseId}/events`
- Override `configure()` — propagates `caseId` and `identity` before `super.configure()`
- Override `willUpdate` — call `this.syncEndpoint()` when `caseId` changes
- Delete `fetchData()` method
- Read events from `this.dataSet` — extract `.content` for the PagedResponse shape

- [ ] **Step 7: Update case-timeline tests + run**

Run: `yarn vitest run components/case-timeline/src/case-timeline.test.ts`
Expected: All tests PASS.

- [ ] **Step 8: Migrate audit-trail-viewer**

Rewrite `components/audit-trail-viewer/src/audit-trail-viewer.ts`:
- Change import: remove `DataEndpointMixin`, add `DataSourceAdapter`, `fetchSource`
- Change class: `extends LiveRegionMixin(DataEndpointMixin(LitElement))` →
  `extends LiveRegionMixin(LitElement)`
- Create two adapters: `entries` and `verify` as `DataSourceAdapter` instances
- Declare `@property() endpoint`, `@property() identity` on the component
- Add `_updateEndpoints()` — builds full URLs with query params
- Add `configure()` — propagates all props, defers via microtask
- Override `willUpdate` — call `_updateEndpoints()` when relevant props change
- Delete `fetchData()` method
- Update `render()` — read entries from `this.entries.dataSet`, verification from
  `this.verify.dataSet`. Handle verify loading/error independently.
- Delete `_fetchAttestations()` — inline fetch stays (it's a secondary action
  triggered by row expand, not part of the main data lifecycle)

- [ ] **Step 9: Update audit-trail-viewer tests + run**

Run: `yarn vitest run components/audit-trail-viewer/src/audit-trail-viewer.test.ts`
Expected: All tests PASS.

- [ ] **Step 10: Delete DataEndpointMixin**

Use `ide_refactor_safe_delete` on each file:
- `packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts`
- `packages/blocks-ui-core/src/data-endpoint/data-endpoint.test.ts`
- `packages/blocks-ui-core/src/data-endpoint/index.ts`

Then remove the directory if empty:
```bash
rmdir /Users/mdproctor/claude/casehub/blocks-ui/packages/blocks-ui-core/src/data-endpoint 2>/dev/null
```

- [ ] **Step 11: Full build + typecheck + test**

```bash
yarn --cwd /Users/mdproctor/claude/casehub/blocks-ui build
yarn --cwd /Users/mdproctor/claude/casehub/blocks-ui typecheck
yarn --cwd /Users/mdproctor/claude/casehub/blocks-ui test
```

All three must pass.

- [ ] **Step 12: Update docs**

Update `CLAUDE.md` — change `DataEndpointMixin` references in the
blocks-ui-core row to `DataSourceMixin, DataSourceAdapter, fetchSource`.

Update `README.md` — change `DataEndpointMixin` references to `DataSourceMixin`.

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add -A
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: migrate DataEndpointMixin → DataSourceMixin + DataSourceAdapter #44

- list-pane, trust-score-panel, case-timeline: DataSourceMixin
- audit-trail-viewer: dual DataSourceAdapter (entries + verify)
- DataEndpointMixin deleted (pre-release, no deprecation)
- fetchSource for raw JSON domain components
- Tests updated for new data flow"
```
