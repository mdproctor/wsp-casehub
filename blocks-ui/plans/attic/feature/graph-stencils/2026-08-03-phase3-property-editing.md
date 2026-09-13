# Phase 3: Property Editing

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — Epic: Visual Diagram Editor — Domain Layer
**Issue group:** #103

**Goal:** Click a node in the diagram, edit its properties in a side panel with YAML round-trip, undo/redo, validation, and complex property editors.

**Architecture:** Property edits flow through `CaseAdapter.applyPropertyEdit()` (adapter owns all YAML mutations). The edit cycle is synchronous — property edits skip ELK re-layout since they don't change graph topology. Undo/redo uses YAML string snapshots. Form rendering is two-tier: flat fields from schema + domain-specific editors for oneOf/nested types.

**Tech Stack:** yaml npm (CST-preserving Document API), lit-html (form templates), graph-core (model), vitest (tests)

## Global Constraints

- TypeScript strict mode with `exactOptionalPropertyTypes: true`
- All events from Shadow DOM components use `composed: true, bubbles: true`
- Field paths are arrays `['outcomePolicy', 'onDecline']` — never dot-separated strings
- Empty optional fields → delete the key from YAML (not empty string)
- Property edits skip `computeElkLayout()` — reuse existing node positions
- `applyPropertyEdit()` creates a fresh `yaml.Document` per call — no persistent Document state

---

### Task 1: applyPropertyEdit — YAML CST editing utility

**Files:**
- Create: `packages/graph-stencil-case/src/adapter/yaml-editor.ts`
- Create: `packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`

**Interfaces:**
- Consumes: `yaml` npm package (`parseDocument`, `Document`)
- Produces: `applyPropertyEdit(yaml: string, nodePath: readonly (string | number)[], field: readonly (string | number)[], value: unknown): string` — used by Task 6 (casehub-diagram edit cycle)

- [ ] **Step 1: Write failing tests**

Create `packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { applyPropertyEdit } from './yaml-editor.js';

const SAMPLE_YAML = `dsl: "1.0.0"
namespace: test
name: sample
version: "1.0.0"
spec:
  bindings:
    - name: scan
      capability: ocr
      when: '.doc != null'
      on:
        contextChange:
          filter: '.doc != null'
  workers:
    - name: ocr-worker
      capabilities:
        - ocr
  milestones:
    - name: extracted
      condition: '.result != null'
`;

describe('applyPropertyEdit', () => {
  it('updates a string property', () => {
    const result = applyPropertyEdit(
      SAMPLE_YAML,
      ['spec', 'bindings', 0],
      ['when'],
      '.ocrResult != null',
    );
    expect(result).toContain("when: '.ocrResult != null'");
    expect(result).toContain('name: scan');
  });

  it('preserves formatting of untouched sections', () => {
    const result = applyPropertyEdit(
      SAMPLE_YAML,
      ['spec', 'bindings', 0],
      ['when'],
      '.changed',
    );
    expect(result).toContain('namespace: test');
    expect(result).toContain("dsl: \"1.0.0\"");
    expect(result).toContain('capabilities:\n        - ocr');
  });

  it('updates a nested property via array path', () => {
    const result = applyPropertyEdit(
      SAMPLE_YAML,
      ['spec', 'milestones', 0],
      ['condition'],
      '.done == true',
    );
    expect(result).toContain("condition: '.done == true'");
  });

  it('coerces number values', () => {
    const yaml = SAMPLE_YAML + '  goals:\n    - name: g1\n      retries: 3\n';
    const result = applyPropertyEdit(
      yaml,
      ['spec', 'goals', 0],
      ['retries'],
      5,
    );
    expect(result).toContain('retries: 5');
    expect(result).not.toContain("retries: '5'");
  });

  it('deletes key when value is undefined', () => {
    const result = applyPropertyEdit(
      SAMPLE_YAML,
      ['spec', 'bindings', 0],
      ['when'],
      undefined,
    );
    expect(result).not.toContain('when:');
    expect(result).toContain('name: scan');
  });

  it('handles deep nested path', () => {
    const yaml = `dsl: "1.0.0"
namespace: test
name: sample
version: "1.0.0"
spec:
  bindings:
    - name: b1
      outcomePolicy:
        onDecline: REROUTE
        maxRerouteAttempts: 3
`;
    const result = applyPropertyEdit(
      yaml,
      ['spec', 'bindings', 0],
      ['outcomePolicy', 'onDecline'],
      'FAULT',
    );
    expect(result).toContain('onDecline: FAULT');
    expect(result).toContain('maxRerouteAttempts: 3');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement applyPropertyEdit**

Create `packages/graph-stencil-case/src/adapter/yaml-editor.ts`:

```typescript
import { parseDocument } from 'yaml';

export function applyPropertyEdit(
  yaml: string,
  nodePath: readonly (string | number)[],
  field: readonly (string | number)[],
  value: unknown,
): string {
  const doc = parseDocument(yaml);
  const fullPath = [...nodePath, ...field];

  if (value === undefined) {
    doc.deleteIn(fullPath);
  } else {
    doc.setIn(fullPath, value);
  }

  return doc.toString();
}
```

- [ ] **Step 4: Add export to index.ts**

Add to `packages/graph-stencil-case/src/index.ts`:
```typescript
export { applyPropertyEdit } from './adapter/yaml-editor.js';
```

- [ ] **Step 5: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: All pass.

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add packages/graph-stencil-case/src/adapter/yaml-editor.ts packages/graph-stencil-case/src/adapter/yaml-editor.test.ts packages/graph-stencil-case/src/index.ts
git -C $PROJECT commit -m "feat(#103): applyPropertyEdit — CST-preserving YAML property editing"
```

---

### Task 2: toGraph → AdapterResult with yamlPaths

**Files:**
- Modify: `packages/graph-stencil-case/src/adapter/case-adapter.ts`
- Modify: `packages/graph-stencil-case/src/adapter/case-adapter.test.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`

**Interfaces:**
- Consumes: existing `toGraph` implementation
- Produces: `AdapterResult { model: GraphModel; yamlPaths: ReadonlyMap<string, readonly (string | number)[]> }` — used by Task 6

- [ ] **Step 1: Write failing test for yamlPaths**

Add to `packages/graph-stencil-case/src/adapter/case-adapter.test.ts`:

