# Runtime State Badges Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #109 — Epic: Runtime state expansion — case, work, group, and node status badges
**Issue group:** #109

**Goal:** A single generic `<status-badge>` component backed by a status registry, producing both pill rendering and graph node decorations, with retrofit of existing ad-hoc badge implementations.

**Architecture:** Status registry maps `(domain, state)` → `StatusDescriptor` in blocks-ui-core. A `<status-badge>` web component renders pills. A separate `toDecoration()` in graph-stencil-case converts descriptors to `NodeDecoration` for the diagram overlay. Existing components (work-item-inbox, session-list, commitment-state-pill, badge-mappings) retrofit to the shared infrastructure.

**Tech Stack:** Lit 3 web components, TypeScript, Vitest, `@casehubio/graph-core` (NodeDecoration type)

## Global Constraints

- All render callback output uses inline styles (protocol PP-20260713-8ea1af)
- Pill colours use scale-3 CSS custom properties via `stateCategoryStyles()`
- Graph badge colours use raw hex via `BADGE_COLORS` (scale-9, vibrant)
- `domain` is open `string` — no union type
- Static Map initialization — no side-effect imports

---

### Task 1: Status types and registry

**Files:**
- Create: `packages/blocks-ui-core/src/types/status.ts`
- Create: `packages/blocks-ui-core/src/types/status.test.ts`
- Modify: `packages/blocks-ui-core/src/types/commitment.ts:7-8` (import StateCategory from status.ts)
- Modify: `packages/blocks-ui-core/src/types/index.ts` (add status export)

**Interfaces:**
- Produces: `StateCategory` type, `StatusDescriptor` interface, `lookupStatus(domain, state)`, `registerStatus(domain, state, descriptor)`, `FALLBACK_DESCRIPTOR`

- [ ] **Step 1: Write the failing tests**

```typescript
// packages/blocks-ui-core/src/types/status.test.ts
import { describe, it, expect } from 'vitest';
import { lookupStatus, registerStatus } from './status.js';

describe('lookupStatus', () => {
  it('returns exact domain:state match', () => {
    const d = lookupStatus('case', 'STARTING');
    expect(d.category).toBe('info');
    expect(d.icon).toBe('◐');
  });

  it('falls back to cross-domain default when domain provided', () => {
    const d = lookupStatus('case', 'COMPLETED');
    expect(d.category).toBe('success');
    expect(d.icon).toBe('✓');
  });

  it('uses cross-domain default when domain is undefined', () => {
    const d = lookupStatus(undefined, 'COMPLETED');
    expect(d.category).toBe('success');
  });

  it('returns fallback for unknown state', () => {
    const d = lookupStatus('case', 'NONEXISTENT');
    expect(d.category).toBe('neutral');
    expect(d.icon).toBe('?');
  });

  it('does not scan per-domain entries when domain is undefined', () => {
    const d = lookupStatus(undefined, 'STARTING');
    expect(d.category).toBe('neutral');
    expect(d.icon).toBe('?');
  });

  describe('all built-in domains resolve', () => {
    const domains: Array<[string, string[]]> = [
      ['case', ['STARTING', 'RUNNING', 'WAITING', 'SUSPENDED', 'COMPLETED', 'FAULTED', 'CANCELLED']],
      ['task', ['PENDING', 'RUNNING', 'DELEGATED', 'SUSPENDED', 'COMPLETED', 'FAULTED', 'REJECTED', 'OBSOLETE', 'CANCELLED']],
      ['workitem', ['PENDING', 'ASSIGNED', 'IN_PROGRESS', 'COMPLETED', 'REJECTED', 'FAULTED', 'DELEGATED', 'SUSPENDED', 'CANCELLED', 'EXPIRED', 'ESCALATED', 'OBSOLETE']],
      ['milestone', ['PENDING', 'ACTIVE', 'COMPLETED']],
      ['outcome', ['SUCCESS', 'DECLINED', 'FAILED', 'EXPIRED', 'ESCALATED', 'COMPLETED']],
      ['group', ['IN_PROGRESS', 'COMPLETED', 'REJECTED']],
      ['sla', ['NOT_STARTED', 'ON_TRACK', 'BREACHED']],
      ['node', ['PENDING', 'DISPATCHED', 'COMPLETED', 'FAILED', 'SKIPPED', 'CANCELLED']],
      ['session', ['ACTIVE', 'WAITING', 'IDLE']],
      ['commitment', ['OPEN', 'ACKNOWLEDGED', 'FULFILLED', 'FAILED', 'DECLINED', 'DELEGATED', 'EXPIRED']],
    ];

    for (const [domain, states] of domains) {
      for (const state of states) {
        it(`${domain}:${state} resolves to a non-fallback descriptor`, () => {
          const d = lookupStatus(domain, state);
          expect(d.icon).not.toBe('?');
        });
      }
    }
  });
});

describe('registerStatus', () => {
  it('overrides existing registration', () => {
    registerStatus('_test', 'FOO', { category: 'danger', icon: 'X' });
    expect(lookupStatus('_test', 'FOO').category).toBe('danger');
    registerStatus('_test', 'FOO', { category: 'success', icon: 'Y' });
    expect(lookupStatus('_test', 'FOO').category).toBe('success');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/blocks-ui-core/src/types/status.test.ts`
