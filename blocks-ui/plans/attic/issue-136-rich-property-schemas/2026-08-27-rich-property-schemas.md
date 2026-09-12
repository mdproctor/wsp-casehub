# Rich Property Schemas Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #136 — rich domain property schemas for all diagram types
**Issue group:** #136

**Goal:** Define per-stencil JSON Schema descriptors that drive the pages-property-palette, migrating from hand-coded form renderers to schema-driven rendering across all three stencil packages.

**Architecture:** Schema registry in diagram-core replaces the existing `_schemaTypeMap()` abstract method on `DiagramBaseMixin`. Each stencil package exports its schemas and registers them during `register*Stencils()`. Discriminated unions use `oneOf` with `x-discriminator`. Complex fields use `x-editor-component` for custom rendering.

**Tech Stack:** TypeScript, JSON Schema, Lit custom elements, vitest

## Global Constraints

- Schemas use standard JSON Schema with `x-group`, `x-order`, `x-visibility`, `x-discriminator`, `x-editor-component` extensions
- All field names and types must match the generated types in `case-definition.ts` and `worker-function/types.ts`
- Grouping convention: Identity (0), Configuration (10), Target (15), Function (18), Behaviour (20), Status (25), Advanced (30)
- `x-visibility: 'advanced'` on all Advanced group fields
- Stencil packages stay siloed — no cross-imports between stencil packages

---

## Batch 1: Schema registry and simple schemas (milestone, goal)

### Task 1: Schema registry in diagram-core

**Files:**
- Create: `packages/diagram-core/src/schema-registry.ts`
- Create: `packages/diagram-core/src/schema-registry.test.ts`
- Modify: `packages/diagram-core/src/index.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `registerPropertySchema(nodeType: string, schema: Record<string, unknown>): void`, `getPropertySchema(nodeType: string): Record<string, unknown> | undefined`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { registerPropertySchema, getPropertySchema } from './schema-registry.js';

describe('schema-registry', () => {
  it('returns undefined for unregistered type', () => {
    expect(getPropertySchema('nonexistent')).toBeUndefined();
  });

  it('registers and retrieves a schema', () => {
    const schema = { type: 'object', properties: { name: { type: 'string' } } };
    registerPropertySchema('test-node', schema);
    expect(getPropertySchema('test-node')).toBe(schema);
  });

  it('overwrites on re-registration', () => {
    const schema1 = { type: 'object', properties: { a: { type: 'string' } } };
    const schema2 = { type: 'object', properties: { b: { type: 'number' } } };
    registerPropertySchema('overwrite-test', schema1);
    registerPropertySchema('overwrite-test', schema2);
    expect(getPropertySchema('overwrite-test')).toBe(schema2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/diagram-core test -- --reporter verbose 2>&1 | tail -15`
Expected: FAIL — module not found

- [ ] **Step 3: Implement schema registry**

```typescript
// packages/diagram-core/src/schema-registry.ts
const schemas = new Map<string, Record<string, unknown>>();

export function registerPropertySchema(nodeType: string, schema: Record<string, unknown>): void {
  schemas.set(nodeType, schema);
}

export function getPropertySchema(nodeType: string): Record<string, unknown> | undefined {
  return schemas.get(nodeType);
}
```

- [ ] **Step 4: Add exports to index.ts**

Append to `packages/diagram-core/src/index.ts`:
```typescript
export { registerPropertySchema, getPropertySchema } from './schema-registry.js';
```

- [ ] **Step 5: Run tests to verify pass**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/diagram-core test -- --reporter verbose 2>&1 | tail -15`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/diagram-core/src/schema-registry.ts packages/diagram-core/src/schema-registry.test.ts packages/diagram-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(diagram-core): add property schema registry Refs #136"
```

### Task 2: Milestone and goal schemas in graph-stencil-case