```typescript
  it('returns yamlPaths mapping node IDs to YAML document paths', () => {
    const result = toGraph(EXAMPLE_YAML);
    expect(result.yamlPaths).toBeDefined();
    expect(result.yamlPaths.get('worker:ocr-worker')).toEqual(['spec', 'workers', 0]);
    expect(result.yamlPaths.get('binding:extract-text')).toEqual(['spec', 'bindings', 1]);
    expect(result.yamlPaths.get('milestone:text-extracted')).toEqual(['spec', 'milestones', 0]);
    expect(result.yamlPaths.get('goal:processingComplete')).toEqual(['spec', 'goals', 0]);
  });
```

Update existing test assertions to use `result.model` instead of `result` directly (since toGraph now returns AdapterResult).

- [ ] **Step 2: Run tests to verify they fail**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: FAIL — `result.yamlPaths` is undefined, `result.model` is undefined (still returns GraphModel directly).

- [ ] **Step 3: Update toGraph to return AdapterResult**

Modify `packages/graph-stencil-case/src/adapter/case-adapter.ts`:

Add the `AdapterResult` interface:
```typescript
export interface AdapterResult {
  readonly model: GraphModel;
  readonly yamlPaths: ReadonlyMap<string, readonly (string | number)[]>;
}
```

Change `toGraph` return type from `GraphModel` to `AdapterResult`. Track paths alongside nodes:

```typescript
export function toGraph(yaml: string): AdapterResult {
  const def = parseYaml(yaml) as CaseDefinition;
  const nodes: GraphNode[] = [];
  const edges: GraphEdge[] = [];
  const yamlPaths = new Map<string, (string | number)[]>();

  const capabilityToWorker = new Map<string, string>();

  let workerIndex = 0;
  for (const worker of def.spec.workers ?? []) {
    const nodeId = `worker:${worker.name}`;
    nodes.push({ id: nodeId, type: 'worker', properties: { ...worker } });
    yamlPaths.set(nodeId, ['spec', 'workers', workerIndex]);
    for (const cap of worker.capabilities) {
      capabilityToWorker.set(cap, nodeId);
    }
    workerIndex++;
  }

  let bindingIndex = 0;
  for (const binding of def.spec.bindings ?? []) {
    const nodeId = binding.name ? `binding:${binding.name}` : `binding:_${bindingIndex}`;
    nodes.push({ id: nodeId, type: 'binding', properties: { ...binding } });
    yamlPaths.set(nodeId, ['spec', 'bindings', bindingIndex]);

    // ... edge derivation unchanged ...

    bindingIndex++;
  }

  let milestoneIndex = 0;
  for (const milestone of def.spec.milestones ?? []) {
    const nodeId = `milestone:${milestone.name}`;
    nodes.push({ id: nodeId, type: 'milestone', properties: { ...milestone } });
    yamlPaths.set(nodeId, ['spec', 'milestones', milestoneIndex]);
    milestoneIndex++;
  }

  let goalIndex = 0;
  for (const goal of def.spec.goals ?? []) {
    const nodeId = `goal:${goal.name}`;
    nodes.push({ id: nodeId, type: 'goal', properties: { ...goal } });
    yamlPaths.set(nodeId, ['spec', 'goals', goalIndex]);
    goalIndex++;
  }

  return { model: createGraph(nodes, edges), yamlPaths };
}
```

- [ ] **Step 4: Update all existing tests**

Change all `toGraph(EXAMPLE_YAML)` assertions to use `result.model`:
```typescript
const result = toGraph(EXAMPLE_YAML);
const workers = result.model.nodes.filter(n => n.type === 'worker');
```

Also update `react-flow-transform.test.ts` if it calls `toGraph`. Update `casehub-diagram.ts` to destructure: `const { model } = toGraph(yamlStr);` then `toReactFlowGraph(model)`.

- [ ] **Step 5: Update index.ts exports**

Add `AdapterResult` type export:
```typescript
export type { AdapterResult } from './adapter/case-adapter.js';
```

- [ ] **Step 6: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```
Expected: All pass including the new yamlPaths test.

Also verify casehub-diagram still builds:
```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case build
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```

- [ ] **Step 7: Commit**

```bash
git -C $PROJECT add packages/graph-stencil-case/src/adapter/case-adapter.ts packages/graph-stencil-case/src/adapter/case-adapter.test.ts packages/graph-stencil-case/src/index.ts components/casehub-diagram/src/casehub-diagram.ts
git -C $PROJECT commit -m "feat(#103): toGraph returns AdapterResult with yamlPaths for property editing"
```

---

### Task 3: Validation + Field Renderer

**Files:**
- Create: `components/casehub-diagram/src/form/validation.ts`
- Create: `components/casehub-diagram/src/form/validation.test.ts`
- Create: `components/casehub-diagram/src/form/field-renderer.ts`
- Create: `components/casehub-diagram/src/form/field-renderer.test.ts`

**Interfaces:**
- Consumes: JSON Schema property definitions (from CaseDefinition.yaml $defs)
- Produces: `validateField(schema, value, required): string | null` and `fieldTypeFor(schema): FieldType` — used by Task 5 (properties panel)

- [ ] **Step 1: Write validation tests**

Create `components/casehub-diagram/src/form/validation.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { validateField } from './validation.js';

describe('validateField', () => {
  it('returns "Required" for empty required string', () => {
    expect(validateField({ type: 'string' }, '', true)).toBe('Required');
  });

  it('returns null for empty optional string', () => {
    expect(validateField({ type: 'string' }, '', false)).toBeNull();
  });

  it('returns null for valid string', () => {
    expect(validateField({ type: 'string' }, 'hello', true)).toBeNull();
  });

  it('validates minimum on number', () => {
    expect(validateField({ type: 'integer', minimum: 1 }, 0, false)).toBe('Must be at least 1');
  });

  it('validates maximum on number', () => {
    expect(validateField({ type: 'integer', maximum: 10 }, 15, false)).toBe('Must be at most 10');
  });

  it('validates minLength on string', () => {
    expect(validateField({ type: 'string', minLength: 3 }, 'ab', true)).toBe('Must be at least 3 characters');
  });

  it('validates pattern on string', () => {
    expect(validateField({ type: 'string', pattern: '^[a-z]+$' }, 'ABC', false)).toBe('Invalid format');
  });

  it('returns null when number is in range', () => {
    expect(validateField({ type: 'integer', minimum: 1, maximum: 10 }, 5, false)).toBeNull();
  });
});
```

- [ ] **Step 2: Implement validation**

Create `components/casehub-diagram/src/form/validation.ts`:

```typescript
export interface FieldSchema {
  readonly type?: string;
  readonly minimum?: number;
  readonly maximum?: number;
  readonly minLength?: number;
  readonly maxLength?: number;
  readonly pattern?: string;
  readonly enum?: readonly string[];
  readonly description?: string;
  readonly [k: string]: unknown;
}