Expected: FAIL — module `./status.js` not found

- [ ] **Step 3: Create `types/status.ts` with registry and all registrations**

```typescript
// packages/blocks-ui-core/src/types/status.ts

export type StateCategory = 'active' | 'info' | 'success' | 'danger'
  | 'neutral' | 'transfer' | 'warning';

export interface StatusDescriptor {
  readonly category: StateCategory;
  readonly icon: string;
  readonly label?: string;
  readonly pulse?: boolean;
  readonly border?: boolean;
}

export const FALLBACK_DESCRIPTOR: StatusDescriptor = { category: 'neutral', icon: '?' };

const REGISTRY = new Map<string, StatusDescriptor>([
  // Cross-domain defaults
  ['*:PENDING',    { category: 'neutral', icon: '○' }],
  ['*:RUNNING',    { category: 'success', icon: '▶', pulse: true, border: true }],
  ['*:COMPLETED',  { category: 'success', icon: '✓' }],
  ['*:FAULTED',    { category: 'danger',  icon: '!' }],
  ['*:CANCELLED',  { category: 'neutral', icon: '/' }],
  ['*:SUSPENDED',  { category: 'warning', icon: '⏸', border: true }],

  // case:
  ['case:STARTING', { category: 'info',    icon: '◐' }],
  ['case:WAITING',  { category: 'warning', icon: '⏳' }],

  // task:
  ['task:DELEGATED', { category: 'info',    icon: '→', border: true }],
  ['task:REJECTED',  { category: 'warning', icon: '✕' }],
  ['task:OBSOLETE',  { category: 'neutral', icon: '—' }],

  // work: (engine WorkStatus — orchestration views)
  ['work:DECLINED', { category: 'neutral', icon: '🚫' }],
  ['work:FAILED',   { category: 'danger',  icon: '✗' }],
  ['work:EXPIRED',  { category: 'warning', icon: '⌛' }],

  // workitem: (UI WorkItemStatus — work-item-inbox)
  ['workitem:ASSIGNED',    { category: 'info',    icon: '●' }],
  ['workitem:IN_PROGRESS', { category: 'active',  icon: '◐', border: true }],
  ['workitem:DELEGATED',   { category: 'info',    icon: '→', border: true }],
  ['workitem:REJECTED',    { category: 'warning', icon: '✕' }],
  ['workitem:ESCALATED',   { category: 'warning', icon: '↑' }],
  ['workitem:OBSOLETE',    { category: 'neutral', icon: '—' }],
  ['workitem:EXPIRED',     { category: 'warning', icon: '⌛' }],

  // milestone:
  ['milestone:ACTIVE', { category: 'info', icon: '◉', pulse: true }],

  // outcome:
  ['outcome:SUCCESS',   { category: 'success', icon: '✓' }],
  ['outcome:DECLINED',  { category: 'neutral', icon: '🚫' }],
  ['outcome:FAILED',    { category: 'danger',  icon: '✗' }],
  ['outcome:EXPIRED',   { category: 'warning', icon: '⌛' }],
  ['outcome:ESCALATED', { category: 'warning', icon: '↑' }],
  ['outcome:COMPLETED', { category: 'success', icon: '✓' }],

  // group:
  ['group:IN_PROGRESS', { category: 'active',  icon: '◐' }],
  ['group:COMPLETED',   { category: 'success', icon: '✓' }],
  ['group:REJECTED',    { category: 'danger',  icon: '✕' }],

  // sla:
  ['sla:NOT_STARTED', { category: 'neutral', icon: '○' }],
  ['sla:ON_TRACK',    { category: 'success', icon: '✓' }],
  ['sla:BREACHED',    { category: 'danger',  icon: '!', pulse: true }],

  // node:
  ['node:DISPATCHED', { category: 'info',    icon: '→' }],
  ['node:SKIPPED',    { category: 'neutral', icon: '⏭' }],
  ['node:FAILED',     { category: 'danger',  icon: '✗' }],

  // session:
  ['session:ACTIVE',  { category: 'success', icon: '▶' }],
  ['session:WAITING', { category: 'warning', icon: '⏳' }],
  ['session:IDLE',    { category: 'neutral', icon: '○' }],

  // commitment:
  ['commitment:OPEN',         { category: 'active',   icon: '⏳' }],
  ['commitment:ACKNOWLEDGED', { category: 'info',     icon: '📋' }],
  ['commitment:FULFILLED',    { category: 'success',  icon: '✓' }],
  ['commitment:FAILED',       { category: 'danger',   icon: '✗' }],
  ['commitment:DECLINED',     { category: 'neutral',  icon: '🚫' }],
  ['commitment:DELEGATED',    { category: 'transfer', icon: '↳' }],
  ['commitment:EXPIRED',      { category: 'warning',  icon: '⌛' }],
]);

export function registerStatus(domain: string, state: string, descriptor: StatusDescriptor): void {
  REGISTRY.set(`${domain}:${state}`, descriptor);
}

export function lookupStatus(domain: string | undefined, state: string): StatusDescriptor {
  if (domain) {
    const exact = REGISTRY.get(`${domain}:${state}`);
    if (exact) return exact;
  }
  return REGISTRY.get(`*:${state}`) ?? FALLBACK_DESCRIPTOR;
}
```

