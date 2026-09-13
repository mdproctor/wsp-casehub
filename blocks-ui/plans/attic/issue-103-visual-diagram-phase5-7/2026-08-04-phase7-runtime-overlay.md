# Phase 7 — Runtime Overlay Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — Epic: Visual Diagram Editor — Domain Layer
**Issue group:** #103

**Goal:** Project live execution state onto the definition-time graph as visual decorations, with property-based data delivery and a design/runtime mode toggle.

**Architecture:** Pure-function RuntimeAdapter maps `CaseRuntimeState` → `Map<string, NodeDecoration>`. Decorations flow through the pages graph-renderer pipeline via `toReactFlowGraph(model, layout, decorations)`. Stencils migrate from the old bridge API to the new `StencilDescriptor`-based registration.

**Tech Stack:** TypeScript, Lit 3.x, `@casehubio/graph-core` (NodeDecoration), `@casehubio/graph-renderer` (StencilDescriptor, registerStencil, toReactFlowGraph, computeElkLayout)

## Global Constraints

- All code strict TypeScript — no `any`
- Stencil render functions: inline styles only, `--pages-*` CSS custom properties
- Events: `composed: true, bubbles: true`
- Pre-release: breaking changes are fine

---

### Task 1: Runtime Types and Badge Mappings

**Files:**
- Create: `packages/graph-stencil-case/src/runtime/types.ts`
- Create: `packages/graph-stencil-case/src/runtime/badge-mappings.ts`
- Create: `packages/graph-stencil-case/src/runtime/badge-mappings.test.ts`

**Interfaces:**
- Produces: `TaskStatus`, `MilestoneLifecycleStatus`, `PlanItemSnapshot`, `MilestoneSnapshot`, `CaseRuntimeState` (types used by Task 2)
- Produces: `TASK_STATUS_DECORATIONS: Record<string, NodeDecoration>`, `MILESTONE_STATUS_DECORATIONS: Record<string, NodeDecoration>`, `UNKNOWN_DECORATION: NodeDecoration`, `TERMINAL_SEVERITY: Record<string, number>`, `isActiveStatus(status: string): boolean` (lookup tables used by Task 2)

- [ ] **Step 1: Create runtime types**

`packages/graph-stencil-case/src/runtime/types.ts`:

```typescript
import type { NodeDecoration } from '@casehubio/graph-core';

export type TaskStatus = 'PENDING' | 'RUNNING' | 'DELEGATED' | 'SUSPENDED'
  | 'COMPLETED' | 'FAULTED' | 'REJECTED' | 'OBSOLETE' | 'CANCELLED';

export type MilestoneLifecycleStatus = 'PENDING' | 'ACTIVE' | 'COMPLETED';

export interface PlanItemSnapshot {
  readonly id: string;
  readonly bindingName: string;
  readonly status: TaskStatus;
  readonly createdAt: string;
}

export interface MilestoneSnapshot {
  readonly name: string;
  readonly status: MilestoneLifecycleStatus;
}

export interface CaseRuntimeState {
  readonly planItems: readonly PlanItemSnapshot[];
  readonly milestones: readonly MilestoneSnapshot[];
  readonly timestamp: string;
}
```

- [ ] **Step 2: Write badge mapping tests**