export function validateField(
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): string | null {
  const str = typeof value === 'string' ? value : '';
  const isEmpty = str === '' && typeof value !== 'number' && typeof value !== 'boolean';

  if (required && isEmpty) return 'Required';
  if (isEmpty) return null;

  if ((schema.type === 'integer' || schema.type === 'number') && typeof value === 'number') {
    if (schema.minimum !== undefined && value < schema.minimum) {
      return `Must be at least ${schema.minimum}`;
    }
    if (schema.maximum !== undefined && value > schema.maximum) {
      return `Must be at most ${schema.maximum}`;
    }
  }

  if (schema.type === 'string' && typeof value === 'string') {
    if (schema.minLength !== undefined && value.length < schema.minLength) {
      return `Must be at least ${schema.minLength} characters`;
    }
    if (schema.pattern !== undefined) {
      const re = new RegExp(schema.pattern);
      if (!re.test(value)) return 'Invalid format';
    }
  }

  return null;
}
```

- [ ] **Step 3: Write field-renderer tests**

Create `components/casehub-diagram/src/form/field-renderer.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { fieldTypeFor } from './field-renderer.js';

describe('fieldTypeFor', () => {
  it('returns "text" for plain string', () => {
    expect(fieldTypeFor({ type: 'string' })).toBe('text');
  });

  it('returns "select" for string with enum', () => {
    expect(fieldTypeFor({ type: 'string', enum: ['A', 'B'] })).toBe('select');
  });

  it('returns "number" for integer', () => {
    expect(fieldTypeFor({ type: 'integer' })).toBe('number');
  });

  it('returns "number" for number', () => {
    expect(fieldTypeFor({ type: 'number' })).toBe('number');
  });

  it('returns "checkbox" for boolean', () => {
    expect(fieldTypeFor({ type: 'boolean' })).toBe('checkbox');
  });

  it('returns "textarea" for JQ expression descriptions', () => {
    expect(fieldTypeFor({ type: 'string', description: 'JQ predicate over CaseContext' })).toBe('textarea');
  });

  it('returns "textarea" for expression descriptions', () => {
    expect(fieldTypeFor({ type: 'string', description: 'JQ expression producing output' })).toBe('textarea');
  });

  it('returns "string-array" for array of strings', () => {
    expect(fieldTypeFor({ type: 'array', items: { type: 'string' } })).toBe('string-array');
  });

  it('returns "object" for nested object with properties', () => {
    expect(fieldTypeFor({ type: 'object', properties: { a: { type: 'string' } } })).toBe('object');
  });

  it('returns "json" for object with only additionalProperties', () => {
    expect(fieldTypeFor({ type: 'object', additionalProperties: true })).toBe('json');
  });

  it('returns "json" for array of objects', () => {
    expect(fieldTypeFor({ type: 'array', items: { type: 'object' } })).toBe('json');
  });

  it('returns "oneOf" when oneOf is present', () => {
    expect(fieldTypeFor({ oneOf: [{ type: 'string' }, { type: 'object' }] })).toBe('oneOf');
  });
});
```

- [ ] **Step 4: Implement field-renderer**

Create `components/casehub-diagram/src/form/field-renderer.ts`:

```typescript
import type { FieldSchema } from './validation.js';

export type FieldType =
  | 'text'
  | 'textarea'
  | 'number'
  | 'checkbox'
  | 'select'
  | 'string-array'
  | 'object'
  | 'json'
  | 'oneOf';

export function fieldTypeFor(schema: FieldSchema): FieldType {
  if (schema.oneOf) return 'oneOf';

  if (schema.type === 'boolean') return 'checkbox';

  if (schema.type === 'integer' || schema.type === 'number') return 'number';

  if (schema.type === 'string') {
    if (schema.enum && (schema.enum as readonly string[]).length > 0) return 'select';
    const desc = (schema.description ?? '').toLowerCase();
    if (desc.includes('jq') || desc.includes('expression')) return 'textarea';
    return 'text';
  }

  if (schema.type === 'array') {
    const items = schema.items as FieldSchema | undefined;
    if (items?.type === 'string') return 'string-array';
    return 'json';
  }

  if (schema.type === 'object') {
    if (schema.properties && Object.keys(schema.properties as object).length > 0) return 'object';
    return 'json';
  }

  return 'text';
}
```

- [ ] **Step 5: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```
Expected: All pass (existing integration tests + new validation + field-renderer tests).

- [ ] **Step 6: Commit**

```bash
git -C $PROJECT add components/casehub-diagram/src/form/
git -C $PROJECT commit -m "feat(#103): validation + field-renderer — form infrastructure for property editing"
```

---

### Task 4: Trigger Editor + Nested Group

**Files:**
- Create: `components/casehub-diagram/src/form/trigger-editor.ts`
- Create: `components/casehub-diagram/src/form/nested-group.ts`
- Create: `components/casehub-diagram/src/form/trigger-editor.test.ts`

**Interfaces:**
- Consumes: `FieldSchema`, `fieldTypeFor`, `validateField` from Task 3
- Produces: `renderTriggerEditor(data, onChange)` and `renderNestedGroup(name, schema, data, onChange)` — lit-html template functions used by Task 5

- [ ] **Step 1: Write trigger-editor tests**

