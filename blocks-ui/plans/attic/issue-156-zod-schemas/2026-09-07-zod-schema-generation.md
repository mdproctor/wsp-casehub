# Zod Schema Generation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #156 — Audit component interfaces and auto-generate Zod schemas for YAML completion
**Issue group:** #156

**Goal:** Create a schema generation pipeline that reads TypeScript
interfaces from blocks-ui components and produces Zod schemas for YAML
validation and editor completion.

**Architecture:** A `BlocksComponentRegistry` interface in the new
`blocks-ui-schema` package maps element tag names to Props interfaces.
A ts-morph generator reads the registry and produces per-component Zod
schemas + a `blocksComponentSchemaMap`. Each component exports a typed
`FooProps` interface extracted from its `@property()` declarations.

**Tech Stack:** TypeScript, ts-morph, Zod, Vitest, Lit (existing)

## Global Constraints

- TypeScript strict mode (`tsconfig.base.json` enables `strict: true`)
- ESM only (`"type": "module"` in all packages)
- `experimentalDecorators: true`, `useDefineForClassFields: false` (Lit requirement)
- Yarn 4 workspace with `workspace:*` protocol
- Filter out function-typed properties from schemas (render callbacks)
- Depth guard: `depth > 6 → z.unknown()` for nested types

---

## Batch 1: Infrastructure + end-to-end validation

### Task 1: Create blocks-ui-schema package and generator

**Files:**
- Create: `packages/blocks-ui-schema/package.json`
- Create: `packages/blocks-ui-schema/tsconfig.json`
- Create: `packages/blocks-ui-schema/tsconfig.build.json`
- Create: `packages/blocks-ui-schema/vitest.config.ts`
- Create: `packages/blocks-ui-schema/src/index.ts`
- Create: `packages/blocks-ui-schema/src/registry.ts`
- Create: `packages/blocks-ui-schema/scripts/generate-schemas.ts`
- Modify: `tsconfig.json` (root — add project reference)

**Interfaces:**
- Produces: `BlocksComponentRegistry` interface, `generate-schemas.ts`
  script, package scaffolding. Later tasks add entries to the registry.

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-schema",
  "version": "0.1.0",
  "description": "Zod schemas for blocks-ui component properties — auto-generated from TypeScript interfaces",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/blocks-ui.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "generate": "tsx scripts/generate-schemas.ts",
    "build": "yarn generate && tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/blocks-ui-sla-indicator": "workspace:*",
    "@casehubio/blocks-ui-execution-monitor": "workspace:*",
    "@casehubio/blocks-ui-kpi-metric-row": "workspace:*",
    "@casehubio/blocks-ui-approval-gate": "workspace:*",
    "@casehubio/blocks-ui-service-card": "workspace:*",
    "rimraf": "^6.1.0",
    "ts-morph": "^24.0.0",
    "tsx": "^4.19.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.1"
  },
  "license": "Apache-2.0"
}
```

Note: devDependencies will grow as more component Props are added in
Batch 2. Only the initial 5 components are listed here.

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"]
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
  },
});
```

- [ ] **Step 5: Create src/index.ts**

```typescript
export * from './component-schemas.generated.js';
```

- [ ] **Step 6: Create src/registry.ts with initial 5 components**

```typescript
import type { SlaIndicatorProps } from '@casehubio/blocks-ui-sla-indicator';
import type { ExecutionMonitorProps } from '@casehubio/blocks-ui-execution-monitor';
import type { KpiMetricRowProps } from '@casehubio/blocks-ui-kpi-metric-row';
import type { ApprovalGateProps } from '@casehubio/blocks-ui-approval-gate';
import type { ServiceCardProps } from '@casehubio/blocks-ui-service-card';

export interface BlocksComponentRegistry {
  'blocks-sla-indicator': SlaIndicatorProps;
  'blocks-execution-monitor': ExecutionMonitorProps;
  'blocks-kpi-metric-row': KpiMetricRowProps;
  'blocks-approval-gate': ApprovalGateProps;
  'blocks-service-card': ServiceCardProps;
}
```

- [ ] **Step 7: Write the generator script**

Create `packages/blocks-ui-schema/scripts/generate-schemas.ts`. Fork the
pages generator from `casehub-pages/packages/pages-schema/scripts/generate-schemas.ts`
with these adaptations:

- Change the ts-morph Project tsconfig path to resolve blocks-ui-schema's own tsconfig
- Change the source file path to read `BlocksComponentRegistry` from `src/registry.ts`
- Update `FUNCTION_PROPS` set for blocks-ui callbacks:
  `renderAgent`, `renderModel`, `renderCandidate`, `renderDetail`,
  `getRowDetail`, `renderBefore`, `renderAfter`
- Update the HEADER comment to reference blocks-ui-schema
- Change output path to `src/component-schemas.generated.ts`
- Schema naming: `SlaIndicatorProps` → `slaIndicatorPropsSchema`
- Map export: `blocksComponentSchemaMap`

The core `typeToZod()` and `propToZodField()` functions are copied
verbatim from the pages generator — they handle primitives, unions,
arrays, records, objects, and the depth guard.

```typescript
import { Project, Type, Symbol as MorphSymbol } from "ts-morph";
import { writeFileSync } from "fs";
import { resolve, dirname } from "path";
import { fileURLToPath } from "url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

const HEADER = `// AUTO-GENERATED by scripts/generate-schemas.ts — DO NOT EDIT
// Re-generate: yarn workspace @casehubio/blocks-ui-schema run generate
// Source: packages/blocks-ui-schema/src/registry.ts
import { z } from "zod";
`;

const FUNCTION_PROPS = new Set([
  "renderAgent", "renderModel", "renderCandidate", "renderDetail",
  "getRowDetail", "renderBefore", "renderAfter",
]);

function isFunction(type: Type): boolean {
  return type.getCallSignatures().length > 0;
}

function typeToZod(type: Type, depth: number): string {
  if (depth > 6) return "z.unknown()";

  const text = type.getText();

  if (type.isString() || type.isStringLiteral()) return "z.string()";
  if (type.isNumber() || type.isNumberLiteral()) return "z.number()";
  if (type.isBoolean() || type.isBooleanLiteral()) return "z.boolean()";
  if (type.isNull()) return "z.null()";
  if (type.isUndefined()) return "z.undefined()";

  if (type.isUnion()) {
    const members = type.getUnionTypes().filter(t => !t.isUndefined());
    if (members.length === 0) return "z.undefined()";
    if (members.every(m => m.isStringLiteral())) {
      const values = members.map(m => JSON.stringify(m.getLiteralValue()));
      return `z.enum([${values.join(", ")}])`;
    }
    if (members.every(m => m.isBooleanLiteral())) return "z.boolean()";
    if (members.length === 1) return typeToZod(members[0], depth + 1);
    const zodMembers = members.map(m => typeToZod(m, depth + 1));
    return `z.union([${zodMembers.join(", ")}])`;
  }

  if (type.isArray()) {
    const elem = type.getArrayElementType();
    if (!elem) return "z.array(z.unknown())";
    return `z.array(${typeToZod(elem, depth + 1)})`;
  }

  if (text.startsWith("readonly ") && text.endsWith("[]")) {
    const inner = type.getTypeArguments()[0];
    if (inner) return `z.array(${typeToZod(inner, depth + 1)})`;
    return "z.array(z.unknown())";
  }

  if (text.startsWith("Record<") || text.startsWith("Readonly<Record<")
      || text.includes("Record<string,")) {
    const typeArgs = type.getAliasTypeArguments();
    if (typeArgs.length === 2) {
      return `z.record(${typeToZod(typeArgs[1], depth + 1)})`;
    }
    const apparentProps = type.getStringIndexType();
    if (apparentProps) return `z.record(${typeToZod(apparentProps, depth + 1)})`;
    return "z.record(z.unknown())";
  }

  if (type.isObject() && !type.isArray()) {
    const props = type.getProperties();
    if (props.length === 0) return "z.object({})";
    const fields = props
      .map(p => propToZodField(p, depth + 1))
      .filter(Boolean);
    if (fields.length === 0) return "z.object({})";
    const indent = "  ".repeat(depth + 2);
    const closingIndent = "  ".repeat(depth + 1);
    return `z.object({\n${indent}${fields.join(`,\n${indent}`)},\n${closingIndent}})`;
  }

  return "z.unknown()";
}

function propToZodField(prop: MorphSymbol, depth: number): string {
  const name = prop.getName();
  if (FUNCTION_PROPS.has(name)) return "";
  if (name.startsWith("x-")) return "";

  const decl = prop.getValueDeclaration();
  if (!decl) return "";

  const type = prop.getTypeAtLocation(decl);
  if (isFunction(type)) return "";

  const isOptional = prop.isOptional();
  const baseType = isOptional ? type.getNonNullableType() : type;
  let zodType = typeToZod(baseType, depth);
  if (!zodType) return "";
  if (isOptional) zodType += ".optional()";

  const safeName = /^[a-zA-Z_$][a-zA-Z0-9_$]*$/.test(name) ? name : `"${name}"`;
  return `${safeName}: ${zodType}`;
}

function kebabToCamel(s: string): string {
  return s.replace(/-([a-z])/g, (_, c) => c.toUpperCase());
}

const project = new Project({
  tsConfigFilePath: resolve(__dirname, "../tsconfig.json"),
  skipAddingFilesFromTsConfig: false,
});

const registryFile = project.getSourceFileOrThrow(
  resolve(__dirname, "../src/registry.ts"),
);

const registry = registryFile.getInterfaceOrThrow("BlocksComponentRegistry");
const registryType = registry.getType();

const output: string[] = [HEADER];
const schemaNames: { key: string; camelName: string }[] = [];

for (const prop of registryType.getProperties()) {
  const registryKey = prop.getName();

  const decl = prop.getValueDeclaration();
  if (!decl) continue;
  const propType = prop.getTypeAtLocation(decl);

  const typeSymbol = propType.getSymbol() || propType.getAliasSymbol();
  const tsTypeName = typeSymbol?.getName();
  let camelName: string;
  if (tsTypeName && tsTypeName.endsWith("Props")) {
    const base = tsTypeName.slice(0, -5);
    camelName = base.charAt(0).toLowerCase() + base.slice(1) + "PropsSchema";
  } else {
    camelName = kebabToCamel(registryKey) + "PropsSchema";
  }

  const allProps = propType.getProperties();
  const fields = allProps
    .map(p => propToZodField(p, 0))
    .filter(Boolean);

  if (fields.length === 0) {
    output.push(`export const ${camelName} = z.object({});`);
  } else {
    output.push(`export const ${camelName} = z.object({\n  ${fields.join(",\n  ")},\n});`);
  }
  output.push("");
  schemaNames.push({ key: registryKey, camelName });
}

const mapEntries = schemaNames
  .map(({ key, camelName }) => `  ["${key}", ${camelName}]`)
  .join(",\n");
output.push(`export const blocksComponentSchemaMap: ReadonlyMap<string, z.ZodType> = new Map([
${mapEntries},
]);
`);

const outPath = resolve(__dirname, "../src/component-schemas.generated.ts");
writeFileSync(outPath, output.join("\n"), "utf-8");
console.log(`Generated ${schemaNames.length} schemas to ${outPath}`);
```

