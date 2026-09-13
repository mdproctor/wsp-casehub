# Global Handle Optimizer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-pages#412 — global handle optimization for autoDetectHandleDirections
**Issue group:** casehubio/blocks-ui#155 (parent branch), casehubio/casehub-pages#412

**Goal:** Replace the greedy per-edge handle assignment algorithm with a global
optimizer that considers all edges simultaneously to minimise straight-line
crossings.

**Architecture:** Brute-force enumeration of all valid (source-side, target-side)
handle assignments across all edges, scored by total edge-edge crossings. Pick
the assignment with minimum crossings. If crossings remain, try small position
offsets (±10/±20px grid search) on involved nodes and re-optimise. Replaces the
existing greedy algorithm entirely.

**Tech Stack:** TypeScript, vitest, ELK (elkjs), React Flow (@xyflow/react)

## Global Constraints

- Code lives in casehub-pages `packages/graph-renderer/src/`
  (canonical source: `/Users/mdproctor/claude/casehub/pages/`)
- Tests live in blocks-ui `components/org-diagram/src/edge-routing-tdd.test.ts`
  (project: `/Users/mdproctor/claude/casehub/slots/175/blocks-ui/`)
- blocks-ui consumes graph-renderer via `.casehub-packages/` portal deps
- All changes to mapping.ts and edge-routing-validator.ts are in the pages repo
- Zero straight-line crossings is the success metric (D9)
- All edges equal weight including excluded-from-layout (D13)

---

## Batch 1: ELK layout prerequisites + dev environment

These changes to `elk-layout.ts` are prerequisites from the compound-node-containment
work (documented in HANDOFF.md). Without them, ELK produces incorrect compound node
layouts and the org-diagram archetype tests cannot run.

### Task 1: ELK layout enhancements for compound nodes

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/layout/elk-layout.ts`
- Test: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/layout/elk-layout.test.ts` (existing)

**Interfaces:**
- Produces: `ElkLayoutOptions` with `algorithm`, `elkOptions`, `headerHeight`, `partitions` fields
- Produces: `computeElkLayout` that supports algorithm selection, excludeFromLayout filtering, edge LCA partitioning

- [ ] **Step 1: Add algorithm, elkOptions, headerHeight to ElkLayoutOptions**

```typescript
export interface ElkLayoutOptions {
  direction?: 'DOWN' | 'RIGHT' | 'LEFT' | 'UP';
  spacing?: number;
  containerPadding?: number;
  nodeSizes?: ReadonlyMap<string, { width: number; height: number }>;
  wrapping?: boolean;
  algorithm?: 'layered' | 'mrtree' | 'radial' | 'force' | 'stress';
  elkOptions?: Readonly<Record<string, string>>;
  headerHeight?: number;
  partitions?: ReadonlyMap<string, number>;
}
```

- [ ] **Step 2: Update buildElkNode to accept headerHeight, spacing, elkOptions**

Add `headerHeight`, `spacing`, and `elkOptions` parameters to `buildElkNode`.
Use `headerHeight` (default `DEFAULT_HEADER_HEIGHT`) for container padding top.
Propagate `elk.spacing.nodeNode` to containers. Spread `elkOptions` into container
`layoutOptions`.

```typescript
function buildElkNode(
  model: GraphModel,
  node: GraphNode,
  visited: Set<string>,
  padding: number,
  headerHeight: number,
  spacing: number,
  nodeSizes?: ReadonlyMap<string, { width: number; height: number }>,
  wrapping?: boolean,
  partitions?: ReadonlyMap<string, number>,
  elkOptions?: Readonly<Record<string, string>>,
): ElkNode {
  // ... existing cycle check and children logic ...
  if (children.length > 0) {
    elkNode.children = children.map(c =>
      buildElkNode(model, c, visited, padding, headerHeight, spacing, nodeSizes, wrapping, partitions, elkOptions));
    const containerOpts: Record<string, string> = {
      'elk.hierarchyHandling': 'INCLUDE_CHILDREN',
      'elk.padding': `[top=${Math.max(padding, headerHeight)},left=${padding},bottom=${padding},right=${padding}]`,
      'elk.spacing.nodeNode': String(spacing),
      ...(elkOptions ?? {}),
    };
    // ... wrapping logic unchanged ...
    elkNode.layoutOptions = containerOpts;
  }
  // partition support
  const part = partitions?.get(node.id);
  if (part !== undefined) {
    elkNode.layoutOptions = { ...(elkNode.layoutOptions ?? {}), 'elk.partitioning.partition': String(part) };
  }
  return elkNode;
}
```

