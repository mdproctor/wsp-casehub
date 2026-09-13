## D1: Component discovery mechanism

**Choice:** Create a `BlocksComponentRegistry` interface in blocks-ui-core that maps element names to prop interfaces, mirroring the pages `ComponentTypeRegistry` pattern
**Alternatives:**
- Scan `@customElement` decorators and extract `@property()` declarations directly — avoids a separate registry but couples the generator to Lit decorator internals and is more brittle
**Rationale:** Single source of truth for the component surface area. Matches the proven pages pattern exactly, making the generator simpler and the two systems consistent. Forces each component to declare a clean Props interface.
**Trade-offs:** Every new component must add an entry to the registry manually — a small maintenance cost
**Sources:** pages `ComponentTypeRegistry` at `packages/pages-component/src/model/type-guards.ts`, pages generator at `packages/pages-schema/scripts/generate-schemas.ts`
**Exploration:** quick
**Status:** captured

## D2: Schema package location

**Choice:** New `packages/blocks-ui-schema` package, mirroring pages-schema structure
**Alternatives:**
- Inside `packages/blocks-ui-core` — fewer packages but mixes domain types with generated schemas and adds ts-morph dev dependency to core
**Rationale:** Clean separation of concerns. Generator machinery (ts-morph, code generation) stays out of the core package. Mirrors pages-schema for cross-repo consistency.
**Trade-offs:** One more package in the workspace to maintain
**Sources:** pages `packages/pages-schema/package.json`
**Exploration:** quick
**Depends on:** D1 (registry interface lives in blocks-ui-core, schemas package reads from it)
**Status:** captured

## D3: Fix untyped configure() methods

**Choice:** Fix the ~16 untyped `configure(props: Record<string, unknown>)` methods as part of this issue — replace with typed Props interface from the registry
**Alternatives:**
- Defer to a separate issue — schemas would still be correct (generated from registry, not configure), but runtime enforcement gap remains
**Rationale:** Mechanical fix that completes the audit task from the issue. Ensures runtime behaviour matches what schemas declare. Leaving untyped configure() creates a gap where undeclared properties still pass through at runtime.
**Trade-offs:** Increases scope slightly (~16 files to touch), but each change is a simple type annotation + cast removal
**Sources:** execution-monitor.ts:186 `configure(props: Record<string, unknown>)`, audit finding of ~16 components with this pattern
**Exploration:** quick
**Depends on:** D1 (Props interfaces must exist before configure() can reference them)
**Status:** captured

## D4: Domain type and callback handling

**Choice:** Filter out function/callback properties, generate inline object schemas for domain types via ts-morph structural walking
**Alternatives:**
- Hand-write Zod schemas for each domain type in blocks-ui-core — explicit control but high maintenance burden for types ts-morph can derive automatically
**Rationale:** Render callbacks (renderAgent, renderModel, renderCandidate) are runtime-only and meaningless in YAML. Domain types like ExecutionSnapshot and TabDefinition are object types that ts-morph can walk structurally. Pages generator already has depth guard (depth > 6 → z.unknown()) for safety. Only add hand-written schemas if recursion or branded types surface.
**Trade-offs:** Generated schemas for complex domain types may be verbose; deeply nested types fall back to z.unknown() at depth 6
**Sources:** pages generator `typeToZod()` and `isFunction()` at `packages/pages-schema/scripts/generate-schemas.ts`, pages `FUNCTION_PROPS` set
**Exploration:** quick
**Depends on:** D2 (generator lives in blocks-ui-schema)
**Status:** captured

## D5: Schema consumption and distribution

**Choice:** Publish blocks-ui-schema as a Maven SNAPSHOT artifact (WebJar pattern) exporting Zod schemas. Pages-code-editor imports the schema map and merges with pages-schema's map.
**Alternatives:**
- Export a JSON schema manifest — more portable but loses Zod runtime validation for desugarers, and blocks-ui already uses the Maven SNAPSHOT WebJar pattern
**Rationale:** Consistent with existing blocks-ui distribution. Zod schemas serve both validation (desugarers strip undeclared props) and completion (editor knows valid properties) from one artifact. No new distribution mechanism needed.
**Trade-offs:** Downstream consumers must add blocks-ui-schema as a dependency to get completion for blocks-ui components
**Sources:** CLAUDE.md Frontend Dependencies section (Maven SNAPSHOT WebJar pattern), pages-code-editor schema integration
**Exploration:** quick
**Depends on:** D2 (schemas package must exist), D4 (generator produces the schemas)
**Status:** captured