- [ ] **Step 4: Update `types/commitment.ts` — import StateCategory from status.ts**

Remove the `StateCategory` definition at lines 7-8. Import from status.ts:

```typescript
// At top of commitment.ts
import type { StateCategory } from './status.js';
export type { StateCategory };
```

The `commitmentStateCategory()` function stays — it's backward-compatible.

- [ ] **Step 5: Update `types/index.ts` — export status module**

Add to `packages/blocks-ui-core/src/types/index.ts`:

```typescript
export * from './status.js';
```

- [ ] **Step 6: Run tests**

Run: `yarn vitest run packages/blocks-ui-core/src/types/status.test.ts`
Expected: PASS

Run: `yarn typecheck`
Expected: PASS (StateCategory import chain intact)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/types/status.ts packages/blocks-ui-core/src/types/status.test.ts packages/blocks-ui-core/src/types/commitment.ts packages/blocks-ui-core/src/types/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#109): status registry with cross-domain defaults and 10 domain registrations"
```

---

### Task 2: Move `stateCategoryStyles()` to `styles/category.ts`

**Files:**
- Create: `packages/blocks-ui-core/src/styles/category.ts`
- Modify: `packages/blocks-ui-core/src/styles/index.ts` (add export)
- Modify: `packages/blocks-ui-core/src/commitment-pill/styles.ts` (re-import from new location)
- Modify: `packages/blocks-ui-core/src/commitment-pill/index.ts` (update re-export)

**Interfaces:**
- Consumes: `StateCategory` from `types/status.ts`
- Produces: `stateCategoryStyles(category): CategoryStyle`, `CategoryStyle` interface, `CATEGORY_STYLES` record

- [ ] **Step 1: Create `styles/category.ts` — move content from `commitment-pill/styles.ts`**

```typescript
// packages/blocks-ui-core/src/styles/category.ts
import type { StateCategory } from '../types/status.js';

export interface CategoryStyle {
  readonly background: string;
  readonly color: string;
}

const CATEGORY_STYLES: Record<StateCategory, CategoryStyle> = {
  active:   { background: 'var(--pages-accent-3, #e0e7ff)',  color: 'var(--pages-accent-11, #3730a3)' },
  info:     { background: 'var(--pages-info-3, #dbeafe)',    color: 'var(--pages-info-11, #1e40af)' },
  success:  { background: 'var(--pages-success-3, #d1fae5)', color: 'var(--pages-success-11, #065f46)' },
  danger:   { background: 'var(--pages-danger-3, #fee2e2)',  color: 'var(--pages-danger-11, #991b1b)' },
  neutral:  { background: 'var(--pages-neutral-3, #e5e5e5)', color: 'var(--pages-neutral-9, #737373)' },
  transfer: { background: 'var(--pages-info-3, #dbeafe)',    color: 'var(--pages-info-11, #1e40af)' },
  warning:  { background: 'var(--pages-warning-3, #fef3c7)', color: 'var(--pages-warning-11, #92400e)' },
};

export function stateCategoryStyles(category: StateCategory): CategoryStyle {
  return CATEGORY_STYLES[category];
}
```

- [ ] **Step 2: Update `styles/index.ts`**

```typescript
export { pulseAnimation } from './animations.js';
export { stateCategoryStyles, type CategoryStyle } from './category.js';
```

- [ ] **Step 3: Update `commitment-pill/styles.ts` — delegate to styles/category.ts**

Replace the entire file:

```typescript
// packages/blocks-ui-core/src/commitment-pill/styles.ts
export { stateCategoryStyles, type CategoryStyle } from '../styles/category.js';
```

- [ ] **Step 4: Update `commitment-pill/index.ts` — keep existing exports**

No change needed — it already exports from `./styles.js` which now delegates.

- [ ] **Step 5: Run tests and typecheck**

Run: `yarn test`
Expected: PASS (all existing tests still pass — same exports, same values)

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/styles/category.ts packages/blocks-ui-core/src/styles/index.ts packages/blocks-ui-core/src/commitment-pill/styles.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#109): move stateCategoryStyles to styles/category.ts — cross-cutting concern"
```

