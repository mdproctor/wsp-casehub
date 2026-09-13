# Rich Org Diagram Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #157 — feat: rich org diagram — full agent properties, relationship types, attestation, escalation chains, and legend
**Issue group:** #157

**Goal:** Enrich the existing org-diagram with full information density — rich agent cards, labeled edges, internal panels, hover tooltips, collapsible units, and relationship highlighting.

**Architecture:** Decompose the adapter into pure functions chained by the component: `toOrgGraph` (unchanged) → `enrichWithDescriptors` → `computeDerivedData` → `resolveKindColors` → `applyCollapsedUnits` → `computeNodeSizes`. Edge post-processing via `applyOrgEdgeLabels` + `applySelectionHighlight`. Panels are internal Lit elements in `org-diagram/src/panels/`.

**Tech Stack:** TypeScript, Lit 3, ReactFlow (@xyflow/react), ELK, Vitest

## Global Constraints

- All new types go in `packages/graph-stencil-org/src/types.ts`
- All new adapter functions export from `packages/graph-stencil-org/src/index.ts`
- Stencils access node data via typed cast: `const data = node.properties as OrgAgentNodeData`
- Long text values truncated with CSS `text-overflow: ellipsis` — no wrapping, keeps height formula deterministic
- ARIA mandatory on all new interactive elements
- Every `blocks-*` component needs Props interface + BlocksComponentRegistry entry
- Commits reference `Refs #157`

---

## Batch 1: Adapter enrichment pipeline

### Task 1: Types and enrichWithDescriptors

**Files:**
- Modify: `packages/graph-stencil-org/src/types.ts`
- Create: `packages/graph-stencil-org/src/adapter/enrichment.ts`
- Create: `packages/graph-stencil-org/src/adapter/enrichment.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel`, `GraphNode` from `@casehubio/graph-core`; existing types from `types.ts`
- Produces: `AgentDescriptor`, `DispositionAxes`, `OrgAgentNodeData`, `OrgUnitNodeData`, `enrichWithDescriptors(model, agents) → GraphModel`

- [ ] **Step 1: Add new types to types.ts**

Add to `packages/graph-stencil-org/src/types.ts`:

```typescript
export interface DispositionAxes {
  autonomy?: string;
  ruleFollowing?: string;
  socialOrient?: string;
  riskAppetite?: string;
  conflictMode?: string;
}

export interface AgentDescriptor {
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  goals?: AgentGoal[];
  constraints?: AgentConstraint[];
  briefing?: string;
}

export interface OrgAgentNodeData {
  agentId: string;
  role?: string;
  roleVocabulary?: string;
  unitId: string;
  label: string;
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  supervisionTargets?: string[];
  escalationChain?: string[];
  backupAgents?: { agentId: string; scope?: string; direction: 'backs' | 'backed-by' }[];
  attestationGrants?: { targetAgentId: string; scope?: string; dimensions: string[]; signalTypes?: string[] }[];
  unitKind?: string;
  unitColorStart?: string;
  unitColorEnd?: string;
}

export interface OrgUnitNodeData {
  unitId: string;
  name: string;
  kind?: string;
  kindVocabulary?: string;
  label: string;
  memberCount: number;
  capabilities: AgentCapability[];
  goals: AgentGoal[];
  constraints: AgentConstraint[];
  kindColorStart?: string;
  kindColorEnd?: string;
}
```

- [ ] **Step 2: Write failing tests for enrichWithDescriptors**

Create `packages/graph-stencil-org/src/adapter/enrichment.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { enrichWithDescriptors } from './enrichment.js';
import { toOrgGraph } from './org-adapter.js';
import type { AgentDescriptor } from '../types.js';

const SIMPLE_YAML = `
organization:
  units:
    - unitId: team1
      name: Team One
      kind: rig
      tenancyId: t1
      members:
        - agentId: alice
          role: worker
        - agentId: bob
          role: lead
      capabilities: []
      goals: []
      constraints: []
  relationships: []
`;

describe('enrichWithDescriptors', () => {
  it('merges descriptor into agent node properties', () => {
    const base = toOrgGraph(SIMPLE_YAML);
    const agents: Record<string, AgentDescriptor> = {
      alice: { slot: 'worker', disposition: { autonomy: 'semi-auto' } },
    };
    const enriched = enrichWithDescriptors(base.model, agents);
    const alice = enriched.nodes.find(n => n.properties['agentId'] === 'alice');
    expect(alice!.properties['slot']).toBe('worker');
    expect((alice!.properties['disposition'] as any).autonomy).toBe('semi-auto');
  });

  it('leaves agent unmodified when descriptor missing', () => {
    const base = toOrgGraph(SIMPLE_YAML);
    const agents: Record<string, AgentDescriptor> = {};
    const enriched = enrichWithDescriptors(base.model, agents);
    const bob = enriched.nodes.find(n => n.properties['agentId'] === 'bob');
    expect(bob!.properties['slot']).toBeUndefined();
  });

  it('does not modify unit nodes', () => {
    const base = toOrgGraph(SIMPLE_YAML);
    const agents: Record<string, AgentDescriptor> = { alice: { slot: 'x' } };
    const enriched = enrichWithDescriptors(base.model, agents);
    const unit = enriched.nodes.find(n => n.type === 'org-unit');
    expect(unit!.properties['slot']).toBeUndefined();
  });

  it('preserves original model (returns new model)', () => {
    const base = toOrgGraph(SIMPLE_YAML);
    const agents: Record<string, AgentDescriptor> = { alice: { slot: 'x' } };
    const enriched = enrichWithDescriptors(base.model, agents);
    const origAlice = base.model.nodes.find(n => n.properties['agentId'] === 'alice');
    expect(origAlice!.properties['slot']).toBeUndefined();
    expect(enriched).not.toBe(base.model);
  });
});
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/enrichment.test.ts`
Expected: FAIL — `enrichWithDescriptors` not found

- [ ] **Step 4: Implement enrichWithDescriptors**

Create `packages/graph-stencil-org/src/adapter/enrichment.ts`:

```typescript
import type { GraphModel, GraphNode } from '@casehubio/graph-core';
import type { AgentDescriptor } from '../types.js';

export function enrichWithDescriptors(
  model: GraphModel,
  agents: Readonly<Record<string, AgentDescriptor>>,
): GraphModel {
  const nodes = model.nodes.map((node): GraphNode => {
    if (node.type !== 'org-agent') return node;
    const agentId = node.properties['agentId'] as string;
    const desc = agents[agentId];
    if (!desc) return node;
    return {
      ...node,
      properties: {
        ...node.properties,
        ...(desc.slot !== undefined ? { slot: desc.slot } : {}),
        ...(desc.capabilities?.length ? { capabilities: desc.capabilities } : {}),
        ...(desc.disposition ? { disposition: desc.disposition } : {}),
        ...(desc.goals?.length ? { goals: desc.goals } : {}),
        ...(desc.constraints?.length ? { constraints: desc.constraints } : {}),
        ...(desc.briefing !== undefined ? { briefing: desc.briefing } : {}),
      },
    };
  });
  return { nodes, edges: model.edges };
}
```

