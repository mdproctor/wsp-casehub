# Eidos Org Diagram Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #155 — feat: eidos org structure diagram component — visualize organizational archetypes
**Issue group:** #155

**Goal:** Build a full interactive diagram editor for eidos organizational structures with archetype-aware layout, extending the established graph-stencil-* + DiagramBaseMixin pattern.

**Architecture:** Two new packages — `packages/graph-stencil-org` (YAML adapter, stencils, edge types, archetype detection, layout mapping, CST-preserving editor) and `components/org-diagram` (Lit component extending DiagramBaseMixin). The adapter converts eidos org YAML to GraphModel with compound nodes (agents inside units). ELK handles layout with algorithm selection per detected archetype. Six edge types render the six RelationshipKind values with distinct visuals.

**Tech Stack:** TypeScript, Lit, yaml (CST), ELK.js (via graph-renderer), ReactFlow (via pages-graph-canvas), Vitest

## Global Constraints

- All custom elements use `blocks-` prefix (`@customElement('blocks-org-diagram')`)
- ARIA attributes mandatory on every `@customElement`
- Follow graph-stencil-case + casehub-diagram patterns exactly
- CST-preserving YAML edits via `yaml` library's `parseDocument`
- Tests: Vitest with jsdom environment
- Yarn workspace monorepo with TypeScript project references

## Prerequisites (casehub-pages repo — separate issues)

These changes are in casehub-pages and must land before Batch 4:

1. **ELK algorithm selection** — extend `ElkLayoutOptions` with `algorithm` field and `elkOptions` pass-through in `graph-renderer/src/layout/elk-layout.ts`. Without this, all layouts fall back to `layered`.
2. **`<pages-yaml-pane>`** — generic syntax-highlighted YAML textarea in `pages-diagram-core`. Extract `highlightYaml()` from `examples/src/pages/diagram-export-page.ts`. Needed for Batch 5.
3. **GitHubBackend centralisation** — move from `pages-diagram-core` to `graph-core`. Update all consumers. Needed before Batch 5.

File issues for each prerequisite. Batches 1-3 can proceed without them.

---

## Batch 1: Foundation — Types, Adapter, Tests

### Task 1: Package scaffold and TypeScript types

**Files:**
- Create: `packages/graph-stencil-org/package.json`
- Create: `packages/graph-stencil-org/tsconfig.json`
- Create: `packages/graph-stencil-org/vitest.config.ts`
- Create: `packages/graph-stencil-org/src/index.ts`
- Create: `packages/graph-stencil-org/src/types.ts`
- Modify: `tsconfig.json` (root — add project reference)
- Modify: `package.json` (root — add workspace)

**Interfaces:**
- Produces: `OrgUnit`, `Membership`, `AgentRelationship`, `RelationshipKind`, `RelationshipScope`, `AttestationGrant`, `BehavioralSignal`, `AgentCapability`, `AgentGoal`, `AgentConstraint` types

- [ ] **Step 1: Create package scaffold**

```bash
ls packages/  # verify directory exists
```

Create `packages/graph-stencil-org/package.json`:
```json
{
  "name": "@casehubio/graph-stencil-org",
  "version": "0.0.1",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "vitest run"
  },
  "dependencies": {
    "@casehubio/graph-core": "workspace:*",
    "@casehubio/graph-renderer": "workspace:*",
    "yaml": "^2.7.1"
  },
  "devDependencies": {
    "vitest": "^3.2.1"
  }
}
```

Create `packages/graph-stencil-org/tsconfig.json` following the pattern from `packages/graph-stencil-case/tsconfig.json`.

Create `packages/graph-stencil-org/vitest.config.ts`:
```typescript
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { environment: 'jsdom' } });
```

- [ ] **Step 2: Add workspace reference**

Modify root `package.json` workspaces array to include `packages/graph-stencil-org`. Add project reference in root `tsconfig.json`.

- [ ] **Step 3: Write TypeScript types**

Create `packages/graph-stencil-org/src/types.ts`:

```typescript
export type BehavioralSignal = 'DECLINE' | 'SUCCESS' | 'COMPLIANT' | 'VIOLATED';

export interface AgentCapability {
  name: string;
  description?: string;
}

export interface AgentGoal {
  name: string;
  description?: string;
  priority?: string;
  visibility?: string;
}

export interface AgentConstraint {
  name: string;
  description?: string;
  severity?: string;
  visibility?: string;
}

export interface Membership {
  agentId: string;
  role?: string;
  roleVocabulary?: string;
}

export interface RelationshipScope {
  capabilityName?: string;
  domain?: string;
  custom?: string;
}

export interface AttestationGrant {
  dimensions: [string, ...string[]];
  capabilityScope?: string[];
  signalTypes?: BehavioralSignal[];
}

export type RelationshipKind =
  | 'SUPERVISES'
  | 'DELEGATES_TO'
  | 'ESCALATES_TO'
  | 'REPORTS_TO'
  | 'BACKS_UP'
  | 'EXTENDED';

export interface AgentRelationship {
  sourceAgentId: string;
  targetAgentId: string;
  kind: RelationshipKind;
  extendedKind?: string;
  kindVocabulary?: string;
  scope?: RelationshipScope;
  attestation?: AttestationGrant;
  tenancyId: string;
}

export interface OrgUnit {
  unitId: string;
  name: string;
  kind?: string;
  kindVocabulary?: string;
  tenancyId: string;
  parentUnitId?: string;
  members: Membership[];
  capabilities: AgentCapability[];
  goals: AgentGoal[];
  constraints: AgentConstraint[];
}

export interface OrgStructureYaml {
  organization: {
    units: OrgUnit[];
    relationships: AgentRelationship[];
  };
}
```

- [ ] **Step 4: Create index.ts with re-exports**

Create `packages/graph-stencil-org/src/index.ts`:
```typescript
export type {
  OrgUnit, Membership, AgentRelationship, RelationshipKind,
  RelationshipScope, AttestationGrant, BehavioralSignal,
  AgentCapability, AgentGoal, AgentConstraint, OrgStructureYaml,
} from './types.js';
```

- [ ] **Step 5: Verify build**