- [ ] **Step 3: Filter excludeFromLayout edges and add edge LCA partitioning**

In `computeElkLayout`, filter edges with `excludeFromLayout` property. For remaining
edges, place each edge on its lowest common ancestor's children array (not root) so
ELK can size containers correctly.

```typescript
const layoutEdges = model.edges.filter(e => !e.properties?.['excludeFromLayout']);

// Edge LCA partitioning — place edges on their lowest common ancestor
function findLCA(a: string, b: string): string | null {
  const ancestorsA = new Set<string>();
  let cur = model.nodes.find(n => n.id === a);
  while (cur?.parentId) { ancestorsA.add(cur.parentId); cur = model.nodes.find(n => n.id === cur!.parentId); }
  cur = model.nodes.find(n => n.id === b);
  while (cur?.parentId) {
    if (ancestorsA.has(cur.parentId)) return cur.parentId;
    cur = model.nodes.find(n => n.id === cur!.parentId);
  }
  return null;
}
const edgesByParent = new Map<string | null, ElkExtendedEdge[]>();
for (const e of layoutEdges) {
  const lca = findLCA(e.source, e.target);
  const list = edgesByParent.get(lca) ?? [];
  list.push({ id: e.id, sources: [e.source], targets: [e.target] });
  edgesByParent.set(lca, list);
}
```

Attach edges to their LCA node's children array during `buildElkNode`, and only
root-level edges go on the root graph.

- [ ] **Step 4: Use algorithm option in root layoutOptions**

```typescript
const algorithm = options.algorithm ?? 'layered';
const layoutOpts: Record<string, string> = {
  'elk.algorithm': algorithm,
  'elk.spacing.nodeNode': String(spacing),
  'elk.hierarchyHandling': 'INCLUDE_CHILDREN',
  ...(options.elkOptions ?? {}),
};
if (algorithm === 'layered') {
  layoutOpts['elk.direction'] = direction;
  layoutOpts['elk.layered.spacing.nodeNodeBetweenLayers'] = String(spacing);
}
```

- [ ] **Step 5: Update computeElkLayout call site to pass new params**

Update the `buildElkNode` calls in `computeElkLayout` to pass `headerHeight`,
`spacing`, and `elkOptions`:

```typescript
const headerHeight = options.headerHeight ?? DEFAULT_HEADER_HEIGHT;
const rootChildren = roots.map(n =>
  buildElkNode(model, n, new Set(), padding, headerHeight, spacing,
    nodeSizes, options.wrapping, options.partitions, options.elkOptions));
```

- [ ] **Step 6: Run existing elk-layout tests**

Run: `npx vitest run packages/graph-renderer/src/layout/elk-layout.test.ts --reporter=verbose`
Expected: All existing tests pass (new fields are optional with defaults matching old behaviour)

- [ ] **Step 7: Build graph-renderer and copy to blocks-ui .casehub-packages**

```bash
# In pages repo
cd packages/graph-renderer && npx tsc -p tsconfig.build.json

# Create .casehub-packages in blocks-ui
mkdir -p /Users/mdproctor/claude/casehub/slots/175/blocks-ui/.casehub-packages/packages/graph-renderer
cp -r dist /Users/mdproctor/claude/casehub/slots/175/blocks-ui/.casehub-packages/packages/graph-renderer/
cp -r src /Users/mdproctor/claude/casehub/slots/175/blocks-ui/.casehub-packages/packages/graph-renderer/
cp package.json /Users/mdproctor/claude/casehub/slots/175/blocks-ui/.casehub-packages/packages/graph-renderer/
```

