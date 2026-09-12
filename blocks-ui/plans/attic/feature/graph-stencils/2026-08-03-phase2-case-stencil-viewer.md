# Phase 2: Case Stencil Read-Only Viewer

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — Epic: Visual Diagram Editor — Domain Layer
**Issue group:** #103

**Goal:** Render a CaseDefinition YAML as a directed graph with auto-layout — Workers, Bindings, Milestones, Goals connected by derived edges.

**Architecture:** `CaseAdapter.toGraph()` parses YAML and produces a `GraphModel` (graph-core). A `toReactFlowGraph()` transform converts to React Flow types. Stencil render functions (lit-html templates) are bridged to React components via `createReactNodeType()`. `computeElkLayout()` positions nodes. `<casehub-diagram>` orchestrates and renders via `<pages-graph-canvas>`.

**Tech Stack:** graph-core (model), graph-renderer (canvas, layout, registry), lit-html (stencil templates), React Flow v12 (rendering), yaml npm (parsing), vitest (tests)

## Global Constraints

- TypeScript strict mode with `exactOptionalPropertyTypes: true`
- Stencil templates use inline styles + `--pages-*` CSS custom properties only
- Node type strings: `binding`, `worker`, `milestone`, `goal`, `subcase`
- Node IDs: `<type>:<name>` (e.g., `worker:ocr-worker`). Nameless bindings use index: `binding:_0`
- Edge IDs: `<source-id>--<type>--<target-id>`
- `react` and `react-dom` as peerDependencies in graph-stencil-case (graph-renderer brings them)
- Portal-resolved packages (graph-core, graph-renderer) must be built in `.casehub-packages/` before building

---

### Task 1: CaseAdapter — toGraph() implementation

**Files:**
- Modify: `packages/graph-stencil-case/src/adapter/case-adapter.ts` (rewrite)
- Create: `packages/graph-stencil-case/src/adapter/case-adapter.test.ts`
- Modify: `packages/graph-stencil-case/package.json` (add react peerDeps)

**Interfaces:**
- Consumes: `CaseDefinition` type from `src/types/case-definition.js` (Phase 0), `GraphModel`/`GraphNode`/`GraphEdge`/`createGraph` from `@casehubio/graph-core`
- Produces: `toGraph(yaml: string): GraphModel` — used by Task 4 (casehub-diagram)

- [ ] **Step 1: Add peerDependencies**

Add to `packages/graph-stencil-case/package.json` peerDependencies:
```json
"peerDependencies": {
  "react": "^18.0.0",
  "react-dom": "^18.0.0",
  "@xyflow/react": "^12.0.0"
}
```

Add `@casehubio/graph-renderer` to dependencies:
```json
"@casehubio/graph-renderer": "*"
```

Run `GH_PACKAGES_TOKEN=dummy yarn install`.

- [ ] **Step 2: Write failing tests for toGraph()**