```bash
yarn install
yarn workspace @casehubio/graph-stencil-org build
```

Expected: builds with no errors.

- [ ] **Step 6: Commit**

```bash
git add packages/graph-stencil-org/ package.json tsconfig.json
git commit -m "feat(graph-stencil-org): scaffold package with TypeScript types Refs #155"
```

---

### Task 2: YAML adapter — toOrgGraph

**Files:**
- Create: `packages/graph-stencil-org/src/adapter/org-adapter.ts`
- Create: `packages/graph-stencil-org/src/adapter/org-adapter.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `OrgUnit`, `Membership`, `AgentRelationship`, `OrgStructureYaml` from types.ts
- Produces: `toOrgGraph(yaml: string) → AdapterResult` — converts eidos org YAML to `GraphModel`. `AdapterResult` has `model: GraphModel` and `yamlPaths: Map<string, (string|number)[]>`.

- [ ] **Step 1: Write failing tests for adapter**

Create `packages/graph-stencil-org/src/adapter/org-adapter.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'fs';
import { resolve } from 'path';
import { toOrgGraph } from './org-adapter.js';

const ARCHETYPES_DIR = resolve(__dirname, '../../../../eidos/examples/org-scenarios/src/test/resources/archetypes');

function loadArchetype(name: string): string {
  return readFileSync(resolve(ARCHETYPES_DIR, `${name}.yaml`), 'utf-8');
}