---

### Task 3: `<status-badge>` component

**Files:**
- Create: `packages/blocks-ui-core/src/status-badge/status-badge.ts`
- Create: `packages/blocks-ui-core/src/status-badge/status-badge.test.ts`
- Create: `packages/blocks-ui-core/src/status-badge/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts` (add export)

**Interfaces:**
- Consumes: `lookupStatus()` from `types/status.ts`, `stateCategoryStyles()` from `styles/category.ts`
- Produces: `<status-badge>` custom element with `state`, `domain`, `size`, `showIcon` properties

- [ ] **Step 1: Write failing tests**

```typescript
// packages/blocks-ui-core/src/status-badge/status-badge.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './status-badge.js';

type BadgeEl = HTMLElement & {
  state?: string;
  domain?: string;
  size: 'sm' | 'md';
  showIcon: boolean;
  updateComplete: Promise<boolean>;
};

describe('status-badge', () => {
  let el: BadgeEl;

  beforeEach(async () => {
    el = document.createElement('status-badge') as BadgeEl;
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => { el.remove(); });

  it('renders nothing when no state set', () => {
    expect(el.shadowRoot!.querySelector('.pill')).toBeNull();
  });

  it('renders nothing for empty string state', async () => {
    el.state = '';
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.pill')).toBeNull();
  });

  it('renders the state label', async () => {
    el.state = 'COMPLETED';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('COMPLETED');
  });

  it('renders domain-specific descriptor', async () => {
    el.domain = 'case';
    el.state = 'STARTING';
    await el.updateComplete;
    const pill = el.shadowRoot!.querySelector('.pill') as HTMLElement;
    expect(pill).toBeTruthy();
    expect(pill.style.background).toContain('--pages-info-3');
  });

  it('falls back to cross-domain default', async () => {
    el.state = 'COMPLETED';
    await el.updateComplete;
    const pill = el.shadowRoot!.querySelector('.pill') as HTMLElement;
    expect(pill.style.background).toContain('--pages-success-3');
  });

  it('shows icon when showIcon is true', async () => {
    el.state = 'RUNNING';
    el.showIcon = true;
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.icon')).toBeTruthy();
  });

  it('hides icon by default', async () => {
    el.state = 'RUNNING';
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.icon')).toBeNull();
  });

  it('applies md size styles', async () => {
    el.state = 'RUNNING';
    el.size = 'md';
    await el.updateComplete;
    const pill = el.shadowRoot!.querySelector('.pill') as HTMLElement;
    expect(pill.style.fontSize).toBe('12px');
  });

  it('defaults to sm size', () => {
    expect(el.size).toBe('sm');
  });

  it('has aria-label with state', async () => {
    el.domain = 'task';
    el.state = 'DELEGATED';
    await el.updateComplete;
    const pill = el.shadowRoot!.querySelector('.pill');
    expect(pill?.getAttribute('aria-label')).toContain('DELEGATED');
  });

  it('uses label override from descriptor', async () => {
    // Register a custom domain with a label
    const { registerStatus } = await import('../types/status.js');
    registerStatus('_testlabel', 'X', { category: 'info', icon: '!', label: 'Custom' });
    el.domain = '_testlabel';
    el.state = 'X';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Custom');
  });

  it('renders fallback for unknown domain+state', async () => {
    el.domain = 'nonexistent';
    el.state = 'UNKNOWN';
    await el.updateComplete;
    const pill = el.shadowRoot!.querySelector('.pill') as HTMLElement;
    expect(pill).toBeTruthy();
    expect(pill.style.background).toContain('--pages-neutral-3');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/blocks-ui-core/src/status-badge/status-badge.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement `status-badge.ts`**

```typescript
// packages/blocks-ui-core/src/status-badge/status-badge.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import { styleMap } from 'lit/directives/style-map.js';
import { lookupStatus } from '../types/status.js';
import { stateCategoryStyles } from '../styles/category.js';

@customElement('status-badge')
export class StatusBadge extends LitElement {
  @property({ type: String }) state?: string;
  @property({ type: String }) domain?: string;
  @property({ type: String }) size: 'sm' | 'md' = 'sm';
  @property({ type: Boolean }) showIcon = false;

  static override styles = css`
    :host { display: inline-block; }
  `;

