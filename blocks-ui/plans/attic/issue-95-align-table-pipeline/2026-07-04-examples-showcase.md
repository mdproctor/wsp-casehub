# Examples Showcase Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a self-contained Vite dev app at `examples/` that showcases all blocks-ui components with mock data and simulated real-time SSE events.

**Architecture:** Three layers — mock data (JSON files + MockSSESource), mock fetch (intercepts browser fetch with stateful handlers), and gallery shell (Lit component with sidebar nav, hash routing, theme/density toggles). Bootstrap installs mocks synchronously, loads mock data async, then renders the shell.

**Tech Stack:** Vite 6, Lit 3, TypeScript, vitest (for mock layer tests)

## Global Constraints

- `examples/` is NOT a yarn workspace member — it uses Vite `resolve.alias` for sibling packages, not `workspace:*` dependencies
- `examples/package.json` lists only Vite, Lit, and dev tooling as dependencies
- All 6 sibling packages resolved via alias: `@casehubio/blocks-ui-core` → `../packages/blocks-ui-core/src`, `@casehubio/blocks-ui-work-item-inbox` → `../components/work-item-inbox/src`, etc.
- Stub components (case-timeline, channel-activity, trust-score-panel) excluded
- SSE event `type` values use `WorkEventType` short forms (`CREATED`, `ASSIGNED`, etc.) not CloudEvent URIs
- Mock fetch resolves known URLs first, falls through to real fetch for unknown — NOT try-catch (Vite SPA fallback returns 200 for unknown routes)
- Bootstrap order: install mocks synchronously → `await initMockState()` → render shell
- Root `package.json` gets `"examples": "yarn --cwd examples dev"`
- Issue: #17

---

### Task 1: Project Scaffold, Vite Config, and Mock Data

**Files:**
- Create: `examples/package.json`
- Create: `examples/vite.config.ts`
- Create: `examples/tsconfig.json`
- Create: `examples/index.html`
- Create: `examples/mock-data/work-items.json`
- Create: `examples/mock-data/inbox-summary.json`
- Create: `examples/mock-data/queues.json`
- Create: `examples/mock-data/activity.json`
- Create: `examples/mock-data/relations.json`
- Create: `examples/mock-data/sse-script.json`
- Modify: `package.json` (root — add `examples` script)

**Interfaces:**
- Consumes: Nothing
- Produces: A Vite project that starts with `yarn --cwd examples dev`. JSON mock data files importable by later tasks.

- [ ] **Step 1: Create `examples/package.json`**

```json
{
  "name": "blocks-ui-examples",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "test": "vitest run"
  },
  "dependencies": {
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "vite": "^6.0.0",
    "vitest": "^3.0.0",
    "jsdom": "^25.0.0"
  }
}
```

- [ ] **Step 2: Create `examples/vite.config.ts`**

```typescript
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@casehubio/blocks-ui-core': resolve(__dirname, '../packages/blocks-ui-core/src'),
      '@casehubio/blocks-ui-work-item-row': resolve(__dirname, '../components/work-item-row/src'),
      '@casehubio/blocks-ui-work-item-inbox': resolve(__dirname, '../components/work-item-inbox/src'),
      '@casehubio/blocks-ui-work-item-detail': resolve(__dirname, '../components/work-item-detail/src'),
      '@casehubio/blocks-ui-queue-board': resolve(__dirname, '../components/queue-board/src'),
      '@casehubio/blocks-ui-work-item-workbench': resolve(__dirname, '../components/work-item-workbench/src'),
    },
  },
  server: {
    port: 3000,
    open: true,
  },
});
```

- [ ] **Step 3: Create `examples/tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false,
    "isolatedModules": true,
    "skipLibCheck": true,
    "paths": {
      "@casehubio/blocks-ui-core": ["../packages/blocks-ui-core/src"],
      "@casehubio/blocks-ui-work-item-row": ["../components/work-item-row/src"],
      "@casehubio/blocks-ui-work-item-inbox": ["../components/work-item-inbox/src"],
      "@casehubio/blocks-ui-work-item-detail": ["../components/work-item-detail/src"],
      "@casehubio/blocks-ui-queue-board": ["../components/queue-board/src"],
      "@casehubio/blocks-ui-work-item-workbench": ["../components/work-item-workbench/src"]
    }
  },
  "include": ["src"]
}
```

- [ ] **Step 4: Create `examples/index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>blocks-ui Examples</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    html, body { height: 100%; font-family: 'Inter', system-ui, -apple-system, sans-serif; }
    body { background: var(--blocks-neutral-1, #fafafa); color: var(--blocks-neutral-12, #111); }
    #app { height: 100%; }
  </style>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

- [ ] **Step 5: Create all 6 mock data JSON files**

Create `examples/mock-data/work-items.json` with 18 work items across 5 domains. Each item is a full `WorkItemResponse` shape with realistic data. Mix of statuses, priorities, deadlines. Items have labels matching queue patterns (e.g. `{name: "domain", value: "aml"}`).

Create `examples/mock-data/inbox-summary.json` with pre-computed `InboxSummary` matching the items.

Create `examples/mock-data/queues.json` with 5 queues using simple `labelPattern` strings (`domain=aml`, `domain=clinical`, etc.).

Create `examples/mock-data/activity.json` with a template lifecycle sequence (6 events: created → assigned → started → suspended → resumed → completed).

Create `examples/mock-data/relations.json` with parent/child/linked items for 2-3 work items.

Create `examples/mock-data/sse-script.json` with 8-10 scripted events at 5s intervals using `WorkEventType` short forms.

(Full JSON content is too large for the plan — the implementer creates realistic data matching the `WorkItemResponse` type from `@casehubio/blocks-ui-core`. Use the spec's example titles and domain descriptions as a guide.)

- [ ] **Step 6: Add `examples` script to root `package.json`**

Add to the `"scripts"` section of `/Users/mdproctor/claude/casehub/blocks-ui/package.json`:
```json
"examples": "yarn --cwd examples dev"
```

- [ ] **Step 7: Install dependencies and verify Vite starts**

```bash
cd /Users/mdproctor/claude/casehub/blocks-ui/examples && yarn install
```

Create a minimal `examples/src/main.ts`:
```typescript
document.getElementById('app')!.textContent = 'blocks-ui examples loading...';
```

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/examples && npx vite --host 127.0.0.1 &` then `curl -s http://127.0.0.1:3000/ | head -5`
Expected: HTML with "blocks-ui examples" in the title. Kill the server after.

