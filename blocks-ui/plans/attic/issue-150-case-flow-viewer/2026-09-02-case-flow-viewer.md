# blocks-case-flow-viewer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #150 — blocks-case-flow-viewer: read-only case flow DAG with runtime state
**Issue group:** #150

**Goal:** Read-only case flow viewer composing the same rendering pipeline as casehub-diagram, with runtime decorations, trust score pills, adaptive decision badges, and ELK-partitioned parallel groups.

**Architecture:** Extends `DiagramBaseMixin` in readonly mode. Overrides `_adaptYaml()` → `toGraph()`, `_decorations()` → `toDecorations()`. New runtime state fields (trust scores, adaptive decisions, parallel groups) extend `CaseRuntimeState` in `graph-stencil-case`. Parallel groups use ELK partition constraints via `_layoutOptions()`.

**Tech Stack:** LitElement, TypeScript, `@casehubio/pages-diagram-core` (DiagramBaseMixin), `@casehubio/graph-stencil-case` (toGraph, toDecorations, registerCaseStencils), `@casehubio/graph-renderer` (computeElkLayout, pages-graph-canvas), `@casehubio/graph-core` (NodeDecoration with pills), vitest

## Global Constraints

- Pre-release project — no backward compatibility shims
- ARIA mandatory on every `@customElement`
- Stencil package isolation: never import from `graph-stencil-swf` or `graph-stencil-htn` (PP-20260806)
- Component customisation via typed config properties + render callbacks, not slots (PP-20260713)
- Render callbacks use inline styles (shadow DOM boundary)
- `NodeDecoration.pills` available in `graph-core` (upstream casehub-pages#404)
- `ElkLayoutOptions.partitions` needed in `graph-renderer` (upstream — see Batch 3 prerequisite)

## Prerequisites (upstream pages changes)

1. **casehub-pages#404** — `NodeDecoration.pills` in `graph-core` ✅ (done by user)
2. **ElkLayoutOptions.partitions** in `graph-renderer` — add `partitions?: ReadonlyMap<string, number>` to `ElkLayoutOptions`, activate `elk.partitioning.activate` when present, set `elk.partitioning.partition` per node in `buildElkNode`. (User does this before Batch 3.)

After pages changes, run `yarn build && mvn install` in casehub-pages, then in blocks-ui run `yarn install` to pick up the updated SNAPSHOTs.

---

## Batch 1: Runtime type extensions + decoration pipeline

Extends `CaseRuntimeState` and `toDecorations()` in `graph-stencil-case` to handle trust scores, adaptive decisions, and parallel groups. After this batch, the existing `casehub-diagram` can also benefit from these decorations.

### Task 1: Extend CaseRuntimeState with flow fields

**Files:**
- Modify: `packages/graph-stencil-case/src/runtime/types.ts`
- Test: `packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`

**Interfaces:**
- Produces: `TrustScoreSnapshot { bindingName: string; workerId: string; score: number }`, `AdaptiveDecisionSnapshot { trigger: string; condition: string; fired: boolean; timestamp: string; affectedBindings?: readonly string[] }`, extended `CaseRuntimeState` with optional `trustScores`, `adaptiveDecisions`, `parallelGroups` fields

- [ ] **Step 1: Write failing test — TrustScoreSnapshot type is usable**

Add to `packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`:

```typescript
import type { CaseRuntimeState, PlanItemSnapshot, TrustScoreSnapshot, AdaptiveDecisionSnapshot } from './types.js';

// Add after the existing makeState helper:
function makeStateWithTrust(
  planItems: PlanItemSnapshot[] = [],
  trustScores: TrustScoreSnapshot[] = [],
): CaseRuntimeState {
  return { planItems, milestones: [], timestamp: '2026-08-04T10:00:00Z', trustScores };
}

function makeStateWithAdaptive(
  planItems: PlanItemSnapshot[] = [],
  adaptiveDecisions: AdaptiveDecisionSnapshot[] = [],
): CaseRuntimeState {
  return { planItems, milestones: [], timestamp: '2026-08-04T10:00:00Z', adaptiveDecisions };
}
```

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: FAIL — `TrustScoreSnapshot` and `AdaptiveDecisionSnapshot` not exported from types

- [ ] **Step 2: Add types to types.ts**

Add to `packages/graph-stencil-case/src/runtime/types.ts`:

```typescript
export interface TrustScoreSnapshot {
  readonly bindingName: string;
  readonly workerId: string;
  readonly score: number;
}

export interface AdaptiveDecisionSnapshot {
  readonly trigger: string;
  readonly condition: string;
  readonly fired: boolean;
  readonly timestamp: string;
  readonly affectedBindings?: readonly string[];
}
```

Extend `CaseRuntimeState`:

```typescript
export interface CaseRuntimeState {
  readonly planItems: readonly PlanItemSnapshot[];
  readonly milestones: readonly MilestoneSnapshot[];
  readonly timestamp: string;
  readonly caseStatus?: string;
  readonly trustScores?: readonly TrustScoreSnapshot[];
  readonly adaptiveDecisions?: readonly AdaptiveDecisionSnapshot[];
  readonly parallelGroups?: readonly (readonly string[])[];
}
```

- [ ] **Step 3: Run test to verify it passes**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: PASS — types compile, existing tests unaffected

- [ ] **Step 4: Export new types from package index**

Check `packages/graph-stencil-case/src/index.ts` and ensure `TrustScoreSnapshot` and `AdaptiveDecisionSnapshot` are re-exported alongside the existing `CaseRuntimeState` export.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/runtime/types.ts packages/graph-stencil-case/src/index.ts packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(graph-stencil-case): extend CaseRuntimeState with trust scores, adaptive decisions, parallel groups

Refs #150"
```

### Task 2: Extend toDecorations for trust score pills and adaptive decision badges

**Files:**
- Modify: `packages/graph-stencil-case/src/runtime/runtime-adapter.ts`
- Modify: `packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`

**Interfaces:**
- Consumes: `TrustScoreSnapshot`, `AdaptiveDecisionSnapshot`, `CaseRuntimeState` from Task 1
- Produces: Extended `toDecorations(state)` that returns `NodeDecoration` entries with `pills` for trust scores and secondary badge + tooltip for fired adaptive decisions

- [ ] **Step 1: Write failing test — trust score pills**

Add to `runtime-adapter.test.ts`:

```typescript
describe('toDecorations — trust scores', () => {
  it('adds trust score pill to binding decoration', () => {
    const result = toDecorations(makeStateWithTrust(
      [planItem('extract-text', 'RUNNING')],
      [{ bindingName: 'extract-text', workerId: 'w1', score: 85 }],
    ));
    const dec = result.get('binding:extract-text');
    expect(dec).toBeDefined();
    expect(dec!.pills).toBeDefined();
    expect(dec!.pills).toHaveLength(1);
    expect(dec!.pills![0].text).toBe('85');
    expect(dec!.pills![0].color).toBe('#22c55e');
  });

  it('uses amber for moderate trust score', () => {
    const result = toDecorations(makeStateWithTrust(
      [planItem('b1', 'COMPLETED')],
      [{ bindingName: 'b1', workerId: 'w1', score: 65 }],
    ));
    expect(result.get('binding:b1')!.pills![0].color).toBe('#eab308');
  });

  it('uses red for low trust score', () => {
    const result = toDecorations(makeStateWithTrust(
      [planItem('b1', 'COMPLETED')],
      [{ bindingName: 'b1', workerId: 'w1', score: 30 }],
    ));
    expect(result.get('binding:b1')!.pills![0].color).toBe('#ef4444');
  });

  it('skips trust pill when no matching binding in plan items', () => {
    const result = toDecorations(makeStateWithTrust(
      [planItem('b1', 'RUNNING')],
      [{ bindingName: 'b-other', workerId: 'w1', score: 90 }],
    ));
    expect(result.get('binding:b1')!.pills).toBeUndefined();
  });

  it('handles boundary score 80 as green', () => {
    const result = toDecorations(makeStateWithTrust(
      [planItem('b1', 'COMPLETED')],
      [{ bindingName: 'b1', workerId: 'w1', score: 80 }],
    ));
    expect(result.get('binding:b1')!.pills![0].color).toBe('#22c55e');
  });

  it('handles boundary score 50 as amber', () => {
    const result = toDecorations(makeStateWithTrust(
      [planItem('b1', 'COMPLETED')],
      [{ bindingName: 'b1', workerId: 'w1', score: 50 }],
    ));
    expect(result.get('binding:b1')!.pills![0].color).toBe('#eab308');
  });
});
```

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: FAIL — pills not set on decorations

- [ ] **Step 2: Implement trust score pill generation in toDecorations**

In `runtime-adapter.ts`, add a trust score colour helper:

```typescript
function trustScoreColor(score: number): string {
  if (score >= 80) return '#22c55e';
  if (score >= 50) return '#eab308';
  return '#ef4444';
}
```

At the end of `toDecorations()`, after the existing binding and milestone loops, add trust score pill mapping:

```typescript
if (state.trustScores) {
  for (const ts of state.trustScores) {
    const key = `binding:${ts.bindingName}`;
    const existing = decorations.get(key);
    if (existing) {
      decorations.set(key, {
        ...existing,
        pills: [{ text: String(ts.score), color: trustScoreColor(ts.score) }],
      });
    }
  }
}
```

- [ ] **Step 3: Run tests to verify trust score pills pass**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: PASS — all trust score tests green, existing tests unaffected

- [ ] **Step 4: Write failing test — adaptive decision badges**

Add to `runtime-adapter.test.ts`:

```typescript
describe('toDecorations — adaptive decisions', () => {
  it('adds tooltip for fired adaptive decision on affected binding', () => {
    const result = toDecorations(makeStateWithAdaptive(
      [planItem('b1', 'RUNNING')],
      [{
        trigger: 'trust-drop', condition: 'score < 50', fired: true,
        timestamp: '2026-08-04T10:01:00Z', affectedBindings: ['b1'],
      }],
    ));
    const dec = result.get('binding:b1');
    expect(dec!.tooltip).toContain('trust-drop');
    expect(dec!.tooltip).toContain('score < 50');
  });

  it('does not modify decoration for unfired adaptive decision', () => {
    const result = toDecorations(makeStateWithAdaptive(
      [planItem('b1', 'RUNNING')],
      [{
        trigger: 'trust-drop', condition: 'score < 50', fired: false,
        timestamp: '2026-08-04T10:01:00Z', affectedBindings: ['b1'],
      }],
    ));
    const dec = result.get('binding:b1');
    expect(dec!.tooltip).toBe('running');
  });

  it('handles adaptive decision with no affectedBindings', () => {
    const result = toDecorations(makeStateWithAdaptive(
      [planItem('b1', 'RUNNING')],
      [{
        trigger: 'trust-drop', condition: 'score < 50', fired: true,
        timestamp: '2026-08-04T10:01:00Z',
      }],
    ));
    const dec = result.get('binding:b1');
    expect(dec!.tooltip).toBe('running');
  });
});
```

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: FAIL — adaptive tooltip not applied

- [ ] **Step 5: Implement adaptive decision decoration in toDecorations**

At the end of `toDecorations()`, after trust score handling, add:

```typescript
if (state.adaptiveDecisions) {
  for (const ad of state.adaptiveDecisions) {
    if (!ad.fired || !ad.affectedBindings) continue;
    for (const bindingName of ad.affectedBindings) {
      const key = `binding:${bindingName}`;
      const existing = decorations.get(key);
      if (existing) {
        const adaptiveTooltip = `⚡ ${ad.trigger}: ${ad.condition}`;
        const currentTooltip = existing.tooltip ?? '';
        decorations.set(key, {
          ...existing,
          tooltip: currentTooltip ? `${currentTooltip}\n${adaptiveTooltip}` : adaptiveTooltip,
        });
      }
    }
  }
}
```

- [ ] **Step 6: Run all tests to verify**

Run: `yarn vitest run packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts`
Expected: PASS — all tests green

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/runtime/runtime-adapter.ts packages/graph-stencil-case/src/runtime/runtime-adapter.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(graph-stencil-case): trust score pills and adaptive decision badges in toDecorations

Refs #150"
```

---

## Batch 2: Case flow viewer component

New component composing DiagramBaseMixin with case stencils. After this batch, the viewer renders case definition graphs with full runtime decorations, trust scores, and adaptive decision badges.

### Task 3: Scaffold component and implement BlocksCaseFlowViewer

**Files:**
- Create: `components/case-flow-viewer/package.json`
- Create: `components/case-flow-viewer/tsconfig.json`
- Create: `components/case-flow-viewer/tsconfig.build.json`
- Create: `components/case-flow-viewer/src/blocks-case-flow-viewer.ts`
- Create: `components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`
- Modify: `tsconfig.base.json` — add project reference

**Interfaces:**
- Consumes: `DiagramBaseMixin` from `@casehubio/pages-diagram-core`, `toGraph` from `@casehubio/graph-stencil-case`, `toDecorations` and `CaseRuntimeState` from `@casehubio/graph-stencil-case`, `registerCaseStencils` from `@casehubio/graph-stencil-case`, `emitPagesEvent` from `@casehubio/pages-data`
- Produces: `<blocks-case-flow-viewer>` custom element with `yaml`, `src`, `runtimeState`, `selectionTopic` properties

- [ ] **Step 1: Create package.json**

Create `components/case-flow-viewer/package.json`:

```json
{
  "name": "@casehubio/blocks-ui-case-flow-viewer",
  "version": "0.1.0",
  "description": "Read-only case flow DAG viewer with runtime state",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/blocks-ui.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/blocks-case-flow-viewer.js",
  "types": "dist/blocks-case-flow-viewer.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "devDependencies": {
    "jsdom": "^29.1.1",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/graph-core": "*",
    "@casehubio/graph-renderer": "*",
    "@casehubio/graph-stencil-case": "workspace:*",
    "@casehubio/pages-data": "*",
    "@casehubio/pages-diagram-core": "*",
    "lit": "^3.3.3"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json and tsconfig.build.json**

Create `components/case-flow-viewer/tsconfig.json`:

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
    { "path": "../../packages/graph-stencil-case" },
    { "path": "../../packages/blocks-ui-core" }
  ]
}
```

Create `components/case-flow-viewer/tsconfig.build.json`:

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "exclude": ["**/*.test.ts", "**/*.spec.ts"]
}
```

- [ ] **Step 3: Add project reference to tsconfig.base.json**

Add `{ "path": "components/case-flow-viewer/tsconfig.build.json" }` to the `references` array in `tsconfig.base.json`, alphabetically near the other `components/` entries.

- [ ] **Step 4: Write failing test — component instantiation and ARIA**

Create `components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`:

```typescript
// @vitest-environment jsdom
import { describe, it, expect } from 'vitest';
import { BlocksCaseFlowViewer } from './blocks-case-flow-viewer.js';

