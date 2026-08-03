# graph-core: Graph Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows
> TDD (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #266 — graph-core: graph model — nodes, edges, containment tree
**Issue group:** #266, #267, #268, #269, #270

**Goal:** Create the `@casehubio/graph-core` package with typed graph model,
validated construction, containment traversal, and edge/lookup queries.

**Architecture:** Plain TypeScript interfaces + standalone pure functions.
Zero dependencies. Immutable via `readonly` + spread-copy. Array storage
with O(n) lookup — acceptable at tens-to-hundreds scale.

**Tech Stack:** TypeScript 5, Vitest 3, Yarn 4 workspaces

## Global Constraints

- Zero runtime dependencies — pure TypeScript only
- All fields `readonly`, all arrays `readonly`
- No `any` types — strict mode throughout
- `graph-*` namespace (no `pages-` prefix)
- Co-located tests (`*.test.ts` next to source)
- Node test environment (no jsdom — no DOM interaction)

---

### Task 1: Package scaffold

**Files:**
- Create: `packages/graph-core/package.json`
- Create: `packages/graph-core/tsconfig.json`
- Create: `packages/graph-core/tsconfig.build.json`
- Create: `packages/graph-core/vitest.config.ts`
- Create: `packages/graph-core/src/index.ts`
- Create: `packages/graph-core/src/model.ts`

**Interfaces:**
- Consumes: `@casehubio/pages-tsconfig` (shared tsconfig base)
- Produces: `GraphNode`, `GraphEdge`, `GraphModel` interfaces

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/graph-core",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "clean": "rimraf dist"
  },
  "devDependencies": {
    "@casehubio/pages-tsconfig": "workspace:*",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "@casehubio/pages-tsconfig/tsconfig.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": ".typecheck",
    "emitDeclarationOnly": true
  },
  "include": ["src"],
  "references": []
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "emitDeclarationOnly": false,
    "composite": false
  },
  "exclude": ["**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    include: ['src/**/*.test.ts'],
  },
});
```

- [ ] **Step 5: Create model.ts with type interfaces**

```typescript
export interface GraphNode {
  readonly id: string;
  readonly type: string;
  readonly parentId?: string;
  readonly properties: Readonly<Record<string, unknown>>;
}

export interface GraphEdge {
  readonly id: string;
  readonly type: string;
  readonly source: string;
  readonly target: string;
  readonly properties?: Readonly<Record<string, unknown>>;
}

export interface GraphModel {
  readonly nodes: readonly GraphNode[];
  readonly edges: readonly GraphEdge[];
  readonly metadata?: Readonly<Record<string, unknown>>;
}
```

- [ ] **Step 6: Create index.ts barrel**

```typescript
export type { GraphNode, GraphEdge, GraphModel } from './model.js';
```

- [ ] **Step 7: Install dependencies and verify build**

Run: `yarn install`
Run: `yarn workspace @casehubio/graph-core run typecheck`
Expected: success, no errors

- [ ] **Step 8: Commit**

```bash
git add packages/graph-core/
git commit -m "feat(#266): scaffold @casehubio/graph-core package with model types

Refs #266"
```

---

### Task 2: Model construction — createGraph + validateGraph

**Files:**
- Create: `packages/graph-core/src/graph.ts`
- Create: `packages/graph-core/src/graph.test.ts`
- Modify: `packages/graph-core/src/index.ts`

**Interfaces:**
- Consumes: `GraphNode`, `GraphEdge`, `GraphModel` from `model.ts`
- Produces: `createGraph(nodes, edges, metadata?) → GraphModel`,
  `validateGraph(nodes, edges) → readonly GraphViolation[]`,
  `GraphValidationError` (extends `Error`, has `violations: readonly GraphViolation[]`),
  `GraphViolation` (has `rule`, `message`, optional `nodeId`, optional `edgeId`)

- [ ] **Step 1: Write failing tests for createGraph**

```typescript
import { describe, it, expect } from 'vitest';
import { createGraph, validateGraph, GraphValidationError } from './graph.js';
import type { GraphNode, GraphEdge, GraphViolation } from './index.js';