- [ ] **Step 8: Commit**

```bash
git add examples/ package.json
git commit -m "feat(examples): scaffold Vite dev app with mock data files

Self-contained examples app at examples/ with Vite config, workspace
aliases, and 6 mock data JSON files (work items, inbox summary, queues,
activity, relations, SSE script). Not a yarn workspace member.

Refs #17"
```

---

### Task 2: Mock Layer (mock-state, mock-sse, mock-fetch)

**Files:**
- Create: `examples/src/mock/mock-state.ts`
- Create: `examples/src/mock/mock-sse.ts`
- Create: `examples/src/mock/mock-fetch.ts`
- Create: `examples/src/mock/mock-state.test.ts`
- Create: `examples/src/mock/mock-sse.test.ts`
- Create: `examples/vitest.config.ts`

**Interfaces:**
- Consumes: JSON files from `examples/mock-data/` (Task 1), types from `@casehubio/blocks-ui-core` (`WorkItemResponse`, `WorkItemRootResponse`, `InboxSummary`, `WorkItemLifecycleEvent`, `WorkEventType`, `WorkItemStatus`, `BulkRequest`, `BulkItemResult`, `QueueView`, `isActiveStatus`)
- Produces:
  - `MockState` class: `init()`, `getItems()`, `getItem(id)`, `getSummary()`, `getQueues()`, `getQueueItems(queueId)`, `getActivity(itemId)`, `getRelations(itemId)`, `applyAction(itemId, action, body?)`, `onEvent(handler)`, `startScript()`, `stopScript()`
  - `MockSSESource` class: drop-in `EventSource` replacement
  - `installMockFetch(state: MockState)` function: installs `window.fetch` interceptor
  - `initMockState()` async function: loads JSON, creates `MockState`, installs `MockSSESource`, installs mock fetch, starts SSE script

- [ ] **Step 1: Create `examples/vitest.config.ts`**

```typescript
import { defineConfig } from 'vitest/config';
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@casehubio/blocks-ui-core': resolve(__dirname, '../packages/blocks-ui-core/src'),
    },
  },
  test: {
    environment: 'jsdom',
  },
});
```

- [ ] **Step 2: Write failing tests for MockState**

```typescript
// examples/src/mock/mock-state.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { MockState } from './mock-state.js';
import type { WorkItemResponse, QueueView } from '@casehubio/blocks-ui-core';

const mockItems: WorkItemResponse[] = [
  {
    id: 'wi-001', title: 'Test item 1', status: 'PENDING', priority: 'HIGH',
    assigneeId: null, candidateGroups: 'compliance', createdAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(), version: 1, labels: [{ name: 'domain', value: 'aml' }],
    description: null, category: 'compliance', formKey: null, owner: null,
    candidateUsers: null, requiredCapabilities: null, createdBy: 'system',
    delegationDeclineTarget: null, delegationChain: null, priorStatus: null,
    payload: null, resolution: null, claimDeadline: null, expiresAt: null,
    followUpDate: null, assignedAt: null, startedAt: null, completedAt: null,
    suspendedAt: null, confidenceScore: null, callerRef: null, templateId: null,
    outcome: null, permittedOutcomes: null, inputDataSchema: null,
    outputDataSchema: null, excludedUsers: null, scope: null,
    percentComplete: null, statusNote: null,
  } as WorkItemResponse,
];

const mockQueues: QueueView[] = [
  { id: 'q-001', name: 'Compliance Review', labelPattern: 'domain=aml', scope: null },
];

describe('MockState', () => {
  let state: MockState;

  beforeEach(() => {
    state = new MockState(mockItems, mockQueues, [], [], []);
  });

  it('returns all items', () => {
    expect(state.getItems()).toHaveLength(1);
  });

  it('returns item by id', () => {
    expect(state.getItem('wi-001')?.title).toBe('Test item 1');
  });

  it('computes inbox summary', () => {
    const summary = state.getSummary();
    expect(summary.total).toBe(1);
    expect(summary.byStatus['PENDING']).toBe(1);
    expect(summary.byPriority['HIGH']).toBe(1);
  });

  it('matches queue items by label pattern', () => {
    const items = state.getQueueItems('q-001');
    expect(items).toHaveLength(1);
    expect(items[0]!.id).toBe('wi-001');
  });

  it('applies claim action and emits event', () => {
    const handler = vi.fn();
    state.onEvent(handler);
    state.applyAction('wi-001', 'claim', { actor: 'user-1' });
    expect(state.getItem('wi-001')?.status).toBe('ASSIGNED');
    expect(handler).toHaveBeenCalledWith(
      expect.objectContaining({ type: 'ASSIGNED', workItemId: 'wi-001' })
    );
  });

  it('applies complete action', () => {
    state.applyAction('wi-001', 'claim', { actor: 'user-1' });
    state.applyAction('wi-001', 'start', { actor: 'user-1' });
    state.applyAction('wi-001', 'complete', { actor: 'user-1', outcome: 'approved' });
    expect(state.getItem('wi-001')?.status).toBe('COMPLETED');
  });
});
```

- [ ] **Step 3: Implement MockState**

