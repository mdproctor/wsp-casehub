# Worker Task Pane Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #152 — blocks-worker-task-pane: generic worker task queue with specialist workspace slots
**Issue group:** #152

**Goal:** Build a generic worker task pane composing list-pane, detail-pane, and a workspace element registry with a response form, following work-item-inbox's fetch pattern.

**Architecture:** Single component extending `LiveRegionMixin(KeyboardShortcutMixin(LitElement))`. Manages its own fetch, stores raw `WorkerTaskResponse[]`, derives `TypedDataSet` via `fromRows()` for list-pane display. Workspace elements created by capability tag, cached in a Map. Response form with optional claim step. SSE via `SSEManager` from pages-data.

**Tech Stack:** Lit 3, TypeScript, pages-data (fromRows, SSEManager, emitPagesEvent), pages-primitives (LiveRegionMixin, KeyboardShortcutMixin), pages-table (TableColumnConfig, ColumnRenderer), pages-split-workbench, blocks-list-pane, blocks-detail-pane, vitest

## Global Constraints

- ARIA on every element — role, aria-label, aria-disabled, aria-live, aria-expanded, aria-busy
- No slots for domain customisation (protocol PP-20260713-8ea1af) — typed properties and callbacks only
- Reuse `TabDefinition` from `components/detail-pane/src/types.ts` — do not duplicate
- list-pane in inline dataset mode (no endpoint set on it)
- All render callbacks use inline styles (shadow DOM boundary, per protocol)
- Event names use colon separators per ARC42STORIES §4: `worker-task:responded`
- Package naming: `@casehubio/blocks-ui-worker-task-pane`
- Test with vitest, co-located test file in `src/`

---

## Batch 1: Types and component skeleton

### Task 1: Types and package setup

**Files:**
- Create: `components/worker-task-pane/package.json`
- Create: `components/worker-task-pane/tsconfig.json`
- Create: `components/worker-task-pane/src/types.ts`
- Create: `components/worker-task-pane/src/index.ts`

**Interfaces:**
- Produces: `WorkerTaskResponse`, `WorkerTaskSubmission`, `WorkerTaskClaimRequest`, `WorkerTaskContext`, `WorkspaceDefinition`, `WorkspaceResultEvent`, `WorkerTaskEventTopics` — consumed by Task 2 and all subsequent tasks

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-worker-task-pane",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc --project tsconfig.build.json",
    "test": "vitest run"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/blocks-ui-detail-pane": "workspace:*",
    "@casehubio/blocks-ui-list-pane": "workspace:*",
    "@casehubio/pages-data": "*",
    "@casehubio/pages-primitives": "*",
    "@casehubio/pages-table": "*",
    "@casehubio/pages-ui-components": "*",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "@vitest/browser": "^2.1.8",
    "jsdom": "^26.0.0",
    "typescript": "~5.7.2",
    "vite": "^6.0.3",
    "vitest": "^2.1.8"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  }
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src/**/*"],
  "references": [
    { "path": "../../packages/blocks-ui-core" },
    { "path": "../detail-pane" },
    { "path": "../list-pane" }
  ]
}
```

- [ ] **Step 3: Create types.ts**

```typescript
import type { TabDefinition } from '@casehubio/blocks-ui-detail-pane';

export type { TabDefinition };

export interface WorkspaceDefinition {
  capabilityTag: string;
  tagName: string;
  label?: string;
  icon?: string;
}

export interface WorkerTaskResponse {
  taskId: string;
  capabilityTag: string;
  caseId: string;
  assigneeId?: string;
  dispatchedAt: string;
  commandParams: Record<string, unknown>;
  investigationSummary: Record<string, unknown>;
}

export interface WorkerTaskSubmission {
  type: 'RESPONSE' | 'DONE' | 'DECLINE';
  taskId: string;
  result?: {
    fields: Record<string, unknown>;
    confidence: number;
  };
  declineReason?: string;
  declineDetail?: string;
}

export interface WorkerTaskClaimRequest {
  taskId: string;
}

export interface WorkerTaskContext {
  taskId: string;
  capabilityTag: string;
  caseId: string;
  commandParams: Record<string, unknown>;
  investigationSummary: Record<string, unknown>;
}

export interface WorkspaceResultEvent extends CustomEvent {
  detail: {
    fields: Record<string, unknown>;
    confidence: number;
  };
}