- [ ] **Step 5: Run tests — verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/enrichment.test.ts`
Expected: PASS

- [ ] **Step 6: Update index.ts exports**

Add to `packages/graph-stencil-org/src/index.ts`:

```typescript
export type { AgentDescriptor, DispositionAxes, OrgAgentNodeData, OrgUnitNodeData } from './types.js';
export { enrichWithDescriptors } from './adapter/enrichment.js';
```

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add packages/graph-stencil-org/src/types.ts packages/graph-stencil-org/src/adapter/enrichment.ts packages/graph-stencil-org/src/adapter/enrichment.test.ts packages/graph-stencil-org/src/index.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(graph-stencil-org): add AgentDescriptor types and enrichWithDescriptors Refs #157"
```

---

### Task 2: computeDerivedData — supervision, escalation, backup, attestation

**Files:**
- Create: `packages/graph-stencil-org/src/adapter/derived-data.ts`
- Create: `packages/graph-stencil-org/src/adapter/derived-data.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel` from `@casehubio/graph-core`
- Produces: `computeDerivedData(model) → DerivedOrgData` where `DerivedOrgData = { model: GraphModel; escalationChains: {path: string[], terminal: string}[]; attestationSummary: {source, target, scope?, dimensions, signalTypes?}[] }`

- [ ] **Step 1: Write failing tests**

Create `packages/graph-stencil-org/src/adapter/derived-data.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { computeDerivedData } from './derived-data.js';
import { toOrgGraph } from './org-adapter.js';
import { readFileSync } from 'fs';
import { resolve } from 'path';

const ARCHETYPES_DIR = resolve(
  __dirname,
  '../../../../../eidos/examples/org-scenarios/src/test/resources/archetypes',
);

const GASTOWN_YAML = readFileSync(
  resolve(__dirname, '../../../../../eidos/org-runtime/src/test/resources/gastown-org.yaml'),
  'utf-8',
);

describe('computeDerivedData', () => {
  it('computes supervision targets', () => {
    const base = toOrgGraph(GASTOWN_YAML);
    const { model } = computeDerivedData(base.model);
    const witnessAlpha = model.nodes.find(n => n.properties['agentId'] === 'witness-alpha');
    const targets = witnessAlpha!.properties['supervisionTargets'] as string[];
    expect(targets).toEqual(expect.arrayContaining(['polecat-1', 'polecat-2']));
  });

  it('computes backup agents with scope', () => {
    const base = toOrgGraph(GASTOWN_YAML);
    const { model } = computeDerivedData(base.model);
    const polecat1 = model.nodes.find(n => n.properties['agentId'] === 'polecat-1');
    const backups = polecat1!.properties['backupAgents'] as any[];
    expect(backups).toHaveLength(1);
    expect(backups[0]).toMatchObject({
      agentId: 'polecat-2',
      scope: 'code-analysis',
      direction: 'backs',
    });
  });

  it('computes attestation grants from any relationship kind', () => {
    const base = toOrgGraph(GASTOWN_YAML);
    const { model, attestationSummary } = computeDerivedData(base.model);
    expect(attestationSummary.length).toBeGreaterThan(0);
    expect(attestationSummary[0]).toMatchObject({
      source: 'deacon',
      target: 'witness-alpha',
      scope: 'rig-monitoring',
      dimensions: ['LATENCY', 'ATTESTATION_RATE'],
    });
  });

  it('detects escalation cycle without infinite loop', () => {
    const cycleYaml = `
organization:
  units:
    - unitId: u1
      name: U1
      tenancyId: t1
      members:
        - agentId: a1
        - agentId: a2
        - agentId: a3
      capabilities: []
      goals: []
      constraints: []
  relationships:
    - sourceAgentId: a1
      targetAgentId: a2
      kind: ESCALATES_TO
      tenancyId: t1
    - sourceAgentId: a2
      targetAgentId: a3
      kind: ESCALATES_TO
      tenancyId: t1
    - sourceAgentId: a3
      targetAgentId: a1
      kind: ESCALATES_TO
      tenancyId: t1
`;
    const base = toOrgGraph(cycleYaml);
    const { model, escalationChains } = computeDerivedData(base.model);
    const a1 = model.nodes.find(n => n.properties['agentId'] === 'a1');
    const chain = a1!.properties['escalationChain'] as string[];
    expect(chain.length).toBeLessThanOrEqual(4);
  });

  it('returns escalation chains for panel', () => {
    const escYaml = `
organization:
  units:
    - unitId: u1
      name: U1
      tenancyId: t1
      members:
        - agentId: worker1
        - agentId: supervisor
        - agentId: boss
      capabilities: []
      goals: []
      constraints: []
  relationships:
    - sourceAgentId: worker1
      targetAgentId: supervisor
      kind: ESCALATES_TO
      tenancyId: t1
    - sourceAgentId: supervisor
      targetAgentId: boss
      kind: ESCALATES_TO
      tenancyId: t1
`;
    const base = toOrgGraph(escYaml);
    const { escalationChains } = computeDerivedData(base.model);
    expect(escalationChains.length).toBeGreaterThan(0);
    const chain = escalationChains.find(c => c.path[0] === 'worker1');
    expect(chain).toBeDefined();
    expect(chain!.path).toEqual(['worker1', 'supervisor', 'boss']);
    expect(chain!.terminal).toBe('boss');
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/derived-data.test.ts`
Expected: FAIL

- [ ] **Step 3: Implement computeDerivedData**

Create `packages/graph-stencil-org/src/adapter/derived-data.ts`:

```typescript
import type { GraphModel, GraphNode, GraphEdge } from '@casehubio/graph-core';
import type { RelationshipScope, AttestationGrant } from '../types.js';

export interface DerivedOrgData {
  model: GraphModel;
  escalationChains: readonly { path: string[]; terminal: string }[];
  attestationSummary: readonly {
    source: string; target: string;
    scope?: string; dimensions: string[];
    signalTypes?: string[];
  }[];
}

function extractAgentId(nodeId: string): string {
  const parts = nodeId.split(':');
  return parts.length >= 3 ? parts[2]! : nodeId;
}

export function computeDerivedData(model: GraphModel): DerivedOrgData {
  const agentNodes = model.nodes.filter(n => n.type === 'org-agent');
  const nodeIdToAgentId = new Map(agentNodes.map(n => [n.id, n.properties['agentId'] as string]));

  // Build per-agent edge indices
  const supervisionTargets = new Map<string, string[]>();
  const escalatesTo = new Map<string, string>();
  const backupEdges: { source: string; target: string; scope?: string }[] = [];
  const attestations: DerivedOrgData['attestationSummary'][number][] = [];

  for (const edge of model.edges) {
    const srcAgent = nodeIdToAgentId.get(edge.source);
    const tgtAgent = nodeIdToAgentId.get(edge.target);
    if (!srcAgent || !tgtAgent) continue;
    const kind = edge.properties?.['kind'] as string | undefined;
    const scope = edge.properties?.['scope'] as RelationshipScope | undefined;
    const attestation = edge.properties?.['attestation'] as AttestationGrant | undefined;

    if (kind === 'SUPERVISES') {
      let targets = supervisionTargets.get(srcAgent);
      if (!targets) { targets = []; supervisionTargets.set(srcAgent, targets); }
      targets.push(tgtAgent);
    }
    if (kind === 'ESCALATES_TO') {
      escalatesTo.set(srcAgent, tgtAgent);
    }
    if (kind === 'BACKS_UP') {
      backupEdges.push({ source: srcAgent, target: tgtAgent, scope: scope?.capabilityName });
    }
    if (attestation) {
      attestations.push({
        source: srcAgent, target: tgtAgent,
        scope: scope?.capabilityName,
        dimensions: [...attestation.dimensions],
        signalTypes: attestation.signalTypes ? [...attestation.signalTypes] : undefined,
      });
    }
  }

  // Walk escalation chains (with cycle detection)
  function walkEscalation(start: string): { chain: string[]; terminal: string } {
    const chain = [start];
    const visited = new Set([start]);
    let current = start;
    while (escalatesTo.has(current)) {
      const next = escalatesTo.get(current)!;
      if (visited.has(next)) {
        return { chain, terminal: `${next} (cycle)` };
      }
      chain.push(next);
      visited.add(next);
      current = next;
    }
    return { chain, terminal: current };
  }

  // Build escalation chains from leaf agents (those that escalate but are not targets of escalation)
  const escalationTargetSet = new Set(escalatesTo.values());
  const leafEscalators: string[] = [];
  for (const src of escalatesTo.keys()) {
    if (!escalationTargetSet.has(src)) leafEscalators.push(src);
  }
  const escalationChains = leafEscalators.map(leaf => {
    const { chain, terminal } = walkEscalation(leaf);
    return { path: chain, terminal };
  });

  // Compute per-agent backup
  const agentBackups = new Map<string, { agentId: string; scope?: string; direction: 'backs' | 'backed-by' }[]>();
  for (const { source, target, scope } of backupEdges) {
    let srcList = agentBackups.get(source);
    if (!srcList) { srcList = []; agentBackups.set(source, srcList); }
    srcList.push({ agentId: target, scope, direction: 'backs' });
    let tgtList = agentBackups.get(target);
    if (!tgtList) { tgtList = []; agentBackups.set(target, tgtList); }
    tgtList.push({ agentId: source, scope, direction: 'backed-by' });
  }

  // Compute per-agent attestation grants (from edges where this agent is the source)
  const agentAttestations = new Map<string, { targetAgentId: string; scope?: string; dimensions: string[]; signalTypes?: string[] }[]>();
  for (const att of attestations) {
    let list = agentAttestations.get(att.source);
    if (!list) { list = []; agentAttestations.set(att.source, list); }
    list.push({ targetAgentId: att.target, scope: att.scope, dimensions: att.dimensions, signalTypes: att.signalTypes });
  }

  // Enrich agent nodes
  const nodes = model.nodes.map((node): GraphNode => {
    if (node.type !== 'org-agent') return node;
    const agentId = node.properties['agentId'] as string;
    const props: Record<string, unknown> = { ...node.properties };
    const targets = supervisionTargets.get(agentId);
    if (targets?.length) props['supervisionTargets'] = targets;
    const { chain } = walkEscalation(agentId);
    if (chain.length > 1) props['escalationChain'] = chain.slice(1);
    const backups = agentBackups.get(agentId);
    if (backups?.length) props['backupAgents'] = backups;
    const grants = agentAttestations.get(agentId);
    if (grants?.length) props['attestationGrants'] = grants;
    return { ...node, properties: props };
  });

  return {
    model: { nodes, edges: model.edges },
    escalationChains,
    attestationSummary: attestations,
  };
}
```

- [ ] **Step 4: Run tests — verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/derived-data.test.ts`
Expected: PASS

- [ ] **Step 5: Update index.ts exports**

Add to `packages/graph-stencil-org/src/index.ts`:

```typescript
export { computeDerivedData } from './adapter/derived-data.js';
export type { DerivedOrgData } from './adapter/derived-data.js';
```

- [ ] **Step 6: Run all existing tests to check for regressions**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run`
Expected: All pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add packages/graph-stencil-org/src/adapter/derived-data.ts packages/graph-stencil-org/src/adapter/derived-data.test.ts packages/graph-stencil-org/src/index.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(graph-stencil-org): add computeDerivedData — escalation chains, supervision, backup, attestation Refs #157"
```

---

## Batch 2: Visual pipeline helpers

### Task 3: resolveKindColors, computeNodeSizes, applyCollapsedUnits

**Files:**
- Create: `packages/graph-stencil-org/src/adapter/kind-colors.ts`
- Create: `packages/graph-stencil-org/src/adapter/node-sizing.ts`
- Create: `packages/graph-stencil-org/src/adapter/collapse.ts`
- Create: `packages/graph-stencil-org/src/adapter/kind-colors.test.ts`
- Create: `packages/graph-stencil-org/src/adapter/node-sizing.test.ts`
- Create: `packages/graph-stencil-org/src/adapter/collapse.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `GraphModel` from `@casehubio/graph-core`; `OrgAdapterResult` from `org-adapter.ts`
- Produces: `resolveKindColors(model, kindColors?) → GraphModel`; `computeNodeSizes(model) → ReadonlyMap<string, {width, height}>`; `applyCollapsedUnits(model, yamlPaths, collapsed) → {model, yamlPaths}`; `DEFAULT_KIND_PALETTE`; `DISPOSITION_SHORT_NAMES`

- [ ] **Step 1: Write failing tests for resolveKindColors**

Create `packages/graph-stencil-org/src/adapter/kind-colors.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { resolveKindColors, DEFAULT_KIND_PALETTE } from './kind-colors.js';
import { toOrgGraph } from './org-adapter.js';

const YAML = `
organization:
  units:
    - unitId: oversight
      name: Oversight
      kind: supervision-hierarchy
      tenancyId: t1
      members:
        - agentId: bot
          role: lead
      capabilities: []
      goals: []
      constraints: []
    - unitId: rig1
      name: Rig One
      kind: rig
      tenancyId: t1
      members:
        - agentId: worker1
          role: worker
      capabilities: []
      goals: []
      constraints: []
  relationships: []
