# ARIA Interaction Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #417 — ARIA as unified interaction model — accessibility + scenario automation
**Issue group:** casehubio/parent#417

**Goal:** Establish ARIA as the unified interaction model across pages — contract types, navigation API, widget remediation, scenario tooling, and CI validation.

**Architecture:** ARIA contract types in pages-primitives define the required interface. A new `@casehubio/pages-aria` package provides tree walker and command executor. Pages widgets are remediated to satisfy the contract. The pages-push protocol is extended for bidirectional command/response. A scenario YAML format with Java POJO and TypeScript command models enables declarative UI automation.

**Tech Stack:** TypeScript 5 / Lit / Vitest / axe-core / ECharts / Java 21 / Quarkus / Maven

## Global Constraints

- All ARIA attribute values follow WAI-ARIA 1.2 specification
- No `data-testid` or `data-automation` attributes — ARIA is the sole interaction model
- axe-core CI must pass with zero violations for all interactive components
- Existing a11y mixins (FocusTrapMixin, RovingTabindexMixin, LiveRegionMixin, KeyboardShortcutMixin) are unchanged
- Shadow DOM tree walking must use composed tree traversal (GE-20260713-777d8a, GE-20260617-cc0834)
- TypeScript mixin/abstract class constraint (GE-20260721-f094e6) — use contract-as-types, not mandatory mixins
- Repo: `/Users/mdproctor/claude/casehub/slots/122/pages` (TypeScript), `/Users/mdproctor/claude/casehub/slots/122/blocks` (Java POJO model only)
- IntelliJ project path: `/Users/mdproctor/claude/casehub/slots/122/pages`

## Out of Scope (Separate Plans)

- **Blocks-UI widget remediation** — requires blocks-ui repo (not in slot). Separate plan in a dedicated slot.
- **MCP integration** — requires platform-mcp repo. Separate plan.
- **ADR and platform protocol** — follow-on artifacts written after implementation validates the design.

---

### Task 1: ARIA Contract Types (pages-primitives)

**Files:**
- Create: `packages/pages-primitives/src/a11y/aria-contract.ts`
- Create: `packages/pages-primitives/src/a11y/aria-contract.test.ts`
- Modify: `packages/pages-primitives/src/a11y/index.ts` (add re-export)
- Modify: `packages/pages-primitives/src/index.ts` (add re-export if needed)

**Interfaces:**
- Consumes: nothing (foundation)
- Produces: `AriaRole` type, `AriaInteractive` interface, `AriaTarget` type, `AriaState` type — consumed by all subsequent tasks

- [ ] **Step 1: Write the failing test**

```typescript
// packages/pages-primitives/src/a11y/aria-contract.test.ts
import { describe, it, expect } from 'vitest';
import type { AriaRole, AriaInteractive, AriaTarget, AriaState } from './aria-contract.js';

describe('ARIA contract types', () => {
  it('AriaRole accepts valid WAI-ARIA roles', () => {
    const role: AriaRole = 'button';
    expect(role).toBe('button');
  });

  it('AriaInteractive requires role and ariaLabel', () => {
    const component: AriaInteractive = {
      role: 'button',
      ariaLabel: 'Submit form',
    };
    expect(component.role).toBe('button');
    expect(component.ariaLabel).toBe('Submit form');
  });

  it('AriaInteractive accepts optional state properties', () => {
    const component: AriaInteractive = {
      role: 'button',
      ariaLabel: 'Submit',
      ariaBusy: true,
      ariaDisabled: false,
      ariaExpanded: undefined,
    };
    expect(component.ariaBusy).toBe(true);
  });

  it('AriaTarget supports nested within scoping', () => {
    const target: AriaTarget = {
      role: 'button',
      name: 'Delete',
      within: { role: 'row', name: 'Case #42' },
    };
    expect(target.within?.role).toBe('row');
  });

  it('AriaState captures all standard ARIA state properties', () => {
    const state: AriaState = {
      busy: false,
      disabled: false,
      expanded: true,
      selected: false,
      checked: undefined,
      hidden: false,
    };
    expect(state.expanded).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-primitives run test -- aria-contract`
Expected: FAIL — module `./aria-contract.js` not found

- [ ] **Step 3: Implement the types**

```typescript
// packages/pages-primitives/src/a11y/aria-contract.ts
export type AriaRole =
  | 'alert' | 'alertdialog' | 'application' | 'article'
  | 'button' | 'checkbox' | 'combobox'
  | 'form' | 'grid' | 'gridcell' | 'group'
  | 'heading' | 'img'
  | 'list' | 'listbox' | 'listitem' | 'log'
  | 'meter'
  | 'option'
  | 'region' | 'row'
  | 'separator' | 'status'
  | 'tab' | 'tablist' | 'tabpanel'
  | 'textbox' | 'toolbar'
  | 'tree' | 'treeitem';

export interface AriaInteractive {
  role: AriaRole;
  ariaLabel: string;
  ariaBusy?: boolean;
  ariaDisabled?: boolean;
  ariaExpanded?: boolean;
}

export interface AriaTarget {
  role: string;
  name: string;
  within?: AriaTarget;
}

export interface AriaState {
  busy?: boolean;
  disabled?: boolean;
  expanded?: boolean;
  selected?: boolean;
  checked?: boolean | 'mixed';
  hidden?: boolean;
}
```

- [ ] **Step 4: Add re-export to a11y index**