export const WorkerTaskEventTopics = {
  SELECTED: 'worker-task:selected',
  DESELECTED: 'worker-task:deselected',
  CLAIMED: 'worker-task:claimed',
  RESPONDED: 'worker-task:responded',
  DECLINED: 'worker-task:declined',
} as const;
```

- [ ] **Step 4: Create index.ts**

```typescript
export { BlocksWorkerTaskPane } from './worker-task-pane.js';
export type {
  WorkspaceDefinition,
  WorkerTaskResponse,
  WorkerTaskSubmission,
  WorkerTaskClaimRequest,
  WorkerTaskContext,
  WorkspaceResultEvent,
  TabDefinition,
} from './types.js';
export { WorkerTaskEventTopics } from './types.js';
```

- [ ] **Step 5: Run `yarn install` to link the new workspace package**

Run: `yarn install`
Expected: New package linked without errors

- [ ] **Step 6: Commit**

```bash
git add components/worker-task-pane/
git commit -m "feat(worker-task-pane): add types and package scaffold Refs #152"
```

### Task 2: Component skeleton with fetch, dataset derivation, and list-pane rendering

**Files:**
- Create: `components/worker-task-pane/src/worker-task-pane.ts`
- Create: `components/worker-task-pane/src/worker-task-pane.test.ts`

**Interfaces:**
- Consumes: All types from Task 1
- Produces: `BlocksWorkerTaskPane` class — consumed by Task 3 (workspace), Task 4 (response form), Task 5 (showcase)

- [ ] **Step 1: Write the failing test — component renders with inline data**

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import type { WorkerTaskResponse } from './types.js';
import './worker-task-pane.js';

const SEED_TASKS: WorkerTaskResponse[] = [
  {
    taskId: 'task-1',
    capabilityTag: 'entity-resolution',
    caseId: 'INV-003',
    dispatchedAt: '2026-09-01T09:00:00Z',
    commandParams: { entityIds: ['E-100', 'E-101'] },
    investigationSummary: { flagReason: 'Name match', riskScore: 0.82 },
  },
  {
    taskId: 'task-2',
    capabilityTag: 'pattern-analysis',
    caseId: 'INV-007',
    assigneeId: 'user-1',
    dispatchedAt: '2026-09-01T10:00:00Z',
    commandParams: { patternType: 'structuring' },
    investigationSummary: { flagReason: 'Threshold split', riskScore: 0.65 },
  },
  {
    taskId: 'task-3',
    capabilityTag: 'osint-screening',
    caseId: 'INV-012',
    dispatchedAt: '2026-09-01T11:00:00Z',
    commandParams: { screeningType: 'pep' },
    investigationSummary: { flagReason: 'PEP match', riskScore: 0.91 },
  },
];

function createElement(data: WorkerTaskResponse[] = SEED_TASKS) {
  const el = document.createElement('blocks-worker-task-pane') as any;
  el.data = data;
  el.selectionTopic = 'test-worker';
  document.body.appendChild(el);
  return el;
}

describe('blocks-worker-task-pane', () => {
  let el: any;

  afterEach(() => {
    el?.remove();
  });

  it('renders with role="region" and aria-label', async () => {
    el = createElement();
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Worker task pane');
  });

  it('renders list-pane with derived dataset from inline data', async () => {
    el = createElement();
    await el.updateComplete;
    const listPane = el.shadowRoot!.querySelector('blocks-list-pane');
    expect(listPane).toBeTruthy();
    expect(listPane!.dataSet).toBeTruthy();
    expect(listPane!.dataSet!.rowCount).toBe(3);
  });

  it('renders split-workbench in split layout mode', async () => {
    el = createElement();
    await el.updateComplete;
    const workbench = el.shadowRoot!.querySelector('pages-split-workbench');
    expect(workbench).toBeTruthy();
  });

  it('renders vertical flex in stacked layout mode', async () => {
    el = createElement();
    el.layout = 'stacked';
    await el.updateComplete;
    const workbench = el.shadowRoot!.querySelector('pages-split-workbench');
    expect(workbench).toBeNull();
    const stacked = el.shadowRoot!.querySelector('.stacked-layout');
    expect(stacked).toBeTruthy();
  });

  it('shows empty message when no data', async () => {
    el = createElement([]);
    await el.updateComplete;
    const listPane = el.shadowRoot!.querySelector('blocks-list-pane');
    expect(listPane).toBeTruthy();
  });

  it('fetches from endpoint when set', async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve(SEED_TASKS),
    });
    vi.stubGlobal('fetch', mockFetch);

    el = document.createElement('blocks-worker-task-pane') as any;
    el.endpoint = '/api/worker-tasks';
    el.selectionTopic = 'test-worker';
    document.body.appendChild(el);
    await el.updateComplete;
    await vi.waitFor(() => {
      expect(mockFetch).toHaveBeenCalledWith('/api/worker-tasks');
    });

    vi.unstubAllGlobals();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose`