`;

describe('resolveKindColors', () => {
  it('applies default palette for known kinds', () => {
    const base = toOrgGraph(YAML);
    const model = resolveKindColors(base.model);
    const oversight = model.nodes.find(n => n.properties['unitId'] === 'oversight');
    expect(oversight!.properties['kindColorStart']).toBe(DEFAULT_KIND_PALETTE['supervision-hierarchy']!.start);
  });

  it('propagates unit colors to agent nodes', () => {
    const base = toOrgGraph(YAML);
    const model = resolveKindColors(base.model);
    const worker = model.nodes.find(n => n.properties['agentId'] === 'worker1');
    expect(worker!.properties['unitKind']).toBe('rig');
    expect(worker!.properties['unitColorStart']).toBe(DEFAULT_KIND_PALETTE['rig']!.start);
  });

  it('applies custom kindColors override', () => {
    const base = toOrgGraph(YAML);
    const custom = { rig: { start: '#ff0000', end: '#00ff00' } };
    const model = resolveKindColors(base.model, custom);
    const rig = model.nodes.find(n => n.properties['unitId'] === 'rig1');
    expect(rig!.properties['kindColorStart']).toBe('#ff0000');
  });

  it('auto-assigns unknown kinds from palette rotation', () => {
    const unknownYaml = `
organization:
  units:
    - unitId: u1
      name: U1
      kind: custom-kind-xyz
      tenancyId: t1
      members: []
      capabilities: []
      goals: []
      constraints: []
  relationships: []
`;
    const base = toOrgGraph(unknownYaml);
    const model = resolveKindColors(base.model);
    const u1 = model.nodes.find(n => n.properties['unitId'] === 'u1');
    expect(u1!.properties['kindColorStart']).toBeDefined();
  });
});
```

- [ ] **Step 2: Write failing tests for computeNodeSizes**

Create `packages/graph-stencil-org/src/adapter/node-sizing.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { computeNodeSizes, AGENT_WIDTH } from './node-sizing.js';
import type { GraphModel } from '@casehubio/graph-core';

describe('computeNodeSizes', () => {
  it('returns min height for agent with no enrichment', () => {
    const model: GraphModel = {
      nodes: [{ id: 'agent:u1:a1', type: 'org-agent', parentId: 'unit:u1', properties: { agentId: 'a1', label: 'a1' } }],
      edges: [],
    };
    const sizes = computeNodeSizes(model);
    const size = sizes.get('agent:u1:a1');
    expect(size).toBeDefined();
    expect(size!.width).toBe(AGENT_WIDTH);
    expect(size!.height).toBeLessThan(60);
  });

  it('returns larger height for fully enriched agent', () => {
    const model: GraphModel = {
      nodes: [{
        id: 'agent:u1:a1', type: 'org-agent', parentId: 'unit:u1',
        properties: {
          agentId: 'a1', label: 'a1',
          slot: 'worker',
          capabilities: [{ name: 'cap1' }],
          disposition: { autonomy: 'high' },
          supervisionTargets: ['b1'],
          escalationChain: ['sup', 'boss'],
          backupAgents: [{ agentId: 'b1', direction: 'backs' }],
          attestationGrants: [{ targetAgentId: 'b1', dimensions: ['LAT'] }],
        },
      }],
      edges: [],
    };
    const sizes = computeNodeSizes(model);
    const size = sizes.get('agent:u1:a1');
    expect(size!.height).toBeGreaterThan(120);
  });

  it('computes unit header height including capability pills', () => {
    const model: GraphModel = {
      nodes: [{
        id: 'unit:u1', type: 'org-unit',
        properties: { unitId: 'u1', name: 'U1', label: 'U1', memberCount: 2, capabilities: [{ name: 'c1' }], goals: [], constraints: [] },
      }],
      edges: [],
    };
    const sizes = computeNodeSizes(model);
    const size = sizes.get('unit:u1');
    expect(size).toBeDefined();
    expect(size!.height).toBeGreaterThan(30);
  });
});
```

- [ ] **Step 3: Write failing tests for applyCollapsedUnits**

Create `packages/graph-stencil-org/src/adapter/collapse.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { applyCollapsedUnits } from './collapse.js';
import { toOrgGraph } from './org-adapter.js';

const YAML = `
organization:
  units:
    - unitId: team1
      name: Team One
      kind: rig
      tenancyId: t1
      members:
        - agentId: alice
          role: worker
        - agentId: bob
          role: lead
      capabilities: []
      goals: []
      constraints: []
  relationships:
    - sourceAgentId: bob
      targetAgentId: alice
      kind: SUPERVISES
      tenancyId: t1
`;

describe('applyCollapsedUnits', () => {
  it('removes agent nodes from collapsed units', () => {
    const base = toOrgGraph(YAML);
    const collapsed = new Set(['team1']);
    const result = applyCollapsedUnits(base.model, base.yamlPaths, collapsed);
    const agents = result.model.nodes.filter(n => n.type === 'org-agent');
    expect(agents).toHaveLength(0);
  });

  it('removes edges connected to collapsed agents', () => {
    const base = toOrgGraph(YAML);
    const collapsed = new Set(['team1']);
    const result = applyCollapsedUnits(base.model, base.yamlPaths, collapsed);
    expect(result.model.edges).toHaveLength(0);
  });

  it('preserves unit node for collapsed unit', () => {
    const base = toOrgGraph(YAML);
    const collapsed = new Set(['team1']);
    const result = applyCollapsedUnits(base.model, base.yamlPaths, collapsed);
    const units = result.model.nodes.filter(n => n.type === 'org-unit');
    expect(units).toHaveLength(1);
  });
});
```

- [ ] **Step 4: Run all three test files — verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/kind-colors.test.ts src/adapter/node-sizing.test.ts src/adapter/collapse.test.ts`
Expected: FAIL

- [ ] **Step 5: Implement resolveKindColors**

Create `packages/graph-stencil-org/src/adapter/kind-colors.ts`:

```typescript
import type { GraphModel, GraphNode } from '@casehubio/graph-core';

export const DEFAULT_KIND_PALETTE: Readonly<Record<string, { start: string; end: string }>> = {
  'supervision-hierarchy': { start: '#553c9a', end: '#6b46c1' },
  'rig': { start: '#2c5282', end: '#3182ce' },
  'department': { start: '#276749', end: '#38a169' },
  'holarchy': { start: '#3730a3', end: '#4f46e5' },
  'team': { start: '#9c4221', end: '#dd6b20' },
};

const FALLBACK_PALETTE = [
  { start: '#065f46', end: '#059669' },
  { start: '#92400e', end: '#d97706' },
  { start: '#831843', end: '#db2777' },
  { start: '#1e3a5f', end: '#3b82f6' },
];