- [ ] **Step 8: Add project reference to root tsconfig.json**

Add to `tsconfig.json` references array:
```json
{ "path": "packages/blocks-ui-schema/tsconfig.build.json" }
```

- [ ] **Step 9: Run `yarn install` to resolve workspace links**

Run: `yarn install`
Expected: clean install, blocks-ui-schema workspace linked

- [ ] **Step 10: Commit**

```bash
git add packages/blocks-ui-schema/ tsconfig.json yarn.lock
git commit -m "feat(#156): scaffold blocks-ui-schema package with generator Refs #156"
```

### Task 2: Extract Props interfaces for 5 initial components

**Files:**
- Modify: `components/sla-indicator/src/sla-indicator.ts`
- Modify: `components/execution-monitor/src/execution-monitor.ts`
- Modify: `components/kpi-metric-row/src/kpi-metric-row.ts`
- Modify: `components/approval-gate/src/approval-gate.ts`
- Modify: `components/service-card/src/service-card.ts`
- Modify: index.ts for each component (add Props export)

**Interfaces:**
- Consumes: existing `@property()` declarations in each component
- Produces: `SlaIndicatorProps`, `ExecutionMonitorProps`,
  `KpiMetricRowProps`, `ApprovalGateProps`, `ServiceCardProps`
  — exported from each component's index.ts

