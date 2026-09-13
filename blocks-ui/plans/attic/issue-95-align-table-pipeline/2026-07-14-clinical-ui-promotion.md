# Clinical UI Promotion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #38 — epic: clinical trial UI — blocks-ui migration plan
**Issue group:** #38, #58, #59, #60, #61, #62, #63

**Goal:** Promote 5 clinical-local Lit web components to blocks-ui as shared platform components, plus add a `commitmentLifecycleStrategy()` factory to blocks-timeline.

**Architecture:** Foundation-first. Add `createTypedFetchSource` utility and `EMPTY_DATASET` to blocks-ui-core, move shared types (TrustLevel) to core, extract shared pulse animation. Then build each component/strategy on the foundation. Components use DataSourceMixin for data lifecycle (except gdpr-erasure-action which extends LitElement directly for mutation-only use). Tabular components render via pages-table; non-tabular components store typed data privately.

**Tech Stack:** Lit 3, TypeScript 5.6, vitest, pages-data (fromRows, TypedDataSet, columnId, ColumnType), pages-table (TableColumnConfig, ColumnRenderer), pages-component (SourceFactory, emitPagesEvent), pages-primitives (FocusTrapMixin via blocks-confirm-dialog)

## Global Constraints

- All CSS colours use `--pages-*` token variables — no hardcoded hex values
- Customisation protocol PP-20260713-8ea1af: typed config properties + render callbacks, no slots for content
- All components: `@customElement()` decorator, `declare global { HTMLElementTagNameMap }`, exported topic constants
- `tsconfig.base.json` settings: `strict`, `exactOptionalPropertyTypes`, `verbatimModuleSyntax`, `noImplicitOverride`
- Package naming: `@casehubio/blocks-ui-<name>` at version `0.2.2`
- `publishConfig.registry`: `https://npm.pkg.github.com`
- IntelliJ MCP mandatory for all code navigation and editing on `.ts` files

---

### Task 1: blocks-ui-core Additions

**Files:**
- Create: `packages/blocks-ui-core/src/data-source/typed-fetch-source.ts`
- Create: `packages/blocks-ui-core/src/data-source/typed-fetch-source.test.ts`
- Create: `packages/blocks-ui-core/src/data-source/empty-dataset.ts`
- Create: `packages/blocks-ui-core/src/styles/animations.ts`
- Create: `packages/blocks-ui-core/src/styles/index.ts`
- Modify: `packages/blocks-ui-core/src/data-source/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`
- Modify: `components/trust-score-panel/src/types.ts` (move TrustLevel/trustLevelFromScore to core)
- Create: `packages/blocks-ui-core/src/types/trust.ts`
- Modify: `packages/blocks-ui-core/src/types/index.ts`
- Test: `packages/blocks-ui-core/src/data-source/typed-fetch-source.test.ts`

**Interfaces:**
- Produces: `createTypedFetchSource<T>(url, handler, options?) → DataSource`
- Produces: `EMPTY_DATASET: TypedDataSet` (zero columns, zero rows)
- Produces: `TrustLevel` type, `trustLevelFromScore(score) → TrustLevel` (moved from trust-score-panel)
- Produces: `pulseAnimation` css template literal

- [ ] **Step 1: Write tests for createTypedFetchSource**