function node(id: string, type = 'default', parentId?: string): GraphNode {
  return { id, type, parentId, properties: {} };
}

function edge(id: string, source: string, target: string, type = 'default'): GraphEdge {
  return { id, type, source, target };
}

describe('createGraph', () => {
  it('creates a model from valid nodes and edges', () => {
    const n = [node('a', 'x'), node('b', 'y')];
    const e = [edge('e1', 'a', 'b')];
    const model = createGraph(n, e, { name: 'test' });

    expect(model.nodes).toEqual(n);
    expect(model.edges).toEqual(e);
    expect(model.metadata).toEqual({ name: 'test' });
  });

  it('creates an empty model', () => {
    const model = createGraph([], []);
    expect(model.nodes).toEqual([]);
    expect(model.edges).toEqual([]);
  });

  it('creates a single-node model with no edges', () => {
    const model = createGraph([node('a')], []);
    expect(model.nodes).toHaveLength(1);
    expect(model.edges).toHaveLength(0);
  });

  it('accepts self-loop edges', () => {
    const model = createGraph([node('a')], [edge('e1', 'a', 'a')]);
    expect(model.edges).toHaveLength(1);
  });

  it('rejects duplicate node IDs', () => {
    expect(() => createGraph([node('a'), node('a')], []))
      .toThrow(GraphValidationError);
  });

  it('rejects duplicate edge IDs', () => {
    const n = [node('a'), node('b')];
    expect(() => createGraph(n, [edge('e1', 'a', 'b'), edge('e1', 'b', 'a')]))
      .toThrow(GraphValidationError);
  });

  it('rejects dangling edge source', () => {
    expect(() => createGraph([node('a')], [edge('e1', 'missing', 'a')]))
      .toThrow(GraphValidationError);
  });

  it('rejects dangling edge target', () => {
    expect(() => createGraph([node('a')], [edge('e1', 'a', 'missing')]))
      .toThrow(GraphValidationError);
  });

  it('rejects invalid parentId', () => {
    expect(() => createGraph([node('a', 'x', 'missing')], []))
      .toThrow(GraphValidationError);
  });

  it('rejects self-referencing parentId', () => {
    expect(() => createGraph([node('a', 'x', 'a')], []))
      .toThrow(GraphValidationError);
  });

  it('rejects containment cycle', () => {
    const n = [
      { id: 'a', type: 'x', parentId: 'b', properties: {} },
      { id: 'b', type: 'x', parentId: 'a', properties: {} },
    ];
    expect(() => createGraph(n, [])).toThrow(GraphValidationError);
  });

  it('rejects empty node ID', () => {
    expect(() => createGraph([node('')], []))
      .toThrow(GraphValidationError);
  });

  it('rejects whitespace-only node ID', () => {
    expect(() => createGraph([node('  ')], []))
      .toThrow(GraphValidationError);
  });

  it('rejects empty edge ID', () => {
    const n = [node('a'), node('b')];
    expect(() => createGraph(n, [edge('', 'a', 'b')]))
      .toThrow(GraphValidationError);
  });

  it('collects multiple violations in single throw', () => {
    try {
      createGraph([node('a'), node('a'), node('')], []);
      expect.fail('should throw');
    } catch (err) {
      expect(err).toBeInstanceOf(GraphValidationError);
      const violations = (err as GraphValidationError).violations;
      expect(violations.length).toBeGreaterThanOrEqual(2);
    }
  });

  it('provides structured violation data', () => {
    try {
      createGraph([node('a'), node('a')], []);
      expect.fail('should throw');
    } catch (err) {
      const v = (err as GraphValidationError).violations[0]!;
      expect(v.rule).toBe('duplicate_node_id');
      expect(v.message).toContain('a');
      expect(v.nodeId).toBe('a');
    }
  });
});

