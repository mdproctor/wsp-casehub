# Generic Workbench Components Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #37 — AML workbench component promotion + blocks-ui consumption
**Issue group:** #37, #35, #44

**Goal:** Build three generic workbench components (`split-workbench`, `list-pane`, `detail-pane`), refactor `work-item-workbench` to use `split-workbench`, and migrate AML to consume the generic components — eliminating all duplicated workbench code.

**Architecture:** Slots-based composition. `split-workbench` is a pure layout shell accepting any children via named slots. `list-pane` wraps `pages-data-table` with data fetching. `detail-pane` renders pluggable tabs configured via a property array. All three coordinate via `pages-event` on `document` using a shared `selection-topic` prefix.

**Tech Stack:** TypeScript, Lit 3, Web Components, vitest + jsdom, `@casehubio/pages-component` event helpers, `DataEndpointMixin` from `blocks-ui-core`

## Global Constraints

- All event topics use colon separators (`:`) per `matchesTopic()` protocol
- All components use `--pages-*` CSS custom properties from `pages-ui-tokens`
- Endpoint guards use `== null` (not `!`) to allow empty-string endpoints
- Package naming: `@casehubio/blocks-ui-<name>`
- TypeScript: `strict`, `ES2022`, `ESNext` module, `verbatimModuleSyntax`, `experimentalDecorators`, `useDefineForClassFields: false`
- Tests: vitest + jsdom, `document.createElement()` + `document.body.appendChild()` pattern
- All code operations use IntelliJ MCP — no bash for source file manipulation

---

### Task 1: Prerequisites — Event Topic Migration + DataEndpointMixin Guard Fix

**Files:**
- Modify: `packages/blocks-ui-core/src/types/events.ts`
- Modify: `packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts`
- Modify: `packages/blocks-ui-core/src/data-endpoint/data-endpoint.test.ts`

**Interfaces:**
- Produces: `WorkItemEventTopics` with colon separators (`work-item:selected`, `work-item:deselected`, `queue:scope-changed`)
- Produces: `DataEndpointMixin` with `== null` endpoint guard

- [ ] **Step 1: Update event topic constants from dots to colons**

In `packages/blocks-ui-core/src/types/events.ts`, change:

```typescript
export const WorkItemEventTopics = {
  SELECTED: 'work-item:selected',
  DESELECTED: 'work-item:deselected',
  QUEUE_SCOPE_CHANGED: 'queue:scope-changed',
} as const;
```

All consumers use the constants — no call-site changes needed.

- [ ] **Step 2: Write failing test for empty-string endpoint**

In `packages/blocks-ui-core/src/data-endpoint/data-endpoint.test.ts`, add:

```typescript
it('fetches when endpoint is empty string', async () => {
  const el = createTestElement();
  el.endpoint = '';
  document.body.appendChild(el);
  await el.updateComplete;
  expect(el.fetchDataCalled).toBe(true);
});
```

Run: `yarn --cwd packages/blocks-ui-core test`
Expected: FAIL — empty string triggers `!this.endpoint` guard

- [ ] **Step 3: Fix DataEndpointMixin endpoint guard**

In `packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts`:

Line 59: Change `if (!this.endpoint) return;` to `if (this.endpoint == null) return;`

Line 45-46: Change `this.endpoint` truthy check in `willUpdate` to `this.endpoint != null`:

```typescript
if (!this._configurePending && changed.has('endpoint') && this.endpoint != null) {
```

- [ ] **Step 4: Run tests to verify**

Run: `yarn --cwd packages/blocks-ui-core test`
Expected: ALL PASS

- [ ] **Step 5: Run full typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/types/events.ts packages/blocks-ui-core/src/data-endpoint/data-endpoint.ts packages/blocks-ui-core/src/data-endpoint/data-endpoint.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "fix: migrate event topics to colon separators and fix DataEndpointMixin endpoint guard (#37)"
```

---

### Task 2: `split-workbench` Component

**Files:**
- Create: `components/split-workbench/package.json`
- Create: `components/split-workbench/tsconfig.json`
- Create: `components/split-workbench/tsconfig.build.json`
- Create: `components/split-workbench/vitest.config.ts`
- Create: `components/split-workbench/src/index.ts`
- Create: `components/split-workbench/src/split-workbench.ts`
- Create: `components/split-workbench/src/split-workbench.test.ts`
- Modify: `tsconfig.json` (root — add reference)

**Interfaces:**
- Consumes: `onPagesEvent`, `emitPagesEvent`, `LiveRegionMixin` from `@casehubio/blocks-ui-core`
- Produces: `<split-workbench>` custom element with `selection-topic`, `title`, `storage-key` properties and `header`, `list`, `detail` slots

- [ ] **Step 1: Create package scaffolding**

`components/split-workbench/package.json`:
```json
{
  "name": "@casehubio/blocks-ui-split-workbench",
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

`components/split-workbench/tsconfig.json`:
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
  "references": [
    { "path": "../../packages/blocks-ui-core" }
  ]
}
```

`components/split-workbench/tsconfig.build.json`:
```json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

`components/split-workbench/vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@casehubio/pages-ui-tokens': path.resolve(__dirname, '../../../pages/packages/pages-ui-tokens/src'),
      '@casehubio/pages-component': path.resolve(__dirname, '../../../pages/packages/pages-component/src'),
      '@casehubio/pages-data/dist/sse/sse-manager.js': path.resolve(__dirname, '../../../pages/packages/pages-data/src/sse/sse-manager.ts'),
      '@casehubio/pages-data': path.resolve(__dirname, '../../../pages/packages/pages-data/src'),
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

`components/split-workbench/src/index.ts`:
```typescript
export { SplitWorkbench } from './split-workbench.js';
```

Add to root `tsconfig.json` references: `{ "path": "components/split-workbench" }`

Run: `yarn install`

- [ ] **Step 2: Write failing tests for split-workbench**

`components/split-workbench/src/split-workbench.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { emitPagesEvent } from '@casehubio/blocks-ui-core';
import './split-workbench.js';

type SplitWorkbenchEl = HTMLElement & {
  selectionTopic: string;
  title: string;
  storageKey: string;
  updateComplete: Promise<boolean>;
};

function createElement(topic = 'test'): SplitWorkbenchEl {
  const el = document.createElement('split-workbench') as SplitWorkbenchEl;
  el.selectionTopic = topic;
  return el;
}