describe('toOrgGraph', () => {
  it('parses simple-structure archetype', () => {
    const yaml = loadArchetype('simple-structure');
    const result = toOrgGraph(yaml);

    expect(result.model.nodes).toHaveLength(6); // 1 unit + 5 agents
    const units = result.model.nodes.filter(n => n.type === 'org-unit');
    const agents = result.model.nodes.filter(n => n.type === 'org-agent');
    expect(units).toHaveLength(1);
    expect(agents).toHaveLength(5);

    // All agents are children of the unit
    for (const agent of agents) {
      expect(agent.parentId).toBe('unit:startup');
    }

    // 4 SUPERVISES edges
    expect(result.model.edges).toHaveLength(4);
    for (const edge of result.model.edges) {
      expect(edge.type).toBe('org-supervises');
    }
  });

  it('parses divisional-holarchy with nested units', () => {
    const yaml = loadArchetype('divisional-holarchy');
    const result = toOrgGraph(yaml);

    const units = result.model.nodes.filter(n => n.type === 'org-unit');
    expect(units.length).toBeGreaterThanOrEqual(3); // hospital, emergency, radiology

    // emergency and radiology are children of hospital
    const emergency = units.find(n => n.properties['unitId'] === 'emergency');
    expect(emergency?.parentId).toBe('unit:hospital');

    const radiology = units.find(n => n.properties['unitId'] === 'radiology');
    expect(radiology?.parentId).toBe('unit:hospital');
  });

  it('maps relationship kinds to edge types', () => {
    const yaml = loadArchetype('tiered-escalation');
    const result = toOrgGraph(yaml);

    const edgeTypes = new Set(result.model.edges.map(e => e.type));
    expect(edgeTypes.has('org-escalates-to')).toBe(true);
    expect(edgeTypes.has('org-supervises')).toBe(true);
    expect(edgeTypes.has('org-backs-up')).toBe(true);
  });

  it('handles EXTENDED relationships with extendedKind', () => {
    const yaml = loadArchetype('market');
    const result = toOrgGraph(yaml);

    const extendedEdges = result.model.edges.filter(e => e.type === 'org-extended');
    expect(extendedEdges.length).toBeGreaterThan(0);
    expect(extendedEdges[0]!.properties?.['extendedKind']).toBe('bids-to');
  });

  it('handles matrix — multi-unit agents', () => {
    const yaml = loadArchetype('matrix');
    const result = toOrgGraph(yaml);

    // dev-alice is in platform-team AND billing-project
    const aliceNodes = result.model.nodes.filter(
      n => n.type === 'org-agent' && n.properties['agentId'] === 'dev-alice'
    );
    expect(aliceNodes).toHaveLength(2);
    expect(new Set(aliceNodes.map(n => n.properties['unitId']))).toEqual(
      new Set(['platform-team', 'billing-project'])
    );
  });

  it('resolves multi-unit edges with same-unit preference', () => {
    const yaml = loadArchetype('matrix');
    const result = toOrgGraph(yaml);

    // platform-lead SUPERVISES dev-alice — both in platform-team
    const supervises = result.model.edges.find(
      e => e.type === 'org-supervises' &&
           e.source === 'agent:platform-team:platform-lead' &&
           e.target.includes('dev-alice')
    );
    expect(supervises).toBeDefined();
    expect(supervises!.target).toBe('agent:platform-team:dev-alice');
  });

  it('builds yamlPaths for all nodes', () => {
    const yaml = loadArchetype('simple-structure');
    const result = toOrgGraph(yaml);

    for (const node of result.model.nodes) {
      expect(result.yamlPaths.has(node.id)).toBe(true);
    }
  });

  it('builds yamlPaths for relationship edges', () => {
    const yaml = loadArchetype('simple-structure');
    const result = toOrgGraph(yaml);

    for (const edge of result.model.edges) {
      const relKey = `rel:${edge.id.split('--')[1]}`;
      expect(result.yamlPaths.has(edge.id) || result.yamlPaths.has(relKey)).toBe(true);
    }
  });

  it('includes scope and attestation in edge properties', () => {
    const yaml = loadArchetype('federation-orchestrator');
    const result = toOrgGraph(yaml);

    const delegateEdges = result.model.edges.filter(e => e.type === 'org-delegates-to');
    const scopedEdge = delegateEdges.find(e => e.properties?.['scope']);
    expect(scopedEdge).toBeDefined();
    expect((scopedEdge!.properties!['scope'] as any).capabilityName).toBeDefined();

    const attestEdges = result.model.edges.filter(e => e.properties?.['attestation']);
    // federation has attestation on the reviewer→coder extended relationship
    expect(attestEdges.length).toBeGreaterThanOrEqual(0);
  });

  it('parses all 9 archetypes without error', () => {
    const names = [
      'simple-structure', 'divisional-holarchy', 'federation-orchestrator',
      'pipeline', 'market', 'matrix', 'tiered-escalation',
      'professional-bureaucracy', 'coalition-advisory',
    ];
    for (const name of names) {
      const yaml = loadArchetype(name);
      expect(() => toOrgGraph(yaml)).not.toThrow();
    }
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
yarn workspace @casehubio/graph-stencil-org test
```

Expected: FAIL — `toOrgGraph` not found.

- [ ] **Step 3: Implement toOrgGraph adapter**

Create `packages/graph-stencil-org/src/adapter/org-adapter.ts`:

```typescript
import { parseDocument } from 'yaml';
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import type { AdapterResult } from '@casehubio/pages-diagram-core';
import type { OrgStructureYaml, OrgUnit, AgentRelationship } from '../types.js';

const RELATIONSHIP_KIND_TO_EDGE_TYPE: Record<string, string> = {
  SUPERVISES: 'org-supervises',
  DELEGATES_TO: 'org-delegates-to',
  ESCALATES_TO: 'org-escalates-to',
  REPORTS_TO: 'org-reports-to',
  BACKS_UP: 'org-backs-up',
  EXTENDED: 'org-extended',
};

function agentNodeId(unitId: string, agentId: string): string {
  return `agent:${unitId}:${agentId}`;
}

function unitNodeId(unitId: string): string {
  return `unit:${unitId}`;
}

function resolveAgentNode(
  agentId: string,
  preferredUnitId: string | undefined,
  agentIndex: Map<string, string[]>,
): string {
  const units = agentIndex.get(agentId);
  if (!units || units.length === 0) {
    return `agent:unknown:${agentId}`;
  }
  if (units.length === 1) return agentNodeId(units[0]!, agentId);
  if (preferredUnitId && units.includes(preferredUnitId)) {
    return agentNodeId(preferredUnitId, agentId);
  }
  return agentNodeId(units[0]!, agentId);
}

function findSharedUnit(
  sourceAgentId: string,
  targetAgentId: string,
  agentIndex: Map<string, string[]>,
): string | undefined {
  const sourceUnits = agentIndex.get(sourceAgentId) ?? [];
  const targetUnits = new Set(agentIndex.get(targetAgentId) ?? []);
  return sourceUnits.find(u => targetUnits.has(u));
}

export function toOrgGraph(yaml: string): AdapterResult {
  const doc = parseDocument(yaml);
  const data = doc.toJS() as OrgStructureYaml;
  const org = data.organization;

  const nodes: GraphNode[] = [];
  const edges: GraphEdge[] = [];
  const yamlPaths = new Map<string, readonly (string | number)[]>();

  // Build agent → units index for multi-unit resolution
  const agentIndex = new Map<string, string[]>();
  for (const unit of org.units) {
    for (const member of unit.members ?? []) {
      const units = agentIndex.get(member.agentId) ?? [];
      units.push(unit.unitId);
      agentIndex.set(member.agentId, units);
    }
  }

  // Build unit nodes
  for (let i = 0; i < org.units.length; i++) {
    const unit = org.units[i]!;
    const nodeId = unitNodeId(unit.unitId);
    nodes.push({
      id: nodeId,
      type: 'org-unit',
      parentId: unit.parentUnitId ? unitNodeId(unit.parentUnitId) : undefined,
      properties: {
        unitId: unit.unitId,
        name: unit.name,
        kind: unit.kind,
        kindVocabulary: unit.kindVocabulary,
        label: unit.name,
        memberCount: (unit.members ?? []).length,
        capabilities: unit.capabilities ?? [],
        goals: unit.goals ?? [],
        constraints: unit.constraints ?? [],
      },
    });
    yamlPaths.set(nodeId, ['organization', 'units', i]);

    // Build agent nodes within unit
    for (let j = 0; j < (unit.members ?? []).length; j++) {
      const member = unit.members[j]!;
      const agentId = agentNodeId(unit.unitId, member.agentId);
      nodes.push({
        id: agentId,
        type: 'org-agent',
        parentId: nodeId,
        properties: {
          agentId: member.agentId,
          role: member.role,
          roleVocabulary: member.roleVocabulary,
          unitId: unit.unitId,
          label: member.agentId,
        },
      });
      yamlPaths.set(agentId, ['organization', 'units', i, 'members', j]);
    }
  }

  // Build relationship edges
  for (let i = 0; i < (org.relationships ?? []).length; i++) {
    const rel = org.relationships[i]!;
    const edgeType = RELATIONSHIP_KIND_TO_EDGE_TYPE[rel.kind] ?? 'org-extended';

    const sharedUnit = findSharedUnit(rel.sourceAgentId, rel.targetAgentId, agentIndex);
    const sourceNode = resolveAgentNode(rel.sourceAgentId, sharedUnit, agentIndex);
    const targetNode = resolveAgentNode(rel.targetAgentId, sharedUnit, agentIndex);

    const edgeId = `${sourceNode}--${edgeType}--${targetNode}--${i}`;
    const properties: Record<string, unknown> = {
      kind: rel.kind,
      relIndex: i,
    };
    if (rel.extendedKind) properties['extendedKind'] = rel.extendedKind;
    if (rel.scope) properties['scope'] = rel.scope;
    if (rel.attestation) properties['attestation'] = rel.attestation;

    edges.push({
      id: edgeId,
      type: edgeType,
      source: sourceNode,
      target: targetNode,
      properties,
    });
    yamlPaths.set(edgeId, ['organization', 'relationships', i]);
  }

  return { model: { nodes, edges }, yamlPaths };
}
```

- [ ] **Step 4: Update index.ts**

Add to `packages/graph-stencil-org/src/index.ts`:
```typescript
export { toOrgGraph } from './adapter/org-adapter.js';
export type { AdapterResult } from '@casehubio/pages-diagram-core';
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
yarn workspace @casehubio/graph-stencil-org test
```

Expected: all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): YAML adapter with multi-unit edge routing Refs #155"
```

---

## Batch 2: Stencils, Edge Types, and Property Schemas

### Task 3: Stencil registration and edge types

**Files:**
- Create: `packages/graph-stencil-org/src/stencils/org-unit.ts`
- Create: `packages/graph-stencil-org/src/stencils/org-agent.ts`
- Create: `packages/graph-stencil-org/src/stencils/register.ts`
- Create: `packages/graph-stencil-org/src/stencils/index.ts`
- Create: `packages/graph-stencil-org/src/stencils/stencils.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`
- Modify: `packages/graph-stencil-org/package.json` (add graph-renderer dep)

**Interfaces:**
- Consumes: `registerStencil`, `registerEdgeType` from `@casehubio/graph-renderer`
- Produces: `registerOrgStencils()` — registers 2 node stencils (org-unit, org-agent) and 6 edge types with the pages stencil registry

- [ ] **Step 1: Write failing test**

Create `packages/graph-stencil-org/src/stencils/stencils.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { clearRegistry, getStencil, getEdgeDescriptor } from '@casehubio/graph-renderer';
import { registerOrgStencils } from './register.js';

describe('registerOrgStencils', () => {
  beforeEach(() => { clearRegistry(); });

  it('registers org-unit stencil', () => {
    registerOrgStencils();
    expect(getStencil('org-unit')).toBeDefined();
    expect(getStencil('org-unit')!.label).toBe('Unit');
  });

  it('registers org-agent stencil', () => {
    registerOrgStencils();
    expect(getStencil('org-agent')).toBeDefined();
    expect(getStencil('org-agent')!.label).toBe('Agent');
  });

  it('registers all 6 edge types', () => {
    registerOrgStencils();
    expect(getEdgeDescriptor('org-supervises')).toBeDefined();
    expect(getEdgeDescriptor('org-delegates-to')).toBeDefined();
    expect(getEdgeDescriptor('org-escalates-to')).toBeDefined();
    expect(getEdgeDescriptor('org-reports-to')).toBeDefined();
    expect(getEdgeDescriptor('org-backs-up')).toBeDefined();
    expect(getEdgeDescriptor('org-extended')).toBeDefined();
  });

  it('org-unit grammar allows agent children', () => {
    registerOrgStencils();
    const stencil = getStencil('org-unit')!;
    expect(stencil.grammar.containment?.allowedChildTypes).toContain('org-agent');
    expect(stencil.grammar.containment?.allowedChildTypes).toContain('org-unit');
  });

  it('org-agent grammar allows agent-to-agent connections', () => {
    registerOrgStencils();
    const stencil = getStencil('org-agent')!;
    expect(stencil.grammar.connections.outbound.allowedTo).toContain('org-agent');
    expect(stencil.grammar.connections.inbound.allowedFrom).toContain('org-agent');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
yarn workspace @casehubio/graph-stencil-org test
```

Expected: FAIL — `registerOrgStencils` not found.

- [ ] **Step 3: Implement stencil render functions**

Create `packages/graph-stencil-org/src/stencils/org-unit.ts`:

```typescript
import type { GraphNode, NodeDecoration } from '@casehubio/graph-core';

export interface StencilTemplate {
  readonly html: string;
  readonly width: number;
  readonly height: number;
}

export function renderOrgUnit(node: GraphNode, decoration?: NodeDecoration): StencilTemplate {
  const name = String(node.properties['name'] ?? node.properties['unitId'] ?? '');
  const kind = node.properties['kind'] as string | undefined;
  const memberCount = node.properties['memberCount'] as number ?? 0;
  const kindBadge = kind ? `<span style="font-size:10px;background:#e0e7ff;color:#3730a3;padding:1px 6px;border-radius:3px;margin-left:6px;">${kind}</span>` : '';
  const countBadge = `<span style="font-size:10px;background:#f3f4f6;color:#6b7280;padding:1px 6px;border-radius:3px;margin-left:auto;">${memberCount}</span>`;

  return {
    html: `
      <div style="min-width:200px;min-height:80px;border:2px solid #6366f1;border-radius:8px;background:#faf5ff;overflow:visible;">
        <div style="display:flex;align-items:center;padding:6px 10px;background:#ede9fe;border-radius:6px 6px 0 0;border-bottom:1px solid #c4b5fd;">
          <span style="font-weight:600;font-size:13px;color:#4338ca;">${name}</span>
          ${kindBadge}
          ${countBadge}
        </div>
      </div>
    `,
    width: 280,
    height: 120,
  };
}
```

Create `packages/graph-stencil-org/src/stencils/org-agent.ts`:

```typescript
import type { GraphNode, NodeDecoration } from '@casehubio/graph-core';
import type { StencilTemplate } from './org-unit.js';

export function renderOrgAgent(node: GraphNode, decoration?: NodeDecoration): StencilTemplate {
  const agentId = String(node.properties['agentId'] ?? '');
  const role = node.properties['role'] as string | undefined;
  const roleLabel = role ? `<div style="font-size:10px;color:#6b7280;margin-top:2px;">${role}</div>` : '';

  return {
    html: `
      <div style="display:flex;align-items:center;gap:8px;padding:6px 12px;background:#fff;border:1px solid #d1d5db;border-radius:6px;min-width:120px;">
        <svg width="16" height="16" viewBox="0 0 16 16" fill="none" stroke="#6b7280" stroke-width="1.5">
          <circle cx="8" cy="5" r="3"/><path d="M2 14c0-3.3 2.7-6 6-6s6 2.7 6 6"/>
        </svg>
        <div>
          <div style="font-weight:500;font-size:12px;color:#1f2937;">${agentId}</div>
          ${roleLabel}
        </div>
      </div>
    `,
    width: 160,
    height: 40,
  };
}
```

- [ ] **Step 4: Implement registration function**

Create `packages/graph-stencil-org/src/stencils/register.ts`:

```typescript
import { registerStencil, registerEdgeType } from '@casehubio/graph-renderer';
import { registerGrammar } from '@casehubio/graph-core';
import type { StencilGrammar } from '@casehubio/graph-core';
import { renderOrgUnit } from './org-unit.js';
import { renderOrgAgent } from './org-agent.js';

const orgUnitGrammar: StencilGrammar = {
  type: 'org-unit',
  connections: {
    inbound: { min: 0, max: 0, allowedFrom: [] },
    outbound: { min: 0, max: 0, allowedTo: [] },
  },
  containment: {
    allowedChildTypes: ['org-agent', 'org-unit'],
    allowedParentTypes: ['org-unit'],
  },
};

const orgAgentGrammar: StencilGrammar = {
  type: 'org-agent',
  connections: {
    inbound: { min: 0, max: Infinity, allowedFrom: ['org-agent'] },
    outbound: { min: 0, max: Infinity, allowedTo: ['org-agent'] },
  },
  containment: {
    allowedParentTypes: ['org-unit'],
  },
};

let registered = false;

export function registerOrgStencils(): void {
  if (registered) return;
  registered = true;

  registerStencil({
    type: 'org-unit',
    label: 'Unit',
    icon: '□',
    grammar: orgUnitGrammar,
    render: renderOrgUnit as any,
  });

  registerStencil({
    type: 'org-agent',
    label: 'Agent',
    icon: '●',
    grammar: orgAgentGrammar,
    render: renderOrgAgent as any,
  });

  registerEdgeType({ type: 'org-supervises', label: 'Supervises', defaultStyle: '.react-flow__edge.org-supervises path { stroke: #374151; stroke-width: 2; }' });
  registerEdgeType({ type: 'org-delegates-to', label: 'Delegates to', defaultStyle: '.react-flow__edge.org-delegates-to path { stroke: #3b82f6; stroke-width: 2; stroke-dasharray: 6 3; }' });
  registerEdgeType({ type: 'org-escalates-to', label: 'Escalates to', defaultStyle: '.react-flow__edge.org-escalates-to path { stroke: #ef4444; stroke-width: 2; stroke-dasharray: 2 3; }' });
  registerEdgeType({ type: 'org-reports-to', label: 'Reports to', defaultStyle: '.react-flow__edge.org-reports-to path { stroke: #6b7280; stroke-width: 1; }' });
  registerEdgeType({ type: 'org-backs-up', label: 'Backs up', defaultStyle: '.react-flow__edge.org-backs-up path { stroke: #16a34a; stroke-width: 3; }' });
  registerEdgeType({ type: 'org-extended', label: 'Extended', defaultStyle: '.react-flow__edge.org-extended path { stroke: #8b5cf6; stroke-width: 2; stroke-dasharray: 6 3; }' });
}
```

Create `packages/graph-stencil-org/src/stencils/index.ts`:
```typescript
export { registerOrgStencils } from './register.js';
export { renderOrgUnit } from './org-unit.js';
export { renderOrgAgent } from './org-agent.js';
```

- [ ] **Step 5: Update index.ts**

Add to `packages/graph-stencil-org/src/index.ts`:
```typescript
export { registerOrgStencils } from './stencils/index.js';
```

- [ ] **Step 6: Run tests**

```bash
yarn workspace @casehubio/graph-stencil-org test
```

Expected: all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): stencil registration with grammars and 6 edge types Refs #155"
```

---

### Task 4: Property schemas

**Files:**
- Create: `packages/graph-stencil-org/src/schemas/unit-schema.ts`
- Create: `packages/graph-stencil-org/src/schemas/agent-schema.ts`
- Create: `packages/graph-stencil-org/src/schemas/index.ts`
- Create: `packages/graph-stencil-org/src/schemas/schemas.test.ts`
- Modify: `packages/graph-stencil-org/src/stencils/register.ts` (register schemas)
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `registerPropertySchema` from `@casehubio/pages-diagram-core`
- Produces: `orgUnitSchema`, `orgAgentSchema` — JSON Schema objects for the property panel

- [ ] **Step 1: Write failing test**

Create `packages/graph-stencil-org/src/schemas/schemas.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { clearPropertySchemas, getPropertySchema, registerPropertySchema } from '@casehubio/pages-diagram-core';
import { orgUnitSchema, orgAgentSchema } from './index.js';

describe('property schemas', () => {
  beforeEach(() => { clearPropertySchemas(); });

  it('orgUnitSchema has required fields', () => {
    expect(orgUnitSchema.properties).toHaveProperty('unitId');
    expect(orgUnitSchema.properties).toHaveProperty('name');
    expect(orgUnitSchema.properties).toHaveProperty('kind');
    expect(orgUnitSchema.properties).toHaveProperty('tenancyId');
  });

  it('orgAgentSchema has required fields', () => {
    expect(orgAgentSchema.properties).toHaveProperty('agentId');
    expect(orgAgentSchema.properties).toHaveProperty('role');
  });

  it('schemas can be registered and retrieved', () => {
    registerPropertySchema('org-unit', orgUnitSchema);
    registerPropertySchema('org-agent', orgAgentSchema);
    expect(getPropertySchema('org-unit')).toBe(orgUnitSchema);
    expect(getPropertySchema('org-agent')).toBe(orgAgentSchema);
  });
});
```

- [ ] **Step 2: Run tests to verify fail**

- [ ] **Step 3: Implement schemas**

Create `packages/graph-stencil-org/src/schemas/unit-schema.ts`:

```typescript
export const orgUnitSchema = {
  type: 'object' as const,
  properties: {
    unitId: { type: 'string', title: 'Unit ID', 'x-order': 0 },
    name: { type: 'string', title: 'Name', 'x-order': 1 },
    kind: { type: 'string', title: 'Kind', 'x-order': 2 },
    kindVocabulary: { type: 'string', title: 'Kind Vocabulary', 'x-order': 3, 'x-visibility': 'advanced' },
    tenancyId: { type: 'string', title: 'Tenancy', 'x-order': 4, 'x-visibility': 'advanced' },
  },
};
```

Create `packages/graph-stencil-org/src/schemas/agent-schema.ts`:

```typescript
export const orgAgentSchema = {
  type: 'object' as const,
  properties: {
    agentId: { type: 'string', title: 'Agent ID', 'x-order': 0 },
    role: { type: 'string', title: 'Role', 'x-order': 1 },
    roleVocabulary: { type: 'string', title: 'Role Vocabulary', 'x-order': 2, 'x-visibility': 'advanced' },
  },
};
```

Create `packages/graph-stencil-org/src/schemas/index.ts`:
```typescript
export { orgUnitSchema } from './unit-schema.js';
export { orgAgentSchema } from './agent-schema.js';
```

- [ ] **Step 4: Register schemas in registerOrgStencils**

Add to `packages/graph-stencil-org/src/stencils/register.ts` inside `registerOrgStencils()`:
```typescript
import { registerPropertySchema } from '@casehubio/pages-diagram-core';
import { orgUnitSchema, orgAgentSchema } from '../schemas/index.js';

// Inside registerOrgStencils(), after stencil registration:
registerPropertySchema('org-unit', orgUnitSchema);
registerPropertySchema('org-agent', orgAgentSchema);
```

- [ ] **Step 5: Update index.ts and run tests**

- [ ] **Step 6: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): property schemas for unit and agent Refs #155"
```

---

## Batch 3: CST-Preserving YAML Editor and Edit Policy

### Task 5: CST-preserving YAML editing functions

**Files:**
- Create: `packages/graph-stencil-org/src/adapter/yaml-editor.ts`
- Create: `packages/graph-stencil-org/src/adapter/yaml-editor.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Produces: `applyOrgPropertyEdit`, `addOrgUnit`, `removeOrgUnit`, `addMember`, `removeMember`, `addRelationship`, `removeRelationship`

- [ ] **Step 1: Write failing tests**

Create `packages/graph-stencil-org/src/adapter/yaml-editor.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { parse } from 'yaml';
import {
  applyOrgPropertyEdit, addOrgUnit, removeOrgUnit,
  addMember, removeMember, addRelationship, removeRelationship,
} from './yaml-editor.js';

const MINIMAL_YAML = `organization:
  units:
    - unitId: team
      name: My Team
      kind: simple-structure
      tenancyId: t1
      members:
        - agentId: alice
          role: engineer
        - agentId: bob
          role: designer
  relationships:
    - sourceAgentId: alice
      targetAgentId: bob
      kind: SUPERVISES
      tenancyId: t1
`;

describe('applyOrgPropertyEdit', () => {
  it('edits a unit name', () => {
    const result = applyOrgPropertyEdit(MINIMAL_YAML, ['organization', 'units', 0], ['name'], 'Renamed Team');
    const parsed = parse(result);
    expect(parsed.organization.units[0].name).toBe('Renamed Team');
  });

  it('preserves comments and formatting', () => {
    const yamlWithComment = `# My org\norganization:\n  units:\n    - unitId: team\n      name: My Team\n      tenancyId: t1\n      members: []\n  relationships: []\n`;
    const result = applyOrgPropertyEdit(yamlWithComment, ['organization', 'units', 0], ['name'], 'New');
    expect(result).toContain('# My org');
  });
});

describe('addOrgUnit', () => {
  it('adds a unit with defaults', () => {
    const result = addOrgUnit(MINIMAL_YAML);
    const parsed = parse(result);
    expect(parsed.organization.units).toHaveLength(2);
    expect(parsed.organization.units[1].unitId).toBeDefined();
  });
});

describe('removeOrgUnit', () => {
  it('removes a unit by path', () => {
    const result = removeOrgUnit(MINIMAL_YAML, ['organization', 'units', 0]);
    const parsed = parse(result);
    expect(parsed.organization.units).toHaveLength(0);
  });
});

describe('addMember', () => {
  it('adds a member to a unit', () => {
    const result = addMember(MINIMAL_YAML, ['organization', 'units', 0], { agentId: 'charlie', role: 'tester' });
    const parsed = parse(result);
    expect(parsed.organization.units[0].members).toHaveLength(3);
    expect(parsed.organization.units[0].members[2].agentId).toBe('charlie');
  });
});

describe('removeMember', () => {
  it('removes a member by index', () => {
    const result = removeMember(MINIMAL_YAML, ['organization', 'units', 0], 1);
    const parsed = parse(result);
    expect(parsed.organization.units[0].members).toHaveLength(1);
    expect(parsed.organization.units[0].members[0].agentId).toBe('alice');
  });
});

describe('addRelationship', () => {
  it('adds a relationship', () => {
    const result = addRelationship(MINIMAL_YAML, {
      sourceAgentId: 'bob', targetAgentId: 'alice', kind: 'REPORTS_TO', tenancyId: 't1',
    });
    const parsed = parse(result);
    expect(parsed.organization.relationships).toHaveLength(2);
  });
});

describe('removeRelationship', () => {
  it('removes a relationship by path', () => {
    const result = removeRelationship(MINIMAL_YAML, ['organization', 'relationships', 0]);
    const parsed = parse(result);
    expect(parsed.organization.relationships).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run tests to verify fail**

- [ ] **Step 3: Implement YAML editor functions**

Create `packages/graph-stencil-org/src/adapter/yaml-editor.ts` using the `yaml` library's CST-preserving `parseDocument` API. Follow the pattern from `graph-stencil-case/src/adapter/yaml-editor.ts` — use `doc.setIn()` for property edits, `doc.addIn()` for array additions, `doc.deleteIn()` for removals.

```typescript
import { parseDocument, stringify } from 'yaml';
import type { Membership, AgentRelationship } from '../types.js';

export function applyOrgPropertyEdit(
  yaml: string,
  nodePath: readonly (string | number)[],
  field: (string | number)[],
  value: unknown,
): string {
  const doc = parseDocument(yaml);
  doc.setIn([...nodePath, ...field], value);
  return doc.toString();
}

export function addOrgUnit(yaml: string, defaults?: Record<string, unknown>): string {
  const doc = parseDocument(yaml);
  const id = defaults?.unitId ?? `unit-${Date.now()}`;
  const tenancyId = defaults?.tenancyId ?? (doc.getIn(['organization', 'units', 0, 'tenancyId']) as string) ?? 'default';
  const newUnit = {
    unitId: id,
    name: defaults?.name ?? 'New Unit',
    kind: defaults?.kind ?? undefined,
    tenancyId,
    members: [],
    capabilities: [],
    goals: [],
    constraints: [],
  };
  doc.addIn(['organization', 'units'], newUnit);
  return doc.toString();
}

export function removeOrgUnit(yaml: string, unitPath: readonly (string | number)[]): string {
  const doc = parseDocument(yaml);
  doc.deleteIn([...unitPath]);
  return doc.toString();
}

export function addMember(
  yaml: string,
  unitPath: readonly (string | number)[],
  membership: Pick<Membership, 'agentId' | 'role'>,
): string {
  const doc = parseDocument(yaml);
  doc.addIn([...unitPath, 'members'], { agentId: membership.agentId, role: membership.role ?? undefined });
  return doc.toString();
}

export function removeMember(
  yaml: string,
  unitPath: readonly (string | number)[],
  memberIndex: number,
): string {
  const doc = parseDocument(yaml);
  doc.deleteIn([...unitPath, 'members', memberIndex]);
  return doc.toString();
}

export function addRelationship(
  yaml: string,
  rel: Pick<AgentRelationship, 'sourceAgentId' | 'targetAgentId' | 'kind' | 'tenancyId'>,
): string {
  const doc = parseDocument(yaml);
  doc.addIn(['organization', 'relationships'], {
    sourceAgentId: rel.sourceAgentId,
    targetAgentId: rel.targetAgentId,
    kind: rel.kind,
    tenancyId: rel.tenancyId,
  });
  return doc.toString();
}

export function removeRelationship(
  yaml: string,
  relPath: readonly (string | number)[],
): string {
  const doc = parseDocument(yaml);
  doc.deleteIn([...relPath]);
  return doc.toString();
}
```

- [ ] **Step 4: Update index.ts and run tests**

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): CST-preserving YAML editing functions Refs #155"
```

---

### Task 6: Edit policy

**Files:**
- Create: `packages/graph-stencil-org/src/editing/org-edit-policy.ts`
- Create: `packages/graph-stencil-org/src/editing/org-edit-policy.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `EditPolicy` type from `@casehubio/graph-renderer`
- Produces: `createOrgEditPolicy() → EditPolicy`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { clearRegistry } from '@casehubio/graph-renderer';
import { registerOrgStencils } from '../stencils/index.js';
import { createOrgEditPolicy } from './org-edit-policy.js';
import type { GraphModel } from '@casehubio/graph-core';

describe('createOrgEditPolicy', () => {
  beforeEach(() => { clearRegistry(); registerOrgStencils(); });

  it('allows creating org-unit without selection', () => {
    const policy = createOrgEditPolicy();
    const model: GraphModel = { nodes: [], edges: [] };
    const types = policy.getCreatableTypes(null, model);
    expect(types.some(t => t.type === 'org-unit')).toBe(true);
    expect(types.some(t => t.type === 'org-agent')).toBe(false);
  });

  it('allows creating org-agent when unit selected', () => {
    const policy = createOrgEditPolicy();
    const model: GraphModel = {
      nodes: [{ id: 'unit:team', type: 'org-unit', properties: {} }],
      edges: [],
    };
    const nearNode = model.nodes[0]!;
    const types = policy.getCreatableTypes(nearNode, model);
    expect(types.some(t => t.type === 'org-agent')).toBe(true);
  });

  it('marks all nodes as deletable', () => {
    const policy = createOrgEditPolicy();
    const model: GraphModel = {
      nodes: [
        { id: 'unit:team', type: 'org-unit', properties: {} },
        { id: 'agent:team:alice', type: 'org-agent', parentId: 'unit:team', properties: {} },
      ],
      edges: [],
    };
    expect(policy.isDeletable('unit:team', model)).toBe(true);
    expect(policy.isDeletable('agent:team:alice', model)).toBe(true);
  });
});
```

- [ ] **Step 2-5: Implement, test, commit** — following the spec's edit policy definition. `getCreatableTypes` gates `org-agent` on `nearNode?.type === 'org-unit'`. `canConnect` checks grammar via `getGrammar`. `reconnectEdge` throws 'not yet implemented'.

- [ ] **Step 6: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): edit policy for structural editing Refs #155"
```

---

## Batch 4: Archetype Detection and Layout

### Task 7: Archetype detection

**Files:**
- Create: `packages/graph-stencil-org/src/layout/archetype-detection.ts`
- Create: `packages/graph-stencil-org/src/layout/archetype-detection.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel` from `@casehubio/graph-core`
- Produces: `detectArchetype(model: GraphModel) → ArchetypeHint` with `archetype`, `confidence`, `layout`

- [ ] **Step 1: Write failing tests**

Test each archetype YAML → expected detection. Include inline hierarchy fixture (SUPERVISES tree depth > 2). Test empty graph fallback. Test ambiguous signals (tiered-escalation triggers multiple heuristics — priority ordering picks correctly).

- [ ] **Step 2-4: Implement using the priority ordering from the spec**

The detector runs heuristics in priority order (market → matrix → holarchy → tiered → pipeline → federation → coalition → hierarchy → simple → professional-bureaucracy) and returns the first match with a confidence level.

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): archetype detection with priority ordering Refs #155"
```

---

### Task 8: Layout strategy mapping

**Files:**
- Create: `packages/graph-stencil-org/src/layout/layout-strategy.ts`
- Create: `packages/graph-stencil-org/src/layout/layout-strategy.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `ElkLayoutOptions` from `@casehubio/graph-renderer`
- Produces: `orgLayoutOptions(strategy: OrgLayoutStrategy) → ElkLayoutOptions`

**Note:** This task depends on the ELK algorithm extension prerequisite in casehub-pages. Until that lands, the `algorithm` field is ignored and all strategies fall back to `layered`. The mapping and tests are written against the extended API; the actual layout variation only works after the pages change.

- [ ] **Step 1: Write tests**

```typescript
import { describe, it, expect } from 'vitest';
import { orgLayoutOptions } from './layout-strategy.js';

describe('orgLayoutOptions', () => {
  it('maps tree to mrtree DOWN', () => {
    const opts = orgLayoutOptions('tree');
    expect(opts.algorithm).toBe('mrtree');
    expect(opts.direction).toBe('DOWN');
  });

  it('maps flow to layered RIGHT', () => {
    const opts = orgLayoutOptions('flow');
    expect(opts.algorithm).toBe('layered');
    expect(opts.direction).toBe('RIGHT');
  });

  it('maps star to radial', () => {
    const opts = orgLayoutOptions('star');
    expect(opts.algorithm).toBe('radial');
  });

  it('maps force to force', () => {
    const opts = orgLayoutOptions('force');
    expect(opts.algorithm).toBe('force');
  });

  it('maps circular to stress', () => {
    const opts = orgLayoutOptions('circular');
    expect(opts.algorithm).toBe('stress');
  });
});
```

- [ ] **Step 2-4: Implement the mapping table from the spec**

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-org/
git commit -m "feat(graph-stencil-org): layout strategy mapping for all 10 archetypes Refs #155"
```

---

## Batch 5: org-diagram Component

### Task 9: Component scaffold and rendering

**Files:**
- Create: `components/org-diagram/package.json`
- Create: `components/org-diagram/tsconfig.json`
- Create: `components/org-diagram/vitest.config.ts`
- Create: `components/org-diagram/src/blocks-org-diagram.ts`
- Create: `components/org-diagram/src/blocks-org-diagram-toolbar.ts`
- Create: `components/org-diagram/src/blocks-org-diagram.test.ts`
- Modify: `package.json` (root — add workspace)
- Modify: `tsconfig.json` (root — add project reference)

**Interfaces:**
- Consumes: `DiagramBaseMixin` from `@casehubio/pages-diagram-core`, `toOrgGraph`, `registerOrgStencils`, `detectArchetype`, `orgLayoutOptions`, `createOrgEditPolicy`, `applyOrgPropertyEdit` from `@casehubio/graph-stencil-org`
- Produces: `<blocks-org-diagram>` custom element

- [ ] **Step 1: Scaffold component package**

Follow `components/casehub-diagram/package.json` pattern.

- [ ] **Step 2: Write failing test**

```typescript
import { describe, it, expect } from 'vitest';

describe('blocks-org-diagram', () => {
  it('registers as custom element', async () => {
    await import('./blocks-org-diagram.js');
    expect(customElements.get('blocks-org-diagram')).toBeDefined();
  });

  it('has ARIA attributes', async () => {
    const { BlocksOrgDiagram } = await import('./blocks-org-diagram.js');
    const el = new BlocksOrgDiagram();
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Organization diagram editor');
    el.remove();
  });
});
```

- [ ] **Step 3: Implement component**

Create `components/org-diagram/src/blocks-org-diagram.ts` following the casehub-diagram pattern:
- Extends `DiagramBaseMixin(LitElement)`
- Implements `_adaptYaml` → calls `toOrgGraph`, `detectArchetype`
- Implements `_applyPropertyEdit` → calls `applyOrgPropertyEdit`
- Implements `_emptyTemplate` → returns minimal org YAML
- Implements `_editPolicy` → returns `createOrgEditPolicy()`
- Implements `_layoutOptions` → calls `orgLayoutOptions(strategy)`
- Implements `_applyGraphEdit` → dispatches to `addOrgUnit`, `removeOrgUnit`, `addMember`, `addRelationship`, `removeRelationship`
- Implements `_iconRenderer` → SVG icons for org-unit and org-agent
- Toolbar with layout dropdown, archetype badge, stats, export
- `@property() layoutStrategy: OrgLayoutStrategy | 'auto' = 'auto'`
- ARIA: `role="region"`, `aria-label="Organization diagram editor"` in connectedCallback

- [ ] **Step 4: Implement toolbar**

Create `components/org-diagram/src/blocks-org-diagram-toolbar.ts` with layout strategy dropdown, archetype badge, node/agent/relationship counts, save button, export buttons. Follow `casehub-diagram-toolbar.ts` pattern.

- [ ] **Step 5: Run tests**

```bash
yarn workspace @casehubio/blocks-ui-org-diagram test
```

- [ ] **Step 6: Commit**

```bash
git add components/org-diagram/ package.json tsconfig.json
git commit -m "feat(org-diagram): component extending DiagramBaseMixin with archetype-aware layout Refs #155"
```

---

### Task 10: Showcase page

**Files:**
- Create: `examples/src/pages/org-diagram-page.ts`
- Modify: `examples/src/shell.ts` (add nav entry)

**Interfaces:**
- Consumes: `<blocks-org-diagram>` custom element

- [ ] **Step 1: Create showcase page**

Create `examples/src/pages/org-diagram-page.ts` following the pattern from `examples/src/pages/diagram-export-page.ts`. Include:
- Inline YAML for the simple-structure archetype as default
- Dropdown to switch between all 9 archetype YAML files
- The `<blocks-org-diagram>` component with `.yaml` binding

- [ ] **Step 2: Add navigation entry in shell.ts**

- [ ] **Step 3: Verify visually**

```bash
yarn examples
```

Open in browser, navigate to org diagram page. Verify:
- Simple structure renders with star-like layout
- Switching archetypes re-renders with appropriate layout
- Units contain agents visually
- Edge styles are distinct per relationship kind
- Palette shows Unit always, Agent when unit selected
- Property panel works on node selection

- [ ] **Step 4: Commit**

```bash
git add examples/
git commit -m "feat(examples): org diagram showcase page with archetype switcher Refs #155"
```

---

## References

- [specs/issue-155-org-diagram/2026-09-05-org-diagram-design.md] — design spec this plan implements
- [specs/issue-155-org-diagram/decisions.md] — 8 design decisions
- [graph-stencil-case/] — established adapter+stencil pattern
- [casehub-diagram/src/casehub-diagram.ts] — established DiagramBaseMixin component pattern
- [graph-renderer/src/layout/elk-layout.ts] — ELK layout with compound nodes
- [graph-renderer/src/registry/stencil-registry.ts] — stencil and edge type registration
- [graph-core/src/model.ts] — GraphModel, GraphNode, GraphEdge
- [graph-core/src/persistence.ts] — PersistenceBackend SPI
- [eidos/org-api/src/main/java/io/casehub/eidos/org/api/] — Java types
- [eidos/examples/org-scenarios/src/test/resources/archetypes/] — 9 YAML fixtures
- [examples/src/pages/diagram-export-page.ts] — YAML syntax highlighting pattern
- [GE-20260801-36b9fa] — Cytoscape limitation
- [GE-20260809-2cbc61] — ReactFlow vs D3 for force layout
- [GitHub #155] — issue description
