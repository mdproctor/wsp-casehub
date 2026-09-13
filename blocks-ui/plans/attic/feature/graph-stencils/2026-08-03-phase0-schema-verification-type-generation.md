# Phase 0: Schema Verification + TypeScript Type Generation

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #103 — Epic: Visual Diagram Editor — Domain Layer
**Issue group:** #103

**Goal:** Verify the CaseDefinition.yaml JSON Schema is current, then generate complete TypeScript types for the visual diagram editor.

**Architecture:** The JSON Schema at `engine/schema/src/main/resources/schema/CaseDefinition.yaml` is the single source of truth — both Java (via jsonschema2pojo) and TypeScript types derive from it. A generator script in graph-stencil-case reads the YAML schema, strips engine-only codegen directives, and produces TypeScript interfaces via `json-schema-to-typescript`. Generated output is checked into git.

**Tech Stack:** `json-schema-to-typescript` (type generation), `yaml` npm (YAML parsing — already a dependency), `tsx` (script runner), `vitest` (tests — already configured)

## Global Constraints

- TypeScript strict mode with `exactOptionalPropertyTypes: true` (tsconfig.base.json)
- No `any` types — strict mode throughout
- `yaml` v2.7+ already in graph-stencil-case dependencies
- Engine repo at `../../engine/` relative to graph-stencil-case (peer repo layout)
- `_codegen*` properties on CaseDefinitionSpec are engine-only — strip before generation

---

### Task 1: Schema Audit

Non-code task. Compare the JSON Schema against the hand-written Java model extensions and example YAML files. Document findings.

**Files:**
- Read: `engine/schema/src/main/resources/schema/CaseDefinition.yaml` (already loaded)
- Read: `engine/schema/src/main/java/io/casehub/model/Worker.java` (already loaded)
- Read: `engine/schema/src/main/resources/examples/document-processing.yaml` (already loaded)
- Create: `specs/feature-graph-stencils/phase0-schema-audit-findings.md` (workspace)

**Produces:** Findings document. Any schema patches needed for engine.

- [ ] **Step 1: Compare Worker.java fields with schema Worker $def**

The hand-written `Worker.java` has custom Jackson serialization and these fields:

| Java field | In schema? | Notes |
|-----------|-----------|-------|
| `name` | Yes | `required` |
| `description` | Yes | |
| `capabilities` | Yes | `required` |
| `executionPolicy` | Yes | `$ref ExecutionPolicy` |
| `sequence` | Yes | |
| `contextType` | Yes | |
| `outputType` | Yes | |
| `inputSchema` | No (caught by `additionalProperties: true`) | Plugin-supplied, not a definition-view concern |
| `outputSchema` | No (caught by `additionalProperties: true`) | Plugin-supplied, not a definition-view concern |
| `workflow` | No (caught by `additionalProperties: true`) | SWF workflow — plugin-supplied via `do:` syntax |
| `agent` | No (caught by `additionalProperties: true`) | AI agent config — plugin-supplied |

**Verdict:** Schema is correct. `additionalProperties: true` on Worker is intentional — worker functions are plugin-supplied. The TypeScript type will have an index signature `[k: string]: unknown` for these properties, which is the right behavior for the editor.

- [ ] **Step 2: Check for stages remnants**

Search the schema for any reference to "stage" or "stages":
```bash
grep -i stage engine/schema/src/main/resources/schema/CaseDefinition.yaml
```
Expected: No matches (stages concept was removed).

- [ ] **Step 3: Verify example YAML uses only schema-declared properties**

Spot-check `document-processing.yaml` — the design spec's reference example. Workers use `do:` syntax (SWF-style workflow definitions) which falls under `additionalProperties: true`. All top-level, binding, milestone, and goal properties match schema declarations.

- [ ] **Step 4: Write findings and commit**

Create `specs/feature-graph-stencils/phase0-schema-audit-findings.md` in the workspace with the audit results. Commit to workspace.

```bash
git -C $WORKSPACE add specs/feature-graph-stencils/phase0-schema-audit-findings.md
git -C $WORKSPACE commit -m "docs(#103): Phase 0 schema audit findings"
```

---

### Task 2: Generator Script

**Files:**
- Create: `packages/graph-stencil-case/scripts/generate-types.ts`
- Modify: `packages/graph-stencil-case/package.json` (add devDeps + script)

**Produces:** A runnable `generate:types` script that reads the YAML schema and outputs TypeScript interfaces.

**Interfaces:**
- Consumes: `CaseDefinition.yaml` (JSON Schema in YAML format)
- Produces: `src/types/generated/case-definition.ts` (TypeScript interfaces for all `$defs`)