For each component, extract the public `@property()` declarations into
a named Props interface. Place the interface immediately above the class
declaration. Export it from the package's index.ts.

**Pattern (shown for sla-indicator, repeat for each):**

- [ ] **Step 1: Extract SlaIndicatorProps**

Add to `components/sla-indicator/src/sla-indicator.ts` above the
`@customElement` decorator:

```typescript
export interface SlaIndicatorProps {
  deadline: string;
  slaWindow: number | null;
  warningThreshold: number;
  criticalThreshold: number;
  escalationStage: string | null;
  compact: boolean;
}
```

Export from the component's index.ts:
```typescript
export type { SlaIndicatorProps } from './sla-indicator.js';
```

- [ ] **Step 2: Extract ExecutionMonitorProps**

Read `components/execution-monitor/src/execution-monitor.ts:19-26`
for the `@property()` declarations. Create the interface filtering out
render callbacks (`renderAgent`, `renderModel`) and keeping only
YAML-meaningful properties:

```typescript
export interface ExecutionMonitorProps {
  endpoint?: string;
  executionId?: string;
  data?: ExecutionSnapshot;
  selectionTopic?: string;
  staleThresholdMs: number;
}
```

Export from index.ts.

- [ ] **Step 3: Extract KpiMetricRowProps**

Read `components/kpi-metric-row/src/kpi-metric-row.ts` for `@property()`
declarations. Create interface excluding render callbacks:

```typescript
export interface KpiMetricRowProps {
  endpoint?: string;
  density: 'comfortable' | 'compact' | 'dense';
  selectionTopic?: string;
  metrics?: MetricDefinition[];
  pushUrl?: string;
  pushTopics: string[];
}
```

Export from index.ts.

- [ ] **Step 4: Extract ApprovalGateProps**

Read `components/approval-gate/src/approval-gate.ts` for `@property()`
declarations. Create interface:

```typescript
export interface ApprovalGateProps {
  endpoint?: string;
  gateId?: string;
  // ... extract all @property() declarations from the file
}
```

Read the actual file to get the complete property list. Export from
index.ts.

- [ ] **Step 5: Extract ServiceCardProps**

Read `components/service-card/src/service-card.ts` for `@property()`
declarations. Create interface. Export from index.ts.

- [ ] **Step 6: Verify typecheck passes**

Run: `yarn typecheck`
Expected: PASS — no new type errors

- [ ] **Step 7: Commit**

```bash
git add components/sla-indicator/ components/execution-monitor/ \
  components/kpi-metric-row/ components/approval-gate/ components/service-card/
git commit -m "feat(#156): extract Props interfaces for initial 5 components Refs #156"
```

### Task 3: Generate schemas + staleness test

**Files:**
- Create: `packages/blocks-ui-schema/src/component-schemas.generated.ts` (via generator)
- Create: `packages/blocks-ui-schema/src/component-schemas.test.ts`

**Interfaces:**
- Consumes: `BlocksComponentRegistry` from `src/registry.ts`, Props
  interfaces from all 5 initial component packages
- Produces: per-component Zod schemas (`slaIndicatorPropsSchema`, etc.),
  `blocksComponentSchemaMap`, staleness test

- [ ] **Step 1: Run the generator**

Run: `yarn workspace @casehubio/blocks-ui-schema run generate`
Expected: `Generated 5 schemas to .../component-schemas.generated.ts`

Inspect the output file. Verify:
- 5 exported schemas
- No function-typed properties in schemas
- String literal unions mapped to `z.enum()`
- Optional properties have `.optional()`
- `blocksComponentSchemaMap` has 5 entries

- [ ] **Step 2: Write the staleness + parse test**