describe('split-workbench', () => {
  let el: SplitWorkbenchEl;

  beforeEach(async () => {
    el = createElement();
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => {
    el.remove();
    vi.restoreAllMocks();
    localStorage.clear();
  });

  it('renders list and detail slots', () => {
    const listSlot = el.shadowRoot!.querySelector('slot[name="list"]');
    const detailSlot = el.shadowRoot!.querySelector('slot[name="detail"]');
    expect(listSlot).toBeTruthy();
    expect(detailSlot).toBeTruthy();
  });

  it('renders header slot with title fallback', () => {
    el.title = 'Test Workbench';
    return el.updateComplete.then(() => {
      const headerSlot = el.shadowRoot!.querySelector('slot[name="header"]');
      expect(headerSlot).toBeTruthy();
      expect(el.shadowRoot!.textContent).toContain('Test Workbench');
    });
  });

  it('renders draggable divider with separator role', () => {
    const divider = el.shadowRoot!.querySelector('[role="separator"]');
    expect(divider).toBeTruthy();
    expect(divider!.getAttribute('aria-orientation')).toBe('vertical');
    expect(divider!.getAttribute('aria-valuemin')).toBe('20');
    expect(divider!.getAttribute('aria-valuemax')).toBe('70');
  });

  it('renders list and detail panels with region roles', () => {
    const regions = el.shadowRoot!.querySelectorAll('[role="region"]');
    const labels = Array.from(regions).map(r => r.getAttribute('aria-label'));
    expect(labels).toContain('List');
    expect(labels).toContain('Detail');
  });

  it('persists divider position to localStorage using storage-key', async () => {
    el.storageKey = 'test-divider';
    await el.updateComplete;

    const divider = el.shadowRoot!.querySelector('[role="separator"]') as HTMLElement;
    const splitPane = el.shadowRoot!.querySelector('.split-pane') as HTMLElement;

    Object.defineProperty(splitPane, 'getBoundingClientRect', {
      value: () => ({ left: 0, width: 1000, top: 0, height: 500, right: 1000, bottom: 500 }),
    });

    divider.dispatchEvent(new MouseEvent('mousedown', { bubbles: true }));
    document.dispatchEvent(new MouseEvent('mousemove', { clientX: 400 }));
    document.dispatchEvent(new MouseEvent('mouseup'));

    expect(localStorage.getItem('test-divider')).toBeTruthy();
  });

  it('defaults storage-key from selection-topic', () => {
    expect(el.storageKey).toBe('split-workbench-test-divider');
  });

  it('sets has-selection attribute on topic:selected event', async () => {
    expect(el.hasAttribute('has-selection')).toBe(false);
    emitPagesEvent(document, 'test:selected', { id: '1' });
    await el.updateComplete;
    expect(el.hasAttribute('has-selection')).toBe(true);
  });

  it('removes has-selection attribute on topic:deselected event', async () => {
    emitPagesEvent(document, 'test:selected', { id: '1' });
    await el.updateComplete;
    expect(el.hasAttribute('has-selection')).toBe(true);

    emitPagesEvent(document, 'test:deselected', {});
    await el.updateComplete;
    expect(el.hasAttribute('has-selection')).toBe(false);
  });

  it('renders back button when has-selection is true', async () => {
    emitPagesEvent(document, 'test:selected', { id: '1' });
    await el.updateComplete;
    const backBtn = el.shadowRoot!.querySelector('[aria-label="Back to list"]');
    expect(backBtn).toBeTruthy();
  });

  it('back button emits topic:deselected and clears has-selection', async () => {
    emitPagesEvent(document, 'test:selected', { id: '1' });
    await el.updateComplete;

    const events: unknown[] = [];
    document.addEventListener('pages-event', ((e: CustomEvent) => {
      if (e.detail?.topic === 'test:deselected') events.push(e.detail);
    }) as EventListener);

    const backBtn = el.shadowRoot!.querySelector('[aria-label="Back to list"]') as HTMLElement;
    backBtn.click();
    await el.updateComplete;

    expect(events.length).toBe(1);
    expect(el.hasAttribute('has-selection')).toBe(false);
  });

  it('does not share selection state between different topics', async () => {
    const el2 = createElement('other');
    document.body.appendChild(el2);
    await el2.updateComplete;

    emitPagesEvent(document, 'test:selected', { id: '1' });
    await el.updateComplete;
    await el2.updateComplete;

    expect(el.hasAttribute('has-selection')).toBe(true);
    expect(el2.hasAttribute('has-selection')).toBe(false);
    el2.remove();
  });

  it('warns in dev mode when selection-topic is not set', async () => {
    const warn = vi.spyOn(console, 'warn').mockImplementation(() => {});
    const noTopic = document.createElement('split-workbench') as SplitWorkbenchEl;
    document.body.appendChild(noTopic);
    await noTopic.updateComplete;
    expect(warn).toHaveBeenCalledWith(expect.stringContaining('selection-topic'));
    noTopic.remove();
  });

  it('always renders both slots regardless of selection state', async () => {
    const listSlot = el.shadowRoot!.querySelector('slot[name="list"]');
    const detailSlot = el.shadowRoot!.querySelector('slot[name="detail"]');
    expect(listSlot).toBeTruthy();
    expect(detailSlot).toBeTruthy();

    emitPagesEvent(document, 'test:selected', { id: '1' });
    await el.updateComplete;

    const listSlotAfter = el.shadowRoot!.querySelector('slot[name="list"]');
    const detailSlotAfter = el.shadowRoot!.querySelector('slot[name="detail"]');
    expect(listSlotAfter).toBeTruthy();
    expect(detailSlotAfter).toBeTruthy();
  });

  it('adjusts divider with arrow keys', async () => {
    const divider = el.shadowRoot!.querySelector('[role="separator"]') as HTMLElement;
    const initial = divider.getAttribute('aria-valuenow');

    divider.dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowRight' }));
    await el.updateComplete;

    const after = divider.getAttribute('aria-valuenow');
    expect(Number(after)).toBeGreaterThan(Number(initial));
  });

  it('cleans up event listeners on disconnect', async () => {
    el.remove();
    emitPagesEvent(document, 'test:selected', { id: '1' });
    await new Promise(r => setTimeout(r, 0));
    expect(el.hasAttribute('has-selection')).toBe(false);
  });
});
```

Run: `yarn --cwd components/split-workbench test`
Expected: FAIL — split-workbench.js not found

- [ ] **Step 3: Implement split-workbench**

`components/split-workbench/src/split-workbench.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { onPagesEvent, emitPagesEvent, LiveRegionMixin } from '@casehubio/blocks-ui-core';

@customElement('split-workbench')
export class SplitWorkbench extends LiveRegionMixin(LitElement) {
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = '';
  @property({ type: String }) title = '';
  @property({ type: String, attribute: 'storage-key' })
  get storageKey(): string {
    return this._storageKey ?? `split-workbench-${this.selectionTopic}-divider`;
  }
  set storageKey(v: string) { this._storageKey = v; }
  private _storageKey?: string;

  @state() private _hasSelection = false;
  @state() private _dividerRatio = 0.5;
  @state() private _isDragging = false;

  private _unsubs: Array<() => void> = [];

  static override styles = css`
    :host {
      display: block;
      height: 100%;
      width: 100%;
      font-family: var(--pages-font-family, system-ui);
      overflow: hidden;
      container-type: inline-size;
    }

    .wrapper {
      display: flex;
      flex-direction: column;
      height: 100%;
    }

    .header {
      padding: var(--pages-space-3, 12px) var(--pages-space-4, 16px);
      border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
      background: var(--pages-neutral-1, #fafafa);
    }

    .header h2 {
      margin: 0;
      font-size: var(--pages-font-size-lg, 16px);
      font-weight: 600;
      color: var(--pages-neutral-11, #0a0a0a);
    }

    .split-pane {
      display: flex;
      flex: 1;
      overflow: hidden;
    }

    .list-panel {
      display: flex;
      flex-direction: column;
      flex: 0 0 auto;
      overflow: hidden;
    }

    .divider {
      width: 4px;
      background: var(--pages-neutral-3, #e5e5e5);
      cursor: col-resize;
      flex-shrink: 0;
    }

    .divider:hover, .divider:focus-visible, .divider.dragging {
      background: var(--pages-accent-9, #3b82f6);
      outline: none;
    }

    .detail-panel {
      flex: 1;
      overflow: hidden;
    }

    .back-button {
      display: none;
      align-items: center;
      gap: var(--pages-space-1, 4px);
      padding: var(--pages-space-2, 8px) var(--pages-space-3, 12px);
      background: none;
      border: none;
      border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
      cursor: pointer;
      font-size: var(--pages-font-size-sm, 12px);
      color: var(--pages-accent-9, #3b82f6);
      width: 100%;
    }

    @container (max-width: 768px) {
      .divider { display: none; }

      .list-panel {
        width: 100% !important;
        min-width: unset !important;
        max-width: unset !important;
      }

      :host(:not([has-selection])) .detail-panel { display: none; }
      :host([has-selection]) .list-panel { display: none; }
      :host([has-selection]) .detail-panel { flex: 1; }
      :host([has-selection]) .back-button { display: flex; }
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    if (!this.selectionTopic) {
      console.warn('[split-workbench] selection-topic is required but not set.');
    }
    this._restoreDivider();
    this._subscribeEvents();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._unsubs.forEach(u => u());
    this._unsubs = [];
  }

  override updated(changed: Map<PropertyKey, unknown>): void {
    super.updated(changed);
    if (changed.has('_dividerRatio')) {
      this.style.setProperty('--divider-ratio', this._dividerRatio.toString());
    }
    if (changed.has('_hasSelection')) {
      if (this._hasSelection) {
        this.setAttribute('has-selection', '');
        this.announce('Showing detail');
        this._focusSlot('detail');
      } else {
        this.removeAttribute('has-selection');
        this.announce('Showing list');
        this._focusSlot('list');
      }
    }
  }

  private _focusSlot(name: string): void {
    const slot = this.shadowRoot?.querySelector(`slot[name="${name}"]`) as HTMLSlotElement | null;
    const child = slot?.assignedElements()[0] as HTMLElement | undefined;
    child?.focus();
  }

  private _restoreDivider(): void {
    if (typeof localStorage === 'undefined') return;
    const saved = localStorage.getItem(this.storageKey);
    if (saved) {
      const ratio = parseFloat(saved);
      if (ratio >= 0.2 && ratio <= 0.7) this._dividerRatio = ratio;
    }
  }

  private _subscribeEvents(): void {
    if (!this.selectionTopic) return;
    this._unsubs.push(
      onPagesEvent(document, `${this.selectionTopic}:selected`, () => {
        this._hasSelection = true;
      }),
      onPagesEvent(document, `${this.selectionTopic}:deselected`, () => {
        this._hasSelection = false;
      }),
    );
  }

  private _handleDividerMouseDown = (e: MouseEvent): void => {
    e.preventDefault();
    this._isDragging = true;
    document.addEventListener('mousemove', this._handleMouseMove);
    document.addEventListener('mouseup', this._handleMouseUp);
  };

  private _handleMouseMove = (e: MouseEvent): void => {
    const pane = this.shadowRoot?.querySelector('.split-pane') as HTMLElement | null;
    if (!pane) return;
    const rect = pane.getBoundingClientRect();
    const ratio = Math.max(0.2, Math.min(0.7, (e.clientX - rect.left) / rect.width));
    this._dividerRatio = ratio;
    this.style.setProperty('--divider-ratio', ratio.toString());
  };

  private _handleMouseUp = (): void => {
    this._isDragging = false;
    document.removeEventListener('mousemove', this._handleMouseMove);
    document.removeEventListener('mouseup', this._handleMouseUp);
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem(this.storageKey, this._dividerRatio.toString());
    }
  };

  private _handleDividerKeyDown = (e: KeyboardEvent): void => {
    const step = 0.05;
    if (e.key === 'ArrowRight') {
      this._dividerRatio = Math.min(0.7, this._dividerRatio + step);
    } else if (e.key === 'ArrowLeft') {
      this._dividerRatio = Math.max(0.2, this._dividerRatio - step);
    } else {
      return;
    }
    e.preventDefault();
    if (typeof localStorage !== 'undefined') {
      localStorage.setItem(this.storageKey, this._dividerRatio.toString());
    }
  };

  private _handleBack = (): void => {
    this._hasSelection = false;
    if (this.selectionTopic) {
      emitPagesEvent(document, `${this.selectionTopic}:deselected`, {});
    }
  };

  override render() {
    const dividerPercent = Math.round(this._dividerRatio * 100);
    return html`
      <div class="wrapper">
        <div class="header">
          <slot name="header">
            ${this.title ? html`<h2>${this.title}</h2>` : nothing}
          </slot>
        </div>

        <div class="split-pane">
          <div class="list-panel" role="region" aria-label="List"
               style="width: ${dividerPercent}%; min-width: 320px; max-width: 70%">
            <slot name="list"></slot>
          </div>

          <div class="divider ${this._isDragging ? 'dragging' : ''}"
               role="separator"
               aria-orientation="vertical"
               aria-valuenow="${dividerPercent}"
               aria-valuemin="20"
               aria-valuemax="70"
               tabindex="0"
               @mousedown=${this._handleDividerMouseDown}
               @keydown=${this._handleDividerKeyDown}></div>

          <div class="detail-panel" role="region" aria-label="Detail">
            <button class="back-button"
                    aria-label="Back to list"
                    @click=${this._handleBack}>
              ← Back to list
            </button>
            <slot name="detail"></slot>
          </div>
        </div>
      </div>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'split-workbench': SplitWorkbench;
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd components/split-workbench test`
Expected: ALL PASS

- [ ] **Step 5: Typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/split-workbench/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add split-workbench — generic slots-based layout shell (#37)"
```

---

### Task 3: `list-pane` Component

**Files:**
- Create: `components/list-pane/package.json`
- Create: `components/list-pane/tsconfig.json`
- Create: `components/list-pane/tsconfig.build.json`
- Create: `components/list-pane/vitest.config.ts`
- Create: `components/list-pane/src/index.ts`
- Create: `components/list-pane/src/list-pane.ts`
- Create: `components/list-pane/src/list-pane.test.ts`
- Modify: `tsconfig.json` (root — add reference)

**Interfaces:**
- Consumes: `DataEndpointMixin`, `emitPagesEvent`, `onPagesEvent` from `@casehubio/blocks-ui-core`; `ColumnDef` from `@casehubio/blocks-ui-data-table`
- Produces: `<list-pane>` custom element with `endpoint`, `columns`, `getRowKey`, `getRowClass`, `selection-topic`, `empty-message`, `page-size` properties; emits `{topic}:selected` events; listens for `{topic}:refresh`; public `refresh()` method

- [ ] **Step 1: Create package scaffolding**

Same pattern as Task 2. `package.json` adds data-table dependency:
```json
{
  "name": "@casehubio/blocks-ui-list-pane",
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
    "@casehubio/blocks-ui-data-table": "workspace:*",
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

`tsconfig.json` references:
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
  "references": [
    { "path": "../../packages/blocks-ui-core" },
    { "path": "../data-table" }
  ]
}
```

`tsconfig.build.json`: `{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }`

`vitest.config.ts`: Same as Task 2.

`src/index.ts`:
```typescript
export { ListPane } from './list-pane.js';
```

Add to root `tsconfig.json` references: `{ "path": "components/list-pane" }`

Run: `yarn install`

- [ ] **Step 2: Write failing tests for list-pane**

`components/list-pane/src/list-pane.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { emitPagesEvent } from '@casehubio/blocks-ui-core';
import type { ColumnDef } from '@casehubio/blocks-ui-data-table';
import './list-pane.js';

type ListPaneEl = HTMLElement & {
  endpoint: string;
  columns: ColumnDef<any>[];
  getRowKey: (row: any) => string;
  getRowClass: (row: any) => string;
  selectionTopic: string;
  emptyMessage: string;
  pageSize: number;
  fetchFn: typeof fetch;
  refresh(): void;
  updateComplete: Promise<boolean>;
};

const testColumns: ColumnDef<any>[] = [
  { id: 'name', header: 'Name', cell: (row: any) => row.name },
  { id: 'status', header: 'Status', cell: (row: any) => row.status },
];

const testRows = [
  { id: '1', name: 'Case A', status: 'open' },
  { id: '2', name: 'Case B', status: 'closed' },
];

function mockFetch(data: unknown = testRows): typeof fetch {
  return vi.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve(data),
  }) as unknown as typeof fetch;
}

function createElement(opts: Partial<{ endpoint: string; topic: string; fetchFn: typeof fetch }> = {}): ListPaneEl {
  const el = document.createElement('list-pane') as ListPaneEl;
  el.endpoint = opts.endpoint ?? '/api/items';
  el.selectionTopic = opts.topic ?? 'test';
  el.columns = testColumns;
  el.getRowKey = (row: any) => row.id;
  el.fetchFn = opts.fetchFn ?? mockFetch();
  return el;
}

describe('list-pane', () => {
  let el: ListPaneEl;

  afterEach(() => {
    el?.remove();
    vi.restoreAllMocks();
  });

  describe('data fetching', () => {
    it('fetches from endpoint on connect', async () => {
      const fetchFn = mockFetch();
      el = createElement({ fetchFn });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      expect(fetchFn).toHaveBeenCalledWith('/api/items', expect.objectContaining({ signal: expect.any(AbortSignal) }));
    });

    it('handles array response', async () => {
      el = createElement({ fetchFn: mockFetch(testRows) });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;
      const table = el.shadowRoot!.querySelector('pages-data-table') as any;
      expect(table?.rows?.length).toBe(2);
    });

    it('handles { items, total } response', async () => {
      el = createElement({ fetchFn: mockFetch({ items: testRows, total: 100 }) });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;
      const table = el.shadowRoot!.querySelector('pages-data-table') as any;
      expect(table?.rows?.length).toBe(2);
      expect(table?.totalRows).toBe(100);
    });

    it('fetches with empty-string endpoint', async () => {
      const fetchFn = mockFetch();
      el = createElement({ endpoint: '', fetchFn });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      expect(fetchFn).toHaveBeenCalled();
    });

    it('shows empty message when no data', async () => {
      el = createElement({ fetchFn: mockFetch([]) });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;
      expect(el.shadowRoot!.textContent).toContain('No items found');
    });

    it('shows custom empty message', async () => {
      el = createElement({ fetchFn: mockFetch([]) });
      el.emptyMessage = 'Nothing here';
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;
      expect(el.shadowRoot!.textContent).toContain('Nothing here');
    });
  });

  describe('selection events', () => {
    it('emits topic:selected on row activation', async () => {
      el = createElement({ fetchFn: mockFetch() });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;

      const events: any[] = [];
      document.addEventListener('pages-event', ((e: CustomEvent) => {
        if (e.detail?.topic === 'test:selected') events.push(e.detail);
      }) as EventListener);

      const table = el.shadowRoot!.querySelector('pages-data-table') as HTMLElement;
      table.dispatchEvent(new CustomEvent('row-activate', {
        bubbles: true,
        detail: { row: testRows[0], index: 0 },
      }));

      expect(events.length).toBe(1);
      expect(events[0].payload).toEqual(testRows[0]);
    });
  });

  describe('refresh', () => {
    it('refresh() re-fetches data', async () => {
      const fetchFn = mockFetch();
      el = createElement({ fetchFn });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));

      (fetchFn as any).mockClear();
      el.refresh();
      await new Promise(r => setTimeout(r, 0));
      expect(fetchFn).toHaveBeenCalled();
    });

    it('responds to topic:refresh event', async () => {
      const fetchFn = mockFetch();
      el = createElement({ fetchFn });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));

      (fetchFn as any).mockClear();
      emitPagesEvent(document, 'test:refresh', {});
      await new Promise(r => setTimeout(r, 0));
      expect(fetchFn).toHaveBeenCalled();
    });
  });

  describe('table configuration', () => {
    it('configures data-table with selection=single, mode=paginated, client-sort, client-filter', async () => {
      el = createElement({ fetchFn: mockFetch() });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;

      const table = el.shadowRoot!.querySelector('pages-data-table') as any;
      expect(table?.getAttribute('selection')).toBe('single');
      expect(table?.getAttribute('mode')).toBe('paginated');
      expect(table?.hasAttribute('client-sort')).toBe(true);
      expect(table?.hasAttribute('client-filter')).toBe(true);
    });

    it('passes page-size to data-table', async () => {
      el = createElement({ fetchFn: mockFetch() });
      el.pageSize = 10;
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;

      const table = el.shadowRoot!.querySelector('pages-data-table') as any;
      expect(table?.pageSize).toBe(10);
    });
  });

  describe('accessibility', () => {
    it('host has tabindex -1 for programmatic focus', async () => {
      el = createElement({ fetchFn: mockFetch() });
      document.body.appendChild(el);
      await el.updateComplete;
      expect(el.getAttribute('tabindex')).toBe('-1');
    });

    it('empty state has role=status', async () => {
      el = createElement({ fetchFn: mockFetch([]) });
      document.body.appendChild(el);
      await el.updateComplete;
      await new Promise(r => setTimeout(r, 0));
      await el.updateComplete;
      const status = el.shadowRoot!.querySelector('[role="status"]');
      expect(status).toBeTruthy();
    });
  });
});
```

Run: `yarn --cwd components/list-pane test`
Expected: FAIL

- [ ] **Step 3: Implement list-pane**

`components/list-pane/src/list-pane.ts`:

```typescript
import { html, css, nothing, type PropertyValues } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { DataEndpointMixin, emitPagesEvent, onPagesEvent } from '@casehubio/blocks-ui-core';
import { LitElement } from 'lit';
import type { ColumnDef } from '@casehubio/blocks-ui-data-table';
import '@casehubio/blocks-ui-data-table';

@customElement('list-pane')
export class ListPane extends DataEndpointMixin(LitElement) {
  @property({ type: Array }) columns: ColumnDef<any>[] = [];
  @property({ attribute: false }) getRowKey?: (row: unknown) => string;
  @property({ attribute: false }) getRowClass?: (row: unknown) => string;
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = '';
  @property({ type: String, attribute: 'empty-message' }) emptyMessage = 'No items found';
  @property({ type: Number, attribute: 'page-size' }) pageSize = 25;

  @state() private _rows: readonly unknown[] = [];
  @state() private _totalRows?: number;
  @state() private _lastActivatedIndex = -1;

  private _refreshUnsub?: () => void;

  static override styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100%;
      overflow: hidden;
    }

    .empty {
      display: flex;
      align-items: center;
      justify-content: center;
      height: 100%;
      color: var(--pages-neutral-7, #525252);
      font-size: var(--pages-font-size-sm, 12px);
    }

    pages-data-table {
      flex: 1;
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute('tabindex', '-1');
    if (this.selectionTopic) {
      this._refreshUnsub = onPagesEvent(document, `${this.selectionTopic}:refresh`, () => {
        this.refresh();
      });
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._refreshUnsub?.();
  }

  async fetchData(): Promise<void> {
    const resp = await this.fetchFn(this.endpoint!, { signal: this.abortSignal });
    if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
    const data = await resp.json();

    if (Array.isArray(data)) {
      this._rows = data;
      this._totalRows = undefined;
    } else if (data && Array.isArray(data.items)) {
      this._rows = data.items;
      this._totalRows = data.total;
    } else {
      this._rows = [];
      this._totalRows = undefined;
    }
  }

  refresh(): void {
    this._doFetch();
  }

  private _handleRowActivate = (e: Event): void => {
    const detail = (e as CustomEvent).detail;
    this._lastActivatedIndex = detail.index ?? -1;
    if (this.selectionTopic && detail.row) {
      emitPagesEvent(document, `${this.selectionTopic}:selected`, detail.row);
    }
  };

  override focus(): void {
    const table = this.shadowRoot?.querySelector('pages-data-table') as HTMLElement | undefined;
    table?.focus();
  }

  override render() {
    if (!this.loading && this._rows.length === 0 && !this.error) {
      return html`<div class="empty" role="status">${this.emptyMessage}</div>`;
    }

    return html`
      <pages-data-table
        .rows=${this._rows}
        .columns=${this.columns}
        .getRowKey=${this.getRowKey}
        .getRowClass=${this.getRowClass}
        .totalRows=${this._totalRows}
        .pageSize=${this.pageSize}
        .loading=${this.loading}
        .emptyMessage=${this.emptyMessage}
        selection="single"
        mode="paginated"
        client-sort
        client-filter
        @row-activate=${this._handleRowActivate}
      ></pages-data-table>
    `;
  }

  private _doFetch(): void {
    if (this.endpoint == null) return;
    (this as any)._abortController?.abort();
    (this as any)._abortController = new AbortController();
    this.loading = true;
    this.error = null;
    this.fetchData()
      .catch((e: unknown) => {
        if (e instanceof DOMException && e.name === 'AbortError') return;
        this.error = e instanceof Error ? e.message : String(e);
      })
      .finally(() => { this.loading = false; });
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'list-pane': ListPane;
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd components/list-pane test`
Expected: ALL PASS

- [ ] **Step 5: Typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/list-pane/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add list-pane — generic data-table wrapper with selection events (#37)"
```

---

### Task 4: `detail-pane` Component

**Files:**
- Create: `components/detail-pane/package.json`
- Create: `components/detail-pane/tsconfig.json`
- Create: `components/detail-pane/tsconfig.build.json`
- Create: `components/detail-pane/vitest.config.ts`
- Create: `components/detail-pane/src/index.ts`
- Create: `components/detail-pane/src/types.ts`
- Create: `components/detail-pane/src/detail-pane.ts`
- Create: `components/detail-pane/src/detail-pane.test.ts`
- Modify: `tsconfig.json` (root — add reference)

**Interfaces:**
- Consumes: `onPagesEvent`, `LiveRegionMixin` from `@casehubio/blocks-ui-core`
- Produces: `<detail-pane>` custom element with `tabs`, `selection-topic`, `empty-message` properties; `TabDefinition` interface

- [ ] **Step 1: Create package scaffolding**

`package.json`:
```json
{
  "name": "@casehubio/blocks-ui-detail-pane",
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
  "references": [
    { "path": "../../packages/blocks-ui-core" }
  ]
}
```

`tsconfig.build.json`: `{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }`

`vitest.config.ts`: Same pattern as Task 2.

`src/types.ts`:
```typescript
export interface TabDefinition {
  id: string;
  label: string;
  tagName: string;
  icon?: string;
  order?: number;
  badge?: (item: unknown) => string | null;
}
```

`src/index.ts`:
```typescript
export { DetailPane } from './detail-pane.js';
export type { TabDefinition } from './types.js';
```

Add to root `tsconfig.json` references: `{ "path": "components/detail-pane" }`

Run: `yarn install`

- [ ] **Step 2: Write failing tests for detail-pane**

`components/detail-pane/src/detail-pane.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { emitPagesEvent } from '@casehubio/blocks-ui-core';
import { LitElement, html } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { TabDefinition } from './types.js';
import './detail-pane.js';

@customElement('test-tab-panel')
class TestTabPanel extends LitElement {
  @property({ attribute: false }) item: any;
  override render() {
    return html`<div class="tab-content">${this.item ? `Item: ${this.item.id}` : 'No item'}</div>`;
  }
}

type DetailPaneEl = HTMLElement & {
  tabs: TabDefinition[];
  selectionTopic: string;
  emptyMessage: string;
  updateComplete: Promise<boolean>;
};

const testTabs: TabDefinition[] = [
  { id: 'overview', label: 'Overview', tagName: 'test-tab-panel', order: 0 },
  { id: 'details', label: 'Details', tagName: 'test-tab-panel', order: 10 },
];

function createElement(topic = 'test'): DetailPaneEl {
  const el = document.createElement('detail-pane') as DetailPaneEl;
  el.selectionTopic = topic;
  el.tabs = testTabs;
  return el;
}

describe('detail-pane', () => {
  let el: DetailPaneEl;

  afterEach(() => {
    el?.remove();
    vi.restoreAllMocks();
  });

  describe('empty state', () => {
    it('shows empty message when no item selected', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;
      expect(el.shadowRoot!.textContent).toContain('Select an item to view details');
    });

    it('shows custom empty message', async () => {
      el = createElement();
      el.emptyMessage = 'Pick something';
      document.body.appendChild(el);
      await el.updateComplete;
      expect(el.shadowRoot!.textContent).toContain('Pick something');
    });

    it('empty state has role=status', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;
      expect(el.shadowRoot!.querySelector('[role="status"]')).toBeTruthy();
    });
  });

  describe('selection', () => {
    it('shows tabs when item is selected via event', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1', name: 'Test' });
      await el.updateComplete;

      const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
      expect(tabs.length).toBe(2);
      const labels = Array.from(tabs).map(t => t.textContent!.trim());
      expect(labels).toContain('Overview');
      expect(labels).toContain('Details');
    });

    it('creates tab element and sets item property', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      const payload = { id: '1', name: 'Test' };
      emitPagesEvent(document, 'test:selected', payload);
      await el.updateComplete;

      const panel = el.shadowRoot!.querySelector('test-tab-panel') as any;
      expect(panel).toBeTruthy();
      expect(panel.item).toEqual(payload);
    });

    it('clears item on deselection', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      emitPagesEvent(document, 'test:deselected', {});
      await el.updateComplete;

      expect(el.shadowRoot!.textContent).toContain('Select an item to view details');
    });
  });

  describe('tab switching', () => {
    it('switches tabs when clicked', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]') as NodeListOf<HTMLElement>;
      expect(tabs[0].getAttribute('aria-selected')).toBe('true');
      expect(tabs[1].getAttribute('aria-selected')).toBe('false');

      tabs[1].click();
      await el.updateComplete;

      expect(tabs[0].getAttribute('aria-selected')).toBe('false');
      expect(tabs[1].getAttribute('aria-selected')).toBe('true');
    });

    it('passes item to newly active tab', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      const payload = { id: '1', name: 'Test' };
      emitPagesEvent(document, 'test:selected', payload);
      await el.updateComplete;

      const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]') as NodeListOf<HTMLElement>;
      tabs[1].click();
      await el.updateComplete;

      const panels = el.shadowRoot!.querySelectorAll('test-tab-panel') as NodeListOf<any>;
      const visiblePanel = Array.from(panels).find(p => p.item);
      expect(visiblePanel?.item).toEqual(payload);
    });
  });

  describe('tab ordering', () => {
    it('sorts tabs by order property', async () => {
      el = createElement();
      el.tabs = [
        { id: 'b', label: 'Second', tagName: 'test-tab-panel', order: 20 },
        { id: 'a', label: 'First', tagName: 'test-tab-panel', order: 5 },
      ];
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
      expect(tabs[0].textContent!.trim()).toBe('First');
      expect(tabs[1].textContent!.trim()).toBe('Second');
    });
  });

  describe('badges', () => {
    it('renders badge from badge function', async () => {
      el = createElement();
      el.tabs = [
        { id: 'a', label: 'Overview', tagName: 'test-tab-panel', badge: (item: any) => item?.count ? `${item.count}` : null },
      ];
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1', count: 5 });
      await el.updateComplete;

      expect(el.shadowRoot!.textContent).toContain('5');
    });
  });

  describe('accessibility', () => {
    it('renders tablist role', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      expect(el.shadowRoot!.querySelector('[role="tablist"]')).toBeTruthy();
    });

    it('renders tabpanel role', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      expect(el.shadowRoot!.querySelector('[role="tabpanel"]')).toBeTruthy();
    });

    it('host has tabindex -1', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;
      expect(el.getAttribute('tabindex')).toBe('-1');
    });

    it('navigates tabs with arrow keys', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      const tablist = el.shadowRoot!.querySelector('[role="tablist"]') as HTMLElement;
      const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]') as NodeListOf<HTMLElement>;

      tabs[0].focus();
      tabs[0].dispatchEvent(new KeyboardEvent('keydown', { key: 'ArrowRight', bubbles: true }));
      await el.updateComplete;

      expect(tabs[1].getAttribute('aria-selected')).toBe('true');
    });
  });

  describe('isolation', () => {
    it('does not respond to events from different topic', async () => {
      el = createElement('test');
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'other:selected', { id: '1' });
      await el.updateComplete;

      expect(el.shadowRoot!.textContent).toContain('Select an item to view details');
    });
  });

  describe('lazy creation and caching', () => {
    it('caches tab elements across tab switches', async () => {
      el = createElement();
      document.body.appendChild(el);
      await el.updateComplete;

      emitPagesEvent(document, 'test:selected', { id: '1' });
      await el.updateComplete;

      const firstPanel = el.shadowRoot!.querySelector('test-tab-panel');

      const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]') as NodeListOf<HTMLElement>;
      tabs[1].click();
      await el.updateComplete;
      tabs[0].click();
      await el.updateComplete;

      const samePanel = el.shadowRoot!.querySelector('test-tab-panel');
      expect(samePanel).toBe(firstPanel);
    });
  });
});
```

Run: `yarn --cwd components/detail-pane test`
Expected: FAIL

- [ ] **Step 3: Implement detail-pane**

`components/detail-pane/src/detail-pane.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { onPagesEvent, LiveRegionMixin } from '@casehubio/blocks-ui-core';
import type { TabDefinition } from './types.js';