export function resolveKindColors(
  model: GraphModel,
  kindColors?: Readonly<Record<string, { start: string; end: string }>>,
): GraphModel {
  const merged = { ...DEFAULT_KIND_PALETTE, ...kindColors };
  const unitKindMap = new Map<string, { kind?: string; start: string; end: string }>();
  let fallbackIdx = 0;

  for (const node of model.nodes) {
    if (node.type !== 'org-unit') continue;
    const kind = node.properties['kind'] as string | undefined;
    const key = kind ?? '__none__';
    if (!unitKindMap.has(key)) {
      const colors = kind && merged[kind]
        ? merged[kind]!
        : FALLBACK_PALETTE[fallbackIdx++ % FALLBACK_PALETTE.length]!;
      unitKindMap.set(key, { kind, start: colors.start, end: colors.end });
    }
  }

  const unitNodeColors = new Map<string, { kind?: string; start: string; end: string }>();
  for (const node of model.nodes) {
    if (node.type !== 'org-unit') continue;
    const kind = node.properties['kind'] as string | undefined;
    const colors = unitKindMap.get(kind ?? '__none__')!;
    unitNodeColors.set(node.id, colors);
  }

  const nodes = model.nodes.map((node): GraphNode => {
    if (node.type === 'org-unit') {
      const colors = unitNodeColors.get(node.id)!;
      return { ...node, properties: { ...node.properties, kindColorStart: colors.start, kindColorEnd: colors.end } };
    }
    if (node.type === 'org-agent' && node.parentId) {
      const parentColors = unitNodeColors.get(node.parentId);
      if (parentColors) {
        return {
          ...node,
          properties: {
            ...node.properties,
            unitKind: parentColors.kind,
            unitColorStart: parentColors.start,
            unitColorEnd: parentColors.end,
          },
        };
      }
    }
    return node;
  });

  return { nodes, edges: model.edges };
}
```

- [ ] **Step 6: Implement computeNodeSizes**

Create `packages/graph-stencil-org/src/adapter/node-sizing.ts`:

```typescript
import type { GraphModel } from '@casehubio/graph-core';

const ROW_HEIGHT = 16;
const HEADER_HEIGHT = 28;
const DISPOSITION_ROW_HEIGHT = 18;
const ATTESTATION_HEIGHT = 32;
const PADDING = 12;
export const AGENT_WIDTH = 280;

const UNIT_HEADER_HEIGHT = 38;
const UNIT_PILLS_ROW = 20;
const UNIT_MIN_WIDTH = 300;

export const DISPOSITION_SHORT_NAMES: Readonly<Record<string, string>> = {
  autonomy: 'autonomy',
  ruleFollowing: 'rules',
  socialOrient: 'social',
  riskAppetite: 'risk',
  conflictMode: 'conflict',
};

export function computeNodeSizes(
  model: GraphModel,
): ReadonlyMap<string, { width: number; height: number }> {
  const sizes = new Map<string, { width: number; height: number }>();

  for (const node of model.nodes) {
    if (node.type === 'org-agent') {
      const p = node.properties;
      let h = HEADER_HEIGHT + PADDING;
      if (p['slot']) h += ROW_HEIGHT;
      if ((p['capabilities'] as unknown[] | undefined)?.length) h += ROW_HEIGHT;
      const disp = p['disposition'] as Record<string, string> | undefined;
      if (disp && Object.keys(disp).length > 0) h += DISPOSITION_ROW_HEIGHT;
      if ((p['supervisionTargets'] as string[] | undefined)?.length) h += ROW_HEIGHT;
      if ((p['escalationChain'] as string[] | undefined)?.length) h += ROW_HEIGHT;
      if ((p['backupAgents'] as unknown[] | undefined)?.length) h += ROW_HEIGHT;
      if ((p['attestationGrants'] as unknown[] | undefined)?.length) h += ATTESTATION_HEIGHT;
      sizes.set(node.id, { width: AGENT_WIDTH, height: h });
    } else if (node.type === 'org-unit') {
      let h = UNIT_HEADER_HEIGHT;
      if ((node.properties['capabilities'] as unknown[] | undefined)?.length) h += UNIT_PILLS_ROW;
      sizes.set(node.id, { width: UNIT_MIN_WIDTH, height: h });
    }
  }
  return sizes;
}
```

- [ ] **Step 7: Implement applyCollapsedUnits**

Create `packages/graph-stencil-org/src/adapter/collapse.ts`:

```typescript
import type { GraphModel } from '@casehubio/graph-core';

export function applyCollapsedUnits(
  model: GraphModel,
  yamlPaths: ReadonlyMap<string, readonly (string | number)[]>,
  collapsedUnits: ReadonlySet<string>,
): { model: GraphModel; yamlPaths: ReadonlyMap<string, readonly (string | number)[]> } {
  const collapsedNodeIds = new Set<string>();
  for (const node of model.nodes) {
    if (node.type === 'org-agent' && node.parentId) {
      const parentUnitId = node.parentId.replace(/^unit:/, '');
      if (collapsedUnits.has(parentUnitId)) {
        collapsedNodeIds.add(node.id);
      }
    }
  }

  const nodes = model.nodes.filter(n => !collapsedNodeIds.has(n.id));
  const edges = model.edges.filter(
    e => !collapsedNodeIds.has(e.source) && !collapsedNodeIds.has(e.target),
  );

  const filteredPaths = new Map<string, readonly (string | number)[]>();
  for (const [key, path] of yamlPaths) {
    if (!collapsedNodeIds.has(key)) filteredPaths.set(key, path);
  }

  return { model: { nodes, edges }, yamlPaths: filteredPaths };
}
```

- [ ] **Step 8: Run tests — verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/kind-colors.test.ts src/adapter/node-sizing.test.ts src/adapter/collapse.test.ts`
Expected: PASS

- [ ] **Step 9: Update index.ts exports**

Add to `packages/graph-stencil-org/src/index.ts`:

```typescript
export { resolveKindColors, DEFAULT_KIND_PALETTE } from './adapter/kind-colors.js';
export { computeNodeSizes, AGENT_WIDTH, DISPOSITION_SHORT_NAMES } from './adapter/node-sizing.js';
export { applyCollapsedUnits } from './adapter/collapse.js';
```

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add packages/graph-stencil-org/src/adapter/kind-colors.ts packages/graph-stencil-org/src/adapter/kind-colors.test.ts packages/graph-stencil-org/src/adapter/node-sizing.ts packages/graph-stencil-org/src/adapter/node-sizing.test.ts packages/graph-stencil-org/src/adapter/collapse.ts packages/graph-stencil-org/src/adapter/collapse.test.ts packages/graph-stencil-org/src/index.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(graph-stencil-org): add color resolution, node sizing, and collapse filtering Refs #157"
```

---

### Task 4: Edge label and selection highlight post-processors

**Files:**
- Create: `packages/graph-stencil-org/src/adapter/edge-labels.ts`
- Create: `packages/graph-stencil-org/src/adapter/edge-labels.test.ts`
- Create: `packages/graph-stencil-org/src/adapter/selection-highlight.ts`
- Create: `packages/graph-stencil-org/src/adapter/selection-highlight.test.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`

**Interfaces:**
- Consumes: `Edge` from `@xyflow/react` (ReactFlow edge type)
- Produces: `applyOrgEdgeLabels(edges) → Edge[]`; `applySelectionHighlight(edges, selectedNodeId?) → Edge[]`

- [ ] **Step 1: Write failing tests for applyOrgEdgeLabels**

Create `packages/graph-stencil-org/src/adapter/edge-labels.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { applyOrgEdgeLabels } from './edge-labels.js';
import type { Edge } from '@xyflow/react';

