# Layout Rule Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #157 — Rich org diagram
**Issue group:** #157

**Goal:** Replace scattered layout logic (5+ files) with a forward-chaining rule engine that gives deterministic layout: same graph → same facts → same rules fire → same layout.

**Architecture:** Two-stage engine. `preLayout(model)` runs classification rules (producing facts) and sizing rules (producing node dimensions), then selects a layout strategy from the fact base. `postLayout(nodes, edges, facts)` runs internal-layout, container-positioning, and edge-routing transform rules, then verifies hard constraints. The component calls ELK/radial layout between the two stages. All existing layout functions become rules with preconditions, guarantees, and fact-base integration. Old scattered files are deleted.

**Tech Stack:** TypeScript, vitest, ELK (via graph-renderer)

## Global Constraints

- All new code in `packages/graph-stencil-org/src/layout/`
- Tests use vitest (`describe`/`it`/`expect`)
- Existing YAML fixtures from archetype test files at `eidos/examples/org-scenarios/src/test/resources/archetypes/` are used for archetype composition tests
- No new external dependencies — engine is standalone
- `LayoutNode`/`LayoutEdge` types must remain compatible with React Flow Node/Edge shapes (superset cast)
- Public API changes must be reflected in `packages/graph-stencil-org/src/index.ts`
- After integration, all existing tests must pass: `yarn test`, `yarn typecheck`
- Regression check: org diagram (4 archetypes), casehub diagram, SWF diagram, diagram workbench must render correctly in examples dev server

---

## Batch 1: Engine framework — types, FactBase, engine core

### Task 1: Rule engine types and FactBase

**Files:**
- Create: `packages/graph-stencil-org/src/layout/types.ts`
- Create: `packages/graph-stencil-org/src/layout/fact-base.ts`
- Test: `packages/graph-stencil-org/src/layout/fact-base.test.ts`

**Interfaces:**
- Consumes: `GraphModel` from `@casehubio/graph-core`
- Produces: `Phase`, `Fact`, `FactBase`, `ClassificationRule`, `LayoutRule`, `HardConstraint`, `CompositionError`, `CompositionReport`, `LayoutExplanation`, `ScopeContext`, `LayoutNode`, `LayoutEdge`, `LayoutViolation`, `ArchetypeName`, `OrgLayoutStrategy`, `ArchetypeHint`, `OrgElkLayoutOptions`, `ElkAlgorithm`, `createFactBase()`

- [ ] **Step 1: Write failing tests for FactBase**

```typescript
// fact-base.test.ts
import { describe, it, expect } from 'vitest';
import { createFactBase } from './fact-base.js';

describe('FactBase', () => {
  it('asserts and retrieves a fact', () => {
    const fb = createFactBase();
    fb.assert('unit:oversight', 'is-linear-chain', true);
    expect(fb.has('unit:oversight', 'is-linear-chain')).toBe(true);
    expect(fb.get('unit:oversight', 'is-linear-chain')).toBe(true);
  });

  it('returns false for missing facts', () => {
    const fb = createFactBase();
    expect(fb.has('unit:x', 'is-leaf')).toBe(false);
    expect(fb.get('unit:x', 'is-leaf')).toBeUndefined();
  });

  it('queries all facts by predicate', () => {
    const fb = createFactBase();
    fb.assert('agent:a1', 'is-leaf', true);
    fb.assert('agent:a2', 'is-leaf', true);
    fb.assert('agent:a3', 'has-backup', true);
    const leafs = fb.query('is-leaf');
    expect(leafs).toHaveLength(2);
    expect(leafs.map(f => f.subject)).toContain('agent:a1');
    expect(leafs.map(f => f.subject)).toContain('agent:a2');
  });

  it('returns all facts', () => {
    const fb = createFactBase();
    fb.assert('graph', 'archetype', 'federation');
    fb.assert('graph', 'strategy', 'hub-spoke');
    expect(fb.facts()).toHaveLength(2);
  });

  it('overwrites existing fact with same subject+predicate', () => {
    const fb = createFactBase();
    fb.assert('graph', 'strategy', 'tree');
    fb.assert('graph', 'strategy', 'layered');
    expect(fb.get('graph', 'strategy')).toBe('layered');
    expect(fb.facts()).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn --cwd packages/graph-stencil-org test -- --run fact-base.test`
Expected: FAIL — modules not found

- [ ] **Step 3: Write types.ts with all rule engine types**

```typescript
// layout/types.ts
import type { GraphModel } from '@casehubio/graph-core';

export type Phase = 'classification' | 'sizing' | 'internal-layout' | 'container-positioning' | 'edge-routing';

export interface Fact {
  readonly subject: string;
  readonly predicate: string;
  readonly value?: unknown;
}

export interface FactBase {
  assert(subject: string, predicate: string, value?: unknown): void;
  has(subject: string, predicate: string): boolean;
  get(subject: string, predicate: string): unknown;
  query(predicate: string): Array<{ subject: string; value: unknown }>;
  facts(): readonly Fact[];
}

export interface ClassificationRule {
  readonly id: string;
  classify(model: GraphModel, facts: FactBase): void;
}

export interface LayoutRule {
  readonly id: string;
  readonly phase: Exclude<Phase, 'classification'>;
  readonly group?: string;
  readonly priority: number;
  readonly scope: 'global' | 'container' | 'edge';
  readonly requires?: readonly string[];
  readonly guarantees?: readonly string[];
  precondition(facts: FactBase): boolean;
  apply(nodes: LayoutNode[], edges: LayoutEdge[], facts: FactBase): void;
}

export interface HardConstraint {
  readonly id: string;
  check(nodes: readonly LayoutNode[], edges: readonly LayoutEdge[]): LayoutViolation[];
}

export interface LayoutNode {
  id: string;
  type?: string | undefined;
  parentId?: string | undefined;
  position: { x: number; y: number };
  width?: number | undefined;
  height?: number | undefined;
  style?: Record<string, unknown> | undefined;
}

export interface LayoutEdge {
  id: string;
  type?: string;
  source: string;
  target: string;
  sourceHandle?: string;
  targetHandle?: string;
  data?: Record<string, unknown>;
  [key: string]: unknown;
}

export interface LayoutViolation {
  rule: string;
  severity: 'hard' | 'soft';
  message: string;
  nodeIds?: string[];
  edgeIds?: string[];
}

export interface CompositionError {
  type: 'missing-provider' | 'circular-dependency' | 'conflicting-guarantees';
  message: string;
  ruleIds: string[];
}

export interface CompositionReport {
  valid: boolean;
  errors: CompositionError[];
  warnings: string[];
  selectedRules: Map<Phase, LayoutRule[]>;
}

export interface RuleSelection {
  group: string;
  candidates: Array<{ rule: string; applicable: boolean; priority: number }>;
  selected: string;
  reason: string;
}

export interface LayoutExplanation {
  classifications: readonly Fact[];
  ruleSelections: RuleSelection[];
  violations: LayoutViolation[];
}

export type ArchetypeName =
  | 'simple-structure'
  | 'hierarchy'
  | 'professional-bureaucracy'
  | 'tiered-escalation'
  | 'divisional-holarchy'
  | 'federation'
  | 'pipeline'
  | 'coalition'
  | 'matrix'
  | 'market';

export type OrgLayoutStrategy =
  | 'star' | 'tree' | 'circular' | 'layered' | 'nested'
  | 'hub-spoke' | 'flow' | 'radial' | 'grid' | 'force';

export interface ArchetypeHint {
  archetype: ArchetypeName;
  confidence: 'high' | 'medium' | 'low';
  layout: OrgLayoutStrategy;
}

export type ElkAlgorithm = 'layered' | 'mrtree' | 'radial' | 'force' | 'stress';

export interface OrgElkLayoutOptions {
  algorithm: ElkAlgorithm;
  direction?: 'DOWN' | 'RIGHT' | 'LEFT' | 'UP';
  spacing: number;
  containerPadding?: number;
  elkOptions?: Readonly<Record<string, string>>;
}
```

