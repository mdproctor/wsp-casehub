# Phase 4 — Structural Editing + Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — Epic: Visual Diagram Editor — Domain Layer
**Issue group:** #103

**Goal:** Add/remove/restructure case definition nodes, save/load via GitHub persistence, with palette, toolbar, delete with dependency checks, and binding target type switching.

**Architecture:** YAML-first mutations — all structural edits go through CST-preserving YAML operations in `yaml-editor.ts`, then re-derive the graph via `toGraph()`. Persistence via `PersistenceBackend` SPI (graph-core) with a new `GitHubBackend`. New Lit sub-components (palette, toolbar) compose into `casehub-diagram`'s layout.

**Tech Stack:** TypeScript, Lit 3.x, `yaml` npm (CST), vitest, graph-core (pages), graph-renderer (pages)

## Global Constraints

- YAML is the source of truth — never persist or mutate GraphModel directly
- All YAML mutations use `parseDocument()` for CST preservation
- Structural edits trigger full re-layout via `computeElkLayout()` (unlike property edits)
- Undo/redo always uses `_fullRender()` (not `_updateWithoutLayout()`)
- Sub-components (palette, toolbar) use Shadow DOM; casehub-diagram skips it
- Events use `composed: true, bubbles: true` to cross Shadow DOM boundaries
- IntelliJ MCP for all code navigation and editing — no bash grep/sed on source files

---

### Task 1: YAML Structural Operations — addElement, removeElement, switchBindingTarget

**Files:**
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.ts`
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`

**Interfaces:**
- Consumes: `parseDocument` from `yaml` npm (already used by `applyPropertyEdit`)
- Produces:
  - `addElement(yaml: string, elementType: 'binding' | 'worker' | 'milestone' | 'goal', defaults?: Record<string, unknown>): string`
  - `removeElement(yaml: string, nodePath: readonly (string | number)[]): string`
  - `switchBindingTarget(yaml: string, bindingPath: readonly (string | number)[], targetType: 'capability' | 'subCase' | 'humanTask'): string`

- [ ] **Step 1: Write failing tests for addElement**

```typescript
describe('addElement', () => {
  it('adds a binding with default name and capability', () => {
    const result = addElement(SAMPLE_YAML, 'binding');
    const parsed = parseYaml(result) as CaseDefinition;
    const bindings = parsed.spec.bindings!;
    const added = bindings[bindings.length - 1];
    expect(added.name).toBe('binding-1');
    expect(added.capability).toBe('');
  });

  it('adds a worker with default name and empty capabilities', () => {
    const result = addElement(SAMPLE_YAML, 'worker');
    const parsed = parseYaml(result) as CaseDefinition;
    const workers = parsed.spec.workers!;
    const added = workers[workers.length - 1];
    expect(added.name).toBe('worker-1');
    expect(added.capabilities).toEqual([]);
  });

  it('adds a milestone with default name', () => {
    const result = addElement(SAMPLE_YAML, 'milestone');
    const parsed = parseYaml(result) as CaseDefinition;
    const milestones = parsed.spec.milestones!;
    const added = milestones[milestones.length - 1];
    expect(added.name).toBe('milestone-1');
  });

  it('adds a goal with default name and kind', () => {
    const result = addElement(SAMPLE_YAML, 'goal');
    const parsed = parseYaml(result) as CaseDefinition;
    expect(parsed.spec.goals![0].name).toBe('goal-1');
    expect(parsed.spec.goals![0].kind).toBe('success');
  });

  it('generates unique names when duplicates exist', () => {
    let yaml = addElement(SAMPLE_YAML, 'milestone');
    yaml = addElement(yaml, 'milestone');
    const parsed = parseYaml(yaml) as CaseDefinition;
    const names = parsed.spec.milestones!.map(m => m.name);
    expect(names).toContain('milestone-1');
    expect(names).toContain('milestone-2');
  });

  it('merges caller-provided defaults over generated defaults', () => {
    const result = addElement(SAMPLE_YAML, 'worker', { name: 'custom', description: 'A worker' });
    const parsed = parseYaml(result) as CaseDefinition;
    const added = parsed.spec.workers![parsed.spec.workers!.length - 1];
    expect(added.name).toBe('custom');
    expect(added.description).toBe('A worker');
  });

  it('preserves CST formatting of untouched sections', () => {
    const result = addElement(SAMPLE_YAML, 'goal');
    expect(result).toContain('dsl: "1.0.0"');
    expect(result).toContain("when: '.doc != null'");
  });

  it('creates the array if it does not exist', () => {
    const yamlNoGoals = `dsl: "1.0.0"\nnamespace: test\nname: sample\nversion: "1.0.0"\nspec:\n  bindings: []\n`;
    const result = addElement(yamlNoGoals, 'goal');
    const parsed = parseYaml(result) as CaseDefinition;
    expect(parsed.spec.goals).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: FAIL — `addElement` not defined

- [ ] **Step 3: Implement addElement**

Add to `yaml-editor.ts`:

```typescript
const ELEMENT_PATHS: Record<string, string> = {
  binding: 'bindings',
  worker: 'workers',
  milestone: 'milestones',
  goal: 'goals',
};

const ELEMENT_DEFAULTS: Record<string, (n: number) => Record<string, unknown>> = {
  binding: (n) => ({ name: `binding-${n}`, capability: '' }),
  worker: (n) => ({ name: `worker-${n}`, capabilities: [] }),
  milestone: (n) => ({ name: `milestone-${n}` }),
  goal: (n) => ({ name: `goal-${n}`, kind: 'success' }),
};

