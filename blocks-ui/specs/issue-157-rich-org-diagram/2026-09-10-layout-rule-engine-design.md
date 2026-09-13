# Org Diagram Layout Rule Engine — Design Spec

**Date:** 2026-09-10
**Issue:** casehubio/blocks-ui#157
**Status:** Draft — brainstormed during visual QA session

## Motivation

The org diagram layout evolved through visual iteration — fixing one issue, breaking another. The layout logic is scattered across 5+ files with implicit dependencies. Rules contradict each other because there's no coordination mechanism.

A forward-chaining rule engine with static validation gives deterministic fidelity: same graph structure -> same facts -> same rules fire -> same layout. No optimizer randomness, no trial-and-error.

Similar in concept to OptaPlanner's constraint satisfaction, but standalone — no external solver dependency. Domain-specific to org diagram layout.

## Architecture

### Phases (executed in order)

1. **Classification** — analyze graph structure, produce facts
2. **Sizing** — compute node dimensions
3. **Internal Layout** — position agents within containers
4. **Container Positioning** — position containers relative to each other
5. **Edge Routing** — select handle positions for edges

Each phase's outputs are available to subsequent phases.

### Core Concepts

#### Rules

```typescript
interface LayoutRule {
  id: string;
  phase: Phase;
  group?: string;              // mutual exclusion group
  priority: number;            // lower = higher priority within group
  scope: 'global' | 'container' | 'edge';
  
  precondition(facts: FactBase, scope: ScopeContext): boolean;
  apply(nodes: LayoutNode[], edges: LayoutEdge[], facts: FactBase): void;
  
  requires?: string[];         // guarantee IDs this rule depends on
  guarantees?: string[];       // guarantee IDs this rule provides
  
  fallback?: string;           // rule ID to fall back to on failure
}
```

#### Classification Rules

Run first. Analyze structure, produce facts. Don't modify layout.

```typescript
interface ClassificationRule {
  id: string;
  classify(model: GraphModel): Fact[];
}

interface Fact {
  subject: string;             // node/edge/graph ID
  predicate: string;           // e.g. 'is-linear-chain', 'is-leaf-agent'
  value?: unknown;
}
```

Examples:
- `linear-chain-detector`: scans units for linear supervision chains -> `('unit:oversight', 'is-linear-chain', true)`
- `leaf-agent-detector`: agents with no outgoing SUPERVISES -> `('agent:polecat-1', 'is-leaf', true)`
- `cross-hierarchy-detector`: edges spanning containers -> `('e:deacon-witness', 'crosses-container', true)`
- `fan-out-detector`: agents supervising into 2+ containers -> `('agent:deacon', 'fans-out-to', ['unit:rig-alpha', 'unit:rig-beta'])`

#### Fact Base

Accumulated facts from classification rules. Queryable by preconditions.

```typescript
interface FactBase {
  has(subject: string, predicate: string): boolean;
  get(subject: string, predicate: string): unknown;
  query(predicate: string): Array<{ subject: string; value: unknown }>;
}
```

#### Mutual Exclusion Groups

Only one rule from a group fires. Highest-priority applicable rule wins.

Examples:
- Group `internal-layout`: `horizontal-flow` (priority 1) vs `hierarchical-tree` (priority 2)
- Group `container-position`: `vertical-stack` (priority 1) vs `grid` (priority 2)
- Group `edge-routing:reverse`: `same-side-top` vs `same-side-left` — precondition selects based on relative position

#### Preconditions

Query the fact base, not raw graph data:

```typescript
// Instead of:
if (model.edges.filter(e => e.kind === 'ESCALATES_TO').length >= 2) { ... }

// Use:
precondition: (facts) => facts.has('graph', 'has-escalation-chains')
```

#### Guarantees and Dependencies

Rules declare what they provide and require:

- `applyHorizontalInternalLayout` **guarantees**: `agents-positioned`, `containers-sized`
- `applyContainerStacking` **requires**: `containers-sized`
- `checkAgentContainment` **requires**: `agents-positioned`, `containers-sized`

### Hard vs Soft Rules

| Type | Behavior | Example |
|------|----------|---------|
| Hard constraint | Must be satisfied. Pipeline fails if violated. | No container overlap, agent containment |
| Soft constraint | Scored. Lower cost = better. | Edge crossing count, total edge length |

Hard constraints are always-on verification rules. They don't have groups or preconditions — they always run after all transforms.

Soft constraints return a numeric cost. The engine can optionally compare alternative rule selections within a bounded search space (not exhaustive — pick from 2-3 candidates, not 10).

### Pre-flight Composition Validation

Before any layout mutation, validate the selected rule combination:

```typescript
interface CompositionReport {
  valid: boolean;
  errors: CompositionError[];    // blocks execution
  warnings: CompositionWarning[]; // advisory
  selectedRules: Map<Phase, LayoutRule[]>;
}
```

Checks:
- **Missing providers**: rule A requires guarantee G, no enabled rule provides G
- **Circular dependencies**: A requires B's guarantee, B requires A's
- **Conflicting guarantees**: two selected rules both guarantee the same property
- **Unsatisfiable constraints**: hard rule requires X, selected layout rules can't produce X
- **Tension detection**: two rules from different groups that are individually valid but contradictory when combined

### User Overrides / Pins

Users can pin a rule for a specific scope:

```yaml
organization:
  units:
    - unitId: oversight
      name: Oversight Chain
      layout: horizontal    # pin: forces horizontal-flow rule
```

Pins bypass precondition selection. Stored in YAML as node properties.

### Explain Mode

For debugging:

```typescript
interface LayoutExplanation {
  classifications: Fact[];
  ruleSelections: Array<{
    group: string;
    candidates: Array<{ rule: string; applicable: boolean; priority: number }>;
    selected: string;
    reason: string;
  }>;
  violations: LayoutViolation[];
}
```

## Current State

The `org-layout-rules.ts` module has:
- Hard rule checks: `checkNoContainerOverlap`, `checkAgentContainment`, `checkNoAgentOverlap`
- Layout functions: `applyHorizontalInternalLayout`, `applyContainerStacking`
- Composition tests using Gastown structure

These will be refactored into the rule engine framework.

## Implementation Plan

1. Define `LayoutRule`, `ClassificationRule`, `FactBase`, `CompositionReport` types
2. Implement `LayoutEngine` with phase execution, group resolution, pre-flight validation
3. Convert existing layout functions into rules with preconditions and guarantees
4. Add classification rules for Gastown patterns
5. Add composition tests for federation, pipeline, coalition archetypes
6. Add explain mode
7. Wire engine into org-diagram component (replacing direct function calls)