- [ ] **Step 1: Add devDependencies**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case add -D json-schema-to-typescript tsx
```

- [ ] **Step 2: Write the generator script**

Create `packages/graph-stencil-case/scripts/generate-types.ts`:

```typescript
import { readFileSync, writeFileSync, mkdirSync } from 'node:fs';
import { resolve, dirname } from 'node:path';
import { parse as parseYaml } from 'yaml';
import { compile } from 'json-schema-to-typescript';

const SCHEMA_DEFAULT_PATH = resolve(
  import.meta.dirname,
  '../../../../engine/schema/src/main/resources/schema/CaseDefinition.yaml',
);

const OUTPUT_PATH = resolve(
  import.meta.dirname,
  '../src/types/generated/case-definition.ts',
);

const CODEGEN_PREFIXES = ['_codegen'];

async function main(): Promise<void> {
  const schemaPath = process.argv[2] ?? SCHEMA_DEFAULT_PATH;

  const yamlContent = readFileSync(schemaPath, 'utf-8');
  const schema = parseYaml(yamlContent) as Record<string, unknown>;

  const spec = (schema as { $defs?: Record<string, unknown> }).$defs?.[
    'CaseDefinitionSpec'
  ] as { properties?: Record<string, unknown> } | undefined;

  if (spec?.properties) {
    for (const key of Object.keys(spec.properties)) {
      if (CODEGEN_PREFIXES.some(prefix => key.startsWith(prefix))) {
        delete spec.properties[key];
      }
    }
  }

  const ts = await compile(schema, 'CaseHub', {
    additionalProperties: false,
    bannerComment: [
      '/* eslint-disable */',
      '/**',
      ' * This file was automatically generated from CaseDefinition.yaml.',
      ' * DO NOT MODIFY BY HAND. Run `yarn generate:types` to regenerate.',
      ' */',
    ].join('\n'),
    style: {
      semi: true,
      singleQuote: true,
      tabWidth: 2,
      trailingComma: 'all',
    },
    strictIndexSignatures: true,
    enableConstEnums: false,
    unknownAny: true,
  });

  mkdirSync(dirname(OUTPUT_PATH), { recursive: true });
  writeFileSync(OUTPUT_PATH, ts, 'utf-8');
  console.log(`Generated: ${OUTPUT_PATH}`);
}

main().catch(err => {
  console.error(err);
  process.exit(1);
});
```

- [ ] **Step 3: Add package.json script**

Add to `packages/graph-stencil-case/package.json` scripts:

```json
"generate:types": "tsx scripts/generate-types.ts"
```

- [ ] **Step 4: Run the generator**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case generate:types
```

Expected: `Generated: .../src/types/generated/case-definition.ts`

- [ ] **Step 5: Inspect and fix generated output**

Read the generated file. Check:
- No `_codegen*` types present
- Worker has `[k: string]: unknown` (from `additionalProperties: true`)
- CaseDefinitionSpec has `[k: string]: unknown` (from `unevaluatedProperties: true`)
- `oneOf` on Binding produces a union type for capability/subCase/humanTask
- `oneOf` on Trigger produces a union type for contextChange/cloudEvent/schedule/scopeActivated
- All `$defs` have corresponding exported interfaces

If `json-schema-to-typescript` produces types that don't compile under `exactOptionalPropertyTypes: true`, post-process the output to fix optional property declarations.

If `json-schema-to-typescript` doesn't handle `unevaluatedProperties` correctly (it's a 2020-12 keyword not universally supported), manually add the index signature to CaseDefinitionSpec in the post-processing step of the script.

- [ ] **Step 6: Commit generator**

```bash
git -C $PROJECT add packages/graph-stencil-case/scripts/generate-types.ts packages/graph-stencil-case/package.json packages/graph-stencil-case/src/types/generated/ yarn.lock
git -C $PROJECT commit -m "feat(#103): add TypeScript type generator from CaseDefinition schema"
```

---

### Task 3: Integration and Verification

**Files:**
- Modify: `packages/graph-stencil-case/src/types/case-definition.ts` (replace placeholder with re-exports)
- Modify: `packages/graph-stencil-case/src/index.ts` (update exports)
- Create: `packages/graph-stencil-case/src/types/case-definition.test.ts` (type-level smoke test)

**Interfaces:**
- Consumes: Generated types from Task 2 (`src/types/generated/case-definition.ts`)
- Produces: Clean public type exports from `@casehubio/graph-stencil-case`

- [ ] **Step 1: Write a smoke test**