@customElement('detail-pane')
export class DetailPane extends LiveRegionMixin(LitElement) {
  @property({ type: Array }) tabs: TabDefinition[] = [];
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = '';
  @property({ type: String, attribute: 'empty-message' }) emptyMessage = 'Select an item to view details';

  @state() private _item: unknown = null;
  @state() private _activeTabId = '';

  private _tabElements = new Map<string, HTMLElement>();
  private _unsubs: Array<() => void> = [];

  static override styles = css`
    :host {
      display: flex;
      flex-direction: column;
      height: 100%;
      overflow: hidden;
    }

    .empty {
      display: flex;
      align-items: center;
      justify-content: center;
      height: 100%;
      color: var(--pages-neutral-7, #525252);
      font-size: var(--pages-font-size-sm, 12px);
    }

    .tab-bar {
      display: flex;
      gap: 0;
      border-bottom: 1px solid var(--pages-neutral-4, #d4d4d4);
      background: var(--pages-neutral-2, #f5f5f5);
      overflow-x: auto;
    }

    .tab-button {
      display: flex;
      align-items: center;
      gap: var(--pages-space-1, 4px);
      padding: var(--pages-space-2, 8px) var(--pages-space-4, 16px);
      border: none;
      border-bottom: 2px solid transparent;
      background: none;
      cursor: pointer;
      font-size: var(--pages-font-size-sm, 12px);
      color: var(--pages-neutral-7, #525252);
      white-space: nowrap;
    }

    .tab-button[aria-selected="true"] {
      color: var(--pages-accent-9, #3b82f6);
      border-bottom-color: var(--pages-accent-9, #3b82f6);
      font-weight: 600;
    }

    .tab-button:hover {
      background: var(--pages-neutral-3, #e5e5e5);
    }

    .badge {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      min-width: 18px;
      height: 18px;
      padding: 0 4px;
      border-radius: 9px;
      background: var(--pages-neutral-5, #a3a3a3);
      color: var(--pages-neutral-1, #fafafa);
      font-size: 10px;
      font-weight: 600;
    }

    .tab-panel {
      flex: 1;
      overflow: auto;
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute('tabindex', '-1');
    if (this.selectionTopic) {
      this._unsubs.push(
        onPagesEvent(document, `${this.selectionTopic}:selected`, (payload: unknown) => {
          this._item = payload;
          if (!this._activeTabId && this._sortedTabs.length > 0) {
            this._activeTabId = this._sortedTabs[0].id;
          }
          this.announce(`${this._activeTab?.label ?? ''} tab`);
        }),
        onPagesEvent(document, `${this.selectionTopic}:deselected`, () => {
          this._item = null;
        }),
      );
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._unsubs.forEach(u => u());
    this._unsubs = [];
  }

  private get _sortedTabs(): TabDefinition[] {
    return [...this.tabs].sort((a, b) => (a.order ?? 0) - (b.order ?? 0));
  }

  private get _activeTab(): TabDefinition | undefined {
    return this._sortedTabs.find(t => t.id === this._activeTabId);
  }

  private _getOrCreateTabElement(tab: TabDefinition): HTMLElement {
    let el = this._tabElements.get(tab.id);
    if (!el) {
      el = document.createElement(tab.tagName);
      this._tabElements.set(tab.id, el);
    }
    (el as any).item = this._item;
    return el;
  }

  private _handleTabClick(tabId: string): void {
    this._activeTabId = tabId;
  }

  private _handleTabKeyDown = (e: KeyboardEvent): void => {
    const sorted = this._sortedTabs;
    const currentIdx = sorted.findIndex(t => t.id === this._activeTabId);
    let nextIdx = currentIdx;

    if (e.key === 'ArrowRight') {
      nextIdx = (currentIdx + 1) % sorted.length;
    } else if (e.key === 'ArrowLeft') {
      nextIdx = (currentIdx - 1 + sorted.length) % sorted.length;
    } else {
      return;
    }

    e.preventDefault();
    this._activeTabId = sorted[nextIdx].id;
    const tabs = this.shadowRoot?.querySelectorAll('[role="tab"]') as NodeListOf<HTMLElement>;
    tabs[nextIdx]?.focus();
  };

  override focus(): void {
    if (this._item) {
      const panel = this.shadowRoot?.querySelector('[role="tabpanel"]') as HTMLElement | undefined;
      panel?.focus();
    }
  }

  override render() {
    if (!this._item) {
      return html`<div class="empty" role="status">${this.emptyMessage}</div>`;
    }

    const sorted = this._sortedTabs;
    const activeTab = this._activeTab;
    const activeElement = activeTab ? this._getOrCreateTabElement(activeTab) : null;

    return html`
      <div class="tab-bar" role="tablist" @keydown=${this._handleTabKeyDown}>
        ${sorted.map(tab => {
          const isActive = tab.id === this._activeTabId;
          const badgeValue = tab.badge?.(this._item);
          return html`
            <button class="tab-button"
                    role="tab"
                    aria-selected="${isActive}"
                    aria-controls="panel-${tab.id}"
                    tabindex="${isActive ? 0 : -1}"
                    @click=${() => this._handleTabClick(tab.id)}>
              ${tab.label}
              ${badgeValue ? html`<span class="badge">${badgeValue}</span>` : nothing}
            </button>
          `;
        })}
      </div>
      <div class="tab-panel"
           role="tabpanel"
           id="panel-${this._activeTabId}"
           aria-labelledby="tab-${this._activeTabId}"
           tabindex="0">
        ${activeElement}
      </div>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'detail-pane': DetailPane;
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd components/detail-pane test`
Expected: ALL PASS

- [ ] **Step 5: Typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/detail-pane/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add detail-pane — generic tabbed detail container with item property contract (#37)"
```

---

### Task 5: Refactor `work-item-workbench` onto `split-workbench`

**Files:**
- Modify: `components/work-item-workbench/src/work-item-workbench.ts`
- Modify: `components/work-item-workbench/src/work-item-workbench.test.ts`
- Modify: `components/work-item-workbench/package.json` (add split-workbench dep)
- Modify: `components/work-item-workbench/tsconfig.json` (add reference)

**Interfaces:**
- Consumes: `<split-workbench>` from Task 2
- Produces: Same public API as before — `endpoint`, `identity`, `userSearchProvider`, `configure()`

- [ ] **Step 1: Add split-workbench dependency**

In `components/work-item-workbench/package.json`, add to dependencies:
```json
"@casehubio/blocks-ui-split-workbench": "workspace:*"
```

In `components/work-item-workbench/tsconfig.json`, add to references:
```json
{ "path": "../split-workbench" }
```

Run: `yarn install`

- [ ] **Step 2: Update existing tests to verify split-workbench is rendered internally**

Add to `work-item-workbench.test.ts`:

```typescript
it('renders split-workbench internally', async () => {
  const sw = el.shadowRoot!.querySelector('split-workbench');
  expect(sw).toBeTruthy();
  expect(sw!.getAttribute('selection-topic')).toBe('work-item');
});

it('passes endpoint to inbox and detail via split-workbench slots', async () => {
  const inbox = el.shadowRoot!.querySelector('work-item-inbox') as any;
  const detail = el.shadowRoot!.querySelector('work-item-detail') as any;
  expect(inbox?.endpoint).toBe(el.endpoint);
  expect(detail?.endpoint).toBe(el.endpoint);
});
```

Run: `yarn --cwd components/work-item-workbench test`
Expected: FAIL — split-workbench not rendered yet

- [ ] **Step 3: Refactor work-item-workbench to use split-workbench**

Replace the entire `work-item-workbench.ts` implementation. The component becomes a thin wrapper:

```typescript
import { LitElement, html, css, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import {
  type WorkIdentity,
  type UserSearchProvider,
  KeyboardShortcutMixin,
} from '@casehubio/blocks-ui-core';
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-work-item-inbox';
import '@casehubio/blocks-ui-work-item-detail';

@customElement('work-item-workbench')
export class WorkItemWorkbench extends KeyboardShortcutMixin(LitElement) {
  @property({ type: Object }) identity!: WorkIdentity;
  @property({ type: String }) endpoint = '';
  @property({ type: Object }) userSearchProvider: UserSearchProvider | null = null;

  @state() private _selectedWorkItemId = '';
  @state() private _showShortcutOverlay = false;

  static override styles = css`
    :host {
      display: block;
      height: 100%;
      width: 100%;
      font-family: var(--pages-font-family, system-ui);
      overflow: hidden;
      container-type: inline-size;
    }