Create `packages/blocks-ui-core/src/data-source/typed-fetch-source.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import type { DataSink } from '@casehubio/pages-data';

// Import will be created in step 3
import { createTypedFetchSource } from './typed-fetch-source.js';

let originalFetch: typeof globalThis.fetch;

describe('createTypedFetchSource', () => {
  let mockFetch: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    originalFetch = globalThis.fetch;
    mockFetch = vi.fn();
    globalThis.fetch = mockFetch as unknown as typeof fetch;
  });

  afterEach(() => {
    globalThis.fetch = originalFetch;
  });

  it('calls handler with parsed JSON and sink on success', async () => {
    const data = { id: 1, name: 'test' };
    mockFetch.mockResolvedValue(new Response(JSON.stringify(data), { status: 200 }));

    const handler = vi.fn();
    const source = createTypedFetchSource<typeof data>('http://test.local/api', handler);

    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    await vi.waitFor(() => expect(handler).toHaveBeenCalled());
    expect(handler).toHaveBeenCalledWith(data, sink, expect.any(AbortSignal));
  });

  it('calls sink.error on HTTP failure', async () => {
    mockFetch.mockResolvedValue(new Response('', { status: 500 }));

    const handler = vi.fn();
    const source = createTypedFetchSource('http://test.local/api', handler);

    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    await vi.waitFor(() => expect(sink.error).toHaveBeenCalled());
    expect(handler).not.toHaveBeenCalled();
    expect(sink.error).toHaveBeenCalledWith(expect.objectContaining({
      message: expect.stringContaining('500'),
      permanent: true,
    }));
  });

  it('calls sink.error on network failure', async () => {
    mockFetch.mockRejectedValue(new Error('Network error'));

    const handler = vi.fn();
    const source = createTypedFetchSource('http://test.local/api', handler);

    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    await vi.waitFor(() => expect(sink.error).toHaveBeenCalled());
    expect(sink.error).toHaveBeenCalledWith(expect.objectContaining({
      message: 'Network error',
      permanent: true,
    }));
  });

  it('aborts fetch on disconnect', async () => {
    let abortSignal: AbortSignal | undefined;
    mockFetch.mockImplementation((_url: string, init?: RequestInit) => {
      abortSignal = init?.signal;
      return new Promise(() => {}); // never resolves
    });

    const source = createTypedFetchSource('http://test.local/api', vi.fn());
    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    await vi.waitFor(() => expect(abortSignal).toBeDefined());
    source.disconnect();
    expect(abortSignal!.aborted).toBe(true);
  });

  it('does not call handler after disconnect', async () => {
    let resolveFetch: (value: Response) => void;
    mockFetch.mockImplementation(() => new Promise((resolve) => { resolveFetch = resolve; }));

    const handler = vi.fn();
    const source = createTypedFetchSource('http://test.local/api', handler);
    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    source.disconnect();
    resolveFetch!(new Response(JSON.stringify({}), { status: 200 }));

    await new Promise(r => setTimeout(r, 10));
    expect(handler).not.toHaveBeenCalled();
  });

  it('does not call sink.error for AbortError after disconnect', async () => {
    mockFetch.mockImplementation((_url: string, init?: RequestInit) => {
      return new Promise((_resolve, reject) => {
        init?.signal?.addEventListener('abort', () => reject(new DOMException('Aborted', 'AbortError')));
      });
    });

    const source = createTypedFetchSource('http://test.local/api', vi.fn());
    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    source.disconnect();
    await new Promise(r => setTimeout(r, 10));
    expect(sink.error).not.toHaveBeenCalled();
  });

  it('passes custom method and headers to fetch', async () => {
    mockFetch.mockResolvedValue(new Response(JSON.stringify({}), { status: 200 }));

    const source = createTypedFetchSource('http://test.local/api', vi.fn(), {
      method: 'POST',
      headers: { 'X-Custom': 'value' },
    });
    const sink: DataSink = { apply: vi.fn(), error: vi.fn() };
    source.connect(sink);

    await vi.waitFor(() => expect(mockFetch).toHaveBeenCalled());
    const [, init] = mockFetch.mock.calls[0]!;
    expect(init.method).toBe('POST');
    expect(init.headers).toEqual(expect.objectContaining({ 'X-Custom': 'value' }));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/blocks-ui-core vitest run src/data-source/typed-fetch-source.test.ts`
Expected: FAIL — module `./typed-fetch-source.js` not found

- [ ] **Step 3: Implement createTypedFetchSource**

Create `packages/blocks-ui-core/src/data-source/typed-fetch-source.ts`:

```typescript
import type { DataSource, DataSink } from '@casehubio/pages-data';

export interface TypedFetchOptions {
  readonly method?: string;
  readonly headers?: Record<string, string>;
}

export function createTypedFetchSource<T>(
  url: string,
  handler: (data: T, sink: DataSink, signal: AbortSignal) => void,
  options?: TypedFetchOptions,
): DataSource {
  let abort: AbortController | undefined;
  return {
    connect(sink: DataSink) {
      abort = new AbortController();
      const signal = abort.signal;
      const init: RequestInit = { signal };
      if (options?.method) init.method = options.method;
      if (options?.headers) init.headers = options.headers;
      globalThis.fetch(url, init)
        .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
        .then((data: T) => {
          if (!signal.aborted) handler(data, sink, signal);
        })
        .catch(err => {
          if (!signal.aborted && err.name !== 'AbortError') {
            sink.error({ message: err instanceof Error ? err.message : String(err), permanent: true });
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

Run: `yarn --cwd packages/blocks-ui-core vitest run src/data-source/typed-fetch-source.test.ts`
Expected: PASS — all 7 tests green

- [ ] **Step 5: Create EMPTY_DATASET**

Create `packages/blocks-ui-core/src/data-source/empty-dataset.ts`:

```typescript
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import type { TypedDataSet } from '@casehubio/pages-data/dist/dataset/types.js';

export const EMPTY_DATASET: TypedDataSet = fromRows([], []);
```

- [ ] **Step 6: Create shared pulse animation**

Create `packages/blocks-ui-core/src/styles/animations.ts`:

```typescript
import { css } from 'lit';

export const pulseAnimation = css`
  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.7; }
  }
  @media (prefers-reduced-motion: reduce) {
    .pulse { animation: none; }
  }
`;
```

Create `packages/blocks-ui-core/src/styles/index.ts`:

```typescript
export { pulseAnimation } from './animations.js';
```

- [ ] **Step 7: Move TrustLevel and trustLevelFromScore to core**

Create `packages/blocks-ui-core/src/types/trust.ts`:

```typescript
export type TrustLevel = 'high' | 'adequate' | 'low' | 'none';