- [ ] **Step 4: Write FactBase implementation**

```typescript
// layout/fact-base.ts
import type { Fact, FactBase } from './types.js';

export function createFactBase(): FactBase {
  const store = new Map<string, Fact>();

  function key(subject: string, predicate: string): string {
    return `${subject}\0${predicate}`;
  }

  return {
    assert(subject, predicate, value) {
      store.set(key(subject, predicate), { subject, predicate, value });
    },
    has(subject, predicate) {
      return store.has(key(subject, predicate));
    },
    get(subject, predicate) {
      return store.get(key(subject, predicate))?.value;
    },
    query(predicate) {
      const results: Array<{ subject: string; value: unknown }> = [];
      for (const fact of store.values()) {
        if (fact.predicate === predicate) {
          results.push({ subject: fact.subject, value: fact.value });
        }
      }
      return results;
    },
    facts() {
      return [...store.values()];
    },
  };
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn --cwd packages/graph-stencil-org test -- --run fact-base.test`
Expected: PASS — all 5 tests

- [ ] **Step 6: Commit**

```
feat(graph-stencil-org): add layout rule engine types and FactBase

Refs casehubio/blocks-ui#157
```

---

### Task 2: OrgLayoutEngine core

**Files:**
- Create: `packages/graph-stencil-org/src/layout/engine.ts`
- Test: `packages/graph-stencil-org/src/layout/engine.test.ts`

**Interfaces:**
- Consumes: `FactBase`, `ClassificationRule`, `LayoutRule`, `HardConstraint`, `CompositionReport`, `Phase`, `LayoutNode`, `LayoutEdge`, `LayoutViolation`, `ArchetypeName`, `OrgLayoutStrategy`, `OrgElkLayoutOptions`, `LayoutExplanation` from `./types.js`; `createFactBase()` from `./fact-base.js`; `GraphModel` from `@casehubio/graph-core`
- Produces: `OrgLayoutEngine` class with `register()`, `preLayout()`, `postLayout()`, `elkOptions()`, `validateComposition()`

- [ ] **Step 1: Write failing tests for engine core**

```typescript
// engine.test.ts
import { describe, it, expect } from 'vitest';
import { OrgLayoutEngine } from './engine.js';
import type { ClassificationRule, LayoutRule, HardConstraint, LayoutNode, LayoutEdge, FactBase } from './types.js';
import type { GraphModel } from '@casehubio/graph-core';

const EMPTY_MODEL: GraphModel = { nodes: [], edges: [] };

const stubClassifier: ClassificationRule = {
  id: 'test-classifier',
  classify(_model, facts) {
    facts.assert('graph', 'test-fact', true);
  },
};

const stubSizingRule: LayoutRule = {
  id: 'test-sizing',
  phase: 'sizing',
  priority: 1,
  scope: 'global',
  guarantees: ['nodes-sized'],
  precondition: () => true,
  apply(_nodes, _edges, facts) {
    facts.assert('graph', 'sized', true);
  },
};

const stubInternalRule: LayoutRule = {
  id: 'test-internal-a',
  phase: 'internal-layout',
  group: 'internal-layout',
  priority: 1,
  scope: 'global',
  requires: ['nodes-sized'],
  guarantees: ['agents-positioned'],
  precondition: () => true,
  apply(nodes) {
    for (const n of nodes) {
      if (n.type === 'org-agent') n.position = { x: 10, y: 10 };
    }
  },
};

const stubInternalRuleB: LayoutRule = {
  id: 'test-internal-b',
  phase: 'internal-layout',
  group: 'internal-layout',
  priority: 2,
  scope: 'global',
  guarantees: ['agents-positioned'],
  precondition: () => true,
  apply() { /* lower priority, should not fire */ },
};

const noOverlapConstraint: HardConstraint = {
  id: 'HR-test',
  check(nodes) {
    return nodes.length > 10
      ? [{ rule: 'HR-test', severity: 'hard', message: 'too many nodes', nodeIds: [] }]
      : [];
  },
};

describe('OrgLayoutEngine', () => {
  describe('preLayout', () => {
    it('runs classification rules and produces facts', () => {
      const engine = new OrgLayoutEngine();
      engine.register(stubClassifier);
      const result = engine.preLayout(EMPTY_MODEL);
      expect(result.facts.has('graph', 'test-fact')).toBe(true);
    });

    it('runs sizing rules during preLayout', () => {
      const engine = new OrgLayoutEngine();
      engine.register(stubSizingRule);
      const result = engine.preLayout(EMPTY_MODEL);
      expect(result.facts.has('graph', 'sized')).toBe(true);
    });

    it('returns strategy and archetype from facts', () => {
      const classifier: ClassificationRule = {
        id: 'arch',
        classify(_m, facts) {
          facts.assert('graph', 'archetype', 'federation');
          facts.assert('graph', 'archetype-confidence', 'high');
          facts.assert('graph', 'recommended-strategy', 'hub-spoke');
        },
      };
      const engine = new OrgLayoutEngine();
      engine.register(classifier);
      const result = engine.preLayout(EMPTY_MODEL);
      expect(result.strategy).toBe('hub-spoke');
      expect(result.archetype.archetype).toBe('federation');
    });

    it('defaults strategy to force when no archetype classified', () => {
      const engine = new OrgLayoutEngine();
      const result = engine.preLayout(EMPTY_MODEL);
      expect(result.strategy).toBe('force');
    });
  });

  describe('postLayout — group resolution', () => {
    it('fires highest-priority rule in a mutual-exclusion group', () => {
      const engine = new OrgLayoutEngine();
      engine.register(stubInternalRule);
      engine.register(stubInternalRuleB);
      const nodes: LayoutNode[] = [
        { id: 'a1', type: 'org-agent', parentId: 'u1', position: { x: 0, y: 0 } },
      ];
      const result = engine.postLayout(nodes, [], engine.preLayout(EMPTY_MODEL).facts);
      expect(nodes[0]!.position).toEqual({ x: 10, y: 10 });
    });

    it('skips group rule when precondition is false', () => {
      const conditional: LayoutRule = {
        id: 'conditional',
        phase: 'internal-layout',
        group: 'test-group',
        priority: 1,
        scope: 'global',
        precondition: (facts) => facts.has('graph', 'needs-special'),
        apply(nodes) { nodes[0]!.position = { x: 99, y: 99 }; },
      };
      const fallback: LayoutRule = {
        id: 'fallback',
        phase: 'internal-layout',
        group: 'test-group',
        priority: 2,
        scope: 'global',
        precondition: () => true,
        apply(nodes) { nodes[0]!.position = { x: 50, y: 50 }; },
      };
      const engine = new OrgLayoutEngine();
      engine.register(conditional);
      engine.register(fallback);
      const nodes: LayoutNode[] = [{ id: 'a1', position: { x: 0, y: 0 } }];
      engine.postLayout(nodes, [], engine.preLayout(EMPTY_MODEL).facts);
      expect(nodes[0]!.position).toEqual({ x: 50, y: 50 });
    });
  });

  describe('postLayout — hard constraints', () => {
    it('returns violations from hard constraints', () => {
      const engine = new OrgLayoutEngine();
      engine.register(noOverlapConstraint);
      const nodes: LayoutNode[] = Array.from({ length: 11 }, (_, i) => ({
        id: `n${i}`, position: { x: 0, y: 0 },
      }));
      const result = engine.postLayout(nodes, [], engine.preLayout(EMPTY_MODEL).facts);
      expect(result.violations).toHaveLength(1);
      expect(result.violations[0]!.rule).toBe('HR-test');
    });
  });

  describe('postLayout — phase ordering', () => {
    it('runs phases in order: internal-layout, container-positioning, edge-routing', () => {
      const order: string[] = [];
      const engine = new OrgLayoutEngine();
      engine.register({
        id: 'r-edge', phase: 'edge-routing', priority: 1, scope: 'global',
        precondition: () => true, apply() { order.push('edge-routing'); },
      } as LayoutRule);
      engine.register({
        id: 'r-internal', phase: 'internal-layout', priority: 1, scope: 'global',
        precondition: () => true, apply() { order.push('internal-layout'); },
      } as LayoutRule);
      engine.register({
        id: 'r-container', phase: 'container-positioning', priority: 1, scope: 'global',
        precondition: () => true, apply() { order.push('container-positioning'); },
      } as LayoutRule);
      engine.postLayout([], [], engine.preLayout(EMPTY_MODEL).facts);
      expect(order).toEqual(['internal-layout', 'container-positioning', 'edge-routing']);
    });
  });

  describe('validateComposition', () => {
    it('detects missing provider', () => {
      const engine = new OrgLayoutEngine();
      const rule: LayoutRule = {
        id: 'needs-sized', phase: 'internal-layout', priority: 1, scope: 'global',
        requires: ['nodes-sized'], precondition: () => true, apply() {},
      };
      engine.register(rule);
      const report = engine.validateComposition(engine.preLayout(EMPTY_MODEL).facts);
      expect(report.valid).toBe(false);
      expect(report.errors[0]!.type).toBe('missing-provider');
    });

    it('passes when all providers satisfied', () => {
      const engine = new OrgLayoutEngine();
      engine.register(stubSizingRule);
      engine.register(stubInternalRule);
      const report = engine.validateComposition(engine.preLayout(EMPTY_MODEL).facts);
      expect(report.valid).toBe(true);
    });

    it('detects conflicting guarantees in same group', () => {
      const engine = new OrgLayoutEngine();
      const ruleA: LayoutRule = {
        id: 'a', phase: 'internal-layout', priority: 1, scope: 'global',
        guarantees: ['agents-positioned'], precondition: () => true, apply() {},
      };
      const ruleB: LayoutRule = {
        id: 'b', phase: 'internal-layout', priority: 1, scope: 'global',
        guarantees: ['agents-positioned'], precondition: () => true, apply() {},
      };
      engine.register(ruleA);
      engine.register(ruleB);
      const report = engine.validateComposition(engine.preLayout(EMPTY_MODEL).facts);
      expect(report.valid).toBe(false);
      expect(report.errors[0]!.type).toBe('conflicting-guarantees');
    });
  });

  describe('elkOptions', () => {
    it('maps strategy to ELK configuration', () => {
      const engine = new OrgLayoutEngine();
      const opts = engine.elkOptions('tree');
      expect(opts.algorithm).toBe('mrtree');
      expect(opts.direction).toBe('DOWN');
    });

    it('includes positive spacing for all strategies', () => {
      const engine = new OrgLayoutEngine();
      const strategies = ['star', 'tree', 'circular', 'layered', 'nested', 'hub-spoke', 'flow', 'radial', 'grid', 'force'] as const;
      for (const s of strategies) {
        expect(engine.elkOptions(s).spacing).toBeGreaterThan(0);
      }
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/graph-stencil-org test -- --run engine.test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement OrgLayoutEngine**

```typescript
// layout/engine.ts
import type { GraphModel } from '@casehubio/graph-core';
import type {
  Phase, FactBase, ClassificationRule, LayoutRule, HardConstraint,
  CompositionReport, CompositionError, LayoutNode, LayoutEdge,
  LayoutViolation, ArchetypeHint, ArchetypeName, OrgLayoutStrategy,
  OrgElkLayoutOptions, LayoutExplanation, RuleSelection,
} from './types.js';
import { createFactBase } from './fact-base.js';