Create `components/casehub-diagram/src/form/trigger-editor.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { detectTriggerType } from './trigger-editor.js';

describe('detectTriggerType', () => {
  it('detects contextChange', () => {
    expect(detectTriggerType({ contextChange: { filter: '.x' } })).toBe('contextChange');
  });

  it('detects cloudEvent string form', () => {
    expect(detectTriggerType({ cloudEvent: 'document.received' })).toBe('cloudEvent');
  });

  it('detects cloudEvent object form', () => {
    expect(detectTriggerType({ cloudEvent: { type: 'document.received' } })).toBe('cloudEvent');
  });

  it('detects schedule', () => {
    expect(detectTriggerType({ schedule: { cron: '*/5 * * * *' } })).toBe('schedule');
  });

  it('detects scopeActivated', () => {
    expect(detectTriggerType({ scopeActivated: {} })).toBe('scopeActivated');
  });

  it('returns null for empty', () => {
    expect(detectTriggerType({})).toBeNull();
  });
});
```

- [ ] **Step 2: Implement trigger-editor**

Create `components/casehub-diagram/src/form/trigger-editor.ts`:

```typescript
import { html, nothing, type TemplateResult } from 'lit-html';

export type TriggerType = 'contextChange' | 'cloudEvent' | 'schedule' | 'scopeActivated';
const TRIGGER_TYPES: TriggerType[] = ['contextChange', 'cloudEvent', 'schedule', 'scopeActivated'];
const TRIGGER_LABELS: Record<TriggerType, string> = {
  contextChange: 'Context Change',
  cloudEvent: 'Cloud Event',
  schedule: 'Schedule',
  scopeActivated: 'Scope Activated',
};

export function detectTriggerType(on: Record<string, unknown>): TriggerType | null {
  for (const t of TRIGGER_TYPES) {
    if (on[t] !== undefined) return t;
  }
  return null;
}

export function renderTriggerEditor(
  data: Record<string, unknown>,
  onChange: (value: Record<string, unknown>) => void,
): TemplateResult {
  const current = detectTriggerType(data);

  const handleTypeChange = (type: TriggerType) => {
    const newTrigger: Record<string, unknown> = {};
    if (type === 'contextChange') newTrigger.contextChange = {};
    else if (type === 'cloudEvent') newTrigger.cloudEvent = '';
    else if (type === 'schedule') newTrigger.schedule = {};
    else if (type === 'scopeActivated') newTrigger.scopeActivated = {};
    onChange(newTrigger);
  };

  const handleSubFieldChange = (field: string, value: unknown) => {
    if (!current) return;
    const sub = (data[current] ?? {}) as Record<string, unknown>;
    const updated = { ...sub, [field]: value };
    onChange({ [current]: updated });
  };

  return html`
    <fieldset style="border: 1px solid var(--pages-border-color, #ddd); border-radius: 6px; padding: 8px; margin: 4px 0;">
      <legend style="font-size: 12px; font-weight: 600; color: var(--pages-text-secondary, #666);">Trigger</legend>
      <div style="display: flex; gap: 8px; margin-bottom: 8px; flex-wrap: wrap;">
        ${TRIGGER_TYPES.map(t => html`
          <label style="display: flex; align-items: center; gap: 4px; font-size: 12px; cursor: pointer;">
            <input type="radio" name="trigger-type" .checked=${current === t}
              @change=${() => handleTypeChange(t)}>
            ${TRIGGER_LABELS[t]}
          </label>
        `)}
      </div>
      ${current === 'contextChange' ? renderContextChangeSub(data.contextChange as Record<string, unknown> ?? {}, handleSubFieldChange) : nothing}
      ${current === 'cloudEvent' ? renderCloudEventSub(data.cloudEvent, handleSubFieldChange, onChange) : nothing}
      ${current === 'schedule' ? renderScheduleSub(data.schedule as Record<string, unknown> ?? {}, handleSubFieldChange) : nothing}
    </fieldset>
  `;
}

function renderContextChangeSub(
  sub: Record<string, unknown>,
  onChange: (field: string, value: unknown) => void,
): TemplateResult {
  return html`
    <div style="display: flex; flex-direction: column; gap: 6px;">
      <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">Filter
        <textarea rows="2" style="width: 100%; font-family: monospace; font-size: 12px;"
          .value=${String(sub['filter'] ?? '')}
          @blur=${(e: Event) => onChange('filter', (e.target as HTMLTextAreaElement).value)}
        ></textarea>
      </label>
      <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">Listen Layer
        <input type="text" style="width: 100%; font-size: 12px;"
          .value=${String(sub['listenLayer'] ?? '')}
          @blur=${(e: Event) => onChange('listenLayer', (e.target as HTMLInputElement).value || undefined)}
        >
      </label>
    </div>
  `;
}

function renderCloudEventSub(
  sub: unknown,
  onSubChange: (field: string, value: unknown) => void,
  onFullChange: (value: Record<string, unknown>) => void,
): TemplateResult {
  if (typeof sub === 'string') {
    return html`
      <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">Event Type
        <input type="text" style="width: 100%; font-size: 12px;"
          .value=${sub}
          @blur=${(e: Event) => onFullChange({ cloudEvent: (e.target as HTMLInputElement).value })}
        >
      </label>
    `;
  }
  const obj = (sub ?? {}) as Record<string, unknown>;
  return html`
    <div style="display: flex; flex-direction: column; gap: 6px;">
      <label style="font-size: 11px;">Type
        <input type="text" style="width: 100%; font-size: 12px;" .value=${String(obj['type'] ?? '')}
          @blur=${(e: Event) => onSubChange('type', (e.target as HTMLInputElement).value)}>
      </label>
      <label style="font-size: 11px;">Source
        <input type="text" style="width: 100%; font-size: 12px;" .value=${String(obj['source'] ?? '')}
          @blur=${(e: Event) => onSubChange('source', (e.target as HTMLInputElement).value || undefined)}>
      </label>
      <label style="font-size: 11px;">Filter
        <textarea rows="2" style="width: 100%; font-family: monospace; font-size: 12px;" .value=${String(obj['filter'] ?? '')}
          @blur=${(e: Event) => onSubChange('filter', (e.target as HTMLTextAreaElement).value || undefined)}>
        </textarea>
      </label>
    </div>
  `;
}

function renderScheduleSub(
  sub: Record<string, unknown>,
  onChange: (field: string, value: unknown) => void,
): TemplateResult {
  const hasCron = sub['cron'] !== undefined;
  return html`
    <div style="display: flex; flex-direction: column; gap: 6px;">
      <div style="display: flex; gap: 8px;">
        <label style="font-size: 11px; display: flex; align-items: center; gap: 4px;">
          <input type="radio" name="schedule-mode" .checked=${hasCron}
            @change=${() => { onChange('every', undefined); onChange('cron', sub['cron'] ?? ''); }}>
          Cron
        </label>
        <label style="font-size: 11px; display: flex; align-items: center; gap: 4px;">
          <input type="radio" name="schedule-mode" .checked=${!hasCron}
            @change=${() => { onChange('cron', undefined); onChange('every', sub['every'] ?? ''); }}>
          Every
        </label>
      </div>
      ${hasCron
        ? html`<input type="text" style="width: 100%; font-size: 12px;" placeholder="*/5 * * * *"
            .value=${String(sub['cron'] ?? '')}
            @blur=${(e: Event) => onChange('cron', (e.target as HTMLInputElement).value)}>`
        : html`<input type="text" style="width: 100%; font-size: 12px;" placeholder="PT5M"
            .value=${String(sub['every'] ?? '')}
            @blur=${(e: Event) => onChange('every', (e.target as HTMLInputElement).value)}>`
      }
      <label style="font-size: 11px;">Timezone
        <input type="text" style="width: 100%; font-size: 12px;" placeholder="America/Vancouver"
          .value=${String(sub['timezone'] ?? '')}
          @blur=${(e: Event) => onChange('timezone', (e.target as HTMLInputElement).value || undefined)}>
      </label>
    </div>
  `;
}
```