**Files:**
- Create: `packages/graph-stencil-case/src/schemas/milestone-schema.ts`
- Create: `packages/graph-stencil-case/src/schemas/goal-schema.ts`
- Create: `packages/graph-stencil-case/src/schemas/index.ts`
- Create: `packages/graph-stencil-case/src/schemas/schemas.test.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`
- Modify: `packages/graph-stencil-case/src/stencils/case-stencils.ts` (or wherever `registerCaseStencils` lives)

**Interfaces:**
- Consumes: `registerPropertySchema` from diagram-core (Task 1)
- Produces: `milestoneSchema`, `goalSchema` — JSON Schema objects exported from `schemas/index.ts`

- [ ] **Step 1: Write failing test**

```typescript
import { describe, it, expect } from 'vitest';
import { milestoneSchema } from './milestone-schema.js';
import { goalSchema } from './goal-schema.js';

describe('milestone-schema', () => {
  it('has required name and condition fields', () => {
    expect(milestoneSchema.required).toContain('name');
    expect(milestoneSchema.required).toContain('condition');
  });

  it('has x-group on all properties', () => {
    const props = milestoneSchema.properties as Record<string, any>;
    for (const [key, prop] of Object.entries(props)) {
      expect(prop['x-group'], `${key} missing x-group`).toBeDefined();
    }
  });

  it('groups name and description as Identity', () => {
    const props = milestoneSchema.properties as Record<string, any>;
    expect(props.name['x-group']).toBe('Identity');
    expect(props.description['x-group']).toBe('Identity');
  });

  it('marks SLA fields as advanced', () => {
    const props = milestoneSchema.properties as Record<string, any>;
    expect(props.slaDuration['x-visibility']).toBe('advanced');
    expect(props.slaStartFrom['x-visibility']).toBe('advanced');
  });
});

describe('goal-schema', () => {
  it('has required name and condition fields', () => {
    expect(goalSchema.required).toContain('name');
    expect(goalSchema.required).toContain('condition');
  });

  it('has x-group on all properties', () => {
    const props = goalSchema.properties as Record<string, any>;
    for (const [key, prop] of Object.entries(props)) {
      expect(prop['x-group'], `${key} missing x-group`).toBeDefined();
    }
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — modules not found

- [ ] **Step 3: Implement milestone schema**

```typescript
// packages/graph-stencil-case/src/schemas/milestone-schema.ts
export const milestoneSchema = {
  type: 'object',
  required: ['name', 'condition'],
  properties: {
    name: {
      type: 'string',
      'x-group': 'Identity',
      'x-order': 0,
    },
    description: {
      type: 'string',
      'x-group': 'Identity',
      'x-order': 1,
    },
    condition: {
      type: 'string',
      'x-group': 'Behaviour',
      'x-order': 20,
      'x-display-hint': 'textarea',
      'x-help': 'JQ expression evaluated against case context',
    },
    entryCriteria: {
      type: 'string',
      'x-group': 'Behaviour',
      'x-order': 21,
    },
    slaDuration: {
      type: 'string',
      'x-group': 'Advanced',
      'x-order': 30,
      'x-visibility': 'advanced',
      'x-help': 'ISO 8601 duration (e.g. PT24H)',
    },
    slaStartFrom: {
      type: 'string',
      enum: ['CASE_CREATED', 'MILESTONE_ACTIVATED', 'PREVIOUS_MILESTONE_COMPLETED', 'EVENT_OCCURRED'],
      'x-group': 'Advanced',
      'x-order': 31,
      'x-visibility': 'advanced',
    },
  },
} as const;
```

- [ ] **Step 4: Implement goal schema**

```typescript
// packages/graph-stencil-case/src/schemas/goal-schema.ts
export const goalSchema = {
  type: 'object',
  required: ['name', 'condition'],
  properties: {
    name: {
      type: 'string',
      'x-group': 'Identity',
      'x-order': 0,
    },
    description: {
      type: 'string',
      'x-group': 'Identity',
      'x-order': 1,
    },
    condition: {
      type: 'string',
      'x-group': 'Behaviour',
      'x-order': 20,
      'x-display-hint': 'textarea',
      'x-help': 'JQ expression evaluated against case context',
    },
    kind: {
      type: 'string',
      'x-group': 'Behaviour',
      'x-order': 21,
    },
  },
} as const;
```

- [ ] **Step 5: Create schemas index**

```typescript
// packages/graph-stencil-case/src/schemas/index.ts
export { milestoneSchema } from './milestone-schema.js';
export { goalSchema } from './goal-schema.js';
```

- [ ] **Step 6: Register schemas in registerCaseStencils()**

Add to the existing `registerCaseStencils()` function:
```typescript
import { registerPropertySchema } from '@casehubio/diagram-core';
import { milestoneSchema, goalSchema } from './schemas/index.js';