Create `packages/blocks-ui-schema/src/component-schemas.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { readFileSync, existsSync } from "fs";
import { execSync } from "child_process";
import { resolve } from "path";

describe("schema generator", () => {
  const generatedPath = resolve(__dirname, "component-schemas.generated.ts");

  it("generated file exists", () => {
    expect(existsSync(generatedPath)).toBe(true);
  });

  it("generated file has AUTO-GENERATED header", () => {
    const content = readFileSync(generatedPath, "utf-8");
    expect(content).toContain("AUTO-GENERATED");
  });

  it("exports a schema for every BlocksComponentRegistry entry", () => {
    const content = readFileSync(generatedPath, "utf-8");
    const expectedSchemas = [
      "slaIndicatorPropsSchema",
      "executionMonitorPropsSchema",
      "kpiMetricRowPropsSchema",
      "approvalGatePropsSchema",
      "serviceCardPropsSchema",
    ];
    for (const name of expectedSchemas) {
      expect(content).toContain(`export const ${name}`);
    }
  });

  it("does not contain function-typed properties", () => {
    const content = readFileSync(generatedPath, "utf-8");
    expect(content).not.toContain("renderAgent:");
    expect(content).not.toContain("renderModel:");
  });

  it("exports blocksComponentSchemaMap", () => {
    const content = readFileSync(generatedPath, "utf-8");
    expect(content).toContain("export const blocksComponentSchemaMap");
  });

  it("generated schemas parse valid sla-indicator data", async () => {
    const { slaIndicatorPropsSchema } = await import(
      "./component-schemas.generated.js"
    );
    const result = slaIndicatorPropsSchema.safeParse({
      deadline: "2026-12-31T23:59:59Z",
      warningThreshold: 0.25,
      criticalThreshold: 0.10,
      compact: true,
    });
    expect(result.success).toBe(true);
  });

  it("generated schemas reject unknown properties in strict mode", async () => {
    const { slaIndicatorPropsSchema } = await import(
      "./component-schemas.generated.js"
    );
    const result = slaIndicatorPropsSchema.strict().safeParse({
      deadline: "2026-12-31T23:59:59Z",
      unknownProp: "should fail",
    });
    expect(result.success).toBe(false);
  });

  it("generated file is not stale", () => {
    const current = readFileSync(generatedPath, "utf-8");
    execSync(
      "yarn workspace @casehubio/blocks-ui-schema run generate",
      { stdio: "pipe" },
    );
    const regenerated = readFileSync(generatedPath, "utf-8");
    expect(regenerated).toBe(current);
  });
});
```

- [ ] **Step 3: Build the package**

Run: `yarn workspace @casehubio/blocks-ui-schema run build`
Expected: generates schemas then compiles to dist/

- [ ] **Step 4: Run the tests**

Run: `yarn workspace @casehubio/blocks-ui-schema run test`
Expected: all tests PASS