export function trustLevelFromScore(score: number | undefined): TrustLevel {
  if (score === undefined || score === null) return 'none';
  if (score >= 0.7) return 'high';
  if (score >= 0.4) return 'adequate';
  return 'low';
}
```

Use `ide_insert_member` to add `export * from './trust.js';` to `packages/blocks-ui-core/src/types/index.ts`.

Update `components/trust-score-panel/src/types.ts`:
- Remove `TrustLevel` type and `trustLevelFromScore` function
- Add re-export: `export { type TrustLevel, trustLevelFromScore } from '@casehubio/blocks-ui-core';`
- Keep `MaturityPhase`, `maturityFromCount`, `TrustScoreResponse`, `CapabilityScoreResponse` in place

- [ ] **Step 8: Update core index exports**

Use `ide_edit_member` on `packages/blocks-ui-core/src/data-source/index.ts` to add:

```typescript
export { createTypedFetchSource, type TypedFetchOptions } from './typed-fetch-source.js';
export { EMPTY_DATASET } from './empty-dataset.js';
```

Use `ide_edit_member` on `packages/blocks-ui-core/src/index.ts` to add:

```typescript
export * from './styles/index.js';
```

(The types/trust.ts exports are already covered by `export * from './types/index.js'`)

- [ ] **Step 9: Verify build**

Run: `yarn --cwd packages/blocks-ui-core build`
Run: `yarn --cwd components/trust-score-panel build`
Expected: Both build clean — trust-score-panel re-exports work

Run: `yarn --cwd components/trust-score-panel vitest run`
Expected: Existing trust-score-panel tests still pass (regression check)

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/data-source/typed-fetch-source.ts packages/blocks-ui-core/src/data-source/typed-fetch-source.test.ts packages/blocks-ui-core/src/data-source/empty-dataset.ts packages/blocks-ui-core/src/data-source/index.ts packages/blocks-ui-core/src/styles/ packages/blocks-ui-core/src/types/trust.ts packages/blocks-ui-core/src/types/index.ts packages/blocks-ui-core/src/index.ts components/trust-score-panel/src/types.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(core): add createTypedFetchSource, EMPTY_DATASET, shared TrustLevel, pulse animation (#38)"
```

---

### Task 2: commitmentLifecycleStrategy in blocks-timeline

**Files:**
- Create: `components/blocks-timeline/src/strategies/commitment-lifecycle.ts`
- Create: `components/blocks-timeline/src/strategies/commitment-lifecycle.test.ts`
- Modify: `components/blocks-timeline/src/index.ts`

**Interfaces:**
- Consumes: `TimelineStrategy<T>`, `StageConfig`, `NodeStatus`, `TimelineNode` from `../types.js`
- Consumes: `linearResolveStatus` from `./state-progression.js`
- Produces: `commitmentLifecycleStrategy(options?) → TimelineStrategy<CommitmentState>`
- Produces: `COMMITMENT_STAGES: readonly StageConfig[]`
- Produces: `CommitmentState` interface (exported type)

- [ ] **Step 1: Write tests for commitmentLifecycleStrategy**