    split-workbench {
      height: 100%;
    }

    .keyboard-hints {
      display: flex;
      gap: var(--pages-space-4, 16px);
      padding: var(--pages-space-2, 8px) var(--pages-space-4, 16px);
      background: var(--pages-neutral-2, #f5f5f5);
      border-top: 1px solid var(--pages-neutral-4, #d4d4d4);
      font-size: var(--pages-font-size-sm, 12px);
      color: var(--pages-neutral-7, #525252);
      overflow-x: auto;
    }

    .hint {
      display: flex;
      align-items: center;
      gap: var(--pages-space-1, 4px);
      white-space: nowrap;
    }

    .key {
      display: inline-block;
      padding: 2px 6px;
      background: var(--pages-neutral-3, #e5e5e5);
      border: 1px solid var(--pages-neutral-5, #a3a3a3);
      border-radius: 3px;
      font-family: monospace;
      font-size: 11px;
      font-weight: 600;
    }

    .shortcut-overlay {
      position: fixed;
      top: 0; left: 0; right: 0; bottom: 0;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1000;
    }

    .shortcut-panel {
      background: var(--pages-neutral-1, #fafafa);
      border: 1px solid var(--pages-neutral-4, #d4d4d4);
      border-radius: 8px;
      padding: var(--pages-space-4, 16px);
      max-width: 600px;
      max-height: 80vh;
      overflow-y: auto;
    }

    .shortcut-title {
      font-size: var(--pages-font-size-lg, 16px);
      font-weight: 600;
      margin-bottom: var(--pages-space-3, 12px);
      color: var(--pages-neutral-11, #0a0a0a);
    }

    .shortcut-list {
      display: flex;
      flex-direction: column;
      gap: var(--pages-space-2, 8px);
    }

    .shortcut-item {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: var(--pages-space-2, 8px);
      background: var(--pages-neutral-2, #f5f5f5);
      border-radius: 4px;
    }

    .shortcut-desc {
      color: var(--pages-neutral-9, #262626);
    }

    @container (max-width: 1024px) {
      .keyboard-hints { display: none; }
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    this._registerKeyboardShortcuts();
  }

  configure(props: {
    endpoint?: string;
    identity?: WorkIdentity;
    userSearchProvider?: UserSearchProvider;
  }): void {
    if (props.endpoint !== undefined) this.endpoint = props.endpoint;
    if (props.identity !== undefined) this.identity = props.identity;
    if (props.userSearchProvider !== undefined) this.userSearchProvider = props.userSearchProvider;
    this.requestUpdate();
  }

  private _registerKeyboardShortcuts(): void {
    this.registerShortcut('?', () => { this._showShortcutOverlay = !this._showShortcutOverlay; }, {
      description: 'Show keyboard shortcuts',
    });
    this.registerShortcut('Escape', () => {
      if (this._showShortcutOverlay) this._showShortcutOverlay = false;
    }, { description: 'Close overlay' });
  }

  override render(): TemplateResult {
    return html`
      <split-workbench selection-topic="work-item">
        <work-item-inbox slot="list"
          .endpoint=${this.endpoint}
          .identity=${this.identity}
        ></work-item-inbox>
        <work-item-detail slot="detail"
          .endpoint=${this.endpoint}
          .identity=${this.identity}
          .userSearchProvider=${this.userSearchProvider}
        ></work-item-detail>
      </split-workbench>

      ${this._renderKeyboardHints()}
      ${this._showShortcutOverlay ? this._renderShortcutOverlay() : ''}
    `;
  }

  private _renderKeyboardHints(): TemplateResult {
    return html`
      <div class="keyboard-hints">
        <div class="hint"><span class="key">↑</span><span class="key">↓</span> Navigate</div>
        <div class="hint"><span class="key">Enter</span> Select</div>
        <div class="hint"><span class="key">Esc</span> Back</div>
        <div class="hint"><span class="key">C</span> Claim</div>
        <div class="hint"><span class="key">S</span> Start</div>
        <div class="hint"><span class="key">E</span> Complete</div>
        <div class="hint"><span class="key">?</span> Shortcuts</div>
      </div>
    `;
  }

  private _renderShortcutOverlay(): TemplateResult {
    const shortcuts = [
      { key: '↑ / ↓', desc: 'Navigate inbox items' },
      { key: 'Enter', desc: 'Open selected item' },
      { key: 'Escape', desc: 'Return to inbox' },
      { key: 'C', desc: 'Claim focused item' },
      { key: 'S', desc: 'Start work on item' },
      { key: 'E', desc: 'Complete item' },
      { key: 'R', desc: 'Reject item' },
      { key: 'Tab', desc: 'Switch between panels' },
      { key: '?', desc: 'Toggle this overlay' },
    ];
    return html`
      <div class="shortcut-overlay" @click=${() => { this._showShortcutOverlay = false; }}>
        <div class="shortcut-panel" @click=${(e: Event) => e.stopPropagation()}>
          <div class="shortcut-title">Keyboard Shortcuts</div>
          <div class="shortcut-list">
            ${shortcuts.map(s => html`
              <div class="shortcut-item">
                <span class="shortcut-desc">${s.desc}</span>
                <span class="key">${s.key}</span>
              </div>
            `)}
          </div>
        </div>
      </div>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'work-item-workbench': WorkItemWorkbench;
  }
}
```

- [ ] **Step 4: Run tests**

Run: `yarn --cwd components/work-item-workbench test`
Expected: ALL PASS (existing tests + new tests)

- [ ] **Step 5: Run full test suite**

Run: `yarn test`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/work-item-workbench/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor: work-item-workbench uses split-workbench internally — no public API change (#37)"
```

---

### Task 6: Migrate AML to Consume blocks-ui Components

**Files:**
- Modify: `app/src/main/webui/package.json` (in AML repo)
- Modify: `app/src/main/webui/src/aml-app.ts`
- Modify: `app/src/main/webui/src/panels/aml-investigation-overview.ts`
- Modify: `app/src/main/webui/src/panels/aml-findings-panel.ts`
- Modify: `app/src/main/webui/src/panels/aml-routing-panel.ts`
- Modify: `app/src/main/webui/src/panels/aml-compliance-panel.ts`
- Modify: `app/src/main/webui/src/panels/aml-audit-trail.ts`
- Delete: `app/src/main/webui/src/components/case-workbench/` (entire directory)

**Interfaces:**
- Consumes: `<split-workbench>`, `<list-pane>`, `<detail-pane>`, `TabDefinition` from blocks-ui

- [ ] **Step 1: Add blocks-ui dependencies to AML**

In AML's `app/src/main/webui/package.json`, add:
```json
"@casehubio/blocks-ui-split-workbench": "workspace:*",
"@casehubio/blocks-ui-list-pane": "workspace:*",
"@casehubio/blocks-ui-detail-pane": "workspace:*"
```

Remove the local case-workbench reference if present.

Run: `yarn install` (in AML webui directory)

- [ ] **Step 2: Update each AML tab panel — rename caseId/caseData to item**

For each of the five panels, replace `caseId` and `caseData` properties with a single `item` property and add getters:

Example pattern (apply to all five panels):
```typescript
// Before:
@property({ type: String }) caseId = '';
@property({ attribute: false }) caseData: any = null;

// After:
@property({ attribute: false }) item: any = null;

get caseId(): string { return this.item?.caseId ?? ''; }
get caseData(): any { return this.item; }
```

This preserves all existing template references to `this.caseId` and `this.caseData` — they become derived getters instead of independent properties.

Apply to:
- `panels/aml-investigation-overview.ts`
- `panels/aml-findings-panel.ts`
- `panels/aml-routing-panel.ts`
- `panels/aml-compliance-panel.ts`
- `panels/aml-audit-trail.ts`

- [ ] **Step 3: Update aml-app.ts to use blocks-ui components**

Replace the local case-workbench imports and usage:

```typescript
// Remove:
import './components/case-workbench/src/case-workbench.js';
import './components/case-workbench/src/tab-registry.js';
import { registerCaseTab } from './components/case-workbench/src/tab-registry.js';

// Add:
import '@casehubio/blocks-ui-split-workbench';
import '@casehubio/blocks-ui-list-pane';
import '@casehubio/blocks-ui-detail-pane';
import type { TabDefinition } from '@casehubio/blocks-ui-detail-pane';

// Remove registerCaseTab() calls, replace with tabs array:
const investigationTabs: TabDefinition[] = [
  { id: 'overview', label: 'Overview', tagName: 'aml-investigation-overview', order: 0 },
  { id: 'findings', label: 'Findings', tagName: 'aml-findings-panel', order: 10 },
  { id: 'routing', label: 'Routing & Trust', tagName: 'aml-routing-panel', order: 20 },
  { id: 'compliance', label: 'Compliance', tagName: 'aml-compliance-panel', order: 25 },
  { id: 'audit', label: 'Audit', tagName: 'aml-audit-trail', order: 30 },
];

// In render(), replace <case-workbench> with:
html`
  <split-workbench selection-topic="case" title="AML Investigations">
    <list-pane slot="list"
      selection-topic="case"
      endpoint="/api/investigations"
      .columns=${this._investigationColumns}
      .getRowKey=${(row: any) => row.caseId}
      .getRowClass=${(row: any) => this._statusClass(row.status)}>
    </list-pane>
    <detail-pane slot="detail"
      selection-topic="case"
      .tabs=${investigationTabs}
      empty-message="Select an investigation to view details">
    </detail-pane>
  </split-workbench>
`
```

- [ ] **Step 4: Delete AML's local case-workbench directory**

Delete the entire `app/src/main/webui/src/components/case-workbench/` directory:
- `case-workbench.ts`
- `case-list-pane.ts`
- `case-detail-pane.ts`
- `tab-registry.ts`

- [ ] **Step 5: Build and typecheck AML**

Run AML's build to verify:
```bash
yarn --cwd /Users/mdproctor/claude/casehub/aml/app/src/main/webui typecheck
```
Expected: No errors

- [ ] **Step 6: Commit AML changes**

```bash
git -C /Users/mdproctor/claude/casehub/aml add app/src/main/webui/
git -C /Users/mdproctor/claude/casehub/aml commit -m "refactor: migrate to blocks-ui generic workbench components — delete local case-workbench (#91, casehubio/blocks-ui#37)"
```

---

### Task 7: Issue Updates and Final Verification

**Files:** None (GitHub issue updates only)

- [ ] **Step 1: Run full blocks-ui test suite**

```bash
yarn --cwd /Users/mdproctor/claude/casehub/blocks-ui test
```
Expected: ALL PASS

- [ ] **Step 2: Run full blocks-ui typecheck**

```bash
yarn --cwd /Users/mdproctor/claude/casehub/blocks-ui typecheck
```
Expected: No errors

- [ ] **Step 3: Update GitHub issues**

Update blocks-ui #37 — link spec, mark components built and AML migrated
Update blocks-ui #35 — update AML row in migration tracking table
Create blocks-ui issue — refactor work-item-workbench onto split-workbench (done in Task 5, close it)
Create blocks-ui issues for prerequisites (event topic migration, DataEndpointMixin guard — done in Task 1, close them)

- [ ] **Step 4: Commit any remaining changes**

Verify both repos are clean:
```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui status --short
git -C /Users/mdproctor/claude/casehub/aml status --short
```