- [ ] **Step 5: Run full repo typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/blocks-ui-schema/
git commit -m "feat(#156): generate Zod schemas for 5 initial components with staleness test Refs #156"
```

---

## Batch 2: Full component coverage

### Task 4: Extract Props interfaces for all remaining components

**Files:**
- Modify: every component's main source file (add Props interface)
- Modify: every component's index.ts (add Props export)
- Modify: `packages/blocks-ui-schema/src/registry.ts` (add all entries)
- Modify: `packages/blocks-ui-schema/package.json` (add devDependencies)

**Interfaces:**
- Consumes: `@property()` declarations from each component
- Produces: `FooProps` interface for each component, complete
  `BlocksComponentRegistry`

For each component below, apply the same pattern as Task 2:
1. Read the component's main .ts file
2. Extract all public `@property()` declarations into a `FooProps` interface
3. Filter out render callbacks and internal state
4. Export the interface from the package's index.ts
5. Add an entry to `BlocksComponentRegistry` in registry.ts
6. Add the package as a devDependency in blocks-ui-schema/package.json

**Complete component list** (grouped by package — main element per package):

**Simple data display:**
- `components/sla-breach-policy/src/sla-breach-policy.ts` → `SlaBreachPolicyProps`
- `components/trust-feedback-display/src/trust-feedback-display.ts` → `TrustFeedbackDisplayProps`
- `components/similarity-panel/src/similarity-panel.ts` → `SimilarityPanelProps`
- `components/compliance-summary/src/compliance-summary.ts` → `ComplianceSummaryProps`
- `components/gdpr-erasure-action/src/gdpr-erasure-action.ts` → `GdprErasureActionProps`
- `components/work-item-row/src/work-item-row.ts` → `WorkItemRowProps`
- `components/dimension-dashboard/src/dimension-dashboard.ts` → `DimensionDashboardProps`
- `components/routing-rationale/src/routing-rationale.ts` → `RoutingRationaleProps`

**Trust / scoring:**
- `components/trust-score-panel/src/trust-score-panel.ts` → `TrustScorePanelProps`

**Data views:**
- `components/grouped-data-view/src/grouped-data-view.ts` → `GroupedDataViewProps`
- `components/audit-trail-viewer/src/audit-trail-viewer.ts` → `AuditTrailViewerProps`
- `components/event-trail/` → check if it has source; if wrapper-only, skip

**List / detail / split:**
- `components/list-pane/src/list-pane.ts` → `ListPaneProps`
- `components/detail-pane/src/detail-pane.ts` → `DetailPaneProps`

**Work item family:**
- `components/work-item-inbox/src/work-item-inbox.ts` → `WorkItemInboxProps`
- `components/work-item-detail/src/work-item-detail.ts` → `WorkItemDetailProps`
- `components/work-item-workbench/src/work-item-workbench.ts` → `WorkItemWorkbenchProps`
- `components/worker-task-pane/src/worker-task-pane.ts` → `WorkerTaskPaneProps`

**Notification family:**
- `components/notification-inbox/src/notification-inbox.ts` → `NotificationInboxProps`
- `components/notification-inbox/src/notification-bell.ts` → `NotificationBellProps`
- `components/notification-inbox/src/subscription-editor.ts` → `SubscriptionEditorProps`
- `components/notification-inbox/src/notification-preferences.ts` → `NotificationPreferencesProps`

**Channel / conversation:**
- `components/channel-activity/src/blocks-channel-activity.ts` → `ChannelActivityProps`
- `components/conversation-viewer/src/blocks-convergence-indicator.ts` → `ConvergenceIndicatorProps`
- `components/conversation-viewer/src/blocks-common-ground-panel.ts` → `CommonGroundPanelProps`
- `components/conversation-viewer/src/blocks-conversation-workbench.ts` → `ConversationWorkbenchProps`
- `components/conversation-viewer/src/blocks-point-list.ts` → `PointListProps`
- `components/conversation-viewer/src/blocks-point-detail.ts` → `PointDetailProps`

**Case / explorer:**
- `components/case-explorer/src/case-explorer.ts` → `CaseExplorerProps`
- `components/case-explorer/src/entity-list.ts` → `EntityListProps`
- `components/case-explorer/src/entity-detail.ts` → `EntityDetailProps`
- `components/case-explorer/src/entity-tree.ts` → `EntityTreeProps`

**Timeline:**
- `components/blocks-timeline/src/blocks-timeline.ts` → `BlocksTimelineProps`

**Orchestration / execution:**
- `components/orchestration-workbench/src/orchestration-workbench.ts` → `OrchestrationWorkbenchProps`
- `components/blocks-dag-viewer/src/blocks-dag-viewer.ts` → `DagViewerProps`
- `components/blocks-decomposition-tree/src/blocks-decomposition-tree.ts` → `DecompositionTreeProps`
- `components/blocks-plan-item-tree/src/blocks-plan-item-tree.ts` → `PlanItemTreeProps`
- `components/blocks-plan-model-dashboard/src/blocks-plan-model-dashboard.ts` → `PlanModelDashboardProps`

**Diagrams:**
- `components/case-flow-viewer/src/blocks-case-flow-viewer.ts` → `CaseFlowViewerProps`
- `components/case-dependency-graph/src/blocks-case-dependency-graph.ts` → `CaseDependencyGraphProps`
- `components/diagram-workbench/src/diagram-workbench.ts` → `DiagramWorkbenchProps`
- `components/casehub-diagram/src/casehub-diagram.ts` → `CasehubDiagramProps`
- `components/swf-diagram/src/swf-diagram.ts` → `SwfDiagramProps`
- `components/htn-diagram/src/htn-diagram.ts` → `HtnDiagramProps`

**Infrastructure / ops:**
- `components/cluster-panel/src/cluster-panel.ts` → `ClusterPanelProps`
- `components/reconciliation-status/src/reconciliation-status.ts` → `ReconciliationStatusProps`
- `components/topology-viewer/src/topology-viewer.ts` → `TopologyViewerProps`

**Session / preferences:**
- `components/session-list/src/session-list.ts` → `SessionListProps`
- `components/session-detail/src/session-detail.ts` → `SessionDetailProps`
- `components/session-workbench/src/session-workbench.ts` → `SessionWorkbenchProps`
- `components/preferences-editor/src/preferences-editor.ts` → `PreferencesEditorProps`

**Workbenches:**
- `components/trust-workbench/src/trust-workbench.ts` → `TrustWorkbenchProps`
- `components/contributor-workbench/src/contributor-workbench.ts` → `ContributorWorkbenchProps`

**Commitment:**
- `components/commitment-viz/src/commitment-range-bar.ts` → `CommitmentRangeBarProps`
- `components/commitment-viz/src/commitment-transition-badge.ts` → `CommitmentTransitionBadgeProps`

**Document workbench** (non-blocks-prefixed sub-components — include if
they are independently usable in YAML, skip if internal-only):
- Check each: `brainstorm-options`, `debate-feed`, `document-timeline`,
  `review-tracker`, `selection-threads`, `document-diff`

- [ ] **Step 1: Read each component file, extract Props interface**

For each component in the list above:
1. Read the main .ts file
2. Identify all `@property()` declarations
3. Create a `FooProps` interface above the class with those properties
4. Filter out render callbacks (any property whose type has call signatures)
5. Export the type from the component's index.ts

- [ ] **Step 2: Update registry.ts with all entries**

Add `import type` for each new Props interface and add the corresponding
entry to `BlocksComponentRegistry`.

- [ ] **Step 3: Update blocks-ui-schema/package.json devDependencies**

Add `workspace:*` devDependency for each new component package.

- [ ] **Step 4: Run `yarn install`**

Run: `yarn install`
Expected: clean install with all workspace links

- [ ] **Step 5: Regenerate schemas**

Run: `yarn workspace @casehubio/blocks-ui-schema run generate`
Expected: `Generated N schemas` (N = total component count)

- [ ] **Step 6: Verify typecheck**

Run: `yarn typecheck`
Expected: PASS

- [ ] **Step 7: Update test expectations**

Update `component-schemas.test.ts` — expand the `expectedSchemas` array
to include all new schema names. Add a parse test for at least one
complex component (e.g. `channelActivityPropsSchema`).

- [ ] **Step 8: Run tests**

Run: `yarn workspace @casehubio/blocks-ui-schema run test`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git add components/ packages/blocks-ui-schema/ yarn.lock
git commit -m "feat(#156): extract Props interfaces for all components, full schema generation Refs #156"
```