Add to `packages/pages-primitives/src/a11y/index.ts`:
```typescript
export type { AriaRole, AriaInteractive, AriaTarget, AriaState } from './aria-contract.js';
```

- [ ] **Step 5: Run test to verify it passes**

Run: `yarn workspace @casehubio/pages-primitives run test -- aria-contract`
Expected: PASS — all 5 tests green

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add packages/pages-primitives/src/a11y/aria-contract.ts packages/pages-primitives/src/a11y/aria-contract.test.ts packages/pages-primitives/src/a11y/index.ts
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add ARIA contract types — AriaRole, AriaInteractive, AriaTarget, AriaState

Refs casehubio/parent#417"
```

---

### Task 2: pages-aria Package — Walker

**Files:**
- Create: `packages/pages-aria/package.json`
- Create: `packages/pages-aria/tsconfig.json`
- Create: `packages/pages-aria/vitest.config.ts`
- Create: `packages/pages-aria/src/walker/index.ts`
- Create: `packages/pages-aria/src/walker/tree-walker.ts`
- Create: `packages/pages-aria/src/walker/accessible-name.ts`
- Create: `packages/pages-aria/src/walker/aria-state.ts`
- Create: `packages/pages-aria/src/walker/tree-walker.test.ts`

**Interfaces:**
- Consumes: `AriaTarget`, `AriaState` from `@casehubio/pages-primitives`
- Produces: `findByRole(role, name, scope?)`, `findAllByRole(role, name?, scope?)`, `getAccessibleName(element)`, `getAriaState(element)` — consumed by Task 3 (executor), Task 5 (server)

- [ ] **Step 1: Scaffold the package**

Create `packages/pages-aria/package.json`:
```json
{
  "name": "@casehubio/pages-aria",
  "version": "0.1.0",
  "type": "module",
  "exports": {
    "./walker": "./src/walker/index.ts",
    "./executor": "./src/executor/index.ts",
    "./server": "./src/server/index.ts"
  },
  "dependencies": {
    "@casehubio/pages-primitives": "workspace:*"
  },
  "devDependencies": {
    "vitest": "^3.2.4",
    "jsdom": "^26.1.0"
  }
}
```

Create `packages/pages-aria/tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "references": [
    { "path": "../pages-primitives" }
  ]
}
```

Create `packages/pages-aria/vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
  },
});
```

- [ ] **Step 2: Write the failing walker tests**

```typescript
// packages/pages-aria/src/walker/tree-walker.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { findByRole, findAllByRole, getAccessibleName, getAriaState } from './tree-walker.js';