`packages/graph-stencil-case/src/runtime/badge-mappings.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import {
  TASK_STATUS_DECORATIONS,
  MILESTONE_STATUS_DECORATIONS,
  UNKNOWN_DECORATION,
  TERMINAL_SEVERITY,
  isActiveStatus,
} from './badge-mappings.js';

describe('TASK_STATUS_DECORATIONS', () => {
  it('maps all 9 TaskStatus values', () => {
    const statuses = [
      'PENDING', 'RUNNING', 'DELEGATED', 'SUSPENDED',
      'COMPLETED', 'FAULTED', 'REJECTED', 'OBSOLETE', 'CANCELLED',
    ];
    for (const s of statuses) {
      expect(TASK_STATUS_DECORATIONS[s]).toBeDefined();
      expect(TASK_STATUS_DECORATIONS[s]!.badge).toBeDefined();
      expect(TASK_STATUS_DECORATIONS[s]!.badge!.icon).toBeTruthy();
      expect(TASK_STATUS_DECORATIONS[s]!.badge!.color).toBeTruthy();
    }
  });

  it('sets pulse only for RUNNING', () => {
    expect(TASK_STATUS_DECORATIONS['RUNNING']!.badge!.pulse).toBe(true);
    for (const s of ['PENDING', 'DELEGATED', 'SUSPENDED', 'COMPLETED', 'FAULTED', 'REJECTED', 'OBSOLETE', 'CANCELLED']) {
      expect(TASK_STATUS_DECORATIONS[s]!.badge!.pulse).toBeFalsy();
    }
  });

  it('sets border for active states except PENDING', () => {
    expect(TASK_STATUS_DECORATIONS['RUNNING']!.border).toBeDefined();
    expect(TASK_STATUS_DECORATIONS['DELEGATED']!.border).toBeDefined();
    expect(TASK_STATUS_DECORATIONS['SUSPENDED']!.border).toBeDefined();
    expect(TASK_STATUS_DECORATIONS['PENDING']!.border).toBeUndefined();
    expect(TASK_STATUS_DECORATIONS['COMPLETED']!.border).toBeUndefined();
  });
});

describe('MILESTONE_STATUS_DECORATIONS', () => {
  it('maps all 3 MilestoneLifecycleStatus values', () => {
    for (const s of ['PENDING', 'ACTIVE', 'COMPLETED']) {
      expect(MILESTONE_STATUS_DECORATIONS[s]).toBeDefined();
      expect(MILESTONE_STATUS_DECORATIONS[s]!.badge).toBeDefined();
    }
  });

  it('sets pulse only for ACTIVE', () => {
    expect(MILESTONE_STATUS_DECORATIONS['ACTIVE']!.badge!.pulse).toBe(true);
    expect(MILESTONE_STATUS_DECORATIONS['PENDING']!.badge!.pulse).toBeFalsy();
    expect(MILESTONE_STATUS_DECORATIONS['COMPLETED']!.badge!.pulse).toBeFalsy();
  });
});

describe('UNKNOWN_DECORATION', () => {
  it('has a gray question mark badge', () => {
    expect(UNKNOWN_DECORATION.badge!.icon).toBe('?');
    expect(UNKNOWN_DECORATION.badge!.color).toBe('#9ca3af');
  });
});

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

- [ ] **Step 3: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/badge-mappings.test.ts`
Expected: FAIL — modules not found

- [ ] **Step 4: Implement badge mappings**

`packages/graph-stencil-case/src/runtime/badge-mappings.ts`:

```typescript
import type { NodeDecoration } from '@casehubio/graph-core';

const ACTIVE_STATES = new Set(['PENDING', 'RUNNING', 'DELEGATED', 'SUSPENDED']);

export function isActiveStatus(status: string): boolean {
  return ACTIVE_STATES.has(status);
}

export const TASK_STATUS_DECORATIONS: Record<string, NodeDecoration> = {
  PENDING:   { badge: { icon: '○', color: '#9ca3af' } },
  RUNNING:   { badge: { icon: '▶', color: '#22c55e', pulse: true }, border: { style: 'solid', color: '#22c55e' } },
  DELEGATED: { badge: { icon: '→', color: '#3b82f6' }, border: { style: 'solid', color: '#3b82f6' } },
  SUSPENDED: { badge: { icon: '⏸', color: '#eab308' }, border: { style: 'solid', color: '#eab308' } },
  COMPLETED: { badge: { icon: '✓', color: '#22c55e' } },
  FAULTED:   { badge: { icon: '!', color: '#ef4444' } },
  REJECTED:  { badge: { icon: '✕', color: '#f97316' } },
  OBSOLETE:  { badge: { icon: '—', color: '#9ca3af' } },
  CANCELLED: { badge: { icon: '/', color: '#9ca3af' } },
};

export const MILESTONE_STATUS_DECORATIONS: Record<string, NodeDecoration> = {
  PENDING:   { badge: { icon: '○', color: '#9ca3af' } },
  ACTIVE:    { badge: { icon: '◉', color: '#3b82f6', pulse: true } },
  COMPLETED: { badge: { icon: '✓', color: '#22c55e' } },
};

export const UNKNOWN_DECORATION: NodeDecoration = {
  badge: { icon: '?', color: '#9ca3af' },
};

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

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/badge-mappings.test.ts`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/runtime/types.ts packages/graph-stencil-case/src/runtime/badge-mappings.ts packages/graph-stencil-case/src/runtime/badge-mappings.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): runtime types and badge mappings for Phase 7 overlay"
```

---

### Task 2: RuntimeAdapter (toDecorations)

**Files:**
- Create: `packages/graph-stencil-case/src/runtime/runtime-adapter.ts`
- Create: `packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`

**Interfaces:**
- Consumes: `CaseRuntimeState`, `PlanItemSnapshot`, `MilestoneSnapshot` from Task 1 types
- Consumes: `TASK_STATUS_DECORATIONS`, `MILESTONE_STATUS_DECORATIONS`, `UNKNOWN_DECORATION`, `TERMINAL_SEVERITY`, `ACTIVE_WORST_PRIORITY`, `isActiveStatus` from Task 1 badge-mappings
- Produces: `toDecorations(state: CaseRuntimeState): ReadonlyMap<string, NodeDecoration>` (used by Task 4)

- [ ] **Step 1: Write RuntimeAdapter tests**

`packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { toDecorations } from './runtime-adapter.js';
import type { CaseRuntimeState, PlanItemSnapshot } from './types.js';