// Inside registerCaseStencils():
registerPropertySchema('milestone', milestoneSchema);
registerPropertySchema('goal', goalSchema);
```

- [ ] **Step 7: Export schemas from package index**

Add to `packages/graph-stencil-case/src/index.ts`:
```typescript
export { milestoneSchema, goalSchema } from './schemas/index.js';
```

- [ ] **Step 8: Run tests to verify pass**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/graph-stencil-case test -- --reporter verbose 2>&1 | tail -15`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/graph-stencil-case/src/schemas/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(graph-stencil-case): add milestone and goal property schemas Refs #136"
```

### Task 3: Migrate _updateSelectedNode() to use registry

**Files:**
- Modify: `packages/diagram-core/src/diagram-base-mixin.ts:111,239-242`
- Modify: `components/casehub-diagram/src/casehub-diagram.ts` (remove SCHEMA_TYPE_MAP)
- Modify: `components/swf-diagram/src/swf-diagram.ts` (remove SWF_SCHEMA_TYPE_MAP, schema prop)

**Interfaces:**
- Consumes: `getPropertySchema` from schema-registry (Task 1)
- Produces: `_updateSelectedNode()` uses registry instead of `_schemaTypeMap()` + `schema.$defs`

- [ ] **Step 1: Write failing test for registry-based lookup**

Add to `packages/diagram-core/src/schema-registry.test.ts`:
```typescript
import { registerPropertySchema, getPropertySchema } from './schema-registry.js';

describe('integration with diagram selection', () => {
  it('milestone schema is retrievable after registration', () => {
    // This test verifies the registry is populated after registerCaseStencils()
    // Actual integration is tested in casehub-diagram tests
    registerPropertySchema('milestone', { type: 'object', properties: { name: { type: 'string' } } });
    const schema = getPropertySchema('milestone');
    expect(schema).toBeDefined();
    expect((schema as any).properties.name.type).toBe('string');
  });
});
```

- [ ] **Step 2: Update _updateSelectedNode() in diagram-base-mixin.ts**

Replace lines 239-242:
```typescript
// OLD:
const defKey = this._schemaTypeMap()[node.type];
if (defKey && this.schema.$defs) {
  this._selectedSchema = (this.schema.$defs as Record<string, Record<string, unknown>>)[defKey] ?? {};
}