```typescript
// examples/src/mock/mock-state.ts
import type {
  WorkItemResponse, WorkItemRootResponse, InboxSummary,
  WorkItemLifecycleEvent, QueueView, BulkItemResult,
} from '@casehubio/blocks-ui-core';
import { isActiveStatus } from '@casehubio/blocks-ui-core';

type EventHandler = (event: WorkItemLifecycleEvent) => void;

interface ActivityEvent {
  type: string;
  actor: string;
  detail: string;
  relativeHoursFromCreation: number;
}

interface RelationSet {
  workItemId: string;
  parent: { id: string; title: string; status: string } | null;
  children: Array<{ id: string; title: string; status: string }>;
  linked: Array<{ id: string; title: string; status: string }>;
}

interface ScriptEntry {
  delay: number;
  event: WorkItemLifecycleEvent;
}

export class MockState {
  private items: Map<string, WorkItemResponse>;
  private queues: QueueView[];
  private activityTemplate: ActivityEvent[];
  private relations: RelationSet[];
  private script: ScriptEntry[];
  private handlers: Set<EventHandler> = new Set();
  private scriptTimers: ReturnType<typeof setTimeout>[] = [];

  constructor(
    items: WorkItemResponse[],
    queues: QueueView[],
    activityTemplate: ActivityEvent[],
    relations: RelationSet[],
    script: ScriptEntry[],
  ) {
    this.items = new Map(items.map(item => [item.id, { ...item }]));
    this.queues = queues;
    this.activityTemplate = activityTemplate;
    this.relations = relations;
    this.script = script;
  }

  getItems(): WorkItemResponse[] {
    return Array.from(this.items.values());
  }

  getItem(id: string): WorkItemResponse | undefined {
    return this.items.get(id);
  }

  getSummary(): InboxSummary {
    const items = this.getItems();
    const byStatus: Record<string, number> = {};
    const byPriority: Record<string, number> = {};
    let overdue = 0;
    let claimDeadlineBreached = 0;
    const now = Date.now();

    for (const item of items) {
      byStatus[item.status] = (byStatus[item.status] ?? 0) + 1;
      byPriority[item.priority] = (byPriority[item.priority] ?? 0) + 1;
      if (item.expiresAt && new Date(item.expiresAt).getTime() < now && isActiveStatus(item.status)) overdue++;
      if (item.claimDeadline && new Date(item.claimDeadline).getTime() < now && item.status === 'PENDING') claimDeadlineBreached++;
    }

    return { total: items.length, byStatus, byPriority, overdue, claimDeadlineBreached };
  }

  getQueues(): QueueView[] {
    return this.queues;
  }

  getQueueItems(queueId: string): WorkItemResponse[] {
    const queue = this.queues.find(q => q.id === queueId);
    if (!queue) return [];
    const [key, value] = queue.labelPattern.split('=');
    if (!key || !value) return [];
    return this.getItems().filter(item =>
      item.labels.some(l => l.name === key && l.value === value)
    );
  }

  getActivity(itemId: string): WorkItemLifecycleEvent[] {
    const item = this.items.get(itemId);
    if (!item) return [];
    const statusOrder = ['PENDING', 'ASSIGNED', 'IN_PROGRESS', 'SUSPENDED', 'COMPLETED'];
    const statusIndex = statusOrder.indexOf(item.status);
    const createdAt = new Date(item.createdAt).getTime();

    return this.activityTemplate
      .filter((_, i) => i <= Math.max(0, statusIndex))
      .map((event, i) => ({
        type: event.type,
        source: 'mock',
        subject: itemId,
        workItemId: itemId,
        status: (statusOrder[i] ?? 'PENDING') as WorkItemResponse['status'],
        occurredAt: new Date(createdAt + event.relativeHoursFromCreation * 3600000).toISOString(),
        actor: event.actor,
        detail: event.detail,
        rationale: null,
        planRef: null,
        outcome: null,
        callerRef: null,
        assigneeId: null,
        resolution: null,
        candidateGroups: null,
      }));
  }

  getRelations(itemId: string): RelationSet | undefined {
    return this.relations.find(r => r.workItemId === itemId);
  }

  applyAction(itemId: string, action: string, body?: Record<string, unknown>): WorkItemResponse | null {
    const item = this.items.get(itemId);
    if (!item) return null;

    const statusMap: Record<string, string> = {
      claim: 'ASSIGNED', start: 'IN_PROGRESS', complete: 'COMPLETED',
      reject: 'REJECTED', suspend: 'SUSPENDED', resume: item.priorStatus ?? 'IN_PROGRESS',
      cancel: 'CANCELLED', release: 'PENDING', delegate: 'DELEGATED',
      escalate: 'ESCALATED', 'accept-delegation': 'ASSIGNED',
      'decline-delegation': 'PENDING', fault: 'FAULTED', obsolete: 'OBSOLETE',
    };

    const eventTypeMap: Record<string, string> = {
      claim: 'ASSIGNED', start: 'STARTED', complete: 'COMPLETED',
      reject: 'REJECTED', suspend: 'SUSPENDED', resume: 'RESUMED',
      cancel: 'CANCELLED', release: 'RELEASED', delegate: 'DELEGATED',
      escalate: 'ESCALATED', 'accept-delegation': 'DELEGATION_ACCEPTED',
      'decline-delegation': 'DELEGATION_DECLINED', fault: 'FAULTED', obsolete: 'OBSOLETE',
    };

    const newStatus = statusMap[action];
    if (!newStatus) return null;

    const updated: WorkItemResponse = {
      ...item,
      priorStatus: item.status,
      status: newStatus as WorkItemResponse['status'],
      assigneeId: action === 'claim' ? (body?.actor as string ?? item.assigneeId) : item.assigneeId,
      outcome: action === 'complete' ? (body?.outcome as string ?? null) : item.outcome,
      resolution: action === 'complete' ? (body?.resolution as string ?? null) : item.resolution,
      updatedAt: new Date().toISOString(),
    };

    this.items.set(itemId, updated);

    const event: WorkItemLifecycleEvent = {
      type: eventTypeMap[action] ?? action.toUpperCase(),
      source: 'mock',
      subject: itemId,
      workItemId: itemId,
      status: updated.status,
      occurredAt: new Date().toISOString(),
      actor: (body?.actor as string) ?? 'demo-user',
      detail: null,
      rationale: (body?.reason as string) ?? null,
      planRef: null,
      outcome: updated.outcome,
      callerRef: null,
      assigneeId: updated.assigneeId,
      resolution: updated.resolution,
      candidateGroups: updated.candidateGroups,
    };

    this.emit(event);
    return updated;
  }

  applyBulk(operation: string, itemIds: string[], actorId: string): BulkItemResult[] {
    return itemIds.map(id => {
      const result = this.applyAction(id, operation, { actor: actorId });
      return result
        ? { id, status: 'SUCCESS', error: null }
        : { id, status: 'FAILED', error: `Item ${id} not found` };
    });
  }

  onEvent(handler: EventHandler): () => void {
    this.handlers.add(handler);
    return () => this.handlers.delete(handler);
  }

  private emit(event: WorkItemLifecycleEvent): void {
    for (const handler of this.handlers) handler(event);
  }

  startScript(): void {
    this.runScriptLoop();
  }

  stopScript(): void {
    for (const timer of this.scriptTimers) clearTimeout(timer);
    this.scriptTimers = [];
  }

  private runScriptLoop(): void {
    let cumulative = 0;
    for (const entry of this.script) {
      cumulative += entry.delay;
      const jitter = (Math.random() - 0.5) * 4000; // ±2s
      const timer = setTimeout(() => {
        // Apply the scripted event to state if the item exists
        if (entry.event.workItemId) {
          const item = this.items.get(entry.event.workItemId);
          if (item) {
            const updated = { ...item, status: entry.event.status, updatedAt: new Date().toISOString() };
            this.items.set(entry.event.workItemId, updated);
          }
        }
        this.emit({ ...entry.event, occurredAt: new Date().toISOString() });
      }, cumulative + jitter);
      this.scriptTimers.push(timer);
    }
    // Loop after all events
    const loopTimer = setTimeout(() => this.runScriptLoop(), cumulative + 5000);
    this.scriptTimers.push(loopTimer);
  }
}
```