const POST_LAYOUT_PHASES: Exclude<Phase, 'classification' | 'sizing'>[] = [
  'internal-layout', 'container-positioning', 'edge-routing',
];

export interface PreLayoutResult {
  facts: FactBase;
  strategy: OrgLayoutStrategy;
  archetype: ArchetypeHint;
  nodeSizes: ReadonlyMap<string, { width: number; height: number }>;
  report: CompositionReport;
}

export interface PostLayoutResult {
  violations: LayoutViolation[];
  explanation?: LayoutExplanation;
}

export class OrgLayoutEngine {
  private classifiers: ClassificationRule[] = [];
  private rules: LayoutRule[] = [];
  private constraints: HardConstraint[] = [];

  register(item: ClassificationRule | LayoutRule | HardConstraint): void {
    if ('classify' in item) this.classifiers.push(item);
    else if ('check' in item) this.constraints.push(item);
    else this.rules.push(item);
  }

  preLayout(model: GraphModel): PreLayoutResult {
    const facts = createFactBase();

    // Run classification rules
    for (const c of this.classifiers) c.classify(model, facts);

    // Run sizing rules
    const sizingRules = this.resolvePhase('sizing', facts);
    const emptyNodes: LayoutNode[] = [];
    const emptyEdges: LayoutEdge[] = [];
    for (const r of sizingRules) r.apply(emptyNodes, emptyEdges, facts);

    // Extract results from facts
    const strategy = (facts.get('graph', 'recommended-strategy') as OrgLayoutStrategy) ?? 'force';
    const archetypeName = (facts.get('graph', 'archetype') as ArchetypeName) ?? 'simple-structure';
    const confidence = (facts.get('graph', 'archetype-confidence') as 'high' | 'medium' | 'low') ?? 'low';
    const nodeSizes = (facts.get('graph', 'node-sizes') as ReadonlyMap<string, { width: number; height: number }>) ?? new Map();
    const report = this.validateComposition(facts);

    return {
      facts,
      strategy,
      archetype: { archetype: archetypeName, confidence, layout: strategy },
      nodeSizes,
      report,
    };
  }

  postLayout(nodes: LayoutNode[], edges: LayoutEdge[], facts: FactBase): PostLayoutResult {
    for (const phase of POST_LAYOUT_PHASES) {
      const selected = this.resolvePhase(phase, facts);
      for (const rule of selected) rule.apply(nodes, edges, facts);
    }
    const violations: LayoutViolation[] = [];
    for (const c of this.constraints) violations.push(...c.check(nodes, edges));
    return { violations };
  }

  validateComposition(facts: FactBase): CompositionReport {
    const errors: CompositionError[] = [];
    const warnings: string[] = [];
    const selectedMap = new Map<Phase, LayoutRule[]>();

    for (const phase of [...POST_LAYOUT_PHASES, 'sizing'] as Phase[]) {
      selectedMap.set(phase, this.resolvePhase(phase, facts));
    }

    // Collect all provided guarantees
    const allProvided = new Set<string>();
    for (const rules of selectedMap.values()) {
      for (const r of rules) {
        for (const g of r.guarantees ?? []) allProvided.add(g);
      }
    }

    // Check missing providers
    for (const rules of selectedMap.values()) {
      for (const r of rules) {
        for (const req of r.requires ?? []) {
          if (!allProvided.has(req)) {
            errors.push({
              type: 'missing-provider',
              message: `Rule "${r.id}" requires "${req}" but no rule provides it`,
              ruleIds: [r.id],
            });
          }
        }
      }
    }

    // Check conflicting guarantees (two non-grouped rules providing same guarantee)
    const guaranteeOwners = new Map<string, string[]>();
    for (const rules of selectedMap.values()) {
      for (const r of rules) {
        for (const g of r.guarantees ?? []) {
          const owners = guaranteeOwners.get(g) ?? [];
          owners.push(r.id);
          guaranteeOwners.set(g, owners);
        }
      }
    }
    for (const [g, owners] of guaranteeOwners) {
      if (owners.length > 1) {
        errors.push({
          type: 'conflicting-guarantees',
          message: `Multiple rules provide "${g}": ${owners.join(', ')}`,
          ruleIds: owners,
        });
      }
    }

    return { valid: errors.length === 0, errors, warnings, selectedRules: selectedMap };
  }