Create `packages/graph-stencil-case/src/adapter/case-adapter.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';
import { toGraph } from './case-adapter.js';

const EXAMPLE_YAML = readFileSync(
  resolve(import.meta.dirname, '../../../../../engine/schema/src/main/resources/examples/document-processing.yaml'),
  'utf-8',
);

describe('toGraph', () => {
  it('creates worker nodes from spec.workers', () => {
    const model = toGraph(EXAMPLE_YAML);
    const workers = model.nodes.filter(n => n.type === 'worker');
    expect(workers).toHaveLength(5);
    expect(workers.map(w => w.id)).toContain('worker:ocr-worker');
    expect(workers[0]!.properties['name']).toBe('ocr-worker');
    expect(workers[0]!.properties['capabilities']).toEqual(['ocr']);
  });

  it('creates binding nodes from spec.bindings', () => {
    const model = toGraph(EXAMPLE_YAML);
    const bindings = model.nodes.filter(n => n.type === 'binding');
    expect(bindings).toHaveLength(6);
    expect(bindings.map(b => b.id)).toContain('binding:extract-text');
  });

  it('creates milestone nodes', () => {
    const model = toGraph(EXAMPLE_YAML);
    const milestones = model.nodes.filter(n => n.type === 'milestone');
    expect(milestones).toHaveLength(3);
    expect(milestones[0]!.properties['name']).toBe('text-extracted');
    expect(milestones[0]!.properties['condition']).toContain('.ocrResult');
  });

  it('creates goal nodes', () => {
    const model = toGraph(EXAMPLE_YAML);
    const goals = model.nodes.filter(n => n.type === 'goal');
    expect(goals).toHaveLength(1);
    expect(goals[0]!.properties['kind']).toBe('success');
  });

  it('derives capability-dispatch edges from binding.capability → worker.capabilities[]', () => {
    const model = toGraph(EXAMPLE_YAML);
    const capEdges = model.edges.filter(e => e.type === 'capability-dispatch');
    expect(capEdges).toHaveLength(6);

    const ocrEdge = capEdges.find(e => e.source === 'binding:extract-text');
    expect(ocrEdge).toBeDefined();
    expect(ocrEdge!.target).toBe('worker:ocr-worker');
  });

  it('creates external nodes for unresolvable capabilities', () => {
    const yamlWithExternal = EXAMPLE_YAML.replace(
      'capabilities: [ "ocr" ]',
      'capabilities: [ "unused-cap" ]',
    );
    const model = toGraph(yamlWithExternal);
    const extNodes = model.nodes.filter(n => n.type === 'external');
    expect(extNodes.length).toBeGreaterThan(0);
    expect(extNodes.some(n => n.id === 'external:ocr')).toBe(true);
  });

  it('carries trigger info in binding properties', () => {
    const model = toGraph(EXAMPLE_YAML);
    const binding = model.nodes.find(n => n.id === 'binding:on-external-document');
    expect(binding).toBeDefined();
    const on = binding!.properties['on'] as Record<string, unknown>;
    expect(on['cloudEvent']).toBeDefined();
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: FAIL — `toGraph` is not exported or returns empty model.

- [ ] **Step 4: Implement toGraph()**

Rewrite `packages/graph-stencil-case/src/adapter/case-adapter.ts`:

```typescript
import { parse as parseYaml } from 'yaml';
import { createGraph } from '@casehubio/graph-core';
import type { GraphNode, GraphEdge, GraphModel } from '@casehubio/graph-core';
import type { CaseDefinition } from '../types/case-definition.js';

export function toGraph(yaml: string): GraphModel {
  const def = parseYaml(yaml) as CaseDefinition;
  const nodes: GraphNode[] = [];
  const edges: GraphEdge[] = [];

  const capabilityToWorker = new Map<string, string>();

  for (const worker of def.spec.workers ?? []) {
    const nodeId = `worker:${worker.name}`;
    nodes.push({
      id: nodeId,
      type: 'worker',
      properties: { ...worker },
    });
    for (const cap of worker.capabilities) {
      capabilityToWorker.set(cap, nodeId);
    }
  }

  for (const cap of def.spec.capabilities ?? []) {
    // capabilities are not separate nodes — they are worker metadata
    // and the connection point for binding→worker edges
  }

  let bindingIndex = 0;
  for (const binding of def.spec.bindings ?? []) {
    const nodeId = binding.name
      ? `binding:${binding.name}`
      : `binding:_${bindingIndex}`;
    nodes.push({
      id: nodeId,
      type: 'binding',
      properties: { ...binding },
    });

    if (binding.capability) {
      const workerNodeId = capabilityToWorker.get(binding.capability);
      if (workerNodeId) {
        edges.push({
          id: `${nodeId}--capability-dispatch--${workerNodeId}`,
          type: 'capability-dispatch',
          source: nodeId,
          target: workerNodeId,
        });
      } else {
        const externalId = `external:${binding.capability}`;
        if (!nodes.some(n => n.id === externalId)) {
          nodes.push({
            id: externalId,
            type: 'external',
            properties: { name: binding.capability },
          });
        }
        edges.push({
          id: `${nodeId}--capability-dispatch--${externalId}`,
          type: 'capability-dispatch',
          source: nodeId,
          target: externalId,
        });
      }
    }

    if (binding.subCase) {
      const subId = `subcase:${binding.subCase.namespace}/${binding.subCase.name}`;
      if (!nodes.some(n => n.id === subId)) {
        nodes.push({
          id: subId,
          type: 'subcase',
          properties: { ...binding.subCase },
        });
      }
      edges.push({
        id: `${nodeId}--subcase-spawn--${subId}`,
        type: 'subcase-spawn',
        source: nodeId,
        target: subId,
      });
    }

    bindingIndex++;
  }

  for (const milestone of def.spec.milestones ?? []) {
    nodes.push({
      id: `milestone:${milestone.name}`,
      type: 'milestone',
      properties: { ...milestone },
    });
  }

  for (const goal of def.spec.goals ?? []) {
    nodes.push({
      id: `goal:${goal.name}`,
      type: 'goal',
      properties: { ...goal },
    });
  }

  return createGraph(nodes, edges);
}
```

- [ ] **Step 5: Update index.ts exports**

Replace `CaseAdapter` class export with `toGraph` function export in `src/index.ts`:

```typescript
export { toGraph } from './adapter/case-adapter.js';
```

Remove the old `CaseAdapter` class export line.

- [ ] **Step 6: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: All 7 adapter tests pass.

- [ ] **Step 7: Commit**

```bash
git -C $PROJECT add packages/graph-stencil-case/src/adapter/ packages/graph-stencil-case/src/index.ts packages/graph-stencil-case/package.json yarn.lock
git -C $PROJECT commit -m "feat(#103): CaseAdapter.toGraph() — YAML to GraphModel with edge derivation"
```

---

### Task 2: React Flow Transform

**Files:**
- Create: `packages/graph-stencil-case/src/adapter/react-flow-transform.ts`
- Create: `packages/graph-stencil-case/src/adapter/react-flow-transform.test.ts`

**Interfaces:**
- Consumes: `GraphModel` from graph-core, `Node`/`Edge` from `@xyflow/react`
- Produces: `toReactFlowGraph(model: GraphModel): { nodes: Node[]; edges: Edge[] }` — used by Task 4

- [ ] **Step 1: Write failing tests**

Create `packages/graph-stencil-case/src/adapter/react-flow-transform.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import type { GraphModel } from '@casehubio/graph-core';
import { toReactFlowGraph } from './react-flow-transform.js';

