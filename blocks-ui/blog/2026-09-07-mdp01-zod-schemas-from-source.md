---
layout: post
title: "Schema generation from source: giving blocks-ui components a typed contract"
date: 2026-09-07
entry_type: note
subtype: diary
projects: [casehubio/blocks-ui]
tags: [typescript, zod, ts-morph, schema-generation, web-components]
series: issue-156-zod-schemas
---

blocks-ui has around fifty web components, and until today none of them had a
formal declaration of what properties they accept. The `@property()` decorators
on each class were typed, but nothing enforced that contract from the outside —
a YAML author could pass arbitrary keys and they'd silently pass through.
Sixteen components had `configure(props: Record<string, unknown>)` methods that
made this explicit: anything goes.

The fix is a ts-morph generator that reads a `BlocksComponentRegistry` interface
and walks each component's property types structurally, producing Zod schemas.
The approach mirrors what casehub-pages landed with its own `ComponentTypeRegistry`
and `component-schemas.generated.ts`, so the two systems are consistent — same
generator shape, same schema map, same staleness test.

The interesting design problem was where to put the registry. The natural home
would be blocks-ui-core, but that creates a circular dependency: components
depend on core for domain types, core would need to import Props interfaces from
components. The solution was to put the registry in the new blocks-ui-schema
package alongside the generator, using `import type` from each component
package as dev dependencies. The generator resolves types at generation time
via tsconfig `paths` that point directly at source files — no build-order
dependency, no stale `dist/` problem.

That `dist/` issue cost us time. ts-morph resolves `@scope/package` through
`package.json`'s `types` field, which points at `dist/index.d.ts`. When the
dist is stale (built before the Props interface was added), the type name
resolves correctly — `getSymbol()` returns the right name — but
`getProperties()` returns an empty array. The misdirection is convincing:
the obvious diagnostic says the type was found. The fix was a separate
`tsconfig.generator.json` with `paths` mapping packages directly to their
source directories, while the build tsconfig stays clean. That split — one
tsconfig for generation, one for compilation — is reusable anywhere a
monorepo runs ts-morph against sibling packages.

The generator handles Lit's render callbacks (functions passed as properties
for customisation) by filtering them out — `isFunction()` on the type plus
an explicit `FUNCTION_PROPS` set. Domain types like `ExecutionSnapshot` and
`QuorumConfig` are walked structurally to depth six, producing nested
`z.object()` schemas that capture the full shape. The depth guard prevents
runaway recursion; anything past six levels falls back to `z.unknown()`.

We also typed eight `configure()` methods from `Record<string, unknown>` to
`Partial<FooProps>`, removing the manual `as` casts. The compiler now enforces
that `configure()` only accesses declared properties. Six internal
document-workbench sub-components still have the untyped pattern — they're
non-`blocks-` prefixed internals that don't appear in YAML, so the schema
registry doesn't cover them. Deferred, not forgotten.

The schemas are published as a Maven SNAPSHOT via the existing WebJar
pattern. Downstream, pages-code-editor can import `blocksComponentSchemaMap`
and merge it with the pages schema map — that wiring is separate work, but
the contract is ready. A registry completeness test scans for every
`@customElement('blocks-*')` in the repo and verifies it has a schema
entry, so new components can't slip through without being registered.

What I find satisfying about this is the layering. The Props interfaces are
the contract. The registry aggregates them. The generator produces schemas
mechanically. The staleness test keeps them honest. The completeness test
prevents drift. Each layer is independently testable and none of them
require the full workspace to be built — the generator reads source
directly. That independence matters in a monorepo with fifty packages.