Create `components/blocks-timeline/src/strategies/commitment-lifecycle.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import {
  commitmentLifecycleStrategy,
  COMMITMENT_STAGES,
  type CommitmentState,
} from './commitment-lifecycle.js';
import { linearResolveStatus } from './state-progression.js';
import type { StageConfig } from '../types.js';

describe('commitmentLifecycleStrategy', () => {
  describe('COMMITMENT_STAGES', () => {
    it('has 4 stages', () => {
      expect(COMMITMENT_STAGES).toHaveLength(4);
    });

    it('defines COMMANDED, ACKNOWLEDGED, DONE, DECLINED', () => {
      expect(COMMITMENT_STAGES.map(s => s.key)).toEqual([
        'COMMANDED', 'ACKNOWLEDGED', 'DONE', 'DECLINED',
      ]);
    });

    it('marks DONE as terminal success', () => {
      expect(COMMITMENT_STAGES.find(s => s.key === 'DONE')!.terminal).toBe('success');
    });

    it('marks DECLINED as terminal failure', () => {
      expect(COMMITMENT_STAGES.find(s => s.key === 'DECLINED')!.terminal).toBe('failure');
    });
  });

  describe('toNodes with default stages', () => {
    it('maps commitment stages to nodes', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        id: 'c1',
        currentStage: 'COMMANDED',
        stages: [],
      });
      expect(nodes).toHaveLength(4);
      expect(nodes.map(n => n.key)).toEqual(['COMMANDED', 'ACKNOWLEDGED', 'DONE', 'DECLINED']);
    });

    it('marks current stage as active for non-terminal', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        id: 'c1',
        currentStage: 'ACKNOWLEDGED',
        stages: [
          { key: 'COMMANDED', status: 'completed', actor: 'sys', timestamp: '2026-01-01T00:00:00Z' },
          { key: 'ACKNOWLEDGED', status: 'active', actor: 'agent-1', timestamp: '2026-01-01T01:00:00Z' },
        ],
      });
      expect(nodes.find(n => n.key === 'ACKNOWLEDGED')!.status).toBe('active');
    });

    it('marks DONE as completed when reached', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        id: 'c1',
        currentStage: 'DONE',
        stages: [
          { key: 'COMMANDED', status: 'completed' },
          { key: 'ACKNOWLEDGED', status: 'completed' },
          { key: 'DONE', status: 'completed' },
        ],
      });
      expect(nodes.find(n => n.key === 'DONE')!.status).toBe('completed');
    });

    it('marks DECLINED as failed when reached', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        id: 'c1',
        currentStage: 'DECLINED',
        stages: [
          { key: 'COMMANDED', status: 'completed' },
          { key: 'DECLINED', status: 'failed' },
        ],
      });
      expect(nodes.find(n => n.key === 'DECLINED')!.status).toBe('failed');
    });

    it('populates actor and timestamp from stages', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        id: 'c1',
        currentStage: 'ACKNOWLEDGED',
        stages: [
          { key: 'COMMANDED', status: 'completed', actor: 'requester', timestamp: '2026-01-01T00:00:00Z' },
        ],
      });
      expect(nodes.find(n => n.key === 'COMMANDED')!.actor).toBe('requester');
      expect(nodes.find(n => n.key === 'COMMANDED')!.timestamp).toBe('2026-01-01T00:00:00Z');
    });
  });

  describe('transformData', () => {
    it('maps CommitmentState to StateData', () => {
      const strategy = commitmentLifecycleStrategy();
      expect(strategy.transformData).toBeDefined();

      const raw: CommitmentState = {
        id: 'c1',
        currentStage: 'ACKNOWLEDGED',
        stages: [
          { key: 'COMMANDED', status: 'completed', actor: 'sys', timestamp: '2026-01-01T00:00:00Z' },
          { key: 'ACKNOWLEDGED', status: 'active', actor: 'agent-1', timestamp: '2026-01-01T01:00:00Z' },
        ],
      };

      const transformed = strategy.transformData!(raw);
      expect(transformed).toEqual({
        currentState: 'ACKNOWLEDGED',
        transitions: [
          { state: 'COMMANDED', actor: 'sys', timestamp: '2026-01-01T00:00:00Z' },
          { state: 'ACKNOWLEDGED', actor: 'agent-1', timestamp: '2026-01-01T01:00:00Z' },
        ],
      });
    });

    it('handles missing stages array', () => {
      const strategy = commitmentLifecycleStrategy();
      const transformed = strategy.transformData!({
        id: 'c1',
        currentStage: 'COMMANDED',
        stages: [],
      });
      expect(transformed).toEqual({
        currentState: 'COMMANDED',
        transitions: [],
      });
    });
  });

  describe('custom stages', () => {
    it('uses custom stage definitions', () => {
      const customStages: StageConfig[] = [
        { key: 'REQUESTED', label: 'Requested' },
        { key: 'IN_PROGRESS', label: 'In Progress' },
        { key: 'COMPLETED', label: 'Completed', terminal: 'success' },
      ];
      const strategy = commitmentLifecycleStrategy({ stages: customStages });
      const nodes = strategy.toNodes({ currentState: 'IN_PROGRESS', transitions: [] });
      expect(nodes).toHaveLength(3);
      expect(nodes[1]!.label).toBe('In Progress');
      expect(nodes[1]!.status).toBe('active');
    });
  });

  describe('defaultLayout', () => {
    it('is horizontal', () => {
      expect(commitmentLifecycleStrategy().defaultLayout).toBe('horizontal');
    });
  });

  describe('uses linearResolveStatus by default', () => {
    it('marks stages before current as completed', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        currentState: 'ACKNOWLEDGED',
        transitions: [],
      });
      expect(nodes.find(n => n.key === 'COMMANDED')!.status).toBe('completed');
    });

    it('marks stages after current as pending', () => {
      const strategy = commitmentLifecycleStrategy();
      const nodes = strategy.toNodes({
        currentState: 'ACKNOWLEDGED',
        transitions: [],
      });
      expect(nodes.find(n => n.key === 'DONE')!.status).toBe('pending');
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd components/blocks-timeline vitest run src/strategies/commitment-lifecycle.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement commitmentLifecycleStrategy**

Create `components/blocks-timeline/src/strategies/commitment-lifecycle.ts`:

```typescript
import type { TimelineStrategy, StageConfig, NodeStatus } from '../types.js';
import { linearResolveStatus } from './state-progression.js';