- [ ] **Step 3: Implement nested-group**

Create `components/casehub-diagram/src/form/nested-group.ts`:

```typescript
import { html, nothing, type TemplateResult } from 'lit-html';
import { fieldTypeFor } from './field-renderer.js';
import { validateField, type FieldSchema } from './validation.js';

export function renderNestedGroup(
  name: string,
  schema: FieldSchema,
  data: Record<string, unknown> | undefined,
  onChange: (field: string, value: unknown) => void,
): TemplateResult {
  const props = schema.properties as Record<string, FieldSchema> | undefined;
  if (!props) return html`<pre style="font-size: 11px; color: var(--pages-text-tertiary, #999);">${JSON.stringify(data, null, 2)}</pre>`;

  const requiredSet = new Set((schema.required as string[] | undefined) ?? []);
  const current = data ?? {};

  return html`
    <details open style="border: 1px solid var(--pages-border-color, #ddd); border-radius: 4px; margin: 4px 0;">
      <summary style="padding: 6px 8px; font-size: 12px; font-weight: 600; cursor: pointer; color: var(--pages-text-secondary, #666);">${name}</summary>
      <div style="padding: 4px 8px 8px; display: flex; flex-direction: column; gap: 6px;">
        ${Object.entries(props).map(([key, fieldSchema]) => {
          const ft = fieldTypeFor(fieldSchema);
          const value = current[key];
          const required = requiredSet.has(key);
          const error = value !== undefined ? validateField(fieldSchema, value, required) : null;
          const label = (fieldSchema as { title?: string }).title ?? key;

          if (ft === 'object') {
            return renderNestedGroup(key, fieldSchema, value as Record<string, unknown> | undefined, (subField, subVal) => {
              const updated = { ...(current[key] as Record<string, unknown> ?? {}), [subField]: subVal };
              onChange(key, updated);
            });
          }

          if (ft === 'json') {
            return html`
              <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">${label}
                <pre style="font-size: 11px; color: var(--pages-text-tertiary, #999); margin: 2px 0;">${JSON.stringify(value, null, 2) ?? '—'}</pre>
              </label>
            `;
          }

          if (ft === 'select') {
            const opts = (fieldSchema.enum ?? []) as string[];
            return html`
              <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">${label}${required ? ' *' : ''}
                <select style="width: 100%; font-size: 12px;"
                  @change=${(e: Event) => onChange(key, (e.target as HTMLSelectElement).value)}>
                  <option value="">—</option>
                  ${opts.map(o => html`<option .selected=${value === o}>${o}</option>`)}
                </select>
              </label>
              ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
            `;
          }

          if (ft === 'number') {
            return html`
              <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">${label}${required ? ' *' : ''}
                <input type="number" style="width: 100%; font-size: 12px;"
                  .value=${String(value ?? '')}
                  @blur=${(e: Event) => {
                    const v = (e.target as HTMLInputElement).value;
                    onChange(key, v === '' ? undefined : Number(v));
                  }}>
              </label>
              ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
            `;
          }

          return html`
            <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">${label}${required ? ' *' : ''}
              <input type="text" style="width: 100%; font-size: 12px;"
                .value=${String(value ?? '')}
                @blur=${(e: Event) => {
                  const v = (e.target as HTMLInputElement).value;
                  onChange(key, v === '' ? undefined : v);
                }}>
            </label>
            ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
          `;
        })}
      </div>
    </details>
  `;
}
```

- [ ] **Step 4: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```
Expected: All pass.

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add components/casehub-diagram/src/form/trigger-editor.ts components/casehub-diagram/src/form/trigger-editor.test.ts components/casehub-diagram/src/form/nested-group.ts
git -C $PROJECT commit -m "feat(#103): trigger editor + nested group — complex property form components"
```

---

### Task 5: casehub-diagram-properties Panel

**Files:**
- Create: `components/casehub-diagram/src/casehub-diagram-properties.ts`
- Create: `components/casehub-diagram/src/casehub-diagram-properties.test.ts`

**Interfaces:**
- Consumes: `fieldTypeFor` (Task 3), `validateField` (Task 3), `renderTriggerEditor` (Task 4), `renderNestedGroup` (Task 4)
- Produces: `<casehub-diagram-properties>` custom element with `schema`, `data`, `readonly` properties. Emits `property-change` events with `composed: true`.

- [ ] **Step 1: Write failing test**