describe('applyOrgEdgeLabels', () => {
  it('adds scope label to scoped SUPERVISES edge', () => {
    const edges: Edge[] = [{
      id: 'e1', type: 'org-supervises', source: 'a', target: 'b',
      data: { scope: { capabilityName: 'rig-monitoring' } },
    }];
    const result = applyOrgEdgeLabels(edges);
    expect(result[0]!.label).toBe('scope: rig-monitoring');
    expect(result[0]!.labelBgStyle).toBeDefined();
  });

  it('adds no label to unscoped SUPERVISES edge', () => {
    const edges: Edge[] = [{
      id: 'e1', type: 'org-supervises', source: 'a', target: 'b', data: {},
    }];
    const result = applyOrgEdgeLabels(edges);
    expect(result[0]!.label).toBeUndefined();
  });

  it('adds BACKS_UP label with scope', () => {
    const edges: Edge[] = [{
      id: 'e1', type: 'org-backs-up', source: 'a', target: 'b',
      data: { scope: { capabilityName: 'code-analysis' } },
    }];
    const result = applyOrgEdgeLabels(edges);
    expect(result[0]!.label).toContain('BACKS_UP');
    expect(result[0]!.label).toContain('code-analysis');
  });

  it('adds DELEGATES_TO label always', () => {
    const edges: Edge[] = [{
      id: 'e1', type: 'org-delegates-to', source: 'a', target: 'b', data: {},
    }];
    const result = applyOrgEdgeLabels(edges);
    expect(result[0]!.label).toBe('DELEGATES_TO');
  });

  it('adds no label to ESCALATES_TO', () => {
    const edges: Edge[] = [{
      id: 'e1', type: 'org-escalates-to', source: 'a', target: 'b', data: {},
    }];
    const result = applyOrgEdgeLabels(edges);
    expect(result[0]!.label).toBeUndefined();
  });
});
```

- [ ] **Step 2: Write failing tests for applySelectionHighlight**

Create `packages/graph-stencil-org/src/adapter/selection-highlight.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { applySelectionHighlight } from './selection-highlight.js';
import type { Edge } from '@xyflow/react';

describe('applySelectionHighlight', () => {
  const edges: Edge[] = [
    { id: 'e1', source: 'agent:u1:alice', target: 'agent:u1:bob' },
    { id: 'e2', source: 'agent:u1:bob', target: 'agent:u1:carol' },
  ];

  it('highlights edges connected to selected node', () => {
    const result = applySelectionHighlight(edges, 'agent:u1:alice');
    expect(result[0]!.className).toContain('org-edge-highlighted');
    expect(result[1]!.style?.opacity).toBe(0.15);
  });

  it('restores all edges when no selection', () => {
    const result = applySelectionHighlight(edges, undefined);
    expect(result[0]!.className).toBeUndefined();
    expect(result[0]!.style?.opacity).toBeUndefined();
  });

  it('highlights edges on both sides of selected node', () => {
    const result = applySelectionHighlight(edges, 'agent:u1:bob');
    expect(result[0]!.className).toContain('org-edge-highlighted');
    expect(result[1]!.className).toContain('org-edge-highlighted');
  });
});
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/edge-labels.test.ts src/adapter/selection-highlight.test.ts`
Expected: FAIL

- [ ] **Step 4: Implement applyOrgEdgeLabels**

Create `packages/graph-stencil-org/src/adapter/edge-labels.ts`:

```typescript
import type { Edge } from '@xyflow/react';

interface ScopeData { capabilityName?: string; }

const LABEL_STYLES: Record<string, { fill: string; stroke: string; color: string }> = {
  'org-supervises': { fill: '#e9d8fd', stroke: '#d6bcfa', color: '#553c9a' },
  'org-delegates-to': { fill: '#e6fffa', stroke: '#81e6d9', color: '#2c7a7b' },
  'org-backs-up': { fill: '#ebf8ff', stroke: '#90cdf4', color: '#2b6cb0' },
  'org-extended': { fill: '#f3e8ff', stroke: '#c4b5fd', color: '#6d28d9' },
};

export function applyOrgEdgeLabels(edges: readonly Edge[]): Edge[] {
  return edges.map(edge => {
    const type = edge.type ?? '';
    const scope = (edge.data as Record<string, unknown> | undefined)?.['scope'] as ScopeData | undefined;
    const extendedKind = (edge.data as Record<string, unknown> | undefined)?.['extendedKind'] as string | undefined;

    let label: string | undefined;
    if (type === 'org-supervises' && scope?.capabilityName) {
      label = `scope: ${scope.capabilityName}`;
    } else if (type === 'org-delegates-to') {
      label = 'DELEGATES_TO';
    } else if (type === 'org-backs-up') {
      label = scope?.capabilityName ? `BACKS_UP (scope: ${scope.capabilityName})` : 'BACKS_UP';
    } else if (type === 'org-extended' && extendedKind) {
      label = extendedKind;
    }

    if (!label) return edge;
    const style = LABEL_STYLES[type];
    if (!style) return { ...edge, label };
    return {
      ...edge,
      label,
      labelStyle: { fontSize: 8, fontWeight: 600, fill: style.color },
      labelBgStyle: { fill: style.fill, stroke: style.stroke, strokeWidth: 0.6 },
      labelBgPadding: [3, 6] as [number, number],
      labelBgBorderRadius: 3,
    };
  });
}
```

- [ ] **Step 5: Implement applySelectionHighlight**

Create `packages/graph-stencil-org/src/adapter/selection-highlight.ts`:

```typescript
import type { Edge } from '@xyflow/react';

export function applySelectionHighlight(
  edges: readonly Edge[],
  selectedNodeId?: string,
): Edge[] {
  if (!selectedNodeId) return edges.map(e => ({ ...e }));
  return edges.map(edge => {
    const connected = edge.source === selectedNodeId || edge.target === selectedNodeId;
    if (connected) {
      return { ...edge, className: 'org-edge-highlighted' };
    }
    return { ...edge, style: { ...edge.style, opacity: 0.15 } };
  });
}
```

- [ ] **Step 6: Run tests — verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/adapter/edge-labels.test.ts src/adapter/selection-highlight.test.ts`
Expected: PASS