const SAMPLE_MODEL: GraphModel = {
  nodes: [
    { id: 'worker:ocr', type: 'worker', properties: { name: 'ocr', capabilities: ['ocr'] } },
    { id: 'binding:scan', type: 'binding', properties: { name: 'scan', capability: 'ocr' } },
  ],
  edges: [
    { id: 'binding:scan--capability-dispatch--worker:ocr', type: 'capability-dispatch', source: 'binding:scan', target: 'worker:ocr' },
  ],
};

describe('toReactFlowGraph', () => {
  it('maps GraphNodes to React Flow Nodes with type and data', () => {
    const { nodes } = toReactFlowGraph(SAMPLE_MODEL);
    expect(nodes).toHaveLength(2);

    const workerNode = nodes.find(n => n.id === 'worker:ocr')!;
    expect(workerNode.type).toBe('worker');
    expect(workerNode.data['name']).toBe('ocr');
    expect(workerNode.position).toEqual({ x: 0, y: 0 });
  });

  it('maps GraphEdges to React Flow Edges', () => {
    const { edges } = toReactFlowGraph(SAMPLE_MODEL);
    expect(edges).toHaveLength(1);
    expect(edges[0]!.source).toBe('binding:scan');
    expect(edges[0]!.target).toBe('worker:ocr');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement toReactFlowGraph()**

Create `packages/graph-stencil-case/src/adapter/react-flow-transform.ts`:

```typescript
import type { Node, Edge } from '@xyflow/react';
import type { GraphModel } from '@casehubio/graph-core';

export function toReactFlowGraph(model: GraphModel): { nodes: Node[]; edges: Edge[] } {
  const nodes: Node[] = model.nodes.map(gn => ({
    id: gn.id,
    type: gn.type,
    position: { x: 0, y: 0 },
    data: { ...gn.properties },
    ...(gn.parentId !== undefined ? { parentId: gn.parentId } : {}),
  }));

  const edges: Edge[] = model.edges.map(ge => ({
    id: ge.id,
    source: ge.source,
    target: ge.target,
    type: ge.type,
  }));

  return { nodes, edges };
}
```

- [ ] **Step 4: Add export to index.ts**

Add to `src/index.ts`:
```typescript
export { toReactFlowGraph } from './adapter/react-flow-transform.js';
```

- [ ] **Step 5: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add packages/graph-stencil-case/src/adapter/react-flow-transform.ts packages/graph-stencil-case/src/adapter/react-flow-transform.test.ts packages/graph-stencil-case/src/index.ts
git -C $PROJECT commit -m "feat(#103): toReactFlowGraph — GraphModel to React Flow nodes/edges"
```

---

### Task 3: Lit→React Bridge + Stencil Render Functions

**Files:**
- Create: `packages/graph-stencil-case/src/bridge/create-react-node-type.tsx`
- Modify: `packages/graph-stencil-case/src/stencils/index.ts` (rewrite)
- Create: `packages/graph-stencil-case/src/stencils/binding.ts`
- Create: `packages/graph-stencil-case/src/stencils/worker.ts`
- Create: `packages/graph-stencil-case/src/stencils/milestone.ts`
- Create: `packages/graph-stencil-case/src/stencils/goal.ts`
- Create: `packages/graph-stencil-case/src/stencils/subcase.ts`
- Create: `packages/graph-stencil-case/src/stencils/register.ts`
- Create: `packages/graph-stencil-case/src/stencils/stencils.test.ts`
- Modify: `packages/graph-stencil-case/tsconfig.json` (add jsx for .tsx)

**Interfaces:**
- Consumes: `registerGrammar`/`StencilGrammar` from graph-core, `registerNodeType`/`NodeTypeDescriptor` from graph-renderer, `TemplateResult` from lit-html
- Produces: `registerCaseStencils(): void` — called by Task 4 (casehub-diagram)

- [ ] **Step 1: Add jsx support to tsconfig**

Add `"jsx": "react-jsx"` to `packages/graph-stencil-case/tsconfig.json` compilerOptions (already in tsconfig.base.json, but explicit here for the .tsx files).

Verify `tsconfig.build.json` includes `.tsx` in its scope:
```json
{ "extends": "./tsconfig.json", "include": ["src"] }
```

- [ ] **Step 2: Create the Lit→React bridge**

Create `packages/graph-stencil-case/src/bridge/create-react-node-type.tsx`:

```tsx
import React, { useRef, useEffect } from 'react';
import { render as litRender, type TemplateResult } from 'lit-html';
import type { NodeProps } from '@xyflow/react';

export type StencilRenderFn = (data: Record<string, unknown>) => TemplateResult;

export function createReactNodeType(
  renderFn: StencilRenderFn,
): React.ComponentType<NodeProps> {
  return function LitNodeWrapper({ data }: NodeProps) {
    const containerRef = useRef<HTMLDivElement>(null);

    useEffect(() => {
      if (containerRef.current) {
        litRender(renderFn(data as Record<string, unknown>), containerRef.current);
      }
    }, [data]);

    return <div ref={containerRef} />;
  };
}
```

- [ ] **Step 3: Write stencil render functions**

Create `packages/graph-stencil-case/src/stencils/binding.ts`:

```typescript
import { html, type TemplateResult } from 'lit-html';
import type { StencilGrammar } from '@casehubio/graph-core';

export const bindingGrammar: StencilGrammar = {
  type: 'binding',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: ['worker', 'milestone'] },
    outbound: { min: 0, max: 1, allowedTo: ['worker', 'subcase', 'external'] },
  },
};

function triggerLabel(on: Record<string, unknown> | undefined): string {
  if (!on) return '?';
  if (on['contextChange']) return 'ctx';
  if (on['cloudEvent']) return 'event';
  if (on['schedule']) return 'sched';
  if (on['scopeActivated']) return 'scope';
  return '?';
}

function targetLabel(data: Record<string, unknown>): string {
  if (data['capability']) return String(data['capability']);
  if (data['subCase']) return 'subcase';
  if (data['humanTask']) return 'task';
  return '?';
}

export function renderBinding(data: Record<string, unknown>): TemplateResult {
  const name = String(data['name'] ?? '');
  const trigger = triggerLabel(data['on'] as Record<string, unknown> | undefined);
  const target = targetLabel(data);
  const when = data['when'] ? String(data['when']).slice(0, 40) : '';

  return html`
    <div style="padding: 8px 12px; border-radius: 8px; border: 2px solid var(--pages-border-color, #ccc); background: var(--pages-surface-color, #fff); min-width: 180px; font-family: var(--pages-font-family, sans-serif); font-size: 13px;">
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

Create `packages/graph-stencil-case/src/stencils/worker.ts`:

```typescript
import { html, type TemplateResult } from 'lit-html';
import type { StencilGrammar } from '@casehubio/graph-core';

export const workerGrammar: StencilGrammar = {
  type: 'worker',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: ['binding'] },
    outbound: { min: 0, max: Infinity, allowedTo: ['binding'] },
  },
};

export function renderWorker(data: Record<string, unknown>): TemplateResult {
  const name = String(data['name'] ?? '');
  const caps = (data['capabilities'] as string[] | undefined) ?? [];
  const desc = data['description'] ? String(data['description']).slice(0, 60) : '';

  return html`
    <div style="padding: 10px 14px; border: 2px solid var(--pages-border-strong, #888); background: var(--pages-surface-raised, #f8f8f8); min-width: 200px; font-family: var(--pages-font-family, sans-serif); font-size: 13px;">
      <div style="font-weight: 700; color: var(--pages-text-color, #333); margin-bottom: 4px;">${name}</div>
      <div style="color: var(--pages-text-secondary, #666); font-size: 11px;">${caps.join(', ')}</div>
      ${desc ? html`<div style="color: var(--pages-text-tertiary, #999); font-size: 11px; margin-top: 2px;">${desc}</div>` : ''}
    </div>
  `;
}
```

Create `packages/graph-stencil-case/src/stencils/milestone.ts`:

```typescript
import { html, type TemplateResult } from 'lit-html';
import type { StencilGrammar } from '@casehubio/graph-core';

export const milestoneGrammar: StencilGrammar = {
  type: 'milestone',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: ['binding', 'goal'] },
    outbound: { min: 0, max: Infinity, allowedTo: ['binding'] },
  },
};