Create `components/casehub-diagram/src/casehub-diagram-properties.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest';
import { renderPropertyForm, emitPropertyChange } from './casehub-diagram-properties.js';

describe('renderPropertyForm', () => {
  it('renders text input for string property', () => {
    const schema = {
      type: 'object',
      properties: { name: { type: 'string' } },
    };
    const result = renderPropertyForm(schema, { name: 'test' }, false, vi.fn());
    expect(result).toBeDefined();
    expect(result.strings).toBeDefined();
  });

  it('renders select for enum property', () => {
    const schema = {
      type: 'object',
      properties: {
        kind: { type: 'string', enum: ['success', 'failure'] },
      },
    };
    const result = renderPropertyForm(schema, { kind: 'success' }, false, vi.fn());
    expect(result).toBeDefined();
  });
});

describe('emitPropertyChange', () => {
  it('creates a composed CustomEvent', () => {
    const event = emitPropertyChange(['when'], '.done');
    expect(event.type).toBe('property-change');
    expect(event.composed).toBe(true);
    expect(event.bubbles).toBe(true);
    expect(event.detail.field).toEqual(['when']);
    expect(event.detail.value).toBe('.done');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement casehub-diagram-properties**

Create `components/casehub-diagram/src/casehub-diagram-properties.ts`:

```typescript
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { customElement, property as litProp } from 'lit/decorators.js';
import { fieldTypeFor } from './form/field-renderer.js';
import { validateField, type FieldSchema } from './form/validation.js';
import { renderTriggerEditor } from './form/trigger-editor.js';
import { renderNestedGroup } from './form/nested-group.js';

export function emitPropertyChange(
  field: (string | number)[],
  value: unknown,
): CustomEvent<{ field: (string | number)[]; value: unknown }> {
  return new CustomEvent('property-change', {
    bubbles: true,
    composed: true,
    detail: { field, value },
  });
}

export function renderPropertyForm(
  schema: Record<string, unknown>,
  data: Record<string, unknown>,
  readonly: boolean,
  onChange: (field: (string | number)[], value: unknown) => void,
): TemplateResult {
  const props = (schema.properties ?? {}) as Record<string, FieldSchema>;
  const requiredSet = new Set((schema.required as string[] | undefined) ?? []);

  const fields = Object.keys(props).filter(k => !k.startsWith('_'));
  if (fields.includes('name')) {
    fields.splice(fields.indexOf('name'), 1);
    fields.unshift('name');
  }

  return html`
    <div style="display: flex; flex-direction: column; gap: 8px;">
      ${fields.map(key => {
        const fieldSchema = props[key]!;
        const ft = fieldTypeFor(fieldSchema);
        const value = data[key];
        const required = requiredSet.has(key);
        const error = value !== undefined ? validateField(fieldSchema, value, required) : null;
        const label = (fieldSchema as { title?: string }).title ?? key.replace(/([A-Z])/g, ' $1').replace(/^./, s => s.toUpperCase());

        if (key === 'on' && ft === 'oneOf') {
          return renderTriggerEditor(
            (value ?? {}) as Record<string, unknown>,
            v => onChange(['on'], v),
          );
        }

        if (ft === 'object') {
          return renderNestedGroup(label, fieldSchema, value as Record<string, unknown> | undefined, (subField, subVal) => {
            onChange([key, subField], subVal);
          });
        }

        if (ft === 'json') {
          return html`
            <label style="font-size: 11px; color: var(--pages-text-secondary, #666);">${label}
              <pre style="font-size: 11px; background: var(--pages-surface-raised, #f5f5f5); padding: 4px 6px; border-radius: 3px; overflow-x: auto; margin: 2px 0;">${JSON.stringify(value, null, 2) ?? '—'}</pre>
            </label>
          `;
        }

        if (ft === 'select') {
          const opts = (fieldSchema.enum ?? []) as string[];
          return html`
            <label style="font-size: 12px; color: var(--pages-text-color, #333);">${label}${required ? ' *' : ''}
              <select style="width: 100%; font-size: 12px; padding: 4px;" ?disabled=${readonly}
                @change=${(e: Event) => onChange([key], (e.target as HTMLSelectElement).value || undefined)}>
                <option value="">—</option>
                ${opts.map(o => html`<option .selected=${value === o}>${o}</option>`)}
              </select>
            </label>
            ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
          `;
        }

        if (ft === 'checkbox') {
          return html`
            <label style="font-size: 12px; display: flex; align-items: center; gap: 6px; color: var(--pages-text-color, #333);">
              <input type="checkbox" ?checked=${Boolean(value)} ?disabled=${readonly}
                @change=${(e: Event) => onChange([key], (e.target as HTMLInputElement).checked)}>
              ${label}
            </label>
          `;
        }

        if (ft === 'number') {
          const min = fieldSchema.minimum;
          const max = fieldSchema.maximum;
          return html`
            <label style="font-size: 12px; color: var(--pages-text-color, #333);">${label}${required ? ' *' : ''}
              <input type="number" style="width: 100%; font-size: 12px; padding: 4px;"
                .value=${String(value ?? '')} ?disabled=${readonly}
                min=${min ?? nothing} max=${max ?? nothing}
                @blur=${(e: Event) => {
                  const v = (e.target as HTMLInputElement).value;
                  onChange([key], v === '' ? undefined : Number(v));
                }}>
            </label>
            ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
          `;
        }

        if (ft === 'textarea') {
          return html`
            <label style="font-size: 12px; color: var(--pages-text-color, #333);">${label}${required ? ' *' : ''}
              <textarea rows="3" style="width: 100%; font-family: monospace; font-size: 12px; padding: 4px;" ?disabled=${readonly}
                .value=${String(value ?? '')}
                @blur=${(e: Event) => {
                  const v = (e.target as HTMLTextAreaElement).value;
                  onChange([key], v === '' && !required ? undefined : v);
                }}></textarea>
            </label>
            ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
          `;
        }

        if (ft === 'string-array') {
          const arr = (value as string[] | undefined) ?? [];
          return html`
            <label style="font-size: 12px; color: var(--pages-text-color, #333);">${label}
              <textarea rows="3" style="width: 100%; font-size: 12px; padding: 4px;" ?disabled=${readonly}
                .value=${arr.join('\n')}
                @blur=${(e: Event) => {
                  const lines = (e.target as HTMLTextAreaElement).value.split('\n').filter(l => l.trim());
                  onChange([key], lines.length > 0 ? lines : undefined);
                }}></textarea>
              <span style="font-size: 10px; color: var(--pages-text-tertiary, #999);">One per line</span>
            </label>
          `;
        }

        return html`
          <label style="font-size: 12px; color: var(--pages-text-color, #333);">${label}${required ? ' *' : ''}
            <input type="text" style="width: 100%; font-size: 12px; padding: 4px;" ?disabled=${readonly}
              .value=${String(value ?? '')}
              @blur=${(e: Event) => {
                const v = (e.target as HTMLInputElement).value;
                onChange([key], v === '' && !required ? undefined : v);
              }}>
          </label>
          ${error ? html`<div style="color: red; font-size: 11px;">${error}</div>` : nothing}
        `;
      })}
    </div>
  `;
}