export const COMMITMENT_STAGES: readonly StageConfig[] = [
  { key: 'COMMANDED', label: 'Commanded' },
  { key: 'ACKNOWLEDGED', label: 'Acknowledged' },
  { key: 'DONE', label: 'Done', terminal: 'success' },
  { key: 'DECLINED', label: 'Declined', terminal: 'failure' },
];

export interface CommitmentState {
  readonly id: string;
  readonly currentStage: string;
  readonly stages: ReadonlyArray<{
    readonly key: string;
    readonly actor?: string;
    readonly timestamp?: string;
    readonly status: string;
  }>;
  readonly messages?: ReadonlyArray<{
    readonly sender: string;
    readonly content: string;
    readonly timestamp: string;
  }>;
}

interface StateData {
  currentState: string;
  transitions?: ReadonlyArray<{ state: string; actor?: string; timestamp?: string }>;
}

type ResolveStatus = (
  stage: StageConfig,
  currentState: string,
  transitions: Array<{ state: string; actor?: string; timestamp?: string }>,
  stages: readonly StageConfig[],
) => NodeStatus;

export function commitmentLifecycleStrategy(options?: {
  stages?: StageConfig[];
  resolveStatus?: ResolveStatus;
}): TimelineStrategy<StateData> {
  const stages = options?.stages ?? COMMITMENT_STAGES;
  const resolve = options?.resolveStatus ?? linearResolveStatus;

  return {
    transformData(raw: unknown): StateData {
      const commitment = raw as CommitmentState;
      return {
        currentState: commitment.currentStage,
        transitions: commitment.stages.map(s => ({
          state: s.key,
          actor: s.actor,
          timestamp: s.timestamp,
        })),
      };
    },
    toNodes(data: StateData) {
      const transitions = data.transitions ? [...data.transitions] : [];
      const transitionMap = new Map(transitions.map(t => [t.state, t]));

      return stages.map(stage => {
        const transition = transitionMap.get(stage.key);
        return {
          key: stage.key,
          label: stage.label,
          status: resolve(stage, data.currentState, transitions as Array<{ state: string; actor?: string; timestamp?: string }>, stages),
          timestamp: transition?.timestamp,
          actor: transition?.actor,
        };
      });
    },
    defaultLayout: 'horizontal',
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd components/blocks-timeline vitest run src/strategies/commitment-lifecycle.test.ts`
Expected: PASS

- [ ] **Step 5: Update blocks-timeline index exports**

Add to `components/blocks-timeline/src/index.ts`:

```typescript
export {
  commitmentLifecycleStrategy,
  COMMITMENT_STAGES,
  type CommitmentState,
} from './strategies/commitment-lifecycle.js';
```

- [ ] **Step 6: Verify build and existing tests**

Run: `yarn --cwd components/blocks-timeline build`
Run: `yarn --cwd components/blocks-timeline vitest run`
Expected: Build clean, all existing + new tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/blocks-timeline/src/strategies/commitment-lifecycle.ts components/blocks-timeline/src/strategies/commitment-lifecycle.test.ts components/blocks-timeline/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(blocks-timeline): add commitmentLifecycleStrategy (#58)"
```

---

### Task 3: similarity-panel Component

**Files:**
- Create: `components/similarity-panel/package.json`
- Create: `components/similarity-panel/tsconfig.json`
- Create: `components/similarity-panel/tsconfig.build.json`
- Create: `components/similarity-panel/vitest.config.ts`
- Create: `components/similarity-panel/src/index.ts`
- Create: `components/similarity-panel/src/types.ts`
- Create: `components/similarity-panel/src/similarity-panel.ts`
- Create: `components/similarity-panel/src/similarity-panel.test.ts`
- Modify: `tsconfig.json` (root — add reference)
- Test: `components/similarity-panel/src/similarity-panel.test.ts`

**Interfaces:**
- Consumes: `DataSourceMixin`, `createTypedFetchSource`, `EMPTY_DATASET`, `emitPagesEvent` from blocks-ui-core
- Consumes: `fromRows`, `columnId`, `ColumnType` from pages-data
- Consumes: `pages-table`, `TableColumnConfig`, `ColumnRenderer` from pages-table
- Produces: `SimilarityPanel` component (`<similarity-panel>`)
- Produces: `Precedent` interface, `ColumnDef` interface
- Produces: `SimilarityPanelTopics.PRECEDENT_SELECTED`

- [ ] **Step 1: Scaffold package**

Create `components/similarity-panel/package.json`:

```json
{
  "name": "@casehubio/blocks-ui-similarity-panel",
  "version": "0.2.2",
  "description": "Similar past cases — similarity scores, outcomes, resolution times via pages-table",
  "repository": { "type": "git", "url": "https://github.com/casehubio/blocks-ui.git" },
  "publishConfig": { "registry": "https://npm.pkg.github.com" },
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
    "@casehubio/pages-data": "^0.2.2",
    "@casehubio/pages-table": "^0.2.2",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

Create `components/similarity-panel/tsconfig.json`:

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

Create `components/similarity-panel/tsconfig.build.json`:

```json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

Create `components/similarity-panel/vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';
import { existsSync } from 'fs';

export default defineConfig({
  resolve: {
    alias: [
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-ui-tokens/src')) ? [{ find: '@casehubio/pages-ui-tokens', replacement: path.resolve(__dirname, '../../../pages/packages/pages-ui-tokens/src') }] : []),
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-component/src')) ? [{ find: '@casehubio/pages-component', replacement: path.resolve(__dirname, '../../../pages/packages/pages-component/src') }] : []),
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-data/src')) ? [{ find: '@casehubio/pages-data/dist/sse/sse-manager.js', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src/sse/sse-manager.ts') }] : []),
      ...(existsSync(path.resolve(__dirname, '../../../pages/packages/pages-data/src')) ? [{ find: '@casehubio/pages-data', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src') }] : []),
    ],
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

- [ ] **Step 2: Create types**

Create `components/similarity-panel/src/types.ts`:

```typescript
export interface Precedent {
  readonly caseId: string;
  readonly similarity: number;
  readonly outcome: string;
  readonly resolutionTime: string;
  readonly [key: string]: unknown;
}

export interface ColumnDef {
  readonly key: string;
  readonly label: string;
}
```

Create `components/similarity-panel/src/index.ts`:

```typescript
export * from './types.js';
export { SimilarityPanel, SimilarityPanelTopics } from './similarity-panel.js';
```

- [ ] **Step 3: Write failing tests**

Create `components/similarity-panel/src/similarity-panel.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import './similarity-panel.js';
import type { Precedent } from './types.js';

type SimilarityPanelEl = HTMLElement & {
  endpoint?: string;
  data: Precedent[] | null;
  emptyMessage: string;
  updateComplete: Promise<boolean>;
};

const SAMPLE_DATA: Precedent[] = [
  { caseId: 'prec-001', similarity: 92, outcome: 'Resolved', resolutionTime: '3 days' },
  { caseId: 'prec-002', similarity: 45, outcome: 'Escalated', resolutionTime: '5 days' },
  { caseId: 'prec-003', similarity: 78, outcome: 'Pending', resolutionTime: '2 days' },
];

describe('similarity-panel', () => {
  let el: SimilarityPanelEl;

  beforeEach(() => {
    el = document.createElement('similarity-panel') as SimilarityPanelEl;
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
  });

  it('renders empty message when no data', async () => {
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('No similar cases found');
  });

  it('renders custom empty message', async () => {
    el.emptyMessage = 'Nothing to show';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Nothing to show');
  });

  it('renders pages-table when data is provided', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table');
    expect(table).toBeTruthy();
  });

  it('emits precedent.selected on row activation', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;

    const handler = vi.fn();
    document.addEventListener('pages-event', handler);

    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    table.dispatchEvent(new CustomEvent('row-activate', {
      bubbles: true,
      detail: { row: { text: (id: string) => id === 'caseId' ? 'prec-001' : 'Resolved', number: () => 92 } },
    }));

    const event = handler.mock.calls.find(
      (c: any) => c[0].detail.topic === 'precedent.selected'
    );
    expect(event).toBeTruthy();
    document.removeEventListener('pages-event', handler);
  });

  it('renders loading state during fetch', async () => {
    const mockFetch = vi.fn(() => new Promise(() => {}));
    globalThis.fetch = mockFetch as any;
    el.endpoint = 'http://test.local/api/precedents';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Loading');
    globalThis.fetch = fetch;
  });

  it('suppresses fetch when data prop is set', async () => {
    const mockFetch = vi.fn();
    globalThis.fetch = mockFetch as any;
    el.data = SAMPLE_DATA;
    el.endpoint = 'http://test.local/api/precedents';
    await el.updateComplete;
    expect(mockFetch).not.toHaveBeenCalled();
    globalThis.fetch = fetch;
  });
});
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `yarn install && yarn --cwd components/similarity-panel vitest run`
Expected: FAIL — `./similarity-panel.js` not found