function makeState(
  planItems: PlanItemSnapshot[] = [],
  milestones: CaseRuntimeState['milestones'] = [],
): CaseRuntimeState {
  return { planItems, milestones, timestamp: '2026-08-04T10:00:00Z' };
}

function planItem(bindingName: string, status: string, createdAt = '2026-08-04T10:00:00Z'): PlanItemSnapshot {
  return { id: `pi-${bindingName}-${status}`, bindingName, status: status as PlanItemSnapshot['status'], createdAt };
}

describe('toDecorations', () => {
  it('returns empty map for empty state', () => {
    const result = toDecorations(makeState());
    expect(result.size).toBe(0);
  });

  it('maps single PlanItem to binding node decoration', () => {
    const result = toDecorations(makeState([planItem('extract-text', 'RUNNING')]));
    const dec = result.get('binding:extract-text');
    expect(dec).toBeDefined();
    expect(dec!.badge!.icon).toBe('▶');
    expect(dec!.badge!.color).toBe('#22c55e');
    expect(dec!.badge!.pulse).toBe(true);
    expect(dec!.border).toBeDefined();
  });

  it('applies active-worst-first: SUSPENDED > RUNNING', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'RUNNING'),
      planItem('b1', 'SUSPENDED'),
    ]));
    const dec = result.get('binding:b1');
    expect(dec!.badge!.icon).toBe('⏸');
    expect(dec!.badge!.count).toBe(2);
  });

  it('applies active-worst-first: any active wins over terminal', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'COMPLETED', '2026-08-04T09:00:00Z'),
      planItem('b1', 'PENDING', '2026-08-04T10:00:00Z'),
    ]));
    expect(result.get('binding:b1')!.badge!.icon).toBe('○');
  });

  it('picks most recent terminal when all terminal', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'COMPLETED', '2026-08-04T09:00:00Z'),
      planItem('b1', 'FAULTED', '2026-08-04T10:00:00Z'),
    ]));
    expect(result.get('binding:b1')!.badge!.icon).toBe('!');
  });

  it('uses severity tiebreaker when terminal timestamps match', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'COMPLETED', '2026-08-04T10:00:00Z'),
      planItem('b1', 'FAULTED', '2026-08-04T10:00:00Z'),
    ]));
    expect(result.get('binding:b1')!.badge!.icon).toBe('!');
  });

  it('sets count when multiple PlanItems per binding', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'COMPLETED'),
      planItem('b1', 'COMPLETED'),
      planItem('b1', 'FAULTED'),
    ]));
    expect(result.get('binding:b1')!.badge!.count).toBe(3);
  });

  it('omits count for single PlanItem', () => {
    const result = toDecorations(makeState([planItem('b1', 'RUNNING')]));
    expect(result.get('binding:b1')!.badge!.count).toBeUndefined();
  });

  it('generates tooltip with breakdown', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'COMPLETED'),
      planItem('b1', 'COMPLETED'),
      planItem('b1', 'FAULTED'),
    ]));
    expect(result.get('binding:b1')!.tooltip).toBe('3 plan items: 2 completed, 1 faulted');
  });

  it('generates tooltip with just status name for single item', () => {
    const result = toDecorations(makeState([planItem('b1', 'RUNNING')]));
    expect(result.get('binding:b1')!.tooltip).toBe('running');
  });

  it('maps milestone to decoration', () => {
    const result = toDecorations(makeState([], [{ name: 'text-extracted', status: 'ACTIVE' }]));
    const dec = result.get('milestone:text-extracted');
    expect(dec).toBeDefined();
    expect(dec!.badge!.icon).toBe('◉');
    expect(dec!.badge!.pulse).toBe(true);
  });

  it('handles unknown TaskStatus with fallback decoration', () => {
    const result = toDecorations(makeState([planItem('b1', 'UNKNOWN_FUTURE_STATUS' as any)]));
    expect(result.get('binding:b1')!.badge!.icon).toBe('?');
  });

  it('handles multiple bindings independently', () => {
    const result = toDecorations(makeState([
      planItem('b1', 'RUNNING'),
      planItem('b2', 'COMPLETED'),
    ]));
    expect(result.get('binding:b1')!.badge!.icon).toBe('▶');
    expect(result.get('binding:b2')!.badge!.icon).toBe('✓');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement toDecorations**

`packages/graph-stencil-case/src/runtime/runtime-adapter.ts`:

```typescript
import type { NodeDecoration } from '@casehubio/graph-core';
import type { CaseRuntimeState, PlanItemSnapshot } from './types.js';
import {
  TASK_STATUS_DECORATIONS,
  MILESTONE_STATUS_DECORATIONS,
  UNKNOWN_DECORATION,
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

  const base = TASK_STATUS_DECORATIONS[statusKey] ?? UNKNOWN_DECORATION;
  const tooltip = buildTooltip(items);

  return {
    ...base,
    badge: { ...base.badge!, count },
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
    const base = MILESTONE_STATUS_DECORATIONS[milestone.status] ?? UNKNOWN_DECORATION;
    decorations.set(`milestone:${milestone.name}`, { ...base, tooltip: milestone.status.toLowerCase() });
  }

  return decorations;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/runtime/runtime-adapter.ts packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): RuntimeAdapter — toDecorations with active-worst-first aggregation"
```

---

### Task 3: Stencil API Migration

**Files:**
- Modify: `packages/graph-stencil-case/src/stencils/binding.ts`
- Modify: `packages/graph-stencil-case/src/stencils/worker.ts`
- Modify: `packages/graph-stencil-case/src/stencils/milestone.ts`
- Modify: `packages/graph-stencil-case/src/stencils/goal.ts`
- Modify: `packages/graph-stencil-case/src/stencils/subcase.ts`
- Modify: `packages/graph-stencil-case/src/stencils/register.ts`
- Modify: `packages/graph-stencil-case/src/stencils/index.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram.ts`
- Delete: `packages/graph-stencil-case/src/bridge/create-react-node-type.tsx` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-case/src/adapter/react-flow-transform.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-case/src/adapter/react-flow-transform.test.ts`

**Interfaces:**
- Consumes: `StencilRenderFn` = `(node: GraphNode, decoration?: NodeDecoration) => StencilTemplate` from `@casehubio/graph-renderer`
- Consumes: `registerStencil(descriptor: StencilDescriptor)` from `@casehubio/graph-renderer`
- Consumes: `toReactFlowGraph(model, layout?, decorations?)` from `@casehubio/graph-renderer`
- Consumes: `computeElkLayout(model, options?)` from `@casehubio/graph-renderer`
- Produces: Updated render functions compatible with `StencilRenderFn` signature

- [ ] **Step 1: Migrate binding.ts render function**

Change `renderBinding` from `(data: Record<string, unknown>) => TemplateResult` to `(node: GraphNode, decoration?: NodeDecoration) => StencilTemplate`. Replace `data` access with `node.properties`.

Use `ide_edit_member` on `renderBinding` in `packages/graph-stencil-case/src/stencils/binding.ts`:

```typescript
import { html } from 'lit-html';
import type { StencilGrammar, GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from '@casehubio/graph-renderer';

// bindingGrammar unchanged

export function renderBinding(node: GraphNode, decoration?: NodeDecoration): StencilTemplate {
  const data = node.properties;
  const name = String(data['name'] ?? '');
  const trigger = triggerLabel(data['on'] as Record<string, unknown> | undefined);
  const target = targetLabel(data);
  const when = data['when'] ? String(data['when']).slice(0, 40) : '';
  const opacity = decoration?.badge?.icon === '—' ? '0.5' : '1';

  return html`
    <div style="padding: 8px 12px; border-radius: 8px; border: 2px solid var(--pages-border-color, #ccc); background: var(--pages-surface-color, #fff); min-width: 180px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; opacity: ${opacity};">
      <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 4px;">
        <span style="font-weight: 600; color: var(--pages-text-color, #333);">${name}</span>
        <span style="background: var(--pages-accent-subtle, #e8f0fe); color: var(--pages-accent-color, #1a73e8); padding: 1px 6px; border-radius: 4px; font-size: 11px;">${trigger}</span>
      </div>
      <div style="color: var(--pages-text-secondary, #666); font-size: 12px;">→ ${target}</div>
      ${when ? html`<div style="color: var(--pages-text-tertiary, #999); font-size: 11px; margin-top: 2px; font-style: italic;">${when}</div>` : ''}
    </div>
  `;
}
```

- [ ] **Step 2: Migrate worker.ts, milestone.ts, goal.ts, subcase.ts**

Same pattern for each — change `(data: Record<string, unknown>) => TemplateResult` to `(node: GraphNode, decoration?: NodeDecoration) => StencilTemplate`, replace `data` with `node.properties`. Update imports. No decoration-specific logic in these stencils (only binding handles OBSOLETE opacity).

worker.ts:
```typescript
export function renderWorker(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const data = node.properties;
  // ... body unchanged
```

milestone.ts:
```typescript
export function renderMilestone(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const data = node.properties;
  // ... body unchanged
```

goal.ts:
```typescript
export function renderGoal(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const data = node.properties;
  // ... body unchanged
```

subcase.ts:
```typescript
export function renderSubCase(node: GraphNode, _decoration?: NodeDecoration): StencilTemplate {
  const data = node.properties;
  // ... body unchanged
```

Each file: update import from `import { html, type TemplateResult } from 'lit-html'` to `import { html } from 'lit-html'` and add `import type { GraphNode, NodeDecoration } from '@casehubio/graph-core'` and `import type { StencilTemplate } from '@casehubio/graph-renderer'`.

- [ ] **Step 3: Migrate register.ts to registerStencil**

Replace the contents of `packages/graph-stencil-case/src/stencils/register.ts`:

```typescript
import { registerStencil } from '@casehubio/graph-renderer';
import { bindingGrammar, renderBinding } from './binding.js';
import { workerGrammar, renderWorker } from './worker.js';
import { milestoneGrammar, renderMilestone } from './milestone.js';
import { goalGrammar, renderGoal } from './goal.js';
import { subcaseGrammar, renderSubCase } from './subcase.js';

let registered = false;

export function registerCaseStencils(): void {
  if (registered) return;
  registered = true;

  registerStencil({ type: 'binding', label: 'Binding', icon: 'link', grammar: bindingGrammar, render: renderBinding });
  registerStencil({ type: 'worker', label: 'Worker', icon: 'cpu', grammar: workerGrammar, render: renderWorker });
  registerStencil({ type: 'milestone', label: 'Milestone', icon: 'flag', grammar: milestoneGrammar, render: renderMilestone });
  registerStencil({ type: 'goal', label: 'Goal', icon: 'target', grammar: goalGrammar, render: renderGoal });
  registerStencil({ type: 'subcase', label: 'SubCase', icon: 'layers', grammar: subcaseGrammar, render: renderSubCase });
}
```

- [ ] **Step 4: Delete old bridge code**

Use `ide_refactor_safe_delete` on:
- `packages/graph-stencil-case/src/bridge/create-react-node-type.tsx`
- `packages/graph-stencil-case/src/adapter/react-flow-transform.ts`

Delete the test file:
- `packages/graph-stencil-case/src/adapter/react-flow-transform.test.ts`

- [ ] **Step 5: Update graph-stencil-case index.ts exports**

Remove `toReactFlowGraph` and `RFNode`/`RFEdge` exports (deleted). Add runtime exports:

```typescript
export { toGraph } from './adapter/case-adapter.js';
export type { AdapterResult } from './adapter/case-adapter.js';
export { applyPropertyEdit, addElement, removeElement, switchBindingTarget } from './adapter/yaml-editor.js';
export { GitHubBackend } from './persistence/github-backend.js';
export type { GitHubBackendConfig } from './persistence/github-backend.js';
export { registerCaseStencils } from './stencils/index.js';
export { renderBinding, renderWorker, renderMilestone, renderGoal, renderSubCase } from './stencils/index.js';
export { toDecorations } from './runtime/runtime-adapter.js';
export type {
  CaseRuntimeState,
  PlanItemSnapshot,
  MilestoneSnapshot,
  TaskStatus,
  MilestoneLifecycleStatus,
} from './runtime/types.js';
export type {
  CaseDefinition,
  CaseDefinitionSpec,
  Binding,
  Worker,
  Milestone,
  Goal,
  SubCase,
  Capability,
  HumanTask,
  Trigger,
} from './types/case-definition.js';
```

- [ ] **Step 6: Migrate casehub-diagram.ts imports and pipeline**

Update imports at top of `components/casehub-diagram/src/casehub-diagram.ts`:

```typescript
import { LitElement, html, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import {
  toGraph,
  registerCaseStencils,
  applyPropertyEdit,
  addElement,
  removeElement,
  switchBindingTarget,
} from '@casehubio/graph-stencil-case';
import type { AdapterResult } from '@casehubio/graph-stencil-case';
import { computeElkLayout, toReactFlowGraph } from '@casehubio/graph-renderer';
import type { ElkLayoutResult } from '@casehubio/graph-renderer';
import type { Node, Edge } from '@xyflow/react';
import { edgesOf } from '@casehubio/graph-core';
import type { PersistenceBackend } from '@casehubio/graph-core';
```

Change `_nodes` and `_edges` types from `RFNode[]`/`RFEdge[]` to `Node[]`/`Edge[]`.

Add new field: `private _lastLayout: ElkLayoutResult | undefined;`

Update `_fullRender`:

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
    this._lastLayout = await computeElkLayout(this._adapterResult.model, { direction: 'DOWN', spacing: 60 });
    const { nodes, edges } = toReactFlowGraph(this._adapterResult.model, this._lastLayout);
    this._nodes = nodes;
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

Update `_updateWithoutLayout`:

```typescript
private _updateWithoutLayout(yamlStr: string): void {
  try {
    this._error = '';
    this._adapterResult = toGraph(yamlStr);
    const { nodes, edges } = toReactFlowGraph(this._adapterResult.model, this._lastLayout);
    this._nodes = nodes;
    this._edges = edges;
    this._updateSelectedNode();
  } catch (e) {
    this._error = `Edit failed: ${e}`;
    this._currentYaml = this._undoStack.pop() ?? this._currentYaml;
  }
}
```

- [ ] **Step 7: Run full test suite**

Run: `yarn test`
Expected: All existing tests pass with the new API

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add -A
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): migrate stencils to pages StencilDescriptor API, delete old bridge"
```

---

### Task 4: Runtime Overlay Integration

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram-toolbar.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`
- Create: `components/casehub-diagram/src/casehub-diagram.runtime.test.ts`

**Interfaces:**
- Consumes: `toDecorations(state: CaseRuntimeState): ReadonlyMap<string, NodeDecoration>` from Task 2
- Consumes: `CaseRuntimeState` from Task 1
- Consumes: `toReactFlowGraph(model, layout, decorations)` from pages (wired in Task 3)

- [ ] **Step 1: Write toolbar mode toggle tests**

Add to `components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`:

```typescript
describe('mode toggle', () => {
  it('hides toggle when runtimeAvailable is false', async () => {
    const el = await fixture<CasehubDiagramToolbar>(html`<casehub-diagram-toolbar .hasBackend=${true}></casehub-diagram-toolbar>`);
    expect(el.shadowRoot!.querySelector('.mode-toggle')).toBeNull();
  });

  it('shows toggle when runtimeAvailable is true', async () => {
    const el = await fixture<CasehubDiagramToolbar>(html`<casehub-diagram-toolbar .hasBackend=${true} .runtimeAvailable=${true}></casehub-diagram-toolbar>`);
    expect(el.shadowRoot!.querySelector('.mode-toggle')).not.toBeNull();
  });

  it('emits toolbar-mode-change on toggle click', async () => {
    const el = await fixture<CasehubDiagramToolbar>(html`<casehub-diagram-toolbar .hasBackend=${true} .runtimeAvailable=${true}></casehub-diagram-toolbar>`);
    const events: CustomEvent[] = [];
    el.addEventListener('toolbar-mode-change', (e) => events.push(e as CustomEvent));
    el.shadowRoot!.querySelector<HTMLButtonElement>('.mode-toggle')!.click();
    expect(events).toHaveLength(1);
    expect(events[0]!.detail.mode).toBe('runtime');
  });

  it('toggles between design and runtime', async () => {
    const el = await fixture<CasehubDiagramToolbar>(html`<casehub-diagram-toolbar .hasBackend=${true} .runtimeAvailable=${true}></casehub-diagram-toolbar>`);
    const events: CustomEvent[] = [];
    el.addEventListener('toolbar-mode-change', (e) => events.push(e as CustomEvent));
    const btn = el.shadowRoot!.querySelector<HTMLButtonElement>('.mode-toggle')!;
    btn.click();
    expect(events[0]!.detail.mode).toBe('runtime');
    el.mode = 'runtime';
    await el.updateComplete;
    btn.click();
    expect(events[1]!.detail.mode).toBe('design');
  });

  it('shows staleness badge when staleSeconds > 0', async () => {
    const el = await fixture<CasehubDiagramToolbar>(html`<casehub-diagram-toolbar .hasBackend=${true} .runtimeAvailable=${true} .staleSeconds=${45}></casehub-diagram-toolbar>`);
    const badge = el.shadowRoot!.querySelector('.stale-badge');
    expect(badge).not.toBeNull();
    expect(badge!.textContent).toContain('45s');
  });

  it('hides staleness badge when staleSeconds is 0', async () => {
    const el = await fixture<CasehubDiagramToolbar>(html`<casehub-diagram-toolbar .hasBackend=${true} .runtimeAvailable=${true} .staleSeconds=${0}></casehub-diagram-toolbar>`);
    expect(el.shadowRoot!.querySelector('.stale-badge')).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`
Expected: FAIL — properties not defined

- [ ] **Step 3: Add mode toggle and staleness to toolbar**

Update `components/casehub-diagram/src/casehub-diagram-toolbar.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('casehub-diagram-toolbar')
export class CasehubDiagramToolbar extends LitElement {
  @property({ type: Boolean }) dirty = false;
  @property({ type: Boolean }) saving = false;
  @property({ type: Boolean }) hasBackend = false;
  @property({ type: Boolean }) runtimeAvailable = false;
  @property({ type: String }) mode: 'design' | 'runtime' = 'design';
  @property({ type: Number }) staleSeconds = 0;

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
    .mode-toggle[aria-pressed="true"] { background: var(--pages-accent-subtle, #e8f0fe); border-color: var(--pages-accent-color, #1a73e8); color: var(--pages-accent-color, #1a73e8); }
    .spacer { flex: 1; }
    .stale-badge { font-size: 11px; color: var(--pages-warning-color, #f59e0b); }
  `;

  override render() {
    const saveSection = this.hasBackend ? html`
      <button ?disabled=${!this.dirty || this.saving} @click=${this._save}>
        ${this.saving ? 'Saving…' : 'Save'}
      </button>
      ${this.dirty ? html`<span class="dirty-dot"></span>` : nothing}
    ` : nothing;

    const modeSection = this.runtimeAvailable ? html`
      <span class="spacer"></span>
      <button class="mode-toggle"
        aria-pressed=${this.mode === 'runtime'}
        @click=${this._toggleMode}>
        ${this.mode === 'design' ? '⚡ Runtime' : '✏️ Design'}
      </button>
      ${this.staleSeconds > 0 ? html`<span class="stale-badge">⚠ stale (${this.staleSeconds}s ago)</span>` : nothing}
    ` : nothing;

    return html`${saveSection}${modeSection}`;
  }

  private _save(): void {
    this.dispatchEvent(new CustomEvent('toolbar-save', { bubbles: true, composed: true }));
  }

  private _toggleMode(): void {
    const newMode = this.mode === 'design' ? 'runtime' : 'design';
    this.dispatchEvent(new CustomEvent('toolbar-mode-change', {
      detail: { mode: newMode },
      bubbles: true,
      composed: true,
    }));
  }
}
```

- [ ] **Step 4: Run toolbar tests**

Run: `yarn vitest run components/casehub-diagram/src/casehub-diagram-toolbar.test.ts`
Expected: PASS

- [ ] **Step 5: Write casehub-diagram runtime integration tests**

Create `components/casehub-diagram/src/casehub-diagram.runtime.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import type { CaseRuntimeState } from '@casehubio/graph-stencil-case';
import { toDecorations } from '@casehubio/graph-stencil-case';

describe('toDecorations integration', () => {
  it('produces decorations from runtime state', () => {
    const state: CaseRuntimeState = {
      planItems: [
        { id: 'p1', bindingName: 'extract-text', status: 'RUNNING', createdAt: '2026-08-04T10:00:00Z' },
        { id: 'p2', bindingName: 'classify-document', status: 'COMPLETED', createdAt: '2026-08-04T09:00:00Z' },
      ],
      milestones: [
        { name: 'text-extracted', status: 'COMPLETED' },
      ],
      timestamp: '2026-08-04T10:00:00Z',
    };
    const decorations = toDecorations(state);
    expect(decorations.get('binding:extract-text')!.badge!.icon).toBe('▶');
    expect(decorations.get('binding:classify-document')!.badge!.icon).toBe('✓');
    expect(decorations.get('milestone:text-extracted')!.badge!.icon).toBe('✓');
  });
});

describe('staleness computation', () => {
  it('returns 0 when timestamp is recent', () => {
    const now = Date.now();
    const timestamp = new Date(now - 5000).toISOString();
    const stale = Math.max(0, Math.floor((now - new Date(timestamp).getTime()) / 1000) - 30);
    expect(stale).toBe(0);
  });

  it('returns positive seconds when timestamp is old', () => {
    const now = Date.now();
    const timestamp = new Date(now - 75000).toISOString();
    const elapsed = Math.floor((now - new Date(timestamp).getTime()) / 1000);
    const stale = Math.max(0, elapsed - 30);
    expect(stale).toBeGreaterThan(0);
  });
});
```

- [ ] **Step 6: Add runtimeState property and mode logic to casehub-diagram**

Add to the `CasehubDiagram` class in `components/casehub-diagram/src/casehub-diagram.ts`:

New imports:
```typescript
import { toDecorations } from '@casehubio/graph-stencil-case';
import type { CaseRuntimeState } from '@casehubio/graph-stencil-case';
import type { NodeDecoration } from '@casehubio/graph-core';
```

New properties and state:
```typescript
@property({ attribute: false }) runtimeState: CaseRuntimeState | null = null;
@state() private _mode: 'design' | 'runtime' = 'design';
@state() private _staleSeconds = 0;
```

In `updated()`, add runtimeState handling:
```typescript
if (changedProperties.has('runtimeState')) {
  if (this.runtimeState === null) {
    this._mode = 'design';
    this._staleSeconds = 0;
  } else {
    this._updateStaleness();
    if (this._mode === 'runtime') {
      this._applyDecorations();
    }
  }
}
```

New methods:
```typescript
private _updateStaleness(): void {
  if (!this.runtimeState) { this._staleSeconds = 0; return; }
  const elapsed = Math.floor((Date.now() - new Date(this.runtimeState.timestamp).getTime()) / 1000);
  this._staleSeconds = Math.max(0, elapsed - 30);
}

private _applyDecorations(): void {
  if (!this._adapterResult || !this.runtimeState) return;
  const decorations = toDecorations(this.runtimeState);
  const { nodes, edges } = toReactFlowGraph(this._adapterResult.model, this._lastLayout, decorations);
  this._nodes = nodes;
  this._edges = edges;
}

private _handleModeChange(e: CustomEvent<{ mode: 'design' | 'runtime' }>): void {
  this._mode = e.detail.mode;
  if (this._mode === 'runtime' && this.runtimeState) {
    this._applyDecorations();
  } else {
    const { nodes, edges } = toReactFlowGraph(this._adapterResult!.model, this._lastLayout);
    this._nodes = nodes;
    this._edges = edges;
  }
}
```

Update the toolbar in `render()` to pass runtime props:
```typescript
<casehub-diagram-toolbar
  .dirty=${this._currentYaml !== this._savedYaml}
  .saving=${this._saving}
  .hasBackend=${this.backend !== null}
  .runtimeAvailable=${this.runtimeState !== null}
  .mode=${this._mode}
  .staleSeconds=${this._staleSeconds}
  @toolbar-save=${this._save}
  @toolbar-mode-change=${this._handleModeChange}
></casehub-diagram-toolbar>
```

- [ ] **Step 7: Run full test suite**

Run: `yarn test`
Expected: All tests pass

- [ ] **Step 8: Run typecheck**

Run: `yarn typecheck`
Expected: No errors

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add -A
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(#103): runtime overlay — mode toggle, decoration flow, staleness indicator"
```