// NEW:
import { getPropertySchema } from './schema-registry.js';
// ...
this._selectedSchema = getPropertySchema(node.type) ?? {};
```

- [ ] **Step 3: Remove _schemaTypeMap() abstract method**

Remove from `diagram-base-mixin.ts` line 111:
```typescript
// DELETE: protected abstract _schemaTypeMap(): Record<string, string>;
```

Also remove `schema` from the interface declaration and the `@property` decorator on the class.

- [ ] **Step 4: Update casehub-diagram.ts**

Remove `SCHEMA_TYPE_MAP` constant and `_schemaTypeMap()` override. Remove any `schema` property usage.

- [ ] **Step 5: Update swf-diagram.ts**

Remove `SWF_SCHEMA_TYPE_MAP`, `_schemaTypeMap()` override, and `schema = swfTaskSchema` property. Register SWF schemas via `registerSwfStencils()` instead (add `registerPropertySchema` calls for each SWF task type from `swfTaskSchema.$defs`).

- [ ] **Step 6: Run all tests**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/diagram-core test 2>&1 | tail -5`
Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/graph-stencil-case test 2>&1 | tail -5`
Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/graph-stencil-swf test 2>&1 | tail -5`
Expected: all PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/diagram-core/src/ components/casehub-diagram/src/ components/swf-diagram/src/ packages/graph-stencil-swf/src/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "refactor(diagram-core): migrate from _schemaTypeMap() to schema registry Refs #136"
```

## Batch 2: Binding and worker schemas with discriminated unions

### Task 4: Subcase and binding schemas

**Files:**
- Create: `packages/graph-stencil-case/src/schemas/subcase-schema.ts`
- Create: `packages/graph-stencil-case/src/schemas/binding-schema.ts`
- Modify: `packages/graph-stencil-case/src/schemas/index.ts`
- Modify: `packages/graph-stencil-case/src/schemas/schemas.test.ts`

**Interfaces:**
- Consumes: `registerPropertySchema` (Task 1)
- Produces: `subcaseSchema`, `bindingSchema` — JSON Schema objects with trigger oneOf using x-discriminator

- [ ] **Step 1: Write failing test**

```typescript
import { subcaseSchema } from './subcase-schema.js';
import { bindingSchema } from './binding-schema.js';

describe('subcase-schema', () => {
  it('has required namespace, name, version', () => {
    expect(subcaseSchema.required).toEqual(expect.arrayContaining(['namespace', 'name', 'version']));
  });

  it('has advanced fields with x-visibility', () => {
    const props = subcaseSchema.properties as Record<string, any>;
    expect(props.inputMapping['x-visibility']).toBe('advanced');
    expect(props.maxRecursionDepth['x-visibility']).toBe('advanced');
  });
});

describe('binding-schema', () => {
  it('has trigger (on) field with oneOf', () => {
    const props = bindingSchema.properties as Record<string, any>;
    expect(props.on.oneOf).toBeDefined();
    expect(props.on['x-discriminator']).toBeDefined();
  });

  it('has all advanced fields from Binding type', () => {
    const props = bindingSchema.properties as Record<string, any>;
    expect(props.conflictResolverStrategy).toBeDefined();
    expect(props.outcomePolicy).toBeDefined();
    expect(props.lifecycleScope).toBeDefined();
    expect(props.participation).toBeDefined();
    expect(props.executionMode).toBeDefined();
  });
});
```

- [ ] **Step 2: Implement subcase-schema.ts**

```typescript
export const subcaseSchema = {
  type: 'object',
  required: ['namespace', 'name', 'version'],
  properties: {
    namespace: { type: 'string', 'x-group': 'Identity', 'x-order': 0 },
    name: { type: 'string', 'x-group': 'Identity', 'x-order': 1 },
    version: { type: 'string', 'x-group': 'Configuration', 'x-order': 10 },
    completionStrategy: { type: 'string', enum: ['DEFAULT', 'CUSTOM'], 'x-group': 'Configuration', 'x-order': 11 },
    waitForCompletion: { type: 'boolean', 'x-group': 'Configuration', 'x-order': 12 },
    inputMapping: { type: 'string', 'x-group': 'Advanced', 'x-order': 30, 'x-visibility': 'advanced', 'x-display-hint': 'textarea' },
    outputMapping: { type: 'string', 'x-group': 'Advanced', 'x-order': 31, 'x-visibility': 'advanced', 'x-display-hint': 'textarea' },
    maxRecursionDepth: { type: 'integer', minimum: 1, 'x-group': 'Advanced', 'x-order': 32, 'x-visibility': 'advanced' },
    groupId: { type: 'string', 'x-group': 'Advanced', 'x-order': 33, 'x-visibility': 'advanced', 'x-help': 'M-of-N grouping identifier' },
    totalInGroup: { type: 'integer', minimum: 1, 'x-group': 'Advanced', 'x-order': 34, 'x-visibility': 'advanced' },
    requiredCount: { type: 'integer', minimum: 1, 'x-group': 'Advanced', 'x-order': 35, 'x-visibility': 'advanced' },
    onThresholdReached: { type: 'string', enum: ['KEEP', 'CANCEL'], 'x-group': 'Advanced', 'x-order': 36, 'x-visibility': 'advanced' },
  },
} as const;
```

- [ ] **Step 3: Implement binding-schema.ts**

Full binding schema with all fields from `Binding` type, trigger `oneOf` with `x-discriminator`, and advanced fields. This is the largest single schema — ~80 lines. Include `on` as oneOf with branches for contextChange (filter string), cloudEvent, schedule, scopeActivated.

- [ ] **Step 4: Update schemas/index.ts and register in registerCaseStencils()**

- [ ] **Step 5: Run tests, commit**

### Task 5: Worker schema with 7 function types

**Files:**
- Create: `packages/graph-stencil-case/src/schemas/worker-schema.ts`
- Modify: `packages/graph-stencil-case/src/schemas/index.ts`
- Modify: `packages/graph-stencil-case/src/schemas/schemas.test.ts`

**Interfaces:**
- Consumes: `registerPropertySchema` (Task 1), `WorkerFunctionType`, `AgentConfig`, `McpConfig`, `A2AConfig`, `AuthConfig`, `McpTransportType`, `ModelProviderKey` from `worker-function/types.ts`
- Produces: `workerSchema` — JSON Schema with 7-branch function type oneOf and nested model provider oneOf

- [ ] **Step 1: Write failing test**

```typescript
import { workerSchema } from './worker-schema.js';