- [ ] **Step 5: Implement similarity-panel component**

Create `components/similarity-panel/src/similarity-panel.ts` — the full component with DataSourceMixin, dual data mode (resolveEndpoint override, willUpdate fromRows conversion), pages-table rendering, column renderers for similarity bar and outcome badge, emitPagesEvent for row selection, ARIA (delegated to pages-table). All CSS uses `--pages-*` tokens.

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn --cwd components/similarity-panel vitest run`
Expected: PASS

- [ ] **Step 7: Add root tsconfig reference**

Add to `tsconfig.json` (root): `{ "path": "components/similarity-panel/tsconfig.build.json" }`

- [ ] **Step 8: Build and verify**

Run: `yarn --cwd components/similarity-panel build`
Expected: Build clean

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/similarity-panel/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add similarity-panel component (#59)"
```

---

### Task 4: compliance-summary Component

**Files:**
- Create: `components/compliance-summary/` (same structure as similarity-panel)
- Modify: `tsconfig.json` (root — add reference)
- Test: `components/compliance-summary/src/compliance-summary.test.ts`

**Interfaces:**
- Consumes: Same as Task 3 (DataSourceMixin, pages-table, fromRows)
- Produces: `ComplianceSummary` component (`<compliance-summary>`)
- Produces: `RequirementDefinition` interface
- Produces: `ComplianceSummaryTopics.REQUIREMENT_SELECTED`