  override render() {
    if (!this.state) return nothing;
    const descriptor = lookupStatus(this.domain, this.state);
    const colors = stateCategoryStyles(descriptor.category);
    const fontSize = this.size === 'md' ? '12px' : '10px';
    const padding = this.size === 'md' ? '2px 8px' : '1px 6px';
    const displayLabel = descriptor.label ?? this.state;

    const styles = {
      display: 'inline-flex',
      alignItems: 'center',
      gap: '3px',
      fontSize,
      fontWeight: '500',
      padding,
      borderRadius: '9999px',
      textTransform: 'uppercase',
      letterSpacing: '0.5px',
      lineHeight: '1.4',
      background: colors.background,
      color: colors.color,
    };

    return html`
      <span class="pill" style=${styleMap(styles)} aria-label="Status: ${displayLabel}">
        ${this.showIcon ? html`<span class="icon">${descriptor.icon}</span>` : nothing}
        ${displayLabel}
      </span>
    `;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'status-badge': StatusBadge;
  }
}
```

- [ ] **Step 4: Create `status-badge/index.ts`**

```typescript
export { StatusBadge } from './status-badge.js';
```

- [ ] **Step 5: Add export to `blocks-ui-core/src/index.ts`**

Add: `export * from './status-badge/index.js';`

- [ ] **Step 6: Run tests**

Run: `yarn vitest run packages/blocks-ui-core/src/status-badge/status-badge.test.ts`
Expected: PASS

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/status-badge/ packages/blocks-ui-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#109): status-badge web component — generic pill renderer for all domains"
```

---

### Task 4: `toDecoration()` in graph-stencil-case

**Files:**
- Create: `packages/graph-stencil-case/src/runtime/decoration.ts`
- Create: `packages/graph-stencil-case/src/runtime/decoration.test.ts`
- Modify: `packages/graph-stencil-case/src/runtime/badge-mappings.ts:3-29` (remove TASK/MILESTONE/UNKNOWN decorations, keep TERMINAL_SEVERITY, ACTIVE_WORST_PRIORITY, isActiveStatus)
- Modify: `packages/graph-stencil-case/src/runtime/runtime-adapter.ts:1-10,30-31,72` (import toDecoration instead of static maps)
- Modify: `packages/graph-stencil-case/src/runtime/badge-mappings.test.ts` (update tests)

**Interfaces:**
- Consumes: `lookupStatus()` from `@casehubio/blocks-ui-core`, `NodeDecoration` from `@casehubio/graph-core`
- Produces: `toDecoration(domain, state): NodeDecoration`, `BADGE_COLORS: Record<StateCategory, string>`

- [ ] **Step 1: Write failing tests**

```typescript
// packages/graph-stencil-case/src/runtime/decoration.test.ts
import { describe, it, expect } from 'vitest';
import { toDecoration, BADGE_COLORS } from './decoration.js';

describe('toDecoration', () => {
  it('produces badge with icon and colour for task:RUNNING', () => {
    const d = toDecoration('task', 'RUNNING');
    expect(d.badge).toBeDefined();
    expect(d.badge!.icon).toBe('▶');
    expect(d.badge!.color).toBe(BADGE_COLORS.success);
    expect(d.badge!.pulse).toBe(true);
  });

  it('produces border for border-flagged states', () => {
    const d = toDecoration('task', 'RUNNING');
    expect(d.border).toBeDefined();
    expect(d.border!.style).toBe('solid');
    expect(d.border!.color).toBe(BADGE_COLORS.success);
  });

  it('produces no border for non-border states', () => {
    const d = toDecoration('task', 'PENDING');
    expect(d.border).toBeUndefined();
  });

  it('produces no border for terminal states', () => {
    const d = toDecoration('task', 'COMPLETED');
    expect(d.border).toBeUndefined();
  });

  it('maps milestone:ACTIVE with pulse', () => {
    const d = toDecoration('milestone', 'ACTIVE');
    expect(d.badge!.pulse).toBe(true);
    expect(d.badge!.color).toBe(BADGE_COLORS.info);
  });

  it('maps milestone:PENDING without pulse', () => {
    const d = toDecoration('milestone', 'PENDING');
    expect(d.badge!.pulse).toBeFalsy();
  });

  it('returns fallback for unknown state', () => {
    const d = toDecoration('task', 'NONEXISTENT');
    expect(d.badge!.icon).toBe('?');
    expect(d.badge!.color).toBe(BADGE_COLORS.neutral);
  });

  it('task:DELEGATED gets border', () => {
    const d = toDecoration('task', 'DELEGATED');
    expect(d.border).toBeDefined();
    expect(d.badge!.icon).toBe('→');
  });

  it('task:SUSPENDED gets border', () => {
    const d = toDecoration('task', 'SUSPENDED');
    expect(d.border).toBeDefined();
  });
});

describe('BADGE_COLORS', () => {
  it('maps all StateCategory values', () => {
    for (const cat of ['active', 'info', 'success', 'danger', 'neutral', 'transfer', 'warning']) {
      expect(BADGE_COLORS[cat as keyof typeof BADGE_COLORS]).toBeTruthy();
    }
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/decoration.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement `decoration.ts`**

```typescript
// packages/graph-stencil-case/src/runtime/decoration.ts
import type { NodeDecoration } from '@casehubio/graph-core';
import { lookupStatus, type StateCategory } from '@casehubio/blocks-ui-core';