  elkOptions(strategy: OrgLayoutStrategy): OrgElkLayoutOptions {
    switch (strategy) {
      case 'star': return { algorithm: 'mrtree', direction: 'DOWN', spacing: 120 };
      case 'tree': return { algorithm: 'mrtree', direction: 'DOWN', spacing: 100 };
      case 'circular': return { algorithm: 'stress', spacing: 120, elkOptions: { 'elk.stress.desiredEdgeLength': '200' } };
      case 'layered': return { algorithm: 'layered', direction: 'DOWN', spacing: 100, elkOptions: { 'elk.layered.crossingMinimization.strategy': 'LAYER_SWEEP' } };
      case 'nested': return { algorithm: 'layered', direction: 'DOWN', spacing: 100, containerPadding: 40 };
      case 'hub-spoke': return { algorithm: 'stress', spacing: 120, elkOptions: { 'elk.stress.desiredEdgeLength': '180' } };
      case 'flow': return { algorithm: 'layered', direction: 'RIGHT', spacing: 80, elkOptions: { 'elk.layered.nodePlacement.strategy': 'LINEAR_SEGMENTS' } };
      case 'radial': return { algorithm: 'radial', spacing: 120 };
      case 'grid': return { algorithm: 'layered', direction: 'DOWN', spacing: 100 };
      case 'force': return { algorithm: 'force', spacing: 120, elkOptions: { 'elk.force.temperature': '0.001' } };
    }
  }