describe('validateGraph', () => {
  it('returns empty array for valid model', () => {
    const violations = validateGraph(
      [node('a'), node('b')],
      [edge('e1', 'a', 'b')],
    );
    expect(violations).toEqual([]);
  });

  it('returns all violations without throwing', () => {
    const violations = validateGraph(
      [node('a'), node('a'), node('')],
      [],
    );
    expect(violations.length).toBeGreaterThanOrEqual(2);
    expect(violations.every((v: GraphViolation) => v.rule && v.message)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: FAIL — `graph.js` does not exist

- [ ] **Step 3: Implement graph.ts**

```typescript
import type { GraphNode, GraphEdge, GraphModel } from './model.js';

export interface GraphViolation {
  readonly rule:
    | 'empty_id'
    | 'duplicate_node_id'
    | 'duplicate_edge_id'
    | 'dangling_edge'
    | 'invalid_parent'
    | 'self_parent'
    | 'containment_cycle';
  readonly message: string;
  readonly nodeId?: string;
  readonly edgeId?: string;
}

export class GraphValidationError extends Error {
  readonly violations: readonly GraphViolation[];

  constructor(violations: readonly GraphViolation[]) {
    const summary = violations.map(v => v.message).join('; ');
    super(`Invalid graph: ${summary}`);
    this.name = 'GraphValidationError';
    this.violations = violations;
  }
}

export function validateGraph(
  nodes: readonly GraphNode[],
  edges: readonly GraphEdge[],
): readonly GraphViolation[] {
  const violations: GraphViolation[] = [];
  const nodeIds = new Set<string>();

  for (const node of nodes) {
    if (!node.id.trim()) {
      violations.push({
        rule: 'empty_id',
        message: `Node has empty or whitespace-only ID`,
        nodeId: node.id,
      });
      continue;
    }
    if (nodeIds.has(node.id)) {
      violations.push({
        rule: 'duplicate_node_id',
        message: `Duplicate node ID '${node.id}'`,
        nodeId: node.id,
      });
    }
    nodeIds.add(node.id);
  }

  const edgeIds = new Set<string>();
  for (const edge of edges) {
    if (!edge.id.trim()) {
      violations.push({
        rule: 'empty_id',
        message: `Edge has empty or whitespace-only ID`,
        edgeId: edge.id,
      });
      continue;
    }
    if (edgeIds.has(edge.id)) {
      violations.push({
        rule: 'duplicate_edge_id',
        message: `Duplicate edge ID '${edge.id}'`,
        edgeId: edge.id,
      });
    }
    edgeIds.add(edge.id);

    if (!nodeIds.has(edge.source)) {
      violations.push({
        rule: 'dangling_edge',
        message: `Edge '${edge.id}' references non-existent source '${edge.source}'`,
        edgeId: edge.id,
        nodeId: edge.source,
      });
    }
    if (!nodeIds.has(edge.target)) {
      violations.push({
        rule: 'dangling_edge',
        message: `Edge '${edge.id}' references non-existent target '${edge.target}'`,
        edgeId: edge.id,
        nodeId: edge.target,
      });
    }
  }

  for (const node of nodes) {
    if (node.parentId === undefined) continue;
    if (node.parentId === node.id) {
      violations.push({
        rule: 'self_parent',
        message: `Node '${node.id}' references itself as parent`,
        nodeId: node.id,
      });
    } else if (!nodeIds.has(node.parentId)) {
      violations.push({
        rule: 'invalid_parent',
        message: `Node '${node.id}' references non-existent parent '${node.parentId}'`,
        nodeId: node.id,
      });
    }
  }

  // Containment cycle detection (skip nodes already flagged for self-parent or invalid parent)
  const flaggedNodes = new Set(
    violations
      .filter(v => v.rule === 'self_parent' || v.rule === 'invalid_parent')
      .map(v => v.nodeId),
  );
  const parentMap = new Map<string, string>();
  for (const node of nodes) {
    if (node.parentId !== undefined && !flaggedNodes.has(node.id)) {
      parentMap.set(node.id, node.parentId);
    }
  }

  const visited = new Set<string>();
  const inStack = new Set<string>();
  for (const nodeId of parentMap.keys()) {
    if (visited.has(nodeId)) continue;
    const path: string[] = [];
    let current: string | undefined = nodeId;
    while (current !== undefined && !visited.has(current) && parentMap.has(current)) {
      if (inStack.has(current)) {
        violations.push({
          rule: 'containment_cycle',
          message: `Containment cycle detected involving node '${current}'`,
          nodeId: current,
        });
        break;
      }
      inStack.add(current);
      path.push(current);
      current = parentMap.get(current);
    }
    for (const p of path) {
      visited.add(p);
      inStack.delete(p);
    }
  }

  return violations;
}

export function createGraph(
  nodes: readonly GraphNode[],
  edges: readonly GraphEdge[],
  metadata?: Readonly<Record<string, unknown>>,
): GraphModel {
  const violations = validateGraph(nodes, edges);
  if (violations.length > 0) {
    throw new GraphValidationError(violations);
  }
  return { nodes, edges, metadata };
}
```

- [ ] **Step 4: Update index.ts barrel**

```typescript
export type { GraphNode, GraphEdge, GraphModel } from './model.js';
export {
  createGraph,
  validateGraph,
  GraphValidationError,
} from './graph.js';
export type { GraphViolation } from './graph.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add packages/graph-core/src/graph.ts packages/graph-core/src/graph.test.ts packages/graph-core/src/index.ts
git commit -m "feat(#266): createGraph and validateGraph with structural validation

Refs #266"
```

---

### Task 3: Containment traversal — childrenOf, ancestorsOf, subtreeOf, rootNodes

**Files:**
- Create: `packages/graph-core/src/traversal.ts`
- Create: `packages/graph-core/src/traversal.test.ts`
- Modify: `packages/graph-core/src/index.ts`

**Interfaces:**
- Consumes: `GraphNode`, `GraphModel` from `model.ts`
- Produces: `childrenOf(model, parentId) → readonly GraphNode[]`,
  `ancestorsOf(model, nodeId) → readonly GraphNode[]`,
  `subtreeOf(model, nodeId) → readonly GraphNode[]`,
  `rootNodes(model) → readonly GraphNode[]`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { childrenOf, ancestorsOf, subtreeOf, rootNodes } from './traversal.js';
import type { GraphModel, GraphNode } from './model.js';

function node(id: string, type = 'default', parentId?: string): GraphNode {
  return { id, type, parentId, properties: {} };
}

function model(nodes: GraphNode[]): GraphModel {
  return { nodes, edges: [] };
}

describe('childrenOf', () => {
  it('returns direct children', () => {
    const m = model([node('a'), node('b', 'x', 'a'), node('c', 'x', 'a')]);
    expect(childrenOf(m, 'a').map(n => n.id)).toEqual(['b', 'c']);
  });

  it('returns empty array when no children', () => {
    expect(childrenOf(model([node('a')]), 'a')).toEqual([]);
  });

  it('returns empty array for non-existent parent', () => {
    expect(childrenOf(model([node('a')]), 'missing')).toEqual([]);
  });

  it('does not return grandchildren', () => {
    const m = model([node('a'), node('b', 'x', 'a'), node('c', 'x', 'b')]);
    expect(childrenOf(m, 'a').map(n => n.id)).toEqual(['b']);
  });
});

describe('ancestorsOf', () => {
  it('returns parent chain nearest-first', () => {
    const m = model([node('root'), node('mid', 'x', 'root'), node('leaf', 'x', 'mid')]);
    expect(ancestorsOf(m, 'leaf').map(n => n.id)).toEqual(['mid', 'root']);
  });

  it('returns empty for root node', () => {
    expect(ancestorsOf(model([node('a')]), 'a')).toEqual([]);
  });

  it('returns empty for non-existent node', () => {
    expect(ancestorsOf(model([node('a')]), 'missing')).toEqual([]);
  });

  it('throws on containment cycle', () => {
    const m: GraphModel = {
      nodes: [
        { id: 'a', type: 'x', parentId: 'b', properties: {} },
        { id: 'b', type: 'x', parentId: 'a', properties: {} },
      ],
      edges: [],
    };
    expect(() => ancestorsOf(m, 'a')).toThrow(/cycle/i);
  });

  it('throws on self-referencing parentId', () => {
    const m: GraphModel = {
      nodes: [{ id: 'a', type: 'x', parentId: 'a', properties: {} }],
      edges: [],
    };
    expect(() => ancestorsOf(m, 'a')).toThrow(/cycle/i);
  });
});

describe('subtreeOf', () => {
  it('returns node itself plus all descendants', () => {
    const m = model([node('a'), node('b', 'x', 'a'), node('c', 'x', 'b')]);
    expect(subtreeOf(m, 'a').map(n => n.id)).toEqual(['a', 'b', 'c']);
  });

  it('returns breadth-first order', () => {
    const m = model([
      node('root'),
      node('l1a', 'x', 'root'),
      node('l1b', 'x', 'root'),
      node('l2a', 'x', 'l1a'),
      node('l2b', 'x', 'l1b'),
    ]);
    const ids = subtreeOf(m, 'root').map(n => n.id);
    expect(ids[0]).toBe('root');
    expect(ids.indexOf('l1a')).toBeLessThan(ids.indexOf('l2a'));
    expect(ids.indexOf('l1b')).toBeLessThan(ids.indexOf('l2b'));
    expect(ids.indexOf('l1a')).toBeLessThan(ids.indexOf('l2a'));
  });

  it('returns just the node when it has no children', () => {
    expect(subtreeOf(model([node('a')]), 'a').map(n => n.id)).toEqual(['a']);
  });

  it('returns empty for non-existent node', () => {
    expect(subtreeOf(model([node('a')]), 'missing')).toEqual([]);
  });

  it('throws on containment cycle', () => {
    const m: GraphModel = {
      nodes: [
        { id: 'a', type: 'x', parentId: 'b', properties: {} },
        { id: 'b', type: 'x', parentId: 'a', properties: {} },
      ],
      edges: [],
    };
    expect(() => subtreeOf(m, 'a')).toThrow(/cycle/i);
  });
});

describe('rootNodes', () => {
  it('returns nodes with no parentId', () => {
    const m = model([node('a'), node('b', 'x', 'a'), node('c')]);
    expect(rootNodes(m).map(n => n.id)).toEqual(['a', 'c']);
  });

  it('returns all nodes when none have parents', () => {
    const m = model([node('a'), node('b')]);
    expect(rootNodes(m)).toHaveLength(2);
  });

  it('returns empty when all nodes have parents', () => {
    const m = model([node('a', 'x', 'b'), node('b', 'x', 'a')]);
    expect(rootNodes(m)).toEqual([]);
  });

  it('returns empty for empty model', () => {
    expect(rootNodes(model([]))).toEqual([]);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: FAIL — `traversal.js` does not exist

- [ ] **Step 3: Implement traversal.ts**

```typescript
import type { GraphNode, GraphModel } from './model.js';

export function childrenOf(model: GraphModel, parentId: string): readonly GraphNode[] {
  return model.nodes.filter(n => n.parentId === parentId);
}

export function ancestorsOf(model: GraphModel, nodeId: string): readonly GraphNode[] {
  const nodeMap = new Map(model.nodes.map(n => [n.id, n]));
  const target = nodeMap.get(nodeId);
  if (!target) return [];

  const ancestors: GraphNode[] = [];
  const visited = new Set<string>();
  visited.add(nodeId);
  let current = target.parentId !== undefined ? nodeMap.get(target.parentId) : undefined;

  while (current) {
    if (visited.has(current.id)) {
      throw new Error(`Containment cycle detected at node '${current.id}'`);
    }
    visited.add(current.id);
    ancestors.push(current);
    current = current.parentId !== undefined ? nodeMap.get(current.parentId) : undefined;
  }

  return ancestors;
}

export function subtreeOf(model: GraphModel, nodeId: string): readonly GraphNode[] {
  const nodeMap = new Map(model.nodes.map(n => [n.id, n]));
  const root = nodeMap.get(nodeId);
  if (!root) return [];

  const result: GraphNode[] = [];
  const visited = new Set<string>();
  const queue: GraphNode[] = [root];

  while (queue.length > 0) {
    const current = queue.shift()!;
    if (visited.has(current.id)) {
      throw new Error(`Containment cycle detected at node '${current.id}'`);
    }
    visited.add(current.id);
    result.push(current);

    for (const child of model.nodes) {
      if (child.parentId === current.id) {
        queue.push(child);
      }
    }
  }

  return result;
}

export function rootNodes(model: GraphModel): readonly GraphNode[] {
  return model.nodes.filter(n => n.parentId === undefined);
}
```

- [ ] **Step 4: Update index.ts barrel**

Add to existing exports:

```typescript
export {
  childrenOf,
  ancestorsOf,
  subtreeOf,
  rootNodes,
} from './traversal.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```bash
git add packages/graph-core/src/traversal.ts packages/graph-core/src/traversal.test.ts packages/graph-core/src/index.ts
git commit -m "feat(#266): containment traversal — childrenOf, ancestorsOf, subtreeOf, rootNodes

Refs #266"
```

---

### Task 4: Edge and lookup queries — edgesOf, inboundEdges, outboundEdges, nodeById, edgeById

**Files:**
- Create: `packages/graph-core/src/query.ts`
- Create: `packages/graph-core/src/query.test.ts`
- Modify: `packages/graph-core/src/index.ts`

**Interfaces:**
- Consumes: `GraphNode`, `GraphEdge`, `GraphModel` from `model.ts`
- Produces: `edgesOf(model, nodeId) → readonly GraphEdge[]`,
  `inboundEdges(model, nodeId) → readonly GraphEdge[]`,
  `outboundEdges(model, nodeId) → readonly GraphEdge[]`,
  `nodeById(model, nodeId) → GraphNode | undefined`,
  `edgeById(model, edgeId) → GraphEdge | undefined`

- [ ] **Step 1: Write failing tests**

```typescript
import { describe, it, expect } from 'vitest';
import { edgesOf, inboundEdges, outboundEdges, nodeById, edgeById } from './query.js';
import type { GraphModel, GraphNode, GraphEdge } from './model.js';

function node(id: string, type = 'default'): GraphNode {
  return { id, type, properties: {} };
}

function edge(id: string, source: string, target: string, type = 'default'): GraphEdge {
  return { id, type, source, target };
}

function model(nodes: GraphNode[], edges: GraphEdge[]): GraphModel {
  return { nodes, edges };
}

describe('edgesOf', () => {
  it('returns all edges connected to a node', () => {
    const m = model(
      [node('a'), node('b'), node('c')],
      [edge('e1', 'a', 'b'), edge('e2', 'c', 'a'), edge('e3', 'b', 'c')],
    );
    const ids = edgesOf(m, 'a').map(e => e.id);
    expect(ids).toContain('e1');
    expect(ids).toContain('e2');
    expect(ids).not.toContain('e3');
  });

  it('returns empty array for node with no edges', () => {
    expect(edgesOf(model([node('a')], []), 'a')).toEqual([]);
  });

  it('returns self-loop edge once', () => {
    const m = model([node('a')], [edge('e1', 'a', 'a')]);
    expect(edgesOf(m, 'a')).toHaveLength(1);
  });

  it('returns empty for empty model', () => {
    expect(edgesOf(model([], []), 'a')).toEqual([]);
  });
});

describe('inboundEdges', () => {
  it('returns edges targeting the node', () => {
    const m = model(
      [node('a'), node('b')],
      [edge('e1', 'a', 'b'), edge('e2', 'b', 'a')],
    );
    expect(inboundEdges(m, 'a').map(e => e.id)).toEqual(['e2']);
  });

  it('includes self-loop', () => {
    const m = model([node('a')], [edge('e1', 'a', 'a')]);
    expect(inboundEdges(m, 'a')).toHaveLength(1);
  });
});

describe('outboundEdges', () => {
  it('returns edges sourced from the node', () => {
    const m = model(
      [node('a'), node('b')],
      [edge('e1', 'a', 'b'), edge('e2', 'b', 'a')],
    );
    expect(outboundEdges(m, 'a').map(e => e.id)).toEqual(['e1']);
  });

  it('includes self-loop', () => {
    const m = model([node('a')], [edge('e1', 'a', 'a')]);
    expect(outboundEdges(m, 'a')).toHaveLength(1);
  });
});

describe('nodeById', () => {
  it('finds a node by ID', () => {
    const m = model([node('a'), node('b')], []);
    expect(nodeById(m, 'b')?.id).toBe('b');
  });

  it('returns undefined for non-existent ID', () => {
    expect(nodeById(model([node('a')], []), 'missing')).toBeUndefined();
  });

  it('returns undefined for empty model', () => {
    expect(nodeById(model([], []), 'a')).toBeUndefined();
  });
});

describe('edgeById', () => {
  it('finds an edge by ID', () => {
    const m = model([node('a'), node('b')], [edge('e1', 'a', 'b')]);
    expect(edgeById(m, 'e1')?.id).toBe('e1');
  });

  it('returns undefined for non-existent ID', () => {
    expect(edgeById(model([], []), 'missing')).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: FAIL — `query.js` does not exist

- [ ] **Step 3: Implement query.ts**

```typescript
import type { GraphNode, GraphEdge, GraphModel } from './model.js';

export function edgesOf(model: GraphModel, nodeId: string): readonly GraphEdge[] {
  return model.edges.filter(e => e.source === nodeId || e.target === nodeId);
}

export function inboundEdges(model: GraphModel, nodeId: string): readonly GraphEdge[] {
  return model.edges.filter(e => e.target === nodeId);
}

export function outboundEdges(model: GraphModel, nodeId: string): readonly GraphEdge[] {
  return model.edges.filter(e => e.source === nodeId);
}

export function nodeById(model: GraphModel, nodeId: string): GraphNode | undefined {
  return model.nodes.find(n => n.id === nodeId);
}

export function edgeById(model: GraphModel, edgeId: string): GraphEdge | undefined {
  return model.edges.find(e => e.id === edgeId);
}
```

- [ ] **Step 4: Update index.ts barrel**

Add to existing exports:

```typescript
export {
  edgesOf,
  inboundEdges,
  outboundEdges,
  nodeById,
  edgeById,
} from './query.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @casehubio/graph-core run test`
Expected: all tests PASS

- [ ] **Step 6: Run full typecheck**

Run: `yarn workspace @casehubio/graph-core run typecheck`
Expected: no errors

- [ ] **Step 7: Commit**

```bash
git add packages/graph-core/src/query.ts packages/graph-core/src/query.test.ts packages/graph-core/src/index.ts
git commit -m "feat(#266): edge and lookup queries — edgesOf, inbound, outbound, nodeById, edgeById

Refs #266"
```