export const BADGE_COLORS: Record<StateCategory, string> = {
  active:   '#6366f1',
  info:     '#3b82f6',
  success:  '#22c55e',
  danger:   '#ef4444',
  neutral:  '#9ca3af',
  transfer: '#3b82f6',
  warning:  '#eab308',
};

export function toDecoration(domain: string, state: string): NodeDecoration {
  const descriptor = lookupStatus(domain, state);
  const color = BADGE_COLORS[descriptor.category];

  const decoration: NodeDecoration = {
    badge: {
      icon: descriptor.icon,
      color,
      ...(descriptor.pulse ? { pulse: true } : {}),
    },
  };

  if (descriptor.border) {
    decoration.border = { style: 'solid', color };
  }

  return decoration;
}
```

- [ ] **Step 4: Update `badge-mappings.ts` — remove decoration maps, keep aggregation logic**

Remove `TASK_STATUS_DECORATIONS`, `MILESTONE_STATUS_DECORATIONS`, `UNKNOWN_DECORATION` (lines 9-29). Keep `ACTIVE_STATES`, `isActiveStatus`, `TERMINAL_SEVERITY`, `ACTIVE_WORST_PRIORITY`.

```typescript
// packages/graph-stencil-case/src/runtime/badge-mappings.ts
const ACTIVE_STATES = new Set(['PENDING', 'RUNNING', 'DELEGATED', 'SUSPENDED']);

export function isActiveStatus(status: string): boolean {
  return ACTIVE_STATES.has(status);
}

export const TERMINAL_SEVERITY: Record<string, number> = {
  COMPLETED: 1,
  OBSOLETE: 2,
  CANCELLED: 3,
  REJECTED: 4,
  FAULTED: 5,
};

export const ACTIVE_WORST_PRIORITY: Record<string, number> = {
  PENDING: 1,
  RUNNING: 2,
  DELEGATED: 3,
  SUSPENDED: 4,
};
```

- [ ] **Step 5: Update `runtime-adapter.ts` — use `toDecoration()`**

Replace imports and usage. The aggregation logic (`aggregateBinding`) calls `toDecoration('task', statusKey)` instead of indexing `TASK_STATUS_DECORATIONS`. Milestones call `toDecoration('milestone', status)`.

```typescript
// packages/graph-stencil-case/src/runtime/runtime-adapter.ts
import type { NodeDecoration } from '@casehubio/graph-core';
import type { CaseRuntimeState, PlanItemSnapshot } from './types.js';
import { toDecoration } from './decoration.js';
import {
  TERMINAL_SEVERITY,
  ACTIVE_WORST_PRIORITY,
  isActiveStatus,
} from './badge-mappings.js';

function aggregateBinding(items: readonly PlanItemSnapshot[]): NodeDecoration {
  const activeItems = items.filter(i => isActiveStatus(i.status));
  const count = items.length > 1 ? items.length : undefined;

  let statusKey: string;

  if (activeItems.length > 0) {
    statusKey = activeItems.reduce((worst, item) =>
      (ACTIVE_WORST_PRIORITY[item.status] ?? 0) > (ACTIVE_WORST_PRIORITY[worst.status] ?? 0) ? item : worst,
    ).status;
  } else {
    const sorted = [...items].sort((a, b) => {
      const timeDiff = b.createdAt.localeCompare(a.createdAt);
      if (timeDiff !== 0) return timeDiff;
      return (TERMINAL_SEVERITY[b.status] ?? 0) - (TERMINAL_SEVERITY[a.status] ?? 0);
    });
    statusKey = sorted[0]!.status;
  }

  const base = toDecoration('task', statusKey);
  const tooltip = buildTooltip(items);

  return {
    ...base,
    badge: count !== undefined ? { ...base.badge!, count } : { ...base.badge! },
    tooltip,
  };
}

function buildTooltip(items: readonly PlanItemSnapshot[]): string {
  if (items.length === 1) {
    return items[0]!.status.toLowerCase();
  }
  const counts = new Map<string, number>();
  for (const item of items) {
    const key = item.status.toLowerCase();
    counts.set(key, (counts.get(key) ?? 0) + 1);
  }
  const parts = Array.from(counts.entries()).map(([status, n]) => `${n} ${status}`);
  return `${items.length} plan items: ${parts.join(', ')}`;
}