describe('BlocksCaseFlowViewer', () => {
  it('can be instantiated', () => {
    const el = new BlocksCaseFlowViewer();
    expect(el).toBeDefined();
  });

  it('defaults readonly to true', () => {
    const el = new BlocksCaseFlowViewer();
    expect(el.readonly).toBe(true);
  });

  it('defaults runtimeState to null', () => {
    const el = new BlocksCaseFlowViewer();
    expect(el.runtimeState).toBeNull();
  });

  it('defaults selectionTopic to empty', () => {
    const el = new BlocksCaseFlowViewer();
    expect(el.selectionTopic).toBe('');
  });

  it('accepts runtimeState property', () => {
    const el = new BlocksCaseFlowViewer();
    el.runtimeState = {
      planItems: [], milestones: [], timestamp: '2026-09-02T10:00:00Z',
    };
    expect(el.runtimeState).toBeDefined();
  });

  it('sets aria-label on connect', () => {
    const el = new BlocksCaseFlowViewer();
    document.body.appendChild(el);
    expect(el.getAttribute('aria-label')).toBe('Case flow viewer');
    expect(el.getAttribute('role')).toBe('region');
    el.remove();
  });
});
```

Run: `yarn vitest run components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`
Expected: FAIL — module not found

- [ ] **Step 5: Implement BlocksCaseFlowViewer**

Create `components/case-flow-viewer/src/blocks-case-flow-viewer.ts`:

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { DiagramBaseMixin } from '@casehubio/pages-diagram-core';
import type { AdapterResult } from '@casehubio/pages-diagram-core';
import { toGraph } from '@casehubio/graph-stencil-case';
import { toDecorations, registerCaseStencils } from '@casehubio/graph-stencil-case';
import type { CaseRuntimeState } from '@casehubio/graph-stencil-case';
import type { NodeDecoration } from '@casehubio/graph-core';
import { emitPagesEvent } from '@casehubio/pages-data';
import { toReactFlowGraph } from '@casehubio/graph-renderer';
import '@casehubio/graph-renderer';
import './blocks-case-flow-toolbar.js';

@customElement('blocks-case-flow-viewer')
export class BlocksCaseFlowViewer extends DiagramBaseMixin(LitElement) {
  @property({ attribute: false }) runtimeState: CaseRuntimeState | null = null;
  @property({ attribute: 'selection-topic' }) selectionTopic = '';

  override connectedCallback(): void {
    super.connectedCallback();
    this.readonly = true;
    this.setAttribute('role', 'region');
    this.setAttribute('aria-label', 'Case flow viewer');
    registerCaseStencils();
  }

  override async updated(changed: Map<PropertyKey, unknown>): Promise<void> {
    await super.updated(changed);
    if (changed.has('runtimeState')) {
      this._applyRuntimeDecorations();
    }
  }

  private _applyRuntimeDecorations(): void {
    if (!this._adapterResult || !this._lastLayout) return;
    const decorations = this._decorations();
    const { nodes, edges } = toReactFlowGraph(
      this._adapterResult.model, this._lastLayout, decorations, this._layoutOptions().direction,
    );
    this._nodes = nodes;
    this._edges = edges;
  }

  protected _adaptYaml(yaml: string): AdapterResult {
    return toGraph(yaml);
  }

  protected _applyPropertyEdit(): string {
    throw new Error('BlocksCaseFlowViewer is read-only');
  }

  protected _emptyTemplate(): string | null {
    return null;
  }

  protected override _decorations(): ReadonlyMap<string, NodeDecoration> | undefined {
    if (this.runtimeState) {
      return toDecorations(this.runtimeState);
    }
    return undefined;
  }

  private _computeStats() {
    if (!this.runtimeState || !this._adapterResult) {
      return { nodeCount: 0, completed: 0, running: 0, failed: 0 };
    }
    const items = this.runtimeState.planItems;
    return {
      nodeCount: this._adapterResult.model.nodes.length,
      completed: items.filter(i => i.status === 'COMPLETED').length,
      running: items.filter(i => i.status === 'RUNNING').length,
      failed: items.filter(i => i.status === 'FAULTED').length,
    };
  }

  private _onNodeClick(nodeId: string): void {
    if (!this.selectionTopic) return;
    const node = this._adapterResult?.model.nodes.find(n => n.id === nodeId);
    emitPagesEvent(this, this.selectionTopic, {
      nodeId,
      nodeType: node?.type ?? '',
      properties: node?.properties ?? {},
    });
  }

  override render() {
    const stats = this._computeStats();
    return html`
      <blocks-case-flow-toolbar
        .nodeCount=${stats.nodeCount}
        .completedCount=${stats.completed}
        .runningCount=${stats.running}
        .failedCount=${stats.failed}
        .caseStatus=${this.runtimeState?.caseStatus ?? null}
        .resultTimestamp=${this.runtimeState?.timestamp ?? null}
        @export-svg=${() => this._exportDiagram('svg')}
        @export-png=${() => this._exportDiagram('png')}
      ></blocks-case-flow-toolbar>
      <div class="canvas-area" role="img" aria-label="Case flow diagram">
        ${this._error
          ? this._renderError()
          : this._adapterResult == null
            ? html`<div class="empty">No case definition loaded</div>`
            : html`<pages-graph-canvas
                .nodes=${this._nodes}
                .edges=${this._edges}
                style="width: 100%; height: 100%;"
                @pages-event=${(e: CustomEvent) => {
                  if (e.detail?.topic === 'graph:node:click') {
                    const nodeId = e.detail.payload?.nodeId as string | undefined;
                    if (nodeId) this._onNodeClick(nodeId);
                  }
                }}
              ></pages-graph-canvas>`}
      </div>
    `;
  }

  static override styles = css`
    :host { display: flex; flex-direction: column; height: 100%; }
    .canvas-area { flex: 1; position: relative; }
    .empty { display: flex; align-items: center; justify-content: center;
      height: 100%; color: var(--pages-text-tertiary, #999); font-style: italic; }
  `;
}
```

- [ ] **Step 6: Run tests to verify**

Run: `yarn install && yarn vitest run components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/case-flow-viewer/ tsconfig.base.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add blocks-case-flow-viewer component — read-only case flow DAG

Extends DiagramBaseMixin in readonly mode, composes toGraph + toDecorations
from graph-stencil-case. Runtime decorations, trust score pills, adaptive
decision badges.

Refs #150"
```

### Task 4: Implement BlocksCaseFlowToolbar

**Files:**
- Create: `components/case-flow-viewer/src/blocks-case-flow-toolbar.ts`
- Create: `components/case-flow-viewer/src/blocks-case-flow-toolbar.test.ts`

**Interfaces:**
- Consumes: nothing external — receives stats via properties
- Produces: `<blocks-case-flow-toolbar>` with `nodeCount`, `completedCount`, `runningCount`, `failedCount`, `caseStatus`, `resultTimestamp` properties. Emits `export-svg` and `export-png` events.

- [ ] **Step 1: Write failing test**

Create `components/case-flow-viewer/src/blocks-case-flow-toolbar.test.ts`:

```typescript
// @vitest-environment happy-dom
import { describe, it, expect } from 'vitest';
import { BlocksCaseFlowToolbar } from './blocks-case-flow-toolbar.js';

describe('BlocksCaseFlowToolbar', () => {
  it('stores stat properties', () => {
    const el = new BlocksCaseFlowToolbar();
    el.nodeCount = 5;
    el.completedCount = 2;
    el.runningCount = 1;
    el.failedCount = 1;
    expect(el.nodeCount).toBe(5);
    expect(el.completedCount).toBe(2);
    expect(el.runningCount).toBe(1);
    expect(el.failedCount).toBe(1);
  });

  it('stores caseStatus property', () => {
    const el = new BlocksCaseFlowToolbar();
    el.caseStatus = 'ACTIVE';
    expect(el.caseStatus).toBe('ACTIVE');
  });

  it('sets role="status" and aria-label on connect', () => {
    const el = new BlocksCaseFlowToolbar();
    document.body.appendChild(el);
    expect(el.getAttribute('role')).toBe('status');
    expect(el.getAttribute('aria-label')).toBe('Case flow status');
    expect(el.getAttribute('aria-live')).toBe('polite');
    el.remove();
  });

  it('computes staleness from timestamp', () => {
    const el = new BlocksCaseFlowToolbar();
    const old = new Date(Date.now() - 45_000).toISOString();
    expect(el._computeStaleness(old)).toBeGreaterThanOrEqual(44);
    expect(el._computeStaleness(old)).toBeLessThanOrEqual(46);
  });

  it('computes zero staleness for recent timestamp', () => {
    const el = new BlocksCaseFlowToolbar();
    const recent = new Date(Date.now() - 2_000).toISOString();
    expect(el._computeStaleness(recent)).toBeLessThanOrEqual(3);
  });
});
```

Run: `yarn vitest run components/case-flow-viewer/src/blocks-case-flow-toolbar.test.ts`
Expected: FAIL — module not found

- [ ] **Step 2: Implement BlocksCaseFlowToolbar**

Create `components/case-flow-viewer/src/blocks-case-flow-toolbar.ts`:

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';

@customElement('blocks-case-flow-toolbar')
export class BlocksCaseFlowToolbar extends LitElement {
  @property({ type: Number }) nodeCount = 0;
  @property({ type: Number }) completedCount = 0;
  @property({ type: Number }) runningCount = 0;
  @property({ type: Number }) failedCount = 0;
  @property({ type: String }) caseStatus: string | null = null;
  @property({ type: String }) resultTimestamp: string | null = null;

  @state() private _staleSeconds = 0;
  private _staleTimer: ReturnType<typeof setInterval> | null = null;

  override connectedCallback(): void {
    super.connectedCallback();
    this.setAttribute('role', 'status');
    this.setAttribute('aria-label', 'Case flow status');
    this.setAttribute('aria-live', 'polite');
    this._startStaleTimer();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    this._stopStaleTimer();
  }

  override updated(changed: Map<PropertyKey, unknown>): void {
    if (changed.has('resultTimestamp')) {
      if (this.resultTimestamp != null) this._startStaleTimer();
      else this._stopStaleTimer();
    }
  }

  private _startStaleTimer(): void {
    this._stopStaleTimer();
    if (this.resultTimestamp == null) return;
    this._updateStaleness();
    this._staleTimer = setInterval(() => this._updateStaleness(), 1000);
  }

  private _stopStaleTimer(): void {
    if (this._staleTimer != null) {
      clearInterval(this._staleTimer);
      this._staleTimer = null;
    }
    this._staleSeconds = 0;
  }

  private _updateStaleness(): void {
    if (this.resultTimestamp == null) { this._staleSeconds = 0; return; }
    this._staleSeconds = this._computeStaleness(this.resultTimestamp);
  }

  _computeStaleness(ts: string): number {
    return Math.max(0, Math.floor((Date.now() - new Date(ts).getTime()) / 1000));
  }

  static override styles = css`
    :host { display: flex; align-items: center; gap: 12px; padding: 6px 12px;
      font-family: var(--pages-font-family, sans-serif); font-size: 13px;
      border-bottom: 1px solid var(--pages-border-color, #e5e7eb); }
    .pill { padding: 2px 8px; border-radius: 4px; font-size: 11px; font-weight: 600; }
    .case-status { background: var(--pages-accent-subtle, #e8f0fe); color: var(--pages-accent-color, #1a73e8); }
    .stat { color: var(--pages-text-secondary, #666); }
    .stale { color: #eab308; }
    .export-btn { background: none; border: 1px solid var(--pages-border-color, #e5e7eb);
      border-radius: 4px; padding: 2px 8px; font-size: 11px; cursor: pointer;
      color: var(--pages-text-secondary, #666); }
    .export-btn:hover { background: var(--pages-hover-color, #f3f4f6); }
  `;

  override render() {
    return html`
      ${this.caseStatus ? html`<span class="pill case-status">${this.caseStatus}</span>` : ''}
      <span class="stat">${this.nodeCount} nodes</span>
      ${this.completedCount > 0 ? html`<span class="stat">✓ ${this.completedCount}</span>` : ''}
      ${this.runningCount > 0 ? html`<span class="stat">▶ ${this.runningCount}</span>` : ''}
      ${this.failedCount > 0 ? html`<span class="stat" style="color: #ef4444;">! ${this.failedCount}</span>` : ''}
      ${this._staleSeconds > 30 ? html`<span class="stale">⚠ stale (${this._staleSeconds}s ago)</span>` : ''}
      <span style="flex: 1;"></span>
      <button class="export-btn" @click=${() => this.dispatchEvent(new Event('export-svg'))}>SVG</button>
      <button class="export-btn" @click=${() => this.dispatchEvent(new Event('export-png'))}>PNG</button>
    `;
  }
}
```

- [ ] **Step 3: Run tests to verify**

Run: `yarn vitest run components/case-flow-viewer/src/blocks-case-flow-toolbar.test.ts`
Expected: PASS

- [ ] **Step 4: Run full component test suite**

Run: `yarn vitest run components/case-flow-viewer/`
Expected: PASS — all viewer + toolbar tests green

- [ ] **Step 5: Run typecheck**

Run: `yarn typecheck`
Expected: PASS — no type errors across the workspace

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/case-flow-viewer/src/blocks-case-flow-toolbar.ts components/case-flow-viewer/src/blocks-case-flow-toolbar.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add blocks-case-flow-toolbar — stats, staleness, case status, export

Refs #150"
```

---

## Batch 3: Parallel group support

**Prerequisite:** `ElkLayoutOptions.partitions` must be available in `graph-renderer` (upstream pages change). Add `partitions?: ReadonlyMap<string, number>` to `ElkLayoutOptions`. In `computeElkLayout`, when `partitions` is set: add `'elk.partitioning.activate': 'true'` to root layout options, and in `buildElkNode` set `elkNode.layoutOptions = { 'elk.partitioning.partition': String(index) }` for each partitioned node. User handles this pages change before starting this batch.

### Task 5: Wire parallel groups in the viewer

**Files:**
- Modify: `components/case-flow-viewer/src/blocks-case-flow-viewer.ts`
- Modify: `components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`

**Interfaces:**
- Consumes: `CaseRuntimeState.parallelGroups` from Task 1, `ElkLayoutOptions.partitions` from upstream pages change
- Produces: `_layoutOptions()` override that maps `parallelGroups` to ELK partition constraints

- [ ] **Step 1: Write failing test — parallel group partition mapping**

Add to `blocks-case-flow-viewer.test.ts`:

```typescript
describe('parallel groups', () => {
  it('maps parallelGroups to layout partition options', () => {
    const el = new BlocksCaseFlowViewer();
    el.runtimeState = {
      planItems: [],
      milestones: [],
      timestamp: '2026-09-02T10:00:00Z',
      parallelGroups: [['extract-text', 'classify'], ['validate']],
    };
    const opts = (el as any)._layoutOptions();
    expect(opts.partitions).toBeDefined();
    expect(opts.partitions.get('binding:extract-text')).toBe(0);
    expect(opts.partitions.get('binding:classify')).toBe(0);
    expect(opts.partitions.get('binding:validate')).toBe(1);
  });

  it('returns no partitions when parallelGroups is absent', () => {
    const el = new BlocksCaseFlowViewer();
    el.runtimeState = {
      planItems: [], milestones: [], timestamp: '2026-09-02T10:00:00Z',
    };
    const opts = (el as any)._layoutOptions();
    expect(opts.partitions).toBeUndefined();
  });

  it('returns no partitions when runtimeState is null', () => {
    const el = new BlocksCaseFlowViewer();
    const opts = (el as any)._layoutOptions();
    expect(opts.partitions).toBeUndefined();
  });
});
```

Run: `yarn vitest run components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`
Expected: FAIL — `_layoutOptions` doesn't return partitions

- [ ] **Step 2: Implement _layoutOptions override**

In `blocks-case-flow-viewer.ts`, add the override:

```typescript
protected override _layoutOptions(): ElkLayoutOptions {
  const base = super._layoutOptions();
  if (!this.runtimeState?.parallelGroups || this.runtimeState.parallelGroups.length === 0) {
    return base;
  }
  const partitions = new Map<string, number>();
  for (let i = 0; i < this.runtimeState.parallelGroups.length; i++) {
    for (const bindingName of this.runtimeState.parallelGroups[i]) {
      partitions.set(`binding:${bindingName}`, i);
    }
  }
  return { ...base, partitions };
}
```

Add the import for `ElkLayoutOptions`:

```typescript
import type { ElkLayoutOptions } from '@casehubio/graph-renderer';
```

- [ ] **Step 3: Run tests to verify**

Run: `yarn vitest run components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts`
Expected: PASS

- [ ] **Step 4: Run full test suite**

Run: `yarn test`
Expected: PASS — all tests green across workspace

- [ ] **Step 5: Run typecheck and build**

Run: `yarn typecheck && yarn build`
Expected: PASS — clean build

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/case-flow-viewer/src/blocks-case-flow-viewer.ts components/case-flow-viewer/src/blocks-case-flow-viewer.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: wire parallel group ELK partitioning in case-flow-viewer

Maps runtimeState.parallelGroups to ElkLayoutOptions.partitions for
side-by-side parallel branch layout.

Refs #150"
```

---

## Batch 4: Documentation and CLAUDE.md update

### Task 6: Update CLAUDE.md key directories and run aria-check

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:** None

- [ ] **Step 1: Add case-flow-viewer to CLAUDE.md key directories table**

Add entry to the `## Key Directories` table:

```markdown
| `components/case-flow-viewer/` | Case flow viewer — read-only case definition DAG with runtime decorations (trust score pills, adaptive decision badges, parallel groups). Extends DiagramBaseMixin in readonly mode, composes toGraph + toDecorations from graph-stencil-case. Toolbar with stats, staleness, case status badge, SVG/PNG export. |
```

- [ ] **Step 2: Run aria-check**

Run: `yarn aria-check`
Expected: PASS — new component has ARIA attributes

- [ ] **Step 3: Run full test suite**

Run: `yarn test`
Expected: PASS — all tests green

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "docs: add case-flow-viewer to CLAUDE.md key directories

Refs #150"
```

## References

- [2026-09-02-case-flow-viewer-design.md] — design spec this plan implements
- [components/blocks-dag-viewer/src/blocks-dag-viewer.ts] — analogous HTN viewer pattern
- [components/casehub-diagram/src/casehub-diagram.ts:100-218] — editor using same pipeline
- [packages/graph-stencil-case/src/runtime/runtime-adapter.ts] — toDecorations function
- [packages/graph-stencil-case/src/runtime/types.ts] — CaseRuntimeState type
- [packages/graph-stencil-case/src/stencils/register.ts] — stencil registrations
- [.casehub-packages/packages/pages-diagram-core/src/diagram-base-mixin.ts] — DiagramBaseMixin
- [.casehub-packages/packages/graph-renderer/src/layout/elk-layout.ts] — ELK layout and ElkLayoutOptions
- [PP-20260806-320d50] — stencil package isolation protocol
- [PP-20260713-8ea1af] — component customisation pattern protocol
- [casehubio/blocks-ui#150] — focal issue
- [casehubio/casehub-pages#404] — upstream NodeDecoration.pills