export function renderMilestone(data: Record<string, unknown>): TemplateResult {
  const name = String(data['name'] ?? '');
  const sla = data['slaDuration'] ? String(data['slaDuration']) : '';

  return html`
    <div style="padding: 10px 16px; background: var(--pages-surface-color, #fff); border: 2px solid var(--pages-warning-color, #f9a825); min-width: 140px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; transform: rotate(45deg); display: flex; align-items: center; justify-content: center;">
      <div style="transform: rotate(-45deg); text-align: center;">
        <div style="font-weight: 600; color: var(--pages-text-color, #333);">◆ ${name}</div>
        ${sla ? html`<div style="font-size: 11px; color: var(--pages-warning-color, #f9a825);">${sla}</div>` : ''}
      </div>
    </div>
  `;
}
```

Create `packages/graph-stencil-case/src/stencils/goal.ts`:

```typescript
import { html, type TemplateResult } from 'lit-html';
import type { StencilGrammar } from '@casehubio/graph-core';

export const goalGrammar: StencilGrammar = {
  type: 'goal',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: ['milestone', 'binding'] },
    outbound: { min: 0, max: 0, allowedTo: [] },
  },
};

const KIND_COLORS: Record<string, string> = {
  success: '#2e7d32',
  failure: '#c62828',
};