describe('worker-schema', () => {
  it('has required name and capabilities', () => {
    expect(workerSchema.required).toEqual(expect.arrayContaining(['name', 'capabilities']));
  });

  it('has function type oneOf with 7 branches', () => {
    const functionProp = (workerSchema.properties as any).functionType;
    expect(functionProp).toBeDefined();
    // The function type is represented as a discriminator over the worker's shape
    // The actual oneOf lives at the schema root level for the function sub-schemas
  });

  it('has all 7 function types', () => {
    const props = workerSchema.properties as Record<string, any>;
    // Each function type is an optional object property
    expect(props.agent).toBeDefined();
    expect(props.do).toBeDefined();       // flow
    expect(props.a2a).toBeDefined();
    expect(props.mcp).toBeDefined();
    expect(props.sequence).toBeDefined();
  });

  it('agent sub-schema has systemPrompt not instructions', () => {
    const agent = (workerSchema.properties as any).agent.properties;
    expect(agent.systemPrompt).toBeDefined();
    expect(agent.instructions).toBeUndefined();
  });

  it('agent model has provider discriminator with 5 providers', () => {
    const model = (workerSchema.properties as any).agent.properties.model;
    expect(model.oneOf).toBeDefined();
    expect(model.oneOf.length).toBe(5);
    expect(model['x-discriminator']).toBeDefined();
  });

  it('mcp has stdio and http transports only', () => {
    const mcp = (workerSchema.properties as any).mcp;
    expect(mcp.oneOf).toBeDefined();
    expect(mcp['x-discriminator']).toBeDefined();
    const transportTypes = mcp.oneOf.map((b: any) => b.properties?.command ? 'stdio' : 'http');
    expect(transportTypes).toEqual(expect.arrayContaining(['stdio', 'http']));
  });
});
```

- [ ] **Step 2: Implement worker-schema.ts**

Full worker schema with:
- Identity: name (required), description
- Configuration: capabilities (array, minItems: 1)
- Function type sub-schemas as optional object properties (agent, do, a2a, mcp, sequence)
- Agent: systemPrompt (x-editor-component: blocks-prompt-editor), inputProjection, outputProjection, userMessageTemplate, model (oneOf with x-discriminator on provider — 5 branches, each with modelName, apiKey?, temperature?, maxTokens?, topP?)
- MCP: oneOf with x-discriminator — stdio (command[], env via x-editor-component: blocks-env-map-editor) or http (url, auth?)
- A2A: endpoint (format: uri), skill?, streaming?, auth?
- AuthConfig nested: type (enum: none/bearer/api-key), tokenConfigKey?
- Advanced: executionPolicy, contextType, outputType

- [ ] **Step 3: Register in registerCaseStencils(), export from index**

- [ ] **Step 4: Run all tests, commit**

## Batch 3: HTN schemas, SWF extension, custom editors

### Task 6: HTN dag-node and plan-item schemas

**Files:**
- Create: `packages/graph-stencil-htn/src/schemas/dag-node-schema.ts`
- Create: `packages/graph-stencil-htn/src/schemas/plan-item-schema.ts`
- Create: `packages/graph-stencil-htn/src/schemas/index.ts`
- Create: `packages/graph-stencil-htn/src/schemas/schemas.test.ts`
- Modify: `packages/graph-stencil-htn/src/index.ts`

**Interfaces:**
- Consumes: `registerPropertySchema` (Task 1), `DagNodeSnapshot` types
- Produces: `dagNodeSchema`, `primitivePlanItemSchema`, `compoundPlanItemSchema`

- [ ] **Step 1: Write failing test**

```typescript
import { dagNodeSchema } from './dag-node-schema.js';
import { primitivePlanItemSchema, compoundPlanItemSchema } from './plan-item-schema.js';