export function toDecorations(state: CaseRuntimeState): ReadonlyMap<string, NodeDecoration> {
  const decorations = new Map<string, NodeDecoration>();

  const byBinding = new Map<string, PlanItemSnapshot[]>();
  for (const item of state.planItems) {
    const list = byBinding.get(item.bindingName);
    if (list) {
      list.push(item);
    } else {
      byBinding.set(item.bindingName, [item]);
    }
  }

  for (const [bindingName, items] of byBinding) {
    decorations.set(`binding:${bindingName}`, aggregateBinding(items));
  }

  for (const milestone of state.milestones) {
    const base = toDecoration('milestone', milestone.status);
    decorations.set(`milestone:${milestone.name}`, { ...base, tooltip: milestone.status.toLowerCase() });
  }

  return decorations;
}
```

- [ ] **Step 6: Update `badge-mappings.test.ts` — remove decoration map tests, keep aggregation tests**

Remove `TASK_STATUS_DECORATIONS`, `MILESTONE_STATUS_DECORATIONS`, `UNKNOWN_DECORATION` test blocks. Keep `TERMINAL_SEVERITY` and `isActiveStatus` tests.

```typescript
// packages/graph-stencil-case/src/runtime/badge-mappings.test.ts
import { describe, it, expect } from 'vitest';
import {
  TERMINAL_SEVERITY,
  isActiveStatus,
} from './badge-mappings.js';

describe('TERMINAL_SEVERITY', () => {
  it('ranks FAULTED highest', () => {
    expect(TERMINAL_SEVERITY['FAULTED']!).toBeGreaterThan(TERMINAL_SEVERITY['REJECTED']!);
    expect(TERMINAL_SEVERITY['REJECTED']!).toBeGreaterThan(TERMINAL_SEVERITY['CANCELLED']!);
    expect(TERMINAL_SEVERITY['CANCELLED']!).toBeGreaterThan(TERMINAL_SEVERITY['OBSOLETE']!);
    expect(TERMINAL_SEVERITY['OBSOLETE']!).toBeGreaterThan(TERMINAL_SEVERITY['COMPLETED']!);
  });
});

describe('isActiveStatus', () => {
  it('returns true for active states', () => {
    for (const s of ['PENDING', 'RUNNING', 'DELEGATED', 'SUSPENDED']) {
      expect(isActiveStatus(s)).toBe(true);
    }
  });

  it('returns false for terminal states', () => {
    for (const s of ['COMPLETED', 'FAULTED', 'REJECTED', 'OBSOLETE', 'CANCELLED']) {
      expect(isActiveStatus(s)).toBe(false);
    }
  });

  it('returns false for unknown states', () => {
    expect(isActiveStatus('UNKNOWN')).toBe(false);
  });
});
```

- [ ] **Step 7: Run all tests**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/`
Expected: PASS

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/runtime/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#109): toDecoration() via status registry — replaces hardcoded decoration maps"
```

---

### Task 5: Retrofit existing components

**Files:**
- Modify: `components/work-item-inbox/src/work-item-inbox.ts:121-131,140-144` (remove _statusColors, use status-badge)
- Modify: `components/session-list/src/session-list.ts:51-55,57-62` (remove _statusColors, use status-badge)
- Modify: `packages/blocks-ui-core/src/commitment-pill/commitment-state-pill.ts` (delegate to status-badge internally)

**Interfaces:**
- Consumes: `<status-badge>` from `@casehubio/blocks-ui-core`

- [ ] **Step 1: Retrofit work-item-inbox**

In `work-item-inbox.ts`:

1. Add import at top: `import '@casehubio/blocks-ui-core/status-badge/status-badge.js';`
2. Remove the static `_statusColors` record (lines 121-131)
3. Replace the STATUS_COL renderer (lines 140-144) with:

```typescript
[STATUS_COL, (cell: CellValue) => {
  const status = cell.type === 'NULL' ? '' : (cell as { value: string }).value;
  return html`<status-badge domain="workitem" state=${status} size="sm" showIcon></status-badge>`;
}],
```

4. Remove dead CSS rules for `.status-pill.status-obsolete`, `.status-pill.status-expired`, `.status-pill.status-escalated` if present in the static styles.

- [ ] **Step 2: Retrofit session-list**

In `session-list.ts`:

1. Add import at top: `import '@casehubio/blocks-ui-core/status-badge/status-badge.js';`
2. Remove the `_statusColors` record (lines 51-55)
3. Replace the STATUS_COL renderer (lines 57-62) with:

```typescript
[STATUS_COL, (cell: CellValue) => {
  const status = cell.type === 'NULL' ? '' : (cell as { value: string }).value;
  return html`<status-badge domain="session" state=${status} size="sm" showIcon></status-badge>`;
}],
```

- [ ] **Step 3: Retrofit commitment-state-pill — delegate internally**

Replace the `render()` method body to delegate to status-badge while keeping the same external API. Add `@deprecated` JSDoc to the class.

```typescript
// packages/blocks-ui-core/src/commitment-pill/commitment-state-pill.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { CommitmentState } from '../types/commitment.js';
import '../status-badge/status-badge.js';

/**
 * @deprecated Use `<status-badge domain="commitment">` instead.
 */