- [ ] **Step 7: Update index.ts and commit**

Add exports, then:
```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add packages/graph-stencil-org/src/adapter/edge-labels.ts packages/graph-stencil-org/src/adapter/edge-labels.test.ts packages/graph-stencil-org/src/adapter/selection-highlight.ts packages/graph-stencil-org/src/adapter/selection-highlight.test.ts packages/graph-stencil-org/src/index.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(graph-stencil-org): add edge label and selection highlight post-processors Refs #157"
```

---

## Batch 3: Rich stencils

### Task 5: Rich org-agent and org-unit stencils

**Files:**
- Modify: `packages/graph-stencil-org/src/stencils/org-agent.ts`
- Modify: `packages/graph-stencil-org/src/stencils/org-unit.ts`
- Modify: `packages/graph-stencil-org/src/stencils/stencils.test.ts`

**Interfaces:**
- Consumes: `OrgAgentNodeData`, `OrgUnitNodeData` from `types.ts`; `DISPOSITION_SHORT_NAMES` from `node-sizing.ts`
- Produces: `renderOrgAgent(node, decoration) → TemplateResult`; `renderOrgUnit(node, decoration) → TemplateResult` (same signatures, richer output)

- [ ] **Step 1: Write failing tests for rich agent card rendering**

Add to `packages/graph-stencil-org/src/stencils/stencils.test.ts`:

```typescript
import { renderOrgAgent, renderOrgUnit } from './index.js';
import type { GraphNode } from '@casehubio/graph-core';

describe('renderOrgAgent — rich card', () => {
  it('renders disposition pills when present', () => {
    const node: GraphNode = {
      id: 'agent:u1:a1', type: 'org-agent', parentId: 'unit:u1',
      properties: {
        agentId: 'polecat-1', role: 'worker', label: 'polecat-1',
        unitId: 'u1',
        disposition: { autonomy: 'semi-auto', ruleFollowing: 'principled' },
        unitColorStart: '#2c5282', unitColorEnd: '#3182ce', unitKind: 'rig',
      },
    };
    const result = renderOrgAgent(node);
    const html = (result as any).strings?.join('') ?? String(result);
    expect(html).toContain('autonomy');
    expect(html).toContain('semi-auto');
  });

  it('renders escalation chain in red', () => {
    const node: GraphNode = {
      id: 'agent:u1:a1', type: 'org-agent', parentId: 'unit:u1',
      properties: {
        agentId: 'polecat-1', label: 'polecat-1', unitId: 'u1',
        escalationChain: ['witness-alpha', 'deacon', 'boot'],
        unitColorStart: '#2c5282', unitColorEnd: '#3182ce',
      },
    };
    const result = renderOrgAgent(node);
    const html = (result as any).strings?.join('') ?? String(result);
    expect(html).toContain('ESCALATES');
  });

  it('degrades gracefully with no enrichment', () => {
    const node: GraphNode = {
      id: 'agent:u1:a1', type: 'org-agent', parentId: 'unit:u1',
      properties: { agentId: 'simple', role: 'worker', label: 'simple', unitId: 'u1' },
    };
    const result = renderOrgAgent(node);
    const html = (result as any).strings?.join('') ?? String(result);
    expect(html).toContain('simple');
  });
});
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/stencils/stencils.test.ts`
Expected: FAIL (new tests fail)

- [ ] **Step 3: Implement rich org-agent stencil**

Replace content of `packages/graph-stencil-org/src/stencils/org-agent.ts` with the full multi-row card rendering using `OrgAgentNodeData` typed cast. Include: header with colored circle + role badge, conditional rows for SLOT, CAPS, DISPOSITION pills, SUPERVISES, ESCALATES (red), BACKUP, ATTESTATION (purple pills). All text values use `overflow:hidden;text-overflow:ellipsis;white-space:nowrap` for truncation.

Reference: spec §Rich Agent Card Stencil for exact colors and layout. Reference SVG for visual target.

- [ ] **Step 4: Implement rich org-unit stencil**

Replace content of `packages/graph-stencil-org/src/stencils/org-unit.ts` with gradient header using `kindColorStart`/`kindColorEnd`, kind badge, member count, capability pills row below header. The unit name is white on gradient background.

Reference: spec §Rich Unit Container Stencil for exact colors. Reference SVG unit containers.

- [ ] **Step 5: Run tests — verify they pass**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run src/stencils/stencils.test.ts`
Expected: PASS

- [ ] **Step 6: Run all graph-stencil-org tests**

Run: `yarn workspace @casehubio/graph-stencil-org run test -- --run`
Expected: All pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add packages/graph-stencil-org/src/stencils/org-agent.ts packages/graph-stencil-org/src/stencils/org-unit.ts packages/graph-stencil-org/src/stencils/stencils.test.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(graph-stencil-org): rich agent cards and unit containers with full information density Refs #157"
```

---

## Batch 4: Component integration, panels, tooltip, registry

### Task 6: Wire pipeline into blocks-org-diagram

**Files:**
- Modify: `components/org-diagram/src/blocks-org-diagram.ts`
- Modify: `components/org-diagram/src/blocks-org-diagram.test.ts`

**Interfaces:**
- Consumes: All functions from Tasks 1-4 via `@casehubio/graph-stencil-org`
- Produces: Updated `BlocksOrgDiagram` with `agents`, `kindColors` properties, full pipeline in `_fullRender`, lightweight `_updateEdgeStyles`, collapsible units state

- [ ] **Step 1: Add new properties and imports**

In `blocks-org-diagram.ts`, add:
- `@property({ type: Object }) agents: Record<string, AgentDescriptor> | undefined`
- `@property({ type: Object }) kindColors: Record<string, {start: string, end: string}> | undefined`
- `@state() private _collapsedUnits = new Set<string>()`
- `@state() private _baseEdges: Edge[] = []`
- `@state() private _derivedData: DerivedOrgData | null = null`
- Import all new functions from `@casehubio/graph-stencil-org`

- [ ] **Step 2: Rewrite _fullRender to use the decomposed pipeline**

Replace the `_fullRender` method to chain: `toOrgGraph` → `enrichWithDescriptors` (if agents set) → `computeDerivedData` → `resolveKindColors` → `applyCollapsedUnits` (if any collapsed) → `computeNodeSizes` → `computeElkLayout` → `toReactFlowGraph` → cache `_baseEdges` → `_updateEdgeStyles()`.

Store `_derivedData` for panels.

- [ ] **Step 3: Add lightweight _updateEdgeStyles method**