---

## Batch 3: configure() cleanup + completeness test

### Task 5: Fix untyped configure() methods

**Files (15 component files with `configure(props: Record<string, unknown>)`):**
- Modify: `components/blocks-timeline/src/blocks-timeline.ts:7`
- Modify: `components/detail-pane/src/detail-pane.ts:16`
- Modify: `components/grouped-data-view/src/grouped-data-view.ts:140`
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.ts:133`
- Modify: `components/work-item-detail/src/work-item-detail.ts:444`
- Modify: `components/orchestration-workbench/src/orchestration-workbench.ts:49`
- Modify: `components/execution-monitor/src/execution-monitor.ts:186`
- Modify: `components/diagram-workbench/src/diagram-workbench.ts:74`
- Modify: `components/channel-activity/src/blocks-channel-activity.ts:107`
- Modify: `components/document-workbench/src/brainstorm-options.ts:19`
- Modify: `components/document-workbench/src/debate-feed.ts:47`
- Modify: `components/document-workbench/src/document-timeline.ts:17`
- Modify: `components/document-workbench/src/review-tracker.ts:59`
- Modify: `components/document-workbench/src/document-diff.ts:343`
- Modify: `components/document-workbench/src/selection-threads.ts:26`

**Interfaces:**
- Consumes: `FooProps` interface from Task 2/4 for each component
- Produces: typed `configure()` methods with `Partial<FooProps>` parameter

For each file, change:
```typescript
// Before
configure(props: Record<string, unknown>): void {
  if (props.endpoint !== undefined) this.endpoint = props.endpoint as string;
  if (props.executionId !== undefined) this.executionId = props.executionId as string;
}