Create `packages/graph-stencil-case/src/types/case-definition.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { parse as parseYaml } from 'yaml';
import { readFileSync } from 'node:fs';
import { resolve } from 'node:path';
import type { CaseHub } from './generated/case-definition.js';

describe('CaseDefinition types', () => {
  it('parses document-processing.yaml as CaseHub', () => {
    const yamlPath = resolve(
      import.meta.dirname,
      '../../../../../engine/schema/src/main/resources/examples/document-processing.yaml',
    );
    const content = readFileSync(yamlPath, 'utf-8');
    const parsed = parseYaml(content) as CaseHub;

    expect(parsed.dsl).toBe('1.0.0');
    expect(parsed.namespace).toBe('casehub-examples');
    expect(parsed.name).toBe('document-processing');
    expect(parsed.version).toBe('1.0.0');
    expect(parsed.spec).toBeDefined();
  });

  it('has typed spec.bindings', () => {
    const yamlPath = resolve(
      import.meta.dirname,
      '../../../../../engine/schema/src/main/resources/examples/document-processing.yaml',
    );
    const content = readFileSync(yamlPath, 'utf-8');
    const parsed = parseYaml(content) as CaseHub;
    const bindings = parsed.spec?.bindings;

    expect(bindings).toBeDefined();
    expect(bindings!.length).toBeGreaterThan(0);

    const first = bindings![0]!;
    expect(first.name).toBe('validate-on-upload');
    expect(first.capability).toBe('validate-format');
    expect(first.on).toBeDefined();
  });

  it('has typed spec.milestones', () => {
    const yamlPath = resolve(
      import.meta.dirname,
      '../../../../../engine/schema/src/main/resources/examples/document-processing.yaml',
    );
    const content = readFileSync(yamlPath, 'utf-8');
    const parsed = parseYaml(content) as CaseHub;
    const milestones = parsed.spec?.milestones;

    expect(milestones).toBeDefined();
    expect(milestones![0]!.name).toBe('text-extracted');
    expect(milestones![0]!.condition).toContain('.ocrResult');
  });

  it('has typed spec.goals', () => {
    const yamlPath = resolve(
      import.meta.dirname,
      '../../../../../engine/schema/src/main/resources/examples/document-processing.yaml',
    );
    const content = readFileSync(yamlPath, 'utf-8');
    const parsed = parseYaml(content) as CaseHub;
    const goals = parsed.spec?.goals;

    expect(goals).toBeDefined();
    expect(goals![0]!.name).toBe('processingComplete');
    expect(goals![0]!.kind).toBe('success');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```

Expected: FAIL — `CaseHub` type not exported from `./generated/case-definition.js` yet (or import mismatch with the placeholder file).

- [ ] **Step 3: Replace placeholder types with re-export barrel**

Replace `packages/graph-stencil-case/src/types/case-definition.ts` content with:

```typescript
export type {
  CaseHub as CaseDefinition,
  CaseDefinitionSpec,
  Binding,
  Worker,
  Milestone,
  Goal,
  SubCase,
  Capability,
  HumanTask,
  Trigger,
  ContextChangeTrigger,
  CloudEventTrigger,
  ScheduleTrigger,
  ScopeActivatedTrigger,
  OutcomePolicy,
  ExecutionPolicy,
  RetryPolicy,
  Authorization,
  Cbr,
  Use,
  CaseCompletion,
  GoalExpression,
  LabelRule,
  InboundSignalMapping,
} from './generated/case-definition.js';
```

Note: the root type name from `json-schema-to-typescript` will be based on the schema's `title` field ("CaseHub"). We re-export it as `CaseDefinition` for domain clarity. Adjust the actual type names once the generated output is inspected — `json-schema-to-typescript` may use different names based on schema structure.

- [ ] **Step 4: Update index.ts exports**

Replace `packages/graph-stencil-case/src/index.ts`:

```typescript
export { CaseAdapter } from './adapter/case-adapter.js';
export { caseStencils } from './stencils/index.js';
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

The full set of types is available via deep import (`@casehubio/graph-stencil-case/types`). The barrel export exposes only the types the adapter and stencils consume directly.

- [ ] **Step 5: Run tests**

```bash
GH_PACKAGES_TOKEN=dummy yarn workspace @casehubio/graph-stencil-case test
```

Expected: PASS — all four smoke tests pass with the generated types.

- [ ] **Step 6: Full build verification**

```bash
GH_PACKAGES_TOKEN=dummy yarn build
```

Expected: Clean build across all workspace packages. No type errors.

- [ ] **Step 7: Commit**

```bash
git -C $PROJECT add packages/graph-stencil-case/src/types/ packages/graph-stencil-case/src/index.ts
git -C $PROJECT commit -m "feat(#103): replace placeholder types with generated CaseDefinition types"
```