export function renderGoal(data: Record<string, unknown>): TemplateResult {
  const name = String(data['name'] ?? '');
  const kind = String(data['kind'] ?? 'success');
  const color = KIND_COLORS[kind] ?? 'var(--pages-accent-color, #1a73e8)';

  return html`
    <div style="padding: 10px 18px; background: var(--pages-surface-color, #fff); border: 3px solid ${color}; min-width: 140px; font-family: var(--pages-font-family, sans-serif); font-size: 13px; clip-path: polygon(25% 0%, 75% 0%, 100% 50%, 75% 100%, 25% 100%, 0% 50%); display: flex; align-items: center; justify-content: center; min-height: 60px;">
      <div style="text-align: center;">
        <div style="font-weight: 700; color: ${color};">⬡ ${name}</div>
        <div style="font-size: 11px; color: ${color}; text-transform: uppercase;">${kind}</div>
      </div>
    </div>
  `;
}
```

Create `packages/graph-stencil-case/src/stencils/subcase.ts`:

```typescript
import { html, type TemplateResult } from 'lit-html';
import type { StencilGrammar } from '@casehubio/graph-core';

export const subcaseGrammar: StencilGrammar = {
  type: 'subcase',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: ['binding'] },
    outbound: { min: 0, max: 0, allowedTo: [] },
  },
};

