# Auto-generate Zod schemas for blocks-ui components

Issue: casehubio/blocks-ui#156
Date: 2026-09-07

## Problem

blocks-ui components are embedded in pages YAML, but there is no schema
validation or editor completion for their properties. ~16 components have
untyped `configure(props: Record<string, unknown>)` methods that let
undeclared properties through. There is no central registry of what
components exist or what they accept.

## Design

Three layers, mirroring the casehub-pages #411 pattern:

### 1. BlocksComponentRegistry (blocks-ui-schema)

A TypeScript interface in the schema package mapping element tag names to
their Props interfaces. Lives in blocks-ui-schema (not blocks-ui-core)
to avoid circular dependencies — component packages depend on core for
domain types, so core cannot depend back on them for Props types:

```typescript
export interface BlocksComponentRegistry {
  'blocks-sla-indicator': SlaIndicatorProps;
  'blocks-execution-monitor': ExecutionMonitorProps;
  'blocks-kpi-metric-row': KpiMetricRowProps;
  // ... all ~50 components
}
```

Each component package exports a `FooProps` interface extracted from its
public `@property()` declarations. The registry aggregates them via
`import type` — the component packages are devDependencies of
blocks-ui-schema, used only for type resolution by ts-morph at
generation time.

**What goes in Props:**
- All public `@property()` declarations (the YAML-facing contract)
- Inherited mixin properties that are part of the YAML API (e.g.
  `endpoint` from DataSourceMixin, `pushUrl`/`pushTopics` from PushMixin)

**What stays out:**
- `@state()` private fields
- Render callbacks (`renderAgent`, `renderModel`, `renderCandidate`,
  `renderDetail`, `getRowDetail`, etc.) — runtime-only, meaningless in YAML
- Internal mixin state (e.g. `pushConnected` from PushMixin)

### 2. blocks-ui-schema package (new)

New package at `packages/blocks-ui-schema/`:

```
packages/blocks-ui-schema/
  package.json
  tsconfig.build.json
  scripts/
    generate-schemas.ts      # ts-morph generator
  src/
    registry.ts                     # BlocksComponentRegistry interface
    component-schemas.generated.ts  # generated output
    component-schemas.test.ts       # staleness + parse tests
    index.ts                        # re-exports
```

**package.json:**
- `dependencies`: zod, @casehubio/pages-data (for lookupSchema if needed)
- `devDependencies`: @casehubio/blocks-ui-core (read-only, for type resolution), ts-morph, tsx
- `scripts.generate`: `tsx scripts/generate-schemas.ts`
- `scripts.build`: `yarn generate && tsc -p tsconfig.build.json`

**Generated output** exports:
- Per-component schema: `export const slaIndicatorPropsSchema = z.object({...})`
- Schema map: `export const blocksComponentSchemaMap: ReadonlyMap<string, z.ZodType>`

### 3. Generator (scripts/generate-schemas.ts)

Fork of the pages generator adapted for blocks-ui:

**Entry point:** Uses ts-morph to load `BlocksComponentRegistry` from
blocks-ui-core and iterate its properties, same as pages reads
`ComponentTypeRegistry`.

**Type mapping (`typeToZod`):**
- Primitives: string → `z.string()`, number → `z.number()`, boolean → `z.boolean()`
- String literal unions → `z.enum([...])`
- Union types → `z.union([...])`
- Arrays → `z.array(...)`
- Records → `z.record(...)`
- Object types → `z.object({...})` with recursive property walking
- Depth guard: `depth > 6 → z.unknown()`

**Function filtering:**
- `isFunction()` check on type (call signatures > 0)
- Explicit `FUNCTION_PROPS` set for blocks-ui render callbacks

**Domain types:** ExecutionSnapshot, TabDefinition, AgentRef, etc. are
object types that ts-morph walks structurally. No hand-written schemas
unless recursion or branded types surface during implementation.

**Schema naming:** derived from TypeScript type name — `SlaIndicatorProps`
→ `slaIndicatorPropsSchema`.

### 4. Props interface extraction (audit)

For each of the ~50 components:
1. Extract public `@property()` declarations into a `FooProps` interface
2. Export the interface from the component package
3. Import into blocks-ui-core and add to `BlocksComponentRegistry`

Components with clean typed properties (~30) need only the interface
extraction. Components are already well-typed at the `@property()` level;
the interfaces formalise what already exists.

### 5. configure() cleanup

For the ~16 components with `configure(props: Record<string, unknown>)`:
- Change parameter type to `Partial<FooProps>`
- Remove manual `as` casts (TypeScript infers the types)
- The compiler enforces only declared properties are accessed

### 6. Distribution

Published as a Maven SNAPSHOT artifact (WebJar pattern), consistent with
how blocks-ui already distributes its components. Downstream consumers
(pages-code-editor, app desugarers) add blocks-ui-schema as a dependency
and merge `blocksComponentSchemaMap` with pages' `componentSchemaMap`.

## Testing

**Staleness test:** Re-runs the generator and asserts output matches the
committed file. Catches interface changes that weren't followed by
regeneration.

**Parse tests:** Representative schemas parse valid data and reject
unknown properties in strict mode. Covers at least one simple component
(sla-indicator), one complex component (execution-monitor), and one
component with domain types (channel-activity).

**Registry completeness test:** Scans all `@customElement('blocks-*')`
declarations in the repo and asserts each has a corresponding entry in
`BlocksComponentRegistry`. Catches new components that forgot to register.

**Build chain integration:** `yarn generate` runs before `tsc` in the
blocks-ui-schema build script. The root `yarn build` (topological) builds
blocks-ui-core first (registry interface), then blocks-ui-schema
(generation), then components.

## Scope boundary

**In scope:**
- BlocksComponentRegistry interface
- blocks-ui-schema package with generator, generated output, tests
- Props interface extraction for all ~50 components
- configure() type fixes for ~16 components
- Staleness, parse, and completeness tests

**Out of scope:**
- Wiring schemas into pages-code-editor completion (downstream consumer work)
- Wiring schemas into desugarers for property stripping (downstream consumer work)
- Schema generation for pages components (casehub-pages #411, separate repo)

## References

- casehub-pages `packages/pages-schema/scripts/generate-schemas.ts` — reference generator
- casehub-pages `packages/pages-schema/src/component-schemas.generated.ts` — reference output
- casehub-pages `packages/pages-schema/src/generator.test.ts` — reference staleness test
- casehub-pages `packages/pages-ui/src/parser/displayer-desugar.ts` — schema-aware property filter
- casehub-pages `packages/pages-component/src/model/type-guards.ts` — ComponentTypeRegistry
- blocks-ui `components/execution-monitor/src/execution-monitor.ts:186` — untyped configure() example
- blocks-ui `components/sla-indicator/src/sla-indicator.ts:59-65` — clean @property() example