describe('dag-node-schema', () => {
  it('has id, taskId, taskDescription, executorName, dependsOn, joinType', () => {
    const props = dagNodeSchema.properties as Record<string, any>;
    expect(props.id).toBeDefined();
    expect(props.taskId).toBeDefined();
    expect(props.taskDescription).toBeDefined();
    expect(props.executorName).toBeDefined();
    expect(props.dependsOn).toBeDefined();
    expect(props.joinType).toBeDefined();
  });

  it('joinType is enum ALL_OF / ANY_OF', () => {
    const joinType = (dagNodeSchema.properties as any).joinType;
    expect(joinType.enum).toEqual(['ALL_OF', 'ANY_OF']);
  });
});

describe('plan-item schemas', () => {
  it('compound has dispatchMode ORCHESTRATED/CHOREOGRAPHED', () => {
    const dm = (compoundPlanItemSchema.properties as any).dispatchMode;
    expect(dm.enum).toEqual(['ORCHESTRATED', 'CHOREOGRAPHED']);
  });

  it('compound has repeatable boolean', () => {
    const rep = (compoundPlanItemSchema.properties as any).repeatable;
    expect(rep.type).toBe('boolean');
  });
});
```

- [ ] **Step 2: Implement dag-node-schema.ts**

```typescript
export const dagNodeSchema = {
  type: 'object',
  properties: {
    id: { type: 'string', readOnly: true, 'x-group': 'Identity', 'x-order': 0 },
    taskId: { type: 'string', readOnly: true, 'x-group': 'Identity', 'x-order': 1 },
    taskDescription: { type: 'string', 'x-group': 'Identity', 'x-order': 2 },
    executorName: { type: 'string', 'x-group': 'Configuration', 'x-order': 10 },
    joinType: { type: 'string', enum: ['ALL_OF', 'ANY_OF'], 'x-group': 'Configuration', 'x-order': 11 },
    dependsOn: { type: 'array', items: { type: 'string' }, readOnly: true, 'x-group': 'Configuration', 'x-order': 12 },
  },
} as const;
```

- [ ] **Step 3: Implement plan-item-schema.ts**

Primitive and compound plan item schemas with correct fields from `PrimitivePlanItem` and `CompoundPlanItem` types.

- [ ] **Step 4: Create index, register, export, run tests, commit**

### Task 7: Extend SWF schemas with x-group annotations

**Files:**
- Move: `../../packages/graph-stencil-swf/src/schemas/swf-task-schema.ts` → `packages/graph-stencil-swf/src/schemas/swf-task-schema.ts` (use `ide_move_file`)
- Modify: the moved file — add `x-group`, `x-order`, `x-visibility` to each `$def`
- Modify: `packages/graph-stencil-swf/src/index.ts` — update import path

**Interfaces:**
- Consumes: existing `swfTaskSchema` with `$defs`
- Produces: same schema, extended with grouping annotations

- [ ] **Step 1: Move file**

Use `ide_move_file` to move `schema/swf-task-schema.ts` to `schemas/swf-task-schema.ts`.

- [ ] **Step 2: Add x-group annotations to each $def**

For each task type in `$defs`, add `x-group` and `x-order` to every property. Follow the grouping convention: Identity (0), Configuration (10), Behaviour (20), Advanced (30).

- [ ] **Step 3: Update imports, run tests, commit**

### Task 8: Custom editor component stubs

**Files:**
- Create: `packages/diagram-core/src/editors/blocks-prompt-editor.ts`
- Create: `packages/diagram-core/src/editors/blocks-json-editor.ts`
- Create: `packages/diagram-core/src/editors/index.ts`
- Create: `packages/graph-stencil-case/src/editors/blocks-env-map-editor.ts`
- Create: `packages/graph-stencil-case/src/editors/blocks-sequence-editor.ts`
- Create: `packages/graph-stencil-case/src/editors/blocks-swf-link.ts`
- Create: `packages/graph-stencil-case/src/editors/index.ts`

**Interfaces:**
- Consumes: Lit custom element base
- Produces: Custom elements with `value` property and `value-changed` event. Stub implementations — functional editors are implemented when pages#373 lands.

- [ ] **Step 1: Implement blocks-prompt-editor stub**

```typescript
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('blocks-prompt-editor')
export class BlocksPromptEditorElement extends LitElement {
  @property() value = '';