export function renderSubCase(data: Record<string, unknown>): TemplateResult {
  const ns = String(data['namespace'] ?? '');
  const name = String(data['name'] ?? '');
  const version = String(data['version'] ?? '');
  const groupId = data['groupId'] as string | undefined;
  const total = data['totalInGroup'] as number | undefined;
  const required = data['requiredCount'] as number | undefined;

  return html`
    <div style="padding: 8px 12px; border: 3px double var(--pages-border-strong, #888); background: var(--pages-surface-color, #fff); min-width: 180px; font-family: var(--pages-font-family, sans-serif); font-size: 13px;">
      <div style="font-weight: 600; color: var(--pages-text-color, #333);">${name}</div>
      <div style="color: var(--pages-text-secondary, #666); font-size: 11px;">${ns} v${version}</div>
      ${groupId ? html`<div style="font-size: 11px; color: var(--pages-accent-color, #1a73e8); margin-top: 2px;">${required ?? total}/${total} (${groupId})</div>` : ''}
    </div>
  `;
}
```

- [ ] **Step 4: Create registration function**

Create `packages/graph-stencil-case/src/stencils/register.ts`:

```typescript
import { registerGrammar } from '@casehubio/graph-core';
import { registerNodeType } from '@casehubio/graph-renderer';
import { createReactNodeType } from '../bridge/create-react-node-type.js';
import { bindingGrammar, renderBinding } from './binding.js';
import { workerGrammar, renderWorker } from './worker.js';
import { milestoneGrammar, renderMilestone } from './milestone.js';
import { goalGrammar, renderGoal } from './goal.js';
import { subcaseGrammar, renderSubCase } from './subcase.js';

let registered = false;

export function registerCaseStencils(): void {
  if (registered) return;
  registered = true;

  const stencils = [
    { grammar: bindingGrammar, render: renderBinding },
    { grammar: workerGrammar, render: renderWorker },
    { grammar: milestoneGrammar, render: renderMilestone },
    { grammar: goalGrammar, render: renderGoal },
    { grammar: subcaseGrammar, render: renderSubCase },
  ];

  for (const s of stencils) {
    registerGrammar(s.grammar);
    registerNodeType({
      type: s.grammar.type,
      component: createReactNodeType(s.render),
    });
  }
}
```

- [ ] **Step 5: Update stencils/index.ts**

Rewrite `packages/graph-stencil-case/src/stencils/index.ts`:

```typescript
export { registerCaseStencils } from './register.js';
export { bindingGrammar, renderBinding } from './binding.js';
export { workerGrammar, renderWorker } from './worker.js';
export { milestoneGrammar, renderMilestone } from './milestone.js';
export { goalGrammar, renderGoal } from './goal.js';
export { subcaseGrammar, renderSubCase } from './subcase.js';
```

- [ ] **Step 6: Write stencil render tests**

Create `packages/graph-stencil-case/src/stencils/stencils.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { renderBinding } from './binding.js';
import { renderWorker } from './worker.js';
import { renderMilestone } from './milestone.js';
import { renderGoal } from './goal.js';
import { renderSubCase } from './subcase.js';

describe('stencil render functions', () => {
  it('renderBinding returns TemplateResult with name and trigger', () => {
    const result = renderBinding({
      name: 'extract-text',
      capability: 'ocr',
      on: { contextChange: { filter: '.doc != null' } },
    });
    expect(result).toBeDefined();
    expect(result.strings).toBeDefined();
  });

  it('renderWorker returns TemplateResult with capabilities', () => {
    const result = renderWorker({
      name: 'ocr-worker',
      capabilities: ['ocr'],
      description: 'Extracts text',
    });
    expect(result).toBeDefined();
  });

  it('renderMilestone returns TemplateResult with name', () => {
    const result = renderMilestone({
      name: 'text-extracted',
      condition: '.ocrResult != null',
    });
    expect(result).toBeDefined();
  });

  it('renderGoal returns TemplateResult with kind', () => {
    const result = renderGoal({
      name: 'processingComplete',
      kind: 'success',
      condition: '.done',
    });
    expect(result).toBeDefined();
  });

  it('renderSubCase returns TemplateResult with namespace/name', () => {
    const result = renderSubCase({
      namespace: 'casehub',
      name: 'child-case',
      version: '1.0.0',
    });
    expect(result).toBeDefined();
  });
});
```

- [ ] **Step 7: Update package index.ts**

Update `src/index.ts` to export the registration function:

```typescript
export { toGraph } from './adapter/case-adapter.js';
export { toReactFlowGraph } from './adapter/react-flow-transform.js';
export { registerCaseStencils } from './stencils/index.js';
export { renderBinding, renderWorker, renderMilestone, renderGoal, renderSubCase } from './stencils/index.js';
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