Also copy graph-core and other required packages from their dist directories.

- [ ] **Step 8: Verify edge routing tests run (may fail — baseline)**

Run: `npx vitest run components/org-diagram/src/edge-routing-tdd.test.ts --config components/org-diagram/vitest.config.ts --reporter=verbose`
Note: Some tests may fail due to the greedy algorithm's limitations. This establishes
the baseline before the optimizer is implemented.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): ELK layout enhancements for compound nodes

Add algorithm, elkOptions, headerHeight to ElkLayoutOptions.
Edge LCA partitioning places edges on lowest common ancestor.
Filter excludeFromLayout edges before layout.

Refs casehubio/casehub-pages#412"
```

---

## Batch 2: Validator tightening + global optimizer

### Task 2: Tighten edge-routing-validator

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/edge-routing-validator.ts`
- Test: Use existing edge-routing-tdd.test.ts (integration) — validator changes are tested
  through the archetype tests

**Interfaces:**
- Consumes: `Node[]`, `Edge[]` from `@xyflow/react`
- Produces: `ValidationResult` with `pass` and `violations` (unchanged interface)

- [ ] **Step 1: Add full ancestor exclusion to line-crosses-shape check**

Replace the simple parent skip (lines 81-83) with full ancestor traversal:

```typescript
function ancestors(nodeId: string): Set<string> {
  const result = new Set<string>();
  let cur = nodeMap.get(nodeId);
  while (cur?.parentId) {
    result.add(cur.parentId);
    cur = nodeMap.get(cur.parentId);
  }
  return result;
}

// In the edge-crosses-shape loop:
for (const edge of edges) {
  const src = nodeMap.get(edge.source);
  const tgt = nodeMap.get(edge.target);
  if (!src || !tgt) continue;
  const p1 = handleCenter(src, 'source', edge, nodeMap);
  const p2 = handleCenter(tgt, 'target', edge, nodeMap);
  const srcAncestors = ancestors(edge.source);
  const tgtAncestors = ancestors(edge.target);

  for (const node of nodes) {
    if (node.id === edge.source || node.id === edge.target) continue;
    if (srcAncestors.has(node.id) || tgtAncestors.has(node.id)) continue;
    if (node.parentId === edge.source || node.parentId === edge.target) continue;
    const r = absoluteRect(node, nodeMap);
    if (lineIntersectsRect(p1, p2, r.x, r.y, r.w, r.h)) {
      violations.push(`Line crosses shape: ${edge.source}→${edge.target} crosses '${node.id}'`);
    }
  }
}
```

- [ ] **Step 2: Verify no intra-container edge-edge crossing skip exists**

The pages/main validator does NOT have the intra-container skip (it was only in
the `.casehub-packages` copy). Confirm lines 98-113 check ALL edge pairs without
skipping same-parent edges. This is correct per D9 (zero straight-line crossings).

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/edge-routing-validator.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "fix(graph-renderer): full ancestor exclusion in edge routing validator

Edges passing through ancestor containers are not crossings.
No intra-container skip — all edge-edge crossings are violations.