- [ ] **Step 4: Run MockState tests, verify pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/examples && npx vitest run`
Expected: All PASS

- [ ] **Step 5: Write MockSSESource test**

```typescript
// examples/src/mock/mock-sse.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { MockSSESource } from './mock-sse.js';

describe('MockSSESource', () => {
  beforeEach(() => MockSSESource.resetAll());
  afterEach(() => MockSSESource.resetAll());

  it('sets readyState to OPEN immediately', () => {
    const source = new MockSSESource('/events');
    expect(source.readyState).toBe(1);
  });

  it('delivers events to all instances with matching URL', () => {
    const h1 = vi.fn();
    const h2 = vi.fn();
    const s1 = new MockSSESource('/events');
    const s2 = new MockSSESource('/events');
    s1.onmessage = h1;
    s2.onmessage = h2;

    MockSSESource.pushEvent('/events', { type: 'ASSIGNED', workItemId: 'wi-1' });
    expect(h1).toHaveBeenCalledOnce();
    expect(h2).toHaveBeenCalledOnce();
  });

  it('does not deliver to closed instances', () => {
    const handler = vi.fn();
    const source = new MockSSESource('/events');
    source.onmessage = handler;
    source.close();

    MockSSESource.pushEvent('/events', { type: 'ASSIGNED', workItemId: 'wi-1' });
    expect(handler).not.toHaveBeenCalled();
  });

  it('does not cross-deliver between URLs', () => {
    const handler = vi.fn();
    const source = new MockSSESource('/events');
    source.onmessage = handler;

    MockSSESource.pushEvent('/other', { type: 'ASSIGNED', workItemId: 'wi-1' });
    expect(handler).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 6: Implement MockSSESource**

```typescript
// examples/src/mock/mock-sse.ts

const instances = new Map<string, Set<MockSSESource>>();

export class MockSSESource {
  readonly url: string;
  readyState: number = 1; // OPEN
  onmessage: ((event: MessageEvent) => void) | null = null;
  onerror: (() => void) | null = null;
  onopen: (() => void) | null = null;

  static readonly CONNECTING = 0;
  static readonly OPEN = 1;
  static readonly CLOSED = 2;

  constructor(url: string) {
    this.url = url;
    let set = instances.get(url);
    if (!set) {
      set = new Set();
      instances.set(url, set);
    }
    set.add(this);
  }

  close(): void {
    this.readyState = 2;
    const set = instances.get(this.url);
    if (set) set.delete(this);
  }

  addEventListener(_type: string, _listener: EventListener): void {
    // Minimal implementation — onmessage is sufficient for SSEManager
  }

  removeEventListener(_type: string, _listener: EventListener): void {}

  dispatchEvent(_event: Event): boolean { return true; }

  static pushEvent(url: string, data: unknown): void {
    const set = instances.get(url);
    if (!set) return;
    const event = new MessageEvent('message', { data: JSON.stringify(data) });
    for (const source of set) {
      if (source.readyState === 1 && source.onmessage) {
        source.onmessage(event);
      }
    }
  }

  static pushToAll(data: unknown): void {
    for (const [url] of instances) {
      MockSSESource.pushEvent(url, data);
    }
  }

  static resetAll(): void {
    for (const set of instances.values()) {
      for (const source of set) source.readyState = 2;
      set.clear();
    }
    instances.clear();
  }
}
```

- [ ] **Step 7: Implement mock-fetch**

```typescript
// examples/src/mock/mock-fetch.ts
import type { MockState } from './mock-state.js';

const realFetch = window.fetch.bind(window);

export function installMockFetch(state: MockState): void {
  window.fetch = async (input: RequestInfo | URL, init?: RequestInit): Promise<Response> => {
    const url = typeof input === 'string' ? input : input instanceof URL ? input.toString() : input.url;
    const method = init?.method?.toUpperCase() ?? 'GET';
    const body = init?.body ? JSON.parse(init.body as string) as Record<string, unknown> : undefined;

    const mock = resolveMock(url, method, body, state);
    if (mock) return mock;
    return realFetch(input, init);
  };
}

function resolveMock(
  url: string,
  method: string,
  body: Record<string, unknown> | undefined,
  state: MockState,
): Response | null {
  // Strip origin if present
  const path = url.replace(/^https?:\/\/[^/]+/, '');

  // GET /workitems/inbox/summary
  if (method === 'GET' && path.match(/\/workitems\/inbox\/summary/)) {
    return json(state.getSummary());
  }

  // GET /workitems/inbox
  if (method === 'GET' && path.match(/\/workitems\/inbox/)) {
    const items = state.getItems().map(item => ({
      item, childCount: 0, completedCount: null, requiredCount: null, groupStatus: null,
    }));
    return json(items);
  }

  // POST /workitems/bulk
  if (method === 'POST' && path.match(/\/workitems\/bulk$/)) {
    if (!body) return json([]);
    const results = state.applyBulk(
      body.operation as string,
      body.workItemIds as string[],
      body.actorId as string,
    );
    return json(results);
  }

  // PUT /workitems/{id}/{action}
  const actionMatch = path.match(/\/workitems\/([^/]+)\/(claim|start|complete|reject|delegate|escalate|suspend|resume|cancel|release|accept-delegation|decline-delegation|fault|obsolete)/);
  if (method === 'PUT' && actionMatch) {
    const [, id, action] = actionMatch;
    const params = new URLSearchParams(url.split('?')[1] ?? '');
    const actionBody = { ...body, actor: params.get('actor') ?? params.get('claimant') ?? body?.actor ?? 'demo-user' };
    const result = state.applyAction(id!, action!, actionBody);
    return result ? json(result) : new Response(null, { status: 404 });
  }

  // GET /workitems/{id}/events
  const eventsMatch = path.match(/\/workitems\/([^/]+)\/events$/);
  if (method === 'GET' && eventsMatch) {
    return json(state.getActivity(eventsMatch[1]!));
  }

  // GET /workitems/{id}/relations
  const relMatch = path.match(/\/workitems\/([^/]+)\/relations$/);
  if (method === 'GET' && relMatch) {
    return json(state.getRelations(relMatch[1]!) ?? { parent: null, children: [], linked: [] });
  }

  // GET /workitems/{id}
  const itemMatch = path.match(/\/workitems\/([^/]+)$/);
  if (method === 'GET' && itemMatch) {
    const item = state.getItem(itemMatch[1]!);
    return item ? json(item) : new Response(null, { status: 404 });
  }

  // GET /queues/{id}/items
  const queueItemsMatch = path.match(/\/queues\/([^/]+)\/items$/);
  if (method === 'GET' && queueItemsMatch) {
    return json(state.getQueueItems(queueItemsMatch[1]!));
  }

  // GET /queues/{id}
  const queueMatch = path.match(/\/queues\/([^/]+)$/);
  if (method === 'GET' && queueMatch) {
    return json(state.getQueueItems(queueMatch[1]!));
  }

  // GET /queues
  if (method === 'GET' && path.match(/\/queues$/)) {
    return json(state.getQueues());
  }

  return null;
}

function json(data: unknown): Response {
  return new Response(JSON.stringify(data), {
    status: 200,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

- [ ] **Step 8: Create `initMockState` bootstrap function**

Add to `mock-state.ts` (or a separate `init.ts`):

```typescript
// Add to examples/src/mock/mock-state.ts (export at bottom)
import { MockSSESource } from './mock-sse.js';
import { installMockFetch } from './mock-fetch.js';

export async function initMockState(): Promise<MockState> {
  const [items, queues, activity, relations, script] = await Promise.all([
    fetch('/mock-data/work-items.json').then(r => r.json()),
    fetch('/mock-data/queues.json').then(r => r.json()),
    fetch('/mock-data/activity.json').then(r => r.json()),
    fetch('/mock-data/relations.json').then(r => r.json()),
    fetch('/mock-data/sse-script.json').then(r => r.json()),
  ]);

  const state = new MockState(items, queues, activity, relations, script);

  // Install mock layer AFTER loading data (data was loaded via real fetch)
  (window as any).EventSource = MockSSESource;
  installMockFetch(state);

  // Wire SSE: state events → MockSSESource
  state.onEvent(event => {
    MockSSESource.pushEvent('/workitems/events', event);
  });

  state.startScript();
  return state;
}
```

Note the bootstrap order correction from the spec: `initMockState` loads data via real `fetch` FIRST, then installs mock fetch. This avoids the mock intercepting its own data loads.

- [ ] **Step 9: Run all tests, verify pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/examples && npx vitest run`
Expected: All PASS (MockState: 6 tests, MockSSESource: 4 tests)

- [ ] **Step 10: Commit**

```bash
git add examples/src/mock/ examples/vitest.config.ts
git commit -m "feat(examples): add mock layer — stateful fetch, SSE simulation, event dispatch

MockState: in-memory work item store with action handlers (claim, complete,
delegate, etc.) that mutate state and emit lifecycle events.
MockSSESource: drop-in EventSource replacement delivering events to all
instances matching a URL.
Mock fetch: URL pattern matching interceptor with stateful responses.

Refs #17"
```

---

### Task 3: Gallery Shell and Bootstrap

**Files:**
- Create: `examples/src/shell.ts`
- Modify: `examples/src/main.ts` (replace placeholder with real bootstrap)

**Interfaces:**
- Consumes: `initMockState()` from Task 2, `injectTheme`, `ThemeConfig`, `generateThemeCSS` from `@casehubio/blocks-ui-core`
- Produces: `<example-shell>` Lit element with sidebar nav, hash routing, theme/density toggles. Dispatches `shell-navigate` event when page changes. Renders page components in a content area.

- [ ] **Step 1: Implement shell component**

```typescript
// examples/src/shell.ts
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { generateThemeCSS, type ThemeConfig } from '@casehubio/blocks-ui-core';

interface NavItem {
  id: string;
  label: string;
  hash: string;
}

interface NavCategory {
  label: string;
  items: NavItem[];
}

const NAV: NavCategory[] = [
  {
    label: 'Components',
    items: [
      { id: 'row', label: 'Work Item Row', hash: '#components/row' },
      { id: 'inbox', label: 'Work Item Inbox', hash: '#components/inbox' },
      { id: 'detail', label: 'Work Item Detail', hash: '#components/detail' },
      { id: 'queue', label: 'Queue Board', hash: '#components/queue' },
      { id: 'schema-form', label: 'Schema Form', hash: '#components/schema-form' },
    ],
  },
  {
    label: 'Composed',
    items: [
      { id: 'workbench', label: 'Full Workbench', hash: '#composed/workbench' },
    ],
  },
];

const THEME_CONFIG: ThemeConfig = {
  baseHue: 220,
  accentHue: 250,
  chroma: 0.12,
  contrast: 0.5,
};

@customElement('example-shell')
export class ExampleShell extends LitElement {
  @state() private currentPage = '';
  @state() private theme: 'light' | 'dark' = 'light';
  @state() private density: 'comfortable' | 'compact' = 'comfortable';

  static override styles = css`
    :host { display: flex; height: 100vh; font-family: var(--blocks-font-family, system-ui); }

    .sidebar {
      width: 240px;
      background: var(--blocks-neutral-2, #f5f5f5);
      border-right: 1px solid var(--blocks-neutral-5, #e0e0e0);
      overflow-y: auto;
      flex-shrink: 0;
      display: flex;
      flex-direction: column;
    }

    .sidebar-header {
      padding: 16px;
      font-size: 16px;
      font-weight: 600;
      color: var(--blocks-neutral-12, #111);
      border-bottom: 1px solid var(--blocks-neutral-5, #e0e0e0);
    }

    .category { padding: 12px 0 4px 16px; font-size: 11px; font-weight: 600; text-transform: uppercase; letter-spacing: 0.05em; color: var(--blocks-neutral-9, #888); }

    .nav-item {
      display: block;
      padding: 8px 16px 8px 24px;
      font-size: 14px;
      color: var(--blocks-neutral-11, #555);
      text-decoration: none;
      cursor: pointer;
      border: none;
      background: none;
      width: 100%;
      text-align: left;
    }
    .nav-item:hover { background: var(--blocks-neutral-3, #eee); color: var(--blocks-neutral-12, #111); }
    .nav-item.active { background: var(--blocks-accent-3, #e0e7ff); color: var(--blocks-accent-11, #1e40af); font-weight: 500; }

    .controls { margin-top: auto; padding: 12px 16px; border-top: 1px solid var(--blocks-neutral-5, #e0e0e0); display: flex; gap: 8px; }
    .toggle { padding: 4px 10px; border-radius: 4px; border: 1px solid var(--blocks-neutral-6, #ccc); background: var(--blocks-neutral-1, #fff); cursor: pointer; font-size: 12px; color: var(--blocks-neutral-11, #555); }
    .toggle.active { background: var(--blocks-accent-9, #2563eb); color: white; border-color: var(--blocks-accent-9, #2563eb); }

    .content { flex: 1; overflow: auto; background: var(--blocks-neutral-1, #fafafa); }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.applyTheme();
    this.currentPage = location.hash || '#composed/workbench';
    window.addEventListener('hashchange', this.onHashChange);
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    window.removeEventListener('hashchange', this.onHashChange);
  }

  private onHashChange = (): void => {
    this.currentPage = location.hash;
  };

  private applyTheme(): void {
    const css = generateThemeCSS(THEME_CONFIG);
    let style = document.querySelector('style[data-blocks-theme]') as HTMLStyleElement | null;
    if (!style) {
      style = document.createElement('style');
      style.setAttribute('data-blocks-theme', '');
      document.head.appendChild(style);
    }
    style.textContent = css;
    document.documentElement.className = `blocks-theme-${this.theme}${this.density === 'compact' ? ' blocks-density-compact' : ''}`;
  }

  private toggleTheme(): void {
    this.theme = this.theme === 'light' ? 'dark' : 'light';
    this.applyTheme();
  }

  private toggleDensity(): void {
    this.density = this.density === 'comfortable' ? 'compact' : 'comfortable';
    this.applyTheme();
  }

  override render() {
    return html`
      <nav class="sidebar">
        <div class="sidebar-header">blocks-ui Examples</div>
        ${NAV.map(cat => html`
          <div class="category">${cat.label}</div>
          ${cat.items.map(item => html`
            <button class="nav-item ${this.currentPage === item.hash ? 'active' : ''}"
              @click=${() => { location.hash = item.hash; }}>
              ${item.label}
            </button>
          `)}
        `)}
        <div class="controls">
          <button class="toggle ${this.theme === 'dark' ? 'active' : ''}" @click=${() => this.toggleTheme()}>
            ${this.theme === 'dark' ? 'Dark' : 'Light'}
          </button>
          <button class="toggle ${this.density === 'compact' ? 'active' : ''}" @click=${() => this.toggleDensity()}>
            ${this.density === 'compact' ? 'Compact' : 'Comfortable'}
          </button>
        </div>
      </nav>
      <main class="content">
        <slot name="${this.currentPage}"></slot>
        ${this.renderPage()}
      </main>
    `;
  }

  private renderPage() {
    switch (this.currentPage) {
      case '#components/row': return html`<row-page></row-page>`;
      case '#components/inbox': return html`<inbox-page></inbox-page>`;
      case '#components/detail': return html`<detail-page></detail-page>`;
      case '#components/queue': return html`<queue-page></queue-page>`;
      case '#components/schema-form': return html`<schema-form-page></schema-form-page>`;
      case '#composed/workbench': return html`<workbench-page></workbench-page>`;
      default: return html`<workbench-page></workbench-page>`;
    }
  }
}
```

- [ ] **Step 2: Implement main.ts bootstrap**

```typescript
// examples/src/main.ts
import { initMockState } from './mock/mock-state.js';

async function bootstrap() {
  const app = document.getElementById('app')!;
  app.textContent = 'Loading mock data...';

  await initMockState();

  // Import pages and shell AFTER mocks are installed
  await import('./shell.js');
  await import('./pages/row-page.js');
  await import('./pages/inbox-page.js');
  await import('./pages/detail-page.js');
  await import('./pages/queue-page.js');
  await import('./pages/schema-form-page.js');
  await import('./pages/workbench-page.js');

  app.textContent = '';
  app.appendChild(document.createElement('example-shell'));
}

bootstrap().catch(err => {
  console.error('Failed to bootstrap examples:', err);
  document.getElementById('app')!.textContent = `Error: ${err.message}`;
});
```

- [ ] **Step 3: Verify shell renders** (requires page stubs — create minimal stubs first)

Create minimal page stubs (one line each) so the shell can render without errors:

```typescript
// examples/src/pages/row-page.ts
import { LitElement, html } from 'lit';
import { customElement } from 'lit/decorators.js';
@customElement('row-page')
export class RowPage extends LitElement {
  override render() { return html`<div>Row Page — coming next</div>`; }
}
```

(Same pattern for inbox-page.ts, detail-page.ts, queue-page.ts, schema-form-page.ts, workbench-page.ts — each just renders a placeholder div.)

Start Vite and verify: sidebar renders, clicking items changes hash, theme/density toggles work.

- [ ] **Step 4: Commit**

```bash
git add examples/src/main.ts examples/src/shell.ts examples/src/pages/
git commit -m "feat(examples): add gallery shell with sidebar nav, theme/density toggles

Shell component with collapsible sidebar, hash routing, light/dark theme
toggle, and comfortable/compact density toggle. Bootstrap loads mock data
before rendering any components.

Refs #17"
```

---

### Task 4: Component Demo Pages

**Files:**
- Modify: `examples/src/pages/row-page.ts` (replace stub)
- Modify: `examples/src/pages/inbox-page.ts`
- Modify: `examples/src/pages/detail-page.ts`
- Modify: `examples/src/pages/queue-page.ts`
- Modify: `examples/src/pages/schema-form-page.ts`
- Modify: `examples/src/pages/workbench-page.ts`

**Interfaces:**
- Consumes: All component custom elements (auto-registered via imports), `WorkIdentity` from core, mock data from the installed mock layer (components fetch from `/workitems/inbox` etc. which the mock layer intercepts)

- [ ] **Step 1: Implement row-page**

```typescript
// examples/src/pages/row-page.ts
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import type { WorkItemResponse } from '@casehubio/blocks-ui-core';
import '@casehubio/blocks-ui-work-item-row';

@customElement('row-page')
export class RowPage extends LitElement {
  @state() private items: WorkItemResponse[] = [];

  static override styles = css`
    :host { display: block; padding: 24px; }
    h2 { margin-bottom: 16px; font-size: 20px; font-weight: 600; color: var(--blocks-neutral-12, #111); }
    p { margin-bottom: 16px; color: var(--blocks-neutral-11, #555); font-size: 14px; }
    .rows { max-width: 800px; border: 1px solid var(--blocks-neutral-5, #e0e0e0); border-radius: 6px; overflow: hidden; }
  `;

  override async connectedCallback() {
    super.connectedCallback();
    const resp = await fetch('/workitems/inbox');
    const data = await resp.json();
    this.items = (data as Array<{ item: WorkItemResponse }>).map(d => d.item).slice(0, 6);
  }

  override render() {
    return html`
      <h2>Work Item Row</h2>
      <p>Shared row component used by the inbox and queue board. Shows priority border, status pill, category, and age.</p>
      <div class="rows">
        ${this.items.map(item => html`<work-item-row .item=${item}></work-item-row>`)}
      </div>
    `;
  }
}
```

- [ ] **Step 2: Implement inbox-page**

```typescript
// examples/src/pages/inbox-page.ts
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import type { WorkIdentity } from '@casehubio/blocks-ui-core';
import '@casehubio/blocks-ui-work-item-inbox';

const IDENTITY: WorkIdentity = {
  userId: 'demo-user',
  displayName: 'Demo User',
  groups: ['compliance', 'clinical-safety', 'household', 'device-ops', 'code-review'],
};

@customElement('inbox-page')
export class InboxPage extends LitElement {
  static override styles = css`
    :host { display: block; height: 100%; }
    h2 { padding: 24px 24px 8px; font-size: 20px; font-weight: 600; color: var(--blocks-neutral-12, #111); }
    p { padding: 0 24px 16px; color: var(--blocks-neutral-11, #555); font-size: 14px; }
    work-item-inbox { display: block; height: calc(100% - 80px); }
  `;

  override render() {
    return html`
      <h2>Work Item Inbox</h2>
      <p>Two modes: My Work (assigned items) and Claimable (pending in your groups). Try claiming an item with the C key.</p>
      <work-item-inbox endpoint="/workitems" .identity=${IDENTITY}></work-item-inbox>
    `;
  }
}
```

- [ ] **Step 3: Implement detail-page**

```typescript
// examples/src/pages/detail-page.ts
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import type { WorkIdentity, WorkItemResponse } from '@casehubio/blocks-ui-core';
import '@casehubio/blocks-ui-work-item-detail';

const IDENTITY: WorkIdentity = {
  userId: 'demo-user',
  displayName: 'Demo User',
  groups: ['compliance', 'clinical-safety', 'household', 'device-ops', 'code-review'],
};

@customElement('detail-page')
export class DetailPage extends LitElement {
  @state() private items: WorkItemResponse[] = [];
  @state() private selectedId: string | null = null;

  static override styles = css`
    :host { display: flex; height: 100%; }
    .picker { width: 260px; border-right: 1px solid var(--blocks-neutral-5, #e0e0e0); overflow-y: auto; padding: 16px; }
    .picker h3 { font-size: 14px; font-weight: 600; margin-bottom: 12px; color: var(--blocks-neutral-12); }
    .pick-item { display: block; width: 100%; text-align: left; padding: 8px; margin-bottom: 4px; border: 1px solid var(--blocks-neutral-5, #e0e0e0); border-radius: 4px; background: var(--blocks-neutral-1); cursor: pointer; font-size: 13px; color: var(--blocks-neutral-12); }
    .pick-item:hover { background: var(--blocks-neutral-3); }
    .pick-item.active { border-color: var(--blocks-accent-9); background: var(--blocks-accent-2); }
    .pick-item .status { font-size: 11px; color: var(--blocks-neutral-9); text-transform: uppercase; }
    .detail-area { flex: 1; overflow: auto; }
    work-item-detail { display: block; height: 100%; }
  `;

  override async connectedCallback() {
    super.connectedCallback();
    const resp = await fetch('/workitems/inbox');
    const data = await resp.json();
    this.items = (data as Array<{ item: WorkItemResponse }>).map(d => d.item);
    if (this.items.length > 0) this.selectedId = this.items[0]!.id;
  }

  override render() {
    return html`
      <div class="picker">
        <h3>Select a work item</h3>
        ${this.items.map(item => html`
          <button class="pick-item ${this.selectedId === item.id ? 'active' : ''}"
            @click=${() => { this.selectedId = item.id; }}>
            ${item.title}
            <div class="status">${item.status} · ${item.priority}</div>
          </button>
        `)}
      </div>
      <div class="detail-area">
        ${this.selectedId
          ? html`<work-item-detail endpoint="/workitems" .workItemId=${this.selectedId} .identity=${IDENTITY}></work-item-detail>`
          : html`<div style="padding: 24px; color: var(--blocks-neutral-9)">Select an item to view details</div>`}
      </div>
    `;
  }
}
```

- [ ] **Step 4: Implement queue-page**

```typescript
// examples/src/pages/queue-page.ts
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import type { WorkIdentity } from '@casehubio/blocks-ui-core';
import '@casehubio/blocks-ui-queue-board';

const IDENTITY: WorkIdentity = {
  userId: 'demo-user',
  displayName: 'Demo User',
  groups: ['compliance', 'clinical-safety', 'household', 'device-ops', 'code-review'],
};

@customElement('queue-page')
export class QueuePage extends LitElement {
  static override styles = css`
    :host { display: block; height: 100%; }
    h2 { padding: 24px 24px 8px; font-size: 20px; font-weight: 600; color: var(--blocks-neutral-12, #111); }
    p { padding: 0 24px 16px; color: var(--blocks-neutral-11, #555); font-size: 14px; }
    queue-board { display: block; height: calc(100% - 80px); }
  `;

  override render() {
    return html`
      <h2>Queue Board</h2>
      <p>Dashboard view of work queues. Click a card to expand its items. Arrow keys navigate, Escape returns.</p>
      <queue-board endpoint="/workitems" .identity=${IDENTITY}></queue-board>
    `;
  }
}
```

- [ ] **Step 5: Implement schema-form-page**

```typescript
// examples/src/pages/schema-form-page.ts
import { LitElement, html, css } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import '@casehubio/blocks-ui-core';

const SAMPLE_SCHEMA = {
  type: 'object',
  properties: {
    transactionId: { type: 'string' },
    amount: { type: 'number' },
    currency: { type: 'string', enum: ['USD', 'EUR', 'GBP'] },
    flagged: { type: 'boolean' },
    notes: { type: 'string', maxLength: 500 },
    parties: {
      type: 'object',
      properties: {
        sender: { type: 'string' },
        receiver: { type: 'string' },
      },
    },
  },
  required: ['transactionId', 'amount'],
};

const SAMPLE_DATA = {
  transactionId: 'TXN-2026-04521',
  amount: 125000,
  currency: 'USD',
  flagged: true,
  notes: 'Multiple rapid transfers to newly opened accounts in high-risk jurisdictions.',
  parties: { sender: 'Acme Holdings Ltd', receiver: 'Shell Corp 42 LLC' },
};

@customElement('schema-form-page')
export class SchemaFormPage extends LitElement {
  @state() private mode: 'display' | 'edit' = 'display';

  static override styles = css`
    :host { display: block; padding: 24px; }
    h2 { margin-bottom: 8px; font-size: 20px; font-weight: 600; color: var(--blocks-neutral-12, #111); }
    p { margin-bottom: 16px; color: var(--blocks-neutral-11, #555); font-size: 14px; }
    .controls { margin-bottom: 16px; display: flex; gap: 8px; }
    .mode-btn { padding: 6px 14px; border: 1px solid var(--blocks-neutral-6); border-radius: 4px; background: var(--blocks-neutral-1); cursor: pointer; font-size: 13px; color: var(--blocks-neutral-11); }
    .mode-btn.active { background: var(--blocks-accent-9); color: white; border-color: var(--blocks-accent-9); }
    schema-form { display: block; max-width: 600px; border: 1px solid var(--blocks-neutral-5); border-radius: 6px; padding: 16px; background: var(--blocks-neutral-1); }
  `;

  override render() {
    return html`
      <h2>Schema Form</h2>
      <p>Renders JSON Schema as read-only display or editable form. Sample: AML suspicious transaction payload.</p>
      <div class="controls">
        <button class="mode-btn ${this.mode === 'display' ? 'active' : ''}" @click=${() => { this.mode = 'display'; }}>Display</button>
        <button class="mode-btn ${this.mode === 'edit' ? 'active' : ''}" @click=${() => { this.mode = 'edit'; }}>Edit</button>
      </div>
      <schema-form .schema=${SAMPLE_SCHEMA} .data=${SAMPLE_DATA} .mode=${this.mode}></schema-form>
    `;
  }
}
```

- [ ] **Step 6: Implement workbench-page**

```typescript
// examples/src/pages/workbench-page.ts
import { LitElement, html, css } from 'lit';
import { customElement } from 'lit/decorators.js';
import type { WorkIdentity } from '@casehubio/blocks-ui-core';
import '@casehubio/blocks-ui-work-item-workbench';

const IDENTITY: WorkIdentity = {
  userId: 'demo-user',
  displayName: 'Demo User',
  groups: ['compliance', 'clinical-safety', 'household', 'device-ops', 'code-review'],
};

@customElement('workbench-page')
export class WorkbenchPage extends LitElement {
  static override styles = css`
    :host { display: block; height: 100%; }
    work-item-workbench { display: block; height: 100%; }
  `;

  override render() {
    return html`
      <work-item-workbench endpoint="/workitems" .identity=${IDENTITY}></work-item-workbench>
    `;
  }
}
```

- [ ] **Step 7: Start Vite, verify all pages render**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui/examples && npx vite --host 127.0.0.1`

Verify in browser:
- Shell renders with sidebar, theme/density toggles work
- `#components/row` shows work item rows with realistic data
- `#components/inbox` shows inbox with summary bar, filters, items
- `#components/detail` shows item picker + detail panel
- `#components/queue` shows queue dashboard cards
- `#components/schema-form` shows display/edit toggle with AML transaction data
- `#composed/workbench` shows full split-pane workbench
- Claiming an item in inbox updates the list in real-time (SSE)
- SSE script fires ambient events (items appearing/disappearing)

- [ ] **Step 8: Commit**

```bash
git add examples/src/pages/
git commit -m "feat(examples): add component demo pages and full workbench showcase

Six demo pages: Work Item Row, Inbox, Detail, Queue Board, Schema Form,
and Full Workbench. Each renders with mock data from the stateful mock
layer. SSE simulation active on all pages. Detail page includes an item
picker for testing different statuses.

Closes #17"
```

---

## Post-Implementation

1. **Run full test suite** — `cd examples && npx vitest run` for mock layer tests
2. **Visual verification** — start Vite, check all 6 pages, verify SSE events and interactivity
3. **Update CLAUDE.md** — add `examples/` to Key Directories table with description "Interactive component showcase — `yarn examples` to start"