export function addElement(
  yaml: string,
  elementType: 'binding' | 'worker' | 'milestone' | 'goal',
  defaults?: Record<string, unknown>,
): string {
  const doc = parseDocument(yaml);
  const arrayKey = ELEMENT_PATHS[elementType];
  const specPath = ['spec', arrayKey];

  let seq = doc.getIn(specPath);
  if (!seq) {
    doc.setIn(specPath, []);
    seq = doc.getIn(specPath);
  }

  const existing = (doc.toJS() as { spec: Record<string, unknown[]> }).spec[arrayKey] ?? [];
  const existingNames = new Set(existing.map((e: Record<string, unknown>) => String(e.name ?? '')));

  let n = 1;
  while (existingNames.has(`${elementType}-${n}`)) n++;

  const generated = ELEMENT_DEFAULTS[elementType](n);
  const merged = defaults ? { ...generated, ...defaults } : generated;

  doc.addIn(specPath, merged);
  return doc.toString();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: All `addElement` tests PASS

- [ ] **Step 5: Write failing tests for removeElement**

```typescript
describe('removeElement', () => {
  it('removes a binding by path', () => {
    const result = removeElement(SAMPLE_YAML, ['spec', 'bindings', 0]);
    const parsed = parseYaml(result) as CaseDefinition;
    expect(parsed.spec.bindings).toHaveLength(0);
  });

  it('removes a worker by path', () => {
    const result = removeElement(SAMPLE_YAML, ['spec', 'workers', 0]);
    const parsed = parseYaml(result) as CaseDefinition;
    expect(parsed.spec.workers).toHaveLength(0);
  });

  it('preserves other elements in the same array', () => {
    let yaml = addElement(SAMPLE_YAML, 'binding');
    const result = removeElement(yaml, ['spec', 'bindings', 0]);
    const parsed = parseYaml(result) as CaseDefinition;
    expect(parsed.spec.bindings).toHaveLength(1);
    expect(parsed.spec.bindings![0].name).toBe('binding-1');
  });

  it('preserves CST formatting of untouched sections', () => {
    const result = removeElement(SAMPLE_YAML, ['spec', 'milestones', 0]);
    expect(result).toContain('dsl: "1.0.0"');
    expect(result).toContain('name: scan');
  });
});
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: FAIL — `removeElement` not defined

- [ ] **Step 7: Implement removeElement**

```typescript
export function removeElement(
  yaml: string,
  nodePath: readonly (string | number)[],
): string {
  const doc = parseDocument(yaml);
  doc.deleteIn([...nodePath]);
  return doc.toString();
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: All `removeElement` tests PASS

- [ ] **Step 9: Write failing tests for switchBindingTarget**

```typescript
describe('switchBindingTarget', () => {
  it('switches from capability to subCase', () => {
    const result = switchBindingTarget(SAMPLE_YAML, ['spec', 'bindings', 0], 'subCase');
    const parsed = parseYaml(result) as CaseDefinition;
    const binding = parsed.spec.bindings![0];
    expect(binding.capability).toBeUndefined();
    expect(binding.subCase).toEqual({ namespace: '', name: '' });
  });

  it('switches from capability to humanTask', () => {
    const result = switchBindingTarget(SAMPLE_YAML, ['spec', 'bindings', 0], 'humanTask');
    const parsed = parseYaml(result) as CaseDefinition;
    const binding = parsed.spec.bindings![0];
    expect(binding.capability).toBeUndefined();
    expect(binding.humanTask).toEqual({ title: '' });
  });

  it('switches from subCase back to capability', () => {
    let yaml = switchBindingTarget(SAMPLE_YAML, ['spec', 'bindings', 0], 'subCase');
    yaml = switchBindingTarget(yaml, ['spec', 'bindings', 0], 'capability');
    const parsed = parseYaml(yaml) as CaseDefinition;
    const binding = parsed.spec.bindings![0];
    expect(binding.subCase).toBeUndefined();
    expect(binding.capability).toBe('');
  });

  it('preserves non-target properties', () => {
    const result = switchBindingTarget(SAMPLE_YAML, ['spec', 'bindings', 0], 'subCase');
    const parsed = parseYaml(result) as CaseDefinition;
    const binding = parsed.spec.bindings![0];
    expect(binding.name).toBe('scan');
    expect(binding.when).toBe('.doc != null');
  });

  it('round-trips through toGraph producing correct topology', () => {
    const result = switchBindingTarget(SAMPLE_YAML, ['spec', 'bindings', 0], 'subCase');
    const { model } = toGraph(result);
    const subcaseNodes = model.nodes.filter(n => n.type === 'subcase');
    expect(subcaseNodes.length).toBeGreaterThan(0);
    const capEdges = model.edges.filter(e => e.type === 'capability-dispatch' && e.source === 'binding:scan');
    expect(capEdges).toHaveLength(0);
  });
});
```

- [ ] **Step 10: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: FAIL — `switchBindingTarget` not defined

- [ ] **Step 11: Implement switchBindingTarget**

```typescript
const TARGET_DEFAULTS: Record<string, unknown> = {
  capability: '',
  subCase: { namespace: '', name: '' },
  humanTask: { title: '' },
};

export function switchBindingTarget(
  yaml: string,
  bindingPath: readonly (string | number)[],
  targetType: 'capability' | 'subCase' | 'humanTask',
): string {
  const doc = parseDocument(yaml);
  for (const key of ['capability', 'subCase', 'humanTask']) {
    doc.deleteIn([...bindingPath, key]);
  }
  doc.setIn([...bindingPath, targetType], TARGET_DEFAULTS[targetType]);
  return doc.toString();
}
```

- [ ] **Step 12: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: All tests PASS

- [ ] **Step 13: Export new functions from index.ts**

Add to `packages/graph-stencil-case/src/index.ts`:

```typescript
export { addElement, removeElement, switchBindingTarget } from './adapter/yaml-editor.js';
```

- [ ] **Step 14: Run full test suite and typecheck**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose && yarn workspace @casehubio/graph-stencil-case run typecheck`
Expected: All PASS, no type errors

- [ ] **Step 15: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/adapter/yaml-editor.ts packages/graph-stencil-case/src/adapter/yaml-editor.test.ts packages/graph-stencil-case/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): addElement, removeElement, switchBindingTarget — YAML structural operations"
```

---

### Task 2: GitHubBackend — Persistence Implementation

**Files:**
- Create: `packages/graph-stencil-case/src/persistence/github-backend.ts`
- Create: `packages/graph-stencil-case/src/persistence/github-backend.test.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`

**Interfaces:**
- Consumes: `PersistenceBackend`, `ReadResult`, `WriteResult` from `@casehubio/graph-core`
- Produces:
  - `GitHubBackendConfig { token: string; owner: string; repo: string; branch?: string; commitMessage?: string }`
  - `GitHubBackend implements PersistenceBackend`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { GitHubBackend } from './github-backend.js';

const mockFetch = vi.fn();
vi.stubGlobal('fetch', mockFetch);

function jsonResponse(status: number, body: unknown): Response {
  return { ok: status >= 200 && status < 300, status, json: () => Promise.resolve(body) } as Response;
}

const config = { token: 'ghp_test', owner: 'org', repo: 'repo', branch: 'main' };

describe('GitHubBackend', () => {
  let backend: GitHubBackend;

  beforeEach(() => {
    mockFetch.mockReset();
    backend = new GitHubBackend(config);
  });

  describe('read', () => {
    it('returns ok with decoded content and sha as version', async () => {
      const content = btoa('dsl: "1.0.0"\nname: test\n');
      mockFetch.mockResolvedValueOnce(jsonResponse(200, { content, sha: 'abc123' }));
      const result = await backend.read('cases/test.yaml');
      expect(result).toEqual({ status: 'ok', yaml: 'dsl: "1.0.0"\nname: test\n', version: 'abc123' });
    });

    it('returns not_found on 404', async () => {
      mockFetch.mockResolvedValueOnce(jsonResponse(404, { message: 'Not Found' }));
      const result = await backend.read('cases/missing.yaml');
      expect(result).toEqual({ status: 'not_found', uri: 'cases/missing.yaml' });
    });
  });

  describe('write', () => {
    it('returns ok with new sha on success', async () => {
      mockFetch.mockResolvedValueOnce(jsonResponse(200, { content: { sha: 'def456' } }));
      const result = await backend.write('cases/test.yaml', 'dsl: "1.0.0"\n', 'abc123');
      expect(result).toEqual({ status: 'ok', version: 'def456' });
      const body = JSON.parse(mockFetch.mock.calls[0][1].body);
      expect(body.sha).toBe('abc123');
    });

    it('returns conflict on 409 after fetching current sha', async () => {
      mockFetch.mockResolvedValueOnce(jsonResponse(409, { message: 'conflict' }));
      mockFetch.mockResolvedValueOnce(jsonResponse(200, { sha: 'current789' }));
      const result = await backend.write('cases/test.yaml', 'dsl: "1.0.0"\n', 'stale');
      expect(result).toEqual({ status: 'conflict', currentVersion: 'current789' });
    });

    it('omits sha for new file creation when expectedVersion is empty', async () => {
      mockFetch.mockResolvedValueOnce(jsonResponse(201, { content: { sha: 'new123' } }));
      const result = await backend.write('cases/new.yaml', 'dsl: "1.0.0"\n', '');
      expect(result).toEqual({ status: 'ok', version: 'new123' });
      const body = JSON.parse(mockFetch.mock.calls[0][1].body);
      expect(body.sha).toBeUndefined();
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: FAIL — module not found

- [ ] **Step 3: Implement GitHubBackend**

Create `packages/graph-stencil-case/src/persistence/github-backend.ts`:

```typescript
import type { PersistenceBackend, ReadResult, WriteResult } from '@casehubio/graph-core';

export interface GitHubBackendConfig {
  readonly token: string;
  readonly owner: string;
  readonly repo: string;
  readonly branch?: string;
  readonly commitMessage?: string;
}

export class GitHubBackend implements PersistenceBackend {
  private readonly token: string;
  private readonly owner: string;
  private readonly repo: string;
  private readonly branch: string;
  private readonly commitMessage: string;

  constructor(config: GitHubBackendConfig) {
    this.token = config.token;
    this.owner = config.owner;
    this.repo = config.repo;
    this.branch = config.branch ?? 'main';
    this.commitMessage = config.commitMessage ?? 'Update case definition';
  }

  async read(uri: string): Promise<ReadResult> {
    const url = `https://api.github.com/repos/${this.owner}/${this.repo}/contents/${uri}?ref=${this.branch}`;
    const res = await fetch(url, {
      headers: { Authorization: `Bearer ${this.token}`, Accept: 'application/vnd.github.v3+json' },
    });

    if (res.status === 404) {
      return { status: 'not_found', uri };
    }

    const data = await res.json() as { content: string; sha: string };
    const yaml = atob(data.content.replace(/\n/g, ''));
    return { status: 'ok', yaml, version: data.sha };
  }

  async write(uri: string, yaml: string, expectedVersion: string): Promise<WriteResult> {
    const url = `https://api.github.com/repos/${this.owner}/${this.repo}/contents/${uri}`;
    const body: Record<string, unknown> = {
      message: this.commitMessage,
      content: btoa(yaml),
      branch: this.branch,
    };
    if (expectedVersion) {
      body.sha = expectedVersion;
    }

    const res = await fetch(url, {
      method: 'PUT',
      headers: { Authorization: `Bearer ${this.token}`, Accept: 'application/vnd.github.v3+json', 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });

    if (res.status === 409) {
      const current = await fetch(`${url}?ref=${this.branch}`, {
        headers: { Authorization: `Bearer ${this.token}`, Accept: 'application/vnd.github.v3+json' },
      });
      const currentData = await current.json() as { sha: string };
      return { status: 'conflict', currentVersion: currentData.sha };
    }

    const data = await res.json() as { content: { sha: string } };
    return { status: 'ok', version: data.content.sha };
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose`
Expected: All GitHubBackend tests PASS

- [ ] **Step 5: Export from index.ts**

Add to `packages/graph-stencil-case/src/index.ts`:

```typescript
export { GitHubBackend } from './persistence/github-backend.js';
export type { GitHubBackendConfig } from './persistence/github-backend.js';
```

- [ ] **Step 6: Typecheck**

Run: `yarn workspace @casehubio/graph-stencil-case run typecheck`
Expected: No errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/persistence/github-backend.ts packages/graph-stencil-case/src/persistence/github-backend.test.ts packages/graph-stencil-case/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): GitHubBackend — persistence via GitHub Contents API"
```

---

### Task 3: Palette Component

**Files:**
- Create: `components/casehub-diagram/src/casehub-diagram-palette.ts`
- Create: `components/casehub-diagram/src/casehub-diagram-palette.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `<casehub-diagram-palette>` custom element
  - Property: `disabled: boolean`
  - Event: `palette-add` with `{ elementType: 'binding' | 'worker' | 'milestone' | 'goal' }` (composed, bubbles)

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi } from 'vitest';
import { fixture, html } from '@open-wc/testing-helpers';
import './casehub-diagram-palette.js';

describe('casehub-diagram-palette', () => {
  it('renders four palette items', async () => {
    const el = await fixture(html`<casehub-diagram-palette></casehub-diagram-palette>`);
    const buttons = el.shadowRoot!.querySelectorAll('button');
    expect(buttons.length).toBe(4);
  });

  it('emits palette-add with correct elementType on click', async () => {
    const el = await fixture(html`<casehub-diagram-palette></casehub-diagram-palette>`);
    const handler = vi.fn();
    el.addEventListener('palette-add', handler);
    const buttons = el.shadowRoot!.querySelectorAll('button');
    buttons[0].click();
    expect(handler).toHaveBeenCalledTimes(1);
    expect(handler.mock.calls[0][0].detail.elementType).toBe('binding');
  });

  it('disables all buttons when disabled property is set', async () => {
    const el = await fixture(html`<casehub-diagram-palette ?disabled=${true}></casehub-diagram-palette>`);
    const buttons = el.shadowRoot!.querySelectorAll('button');
    for (const btn of buttons) {
      expect(btn.disabled).toBe(true);
    }
  });

  it('emits correct types for each button', async () => {
    const el = await fixture(html`<casehub-diagram-palette></casehub-diagram-palette>`);
    const handler = vi.fn();
    el.addEventListener('palette-add', handler);
    const buttons = el.shadowRoot!.querySelectorAll('button');
    const expected = ['binding', 'worker', 'milestone', 'goal'];
    for (let i = 0; i < buttons.length; i++) {
      buttons[i].click();
      expect(handler.mock.calls[i][0].detail.elementType).toBe(expected[i]);
    }
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace casehub-diagram test -- --reporter verbose`
Expected: FAIL — module not found

- [ ] **Step 3: Implement casehub-diagram-palette**

Create `components/casehub-diagram/src/casehub-diagram-palette.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

type ElementType = 'binding' | 'worker' | 'milestone' | 'goal';

const ITEMS: { type: ElementType; label: string; shape: string }[] = [
  { type: 'binding', label: 'Binding', shape: '╭─╮' },
  { type: 'worker', label: 'Worker', shape: '┌─┐' },
  { type: 'milestone', label: 'Milestone', shape: '◇' },
  { type: 'goal', label: 'Goal', shape: '⬡' },
];

@customElement('casehub-diagram-palette')
export class CasehubDiagramPalette extends LitElement {
  @property({ type: Boolean }) disabled = false;

  static override styles = css`
    :host { display: flex; flex-direction: column; gap: 4px; padding: 6px; width: 56px; box-sizing: border-box; }
    button {
      display: flex; flex-direction: column; align-items: center; gap: 2px;
      border: 1px solid var(--pages-border-color, #ddd); border-radius: 6px;
      background: var(--pages-surface-color, #fff); cursor: pointer;
      padding: 6px 2px; font-size: 9px;
      color: var(--pages-text-secondary, #666);
      font-family: var(--pages-font-family, system-ui, sans-serif);
    }
    button:hover:not(:disabled) { background: var(--pages-surface-raised, #f5f5f5); }
    button:disabled { opacity: 0.4; cursor: default; }
    .shape { font-size: 16px; line-height: 1; }
  `;

  override render() {
    return html`${ITEMS.map(item => html`
      <button ?disabled=${this.disabled} @click=${() => this._emit(item.type)}>
        <span class="shape">${item.shape}</span>
        ${item.label}
      </button>
    `)}`;
  }

  private _emit(elementType: ElementType): void {
    this.dispatchEvent(new CustomEvent('palette-add', {
      bubbles: true, composed: true, detail: { elementType },
    }));
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace casehub-diagram test -- --reporter verbose`
Expected: All palette tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/casehub-diagram/src/casehub-diagram-palette.ts components/casehub-diagram/src/casehub-diagram-palette.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): casehub-diagram-palette — click-to-add palette component"
```

---

### Task 4: Toolbar Component

**Files:**
- Create: `components/casehub-diagram/src/casehub-diagram-toolbar.ts`
- Create: `components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `<casehub-diagram-toolbar>` custom element
  - Properties: `dirty: boolean`, `saving: boolean`, `hasBackend: boolean`
  - Event: `toolbar-save` (composed, bubbles)

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect, vi } from 'vitest';
import { fixture, html } from '@open-wc/testing-helpers';
import './casehub-diagram-toolbar.js';

describe('casehub-diagram-toolbar', () => {
  it('shows save button when hasBackend is true', async () => {
    const el = await fixture(html`<casehub-diagram-toolbar ?hasBackend=${true}></casehub-diagram-toolbar>`);
    const btn = el.shadowRoot!.querySelector('button');
    expect(btn).not.toBeNull();
  });

  it('hides save button when hasBackend is false', async () => {
    const el = await fixture(html`<casehub-diagram-toolbar ?hasBackend=${false}></casehub-diagram-toolbar>`);
    const btn = el.shadowRoot!.querySelector('button');
    expect(btn).toBeNull();
  });

  it('emits toolbar-save on save button click', async () => {
    const el = await fixture(html`<casehub-diagram-toolbar ?hasBackend=${true} ?dirty=${true}></casehub-diagram-toolbar>`);
    const handler = vi.fn();
    el.addEventListener('toolbar-save', handler);
    el.shadowRoot!.querySelector('button')!.click();
    expect(handler).toHaveBeenCalledTimes(1);
  });

  it('disables save button when not dirty', async () => {
    const el = await fixture(html`<casehub-diagram-toolbar ?hasBackend=${true} ?dirty=${false}></casehub-diagram-toolbar>`);
    const btn = el.shadowRoot!.querySelector('button') as HTMLButtonElement;
    expect(btn.disabled).toBe(true);
  });

  it('shows dirty indicator when dirty is true', async () => {
    const el = await fixture(html`<casehub-diagram-toolbar ?hasBackend=${true} ?dirty=${true}></casehub-diagram-toolbar>`);
    const dot = el.shadowRoot!.querySelector('.dirty-dot');
    expect(dot).not.toBeNull();
  });

  it('disables save button while saving', async () => {
    const el = await fixture(html`<casehub-diagram-toolbar ?hasBackend=${true} ?dirty=${true} ?saving=${true}></casehub-diagram-toolbar>`);
    const btn = el.shadowRoot!.querySelector('button') as HTMLButtonElement;
    expect(btn.disabled).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace casehub-diagram test -- --reporter verbose`
Expected: FAIL — module not found

- [ ] **Step 3: Implement casehub-diagram-toolbar**

Create `components/casehub-diagram/src/casehub-diagram-toolbar.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('casehub-diagram-toolbar')
export class CasehubDiagramToolbar extends LitElement {
  @property({ type: Boolean }) dirty = false;
  @property({ type: Boolean }) saving = false;
  @property({ type: Boolean }) hasBackend = false;

  static override styles = css`
    :host { display: flex; align-items: center; gap: 8px; padding: 4px 12px; border-bottom: 1px solid var(--pages-border-color, #ddd); height: 32px; box-sizing: border-box; font-family: var(--pages-font-family, system-ui, sans-serif); }
    button {
      border: 1px solid var(--pages-border-color, #ccc); border-radius: 4px;
      background: var(--pages-surface-color, #fff); cursor: pointer;
      padding: 2px 10px; font-size: 12px; color: var(--pages-text-color, #333);
      display: flex; align-items: center; gap: 4px;
    }
    button:hover:not(:disabled) { background: var(--pages-surface-raised, #f5f5f5); }
    button:disabled { opacity: 0.4; cursor: default; }
    .dirty-dot { width: 6px; height: 6px; border-radius: 50%; background: var(--pages-warning-color, #f59e0b); }
  `;

  override render() {
    if (!this.hasBackend) return nothing;

    return html`
      <button ?disabled=${!this.dirty || this.saving} @click=${this._save}>
        ${this.saving ? 'Saving…' : 'Save'}
      </button>
      ${this.dirty ? html`<span class="dirty-dot"></span>` : nothing}
    `;
  }

  private _save(): void {
    this.dispatchEvent(new CustomEvent('toolbar-save', { bubbles: true, composed: true }));
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace casehub-diagram test -- --reporter verbose`
Expected: All toolbar tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/casehub-diagram/src/casehub-diagram-toolbar.ts components/casehub-diagram/src/casehub-diagram-toolbar.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): casehub-diagram-toolbar — save button with dirty/saving state"
```

---

### Task 5: Properties Panel — Binding Target Type Selector

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram-properties.ts`

**Interfaces:**
- Consumes: existing `data` and `schema` properties
- Produces:
  - Event: `target-type-change` with `{ targetType: 'capability' | 'subCase' | 'humanTask' }` (composed, bubbles)

- [ ] **Step 1: Identify current target type detection**

The properties panel receives binding data. The current target type is determined by which field is present: `data.capability`, `data.subCase`, or `data.humanTask`.

- [ ] **Step 2: Add target type selector to the render method**

In `casehub-diagram-properties.ts`, modify the `render()` method to insert a target type selector before the property form when the data represents a binding (detectable by the presence of `capability`, `subCase`, or `humanTask`):

```typescript
private _currentTargetType(): string | null {
  if (this.data['capability'] !== undefined) return 'capability';
  if (this.data['subCase'] !== undefined) return 'subCase';
  if (this.data['humanTask'] !== undefined) return 'humanTask';
  return null;
}

private _renderTargetSelector(): TemplateResult | typeof nothing {
  const targetType = this._currentTargetType();
  if (!targetType) return nothing;

  return html`
    <label style="font-size: 12px; color: var(--pages-text-color, #333); margin-bottom: 8px; display: block;">
      Target type
      <select style="width: 100%; font-size: 12px; padding: 4px; margin-top: 2px;"
        ?disabled=${this.readonly}
        @change=${(e: Event) => {
          const newType = (e.target as HTMLSelectElement).value as 'capability' | 'subCase' | 'humanTask';
          if (newType !== targetType) {
            this.dispatchEvent(new CustomEvent('target-type-change', {
              bubbles: true, composed: true, detail: { targetType: newType },
            }));
          }
        }}>
        <option value="capability" ?selected=${targetType === 'capability'}>Capability</option>
        <option value="subCase" ?selected=${targetType === 'subCase'}>SubCase</option>
        <option value="humanTask" ?selected=${targetType === 'humanTask'}>HumanTask</option>
      </select>
    </label>
  `;
}
```

Update `render()`:

```typescript
override render() {
  const nodeName = String(this.data['name'] ?? this.data['type'] ?? 'Properties');
  return html`
    <div class="panel">
      <div class="panel-header">${nodeName}</div>
      ${this._renderTargetSelector()}
      ${renderPropertyForm(this.schema, this.data, this.readonly, (field, value) => {
        this.dispatchEvent(emitPropertyChange(field, value));
      })}
    </div>
  `;
}
```

- [ ] **Step 3: Typecheck**

Run: `yarn workspace casehub-diagram run typecheck`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/casehub-diagram/src/casehub-diagram-properties.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): binding target type selector — capability/subCase/humanTask switching"
```

---

### Task 6: casehub-diagram Integration — Layout, Structural Editing, Persistence, Render Guard

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram.ts`

**Interfaces:**
- Consumes:
  - `addElement`, `removeElement`, `switchBindingTarget` from Task 1
  - `PersistenceBackend`, `ReadResult`, `WriteResult` from `@casehubio/graph-core`
  - `<casehub-diagram-palette>` from Task 3
  - `<casehub-diagram-toolbar>` from Task 4
  - `target-type-change` event from Task 5
  - `edgesOf` from `@casehubio/graph-core` (for dependency check)
- Produces: Updated `<casehub-diagram>` with full Phase 4 capabilities

This is the largest task — it wires everything together. Broken into substeps.

- [ ] **Step 1: Add imports for new dependencies**

```typescript
import {
  toGraph,
  toReactFlowGraph,
  registerCaseStencils,
  applyPropertyEdit,
  addElement,
  removeElement,
  switchBindingTarget,
} from '@casehubio/graph-stencil-case';
import { computeElkLayout } from '@casehubio/graph-renderer';
import { edgesOf } from '@casehubio/graph-core';
import type { PersistenceBackend } from '@casehubio/graph-core';
import './casehub-diagram-palette.js';
import './casehub-diagram-toolbar.js';
import './casehub-diagram-properties.js';
```

- [ ] **Step 2: Add new properties and state**

```typescript
@property({ attribute: false }) backend: PersistenceBackend | null = null;
@property() uri = '';

private _savedYaml = '';
private _version = '';
private _saving = false;
private _renderInProgress = false;
private _pendingRenderYaml = '';
```

- [ ] **Step 3: Update _fullRender with render guard**

```typescript
private async _fullRender(yamlStr: string): Promise<void> {
  if (this._renderInProgress) {
    this._pendingRenderYaml = yamlStr;
    return;
  }
  this._renderInProgress = true;
  try {
    this._error = '';
    this._adapterResult = toGraph(yamlStr);
    const { nodes, edges } = toReactFlowGraph(this._adapterResult.model);
    this._nodes = await computeElkLayout(nodes, edges, { direction: 'DOWN', spacing: 60 }) as RFNode[];
    this._edges = edges;
  } catch (e) {
    this._error = String(e);
  } finally {
    this._renderInProgress = false;
    if (this._pendingRenderYaml && this._pendingRenderYaml !== yamlStr) {
      const pending = this._pendingRenderYaml;
      this._pendingRenderYaml = '';
      await this._fullRender(pending);
    } else {
      this._pendingRenderYaml = '';
    }
  }
}
```

- [ ] **Step 4: Update undo/redo to always use _fullRender**

```typescript
private async _undo(): Promise<void> {
  if (this._undoStack.length === 0) return;
  this._redoStack.push(this._currentYaml);
  this._currentYaml = this._undoStack.pop()!;
  await this._fullRender(this._currentYaml);
  this._updateSelectedNode();
}

private async _redo(): Promise<void> {
  if (this._redoStack.length === 0) return;
  this._undoStack.push(this._currentYaml);
  this._currentYaml = this._redoStack.pop()!;
  await this._fullRender(this._currentYaml);
  this._updateSelectedNode();
}
```

- [ ] **Step 5: Add palette-add handler**

```typescript
private _handlePaletteAdd = async (e: Event): Promise<void> => {
  const detail = (e as CustomEvent<{ elementType: 'binding' | 'worker' | 'milestone' | 'goal' }>).detail;
  this._undoStack.push(this._currentYaml);
  if (this._undoStack.length > MAX_UNDO) this._undoStack.shift();
  this._redoStack = [];
  this._currentYaml = addElement(this._currentYaml, detail.elementType);
  await this._fullRender(this._currentYaml);
};
```

- [ ] **Step 6: Add delete handler with dependency check and text input guard**

```typescript
private _handleDelete = async (): Promise<void> => {
  if (!this._selectedNodeId || !this._adapterResult) return;
  const node = this._adapterResult.model.nodes.find(n => n.id === this._selectedNodeId);
  if (!node || node.type === 'external') return;

  const nodePath = this._adapterResult.yamlPaths.get(this._selectedNodeId);
  if (!nodePath) return;

  const edges = edgesOf(this._adapterResult.model, this._selectedNodeId);
  if (edges.length > 0) {
    const name = String(node.properties['name'] ?? this._selectedNodeId);
    const confirmed = await this._confirmDelete(node.type, name, edges.length);
    if (!confirmed) return;
  }

  this._undoStack.push(this._currentYaml);
  if (this._undoStack.length > MAX_UNDO) this._undoStack.shift();
  this._redoStack = [];
  this._currentYaml = removeElement(this._currentYaml, nodePath);
  this._selectedNodeId = '';
  this._selectedData = {};
  this._selectedSchema = {};
  await this._fullRender(this._currentYaml);
};

private _confirmDelete(type: string, name: string, edgeCount: number): Promise<boolean> {
  return new Promise(resolve => {
    this._pendingConfirm = resolve;
    this._confirmMessage = type === 'worker'
      ? `Worker '${name}' has ${edgeCount} binding(s) dispatching to its capabilities. Those bindings will reference external capabilities after removal.`
      : `Remove ${type} '${name}'? It has ${edgeCount} connection(s).`;
    this.requestUpdate();
  });
}
```

- [ ] **Step 7: Add target-type-change handler**

```typescript
private _handleTargetTypeChange = async (e: Event): Promise<void> => {
  const detail = (e as CustomEvent<{ targetType: 'capability' | 'subCase' | 'humanTask' }>).detail;
  if (!this._selectedNodeId || !this._adapterResult) return;
  const nodePath = this._adapterResult.yamlPaths.get(this._selectedNodeId);
  if (!nodePath) return;

  this._undoStack.push(this._currentYaml);
  if (this._undoStack.length > MAX_UNDO) this._undoStack.shift();
  this._redoStack = [];
  this._currentYaml = switchBindingTarget(this._currentYaml, nodePath, detail.targetType);
  await this._fullRender(this._currentYaml);
  this._updateSelectedNode();
};
```

- [ ] **Step 8: Add persistence — load and save flows**

```typescript
override async updated(changed: Map<string, unknown>): Promise<void> {
  // ... existing yaml/src handling ...

  if ((changed.has('backend') || changed.has('uri')) && this.backend && this.uri) {
    await this._load();
  }
}

private async _load(): Promise<void> {
  if (!this.backend || this._saving) return;
  try {
    const result = await this.backend.read(this.uri);
    if (result.status === 'ok') {
      this._currentYaml = result.yaml;
      this._savedYaml = result.yaml;
      this._version = result.version;
      this._undoStack = [];
      this._redoStack = [];
      this._selectedNodeId = '';
      await this._fullRender(result.yaml);
    } else if (result.status === 'not_found') {
      const empty = 'dsl: "1.0.0"\nnamespace: \nname: \nversion: "1.0.0"\nspec:\n  bindings: []\n  workers: []\n';
      this._currentYaml = empty;
      this._savedYaml = empty;
      this._version = '';
      await this._fullRender(empty);
    } else if (result.status === 'parse_error') {
      this._error = result.message;
    } else if (result.status === 'schema_error') {
      this._currentYaml = result.yaml;
      this._savedYaml = result.yaml;
      this._version = result.version;
      await this._fullRender(result.yaml);
    }
  } catch (e) {
    this._error = `Load failed: ${e}`;
  }
}

private async _save(): Promise<void> {
  if (!this.backend || this._currentYaml === this._savedYaml || this._saving || this._renderInProgress) return;
  this._saving = true;
  this.requestUpdate();
  try {
    const result = await this.backend.write(this.uri, this._currentYaml, this._version);
    if (result.status === 'ok') {
      this._version = result.version;
      this._savedYaml = this._currentYaml;
    } else if (result.status === 'conflict') {
      this._conflictVersion = result.currentVersion;
      this._showConflict = true;
    }
  } catch (e) {
    this._error = `Save failed: ${e}`;
  } finally {
    this._saving = false;
    this.requestUpdate();
  }
}
```

- [ ] **Step 9: Update keydown handler for delete guard and Ctrl+S**

```typescript
private _handleKeydown = (e: KeyboardEvent): void => {
  const tag = (e.target as HTMLElement).tagName;
  const isTextInput = tag === 'INPUT' || tag === 'TEXTAREA' || (e.target as HTMLElement).isContentEditable;

  if (e.key === 'Escape') {
    this._selectedNodeId = '';
    this._selectedData = {};
    this._selectedSchema = {};
    return;
  }
  if ((e.key === 'Delete' || e.key === 'Backspace') && !isTextInput) {
    e.preventDefault();
    this._handleDelete();
    return;
  }
  if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) {
    e.preventDefault();
    this._undo();
  }
  if ((e.ctrlKey || e.metaKey) && e.key === 'z' && e.shiftKey) {
    e.preventDefault();
    this._redo();
  }
  if ((e.ctrlKey || e.metaKey) && e.key === 's') {
    e.preventDefault();
    this._save();
  }
};
```

- [ ] **Step 10: Update render method with new layout**

```typescript
override render() {
  if (this._error) {
    return html`<div style="color: red; padding: 16px;">${this._error}</div>`;
  }
  const hasSelection = this._selectedNodeId !== '';
  const isExternal = hasSelection && this._adapterResult?.model.nodes.find(n => n.id === this._selectedNodeId)?.type === 'external';
  const isDirty = this._currentYaml !== this._savedYaml;

  return html`
    <div style="display: flex; flex-direction: column; width: 100%; height: 100%;">
      <casehub-diagram-toolbar
        ?hasBackend=${this.backend != null}
        ?dirty=${isDirty}
        ?saving=${this._saving}
        @toolbar-save=${() => this._save()}
      ></casehub-diagram-toolbar>
      <div style="display: flex; flex: 1; overflow: hidden;">
        <casehub-diagram-palette
          ?disabled=${false}
          @palette-add=${this._handlePaletteAdd}
        ></casehub-diagram-palette>
        <pages-graph-canvas
          .nodes=${this._nodes}
          .edges=${this._edges}
          style="flex: 1; height: 100%;"
          @pages-event=${(e: CustomEvent) => {
            const topic = e.detail?.topic as string | undefined;
            if (topic === 'graph:node-click') this._handleNodeClick(e);
            if (topic === 'graph:selection-change') this._handleSelectionChange(e);
          }}
        ></pages-graph-canvas>
        ${hasSelection ? html`
          <div style="width: 300px; border-left: 1px solid var(--pages-border-color, #ddd); overflow-y: auto;">
            <casehub-diagram-properties
              .schema=${this._selectedSchema}
              .data=${this._selectedData}
              ?readonly=${isExternal ?? false}
              @property-change=${this._handlePropertyChange}
              @target-type-change=${this._handleTargetTypeChange}
            ></casehub-diagram-properties>
          </div>
        ` : nothing}
      </div>
      ${this._showConflict ? this._renderConflictDialog() : nothing}
      ${this._confirmMessage ? this._renderDeleteConfirm() : nothing}
    </div>
  `;
}
```

- [ ] **Step 11: Add conflict and delete confirmation dialogs**

```typescript
private _showConflict = false;
private _conflictVersion = '';
private _confirmMessage = '';
private _pendingConfirm: ((v: boolean) => void) | null = null;

private _renderConflictDialog() {
  return html`
    <div style="position: fixed; inset: 0; background: rgba(0,0,0,0.3); display: flex; align-items: center; justify-content: center; z-index: 1000;">
      <div style="background: var(--pages-surface-color, #fff); padding: 20px; border-radius: 8px; max-width: 400px; box-shadow: 0 4px 12px rgba(0,0,0,0.15);">
        <div style="font-weight: 600; margin-bottom: 12px;">Conflict detected</div>
        <div style="font-size: 13px; margin-bottom: 16px;">The file was modified externally since your last load.</div>
        <div style="display: flex; gap: 8px; justify-content: flex-end;">
          <button @click=${() => this._resolveConflict('cancel')}>Keep editing</button>
          <button @click=${() => this._resolveConflict('reload')}>Discard my changes</button>
          <button @click=${() => this._resolveConflict('overwrite')}>Save anyway</button>
        </div>
      </div>
    </div>
  `;
}

private async _resolveConflict(action: 'overwrite' | 'reload' | 'cancel'): Promise<void> {
  this._showConflict = false;
  if (action === 'overwrite' && this.backend) {
    this._version = this._conflictVersion;
    await this._save();
  } else if (action === 'reload') {
    await this._load();
  }
  this.requestUpdate();
}

private _renderDeleteConfirm() {
  return html`
    <div style="position: fixed; inset: 0; background: rgba(0,0,0,0.3); display: flex; align-items: center; justify-content: center; z-index: 1000;">
      <div style="background: var(--pages-surface-color, #fff); padding: 20px; border-radius: 8px; max-width: 400px; box-shadow: 0 4px 12px rgba(0,0,0,0.15);">
        <div style="font-size: 13px; margin-bottom: 16px;">${this._confirmMessage}</div>
        <div style="display: flex; gap: 8px; justify-content: flex-end;">
          <button @click=${() => { this._confirmMessage = ''; this._pendingConfirm?.(false); this.requestUpdate(); }}>Cancel</button>
          <button @click=${() => { this._confirmMessage = ''; this._pendingConfirm?.(true); this.requestUpdate(); }}>Remove</button>
        </div>
      </div>
    </div>
  `;
}
```

- [ ] **Step 12: Run full test suite and typecheck**

Run: `yarn test && yarn typecheck`
Expected: All PASS, no type errors

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/casehub-diagram/src/casehub-diagram.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): Phase 4 integration — structural editing, persistence, render guard, delete, layout"
```

---

### Task 7: Integration Tests

**Files:**
- Create/Modify: `components/casehub-diagram/src/casehub-diagram.test.ts` (or extend existing)

**Interfaces:**
- Consumes: All components from Tasks 1-6

- [ ] **Step 1: Write integration tests for structural editing and persistence**

Tests covering spec §10 items 7-8, 11-14:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { addElement, removeElement, switchBindingTarget, toGraph } from '@casehubio/graph-stencil-case';
import { InMemoryBackend } from '@casehubio/graph-core';

const SAMPLE = `dsl: "1.0.0"\nnamespace: test\nname: sample\nversion: "1.0.0"\nspec:\n  bindings:\n    - name: b1\n      capability: cap1\n  workers:\n    - name: w1\n      capabilities:\n        - cap1\n`;

describe('structural editing round-trip', () => {
  it('add binding → toGraph → new node exists', () => {
    const yaml = addElement(SAMPLE, 'binding');
    const { model } = toGraph(yaml);
    expect(model.nodes.some(n => n.id === 'binding:binding-1')).toBe(true);
  });

  it('remove worker → toGraph → binding becomes external', () => {
    const yaml = removeElement(SAMPLE, ['spec', 'workers', 0]);
    const { model } = toGraph(yaml);
    expect(model.nodes.some(n => n.type === 'external')).toBe(true);
  });

  it('switchBindingTarget → toGraph → topology changes', () => {
    const yaml = switchBindingTarget(SAMPLE, ['spec', 'bindings', 0], 'subCase');
    const { model } = toGraph(yaml);
    const capEdges = model.edges.filter(e => e.type === 'capability-dispatch' && e.source === 'binding:b1');
    expect(capEdges).toHaveLength(0);
    expect(model.nodes.some(n => n.type === 'subcase')).toBe(true);
  });
});

describe('persistence round-trip', () => {
  it('write then read returns the same yaml', async () => {
    const backend = new InMemoryBackend();
    const writeResult = await backend.write('test.yaml', SAMPLE, '');
    expect(writeResult.status).toBe('ok');
    const readResult = await backend.read('test.yaml');
    expect(readResult.status).toBe('ok');
    if (readResult.status === 'ok') {
      expect(readResult.yaml).toBe(SAMPLE);
    }
  });

  it('write with stale version returns conflict', async () => {
    const backend = new InMemoryBackend();
    await backend.write('test.yaml', SAMPLE, '');
    await backend.write('test.yaml', 'changed\n', '1');
    const result = await backend.write('test.yaml', 'stale\n', '1');
    expect(result.status).toBe('conflict');
  });

  it('dirty tracking: current !== saved after edit', () => {
    const saved = SAMPLE;
    const current = addElement(SAMPLE, 'binding');
    expect(current !== saved).toBe(true);
  });

  it('dirty tracking: undo past save point marks dirty', () => {
    const edited = addElement(SAMPLE, 'binding');
    const savedYaml = edited;
    const afterUndo = SAMPLE;
    expect(afterUndo !== savedYaml).toBe(true);
  });
});
```

- [ ] **Step 2: Run all tests**

Run: `yarn test`
Expected: All PASS

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/casehub-diagram/src/casehub-diagram.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "test(#103): Phase 4 integration tests — structural editing, persistence, dirty tracking"
```

---

## Dependency Graph

```
Task 1 (yaml-editor ops)    ──┐
Task 2 (GitHubBackend)       ──┤
Task 3 (Palette)             ──┼── Task 6 (Integration) ── Task 7 (Integration Tests)
Task 4 (Toolbar)             ──┤
Task 5 (Target selector)    ──┘
        └── depends on Task 1
```

Tasks 1-4 are independent. Task 5 depends on Task 1. Task 6 depends on all. Task 7 depends on Task 6.