  private resolvePhase(phase: Phase, facts: FactBase): LayoutRule[] {
    const phaseRules = this.rules.filter(r => r.phase === phase);
    const groups = new Map<string, LayoutRule[]>();
    const ungrouped: LayoutRule[] = [];

    for (const r of phaseRules) {
      if (r.group) {
        const list = groups.get(r.group) ?? [];
        list.push(r);
        groups.set(r.group, list);
      } else {
        ungrouped.push(r);
      }
    }

    const selected: LayoutRule[] = [];

    // Resolve groups: pick highest-priority applicable rule
    for (const [, candidates] of groups) {
      const sorted = [...candidates].sort((a, b) => a.priority - b.priority);
      const winner = sorted.find(r => r.precondition(facts));
      if (winner) selected.push(winner);
    }

    // Add ungrouped rules that pass precondition
    for (const r of ungrouped) {
      if (r.precondition(facts)) selected.push(r);
    }

    return selected;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/graph-stencil-org test -- --run engine.test`
Expected: PASS — all tests

- [ ] **Step 5: Commit**

```
feat(graph-stencil-org): add OrgLayoutEngine with phase execution and composition validation

Refs casehubio/blocks-ui#157
```

---

## Batch 2: Classification rules — replace archetype-detection

### Task 3: Classification rules

**Files:**
- Create: `packages/graph-stencil-org/src/layout/classification-rules.ts`
- Test: `packages/graph-stencil-org/src/layout/classification-rules.test.ts`

**Interfaces:**
- Consumes: `ClassificationRule`, `FactBase` from `./types.js`; `GraphModel`, `GraphEdge`, `GraphNode` from `@casehubio/graph-core`
- Produces: `orgClassificationRules()` — returns array of all built-in classification rules

The classification rules are broken into two layers:

1. **Structural detectors** — assert raw structural facts about the graph (e.g., `has-escalation-chains`, `supervision-depth`, `is-linear-delegation-chain`). These are independent, order-free.
2. **Archetype selector** — reads structural facts and asserts `archetype`, `archetype-confidence`, `recommended-strategy`. Uses the same priority cascade as the current `detectArchetype()` function.

This produces the same output as `detectArchetype()` but through the engine framework. The structural facts are available to later layout rules for per-container decisions.

- [ ] **Step 1: Write failing tests for classification rules**

Tests verify that the classification rules produce the same archetype + strategy as the current `detectArchetype()` function for each archetype scenario. Use the same YAML fixtures from the existing `archetype-detection.test.ts`.

```typescript
// classification-rules.test.ts
import { describe, it, expect } from 'vitest';
import { readFileSync } from 'fs';
import { resolve } from 'path';
import { toOrgGraph } from '../adapter/org-adapter.js';
import { OrgLayoutEngine } from './engine.js';
import { orgClassificationRules } from './classification-rules.js';
import type { ArchetypeName, OrgLayoutStrategy } from './types.js';

function loadYaml(archetype: string): string {
  return readFileSync(
    resolve(__dirname, '../../../../eidos/examples/org-scenarios/src/test/resources/archetypes', `${archetype}.yaml`),
    'utf-8',
  );
}

function classifyYaml(yaml: string): { archetype: ArchetypeName; strategy: OrgLayoutStrategy; confidence: string } {
  const { model } = toOrgGraph(yaml);
  const engine = new OrgLayoutEngine();
  for (const rule of orgClassificationRules()) engine.register(rule);
  const result = engine.preLayout(model);
  return {
    archetype: result.archetype.archetype,
    strategy: result.strategy,
    confidence: result.archetype.confidence,
  };
}

describe('classification rules', () => {
  it('detects simple-structure archetype', () => {
    const r = classifyYaml(loadYaml('simple-structure'));
    expect(r.archetype).toBe('simple-structure');
    expect(r.strategy).toBe('star');
  });

  it('detects hierarchy archetype', () => {
    const r = classifyYaml(loadYaml('hierarchy'));
    expect(r.archetype).toBe('hierarchy');
    expect(r.strategy).toBe('tree');
  });

  it('detects professional-bureaucracy archetype', () => {
    const r = classifyYaml(loadYaml('professional-bureaucracy'));
    expect(r.archetype).toBe('professional-bureaucracy');
    expect(r.strategy).toBe('circular');
  });

  it('detects tiered-escalation archetype', () => {
    const r = classifyYaml(loadYaml('tiered-escalation'));
    expect(r.archetype).toBe('tiered-escalation');
    expect(r.strategy).toBe('layered');
  });

  it('detects divisional-holarchy archetype', () => {
    const r = classifyYaml(loadYaml('divisional-holarchy'));
    expect(r.archetype).toBe('divisional-holarchy');
    expect(r.strategy).toBe('nested');
  });

  it('detects federation archetype', () => {
    const r = classifyYaml(loadYaml('federation'));
    expect(r.archetype).toBe('federation');
    expect(r.strategy).toBe('hub-spoke');
  });

  it('detects pipeline archetype', () => {
    const r = classifyYaml(loadYaml('pipeline'));
    expect(r.archetype).toBe('pipeline');
    expect(r.strategy).toBe('flow');
  });

  it('detects coalition archetype', () => {
    const r = classifyYaml(loadYaml('coalition'));
    expect(r.archetype).toBe('coalition');
    expect(r.strategy).toBe('radial');
  });

  it('detects matrix archetype', () => {
    const r = classifyYaml(loadYaml('matrix'));
    expect(r.archetype).toBe('matrix');
    expect(r.strategy).toBe('grid');
  });

  it('detects market archetype', () => {
    const r = classifyYaml(loadYaml('market'));
    expect(r.archetype).toBe('market');
    expect(r.strategy).toBe('radial');
  });

  it('falls back to force for empty graph', () => {
    const r = classifyYaml('organization:\n  units: []\n  relationships: []');
    expect(r.strategy).toBe('force');
    expect(r.confidence).toBe('low');
  });

  describe('structural facts', () => {
    it('asserts escalation-count for tiered-escalation', () => {
      const yaml = loadYaml('tiered-escalation');
      const { model } = toOrgGraph(yaml);
      const engine = new OrgLayoutEngine();
      for (const rule of orgClassificationRules()) engine.register(rule);
      const result = engine.preLayout(model);
      expect(result.facts.has('graph', 'has-escalation-chains')).toBe(true);
      expect(result.facts.get('graph', 'escalation-count')).toBeGreaterThanOrEqual(2);
    });

    it('asserts supervision-depth for hierarchy', () => {
      const yaml = loadYaml('hierarchy');
      const { model } = toOrgGraph(yaml);
      const engine = new OrgLayoutEngine();
      for (const rule of orgClassificationRules()) engine.register(rule);
      const result = engine.preLayout(model);
      expect(result.facts.get('graph', 'supervision-depth')).toBeGreaterThan(2);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/graph-stencil-org test -- --run classification-rules.test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement classification rules**

Port the heuristic logic from `archetype-detection.ts` into `ClassificationRule` objects. Each structural detector asserts facts; the archetype selector reads facts and picks the winner.

The implementation has two parts:
1. **Structural detectors** (one per signal): `market-detector`, `multi-unit-detector`, `nested-unit-detector`, `escalation-detector`, `delegation-chain-detector`, `federation-detector`, `coalition-detector`, `supervision-tree-detector`, `simple-structure-detector`, `backup-only-detector`
2. **Archetype selector**: reads all structural facts, applies priority cascade, asserts `archetype`, `archetype-confidence`, `recommended-strategy`

```typescript
// classification-rules.ts — key structure (full implementation follows TDD)
import type { GraphModel, GraphEdge, GraphNode } from '@casehubio/graph-core';
import type { ClassificationRule, FactBase, ArchetypeName, OrgLayoutStrategy } from './types.js';

const ARCHETYPE_LAYOUT: Record<ArchetypeName, OrgLayoutStrategy> = {
  'simple-structure': 'star',
  'hierarchy': 'tree',
  'professional-bureaucracy': 'circular',
  'tiered-escalation': 'layered',
  'divisional-holarchy': 'nested',
  'federation': 'hub-spoke',
  'pipeline': 'flow',
  'coalition': 'radial',
  'matrix': 'grid',
  'market': 'radial',
};

// Each detector is a ClassificationRule that asserts structural facts.
// The archetype-selector reads facts and picks the winning archetype.

export function orgClassificationRules(): ClassificationRule[] {
  return [
    ...structuralDetectors(),
    archetypeSelector(),
  ];
}
```

The structural detectors reuse the same graph analysis functions currently in `archetype-detection.ts` (`edgesByKind`, `supervisesTreeDepth`, `isLinearDelegatesToChain`, `hasMultiUnitAgents`, `hasNestedUnits`) — but instead of returning a single result, they assert facts onto the FactBase.

The `archetypeSelector` reads facts in the same priority order as the current `detectArchetype` cascade and asserts the winning archetype + strategy + confidence.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/graph-stencil-org test -- --run classification-rules.test`
Expected: PASS — all 12 tests

- [ ] **Step 5: Commit**

```
feat(graph-stencil-org): add classification rules replacing archetype detection

Refs casehubio/blocks-ui#157
```

---

## Batch 3: Layout transform rules — replace scattered functions

### Task 4: Sizing, internal layout, and container stacking rules

**Files:**
- Create: `packages/graph-stencil-org/src/layout/layout-rules.ts`
- Test: `packages/graph-stencil-org/src/layout/layout-rules.test.ts`

**Interfaces:**
- Consumes: `LayoutRule`, `HardConstraint`, `LayoutNode`, `LayoutEdge`, `LayoutViolation`, `FactBase` from `./types.js`; `GraphModel` from `@casehubio/graph-core`; `DispositionAxes` from `../types.js`
- Produces: `orgLayoutRules()` — returns array of all built-in layout rules; `orgHardConstraints()` — returns array of all hard constraints; exported constants: `AGENT_WIDTH`, `DISPOSITION_SHORT_NAMES`, `INTERNAL_PAD`, `HEADER_HEIGHT`

This task ports four existing functions into rules:

| Current function | New rule | Phase | Group |
|---|---|---|---|
| `computeNodeSizes()` | `agent-sizing` | sizing | — |
| `applyHorizontalInternalLayout()` | `horizontal-internal` | internal-layout | `internal-layout` |
| `applyContainerStacking()` | `vertical-stacking` | container-positioning | `container-position` |
| `applyPositionAwareHandles()` | `position-aware-handles` | edge-routing | — |
| `checkNoContainerOverlap()` | `HR1:no-container-overlap` | — (HardConstraint) | — |
| `checkAgentContainment()` | `HR2:agent-containment` | — (HardConstraint) | — |
| `checkNoAgentOverlap()` | `HR3:no-agent-overlap` | — (HardConstraint) | — |

- [ ] **Step 1: Write failing tests**

```typescript
// layout-rules.test.ts
import { describe, it, expect } from 'vitest';
import { toOrgGraph } from '../adapter/org-adapter.js';
import { enrichWithDescriptors } from '../adapter/enrichment.js';
import { computeDerivedData } from '../adapter/derived-data.js';
import { resolveKindColors } from '../adapter/kind-colors.js';
import { OrgLayoutEngine } from './engine.js';
import { orgClassificationRules } from './classification-rules.js';
import { orgLayoutRules, orgHardConstraints, AGENT_WIDTH } from './layout-rules.js';
import type { LayoutNode, LayoutEdge } from './types.js';

// ─── Gastown fixture (same as current org-layout-rules.test.ts) ──────
const GASTOWN_YAML = `organization:
  units:
    - unitId: oversight
      name: Oversight Chain
      kind: supervision-hierarchy
      tenancyId: gastown
      members:
        - agentId: boot
          role: root-watchdog
        - agentId: deacon
          role: cross-rig-watchdog
      capabilities: []
      goals: []
      constraints: []
    - unitId: rig-alpha
      name: Rig Alpha
      kind: rig
      tenancyId: gastown
      members:
        - agentId: witness-alpha
          role: witness
        - agentId: polecat-1
          role: worker
        - agentId: polecat-2
          role: worker
      capabilities:
        - name: full-stack-code-work
    - unitId: rig-beta
      name: Rig Beta
      kind: rig
      tenancyId: gastown
      members:
        - agentId: witness-beta
          role: witness
        - agentId: polecat-3
          role: worker
      capabilities:
        - name: full-stack-code-work
  relationships:
    - sourceAgentId: boot
      targetAgentId: deacon
      kind: SUPERVISES
      tenancyId: gastown
    - sourceAgentId: deacon
      targetAgentId: witness-alpha
      kind: SUPERVISES
      tenancyId: gastown
    - sourceAgentId: deacon
      targetAgentId: witness-beta
      kind: SUPERVISES
      tenancyId: gastown
    - sourceAgentId: witness-alpha
      targetAgentId: polecat-1
      kind: SUPERVISES
      tenancyId: gastown
    - sourceAgentId: witness-alpha
      targetAgentId: polecat-2
      kind: SUPERVISES
      tenancyId: gastown
    - sourceAgentId: witness-beta
      targetAgentId: polecat-3
      kind: SUPERVISES
      tenancyId: gastown
    - sourceAgentId: polecat-1
      targetAgentId: witness-alpha
      kind: ESCALATES_TO
      tenancyId: gastown
    - sourceAgentId: polecat-2
      targetAgentId: witness-alpha
      kind: ESCALATES_TO
      tenancyId: gastown
    - sourceAgentId: polecat-3
      targetAgentId: witness-beta
      kind: ESCALATES_TO
      tenancyId: gastown
    - sourceAgentId: witness-alpha
      targetAgentId: deacon
      kind: ESCALATES_TO
      tenancyId: gastown
    - sourceAgentId: witness-beta
      targetAgentId: deacon
      kind: ESCALATES_TO
      tenancyId: gastown
    - sourceAgentId: deacon
      targetAgentId: boot
      kind: ESCALATES_TO
      tenancyId: gastown
    - sourceAgentId: polecat-1
      targetAgentId: polecat-2
      kind: BACKS_UP
      tenancyId: gastown
`;

const GASTOWN_AGENTS = {
  boot: { slot: 'root-watchdog', disposition: { autonomy: 'high' as const }, capabilities: [{ name: 'system-oversight' }] },
  deacon: { slot: 'cross-rig-watchdog', disposition: { ruleFollowing: 'principled' as const }, capabilities: [{ name: 'rig-monitoring' }] },
  'witness-alpha': { slot: 'witness', disposition: { socialOrient: 'collaborative' as const }, capabilities: [{ name: 'code-review' }] },
  'witness-beta': { slot: 'witness', disposition: { socialOrient: 'collaborative' as const }, capabilities: [{ name: 'code-review' }] },
  'polecat-1': { slot: 'worker', disposition: { autonomy: 'semi-auto' as const }, capabilities: [{ name: 'full-stack-code-work' }] },
  'polecat-2': { slot: 'worker', disposition: { autonomy: 'semi-auto' as const }, capabilities: [{ name: 'full-stack-code-work' }] },
  'polecat-3': { slot: 'worker', disposition: { autonomy: 'semi-auto' as const }, capabilities: [{ name: 'full-stack-code-work' }] },
};

function buildEngine(): OrgLayoutEngine {
  const engine = new OrgLayoutEngine();
  for (const r of orgClassificationRules()) engine.register(r);
  for (const r of orgLayoutRules()) engine.register(r);
  for (const c of orgHardConstraints()) engine.register(c);
  return engine;
}

function buildGastownModel() {
  const base = toOrgGraph(GASTOWN_YAML);
  const enriched = enrichWithDescriptors(base.model, GASTOWN_AGENTS);
  const derived = computeDerivedData(enriched);
  return resolveKindColors(derived.model);
}

function buildGastownNodes(model: ReturnType<typeof buildGastownModel>, sizes: ReadonlyMap<string, { width: number; height: number }>): LayoutNode[] {
  return model.nodes.map(n => {
    const size = sizes.get(n.id);
    return {
      id: n.id,
      type: n.type,
      parentId: n.parentId,
      position: { x: 0, y: 0 },
      width: size?.width ?? 280,
      height: size?.height ?? 50,
      style: n.type === 'org-unit' ? { width: 280, height: 180 } : undefined,
    };
  });
}

describe('sizing rule', () => {
  it('produces node sizes via engine preLayout', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const result = engine.preLayout(model);
    expect(result.nodeSizes.size).toBeGreaterThan(0);
    for (const [id, size] of result.nodeSizes) {
      expect(size.width).toBe(AGENT_WIDTH);
      expect(size.height).toBeGreaterThan(0);
    }
  });

  it('returns larger height for enriched agents', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const result = engine.preLayout(model);
    const boot = result.nodeSizes.get(model.nodes.find(n => n.id.includes('boot'))!.id);
    expect(boot).toBeDefined();
    expect(boot!.height).toBeGreaterThan(40);
  });
});

describe('Gastown composition via engine', () => {
  it('horizontal internal layout produces no agent overlap', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const pre = engine.preLayout(model);
    const nodes = buildGastownNodes(model, pre.nodeSizes);
    const result = engine.postLayout(nodes, [], pre.facts);
    expect(result.violations.filter(v => v.rule === 'HR3:no-agent-overlap')).toEqual([]);
  });

  it('agents remain within containers after full pipeline', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const pre = engine.preLayout(model);
    const nodes = buildGastownNodes(model, pre.nodeSizes);
    const result = engine.postLayout(nodes, [], pre.facts);
    expect(result.violations.filter(v => v.rule === 'HR2:agent-containment')).toEqual([]);
  });

  it('all hard rules pass after full engine pipeline', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const pre = engine.preLayout(model);
    const nodes = buildGastownNodes(model, pre.nodeSizes);
    const result = engine.postLayout(nodes, [], pre.facts);
    expect(result.violations).toEqual([]);
  });

  it('Oversight Chain has agents in horizontal row', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const pre = engine.preLayout(model);
    const nodes = buildGastownNodes(model, pre.nodeSizes);
    engine.postLayout(nodes, [], pre.facts);
    const boot = nodes.find(n => n.id.includes('boot'))!;
    const deacon = nodes.find(n => n.id.includes('deacon'))!;
    expect(boot.position.y).toBe(deacon.position.y);
    expect(boot.position.x).not.toBe(deacon.position.x);
  });

  it('containers resize to fit children', () => {
    const model = buildGastownModel();
    const engine = buildEngine();
    const pre = engine.preLayout(model);
    const nodes = buildGastownNodes(model, pre.nodeSizes);
    engine.postLayout(nodes, [], pre.facts);
    for (const container of nodes.filter(n => n.type === 'org-unit')) {
      const children = nodes.filter(n => n.parentId === container.id);
      if (children.length < 2) continue;
      const cw = container.width ?? (container.style?.width as number) ?? 0;
      const ch = container.height ?? (container.style?.height as number) ?? 0;
      for (const child of children) {
        expect(child.position.x + (child.width ?? 280)).toBeLessThanOrEqual(cw as number);
        expect(child.position.y + (child.height ?? 50)).toBeLessThanOrEqual(ch as number);
      }
    }
  });
});

describe('edge routing rule', () => {
  it('assigns forward handles for SUPERVISES edges', () => {
    const engine = buildEngine();
    const facts = engine.preLayout({ nodes: [], edges: [] }).facts;
    const nodes: LayoutNode[] = [
      { id: 'u1', type: 'org-unit', position: { x: 0, y: 0 }, width: 800, height: 400 },
      { id: 'a1', type: 'org-agent', parentId: 'u1', position: { x: 10, y: 68 }, width: 280, height: 50 },
      { id: 'a2', type: 'org-agent', parentId: 'u1', position: { x: 320, y: 68 }, width: 280, height: 50 },
    ];
    const edges: LayoutEdge[] = [
      { id: 'e1', type: 'org-supervises', source: 'a1', target: 'a2' },
    ];
    engine.postLayout(nodes, edges, facts);
    expect(edges[0]!.sourceHandle).toBeDefined();
    expect(edges[0]!.targetHandle).toBeDefined();
  });
});

describe('hard constraint rules', () => {
  it('detects overlapping containers', () => {
    const engine = buildEngine();
    const facts = engine.preLayout({ nodes: [], edges: [] }).facts;
    const nodes: LayoutNode[] = [
      { id: 'u1', type: 'org-unit', position: { x: 0, y: 0 }, width: 150, height: 100 },
      { id: 'u2', type: 'org-unit', position: { x: 100, y: 0 }, width: 150, height: 100 },
    ];
    const result = engine.postLayout(nodes, [], facts);
    expect(result.violations.some(v => v.rule === 'HR1:no-container-overlap')).toBe(true);
  });

  it('detects agent exceeding container', () => {
    const engine = buildEngine();
    const facts = engine.preLayout({ nodes: [], edges: [] }).facts;
    const nodes: LayoutNode[] = [
      { id: 'u1', type: 'org-unit', position: { x: 0, y: 0 }, width: 200, height: 200 },
      { id: 'a1', type: 'org-agent', parentId: 'u1', position: { x: 10, y: 68 }, width: 280, height: 100 },
    ];
    const result = engine.postLayout(nodes, [], facts);
    expect(result.violations.some(v => v.rule === 'HR2:agent-containment')).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/graph-stencil-org test -- --run layout-rules.test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement layout rules**

Port the function bodies from `org-layout-rules.ts`, `adapter/node-sizing.ts`, and `adapter/edge-labels.ts` (`applyPositionAwareHandles`) into rule objects. Each rule has:
- `precondition`: returns `true` (these are the default rules; alternative rules added later have selective preconditions)
- `apply`: contains the same mutation logic as the original function
- `guarantees`: declares what the rule produces

The sizing rule is special — it stores its result in the FactBase via `facts.assert('graph', 'node-sizes', sizes)` because it runs during `preLayout` (before nodes exist).

```typescript
// layout-rules.ts — structure
import type { GraphModel } from '@casehubio/graph-core';
import type { LayoutRule, HardConstraint, LayoutNode, LayoutEdge, LayoutViolation, FactBase } from './types.js';

export const AGENT_WIDTH = 280;
export const DISPOSITION_SHORT_NAMES = { /* same as current */ };
export const INTERNAL_PAD = 30;
export const HEADER_HEIGHT = 68;

function agentSizingRule(): LayoutRule { /* computeNodeSizes logic, stores in facts */ }
function horizontalInternalRule(): LayoutRule { /* applyHorizontalInternalLayout logic */ }
function verticalStackingRule(): LayoutRule { /* applyContainerStacking logic */ }
function positionAwareHandlesRule(): LayoutRule { /* applyPositionAwareHandles logic */ }

function noContainerOverlapConstraint(): HardConstraint { /* checkNoContainerOverlap logic */ }
function agentContainmentConstraint(): HardConstraint { /* checkAgentContainment logic */ }
function noAgentOverlapConstraint(): HardConstraint { /* checkNoAgentOverlap logic */ }

export function orgLayoutRules(): LayoutRule[] {
  return [agentSizingRule(), horizontalInternalRule(), verticalStackingRule(), positionAwareHandlesRule()];
}

export function orgHardConstraints(): HardConstraint[] {
  return [noContainerOverlapConstraint(), agentContainmentConstraint(), noAgentOverlapConstraint()];
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/graph-stencil-org test -- --run layout-rules.test`
Expected: PASS — all tests

- [ ] **Step 5: Commit**

```
feat(graph-stencil-org): add layout transform and hard constraint rules

Refs casehubio/blocks-ui#157
```

---

## Batch 4: Integration — wire engine, delete old files, update exports

### Task 5: Wire engine into org-diagram component

**Files:**
- Modify: `components/org-diagram/src/blocks-org-diagram.ts`
- Modify: `packages/graph-stencil-org/src/index.ts`
- Delete: `packages/graph-stencil-org/src/layout/org-layout-rules.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/layout/org-layout-rules.test.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/layout/archetype-detection.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/layout/archetype-detection.test.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/layout/layout-strategy.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/layout/layout-strategy.test.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/adapter/node-sizing.ts` (use `ide_refactor_safe_delete`)
- Delete: `packages/graph-stencil-org/src/adapter/node-sizing.test.ts` (use `ide_refactor_safe_delete`)
- Modify: `packages/graph-stencil-org/src/adapter/edge-labels.ts` — remove `applyPositionAwareHandles` and its types (`NodePos`, `FORWARD_TYPES`, `REVERSE_TYPES`, `PEER_TYPES`)

**Interfaces:**
- Consumes: `OrgLayoutEngine`, `PreLayoutResult`, `PostLayoutResult` from engine; `orgClassificationRules` from classification-rules; `orgLayoutRules`, `orgHardConstraints` from layout-rules; `ArchetypeHint`, `OrgLayoutStrategy`, `OrgElkLayoutOptions`, `LayoutNode`, `LayoutEdge` from types
- Produces: Updated `blocks-org-diagram.ts` using engine for all layout decisions

- [ ] **Step 1: Update index.ts exports**

Replace old exports with new engine exports. The public API changes:

**Removed exports:**
- `detectArchetype` (replaced by engine + classification rules)
- `orgLayoutOptions` (replaced by `engine.elkOptions()`)
- `computeNodeSizes`, `AGENT_WIDTH`, `DISPOSITION_SHORT_NAMES` (replaced by engine sizing rule; re-export constants from layout-rules.ts)
- `applyHorizontalInternalLayout`, `applyContainerStacking`, `verifyHardRules` (replaced by engine postLayout)
- `applyPositionAwareHandles` (replaced by engine edge-routing rule)

**New exports:**
- `OrgLayoutEngine` from engine
- `orgClassificationRules` from classification-rules
- `orgLayoutRules`, `orgHardConstraints` from layout-rules
- `createFactBase` from fact-base
- All types from types.ts (`Phase`, `Fact`, `FactBase`, `ClassificationRule`, `LayoutRule`, `HardConstraint`, `CompositionReport`, `LayoutExplanation`, `LayoutNode`, `LayoutEdge`, `LayoutViolation`)

**Preserved exports (unchanged):**
- `ArchetypeHint`, `ArchetypeName`, `OrgLayoutStrategy` (from types.ts now, not archetype-detection)
- `OrgElkLayoutOptions`, `ElkAlgorithm` (from types.ts now, not layout-strategy)
- `AGENT_WIDTH`, `DISPOSITION_SHORT_NAMES` (from layout-rules.ts now, not node-sizing)
- All adapter/stencil/schema/editing exports stay unchanged

```typescript
// index.ts — new engine exports (add these, remove old ones)
export { OrgLayoutEngine } from './layout/engine.js';
export type { PreLayoutResult, PostLayoutResult } from './layout/engine.js';
export { orgClassificationRules } from './layout/classification-rules.js';
export { orgLayoutRules, orgHardConstraints, AGENT_WIDTH, DISPOSITION_SHORT_NAMES } from './layout/layout-rules.js';
export { createFactBase } from './layout/fact-base.js';
export type {
  Phase, Fact, FactBase, ClassificationRule, LayoutRule, HardConstraint,
  CompositionReport, LayoutExplanation, RuleSelection,
  LayoutNode, LayoutEdge, LayoutViolation,
  ArchetypeHint, ArchetypeName, OrgLayoutStrategy,
  OrgElkLayoutOptions, ElkAlgorithm,
} from './layout/types.js';
export { computeRadialLayout } from './layout/radial-layout.js';
```

- [ ] **Step 2: Rewrite component pipeline to use engine**

Replace the scattered import list with engine imports. The `_adaptYaml` and `_fullRender` methods change:

**Before (current):**
```typescript
// _adaptYaml: toOrgGraph → enrichWithDescriptors → computeDerivedData → resolveKindColors → applyCollapsedUnits → computeNodeSizes → detectArchetype
// _fullRender: _adaptYaml → auto-strategy selection → ELK/radial → toReactFlowGraph → applyHorizontalInternalLayout → applyContainerStacking
// _updateEdgeStyles: applyOrgEdgeLabels → applyPositionAwareHandles → applySelectionHighlight
```

**After (engine):**
```typescript
// _adaptYaml: toOrgGraph → enrichWithDescriptors → computeDerivedData → resolveKindColors → applyCollapsedUnits → engine.preLayout (classifies + sizes + picks strategy)
// _fullRender: _adaptYaml → engine strategy → ELK/radial → toReactFlowGraph → engine.postLayout (internal layout + container stacking + edge handles + hard rules)
// _updateEdgeStyles: applyOrgEdgeLabels → applySelectionHighlight (no more applyPositionAwareHandles — engine handles it)
```

Key changes to `blocks-org-diagram.ts`:
1. Add `private _engine: OrgLayoutEngine` field, initialized in constructor
2. In `_adaptYaml`: replace `computeNodeSizes()` + `detectArchetype()` with `engine.preLayout(model)`. Store `PreLayoutResult` for use in `_fullRender`.
3. In `_fullRender`: replace `applyHorizontalInternalLayout` + `applyContainerStacking` with `engine.postLayout(nodes, edges, facts)`. The engine's edge-routing rule replaces `applyPositionAwareHandles`.
4. In `_updateEdgeStyles`: remove `applyPositionAwareHandles` call (handled by postLayout). Keep `applyOrgEdgeLabels` and `applySelectionHighlight`.
5. `_buildElkOpts` uses `engine.elkOptions(strategy)` instead of `orgLayoutOptions`.
6. `_layoutOptions` uses `engine.elkOptions(strategy)`.

- [ ] **Step 3: Remove applyPositionAwareHandles from edge-labels.ts**

Remove the `applyPositionAwareHandles` function and its private types (`NodePos`, `FORWARD_TYPES`, `REVERSE_TYPES`, `PEER_TYPES`) from `adapter/edge-labels.ts`. Keep `applyOrgEdgeLabels` and its types (`ScopeData`, `RfEdge`, `LABEL_STYLES`).

Update `adapter/edge-labels.test.ts` to remove any tests for `applyPositionAwareHandles` (those tests now live in `layout-rules.test.ts`).

- [ ] **Step 4: Delete old files**

Use `ide_refactor_safe_delete` for each file to ensure no remaining references:

1. `packages/graph-stencil-org/src/layout/org-layout-rules.ts`
2. `packages/graph-stencil-org/src/layout/org-layout-rules.test.ts`
3. `packages/graph-stencil-org/src/layout/archetype-detection.ts`
4. `packages/graph-stencil-org/src/layout/archetype-detection.test.ts`
5. `packages/graph-stencil-org/src/layout/layout-strategy.ts`
6. `packages/graph-stencil-org/src/layout/layout-strategy.test.ts`
7. `packages/graph-stencil-org/src/adapter/node-sizing.ts`
8. `packages/graph-stencil-org/src/adapter/node-sizing.test.ts`

- [ ] **Step 5: Run full test suite + typecheck**

Run: `yarn test && yarn typecheck`
Expected: PASS — all tests pass, no type errors

If failures: fix import paths, type mismatches, or missing exports. Common issues:
- Stale imports in test files referencing deleted modules
- The `blocks-org-diagram.test.ts` may import from deleted modules — update to use engine
- Schema test (`blocks-ui-schema`) may reference `OrgDiagramProps` — should be unaffected since the interface shape hasn't changed

- [ ] **Step 6: Commit**

```
refactor(graph-stencil-org): wire layout engine, delete old scattered layout files

Replaces archetype-detection, layout-strategy, org-layout-rules,
node-sizing, and applyPositionAwareHandles with the unified
OrgLayoutEngine pipeline.

Refs casehubio/blocks-ui#157
```

---

## Batch 5: Explain mode and archetype composition tests

### Task 6: Explain mode

**Files:**
- Modify: `packages/graph-stencil-org/src/layout/engine.ts`
- Modify: `packages/graph-stencil-org/src/layout/engine.test.ts`

**Interfaces:**
- Consumes: `LayoutExplanation`, `RuleSelection` from `./types.js`
- Produces: `postLayout(nodes, edges, facts, { explain: true })` returns `explanation` field in result

- [ ] **Step 1: Write failing tests for explain mode**

```typescript
// Add to engine.test.ts
describe('explain mode', () => {
  it('returns classification facts in explanation', () => {
    const engine = new OrgLayoutEngine();
    engine.register(stubClassifier);
    engine.register(stubInternalRule);
    const pre = engine.preLayout(EMPTY_MODEL);
    const nodes: LayoutNode[] = [{ id: 'a1', type: 'org-agent', parentId: 'u1', position: { x: 0, y: 0 } }];
    const result = engine.postLayout(nodes, [], pre.facts, { explain: true });
    expect(result.explanation).toBeDefined();
    expect(result.explanation!.classifications.length).toBeGreaterThan(0);
  });

  it('returns rule selections with group and reason', () => {
    const engine = new OrgLayoutEngine();
    engine.register(stubInternalRule);
    engine.register(stubInternalRuleB);
    const pre = engine.preLayout(EMPTY_MODEL);
    const nodes: LayoutNode[] = [{ id: 'a1', position: { x: 0, y: 0 } }];
    const result = engine.postLayout(nodes, [], pre.facts, { explain: true });
    const sel = result.explanation!.ruleSelections.find(s => s.group === 'internal-layout');
    expect(sel).toBeDefined();
    expect(sel!.selected).toBe('test-internal-a');
    expect(sel!.candidates).toHaveLength(2);
  });

  it('omits explanation when explain is false', () => {
    const engine = new OrgLayoutEngine();
    const result = engine.postLayout([], [], engine.preLayout(EMPTY_MODEL).facts);
    expect(result.explanation).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd packages/graph-stencil-org test -- --run engine.test`
Expected: FAIL — `postLayout` doesn't accept options parameter yet

- [ ] **Step 3: Add explain mode to engine**

Extend `postLayout` signature:
```typescript
postLayout(
  nodes: LayoutNode[],
  edges: LayoutEdge[],
  facts: FactBase,
  options?: { explain?: boolean },
): PostLayoutResult
```

When `explain: true`:
1. Capture all facts from the FactBase as `classifications`
2. During group resolution, record each group's candidates, which were applicable, which was selected, and why
3. Include violations
4. Return as `LayoutExplanation`

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn --cwd packages/graph-stencil-org test -- --run engine.test`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(graph-stencil-org): add explain mode to layout engine

Refs casehubio/blocks-ui#157
```

---

### Task 7: Archetype composition tests

**Files:**
- Modify: `packages/graph-stencil-org/src/layout/layout-rules.test.ts`

**Interfaces:**
- Consumes: full engine pipeline (classification + layout rules + hard constraints)
- Produces: composition tests verifying no hard rule violations for multiple archetype structures

- [ ] **Step 1: Write composition tests for additional archetypes**

```typescript
// Add to layout-rules.test.ts
import { readFileSync } from 'fs';
import { resolve } from 'path';

function loadArchetypeYaml(name: string): string {
  return readFileSync(
    resolve(__dirname, '../../../../eidos/examples/org-scenarios/src/test/resources/archetypes', `${name}.yaml`),
    'utf-8',
  );
}

function runFullPipeline(yaml: string) {
  const { model } = toOrgGraph(yaml);
  const engine = buildEngine();
  const pre = engine.preLayout(model);
  const nodes: LayoutNode[] = model.nodes.map(n => {
    const size = pre.nodeSizes.get(n.id);
    return {
      id: n.id, type: n.type, parentId: n.parentId,
      position: { x: 0, y: 0 },
      width: size?.width ?? 280, height: size?.height ?? 50,
      style: n.type === 'org-unit' ? { width: 280, height: 180 } : undefined,
    };
  });
  return engine.postLayout(nodes, [], pre.facts);
}

describe('archetype composition — no hard rule violations', () => {
  const archetypes = [
    'simple-structure', 'hierarchy', 'federation', 'pipeline',
    'coalition', 'tiered-escalation', 'professional-bureaucracy',
  ];

  for (const name of archetypes) {
    it(`${name}: passes all hard rules after engine pipeline`, () => {
      const result = runFullPipeline(loadArchetypeYaml(name));
      expect(result.violations).toEqual([]);
    });
  }
});
```

- [ ] **Step 2: Run tests to verify they pass**

Run: `yarn --cwd packages/graph-stencil-org test -- --run layout-rules.test`
Expected: PASS

If any archetype fails hard rules, investigate and fix the layout rule. The horizontal internal layout or container stacking may need adjustments for non-Gastown structures (e.g., federation with hub-spoke, pipeline with linear chain).

- [ ] **Step 3: Run full test suite**

Run: `yarn test && yarn typecheck`
Expected: PASS — all tests, no type errors

- [ ] **Step 4: Regression check — diagram examples**

Start the examples dev server and verify these diagrams render correctly:

Run: `yarn --cwd examples dev`

Check (in priority order):
1. **CaseHub diagram** (`#diagrams/casehub-diagram`) — should be unaffected (uses case stencils, not org layout)
2. **SWF diagram** (`#diagrams/swf-diagram`) — should be unaffected (uses SWF stencils, not org layout)
3. **Diagram workbench** (`#diagrams/diagram-workbench`) — should be unaffected
4. **Org diagram** (`#diagrams/org-diagram`) — verify all 4 archetypes (Gastown, Simple, Federation, Pipeline) render with correct layouts, agent cards, panels, legend

CaseHub and SWF diagrams should be completely unaffected since the refactoring only touches org-specific layout code. The org diagram is the one that exercises the engine.

- [ ] **Step 5: Commit**

```
test(graph-stencil-org): add archetype composition tests for layout engine

Refs casehubio/blocks-ui#157
```

---

## References

- [2026-09-10-layout-rule-engine-design.md] — design spec this plan implements
- [packages/graph-stencil-org/src/layout/org-layout-rules.ts] — current hard rules + layout functions (to be replaced)
- [packages/graph-stencil-org/src/layout/archetype-detection.ts] — current archetype classifier (to be replaced)
- [packages/graph-stencil-org/src/layout/layout-strategy.ts] — current strategy-to-ELK mapping (to be replaced)
- [packages/graph-stencil-org/src/adapter/node-sizing.ts] — current node sizing (to be replaced)
- [packages/graph-stencil-org/src/adapter/edge-labels.ts:63-97] — applyPositionAwareHandles (to move to edge routing rule)
- [components/org-diagram/src/blocks-org-diagram.ts:289-351] — current layout pipeline in _fullRender (to be rewired)
- [GitHub #157] — focal issue