@customElement('commitment-state-pill')
export class CommitmentStatePill extends LitElement {
  @property({ type: String }) state?: CommitmentState;
  @property({ type: String }) size: 'sm' | 'md' = 'sm';
  @property({ type: Boolean }) showIcon = false;

  static override styles = css`
    :host { display: inline-block; }
  `;

  override render() {
    if (!this.state) return nothing;
    return html`<status-badge
      domain="commitment"
      state=${this.state}
      size=${this.size}
      ?showIcon=${this.showIcon}
    ></status-badge>`;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'commitment-state-pill': CommitmentStatePill;
  }
}
```

- [ ] **Step 4: Run all tests**

Run: `yarn test`
Expected: PASS — all existing tests pass (commitment-state-pill tests verify same DOM structure and content)

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/work-item-inbox/src/work-item-inbox.ts components/session-list/src/session-list.ts packages/blocks-ui-core/src/commitment-pill/commitment-state-pill.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(#109): retrofit work-item-inbox, session-list, commitment-state-pill to use status-badge"
```

---

### Task 6: Case-level status badge in diagram toolbar

**Files:**
- Modify: `packages/graph-stencil-case/src/runtime/types.ts:18` (add `caseStatus?: string` to `CaseRuntimeState`)
- Modify: `components/casehub-diagram/src/casehub-diagram-toolbar.ts` (add caseStatus prop, render badge)

**Interfaces:**
- Consumes: `<status-badge>` from `@casehubio/blocks-ui-core`, `CaseRuntimeState` from `graph-stencil-case`

- [ ] **Step 1: Extend `CaseRuntimeState`**

Add `caseStatus` to `packages/graph-stencil-case/src/runtime/types.ts`:

```typescript
export interface CaseRuntimeState {
  readonly planItems: readonly PlanItemSnapshot[];
  readonly milestones: readonly MilestoneSnapshot[];
  readonly timestamp: string;
  readonly caseStatus?: string;
}
```

- [ ] **Step 2: Add `caseStatus` to toolbar**

In `casehub-diagram-toolbar.ts`:

1. Add import: `import '@casehubio/blocks-ui-core/status-badge/status-badge.js';`
2. Add property: `@property({ type: String }) caseStatus?: string;`
3. Add badge rendering between the spacer and mode toggle:

```typescript
const caseBadge = this.caseStatus ? html`
  <status-badge domain="case" state=${this.caseStatus} size="sm" showIcon></status-badge>
` : nothing;

const modeSection = this.runtimeAvailable ? html`
  <span class="spacer"></span>
  ${caseBadge}
  <button class="mode-toggle" ...>
  ...
` : nothing;
```

- [ ] **Step 3: Run tests and typecheck**

Run: `yarn typecheck`
Expected: PASS

Run: `yarn test`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/runtime/types.ts components/casehub-diagram/src/casehub-diagram-toolbar.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#109): case-level status badge in diagram toolbar"
```

---

### Task 7: Final verification and docs

**Files:**
- Modify: `docs/guides/consumer-guide.md` (add status-badge section)
- Modify: `docs/guides/contributor-guide.md` (add registry extension docs)

**Interfaces:**
- Consumes: all previous tasks

- [ ] **Step 1: Run full test suite**

Run: `yarn test`
Expected: PASS

Run: `yarn typecheck`
Expected: PASS

Run: `yarn build`
Expected: PASS

- [ ] **Step 2: Update consumer guide — add status-badge usage section**

Add a section under the existing component documentation:

```markdown
### status-badge

Generic status pill for any domain's state enum.

\`\`\`html
<status-badge domain="case" state="RUNNING" showIcon></status-badge>
<status-badge domain="workitem" state="ASSIGNED" size="md"></status-badge>
<status-badge state="COMPLETED"></status-badge> <!-- cross-domain default -->
\`\`\`

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `state` | `string` | — | The state value to display |
| `domain` | `string` | — | Optional domain for domain-specific rendering |
| `size` | `'sm' \| 'md'` | `'sm'` | Pill size |
| `showIcon` | `boolean` | `false` | Show state icon |

Built-in domains: `case`, `task`, `workitem`, `work`, `milestone`, `outcome`, `group`, `sla`, `node`, `session`, `commitment`.
```

- [ ] **Step 3: Update contributor guide — add registry extension docs**

Add under the extension points section:

```markdown
### Extending the status registry

Register new domains for status-badge rendering:

\`\`\`typescript
import { registerStatus } from '@casehubio/blocks-ui-core';

registerStatus('myDomain', 'ACTIVE', { category: 'info', icon: '◉', pulse: true });
registerStatus('myDomain', 'DONE', { category: 'success', icon: '✓' });
\`\`\`

For graph decorations, use `toDecoration(domain, state)` from `@casehubio/graph-stencil-case`.
```

- [ ] **Step 4: Commit docs**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add docs/guides/consumer-guide.md docs/guides/contributor-guide.md
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "docs(#109): document status-badge component and registry extension API"
```