Expected: FAIL — module not found (worker-task-pane.ts doesn't exist yet)

- [ ] **Step 3: Implement the component skeleton**

```typescript
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { KeyboardShortcutMixin, LiveRegionMixin } from '@casehubio/pages-primitives';
import { emitPagesEvent, onPagesEvent } from '@casehubio/pages-data';
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { TypedDataSet, TypedRow, ColumnId as ColId } from '@casehubio/pages-data/dist/dataset/types.js';
import type { TableColumnConfig, ColumnRenderer } from '@casehubio/pages-table';
import type { TabDefinition } from '@casehubio/blocks-ui-detail-pane';
import type { WorkIdentity } from '@casehubio/blocks-ui-core';
import '@casehubio/pages-ui-components';
import '@casehubio/blocks-ui-list-pane';
import '@casehubio/blocks-ui-detail-pane';
import type {
  WorkerTaskResponse,
  WorkerTaskSubmission,
  WorkerTaskClaimRequest,
  WorkerTaskContext,
  WorkspaceDefinition,
  WorkspaceResultEvent,
} from './types.js';
import { WorkerTaskEventTopics } from './types.js';

const TASK_ID_COL = columnId('taskId');
const CASE_ID_COL = columnId('caseId');
const CAPABILITY_COL = columnId('capabilityTag');
const DISPATCHED_COL = columnId('dispatchedAt');

const DEFAULT_COL_DEFS = [
  { id: TASK_ID_COL, type: ColumnType.TEXT, getValue: (row: WorkerTaskResponse) => row.taskId },
  { id: CASE_ID_COL, name: 'Case', type: ColumnType.TEXT, getValue: (row: WorkerTaskResponse) => row.caseId },
  { id: CAPABILITY_COL, name: 'Capability', type: ColumnType.TEXT, getValue: (row: WorkerTaskResponse) => row.capabilityTag },
  { id: DISPATCHED_COL, name: 'Dispatched', type: ColumnType.DATE, getValue: (row: WorkerTaskResponse) => row.dispatchedAt },
] as const;

const DEFAULT_COL_CONFIG: readonly TableColumnConfig[] = [
  { id: TASK_ID_COL, visible: false },
  { id: CASE_ID_COL, sortable: true, width: '1fr' },
  { id: CAPABILITY_COL, sortable: true, width: '1fr' },
  { id: DISPATCHED_COL, sortable: true, width: '1fr' },
];

const WorkerTaskPaneBase = LiveRegionMixin(KeyboardShortcutMixin(LitElement));

@customElement('blocks-worker-task-pane')
export class BlocksWorkerTaskPane extends WorkerTaskPaneBase {
  @property() layout: 'split' | 'stacked' = 'split';
  @property() endpoint = '';
  @property({ attribute: false }) data?: WorkerTaskResponse[];
  @property({ attribute: 'selection-topic' }) selectionTopic = 'worker-task';
  @property({ attribute: false }) identity: WorkIdentity = { userId: '', displayName: '', groups: [] };
  @property({ attribute: false }) columnConfig?: TableColumnConfig[];
  @property({ attribute: false }) columnRenderers?: ReadonlyMap<ColId, ColumnRenderer>;
  @property({ attribute: false }) getRowKey?: (row: TypedRow) => string;
  @property({ attribute: false }) contextTabs: TabDefinition[] = [];
  @property({ attribute: false }) workspaces: WorkspaceDefinition[] = [];
  @property({ attribute: 'respond-endpoint' }) respondEndpoint = '';
  @property({ attribute: false }) declineReasons: string[] = ['Out of clearance', 'Insufficient data', 'Conflict of interest'];
  @property({ attribute: 'claim-endpoint' }) claimEndpoint = '';
  @property({ attribute: 'event-stream-endpoint' }) eventStreamEndpoint = '';
  @property({ type: Boolean, attribute: 'show-context' }) showContext = true;
  @property({ type: Boolean, attribute: 'show-workspace' }) showWorkspace = true;

  @state() private _items: WorkerTaskResponse[] = [];
  @state() private _selectedItem: WorkerTaskResponse | null = null;
  @state() private _tableDataSet?: TypedDataSet;
  @state() private _loading = false;
  @state() private _error: string | null = null;
  @state() private _claimed = false;
  @state() private _workspaceResult: WorkspaceResultEvent['detail'] | null = null;
  @state() private _submitting = false;
  @state() private _submitError: string | null = null;
  @state() private _showDeclineForm = false;

  private _workspaceElements = new Map<string, HTMLElement>();
  private _unsubscribeSelection?: () => void;

  static override styles = css`
    :host { display: flex; flex-direction: column; height: 100%; box-sizing: border-box; }
    .stacked-layout { display: flex; flex-direction: column; height: 100%; gap: 8px; overflow-y: auto; }
    .detail-column { display: flex; flex-direction: column; gap: 8px; overflow-y: auto; padding: 8px; }
    .section { border: 1px solid var(--pages-neutral-5, #e0e0e0); border-radius: 6px; overflow: hidden; }
    .section-header { display: flex; align-items: center; padding: 8px 12px; background: var(--pages-neutral-2, #f5f5f5); font-size: 13px; font-weight: 600; color: var(--pages-neutral-11, #555); cursor: pointer; }
    .workspace-container { min-height: var(--worker-task-workspace-min-height, 200px); }
    .context-container { min-height: var(--worker-task-context-min-height, 120px); }
    .response-container { min-height: var(--worker-task-response-min-height, auto); }
    .empty-detail { display: flex; align-items: center; justify-content: center; height: 100%; color: var(--pages-neutral-9, #888); font-size: 14px; }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute('role', 'region');
    this.setAttribute('aria-label', 'Worker task pane');
    this._unsubscribeSelection = onPagesEvent(document, `${this.selectionTopic}:selected`, (payload: any) => {
      this._handleSelection(payload);
    });
    if (this.endpoint && !this.data) {
      this._fetchItems();
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._unsubscribeSelection?.();
  }

  override willUpdate(changed: Map<string, unknown>): void {
    super.willUpdate(changed);
    if (changed.has('data') && this.data) {
      this._items = this.data;
      this._rebuildDataSet();
    }
    if (changed.has('endpoint') && this.endpoint && !this.data) {
      this._fetchItems();
    }
  }

  configure(props: Partial<Record<string, unknown>>): void {
    Object.assign(this, props);
  }

  private async _fetchItems(): Promise<void> {
    this._loading = true;
    this._error = null;
    try {
      const resp = await fetch(this.endpoint);
      if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
      this._items = await resp.json();
      this._rebuildDataSet();
    } catch (e: any) {
      this._error = e.message ?? 'Failed to load tasks';
      this.announce('Failed to load tasks');
    } finally {
      this._loading = false;
    }
  }

  private _rebuildDataSet(): void {
    const filtered = this._filterByIdentity(this._items);
    this._tableDataSet = filtered.length > 0 ? fromRows(filtered, DEFAULT_COL_DEFS) : undefined;
  }

  private _filterByIdentity(items: WorkerTaskResponse[]): WorkerTaskResponse[] {
    if (!this.identity.groups.length) return items;
    return items.filter(task => {
      if (task.assigneeId && task.assigneeId !== this.identity.userId) return false;
      return this.identity.groups.includes(task.capabilityTag);
    });
  }

  private _handleSelection(payload: any): void {
    const taskId = payload?.taskId ?? payload?.id;
    if (!taskId) return;
    const item = this._items.find(i => i.taskId === taskId);
    if (!item) return;
    this._selectedItem = item;
    this._workspaceResult = null;
    this._submitError = null;
    this._showDeclineForm = false;
    this._claimed = !this.claimEndpoint || item.assigneeId === this.identity.userId;
  }

  private _defaultGetRowKey = (row: TypedRow): string => row.text(TASK_ID_COL);

  private _renderListPane(): TemplateResult {
    return html`
      <blocks-list-pane
        .dataSet=${this._tableDataSet}
        .columnConfig=${this.columnConfig ?? DEFAULT_COL_CONFIG}
        .columnRenderers=${this.columnRenderers}
        .getRowKey=${this.getRowKey ?? this._defaultGetRowKey}
        selection-topic=${this.selectionTopic}
        empty-message="No tasks available"
        aria-busy=${this._loading ? 'true' : 'false'}
      ></blocks-list-pane>
    `;
  }

  private _renderDetailColumn(): TemplateResult {
    if (!this._selectedItem) {
      return html`<div class="empty-detail">Select a task to view details</div>`;
    }
    return html`
      <div class="detail-column">
        ${this.showContext ? this._renderContextSection() : nothing}
        ${this.showWorkspace ? this._renderWorkspaceSection() : nothing}
        ${this._renderResponseSection()}
      </div>
    `;
  }

  private _renderContextSection(): TemplateResult {
    return html`
      <div class="section context-container">
        <blocks-detail-pane
          .tabs=${this.contextTabs}
          selection-topic=${this.selectionTopic}
        ></blocks-detail-pane>
      </div>
    `;
  }

  private _renderWorkspaceSection(): TemplateResult {
    return html`
      <div class="section workspace-container" role="region" aria-label="Specialist workspace" aria-live="polite">
        ${this._renderWorkspaceElement()}
      </div>
    `;
  }

  private _renderWorkspaceElement(): TemplateResult | typeof nothing {
    if (!this._selectedItem) return nothing;
    const def = this.workspaces.find(w => w.capabilityTag === this._selectedItem!.capabilityTag);
    if (!def) return html`<div class="empty-detail">No workspace registered for ${this._selectedItem.capabilityTag}</div>`;
    return nothing; // Placeholder — Task 3 implements workspace element lifecycle
  }

  private _renderResponseSection(): TemplateResult {
    return html`
      <div class="section response-container">
        <!-- Placeholder — Task 4 implements response form -->
      </div>
    `;
  }

  override render(): TemplateResult {
    if (this.layout === 'stacked') {
      return html`
        <div class="stacked-layout">
          ${this._renderListPane()}
          ${this._renderDetailColumn()}
        </div>
      `;
    }
    return html`
      <pages-split-workbench selection-topic=${this.selectionTopic}>
        <div slot="list">${this._renderListPane()}</div>
        <div slot="detail">${this._renderDetailColumn()}</div>
      </pages-split-workbench>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'blocks-worker-task-pane': BlocksWorkerTaskPane;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose`
Expected: All 6 tests PASS

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors in the new component

- [ ] **Step 6: Commit**

```bash
git add components/worker-task-pane/
git commit -m "feat(worker-task-pane): component skeleton with fetch, dataset derivation, list-pane rendering Refs #152"
```

---

## Batch 2: Workspace element lifecycle and response form

### Task 3: Workspace element creation, caching, and result collection

**Files:**
- Modify: `components/worker-task-pane/src/worker-task-pane.ts` — replace workspace placeholder
- Modify: `components/worker-task-pane/src/worker-task-pane.test.ts` — add workspace tests

**Interfaces:**
- Consumes: `WorkspaceDefinition`, `WorkerTaskContext`, `WorkspaceResultEvent` from Task 1
- Produces: Working workspace lifecycle (create, cache, taskContext set, workspace-result capture)

- [ ] **Step 1: Write failing tests for workspace element lifecycle**

Add to the test file:

```typescript
describe('workspace element lifecycle', () => {
  it('creates workspace element matching capabilityTag', async () => {
    customElements.define('test-workspace-entity', class extends HTMLElement {});
    el = createElement();
    el.workspaces = [{ capabilityTag: 'entity-resolution', tagName: 'test-workspace-entity' }];
    await el.updateComplete;

    // Simulate selection
    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;

    const workspace = el.shadowRoot!.querySelector('test-workspace-entity');
    expect(workspace).toBeTruthy();
    expect((workspace as any).taskContext).toEqual({
      taskId: 'task-1',
      capabilityTag: 'entity-resolution',
      caseId: 'INV-003',
      commandParams: { entityIds: ['E-100', 'E-101'] },
      investigationSummary: { flagReason: 'Name match', riskScore: 0.82 },
    });
  });

  it('caches workspace elements by capabilityTag', async () => {
    let createCount = 0;
    const origCreate = document.createElement.bind(document);
    vi.spyOn(document, 'createElement').mockImplementation((tag: string) => {
      if (tag === 'test-workspace-entity') createCount++;
      return origCreate(tag);
    });

    el = createElement();
    el.workspaces = [{ capabilityTag: 'entity-resolution', tagName: 'test-workspace-entity' }];
    await el.updateComplete;

    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;
    expect(createCount).toBe(1);

    // Select a different task with same capability — element should be reused
    emitPagesEvent(document, 'test-worker:deselected', {});
    await el.updateComplete;
    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;
    expect(createCount).toBe(1); // Still 1 — reused from cache

    vi.restoreAllMocks();
  });

  it('captures workspace-result event', async () => {
    customElements.define('test-workspace-pattern', class extends HTMLElement {
      set taskContext(_ctx: any) {
        setTimeout(() => {
          this.dispatchEvent(new CustomEvent('workspace-result', {
            detail: { fields: { risk: 'high' }, confidence: 0.85 },
          }));
        }, 0);
      }
    });

    el = createElement();
    el.workspaces = [{ capabilityTag: 'pattern-analysis', tagName: 'test-workspace-pattern' }];
    await el.updateComplete;

    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-2' });
    await el.updateComplete;
    await vi.waitFor(() => {
      expect(el._workspaceResult).toEqual({ fields: { risk: 'high' }, confidence: 0.85 });
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose`
Expected: FAIL — workspace element not created

- [ ] **Step 3: Implement workspace element lifecycle**

Replace `_renderWorkspaceElement()` and add workspace management methods in `worker-task-pane.ts`:

```typescript
private _renderWorkspaceElement(): TemplateResult | typeof nothing {
  if (!this._selectedItem) return nothing;
  const def = this.workspaces.find(w => w.capabilityTag === this._selectedItem!.capabilityTag);
  if (!def) return html`<div class="empty-detail">No workspace registered for ${this._selectedItem.capabilityTag}</div>`;

  const element = this._getOrCreateWorkspace(def);
  (element as any).taskContext = this._buildTaskContext(this._selectedItem);
  return html`${element}`;
}

private _getOrCreateWorkspace(def: WorkspaceDefinition): HTMLElement {
  let element = this._workspaceElements.get(def.capabilityTag);
  if (!element) {
    element = document.createElement(def.tagName);
    element.addEventListener('workspace-result', ((e: CustomEvent) => {
      this._workspaceResult = e.detail;
    }) as EventListener);
    this._workspaceElements.set(def.capabilityTag, element);
  }
  return element;
}

private _buildTaskContext(task: WorkerTaskResponse): WorkerTaskContext {
  return {
    taskId: task.taskId,
    capabilityTag: task.capabilityTag,
    caseId: task.caseId,
    commandParams: task.commandParams,
    investigationSummary: task.investigationSummary,
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose`
Expected: All workspace tests PASS

- [ ] **Step 5: Commit**

```bash
git add components/worker-task-pane/src/
git commit -m "feat(worker-task-pane): workspace element creation, caching, and result collection Refs #152"
```

### Task 4: Response form with claim, submit, and decline

**Files:**
- Modify: `components/worker-task-pane/src/worker-task-pane.ts` — replace response section placeholder
- Modify: `components/worker-task-pane/src/worker-task-pane.test.ts` — add response form tests

**Interfaces:**
- Consumes: `WorkerTaskSubmission`, `WorkerTaskClaimRequest`, `WorkerTaskEventTopics` from Task 1; `_workspaceResult`, `_selectedItem`, `_claimed` from Task 2/3
- Produces: Complete response form with claim gate, submit, decline, event dispatch, optional POST

- [ ] **Step 1: Write failing tests for the response form**

Add to test file:

```typescript
describe('response form', () => {
  it('shows claim button when claimEndpoint set and task not claimed', async () => {
    el = createElement();
    el.claimEndpoint = '/api/claim';
    el.identity = { userId: 'user-1', displayName: 'Test', groups: [] };
    await el.updateComplete;

    // Select unassigned task (task-1 has no assigneeId)
    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;

    const claimBtn = el.shadowRoot!.querySelector('[data-action="claim"]');
    expect(claimBtn).toBeTruthy();
    const submitBtn = el.shadowRoot!.querySelector('[data-action="submit"]');
    expect(submitBtn).toBeNull();
  });

  it('skips claim when task is pre-assigned to current user', async () => {
    el = createElement();
    el.claimEndpoint = '/api/claim';
    el.identity = { userId: 'user-1', displayName: 'Test', groups: [] };
    await el.updateComplete;

    // Select task-2 which has assigneeId: 'user-1'
    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-2' });
    await el.updateComplete;

    const claimBtn = el.shadowRoot!.querySelector('[data-action="claim"]');
    expect(claimBtn).toBeNull();
    const submitBtn = el.shadowRoot!.querySelector('[data-action="submit"]');
    expect(submitBtn).toBeTruthy();
  });

  it('shows response form when no claimEndpoint', async () => {
    el = createElement();
    await el.updateComplete;

    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;

    const submitBtn = el.shadowRoot!.querySelector('[data-action="submit"]');
    expect(submitBtn).toBeTruthy();
  });

  it('dispatches worker-task:responded event on submit', async () => {
    el = createElement();
    await el.updateComplete;

    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;

    // Simulate workspace result
    el._workspaceResult = { fields: { risk: 'low' }, confidence: 0.9 };
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('worker-task:responded', handler);

    const submitBtn = el.shadowRoot!.querySelector('[data-action="submit"]') as HTMLButtonElement;
    submitBtn.click();
    await el.updateComplete;

    expect(handler).toHaveBeenCalledOnce();
    const detail = handler.mock.calls[0][0].detail;
    expect(detail.type).toBe('RESPONSE');
    expect(detail.taskId).toBe('task-1');
    expect(detail.result.confidence).toBe(0.9);
  });

  it('dispatches worker-task:declined event with reason', async () => {
    el = createElement();
    await el.updateComplete;

    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;

    const handler = vi.fn();
    el.addEventListener('worker-task:declined', handler);

    const declineBtn = el.shadowRoot!.querySelector('[data-action="decline"]') as HTMLButtonElement;
    declineBtn.click();
    await el.updateComplete;

    // Fill decline form
    const select = el.shadowRoot!.querySelector('[data-field="decline-reason"]') as HTMLSelectElement;
    select.value = 'Conflict of interest';
    select.dispatchEvent(new Event('change'));

    const confirmBtn = el.shadowRoot!.querySelector('[data-action="confirm-decline"]') as HTMLButtonElement;
    confirmBtn.click();
    await el.updateComplete;

    expect(handler).toHaveBeenCalledOnce();
    expect(handler.mock.calls[0][0].detail.declineReason).toBe('Conflict of interest');
  });

  it('submit button disabled when no workspace result', async () => {
    el = createElement();
    await el.updateComplete;

    emitPagesEvent(document, 'test-worker:selected', { taskId: 'task-1' });
    await el.updateComplete;

    const submitBtn = el.shadowRoot!.querySelector('[data-action="submit"]') as HTMLButtonElement;
    expect(submitBtn.getAttribute('aria-disabled')).toBe('true');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose`
Expected: FAIL — response form elements not rendered

- [ ] **Step 3: Implement the response form**

Replace `_renderResponseSection()` and add claim/submit/decline methods:

```typescript
private _renderResponseSection(): TemplateResult {
  if (!this._selectedItem) return html`<div class="section response-container"></div>`;

  return html`
    <div class="section response-container" role="form" aria-label="Task response">
      ${!this._claimed ? this._renderClaimButton() : this._renderResponseForm()}
      ${this._submitError ? html`<div role="alert" class="error-banner">${this._submitError}</div>` : nothing}
    </div>
  `;
}

private _renderClaimButton(): TemplateResult {
  return html`
    <div style="padding: 12px; text-align: center;">
      <button data-action="claim"
        @click=${this._handleClaim}
        ?disabled=${this._submitting}
        aria-disabled=${this._submitting ? 'true' : 'false'}>
        Claim Task
      </button>
    </div>
  `;
}

private _renderResponseForm(): TemplateResult {
  const hasResult = !!this._workspaceResult;
  return html`
    <div style="padding: 12px; display: flex; gap: 8px; align-items: flex-start; flex-wrap: wrap;">
      <button data-action="submit"
        @click=${this._handleSubmit}
        ?disabled=${!hasResult || this._submitting}
        aria-disabled=${!hasResult || this._submitting ? 'true' : 'false'}>
        Submit${this._workspaceResult ? html` (${(this._workspaceResult.confidence * 100).toFixed(0)}%)` : nothing}
      </button>
      <button data-action="decline" @click=${() => { this._showDeclineForm = true; }}>
        Decline
      </button>
      ${this._showDeclineForm ? this._renderDeclineForm() : nothing}
    </div>
  `;
}

private _renderDeclineForm(): TemplateResult {
  return html`
    <div style="width: 100%; margin-top: 8px; display: flex; flex-direction: column; gap: 8px;">
      <select data-field="decline-reason" aria-label="Decline reason">
        <option value="">Select reason...</option>
        ${this.declineReasons.map(r => html`<option value=${r}>${r}</option>`)}
      </select>
      <textarea data-field="decline-detail" aria-label="Decline detail"
        placeholder="Additional details (optional)" rows="2"
        style="resize: vertical; font-family: inherit; font-size: 13px;"></textarea>
      <button data-action="confirm-decline" @click=${this._handleDecline}>
        Confirm Decline
      </button>
    </div>
  `;
}

private async _handleClaim(): Promise<void> {
  if (!this._selectedItem) return;
  this._submitting = true;
  this._submitError = null;

  const detail: WorkerTaskClaimRequest = { taskId: this._selectedItem.taskId };
  const event = new CustomEvent(WorkerTaskEventTopics.CLAIMED, { detail, cancelable: true });
  this.dispatchEvent(event);

  if (!event.defaultPrevented && this.claimEndpoint) {
    try {
      const resp = await fetch(`${this.claimEndpoint}/${this._selectedItem.taskId}`, {
        method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(detail),
      });
      if (!resp.ok) throw new Error(`Claim failed: HTTP ${resp.status}`);
    } catch (e: any) {
      this._submitError = e.message;
      this._submitting = false;
      this.announce('Claim failed');
      return;
    }
  }

  this._claimed = true;
  this._submitting = false;
  this.announce('Task claimed');
}

private async _handleSubmit(): Promise<void> {
  if (!this._selectedItem || !this._workspaceResult) return;
  this._submitting = true;
  this._submitError = null;

  const submission: WorkerTaskSubmission = {
    type: 'RESPONSE',
    taskId: this._selectedItem.taskId,
    result: this._workspaceResult,
  };

  const event = new CustomEvent(WorkerTaskEventTopics.RESPONDED, { detail: submission, cancelable: true });
  this.dispatchEvent(event);

  if (!event.defaultPrevented && this.respondEndpoint) {
    try {
      const resp = await fetch(`${this.respondEndpoint}/${this._selectedItem.taskId}`, {
        method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(submission),
      });
      if (!resp.ok) throw new Error(`Submit failed: HTTP ${resp.status}`);
    } catch (e: any) {
      this._submitError = e.message;
      this._submitting = false;
      this.announce('Submission failed');
      return;
    }
  }

  this._items = this._items.filter(i => i.taskId !== this._selectedItem!.taskId);
  this._rebuildDataSet();
  this._selectedItem = null;
  this._workspaceResult = null;
  this._submitting = false;
  this.announce('Response submitted');
}

private async _handleDecline(): Promise<void> {
  if (!this._selectedItem) return;
  const reason = (this.shadowRoot!.querySelector('[data-field="decline-reason"]') as HTMLSelectElement)?.value || '';
  const detail = (this.shadowRoot!.querySelector('[data-field="decline-detail"]') as HTMLTextAreaElement)?.value || '';

  const submission: WorkerTaskSubmission = {
    type: 'DECLINE',
    taskId: this._selectedItem.taskId,
    declineReason: reason,
    declineDetail: detail,
  };

  const event = new CustomEvent(WorkerTaskEventTopics.DECLINED, { detail: submission, cancelable: true });
  this.dispatchEvent(event);

  if (!event.defaultPrevented && this.respondEndpoint) {
    try {
      const resp = await fetch(`${this.respondEndpoint}/${this._selectedItem.taskId}`, {
        method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(submission),
      });
      if (!resp.ok) throw new Error(`Decline failed: HTTP ${resp.status}`);
    } catch (e: any) {
      this._submitError = e.message;
      this.announce('Decline failed');
      return;
    }
  }

  this._items = this._items.filter(i => i.taskId !== this._selectedItem!.taskId);
  this._rebuildDataSet();
  this._selectedItem = null;
  this._showDeclineForm = false;
  this.announce('Task declined');
}
```

Add error-banner style to `static styles`:
```css
.error-banner { padding: 8px 12px; background: var(--pages-error-3, #fee); color: var(--pages-error-11, #c00); border-radius: 4px; font-size: 13px; margin-top: 8px; }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose`
Expected: All response form tests PASS

- [ ] **Step 5: Add keyboard shortcuts**

In `connectedCallback`, after existing setup:

```typescript
this.registerShortcut('c', () => this._handleClaim(), 'Claim task');
this.registerShortcut('d', () => { this._showDeclineForm = true; }, 'Decline task');
this.registerShortcut('Escape', () => {
  if (this._showDeclineForm) { this._showDeclineForm = false; }
  else { this._selectedItem = null; emitPagesEvent(document, `${this.selectionTopic}:deselected`, {}); }
}, 'Close / Deselect');
this.registerShortcut('?', () => { /* overlay handled by KeyboardShortcutMixin */ }, 'Show shortcuts');
```

- [ ] **Step 6: Run full test suite and typecheck**

Run: `cd components/worker-task-pane && npx vitest run --reporter=verbose && yarn typecheck`
Expected: All tests PASS, no type errors

- [ ] **Step 7: Commit**

```bash
git add components/worker-task-pane/src/
git commit -m "feat(worker-task-pane): response form with claim, submit, decline Refs #152"
```

---

## Batch 3: Examples gallery showcase

### Task 5: Showcase page with seeded data

**Files:**
- Create: `examples/src/pages/worker-task-pane-page.ts`
- Modify: `examples/src/main.ts` — add page module import
- Modify: `examples/src/shell.ts` — add nav item and case route

**Interfaces:**
- Consumes: `BlocksWorkerTaskPane`, all types from Task 1

- [ ] **Step 1: Create the showcase page**

Create `examples/src/pages/worker-task-pane-page.ts`:

```typescript
import { LitElement, html, css, type TemplateResult } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import '../../../components/worker-task-pane/src/worker-task-pane.js';
import type { WorkerTaskResponse, WorkspaceDefinition } from '../../../components/worker-task-pane/src/types.js';
import type { TabDefinition } from '../../../components/detail-pane/src/types.js';

const SEED_TASKS: WorkerTaskResponse[] = [
  {
    taskId: 'wt-001', capabilityTag: 'entity-resolution', caseId: 'INV-003',
    dispatchedAt: '2026-09-01T09:00:00Z',
    commandParams: { entityIds: ['E-100', 'E-101'], matchThreshold: 0.7 },
    investigationSummary: { flagReason: 'Name match across jurisdictions', riskScore: 0.82, amount: 125000, currency: 'USD', status: 'Open' },
  },
  {
    taskId: 'wt-002', capabilityTag: 'pattern-analysis', caseId: 'INV-007', assigneeId: 'user-demo',
    dispatchedAt: '2026-09-01T10:30:00Z',
    commandParams: { patternType: 'structuring', windowHours: 48 },
    investigationSummary: { flagReason: 'Threshold splitting detected', riskScore: 0.65, amount: 28500, currency: 'EUR', status: 'In Review' },
  },
  {
    taskId: 'wt-003', capabilityTag: 'osint-screening', caseId: 'INV-012',
    dispatchedAt: '2026-09-01T11:15:00Z',
    commandParams: { screeningType: 'pep', region: 'EU' },
    investigationSummary: { flagReason: 'PEP match — ministerial appointment 2024', riskScore: 0.91, amount: 500000, currency: 'GBP', status: 'Pending' },
  },
  {
    taskId: 'wt-004', capabilityTag: 'entity-resolution', caseId: 'INV-019', assigneeId: 'user-demo',
    dispatchedAt: '2026-09-02T08:00:00Z',
    commandParams: { entityIds: ['E-205'], matchThreshold: 0.85 },
    investigationSummary: { flagReason: 'Address discrepancy', riskScore: 0.45, amount: 12000, currency: 'USD', status: 'Open' },
  },
  {
    taskId: 'wt-005', capabilityTag: 'pattern-analysis', caseId: 'INV-021',
    dispatchedAt: '2026-09-02T09:30:00Z',
    commandParams: { patternType: 'layering', windowHours: 72 },
    investigationSummary: { flagReason: 'Rapid fund transfers across 4 accounts', riskScore: 0.78, amount: 95000, currency: 'USD', status: 'Escalated' },
  },
];

// Stub workspace: entity resolution
class StubEntityWorkspace extends LitElement {
  static override styles = css`
    :host { display: block; padding: 16px; }
    .fields { display: grid; grid-template-columns: auto 1fr; gap: 4px 12px; font-size: 13px; }
    .label { font-weight: 600; color: var(--pages-neutral-11, #555); }
    button { margin-top: 12px; padding: 6px 16px; cursor: pointer; }
  `;
  @state() private _ctx: any = null;
  set taskContext(ctx: any) { this._ctx = ctx; }
  private _emitResult(): void {
    this.dispatchEvent(new CustomEvent('workspace-result', {
      detail: { fields: { resolvedEntityId: 'E-RESOLVED', matchScore: 0.92 }, confidence: 0.88 },
    }));
  }
  override render(): TemplateResult {
    if (!this._ctx) return html`<p>Waiting for task context...</p>`;
    return html`
      <h4 style="margin:0 0 8px">Entity Resolution Workspace</h4>
      <div class="fields">
        <span class="label">Entity IDs:</span><span>${JSON.stringify(this._ctx.commandParams?.entityIds)}</span>
        <span class="label">Case:</span><span>${this._ctx.caseId}</span>
      </div>
      <button @click=${this._emitResult}>Simulate Result (confidence: 88%)</button>
    `;
  }
}
customElements.define('stub-workspace-entity', StubEntityWorkspace);

// Stub workspace: pattern analysis
class StubPatternWorkspace extends LitElement {
  static override styles = css`:host { display: block; padding: 16px; } button { margin-top: 12px; padding: 6px 16px; cursor: pointer; }`;
  @state() private _ctx: any = null;
  set taskContext(ctx: any) { this._ctx = ctx; }
  private _emitResult(): void {
    this.dispatchEvent(new CustomEvent('workspace-result', {
      detail: { fields: { patternConfirmed: true, transactionCount: 4 }, confidence: 0.72 },
    }));
  }
  override render(): TemplateResult {
    if (!this._ctx) return html`<p>Waiting...</p>`;
    return html`
      <h4 style="margin:0 0 8px">Pattern Analysis Workspace</h4>
      <p style="font-size:13px">Pattern: ${this._ctx.commandParams?.patternType}, Window: ${this._ctx.commandParams?.windowHours}h</p>
      <button @click=${this._emitResult}>Simulate Result (confidence: 72%)</button>
    `;
  }
}
customElements.define('stub-workspace-pattern', StubPatternWorkspace);

// Stub workspace: OSINT screening
class StubOsintWorkspace extends LitElement {
  static override styles = css`:host { display: block; padding: 16px; } button { margin-top: 12px; padding: 6px 16px; cursor: pointer; }`;
  @state() private _ctx: any = null;
  set taskContext(ctx: any) { this._ctx = ctx; }
  private _emitResult(): void {
    this.dispatchEvent(new CustomEvent('workspace-result', {
      detail: { fields: { pepConfirmed: true, sanctionsHit: false }, confidence: 0.95 },
    }));
  }
  override render(): TemplateResult {
    if (!this._ctx) return html`<p>Waiting...</p>`;
    return html`
      <h4 style="margin:0 0 8px">OSINT Screening Workspace</h4>
      <p style="font-size:13px">Type: ${this._ctx.commandParams?.screeningType}, Region: ${this._ctx.commandParams?.region}</p>
      <button @click=${this._emitResult}>Simulate Result (confidence: 95%)</button>
    `;
  }
}
customElements.define('stub-workspace-osint', StubOsintWorkspace);

// Stub context tabs
class StubContextSummary extends LitElement {
  static override styles = css`:host { display: block; padding: 12px; font-size: 13px; } .row { display: flex; gap: 8px; margin-bottom: 4px; } .key { font-weight: 600; min-width: 100px; }`;
  @state() item: any = null;
  override render(): TemplateResult {
    if (!this.item) return html`<p>No item</p>`;
    const summary = this.item.investigationSummary ?? {};
    return html`${Object.entries(summary).map(([k, v]) => html`<div class="row"><span class="key">${k}:</span><span>${String(v)}</span></div>`)}`;
  }
}
customElements.define('stub-context-summary', StubContextSummary);

class StubContextHistory extends LitElement {
  static override styles = css`:host { display: block; padding: 12px; font-size: 13px; color: var(--pages-neutral-9, #888); }`;
  @state() item: any = null;
  override render(): TemplateResult {
    return html`<p>History for case ${this.item?.caseId ?? '—'} would appear here.</p>`;
  }
}
customElements.define('stub-context-history', StubContextHistory);

const WORKSPACES: WorkspaceDefinition[] = [
  { capabilityTag: 'entity-resolution', tagName: 'stub-workspace-entity', label: 'Entity Resolution' },
  { capabilityTag: 'pattern-analysis', tagName: 'stub-workspace-pattern', label: 'Pattern Analysis' },
  { capabilityTag: 'osint-screening', tagName: 'stub-workspace-osint', label: 'OSINT Screening' },
];

const CONTEXT_TABS: TabDefinition[] = [
  { id: 'summary', label: 'Summary', tagName: 'stub-context-summary' },
  { id: 'history', label: 'History', tagName: 'stub-context-history' },
];

@customElement('blocks-example-worker-task-pane')
export class WorkerTaskPanePage extends LitElement {
  @state() private _layout: 'split' | 'stacked' = 'split';
  @state() private _showContext = true;
  @state() private _showWorkspace = true;
  @state() private _claimEnabled = false;
  @state() private _eventLog: string[] = [];

  static override styles = css`
    :host { display: flex; flex-direction: column; padding: 24px; height: 100%; box-sizing: border-box; }
    h2 { margin: 0 0 8px; font-size: 20px; font-weight: 600; color: var(--pages-neutral-12, #111); flex-shrink: 0; }
    .controls { display: flex; gap: 12px; align-items: center; margin-bottom: 16px; flex-shrink: 0; flex-wrap: wrap; }
    .controls label { font-size: 13px; color: var(--pages-neutral-11, #555); font-weight: 600; display: flex; align-items: center; gap: 4px; }
    .controls select { padding: 6px 12px; border: 1px solid var(--pages-neutral-6, #ccc); border-radius: 4px; font-size: 13px; }
    .pane-container { flex: 1; min-height: 0; border: 1px solid var(--pages-neutral-5, #e0e0e0); border-radius: 6px; overflow: hidden; }
    .event-log { flex-shrink: 0; margin-top: 12px; padding: 8px 12px; background: var(--pages-neutral-2, #f5f5f5); border-radius: 4px; max-height: 80px; overflow-y: auto; font-size: 12px; font-family: monospace; color: var(--pages-neutral-11, #555); }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.addEventListener('worker-task:responded', this._logAction);
    this.addEventListener('worker-task:declined', this._logAction);
    this.addEventListener('worker-task:claimed', this._logAction);
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this.removeEventListener('worker-task:responded', this._logAction);
    this.removeEventListener('worker-task:declined', this._logAction);
    this.removeEventListener('worker-task:claimed', this._logAction);
  }

  private _logAction = (e: Event): void => {
    const ce = e as CustomEvent;
    this._eventLog = [
      `[${new Date().toLocaleTimeString()}] ${ce.type}: ${JSON.stringify(ce.detail).slice(0, 100)}`,
      ...this._eventLog.slice(0, 9),
    ];
  };

  override render(): TemplateResult {
    return html`
      <h2>Worker Task Pane</h2>
      <div class="controls">
        <label>Layout:
          <select @change=${(e: Event) => { this._layout = (e.target as HTMLSelectElement).value as any; }}>
            <option value="split">Split</option>
            <option value="stacked">Stacked</option>
          </select>
        </label>
        <label><input type="checkbox" ?checked=${this._showContext}
          @change=${(e: Event) => { this._showContext = (e.target as HTMLInputElement).checked; }}>
          Show Context</label>
        <label><input type="checkbox" ?checked=${this._showWorkspace}
          @change=${(e: Event) => { this._showWorkspace = (e.target as HTMLInputElement).checked; }}>
          Show Workspace</label>
        <label><input type="checkbox" ?checked=${this._claimEnabled}
          @change=${(e: Event) => { this._claimEnabled = (e.target as HTMLInputElement).checked; }}>
          Enable Claim</label>
      </div>
      <div class="pane-container">
        <blocks-worker-task-pane
          .layout=${this._layout}
          .data=${SEED_TASKS}
          .workspaces=${WORKSPACES}
          .contextTabs=${CONTEXT_TABS}
          .identity=${{ userId: 'user-demo', displayName: 'Demo User', groups: ['entity-resolution', 'pattern-analysis', 'osint-screening'] }}
          .declineReasons=${['Out of clearance', 'Insufficient data', 'Conflict of interest']}
          ?show-context=${this._showContext}
          ?show-workspace=${this._showWorkspace}
          claim-endpoint=${this._claimEnabled ? '/api/mock-claim' : ''}
          selection-topic="demo-worker-task"
        ></blocks-worker-task-pane>
      </div>
      ${this._eventLog.length ? html`<div class="event-log">${this._eventLog.map(l => html`<div>${l}</div>`)}</div>` : ''}
    `;
  }
}
```

- [ ] **Step 2: Add to examples/src/main.ts**

Add to `PAGE_MODULES` array:
```typescript
'./pages/worker-task-pane-page.js',
```

- [ ] **Step 3: Add to examples/src/shell.ts**

Add nav item to the `Composed` section:
```typescript
{ id: 'worker-task-pane', label: 'Worker Task Pane', hash: '#composed/worker-task-pane' },
```

Add case to the `renderPage()` switch:
```typescript
case '#composed/worker-task-pane': return html`<blocks-example-worker-task-pane></blocks-example-worker-task-pane>`;
```

- [ ] **Step 4: Run the dev server and verify in browser**

Run: `cd examples && yarn dev`
Navigate to the Worker Task Pane page. Verify:
- Split layout renders with task queue left, detail right
- Clicking a task shows context tabs and workspace
- "Simulate Result" button enables Submit
- Submit dispatches event (visible in event log)
- Decline flow works
- Layout toggle switches between split and stacked
- Section visibility checkboxes hide/show sections
- Claim toggle gates response form behind claim button

- [ ] **Step 5: Commit**

```bash
git add examples/src/pages/worker-task-pane-page.ts examples/src/main.ts examples/src/shell.ts
git commit -m "feat(examples): add worker-task-pane showcase with seeded data Refs #152"
```

---

## Batch 4: CLAUDE.md and docs

### Task 6: Update CLAUDE.md component inventory and docs

**Files:**
- Modify: `CLAUDE.md` — add worker-task-pane to Key Directories table
- Modify: `docs/guides/consumer-guide.md` — add worker-task-pane section (if exists)

**Interfaces:**
- Consumes: Final component API from Tasks 1–4

- [ ] **Step 1: Add entry to CLAUDE.md Key Directories table**

Add after the `case-explorer` entry:
```markdown
| `components/worker-task-pane/` | Worker task pane — generic specialist task queue with context tabs, workspace element registry (WorkspaceDefinition[]), response/claim/decline form. Extends LiveRegionMixin. DataSourceMixin-free (direct fetch like work-item-inbox). Optional SSE via eventStreamEndpoint + SSEManager. |
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add worker-task-pane to component inventory Refs #152"
```

## References

- [2026-09-03-worker-task-pane-design.md](/Users/mdproctor/claude/public/casehub/blocks-ui/specs/issue-152-worker-task-pane/2026-09-03-worker-task-pane-design.md) — design spec this plan implements
- [decisions.md](/Users/mdproctor/claude/public/casehub/blocks-ui/specs/issue-152-worker-task-pane/decisions.md) — decision log (D1–D9)
- [components/work-item-inbox/src/work-item-inbox.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/work-item-inbox/src/work-item-inbox.ts) — direct fetch + SSEManager pattern, fromRows(), identity filtering
- [components/detail-pane/src/types.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/detail-pane/src/types.ts) — TabDefinition reused for contextTabs
- [components/detail-pane/src/detail-pane.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/detail-pane/src/detail-pane.ts) — element caching pattern (_tabElements Map)
- [examples/src/pages/channel-activity-page.ts](/Users/mdproctor/claude/casehub/blocks-ui/examples/src/pages/channel-activity-page.ts) — showcase page pattern with seed data
- [examples/src/shell.ts](/Users/mdproctor/claude/casehub/blocks-ui/examples/src/shell.ts) — nav items and case routing
- [component-customisation-pattern protocol (PP-20260713-8ea1af)](/Users/mdproctor/claude/casehub/blocks-ui/docs/protocols/blocks-ui/component-customisation-pattern.md) — no slots for content
- [GitHub #152](https://github.com/casehubio/blocks-ui/issues/152) — focal issue
