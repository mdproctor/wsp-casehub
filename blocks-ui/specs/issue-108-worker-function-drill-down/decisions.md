# Decisions — Issue #108: Worker Function Drill-Down

## D1: Function type indicator style on worker stencil

**Choice:** Subtle badge — small coloured pill next to worker name
**Alternatives:**
- Icon + badge — more scannable but takes more space
- Icon only — compact but relies on tooltips, less legible
**Rationale:** Consistent with StatusBadge pattern used elsewhere in blocks-ui. Minimal visual disruption to existing stencil layout.
**Trade-offs:** Less scannable than icons at a glance; badge text must be short.
**Exploration:** quick
**Status:** captured

## D2: Function type switching in property panel

**Choice:** Dropdown selector — mirrors the binding target-type pattern
**Alternatives:**
- Read-only detection — detect from YAML, no switching UI
- Dropdown + confirmation dialog — safer but adds friction for an easily undoable action
**Rationale:** Consistent with the binding properties pattern (capability/subCase/humanTask selector). The undo stack handles accidental type switches.
**Trade-offs:** Switching type destroys existing function config without confirmation (undo available).
**Exploration:** quick
**Status:** captured

## D3: Agent model provider selection UX

**Choice:** Provider dropdown + dynamic fields — select provider, render its specific fields below
**Alternatives:**
- Tabbed providers — more discoverable but implies multiple providers can coexist
- Flat fields with provider prefix — simpler but loses oneOf mutual-exclusion semantics
**Rationale:** Matches the discriminated-union pattern used elsewhere. Reuses existing field types (select, text, number). Switching provider clears old block and creates defaults.
**Trade-offs:** Two nested discriminators (function type → agent → provider) creates depth; manageable because provider fields are flat (3-5 fields each).
**Exploration:** quick
**Status:** captured
**Depends on:** D2 (function type selector)

## D4: Auth config sub-form sharing

**Choice:** Shared renderAuthConfig() function used by MCP-HTTP and A2A
**Alternatives:**
- Independent per type — simpler initially but duplicates auth UI
**Rationale:** Both MCP-HTTP and A2A use the identical AuthConfig shape (type + tokenConfigKey). Single source of truth reduces drift.
**Trade-offs:** Coupling between MCP and A2A rendering — acceptable because the engine enforces the shared AuthConfig type.
**Exploration:** quick
**Status:** captured

## D5: Sequence editor interaction

**Choice:** Add/remove + drag reorder
**Alternatives:**
- Add/remove only — simpler, order by insertion
- Textarea one-per-line — minimal code, reuses string-array, but no validation
**Rationale:** Sequence ordering is semantically meaningful (execution order). Drag reorder makes the ordering explicit and editable. Sequences are typically short (2-5 items).
**Trade-offs:** Drag-reorder is more implementation effort than textarea. Worth it for explicit ordering UX.
**Exploration:** quick
**Status:** captured

## D6: Package ownership

**Choice:** graph-stencil-case owns function type schemas, detection, and sub-form renderers
**Alternatives:**
- blocks-ui-core — makes types globally available but they're tightly coupled to case diagrams
- New graph-stencil-worker package — clean isolation but over-packages for what's part of the case stencil
**Rationale:** Function types are part of the case domain model. graph-stencil-case already owns worker rendering, YAML editing, and type generation. Keeps related concerns together.
**Trade-offs:** If another diagram type needs worker function types, they'd import from graph-stencil-case (acceptable per stencil isolation protocol since it's the owner, not a cross-import between peers).
**Exploration:** quick
**Status:** captured

## D7: Drill-down scope

**Choice:** Inline-only for agent/A2A/MCP/sequence; Flow keeps its existing SWF drill-down
**Alternatives:**
- Drill-down for agent too — better editing experience for complex prompt/model config
- Drill-down for all — maximum space but heavy infrastructure for simple forms
**Rationale:** Agent/A2A/MCP/Sequence configs are flat enough for a property panel section. Flow drill-down is justified because it opens an entire SWF diagram, not a form.
**Trade-offs:** Agent systemPrompt editing in a 300px-wide panel textarea may feel cramped. Acceptable for now; drill-down can be added later if needed.
**Exploration:** quick
**Status:** captured

## D8: Form integration architecture

**Choice:** Custom render section — core fields via renderPropertyForm, function-type sub-forms via dedicated renderers below
**Alternatives:**
- Extend property-form generics — add oneOf/discriminator/reorderable-list to generic form infrastructure
- Schema augmentation — merge type-specific sub-schema into Worker schema at runtime
**Rationale:** Function types need special UX (discriminated unions for provider/transport, drag-reorder for sequence) that the generic property form doesn't support. Custom renderers follow the same pattern as the binding target-type selector. Each function type has unique needs that are better served by dedicated code than generic infrastructure.
**Trade-offs:** Function type forms are not schema-driven — changes to engine types require manual UI updates. Acceptable because function types change infrequently and their UX is intentionally specialised.
**Exploration:** quick
**Depends on:** D2 (function type selector), D6 (package ownership)
**Status:** captured