// After
configure(props: Partial<ExecutionMonitorProps>): void {
  if (props.endpoint !== undefined) this.endpoint = props.endpoint;
  if (props.executionId !== undefined) this.executionId = props.executionId;
}
```

The `as` casts are removed — TypeScript infers the correct types from the
Props interface.

- [ ] **Step 1: Write a failing test for typed configure**

Add a type-level test in `packages/blocks-ui-schema/src/component-schemas.test.ts`:

```typescript
it("configure() rejects undeclared properties at type level", () => {
  // This is a compile-time check — if configure() still accepts
  // Record<string, unknown>, the TypeScript compiler would allow
  // any property. The Props interface constrains it.
  // Runtime verification: pick a component with configure()
  // and verify it only reads declared properties.
  const content = readFileSync(
    resolve(__dirname, "../../components/execution-monitor/src/execution-monitor.ts"),
    "utf-8",
  );
  expect(content).not.toContain("Record<string, unknown>");
  expect(content).toContain("Partial<ExecutionMonitorProps>");
});
```

- [ ] **Step 2: Fix all 15 configure() methods**

For each file in the list above:
1. Import the component's Props interface
2. Change `props: Record<string, unknown>` to `props: Partial<FooProps>`
3. Remove `as` type casts from property assignments
4. Verify the component's `configure()` only accesses properties that
   exist in the Props interface

- [ ] **Step 3: Verify typecheck**

Run: `yarn typecheck`
Expected: PASS — compiler validates all configure() bodies

- [ ] **Step 4: Run all tests**

Run: `yarn test`
Expected: all tests PASS across all packages

- [ ] **Step 5: Commit**

```bash
git add components/ packages/blocks-ui-schema/
git commit -m "feat(#156): replace untyped configure() with Partial<Props> in 15 components Refs #156"
```

### Task 6: Registry completeness test

**Files:**
- Create: `packages/blocks-ui-schema/src/registry-completeness.test.ts`

**Interfaces:**
- Consumes: `blocksComponentSchemaMap` from generated schemas
- Produces: test that catches new components missing from the registry

- [ ] **Step 1: Write the completeness test**

```typescript
import { describe, it, expect } from "vitest";
import { execSync } from "child_process";

describe("registry completeness", () => {
  it("every blocks-* custom element has a registry entry", () => {
    const output = execSync(
      'grep -rn "@customElement" components/*/src/*.ts packages/*/src/**/*.ts 2>/dev/null || true',
      { encoding: "utf-8" },
    );

    const elementNames = new Set<string>();
    for (const line of output.split("\n")) {
      const match = line.match(/@customElement\(['"]([^'"]+)['"]\)/);
      if (match) elementNames.add(match[1]);
    }

    const { blocksComponentSchemaMap } = require("./component-schemas.generated.js");
    const registeredNames = new Set(blocksComponentSchemaMap.keys());

    const missing = [...elementNames].filter(
      name => name.startsWith("blocks-") && !registeredNames.has(name),
    );

    expect(missing).toEqual([]);
  });
});
```

Note: this test only checks `blocks-*` prefixed elements. Non-prefixed
elements (commitment-viz sub-components, document-workbench internals)
are excluded — they are internal and not part of the YAML-facing API.

- [ ] **Step 2: Run the test**

Run: `yarn workspace @casehubio/blocks-ui-schema run test`
Expected: PASS

- [ ] **Step 3: Run full test suite**

Run: `yarn test`
Expected: all packages PASS

- [ ] **Step 4: Final build**

Run: `yarn build`
Expected: full topological build succeeds

- [ ] **Step 5: Commit**

```bash
git add packages/blocks-ui-schema/
git commit -m "feat(#156): add registry completeness test Refs #156"
```

---

## References

- [2026-09-07-zod-schema-generation-design.md] — design spec this plan implements
- [casehub-pages packages/pages-schema/scripts/generate-schemas.ts] — reference generator
- [casehub-pages packages/pages-schema/src/generator.test.ts] — reference staleness test
- [casehub-pages packages/pages-component/src/model/type-guards.ts] — ComponentTypeRegistry pattern
- [components/sla-indicator/src/sla-indicator.ts:59-65] — clean @property() example
- [components/execution-monitor/src/execution-monitor.ts:186] — untyped configure() example
- [GitHub #156] — focal issue