Refs casehubio/casehub-pages#412"
```

### Task 3: Replace autoDetectHandleDirections with global optimizer

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/mapping.ts`
- Test: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/mapping.test.ts`
  (add unit tests for optimizer)

**Interfaces:**
- Consumes: `Node[]`, `Edge[]` from `@xyflow/react`, optional `direction` string
- Produces: Mutated `Edge[]` with `sourceHandle` and `targetHandle` set, mutated
  `Node[]` with `_sourceHandlePosition` and `_targetHandlePosition` in data

- [ ] **Step 1: Write failing unit test for global optimizer**

Add a test in `mapping.test.ts` that creates a diamond layout (4 nodes, 4 edges)
where the greedy algorithm produces crossings but the optimal assignment doesn't:

```typescript
describe('autoDetectHandleDirections — global optimizer', () => {
  it('avoids crossings in diamond layout', () => {
    // Diamond: A at top, B left, C right, D bottom
    // Edges: A→B, A→C, B→D, C→D
    // Greedy assigns A→B:bottom→top, A→C:bottom→top → C gets top blocked
    // Optimal: A→B:left→top, A→C:right→top, B→D:bottom→top, C→D:bottom→top
    const nodes: Node[] = [
      { id: 'A', position: { x: 150, y: 0 }, data: {}, width: 100, height: 50 },
      { id: 'B', position: { x: 0, y: 150 }, data: {}, width: 100, height: 50 },
      { id: 'C', position: { x: 300, y: 150 }, data: {}, width: 100, height: 50 },
      { id: 'D', position: { x: 150, y: 300 }, data: {}, width: 100, height: 50 },
    ];
    const edges: Edge[] = [
      { id: 'e1', source: 'A', target: 'B' },
      { id: 'e2', source: 'A', target: 'C' },
      { id: 'e3', source: 'B', target: 'D' },
      { id: 'e4', source: 'C', target: 'D' },
    ];
    // Import toReactFlowGraph or directly call the internal function
    // For now, test via toReactFlowGraph with a mock layout
    const model = {
      nodes: nodes.map(n => ({ id: n.id, type: 'test', properties: {} })),
      edges: edges.map(e => ({ id: e.id, type: 'default', source: e.source, target: e.target })),
    };
    const layout = {
      nodeLayouts: new Map(nodes.map(n => [n.id, {
        x: n.position.x, y: n.position.y,
        width: n.width!, height: n.height!,
      }])),
    };
    const result = toReactFlowGraph(model, layout, undefined, 'DOWN');
    // Validate no crossings
    const { validateEdgeRouting } = await import('./edge-routing-validator.js');
    const validation = validateEdgeRouting(result.nodes, result.edges);
    expect(validation.violations).toEqual([]);
  });

  it('handles empty edges', () => {
    const nodes: Node[] = [
      { id: 'A', position: { x: 0, y: 0 }, data: {}, width: 100, height: 50 },
    ];
    const model = { nodes: [{ id: 'A', type: 'test', properties: {} }], edges: [] };
    const layout = { nodeLayouts: new Map([['A', { x: 0, y: 0, width: 100, height: 50 }]]) };
    const result = toReactFlowGraph(model, layout);
    expect(result.edges).toEqual([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/graph-renderer/src/mapping.test.ts --reporter=verbose`
Expected: New test fails (greedy algorithm produces crossings in diamond layout)

- [ ] **Step 3: Implement global optimizer**

Replace the body of `autoDetectHandleDirections` (lines 91-253) with the global
optimizer. Keep the function signature identical.

The algorithm:
1. Build node map and absolute bounds
2. For each edge, compute all valid (srcSide, tgtSide) pairs (filter: no same-side,
   no line-crosses-node)
3. Enumerate all combinations of valid pairs across all edges
4. Score each combination by counting edge-edge crossings (segmentsIntersect)
5. Pick the combination with minimum crossings (tie-break by total edge length)
6. Apply the winning assignment to edges (sourceHandle, targetHandle)
7. Update node default handle positions from majority vote

Key implementation detail: for pruning, use recursive backtracking with early
termination when the current partial assignment already has more crossings than
the best complete assignment found so far.

```typescript
function autoDetectHandleDirections(nodes: Node[], edges: Edge[], _direction?: string): void {
  if (!nodes.length || !edges.length) return;

  const nodeMap = new Map(nodes.map(n => [n.id, n]));
  const SIDES = ['top', 'bottom', 'left', 'right'] as const;

  function absBounds(node: Node): { x: number; y: number; w: number; h: number } {
    let x = node.position.x, y = node.position.y;
    let cur = node;
    while (cur.parentId) {
      const parent = nodeMap.get(cur.parentId);
      if (!parent) break;
      x += parent.position.x; y += parent.position.y;
      cur = parent;
    }
    return { x, y, w: node.width ?? 280, h: node.height ?? 50 };
  }

  function handlePoint(bounds: { x: number; y: number; w: number; h: number }, side: string) {
    switch (side) {
      case 'top': return { x: bounds.x + bounds.w / 2, y: bounds.y };
      case 'bottom': return { x: bounds.x + bounds.w / 2, y: bounds.y + bounds.h };
      case 'left': return { x: bounds.x, y: bounds.y + bounds.h / 2 };
      case 'right': return { x: bounds.x + bounds.w, y: bounds.y + bounds.h / 2 };
      default: return { x: bounds.x + bounds.w / 2, y: bounds.y + bounds.h };
    }
  }

  function segmentsIntersect(
    a1: { x: number; y: number }, a2: { x: number; y: number },
    b1: { x: number; y: number }, b2: { x: number; y: number },
  ): boolean {
    const d1x = a2.x - a1.x, d1y = a2.y - a1.y;
    const d2x = b2.x - b1.x, d2y = b2.y - b1.y;
    const cross = d1x * d2y - d1y * d2x;
    if (Math.abs(cross) < 1e-10) return false;
    const t = ((b1.x - a1.x) * d2y - (b1.y - a1.y) * d2x) / cross;
    const u = ((b1.x - a1.x) * d1y - (b1.y - a1.y) * d1x) / cross;
    return t > 0.01 && t < 0.99 && u > 0.01 && u < 0.99;
  }

  // Collect obstacle nodes per edge (for line-crosses-node check)
  const parentIds = new Set(nodes.filter(n => n.parentId).map(n => n.parentId!));
  function ancestors(nodeId: string): Set<string> {
    const result = new Set<string>();
    let cur = nodeMap.get(nodeId);
    while (cur?.parentId) { result.add(cur.parentId); cur = nodeMap.get(cur.parentId); }
    return result;
  }
  function obstacleNodes(srcId: string, tgtId: string): Node[] {
    const srcAnc = ancestors(srcId);
    const tgtAnc = ancestors(tgtId);
    return nodes.filter(n => {
      if (n.id === srcId || n.id === tgtId) return false;
      if (srcAnc.has(n.id) || tgtAnc.has(n.id)) return false;
      if (n.parentId === srcId || n.parentId === tgtId) return false;
      return true;
    });
  }

  function lineCrossesNode(
    s: { x: number; y: number }, t: { x: number; y: number },
    srcId: string, tgtId: string,
  ): boolean {
    for (const node of obstacleNodes(srcId, tgtId)) {
      const r = absBounds(node);
      if (lineIntersectsRect(s, t, r.x, r.y, r.w, r.h)) return true;
    }
    return false;
  }

  // For each edge, compute valid (srcSide, tgtSide) candidates
  interface EdgeCandidate { srcSide: string; tgtSide: string; srcPt: { x: number; y: number }; tgtPt: { x: number; y: number }; dist: number; }
  const edgeCandidates: EdgeCandidate[][] = [];
  const validEdges: Edge[] = [];

  for (const edge of edges) {
    const srcNode = nodeMap.get(edge.source);
    const tgtNode = nodeMap.get(edge.target);
    if (!srcNode || !tgtNode) continue;
    const srcB = absBounds(srcNode);
    const tgtB = absBounds(tgtNode);
    const candidates: EdgeCandidate[] = [];
    for (const ss of SIDES) {
      for (const ts of SIDES) {
        if (ss === ts) continue;
        const sp = handlePoint(srcB, ss);
        const tp = handlePoint(tgtB, ts);
        if (lineCrossesNode(sp, tp, edge.source, edge.target)) continue;
        const dist = Math.sqrt((sp.x - tp.x) ** 2 + (sp.y - tp.y) ** 2);
        candidates.push({ srcSide: ss, tgtSide: ts, srcPt: sp, tgtPt: tp, dist });
      }
    }
    if (candidates.length === 0) {
      // Fallback: use shortest distance pair ignoring node crossings
      for (const ss of SIDES) {
        for (const ts of SIDES) {
          if (ss === ts) continue;
          const sp = handlePoint(srcB, ss);
          const tp = handlePoint(tgtB, ts);
          const dist = Math.sqrt((sp.x - tp.x) ** 2 + (sp.y - tp.y) ** 2);
          candidates.push({ srcSide: ss, tgtSide: ts, srcPt: sp, tgtPt: tp, dist });
        }
      }
    }
    // Sort by distance (prefer shorter edges as tie-breaker)
    candidates.sort((a, b) => a.dist - b.dist);
    edgeCandidates.push(candidates);
    validEdges.push(edge);
  }

  if (validEdges.length === 0) return;

  // Score a complete assignment by counting edge-edge crossings
  function countCrossings(assignment: EdgeCandidate[]): number {
    let crossings = 0;
    for (let i = 0; i < assignment.length; i++) {
      for (let j = i + 1; j < assignment.length; j++) {
        const a = assignment[i]!, b = assignment[j]!;
        const ae = validEdges[i]!, be = validEdges[j]!;
        // Skip edges that share a node
        if (ae.source === be.source || ae.target === be.target ||
            ae.source === be.target || ae.target === be.source) continue;
        if (segmentsIntersect(a.srcPt, a.tgtPt, b.srcPt, b.tgtPt)) crossings++;
      }
    }
    return crossings;
  }

  // Branch-and-bound search
  let bestCrossings = Infinity;
  let bestAssignment: EdgeCandidate[] = [];
  const current: EdgeCandidate[] = new Array(validEdges.length);

  function countPartialCrossings(depth: number): number {
    let crossings = 0;
    for (let i = 0; i < depth; i++) {
      for (let j = i + 1; j < depth; j++) {
        const a = current[i]!, b = current[j]!;
        const ae = validEdges[i]!, be = validEdges[j]!;
        if (ae.source === be.source || ae.target === be.target ||
            ae.source === be.target || ae.target === be.source) continue;
        if (segmentsIntersect(a.srcPt, a.tgtPt, b.srcPt, b.tgtPt)) crossings++;
      }
    }
    return crossings;
  }

  function search(depth: number): void {
    if (depth === validEdges.length) {
      const crossings = countCrossings(current);
      if (crossings < bestCrossings ||
          (crossings === bestCrossings && totalDist(current) < totalDist(bestAssignment))) {
        bestCrossings = crossings;
        bestAssignment = [...current];
      }
      return;
    }
    for (const candidate of edgeCandidates[depth]!) {
      current[depth] = candidate;
      // Prune: if partial assignment already has >= best, skip
      const partialCrossings = countPartialCrossings(depth + 1);
      if (partialCrossings >= bestCrossings) continue;
      search(depth + 1);
      if (bestCrossings === 0) return; // Can't do better than zero
    }
  }

  function totalDist(assignment: EdgeCandidate[]): number {
    return assignment.reduce((sum, c) => sum + c.dist, 0);
  }

  search(0);

  // Apply best assignment
  for (let i = 0; i < validEdges.length; i++) {
    const edge = validEdges[i]!;
    const candidate = bestAssignment[i]!;
    edge.sourceHandle = `source-${candidate.srcSide}`;
    edge.targetHandle = `target-${candidate.tgtSide}`;
  }

  // Update node default handle positions (majority vote)
  const srcCounts = new Map<string, Record<string, number>>();
  const tgtCounts = new Map<string, Record<string, number>>();
  for (const edge of edges) {
    const sp = edge.sourceHandle?.replace(/^source-/, '') ?? 'bottom';
    const tp = edge.targetHandle?.replace(/^target-/, '') ?? 'top';
    const sc = srcCounts.get(edge.source) ?? {};
    sc[sp] = (sc[sp] ?? 0) + 1;
    srcCounts.set(edge.source, sc);
    const tc = tgtCounts.get(edge.target) ?? {};
    tc[tp] = (tc[tp] ?? 0) + 1;
    tgtCounts.set(edge.target, tc);
  }
  const hasOut = new Set(edges.map(e => e.source));
  const hasIn = new Set(edges.map(e => e.target));
  for (const node of nodes) {
    const sc = srcCounts.get(node.id);
    const tc = tgtCounts.get(node.id);
    const updates: Record<string, unknown> = {};
    if (sc) updates._sourceHandlePosition = Object.entries(sc).sort((a, b) => b[1] - a[1])[0]![0];
    else if (!hasOut.has(node.id)) updates._sourceHandlePosition = undefined;
    if (tc) updates._targetHandlePosition = Object.entries(tc).sort((a, b) => b[1] - a[1])[0]![0];
    else if (!hasIn.has(node.id)) updates._targetHandlePosition = undefined;
    if (Object.keys(updates).length > 0) node.data = { ...node.data, ...updates };
  }
}
```

- [ ] **Step 4: Run unit tests**

Run: `npx vitest run packages/graph-renderer/src/mapping.test.ts --reporter=verbose`
Expected: All tests pass including the new diamond layout test

- [ ] **Step 5: Build, copy to .casehub-packages, run integration tests**

Build graph-renderer, copy dist to blocks-ui `.casehub-packages`, then run:

Run: `npx vitest run components/org-diagram/src/edge-routing-tdd.test.ts --config components/org-diagram/vitest.config.ts --reporter=verbose`
Expected: All 9 archetype tests pass with zero violations

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): global handle optimizer replaces greedy algorithm

Brute-force enumeration with branch-and-bound pruning considers all
edges simultaneously. Guarantees minimum edge-edge crossings.

Refs casehubio/casehub-pages#412"
```

---

## Batch 3: Position offsets

### Task 4: Grid search position offsets for remaining crossings

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/mapping.ts`
- Test: `/Users/mdproctor/claude/casehub/pages/packages/graph-renderer/src/mapping.test.ts`

**Interfaces:**
- Consumes: Same as Task 3
- Produces: Same as Task 3 (nodes may have adjusted positions)

- [ ] **Step 1: Write failing test for position offset case**

Create a K2,2 layout (2 sources, 2 targets, all 4 edges) where no handle
assignment avoids crossings at fixed positions, but a small offset resolves it:

```typescript
it('resolves K2,2 crossing via position offset', () => {
  // Two sources side by side, two targets side by side, all connected
  // At exactly symmetric positions, any handle assignment crosses
  const nodes: Node[] = [
    { id: 'S1', position: { x: 0, y: 0 }, data: {}, width: 80, height: 40 },
    { id: 'S2', position: { x: 200, y: 0 }, data: {}, width: 80, height: 40 },
    { id: 'T1', position: { x: 0, y: 200 }, data: {}, width: 80, height: 40 },
    { id: 'T2', position: { x: 200, y: 200 }, data: {}, width: 80, height: 40 },
  ];
  const edges: Edge[] = [
    { id: 'e1', source: 'S1', target: 'T1' },
    { id: 'e2', source: 'S1', target: 'T2' },
    { id: 'e3', source: 'S2', target: 'T1' },
    { id: 'e4', source: 'S2', target: 'T2' },
  ];
  const model = {
    nodes: nodes.map(n => ({ id: n.id, type: 'test', properties: {} })),
    edges: edges.map(e => ({ ...e, type: 'default' })),
  };
  const layout = {
    nodeLayouts: new Map(nodes.map(n => [n.id, {
      x: n.position.x, y: n.position.y, width: n.width!, height: n.height!,
    }])),
  };
  const result = toReactFlowGraph(model, layout, undefined, 'DOWN');
  const { validateEdgeRouting } = await import('./edge-routing-validator.js');
  const validation = validateEdgeRouting(result.nodes, result.edges);
  expect(validation.violations).toEqual([]);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run packages/graph-renderer/src/mapping.test.ts --reporter=verbose`
Expected: K2,2 test fails (crossings unavoidable at fixed positions)

- [ ] **Step 3: Add position offset phase after handle optimization**

After the branch-and-bound search in `autoDetectHandleDirections`, if
`bestCrossings > 0`, try position offsets:

```typescript
// Phase 2: Position offsets for remaining crossings
if (bestCrossings > 0) {
  const OFFSETS = [10, -10, 20, -20];
  // Identify nodes involved in crossings
  const crossingNodes = new Set<string>();
  for (let i = 0; i < bestAssignment.length; i++) {
    for (let j = i + 1; j < bestAssignment.length; j++) {
      const a = bestAssignment[i]!, b = bestAssignment[j]!;
      const ae = validEdges[i]!, be = validEdges[j]!;
      if (ae.source === be.source || ae.target === be.target ||
          ae.source === be.target || ae.target === be.source) continue;
      if (segmentsIntersect(a.srcPt, a.tgtPt, b.srcPt, b.tgtPt)) {
        crossingNodes.add(ae.source); crossingNodes.add(ae.target);
        crossingNodes.add(be.source); crossingNodes.add(be.target);
      }
    }
  }

  const crossingNodeIds = [...crossingNodes];
  // Try offsets on each crossing node independently
  for (const nodeId of crossingNodeIds) {
    const node = nodeMap.get(nodeId);
    if (!node) continue;
    const origX = node.position.x;
    const origY = node.position.y;
    for (const dx of [0, ...OFFSETS]) {
      for (const dy of [0, ...OFFSETS]) {
        if (dx === 0 && dy === 0) continue;
        node.position = { x: origX + dx, y: origY + dy };
        // Recompute candidates and re-run search
        // (rebuild edgeCandidates for affected edges, re-run search)
        rebuildAndSearch();
        if (bestCrossings === 0) break;
      }
      if (bestCrossings === 0) break;
    }
    if (bestCrossings === 0) break;
    node.position = { x: origX, y: origY }; // restore if no improvement
  }
}
```

The full implementation extracts the candidate-building and search logic into
reusable functions so the offset phase can call them.

- [ ] **Step 4: Run all tests**

Run: `npx vitest run packages/graph-renderer/src/mapping.test.ts --reporter=verbose`
Expected: All tests pass including K2,2 offset test

- [ ] **Step 5: Build, copy, run integration tests**

Build graph-renderer, copy to blocks-ui `.casehub-packages`, run all 9 archetype tests.

Run: `npx vitest run components/org-diagram/src/edge-routing-tdd.test.ts --config components/org-diagram/vitest.config.ts --reporter=verbose`
Expected: All 9 archetype tests pass with zero violations

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/graph-renderer/src/
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(graph-renderer): position offset phase for remaining crossings

Grid search ±10/±20px on nodes involved in crossings.
Re-runs handle optimization at each offset position.

Refs casehubio/casehub-pages#412"
```

---

## References

- [2026-09-07-global-handle-optimizer-design.md] — design spec this plan implements
- [mapping.ts:91-253] — current greedy autoDetectHandleDirections (pages/main)
- [edge-routing-validator.ts:55-116] — current validator (pages/main)
- [elk-layout.ts:1-143] — current ELK layout (pages/main, needs enhancements)
- [edge-routing-tdd.test.ts] — 9 archetype integration tests (blocks-ui)
- [HANDOFF.md] — compound-node-containment changes documentation
- [decisions.md D9-D15] — design decisions
- [casehubio/casehub-pages#412] — issue
- [casehubio/blocks-ui#155] — parent issue