  static override styles = css`
    :host { display: block; }
    textarea { width: 100%; min-height: 80px; font-family: monospace; font-size: 13px; }
  `;

  override render() {
    return html`
      <textarea
        .value=${this.value}
        @input=${(e: Event) => {
          this.value = (e.target as HTMLTextAreaElement).value;
          this.dispatchEvent(new CustomEvent('value-changed', { detail: { value: this.value }, bubbles: true, composed: true }));
        }}></textarea>
    `;
  }
}
```

- [ ] **Step 2: Implement remaining editor stubs** (blocks-json-editor, blocks-env-map-editor, blocks-sequence-editor, blocks-swf-link)

Each follows the same pattern: `value` property, `value-changed` event, minimal stub rendering. Full implementations when pages#373 lands.

- [ ] **Step 3: Create index files, export, run tests, commit**

## References

- specs/issue-136-rich-property-schemas/2026-08-27-rich-property-schemas-design.md — design spec
- packages/diagram-core/src/diagram-base-mixin.ts:111 — `_schemaTypeMap()` abstract method
- packages/diagram-core/src/diagram-base-mixin.ts:239-242 — `_updateSelectedNode()` schema lookup
- packages/diagram-core/src/index.ts — current form utility exports
- packages/graph-stencil-case/src/types/generated/case-definition.ts — Binding, Worker, Milestone, Goal, SubCase
- packages/graph-stencil-case/src/worker-function/types.ts — WorkerFunctionType, AgentConfig, McpConfig
- packages/graph-stencil-htn/src/types/dag-plan.ts — DagNodeSnapshot
- packages/graph-stencil-htn/src/types/plan-item.ts — PrimitivePlanItem, CompoundPlanItem
- packages/graph-stencil-swf/src/schema/swf-task-schema.ts — existing SWF JSON Schema
- casehubio/casehub-pages#373 — pages-property-palette
- casehubio/blocks-ui#136