```typescript
private _updateEdgeStyles(): void {
  let edges = applyOrgEdgeLabels(this._baseEdges);
  edges = applySelectionHighlight(edges, this._selectedNodeId || undefined);
  (this as any)._edges = edges;
}
```

Call this from selection change handler instead of `_fullRender`.

- [ ] **Step 4: Add unit collapse toggle handler**

Handle click on unit headers to toggle `_collapsedUnits` and re-run `_fullRender`.

- [ ] **Step 5: Write tests for new properties**

Add to `blocks-org-diagram.test.ts`:
- Test that setting `agents` triggers enriched rendering
- Test that setting `kindColors` overrides unit colors
- Test `aria-expanded` on collapsible unit headers

- [ ] **Step 6: Run tests**

Run: `yarn workspace blocks-org-diagram run test -- --run`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add components/org-diagram/src/blocks-org-diagram.ts components/org-diagram/src/blocks-org-diagram.test.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(org-diagram): wire decomposed adapter pipeline with agents, kindColors, collapse Refs #157"
```

---

### Task 7: Internal panels and hover tooltip

**Files:**
- Create: `components/org-diagram/src/panels/escalation-chain-panel.ts`
- Create: `components/org-diagram/src/panels/attestation-panel.ts`
- Create: `components/org-diagram/src/panels/org-legend.ts`
- Create: `components/org-diagram/src/panels/tooltip.ts`
- Modify: `components/org-diagram/src/blocks-org-diagram.ts` (compose panels + tooltip in render)
- Modify: `components/org-diagram/src/blocks-org-diagram.test.ts`

**Interfaces:**
- Consumes: `DerivedOrgData` from Task 2; `DEFAULT_KIND_PALETTE` from Task 3; `OrgAgentNodeData` from Task 1
- Produces: `OrgEscalationChainPanel`, `OrgAttestationPanel`, `OrgLegend`, `OrgTooltip` (internal Lit elements)

- [ ] **Step 1: Write failing tests for panels**

Add panel tests to `blocks-org-diagram.test.ts`:
- Escalation chain panel renders paths with arrows
- Attestation panel renders dimension pills
- Legend renders relationship line samples and kind swatches

- [ ] **Step 2: Run tests — verify they fail**

- [ ] **Step 3: Implement escalation-chain-panel.ts**

Lit element rendering red-tinted panel with chain paths. `role="region"`, `aria-label="Escalation chains"`. Emits `agent-click` event.

- [ ] **Step 4: Implement attestation-panel.ts**

Lit element rendering purple-tinted panel with grants. Dimension pills, signal pills, scope text.

- [ ] **Step 5: Implement org-legend.ts**

Lit element with 6 sections: relationships (SVG line samples), unit kinds (gradient swatches), agent properties (dot indicators), disposition axes (color swatches), scope & attestation (pill samples), eidos model layers (text).

- [ ] **Step 6: Implement tooltip.ts**

Lit element rendering a floating card with agent properties (slot, disposition pills, capabilities, goals, constraints, briefing truncated). `role="tooltip"`. Positioned above the hovered node, clamped to viewport. 150ms dismiss delay.

- [ ] **Step 7: Compose panels and tooltip in blocks-org-diagram render()**

Add panels below the canvas in the component's render method. Wire `_derivedData` to panel properties. Add mouseover/mouseout handlers for tooltip. Wire panel `agent-click` events to node selection.

- [ ] **Step 8: Run tests — verify they pass**

Run: `yarn workspace blocks-org-diagram run test -- --run`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add components/org-diagram/src/panels/ components/org-diagram/src/blocks-org-diagram.ts components/org-diagram/src/blocks-org-diagram.test.ts
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(org-diagram): add escalation chain, attestation, legend panels and hover tooltip Refs #157"
```

---

### Task 8: OrgDiagramProps registry entry and exports

**Files:**
- Modify: `components/org-diagram/src/index.ts`
- Modify: `packages/blocks-ui-schema/src/registry.ts`

**Interfaces:**
- Consumes: `BlocksComponentRegistry` from `registry.ts`
- Produces: `OrgDiagramProps` interface exported from org-diagram index

- [ ] **Step 1: Export OrgDiagramProps from org-diagram index**

Add to `components/org-diagram/src/index.ts`:

```typescript
export type { AgentDescriptor, OrgLayoutStrategy } from '@casehubio/graph-stencil-org';

export interface OrgDiagramProps {
  yaml?: string;
  src?: string;
  agents?: Record<string, import('@casehubio/graph-stencil-org').AgentDescriptor>;
  kindColors?: Record<string, { start: string; end: string }>;
  layoutStrategy?: import('@casehubio/graph-stencil-org').OrgLayoutStrategy | 'auto';
  selectionTopic?: string;
  readonly?: boolean;
}
```

- [ ] **Step 2: Add BlocksComponentRegistry entry**

In `packages/blocks-ui-schema/src/registry.ts`, add:

```typescript
'blocks-org-diagram': OrgDiagramProps;
```

Import `OrgDiagramProps` from the org-diagram package.

- [ ] **Step 3: Regenerate Zod schemas**

Run: `yarn workspace @casehubio/blocks-ui-schema run generate`
Expected: Schema generated without errors

- [ ] **Step 4: Run schema tests**

Run: `yarn workspace @casehubio/blocks-ui-schema run test -- --run`
Expected: PASS (completeness test passes with new entry)

- [ ] **Step 5: Run full build**

Run: `yarn build && yarn test && yarn typecheck`
Expected: All pass

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui add components/org-diagram/src/index.ts packages/blocks-ui-schema/src/registry.ts packages/blocks-ui-schema/src/
git -C /Users/mdproctor/claude/casehub/slots/175/blocks-ui commit -m "feat(org-diagram): add OrgDiagramProps to BlocksComponentRegistry Refs #157"
```

---

## References

- [specs/issue-157-rich-org-diagram/2026-09-08-rich-org-diagram-design.md] — design spec this plan implements
- [specs/issue-157-rich-org-diagram/decisions.md] — 10 design decisions
- [packages/graph-stencil-org/src/adapter/org-adapter.ts] — existing adapter (toOrgGraph)
- [packages/graph-stencil-org/src/stencils/org-agent.ts] — current minimal agent stencil
- [packages/graph-stencil-org/src/stencils/org-unit.ts] — current minimal unit stencil
- [packages/graph-stencil-org/src/types.ts] — existing org domain types
- [components/org-diagram/src/blocks-org-diagram.ts] — existing diagram component
- [eidos/org-runtime/src/test/resources/gastown-org.yaml] — test fixture for rich rendering
- [eidos/docs/diagrams/gastown-org-structure.svg] — reference SVG target
- [docs/protocols/blocks-ui/node-decoration-runtime-boundary.md] — descriptor data goes in properties
- [docs/protocols/blocks-ui/component-registry-props.md] — registry entry required
- [GitHub #157] — issue with full requirements