@customElement('casehub-diagram-properties')
export class CasehubDiagramProperties extends LitElement {
  @litProp({ attribute: false }) schema: Record<string, unknown> = {};
  @litProp({ attribute: false }) data: Record<string, unknown> = {};
  @litProp({ type: Boolean }) readonly = false;

  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .panel { padding: 12px; overflow-y: auto; height: 100%; box-sizing: border-box; }
    .panel-header { font-size: 14px; font-weight: 700; margin-bottom: 12px; color: var(--pages-text-color, #333); }
  `;

  override render() {
    const nodeName = String(this.data['name'] ?? this.data['type'] ?? 'Properties');

    return html`
      <div class="panel">
        <div class="panel-header">${nodeName}</div>
        ${renderPropertyForm(this.schema, this.data, this.readonly, (field, value) => {
          this.dispatchEvent(emitPropertyChange(field, value));
        })}
      </div>
    `;
  }
}
```

- [ ] **Step 4: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```
Expected: All pass.

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add components/casehub-diagram/src/casehub-diagram-properties.ts components/casehub-diagram/src/casehub-diagram-properties.test.ts
git -C $PROJECT commit -m "feat(#103): casehub-diagram-properties — schema-driven property panel with Shadow DOM"
```

---

### Task 6: casehub-diagram — Split Layout, Selection, Undo/Redo, Edit Cycle

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram.ts` (major rewrite)
- Modify: `components/casehub-diagram/src/casehub-diagram.test.ts` (add edit cycle + undo tests)

**Interfaces:**
- Consumes: `toGraph` → `AdapterResult` (Task 2), `applyPropertyEdit` (Task 1), `toReactFlowGraph`, `registerCaseStencils`, `computeElkLayout`, `<casehub-diagram-properties>` (Task 5)
- Produces: Updated `<casehub-diagram>` with split layout, selection, property editing, undo/redo

- [ ] **Step 1: Write failing tests for edit cycle and undo**

Add to `components/casehub-diagram/src/casehub-diagram.test.ts`:

```typescript
import { applyPropertyEdit, toGraph, toReactFlowGraph } from '@casehubio/graph-stencil-case';

describe('edit cycle', () => {
  it('applyPropertyEdit updates YAML and re-parse produces updated model', () => {
    const result = toGraph(EXAMPLE_YAML);
    const bindingPath = result.yamlPaths.get('binding:extract-text');
    expect(bindingPath).toBeDefined();

    const newYaml = applyPropertyEdit(
      EXAMPLE_YAML,
      [...bindingPath!],
      ['when'],
      '.changed == true',
    );
    const updated = toGraph(newYaml);
    const binding = updated.model.nodes.find(n => n.id === 'binding:extract-text');
    expect(binding!.properties['when']).toBe('.changed == true');
  });

  it('skipping re-layout preserves node positions', () => {
    const result = toGraph(EXAMPLE_YAML);
    const { nodes } = toReactFlowGraph(result.model);
    const positioned = nodes.map(n => ({ ...n, position: { x: 100, y: 200 } }));

    const newYaml = applyPropertyEdit(
      EXAMPLE_YAML,
      [...result.yamlPaths.get('binding:extract-text')!],
      ['when'],
      '.changed',
    );
    const updated = toGraph(newYaml);
    const { nodes: newNodes } = toReactFlowGraph(updated.model);

    const merged = newNodes.map(n => {
      const existing = positioned.find(p => p.id === n.id);
      return existing ? { ...n, position: existing.position } : n;
    });

    const binding = merged.find(n => n.id === 'binding:extract-text')!;
    expect(binding.position).toEqual({ x: 100, y: 200 });
    expect(binding.data['when']).toBe('.changed');
  });
});
```

- [ ] **Step 2: Implement casehub-diagram rewrite**

Rewrite `components/casehub-diagram/src/casehub-diagram.ts`:

```typescript
import { LitElement, html, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import {
  toGraph,
  toReactFlowGraph,
  registerCaseStencils,
  applyPropertyEdit,
} from '@casehubio/graph-stencil-case';
import type { AdapterResult, RFNode, RFEdge } from '@casehubio/graph-stencil-case';
import { computeElkLayout } from '@casehubio/graph-renderer';
import type { GraphModel } from '@casehubio/graph-core';
import '@casehubio/graph-renderer';
import './casehub-diagram-properties.js';

const SCHEMA_TYPE_MAP: Record<string, string> = {
  binding: 'Binding',
  worker: 'Worker',
  milestone: 'Milestone',
  goal: 'Goal',
  subcase: 'SubCase',
};

const MAX_UNDO = 50;

@customElement('casehub-diagram')
export class CasehubDiagram extends LitElement {
  @property() yaml = '';
  @property() src = '';
  @property({ attribute: false }) schema: Record<string, unknown> = {};

  @state() private _nodes: RFNode[] = [];
  @state() private _edges: RFEdge[] = [];
  @state() private _error = '';
  @state() private _selectedNodeId = '';
  @state() private _selectedData: Record<string, unknown> = {};
  @state() private _selectedSchema: Record<string, unknown> = {};

  private _currentYaml = '';
  private _adapterResult: AdapterResult | null = null;
  private _undoStack: string[] = [];
  private _redoStack: string[] = [];

  override createRenderRoot(): HTMLElement {
    return this;
  }

  override connectedCallback(): void {
    super.connectedCallback();
    registerCaseStencils();
    this.addEventListener('keydown', this._handleKeydown);
  }

  override disconnectedCallback(): void {
    this.removeEventListener('keydown', this._handleKeydown);
    super.disconnectedCallback();
  }

  override async updated(changed: Map<string, unknown>): Promise<void> {
    if (changed.has('yaml') && this.yaml) {
      this._currentYaml = this.yaml;
      this._undoStack = [];
      this._redoStack = [];
      this._selectedNodeId = '';
      await this._fullRender(this.yaml);
    }
    if (changed.has('src') && this.src) {
      try {
        const response = await fetch(this.src);
        const text = await response.text();
        this._currentYaml = text;
        this._undoStack = [];
        this._redoStack = [];
        this._selectedNodeId = '';
        await this._fullRender(text);
      } catch (e) {
        this._error = `Failed to fetch ${this.src}: ${e}`;
      }
    }
  }

  private async _fullRender(yamlStr: string): Promise<void> {
    try {
      this._error = '';
      this._adapterResult = toGraph(yamlStr);
      const { nodes, edges } = toReactFlowGraph(this._adapterResult.model);
      this._nodes = await computeElkLayout(nodes, edges, { direction: 'DOWN', spacing: 60 }) as RFNode[];
      this._edges = edges;
    } catch (e) {
      this._error = String(e);
    }
  }

  private _updateWithoutLayout(yamlStr: string): void {
    try {
      this._error = '';
      this._adapterResult = toGraph(yamlStr);
      const { nodes, edges } = toReactFlowGraph(this._adapterResult.model);
      const posMap = new Map(this._nodes.map(n => [n.id, n.position]));
      this._nodes = nodes.map(n => ({
        ...n,
        position: posMap.get(n.id) ?? n.position,
      }));
      this._edges = edges;
      this._updateSelectedNode();
    } catch (e) {
      this._error = `Edit failed: ${e}`;
      this._currentYaml = this._undoStack.pop() ?? this._currentYaml;
    }
  }

  private _updateSelectedNode(): void {
    if (!this._selectedNodeId || !this._adapterResult) {
      this._selectedData = {};
      this._selectedSchema = {};
      return;
    }
    const node = this._adapterResult.model.nodes.find(n => n.id === this._selectedNodeId);
    if (!node) {
      this._selectedNodeId = '';
      this._selectedData = {};
      this._selectedSchema = {};
      return;
    }
    this._selectedData = { ...node.properties };
    const defKey = SCHEMA_TYPE_MAP[node.type];
    if (defKey && this.schema.$defs) {
      this._selectedSchema = (this.schema.$defs as Record<string, Record<string, unknown>>)[defKey] ?? {};
    }
  }

  private _handleNodeClick = (e: Event): void => {
    const detail = (e as CustomEvent<{ nodeId: string }>).detail;
    this._selectedNodeId = detail.nodeId;
    this._updateSelectedNode();
  };

  private _handleSelectionChange = (e: Event): void => {
    const detail = (e as CustomEvent<{ nodeIds: string[] }>).detail;
    if (detail.nodeIds.length === 0) {
      this._selectedNodeId = '';
      this._selectedData = {};
      this._selectedSchema = {};
    }
  };

  private _handlePropertyChange = (e: Event): void => {
    const detail = (e as CustomEvent<{ field: (string | number)[]; value: unknown }>).detail;
    if (!this._selectedNodeId || !this._adapterResult) return;

    const nodePath = this._adapterResult.yamlPaths.get(this._selectedNodeId);
    if (!nodePath) return;

    this._undoStack.push(this._currentYaml);
    if (this._undoStack.length > MAX_UNDO) this._undoStack.shift();
    this._redoStack = [];

    try {
      this._currentYaml = applyPropertyEdit(
        this._currentYaml,
        nodePath,
        detail.field,
        detail.value,
      );
      this._updateWithoutLayout(this._currentYaml);
    } catch (e) {
      this._currentYaml = this._undoStack.pop() ?? this._currentYaml;
      this._error = `Edit failed: ${e}`;
    }
  };

  private _handleKeydown = (e: KeyboardEvent): void => {
    if (e.key === 'Escape') {
      this._selectedNodeId = '';
      this._selectedData = {};
      this._selectedSchema = {};
      return;
    }
    if ((e.ctrlKey || e.metaKey) && e.key === 'z' && !e.shiftKey) {
      e.preventDefault();
      this._undo();
    }
    if ((e.ctrlKey || e.metaKey) && e.key === 'z' && e.shiftKey) {
      e.preventDefault();
      this._redo();
    }
  };

  private _undo(): void {
    if (this._undoStack.length === 0) return;
    this._redoStack.push(this._currentYaml);
    this._currentYaml = this._undoStack.pop()!;
    this._updateWithoutLayout(this._currentYaml);
  }

  private _redo(): void {
    if (this._redoStack.length === 0) return;
    this._undoStack.push(this._currentYaml);
    this._currentYaml = this._redoStack.pop()!;
    this._updateWithoutLayout(this._currentYaml);
  }

  override render() {
    if (this._error) {
      return html`<div style="color: red; padding: 16px;">${this._error}</div>`;
    }
    const hasSelection = this._selectedNodeId !== '';
    const isExternal = hasSelection && this._adapterResult?.model.nodes.find(n => n.id === this._selectedNodeId)?.type === 'external';

    return html`
      <div style="display: flex; width: 100%; height: 100%;">
        <pages-graph-canvas
          .nodes=${this._nodes}
          .edges=${this._edges}
          style="flex: 1; height: 100%;"
          @pages-event=${(e: CustomEvent) => {
            const topic = e.detail?.topic as string | undefined;
            if (topic === 'graph:node-click') this._handleNodeClick(e);
            if (topic === 'graph:selection-change') this._handleSelectionChange(e);
          }}
        ></pages-graph-canvas>
        ${hasSelection ? html`
          <div style="width: 300px; border-left: 1px solid var(--pages-border-color, #ddd); overflow-y: auto;">
            <casehub-diagram-properties
              .schema=${this._selectedSchema}
              .data=${this._selectedData}
              ?readonly=${isExternal ?? false}
              @property-change=${this._handlePropertyChange}
            ></casehub-diagram-properties>
          </div>
        ` : nothing}
      </div>
    `;
  }
}
```

- [ ] **Step 3: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram test
```
Expected: All pass — existing integration tests + new edit cycle + undo tests.

- [ ] **Step 4: Build verification**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case build
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/blocks-ui-casehub-diagram build
```
Expected: Both build clean.

- [ ] **Step 5: Commit**

```bash
git -C $PROJECT add components/casehub-diagram/src/casehub-diagram.ts components/casehub-diagram/src/casehub-diagram.test.ts
git -C $PROJECT commit -m "feat(#103): property editing — split layout, selection, undo/redo, YAML round-trip"
```