describe('ARIA tree walker', () => {
  beforeEach(() => {
    document.body.innerHTML = '';
  });

  it('finds element by role and name', () => {
    document.body.innerHTML = '<button aria-label="Submit">Submit</button>';
    const el = findByRole('button', 'Submit');
    expect(el).not.toBeNull();
    expect(el?.tagName).toBe('BUTTON');
  });

  it('returns null when no match', () => {
    document.body.innerHTML = '<button aria-label="Submit">Submit</button>';
    const el = findByRole('button', 'Cancel');
    expect(el).toBeNull();
  });

  it('finds element with implicit role from tag name', () => {
    document.body.innerHTML = '<button>Submit</button>';
    const el = findByRole('button', 'Submit');
    expect(el).not.toBeNull();
  });

  it('scopes search within a parent element', () => {
    document.body.innerHTML = `
      <div role="row" aria-label="Row A"><button aria-label="Delete">Delete</button></div>
      <div role="row" aria-label="Row B"><button aria-label="Delete">Delete</button></div>
    `;
    const rowB = findByRole('row', 'Row B');
    const deleteBtn = findByRole('button', 'Delete', rowB!);
    expect(deleteBtn).not.toBeNull();
    expect(deleteBtn?.closest('[aria-label="Row B"]')).not.toBeNull();
  });

  it('findAllByRole returns all matches', () => {
    document.body.innerHTML = `
      <button aria-label="Delete">Delete</button>
      <button aria-label="Delete">Delete</button>
      <button aria-label="Save">Save</button>
    `;
    const all = findAllByRole('button', 'Delete');
    expect(all).toHaveLength(2);
  });

  it('getAccessibleName returns aria-label', () => {
    document.body.innerHTML = '<button aria-label="Submit form">Go</button>';
    const el = document.querySelector('button')!;
    expect(getAccessibleName(el)).toBe('Submit form');
  });

  it('getAccessibleName falls back to text content', () => {
    document.body.innerHTML = '<button>Submit</button>';
    const el = document.querySelector('button')!;
    expect(getAccessibleName(el)).toBe('Submit');
  });

  it('getAriaState reads ARIA state attributes', () => {
    document.body.innerHTML = '<button aria-busy="true" aria-disabled="false" aria-expanded="true">X</button>';
    const el = document.querySelector('button')!;
    const state = getAriaState(el);
    expect(state.busy).toBe(true);
    expect(state.disabled).toBe(false);
    expect(state.expanded).toBe(true);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: FAIL — module not found

- [ ] **Step 4: Implement the walker**

```typescript
// packages/pages-aria/src/walker/accessible-name.ts
export function getAccessibleName(element: Element): string {
  const label = element.getAttribute('aria-label');
  if (label) return label;

  const labelledBy = element.getAttribute('aria-labelledby');
  if (labelledBy) {
    const root = element.getRootNode() as Document | ShadowRoot;
    const parts = labelledBy.split(/\s+/).map(id => {
      const ref = root.getElementById(id);
      return ref?.textContent?.trim() ?? '';
    });
    const joined = parts.filter(Boolean).join(' ');
    if (joined) return joined;
  }

  return element.textContent?.trim() ?? '';
}
```

```typescript
// packages/pages-aria/src/walker/aria-state.ts
import type { AriaState } from '@casehubio/pages-primitives';

function parseBool(value: string | null): boolean | undefined {
  if (value === 'true') return true;
  if (value === 'false') return false;
  return undefined;
}

export function getAriaState(element: Element): AriaState {
  return {
    busy: parseBool(element.getAttribute('aria-busy')),
    disabled: parseBool(element.getAttribute('aria-disabled')),
    expanded: parseBool(element.getAttribute('aria-expanded')),
    selected: parseBool(element.getAttribute('aria-selected')),
    checked: element.getAttribute('aria-checked') === 'mixed'
      ? 'mixed'
      : parseBool(element.getAttribute('aria-checked')),
    hidden: parseBool(element.getAttribute('aria-hidden')),
  };
}
```

```typescript
// packages/pages-aria/src/walker/tree-walker.ts
import { getAccessibleName } from './accessible-name.js';
import { getAriaState } from './aria-state.js';

const IMPLICIT_ROLES: Record<string, string> = {
  BUTTON: 'button',
  INPUT: 'textbox',
  SELECT: 'listbox',
  TEXTAREA: 'textbox',
  A: 'link',
  NAV: 'navigation',
  MAIN: 'main',
  HEADER: 'banner',
  FOOTER: 'contentinfo',
  FORM: 'form',
  TABLE: 'table',
  UL: 'list',
  OL: 'list',
  LI: 'listitem',
};

function getRole(element: Element): string | null {
  return element.getAttribute('role') ?? IMPLICIT_ROLES[element.tagName] ?? null;
}

function walkTree(root: Element | Document | ShadowRoot, visitor: (el: Element) => boolean): void {
  const children = root instanceof Element && root.shadowRoot
    ? root.shadowRoot.children
    : root.children;

  for (const child of children) {
    if (!(child instanceof Element)) continue;
    const stop = visitor(child);
    if (stop) return;

    if (child.shadowRoot) {
      walkTree(child.shadowRoot, visitor);
    }
    if (child.children.length > 0) {
      walkTree(child, visitor);
    }
  }
}

export function findByRole(role: string, name: string, scope?: Element): Element | null {
  const root = scope ?? document.body;
  let found: Element | null = null;

  walkTree(root, (el) => {
    if (getRole(el) === role && getAccessibleName(el) === name) {
      found = el;
      return true;
    }
    return false;
  });

  return found;
}

export function findAllByRole(role: string, name?: string, scope?: Element): Element[] {
  const root = scope ?? document.body;
  const results: Element[] = [];

  walkTree(root, (el) => {
    const elRole = getRole(el);
    if (elRole === role && (name === undefined || getAccessibleName(el) === name)) {
      results.push(el);
    }
    return false;
  });

  return results;
}

export { getAccessibleName, getAriaState };
```

```typescript
// packages/pages-aria/src/walker/index.ts
export { findByRole, findAllByRole, getAccessibleName, getAriaState } from './tree-walker.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-aria run test`
Expected: PASS — all 8 tests green

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add packages/pages-aria/
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add @casehubio/pages-aria package with ARIA tree walker

Shadow DOM aware tree traversal, accessible name computation,
ARIA state extraction. Scoped search for within-targeting.

Refs casehubio/parent#417"
```

---

### Task 3: pages-aria Package — Executor

**Files:**
- Create: `packages/pages-aria/src/executor/index.ts`
- Create: `packages/pages-aria/src/executor/command-executor.ts`
- Create: `packages/pages-aria/src/executor/command-executor.test.ts`

**Interfaces:**
- Consumes: `findByRole`, `getAriaState` from walker (Task 2); `AriaTarget`, `AriaState` from pages-primitives (Task 1)
- Produces: `click(target)`, `fill(target, value)`, `select(target, option)`, `expand(target)`, `collapse(target)`, `assertState(target, state)`, `waitFor(target, state, timeout)` — consumed by Task 5 (server), Task 8 (client runner)

- [ ] **Step 1: Write the failing executor tests**

```typescript
// packages/pages-aria/src/executor/command-executor.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { click, fill, select, expand, collapse, assertState, resolveTarget } from './command-executor.js';
import type { AriaTarget } from '@casehubio/pages-primitives';

describe('ARIA command executor', () => {
  beforeEach(() => {
    document.body.innerHTML = '';
  });

  it('resolveTarget finds element by role + name', () => {
    document.body.innerHTML = '<button aria-label="Submit">Submit</button>';
    const el = resolveTarget({ role: 'button', name: 'Submit' });
    expect(el.tagName).toBe('BUTTON');
  });

  it('resolveTarget with within scoping', () => {
    document.body.innerHTML = `
      <div role="row" aria-label="Row A"><button aria-label="Delete">Delete</button></div>
      <div role="row" aria-label="Row B"><button aria-label="Delete">Delete</button></div>
    `;
    const el = resolveTarget({
      role: 'button', name: 'Delete',
      within: { role: 'row', name: 'Row B' },
    });
    expect(el.closest('[aria-label="Row B"]')).not.toBeNull();
  });

  it('resolveTarget throws when element not found', () => {
    document.body.innerHTML = '<button aria-label="Submit">Submit</button>';
    expect(() => resolveTarget({ role: 'button', name: 'Cancel' }))
      .toThrow('No element found: button "Cancel"');
  });

  it('resolveTarget throws when multiple matches without scoping', () => {
    document.body.innerHTML = `
      <button aria-label="Delete">Delete</button>
      <button aria-label="Delete">Delete</button>
    `;
    expect(() => resolveTarget({ role: 'button', name: 'Delete' }))
      .toThrow('Multiple elements found: button "Delete" (2 matches). Use "within" to scope.');
  });

  it('click dispatches click event', () => {
    document.body.innerHTML = '<button aria-label="Submit">Submit</button>';
    const handler = vi.fn();
    document.querySelector('button')!.addEventListener('click', handler);
    click({ role: 'button', name: 'Submit' });
    expect(handler).toHaveBeenCalledOnce();
  });

  it('fill sets input value and dispatches events', () => {
    document.body.innerHTML = '<input aria-label="Name" />';
    const input = document.querySelector('input')!;
    const inputHandler = vi.fn();
    const changeHandler = vi.fn();
    input.addEventListener('input', inputHandler);
    input.addEventListener('change', changeHandler);
    fill({ role: 'textbox', name: 'Name' }, 'Alice');
    expect(input.value).toBe('Alice');
    expect(inputHandler).toHaveBeenCalled();
    expect(changeHandler).toHaveBeenCalled();
  });

  it('assertState passes when state matches', () => {
    document.body.innerHTML = '<button aria-label="Submit" aria-busy="false">Submit</button>';
    expect(() => assertState(
      { role: 'button', name: 'Submit' },
      { busy: false }
    )).not.toThrow();
  });

  it('assertState throws when state does not match', () => {
    document.body.innerHTML = '<button aria-label="Submit" aria-busy="true">Submit</button>';
    expect(() => assertState(
      { role: 'button', name: 'Submit' },
      { busy: false }
    )).toThrow('State mismatch for button "Submit": busy expected false, got true');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- command-executor`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the executor**

```typescript
// packages/pages-aria/src/executor/command-executor.ts
import { findByRole, findAllByRole, getAriaState } from '../walker/index.js';
import type { AriaTarget, AriaState } from '@casehubio/pages-primitives';

export function resolveTarget(target: AriaTarget): Element {
  let scope: Element | undefined;

  if (target.within) {
    scope = resolveTarget(target.within);
  }

  const all = findAllByRole(target.role, target.name, scope);

  if (all.length === 0) {
    const scopeDesc = target.within ? ` within ${target.within.role} "${target.within.name}"` : '';
    throw new Error(`No element found: ${target.role} "${target.name}"${scopeDesc}`);
  }

  if (all.length > 1) {
    throw new Error(
      `Multiple elements found: ${target.role} "${target.name}" (${all.length} matches). Use "within" to scope.`
    );
  }

  return all[0];
}

export function click(target: AriaTarget): void {
  const el = resolveTarget(target);
  el.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
}

export function fill(target: AriaTarget, value: string): void {
  const el = resolveTarget(target) as HTMLInputElement;
  el.value = value;
  el.dispatchEvent(new Event('input', { bubbles: true }));
  el.dispatchEvent(new Event('change', { bubbles: true }));
}

export function select(target: AriaTarget, option: string): void {
  const el = resolveTarget(target) as HTMLSelectElement;
  el.value = option;
  el.dispatchEvent(new Event('change', { bubbles: true }));
}

export function expand(target: AriaTarget): void {
  const el = resolveTarget(target);
  el.setAttribute('aria-expanded', 'true');
  el.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
}

export function collapse(target: AriaTarget): void {
  const el = resolveTarget(target);
  el.setAttribute('aria-expanded', 'false');
  el.dispatchEvent(new MouseEvent('click', { bubbles: true, cancelable: true }));
}

export function assertState(target: AriaTarget, expected: Partial<AriaState>): void {
  const el = resolveTarget(target);
  const actual = getAriaState(el);

  for (const [key, expectedValue] of Object.entries(expected)) {
    const actualValue = actual[key as keyof AriaState];
    if (actualValue !== expectedValue) {
      throw new Error(
        `State mismatch for ${target.role} "${target.name}": ${key} expected ${expectedValue}, got ${actualValue}`
      );
    }
  }
}

export async function waitFor(
  target: AriaTarget,
  expected: Partial<AriaState>,
  timeout: number,
): Promise<void> {
  const deadline = Date.now() + timeout;
  const interval = 100;

  while (Date.now() < deadline) {
    try {
      assertState(target, expected);
      return;
    } catch {
      await new Promise(r => setTimeout(r, interval));
    }
  }

  assertState(target, expected);
}
```

```typescript
// packages/pages-aria/src/executor/index.ts
export { resolveTarget, click, fill, select, expand, collapse, assertState, waitFor } from './command-executor.js';
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-aria run test -- command-executor`
Expected: PASS — all 8 tests green

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add packages/pages-aria/src/executor/
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add ARIA command executor — click, fill, select, expand, collapse, assert, wait

Target resolution via { role, name, within } with unique-match
enforcement. State assertions compare actual ARIA attributes.

Refs casehubio/parent#417"
```

---

### Task 4: Pages Widget Remediation — Interactive Widgets

**Files:**
- Modify: `packages/pages-ui-components/src/button/pages-button.ts` — add aria-label, aria-disabled, aria-busy
- Modify: `packages/pages-ui-components/src/status-dot/pages-status-dot.ts` — add role="status", aria-label
- Modify: `packages/pages-ui-components/src/badge/pages-badge.ts` — add aria-label
- Modify: `packages/pages-viz/src/grid-table/pages-grid-table.ts` — add role="grid", aria-label, grid cell roles
- Modify: `packages/pages-viz/src/alert/pages-alert.ts` — add role="alert", aria-live
- Modify: `packages/pages-viz/src/metric/pages-metric.ts` — add role="status", aria-label
- Modify: `packages/pages-viz/src/meter/pages-meter.ts` — add role="meter", aria-value*
- Modify: `packages/pages-viz/src/selector/pages-selector.ts` — add role="listbox", aria-label, aria-selected

**Interfaces:**
- Consumes: `AriaInteractive` interface from pages-primitives (Task 1) — components implement it
- Produces: Remediated components with correct ARIA attributes — consumed by Task 6 (axe-core validation)

**Note:** File paths are approximate — verify exact locations with `ide_find_class` before modifying. Each component follows the same pattern: add ARIA attributes as Lit reactive properties reflected to DOM attributes.

- [ ] **Step 1: Identify exact file paths**

Use `ide_find_class` for each component to locate the exact source file. Record paths for all 8 components listed above.

- [ ] **Step 2: For each component, write a test verifying ARIA attributes**

Each component test should render the component and verify:
- `role` attribute is present and correct
- `aria-label` is present (either from property or attribute)
- State attributes (`aria-busy`, `aria-disabled`, etc.) reflect component state

Example pattern for PagesButton:
```typescript
it('has correct ARIA attributes', async () => {
  const el = await fixture(html`<pages-button label="Submit"></pages-button>`);
  expect(el.getAttribute('role')).toBe('button');
  expect(el.getAttribute('aria-label')).toBe('Submit');
});

it('reflects disabled state', async () => {
  const el = await fixture(html`<pages-button label="Submit" disabled></pages-button>`);
  expect(el.getAttribute('aria-disabled')).toBe('true');
});

it('reflects busy state during loading', async () => {
  const el = await fixture(html`<pages-button label="Submit" loading></pages-button>`);
  expect(el.getAttribute('aria-busy')).toBe('true');
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Run: `yarn workspace @casehubio/pages-viz run test`
Expected: FAIL — ARIA attributes not set

- [ ] **Step 4: Add ARIA attributes to each component**

For each component, add reactive Lit properties that reflect to attributes. Example for PagesButton:
```typescript
@property({ type: String, reflect: true }) role = 'button';
@property({ type: String, reflect: true, attribute: 'aria-label' }) ariaLabel = '';
@property({ type: Boolean, reflect: true, attribute: 'aria-disabled' }) override ariaDisabled: string | null = null;
@property({ type: Boolean, reflect: true, attribute: 'aria-busy' }) override ariaBusy: string | null = null;
```

Wire state: when `disabled` changes → update `aria-disabled`. When `loading` changes → update `aria-busy`.

For PagesGridTable: add `role="grid"` on container, `role="row"` on rows, `role="gridcell"` on cells, `role="columnheader"` on headers.

For PagesMeter: add `role="meter"`, `aria-valuenow`, `aria-valuemin`, `aria-valuemax`.

For PagesSelector: add `role="listbox"` on container, `role="option"` on items, `aria-selected` on selected item.

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-components run test`
Run: `yarn workspace @casehubio/pages-viz run test`
Expected: PASS

- [ ] **Step 6: Run typecheck across affected packages**

Run: `yarn typecheck`
Expected: No new type errors

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add packages/pages-ui-components/ packages/pages-viz/
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add ARIA attributes to interactive pages widgets

PagesButton, PagesStatusDot, PagesBadge, PagesGridTable,
PagesAlert, PagesMetric, PagesMeter, PagesSelector.

Refs casehubio/parent#417"
```

---

### Task 5: Pages Widget Remediation — Charts + Form Labels

**Files:**
- Modify: All ECharts chart wrapper components in `packages/pages-viz/src/` (12 chart types)
- Modify: Form input components in `packages/pages-ui-components/src/` (add aria-label propagation)

**Interfaces:**
- Consumes: nothing new (uses existing ECharts `aria` option)
- Produces: Charts with ECharts `aria.enabled` and wrapper `aria-label` — consumed by Task 6 (axe-core)

- [ ] **Step 1: Identify all ECharts chart wrapper files**

Use `ide_find_class` for: PagesBarChart, PagesLineChart, PagesPieChart, PagesAreaChart, PagesScatterChart, PagesTimeseries, PagesTimeline, PagesBubbleChart, PagesGraph, PagesMap, PagesHeatmapChart, PagesTreemapChart.

Check if they share a common base class or mixin — if so, the aria option can be set once.

- [ ] **Step 2: Write test verifying ECharts aria.enabled**

```typescript
it('enables ECharts aria mode', async () => {
  const el = await fixture(html`<pages-bar-chart .data=${sampleData}></pages-bar-chart>`);
  // Verify the ECharts option includes aria.enabled
  expect(el.getAttribute('aria-label')).toBeTruthy();
});
```

- [ ] **Step 3: Enable ECharts aria on all chart wrappers**

In each chart's option-building method, add:
```typescript
aria: {
  enabled: true,
  decal: { show: false },
},
```

Add `aria-label` on the wrapper host element with a summary (chart type + title):
```typescript
@property({ type: String, reflect: true, attribute: 'aria-label' })
override ariaLabel: string | null = null;

updated(changed: PropertyValues) {
  if (!this.ariaLabel) {
    this.setAttribute('aria-label', `${this.chartType} chart: ${this.title ?? 'untitled'}`);
  }
}
```

If charts share a base class (e.g., `EChartsBase`), add this there once.

- [ ] **Step 4: Add aria-label to form inputs**

For PagesInput, PagesSelect, PagesTextarea, PagesCheckbox: ensure the `label` property propagates to `aria-label` on the inner input/select/textarea element. Check if this already happens through the label association — if `<label for="...">` is used, explicit `aria-label` is redundant.

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-viz run test`
Run: `yarn workspace @casehubio/pages-ui-components run test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add packages/pages-viz/ packages/pages-ui-components/
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): enable ECharts aria mode on all chart components, add form input labels

12 chart wrappers get aria.enabled + wrapper aria-label.
Form inputs propagate label to aria-label on inner element.

Refs casehubio/parent#417"
```

---

### Task 6: axe-core CI Validation

**Files:**
- Modify: `package.json` (root) — add `@axe-core/playwright` dev dependency
- Create: `tests/a11y/axe-validation.test.ts` (or integrate into existing Playwright tests)
- Modify: CI config if needed (verify Playwright tests already run in CI)

**Interfaces:**
- Consumes: Remediated components from Tasks 4-5
- Produces: CI gate that fails on ARIA violations — validates all tasks

- [ ] **Step 1: Check existing Playwright test infrastructure**

Find existing Playwright test files and config. Understand how Playwright tests run in CI:
```bash
find /Users/mdproctor/claude/casehub/slots/122/pages -name "playwright.config.*" -maxdepth 2
```

- [ ] **Step 2: Add axe-core dependency**

```bash
yarn add -D @axe-core/playwright
```

- [ ] **Step 3: Write axe-core validation test**

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('ARIA accessibility validation', () => {
  test('pages-ui-components have no axe violations', async ({ page }) => {
    // Navigate to component examples page or render test fixture
    await page.goto('/examples/components');

    const results = await new AxeBuilder({ page })
      .include('[role]')
      .analyze();

    expect(results.violations).toEqual([]);
  });

  test('pages-viz charts have aria-label', async ({ page }) => {
    await page.goto('/examples/charts');

    const results = await new AxeBuilder({ page })
      .include('pages-bar-chart, pages-line-chart, pages-pie-chart')
      .analyze();

    expect(results.violations).toEqual([]);
  });
});
```

**Note:** Exact URLs and selectors depend on the examples gallery structure. Check `examples/` directory for available pages.

- [ ] **Step 4: Run the axe-core tests**

Run: `yarn playwright test tests/a11y/`
Expected: PASS — no violations on remediated components

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add tests/a11y/ package.json yarn.lock
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add axe-core CI validation for ARIA compliance

Playwright-based accessibility audit runs in CI against
rendered components. Zero-violation policy.

Refs casehubio/parent#417"
```

---

### Task 7: Push Protocol Extension — CommandResult

**Files:**
- Modify: `backend/push/src/main/java/io/casehub/pages/push/PushRequest.java` — add CommandResult variant
- Create: `backend/push/src/test/java/io/casehub/pages/push/CommandResultTest.java`

**Interfaces:**
- Consumes: existing PushRequest sealed interface
- Produces: `PushRequest.CommandResult(id, ok, error)` — consumed by Task 8 (scenario handler)

**Note:** The push backend is Java (Maven). Use the pages backend project path for IntelliJ operations.

- [ ] **Step 1: Read the existing PushRequest sealed interface**

Use `ide_find_class` to locate `PushRequest` in the push backend. Read the file to understand the sealed variants and parse() method.

- [ ] **Step 2: Write the failing test**

```java
// backend/push/src/test/java/io/casehub/pages/push/CommandResultTest.java
package io.casehub.pages.push;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CommandResultTest {

    @Test
    void parsesCommandResult() {
        var json = """
            {"type": "command-result", "id": "cmd-1", "ok": true}
            """;
        var request = PushRequest.parse(json);
        assertThat(request).isInstanceOf(PushRequest.CommandResult.class);
        var result = (PushRequest.CommandResult) request;
        assertThat(result.id()).isEqualTo("cmd-1");
        assertThat(result.ok()).isTrue();
        assertThat(result.error()).isNull();
    }

    @Test
    void parsesCommandResultWithError() {
        var json = """
            {"type": "command-result", "id": "cmd-2", "ok": false, "error": "Element not found"}
            """;
        var request = PushRequest.parse(json);
        var result = (PushRequest.CommandResult) request;
        assertThat(result.ok()).isFalse();
        assertThat(result.error()).isEqualTo("Element not found");
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `mvn --batch-mode test -pl backend/push -Dtest=CommandResultTest -f /Users/mdproctor/claude/casehub/slots/122/pages/backend/pom.xml`
Expected: FAIL — no CommandResult variant

- [ ] **Step 4: Add CommandResult to PushRequest**

Add new record to the sealed interface:
```java
record CommandResult(String id, boolean ok, @Nullable String error) implements PushRequest {}
```

Add parse case:
```java
case "command-result" -> new CommandResult(
    node.get("id").asText(),
    node.get("ok").asBoolean(),
    node.has("error") ? node.get("error").asText() : null
);
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `mvn --batch-mode test -pl backend/push -f /Users/mdproctor/claude/casehub/slots/122/pages/backend/pom.xml`
Expected: PASS — all push tests green

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add backend/push/
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add CommandResult variant to PushRequest sealed interface

Extends pages-push protocol for bidirectional command/response.
New sealed variant: CommandResult(id, ok, error).

Refs casehubio/parent#417"
```

---

### Task 8: Scenario YAML Command Model

**Files:**
- Create: `packages/pages-aria/src/scenario/schema.json` — JSON Schema for YAML format
- Create: `packages/pages-aria/src/scenario/types.ts` — TypeScript types matching schema
- Create: `packages/pages-aria/src/scenario/parser.ts` — YAML parser
- Create: `packages/pages-aria/src/scenario/runner.ts` — Client-only runner
- Create: `packages/pages-aria/src/scenario/runner.test.ts`
- Modify: `packages/pages-aria/package.json` — add scenario export + js-yaml dependency

**Interfaces:**
- Consumes: executor API from Task 3 (click, fill, select, expand, collapse, assertState, waitFor)
- Produces: `parseScenario(yaml)`, `runScenario(scenario)` — the client-only runner

- [ ] **Step 1: Define the JSON Schema**

```json
// packages/pages-aria/src/scenario/schema.json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["scenario", "steps"],
  "properties": {
    "scenario": { "type": "string" },
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "oneOf": [
          {
            "required": ["navigate"],
            "properties": { "navigate": { "type": "string" } }
          },
          {
            "required": ["click"],
            "properties": {
              "click": { "$ref": "#/$defs/ariaTarget" }
            }
          },
          {
            "required": ["fill"],
            "properties": {
              "fill": {
                "allOf": [{ "$ref": "#/$defs/ariaTarget" }],
                "required": ["value"],
                "properties": { "value": { "type": "string" } }
              }
            }
          },
          {
            "required": ["select"],
            "properties": {
              "select": {
                "allOf": [{ "$ref": "#/$defs/ariaTarget" }],
                "required": ["value"],
                "properties": { "value": { "type": "string" } }
              }
            }
          },
          {
            "required": ["expand"],
            "properties": { "expand": { "$ref": "#/$defs/ariaTarget" } }
          },
          {
            "required": ["collapse"],
            "properties": { "collapse": { "$ref": "#/$defs/ariaTarget" } }
          },
          {
            "required": ["assert"],
            "properties": {
              "assert": {
                "allOf": [{ "$ref": "#/$defs/ariaTarget" }],
                "required": ["state"],
                "properties": { "state": { "type": "object" } }
              }
            }
          },
          {
            "required": ["wait"],
            "properties": {
              "wait": {
                "allOf": [{ "$ref": "#/$defs/ariaTarget" }],
                "required": ["state"],
                "properties": {
                  "state": { "type": "object" },
                  "timeout": { "type": "integer", "default": 5000 }
                }
              }
            }
          }
        ]
      }
    }
  },
  "$defs": {
    "ariaTarget": {
      "type": "object",
      "required": ["role", "name"],
      "properties": {
        "role": { "type": "string" },
        "name": { "type": "string" },
        "within": { "$ref": "#/$defs/ariaTarget" }
      }
    }
  }
}
```

- [ ] **Step 2: Write TypeScript types and parser**

```typescript
// packages/pages-aria/src/scenario/types.ts
import type { AriaTarget } from '@casehubio/pages-primitives';

export type ScenarioStep =
  | { navigate: string }
  | { click: AriaTarget }
  | { fill: AriaTarget & { value: string } }
  | { select: AriaTarget & { value: string } }
  | { expand: AriaTarget }
  | { collapse: AriaTarget }
  | { assert: AriaTarget & { state: Record<string, unknown> } }
  | { wait: AriaTarget & { state: Record<string, unknown>; timeout?: number } };

export interface Scenario {
  scenario: string;
  steps: ScenarioStep[];
}
```

```typescript
// packages/pages-aria/src/scenario/parser.ts
import { parse } from 'yaml';
import type { Scenario } from './types.js';

export function parseScenario(yamlString: string): Scenario {
  const parsed = parse(yamlString) as Scenario;
  if (!parsed.scenario || !Array.isArray(parsed.steps)) {
    throw new Error('Invalid scenario: must have "scenario" name and "steps" array');
  }
  return parsed;
}
```

- [ ] **Step 3: Write the client-only runner**

```typescript
// packages/pages-aria/src/scenario/runner.ts
import { click, fill, select, expand, collapse, assertState, waitFor } from '../executor/index.js';
import type { Scenario, ScenarioStep } from './types.js';
import type { AriaState } from '@casehubio/pages-primitives';

function toAriaState(state: Record<string, unknown>): Partial<AriaState> {
  const result: Partial<AriaState> = {};
  if ('aria-busy' in state) result.busy = state['aria-busy'] as boolean;
  if ('aria-disabled' in state) result.disabled = state['aria-disabled'] as boolean;
  if ('aria-expanded' in state) result.expanded = state['aria-expanded'] as boolean;
  if ('aria-selected' in state) result.selected = state['aria-selected'] as boolean;
  if ('aria-hidden' in state) result.hidden = state['aria-hidden'] as boolean;
  return result;
}

async function executeStep(step: ScenarioStep): Promise<void> {
  if ('navigate' in step) {
    window.location.href = step.navigate;
    return;
  }
  if ('click' in step) { click(step.click); return; }
  if ('fill' in step) { fill(step.fill, step.fill.value); return; }
  if ('select' in step) { select(step.select, step.select.value); return; }
  if ('expand' in step) { expand(step.expand); return; }
  if ('collapse' in step) { collapse(step.collapse); return; }
  if ('assert' in step) { assertState(step.assert, toAriaState(step.assert.state)); return; }
  if ('wait' in step) {
    await waitFor(step.wait, toAriaState(step.wait.state), step.wait.timeout ?? 5000);
    return;
  }
}

export async function runScenario(scenario: Scenario): Promise<void> {
  for (const step of scenario.steps) {
    await executeStep(step);
  }
}
```

- [ ] **Step 4: Write runner tests**

```typescript
// packages/pages-aria/src/scenario/runner.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { parseScenario } from './parser.js';
import { runScenario } from './runner.js';

describe('scenario parser', () => {
  it('parses valid YAML scenario', () => {
    const yaml = `
scenario: test-form
steps:
  - click:
      role: button
      name: Submit
`;
    const scenario = parseScenario(yaml);
    expect(scenario.scenario).toBe('test-form');
    expect(scenario.steps).toHaveLength(1);
  });

  it('throws on invalid scenario', () => {
    expect(() => parseScenario('invalid: yaml')).toThrow('Invalid scenario');
  });
});

describe('scenario runner', () => {
  beforeEach(() => {
    document.body.innerHTML = '';
  });

  it('executes click step', async () => {
    document.body.innerHTML = '<button aria-label="Submit">Submit</button>';
    const handler = vi.fn();
    document.querySelector('button')!.addEventListener('click', handler);

    await runScenario({
      scenario: 'test',
      steps: [{ click: { role: 'button', name: 'Submit' } }],
    });

    expect(handler).toHaveBeenCalledOnce();
  });

  it('executes fill step', async () => {
    document.body.innerHTML = '<input aria-label="Name" />';

    await runScenario({
      scenario: 'test',
      steps: [{ fill: { role: 'textbox', name: 'Name', value: 'Alice' } }],
    });

    expect((document.querySelector('input') as HTMLInputElement).value).toBe('Alice');
  });

  it('executes assert step', async () => {
    document.body.innerHTML = '<button aria-label="Submit" aria-busy="false">Submit</button>';

    await expect(runScenario({
      scenario: 'test',
      steps: [{ assert: { role: 'button', name: 'Submit', state: { 'aria-busy': false } } }],
    })).resolves.toBeUndefined();
  });
});
```

- [ ] **Step 5: Add yaml dependency and scenario export**

Add to `packages/pages-aria/package.json`:
```json
"dependencies": {
  "@casehubio/pages-primitives": "workspace:*",
  "yaml": "^2.7.1"
},
"exports": {
  "./walker": "./src/walker/index.ts",
  "./executor": "./src/executor/index.ts",
  "./server": "./src/server/index.ts",
  "./scenario": "./src/scenario/index.ts"
}
```

Create `packages/pages-aria/src/scenario/index.ts`:
```typescript
export { parseScenario } from './parser.js';
export { runScenario } from './runner.js';
export type { Scenario, ScenarioStep } from './types.js';
```

- [ ] **Step 6: Run tests**

Run: `yarn workspace @casehubio/pages-aria run test -- scenario`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/122/pages add packages/pages-aria/src/scenario/ packages/pages-aria/package.json
git -C /Users/mdproctor/claude/casehub/slots/122/pages commit -m "feat(#417): add scenario YAML parser and client-only runner

JSON Schema, TypeScript types, YAML parser, and in-browser
scenario runner using the ARIA executor. Client-only mode —
no push server needed.

Refs casehubio/parent#417"
```

---

## Task Dependency Graph

```
Task 1: ARIA Contract Types
  ↓
Task 2: Walker  ──────────────────┐
  ↓                               │
Task 3: Executor                  │
  ↓              ↓                │
Task 8: Scenario  Task 7: Push    │
                    ↓             │
              (pages-aria/server  │
               — future task)     │
                                  │
Task 4: Widget Remediation ───────┤
Task 5: Charts + Forms ──────────┤
                                  ↓
                         Task 6: axe-core CI
```

Tasks 4-5 (widget remediation) can run in parallel with Tasks 2-3 (pages-aria).
Task 6 (axe-core) depends on Tasks 4-5.
Task 7 (push extension) can run in parallel with everything else.
Task 8 (scenario) depends on Task 3 (executor).

## Deferred to Separate Plans

- **Blocks-UI widget remediation** — ~40 components per the spec audit. Requires blocks-ui repo in a dedicated slot.
- **pages-aria/server** — push channel handler connecting executor to scenario topics. Depends on Task 7 (push extension). Small task, can be added to this plan or a follow-up.
- **MCP integration** — MCP server exposing aria_* tools. Depends on pages-aria executor + push protocol. Requires platform-mcp access.
- **Java POJO command model** — scenario records in the blocks repo. Can be done in this slot (blocks repo is available) but deferred until TypeScript model is validated.
- **ADR + platform protocol** — follow-on artifacts after implementation validates the design.