Same structure as Task 3. Key differences:

- **Types:** `RequirementDefinition { regulation, requirement, mechanism, status: 'MET'|'PARTIAL'|'GAP'|'BREACHED', evidenceUrl? }`
- **Columns:** REGULATION_COL, REQUIREMENT_COL, MECHANISM_COL, STATUS_COL, EVIDENCE_COL
- **Column renderers:** STATUS_COL → colour-coded badge (MET→`--pages-success-*`, PARTIAL→`--pages-warning-*`, GAP→`--pages-orange-*`, BREACHED→`--pages-danger-*`). EVIDENCE_COL → `<a>` link or `—` dash.
- **Event:** `compliance.requirement-selected`

Steps follow Tasks 3 pattern: scaffold → types → failing tests → implement → pass → tsconfig ref → build → commit.

Tests must cover: all 4 status badge colours, evidence link rendering, evidence dash for missing URL, empty state, row click event, endpoint vs data property dual mode, loading state.

Commit message: `feat: add compliance-summary component (#61)`

---

### Task 5: trust-feedback-display Component

**Files:**
- Create: `components/trust-feedback-display/` (same package structure)
- Modify: `tsconfig.json` (root — add reference)
- Test: `components/trust-feedback-display/src/trust-feedback-display.test.ts`

**Interfaces:**
- Consumes: `DataSourceMixin`, `createTypedFetchSource`, `EMPTY_DATASET`, `TrustLevel`, `trustLevelFromScore` from blocks-ui-core
- Produces: `TrustFeedbackDisplay` component (`<trust-feedback-display>`)
- Produces: `GateDecision` interface (exported)

Key design points:

- **GateDecision:** `{ decision, actor, attestation, trustScoreBefore, trustScoreAfter, dimension }` — `actor` not `investigator`
- **Two render modes:** full (card with label/value rows) and compact (inline badges)
- **Dual data mode (non-tabular):** Property primary (`gateDecision`), endpoint secondary (stores as private `_gateDecision`, sends EMPTY_DATASET to sink)
- **Imports TrustLevel from blocks-ui-core** (not from trust-score-panel)
- **Decision badge:** approved → `--pages-success-*`, rejected → `--pages-danger-*`
- **Attestation badge:** endorsed → `--pages-accent-*`, overruled → `--pages-orange-*`
- **Trust delta arrow:** ↑ up (green), ↓ down (red), → neutral (grey)
- **ARIA:** `role="region"` with `aria-label="Gate decision"`
- **No dependencies** on pages-table or pages-data (except via core's EMPTY_DATASET)

Tests must cover: full mode all rows, compact mode inline, decision badge classes, attestation badge classes, trust delta arrows (up/down/neutral), no-data state, `actor` field, property injection suppresses fetch, endpoint fetch stores private data + sends EMPTY_DATASET.

Commit message: `feat: add trust-feedback-display component (#60)`

---

### Task 6: sla-breach-policy Component

**Files:**
- Create: `components/sla-breach-policy/` (same package structure)
- Modify: `tsconfig.json` (root — add reference)
- Test: `components/sla-breach-policy/src/sla-breach-policy.test.ts`

**Interfaces:**
- Consumes: `DataSourceMixin`, `createTypedFetchSource`, `EMPTY_DATASET`, `pulseAnimation` from blocks-ui-core
- Consumes: `@casehubio/blocks-ui-sla-indicator` (for embedded countdown)
- Produces: `SlaBreachPolicy` component (`<sla-breach-policy>`)
- Produces: `TierDefinition` interface

Key design points:

- **TierDefinition:** `{ threshold: number, label, consequence, regulation? }`
- **Package dependencies:** adds `@casehubio/blocks-ui-sla-indicator: "workspace:*"` and `@casehubio/blocks-ui-core: "workspace:*"`
- **Dual data mode (non-tabular):** Property primary (`tiers`), endpoint secondary
- **Active tier calculation:** Based on `timeRemaining` percentage against tier thresholds
- **`deadline` prop:** When set, renders embedded `<sla-indicator .deadline=${this.deadline} compact>`
- **Shared pulse animation:** `import { pulseAnimation } from '@casehubio/blocks-ui-core'` — used on active tier node
- **ARIA:** `role="list"` on tier container, `role="listitem"` on each tier card
- **Prefers-reduced-motion:** Inherited from shared pulseAnimation

Tests must cover: all tiers render, active tier class, threshold percentage display, consequence text, optional regulation, empty state, deadline prop renders embedded sla-indicator, pulse animation applied to active tier, tiers property injection suppresses fetch.

Commit message: `feat: add sla-breach-policy component (#63)`

---

### Task 7: gdpr-erasure-action Component

**Files:**
- Create: `components/gdpr-erasure-action/` (same package structure)
- Modify: `tsconfig.json` (root — add reference)
- Test: `components/gdpr-erasure-action/src/gdpr-erasure-action.test.ts`

**Interfaces:**
- Consumes: `emitPagesEvent` from blocks-ui-core, `blocks-confirm-dialog` from blocks-ui-core
- Produces: `GdprErasureAction` component (`<gdpr-erasure-action>`)
- Produces: `ErasureReceipt` interface
- Produces: `GdprErasureTopics.ERASURE_COMPLETED`

Key design points:

- **Extends LitElement directly** — NOT DataSourceMixin. No read-path fetch.
- **Package dependencies:** `@casehubio/blocks-ui-core: "workspace:*"`, `lit`
- **Three-phase form:** input → confirmation (blocks-confirm-dialog) → receipt
- **blocks-confirm-dialog integration:** `confirmVariant="danger"`, `persistent`, composed message from subject + reason
- **Own mutation state:** `_loading`, `_error`, `_receipt` as `@state()` properties
- **POST:** `fetch(endpoint, { method: 'POST', body: JSON.stringify({ subjectId, reason }) })`
- **ErasureReceipt:** `{ erasureId?, subjectId, reason, status, timestamp, entryCount? }`
- **Customisation:** `subjectLabel` (default "Subject"), `reasonOptions: string[]`
- **ARIA:** `role="alert"` with `aria-live="assertive"` on confirmation warning

Tests must cover: form renders with fields, validation (empty fields rejected), confirmation dialog opens on submit, POST on confirm, receipt renders on success, error state on POST failure, loading disables form, reset returns to input, erasure-completed event, custom subjectLabel in UI text, custom reasonOptions in dropdown, does NOT extend DataSourceMixin (verify no `endpoint` property from mixin).

Commit message: `feat: add gdpr-erasure-action component (#62)`

---

### Task 8: Example Showcase Pages

**Files:**
- Create: `examples/mock-data/commitments.json`
- Create: `examples/mock-data/precedents.json`
- Create: `examples/mock-data/compliance.json`
- Create: `examples/src/pages/commitment-lifecycle-page.ts`
- Create: `examples/src/pages/similarity-panel-page.ts`
- Create: `examples/src/pages/trust-feedback-page.ts`
- Create: `examples/src/pages/compliance-summary-page.ts`
- Create: `examples/src/pages/gdpr-erasure-page.ts`
- Create: `examples/src/pages/sla-breach-policy-page.ts`
- Modify: `examples/src/shell.ts` (NAV array + renderPage switch)
- Modify: `examples/src/main.ts` (6 page imports)

**Interfaces:**
- Consumes: All 5 new component packages + commitmentLifecycleStrategy from blocks-timeline

Each page follows the sla-indicator-page pattern: controls, multiple demo scenarios, event logging.

- [ ] **Step 1: Create mock data files**

`commitments.json` — 4 commitment states: mid-flow, completed, declined, with-messages
`precedents.json` — 5 precedents with varying similarity, outcomes, resolution times
`compliance.json` — 6 requirements with all 4 status values, some with evidence URLs

- [ ] **Step 2: Create 6 page components**

Each page component: `@customElement`, inline mock data or mock fetch, controls, event log. Follow trust-score-page pattern.

- [ ] **Step 3: Register in shell.ts**

Add 6 NAV entries under Components section. Add 6 cases to renderPage switch.

- [ ] **Step 4: Register in main.ts**

Add 6 `await import('./pages/<name>-page.js');` lines.

- [ ] **Step 5: Verify examples dev server**

Run: `yarn examples`
Open each page in browser. Verify: rendering, controls, theme toggle, event logging.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add examples/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(examples): add showcase pages for clinical promotion components (#38)"
```

---

### Task 9: Integration Verification

**Files:**
- No new files

- [ ] **Step 1: Install dependencies**

Run: `yarn install`

- [ ] **Step 2: Full build**

Run: `yarn build`
Expected: All packages build clean

- [ ] **Step 3: Full typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 4: Full test suite**

Run: `yarn test`
Expected: All tests pass (existing + new)

- [ ] **Step 5: IntelliJ diagnostics**

Run `ide_diagnostics` on each new component's main `.ts` file. Expected: no errors.

- [ ] **Step 6: Commit any fixups**

If Steps 1-5 required fixups, commit them:
```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -am "fix: integration fixups for clinical promotion (#38)"
```