- [ ] **Step 8: Run all tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: All tests pass (adapter + transform + stencils + Phase 0 type tests).

- [ ] **Step 9: Commit**

```bash
git -C $PROJECT add packages/graph-stencil-case/src/bridge/ packages/graph-stencil-case/src/stencils/ packages/graph-stencil-case/src/index.ts packages/graph-stencil-case/tsconfig.json
git -C $PROJECT commit -m "feat(#103): case stencil render functions + Lit→React bridge + registration"
```

---

### Task 4: casehub-diagram Component + Example Page

**Files:**
- Create: `components/casehub-diagram/package.json`
- Create: `components/casehub-diagram/tsconfig.json`
- Create: `components/casehub-diagram/tsconfig.build.json`
- Create: `components/casehub-diagram/src/casehub-diagram.ts`
- Create: `components/casehub-diagram/src/casehub-diagram.test.ts`
- Create: `components/casehub-diagram/examples/index.html`

**Interfaces:**
- Consumes: `toGraph` (Task 1), `toReactFlowGraph` (Task 2), `registerCaseStencils` (Task 3), `computeElkLayout` + `GraphCanvas` from graph-renderer
- Produces: `<casehub-diagram>` custom element with `yaml` and `src` properties

- [ ] **Step 1: Create package scaffold**

Create `components/casehub-diagram/package.json`:

```json
{
  "name": "@casehubio/blocks-ui-casehub-diagram",
  "version": "0.1.0",
  "description": "CaseHub visual diagram editor component",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/blocks-ui.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/casehub-diagram.js",
  "types": "dist/casehub-diagram.d.ts",
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
    "@casehubio/graph-stencil-case": "workspace:*",
    "@casehubio/graph-core": "*",
    "@casehubio/graph-renderer": "*",
    "lit": "^3.3.3",
    "yaml": "^2.7.0"
  },
  "license": "Apache-2.0"
}
```

Create `components/casehub-diagram/tsconfig.json`:
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
    { "path": "../../packages/graph-stencil-case" }
  ]
}
```

Create `components/casehub-diagram/tsconfig.build.json`:
```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

Register in workspace root `package.json` workspaces if needed (check if glob already covers `components/*`).

Run `GH_PACKAGES_TOKEN=dummy yarn install`.

- [ ] **Step 2: Write the component**

Create `components/casehub-diagram/src/casehub-diagram.ts`:

```typescript
import { LitElement, html } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { toGraph, toReactFlowGraph, registerCaseStencils } from '@casehubio/graph-stencil-case';
import { computeElkLayout } from '@casehubio/graph-renderer';
import type { GraphModel } from '@casehubio/graph-core';
import type { Node, Edge } from '@xyflow/react';
import '@casehubio/graph-renderer';

@customElement('casehub-diagram')
export class CasehubDiagram extends LitElement {
  @property() yaml = '';
  @property() src = '';

  @state() private _nodes: Node[] = [];
  @state() private _edges: Edge[] = [];
  @state() private _error = '';

  override createRenderRoot(): HTMLElement {
    return this;
  }

  override connectedCallback(): void {
    super.connectedCallback();
    registerCaseStencils();
  }

  override async updated(changed: Map<string, unknown>): Promise<void> {
    if (changed.has('yaml') && this.yaml) {
      await this._renderGraph(this.yaml);
    }
    if (changed.has('src') && this.src) {
      try {
        const response = await fetch(this.src);
        const text = await response.text();
        await this._renderGraph(text);
      } catch (e) {
        this._error = `Failed to fetch ${this.src}: ${e}`;
      }
    }
  }

  private async _renderGraph(yamlStr: string): Promise<void> {
    try {
      this._error = '';
      const model: GraphModel = toGraph(yamlStr);
      const { nodes, edges } = toReactFlowGraph(model);
      this._nodes = await computeElkLayout(nodes, edges, { direction: 'DOWN', spacing: 60 });
      this._edges = edges;
    } catch (e) {
      this._error = String(e);
    }
  }

  override render() {
    if (this._error) {
      return html`<div style="color: red; padding: 16px;">${this._error}</div>`;
    }
    return html`
      <pages-graph-canvas
        .nodes=${this._nodes}
        .edges=${this._edges}
        style="width: 100%; height: 100%;"
      ></pages-graph-canvas>
    `;
  }
}
```

- [ ] **Step 3: Write component test**

Create `components/casehub-diagram/src/casehub-diagram.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { toGraph, toReactFlowGraph } from '@casehubio/graph-stencil-case';
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';

const EXAMPLE_YAML = readFileSync(
  resolve(import.meta.dirname, '../../../../engine/schema/src/main/resources/examples/document-processing.yaml'),
  'utf-8',
);

describe('casehub-diagram integration', () => {
  it('end-to-end: YAML → GraphModel → React Flow nodes', () => {
    const model = toGraph(EXAMPLE_YAML);
    const { nodes, edges } = toReactFlowGraph(model);

    expect(nodes.length).toBeGreaterThan(0);
    expect(edges.length).toBeGreaterThan(0);

    const workerNodes = nodes.filter(n => n.type === 'worker');
    expect(workerNodes).toHaveLength(5);

    const bindingNodes = nodes.filter(n => n.type === 'binding');
    expect(bindingNodes).toHaveLength(6);

    expect(edges.every(e => nodes.some(n => n.id === e.source))).toBe(true);
    expect(edges.every(e => nodes.some(n => n.id === e.target))).toBe(true);
  });
});
```

- [ ] **Step 4: Create example page**

Create `components/casehub-diagram/examples/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <title>CaseHub Diagram — document-processing example</title>
  <style>
    body { margin: 0; font-family: sans-serif; }
    casehub-diagram { display: block; width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <casehub-diagram id="diagram"></casehub-diagram>
  <script type="module">
    import '../src/casehub-diagram.js';

    const yaml = await fetch('../../../../engine/schema/src/main/resources/examples/document-processing.yaml')
      .then(r => r.text())
      .catch(() => null);

    if (yaml) {
      document.getElementById('diagram').yaml = yaml;
    } else {
      document.getElementById('diagram').innerHTML =
        '<p style="padding:16px">Could not load document-processing.yaml. Run from the repo root.</p>';
    }
  </script>
</body>
</html>
```

- [ ] **Step 5: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```
Expected: PASS — integration test verifies YAML → nodes/edges pipeline.

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add components/casehub-diagram/
git -C $PROJECT commit -m "feat(#103): casehub-diagram component — read-only viewer with ELK layout"
```

---

### Task 5: CLAUDE.md + Full Build Verification

**Files:**
- Modify: `CLAUDE.md` (add casehub-diagram to Key Directories)

**Interfaces:**
- Consumes: All previous tasks
- Produces: Updated documentation, verified build

- [ ] **Step 1: Update CLAUDE.md Key Directories**

Add entry for casehub-diagram:

```markdown
| `components/casehub-diagram/` | CaseHub visual diagram — viewer component for CaseDefinition YAML. Orchestrates graph-stencil-case adapter + stencils, pages-graph-canvas rendering, ELK auto-layout. Phase 2: viewer only. |
```

- [ ] **Step 2: Full build**

```bash
GH_PACKAGES_TOKEN=dummy yarn build
```

Verify graph-stencil-case and casehub-diagram both build. Pre-existing failures in other packages (document-workbench, channel-activity) are not blockers.

- [ ] **Step 3: Full test suite**

```bash
GH_PACKAGES_TOKEN=dummy yarn test
```

Verify graph-stencil-case and casehub-diagram tests pass. Pre-existing failures in channel-activity are not blockers.

- [ ] **Step 4: Commit**

```bash
git -C $PROJECT add CLAUDE.md
git -C $PROJECT commit -m "docs(#103): add casehub-diagram to CLAUDE.md Key Directories"
```
