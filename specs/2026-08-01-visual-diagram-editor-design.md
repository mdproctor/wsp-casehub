# CaseHub Visual Diagram Editor — Design Spec

**Date:** 2026-08-01
**Status:** Draft
**Scope:** Visual viewer and editor for CaseHub case definitions, with SWF drill-down and runtime overlay

---

## 1. Problem

CaseHub case definitions are authored in YAML. There is no visual tool to view, navigate, or edit the structure of a case — its compounds, primitives, milestones, goals, sub-cases, bindings, or embedded SWF workflows. Users need:

- A **read-only viewer** that renders a case definition as a graph with auto-layout
- A **property editor** that lets users select a node and edit its properties in a side panel
- A **structural editor** that lets users add, remove, and replace nodes at specific points (auto-layout recalculates)
- A **runtime overlay** that projects live execution state (PlanItem status, milestone progression, heatmaps) onto the same graph
- A **multi-model view** that can drill into SWF workflows embedded inside Workers, rendering them in the same canvas
- **Pluggable work stencils** discovered at runtime from marketplace YAML definitions

Free-form drag-and-drop is explicitly deferred. Auto-layout and structural editing come first.

## 2. Decisions

### 2.1 YAML is the source of truth

The editor reads and writes YAML. There is no intermediate JSON graph model. The flow is:

```
YAML → parse (js-yaml) → domain objects → domain adapter → React Flow nodes/edges → ELK layout → render
Edit → mutate domain objects → serialize (js-yaml) → write YAML via persistence backend
```

### 2.2 React Flow + Lit bridge (now), @xyflow/lit (later)

The CaseHub frontend is Lit 3.x Web Components. React Flow is the most capable open-source graph editor library (MIT). The bridge pattern is straightforward:

- Skip Shadow DOM on the wrapper component (`createRenderRoot() { return this; }`)
- Mount React Flow via `ReactDOM.createRoot` in `connectedCallback`
- Pass Lit properties as React props; React callbacks emit Lit custom events
- ~50 lines of bridge code, ~45KB bundle overhead for React + ReactDOM

React is isolated to a single package (`graph-renderer`). When @xyflow/lit is built (or if the xyflow project ships one), the renderer swaps with no impact on the graph model, stencils, or domain adapters.

**Rationale against alternatives:**
- **@xyflow/system (headless core)**: Not viable standalone — it's a utility layer (25% of total logic), not a rendering framework. Building a Lit renderer on it = replicating Svelte Flow (~7,700 LOC).
- **Cytoscape.js**: Canvas-based rendering with no public custom node drawing API. Node rendering limited to CSS shapes and SVG-as-bitmap backgrounds. No stencil, palette, or editor UI.
- **GoJS / JointJS+**: Commercial licenses ($3,400–4,000/dev). Ruled out.
- **JointJS open-source**: MPL 2.0. Ruled out (must be MIT/BSD/Apache).
- **Lienzo port**: 6–10 person-weeks to port GWT → TypeScript. Strategic option for the future but premature now.

**SWF team alignment:** Same rendering framework as the SWF editor. Shared knowledge, same mental model for graph editing, potential for shared custom node utilities.

### 2.3 Two tiers of stencils

**Structural stencils (compile-time)** — the graph grammar. Fixed set per domain:
- Case: Compound, Primitive, Milestone, Goal, SubCase
- SWF: Call, Switch, Raise, Catch, Entry, Exit (drill-down view)

Each structural stencil defines:
- Containment rules ("Compound can contain: Primitive, Compound")
- Connection rules ("Goal has zero outgoing edges")
- Port cardinality ("Primitive has 0..* in, 0..1 out")
- Property schema (typed fields for the side panel editor)
- SVG rendering template (function of node state → visual)

**Work stencils (runtime-discoverable)** — the leaf vocabulary. What a Primitive actually does:
- Discovered from marketplace YAML at configurable URLs
- Grouped by category (connectors/messaging, ai/agents, human/tasks, etc.)
- Each defines: icon, property schema, sync/async, input/output contract
- Rendered as Primitive nodes with work-type-specific visuals

### 2.4 Persistence is pluggable

```typescript
interface PersistenceBackend {
  read(uri: string): Promise<string>;   // returns YAML
  write(uri: string, yaml: string): Promise<void>;
}
```

Backends: Git (read/write GitHub URLs), Electron (local filesystem), REST API, in-memory (playground). The editor doesn't know or care where the YAML lives.

### 2.5 All TypeScript is type-safe

- Types generated from the CaseDefinition JSON Schema (`engine/schema/src/main/resources/schema/CaseDefinition.yaml`)
- Type-safe overlays for all library integrations (React Flow, ELK, js-yaml)
- No `any` types. Strict mode throughout.

**Prerequisite:** Verify the JSON Schema is current against the Java domain model. The schema may be stale after the stages removal and recent model changes.

### 2.6 Runtime overlay — lightweight, not a separate view

Same graph, same layout. Runtime data is projected as decoration:
- Node state badges (RUNNING → green pulse, COMPLETED → checkmark, FAULTED → red)
- Heatmaps for usage frequency (hot/cold path colouring)
- Active compound highlighting (which compound the planning strategy is expanding)
- Worker execution indicators (SWF workflow progress inside a Primitive)
- Milestone progression (PENDING → ACTIVE → COMPLETED)
- Blackboard data preview on hover

The stencil rendering function takes optional runtime state:
```typescript
render(node: NodeDefinition, runtime?: RuntimeState): SVGTemplate
```

No runtime state → design mode. Runtime state present → overlay decorations.

## 3. Architecture

### 3.1 Package structure

```
Pages (framework-level, domain-agnostic)
│
├── @casehubio/graph-core
│   ├── Graph model (nodes, edges, containment tree)
│   ├── Stencil registry (register/lookup structural + work stencils)
│   ├── Constraint validator (containment rules, connection rules, cardinality)
│   ├── Persistence SPI (PersistenceBackend interface)
│   ├── Runtime overlay data model (RuntimeState, badges, heatmap data)
│   └── Edit operations (add, remove, replace node — validated against stencil rules)
│
├── @casehubio/graph-renderer
│   ├── React Flow bridge (Lit wrapper, React lifecycle management)
│   ├── ELK layout integration (hierarchical auto-layout with containment)
│   ├── Stencil → React Flow node type mapping
│   ├── SVG rendering pipeline (stencil templates → React Flow custom nodes)
│   ├── Property panel contract (selected node → property schema → form)
│   ├── Runtime overlay renderer (badge, heatmap, pulse decorations)
│   └── Mode switching (design / runtime)
│
└── @casehubio/graph-work-registry
    ├── Marketplace YAML loader (fetch + parse work stencil descriptors)
    ├── Work stencil schema (name, category, icon, properties, I/O contract)
    ├── Category index (grouped palette data)
    └── Property schema → form generation (typed form from YAML descriptor)

blocks-ui (domain-specific, consumes Pages)
│
├── @casehubio/graph-stencil-case
│   ├── Case domain adapter
│   │   ├── toGraph(caseYaml): GraphModel — parse YAML, produce nodes/edges/containment
│   │   └── applyEdit(model, edit): CaseYaml — apply structural edit, serialize back
│   ├── TypeScript types generated from CaseDefinition.yaml JSON Schema
│   ├── Structural stencils: Compound, Primitive, Milestone, Goal, SubCase
│   ├── SVG templates per node type (with state variants)
│   └── Case runtime overlay adapter (PlanItem states, milestone progression)
│
├── @casehubio/graph-stencil-swf
│   ├── SWF domain adapter (workflow YAML ↔ graph)
│   ├── Workflow step stencils (call, switch, raise, catch, entry, exit)
│   ├── SVG templates per step type
│   ├── Drill-down integration (expand SWF Worker → sub-graph of workflow steps)
│   └── SWF runtime overlay adapter (step execution state, lineage)
│
└── casehub-diagram (Lit wrapper component for blocks-ui)
    ├── Assembles graph-core + graph-renderer + stencil sets
    ├── Persistence backend configuration
    ├── Palette UI (structural stencils + work registry results)
    ├── Property panel UI (bound to selected node's stencil schema)
    ├── Toolbar (mode switch, zoom controls, layout reset)
    └── Design / runtime mode toggle
```

### 3.2 Data flow

```
                    ┌─────────────────────┐
                    │  Persistence Backend │
                    │  (Git/File/REST)     │
                    └──────────┬──────────┘
                               │ YAML string
                    ┌──────────▼──────────┐
                    │  Domain Adapter      │
                    │  (Case or SWF)       │
                    │  parse ↔ serialize   │
                    └──────────┬──────────┘
                               │ Domain objects
                    ┌──────────▼──────────┐
                    │  graph-core          │
                    │  GraphModel          │
                    │  (nodes, edges,      │
                    │   containment)       │
                    └──────────┬──────────┘
                               │ validated graph
                    ┌──────────▼──────────┐
                    │  graph-renderer      │
                    │  ELK layout          │
                    │  React Flow render   │
                    │  + runtime overlay   │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  casehub-diagram     │
                    │  (Lit component)     │
                    │  palette + panel +   │
                    │  toolbar             │
                    └─────────────────────┘
```

### 3.3 Stencil definition contract

```typescript
interface StructuralStencil {
  type: string;                           // e.g. "compound", "primitive", "milestone"
  label: string;
  icon: string;                           // icon identifier
  containment: {
    canContain: string[];                 // types this can contain ([] = leaf)
    canBeContainedBy: string[];           // types that can contain this
  };
  connections: {
    inbound: { min: number; max: number; allowedFrom: string[] };
    outbound: { min: number; max: number; allowedTo: string[] };
  };
  properties: PropertySchema;            // typed property definitions for side panel
  render: (node: GraphNode, runtime?: RuntimeState) => SVGTemplate;
}
```

```typescript
interface WorkStencil {
  name: string;                           // e.g. "send-email"
  displayName: string;
  category: string;                       // e.g. "connectors/messaging"
  icon: string;
  async: boolean;
  properties: PropertySchema;
  input: JsonSchema;
  output: JsonSchema;
  render: (node: GraphNode, runtime?: RuntimeState) => SVGTemplate;
}
```

### 3.4 Execution ownership boundary

The viewer must visually distinguish:

- **Case-controlled zone** — PlanItems on the agenda, compound expansion, milestone progression. The Case engine owns lifecycle.
- **Worker-controlled zone** — once a Primitive dispatches to a Worker, the Worker owns execution. If the Worker runs a SWF workflow, the workflow has its own state machine.
- **The boundary** — a Primitive PlanItem transitions RUNNING → DELEGATED when handed to a Worker. Visually: a containment border change (different stroke/fill for delegated nodes) or an explicit delegation indicator.

## 4. Implementation Order

### Phase 0 — Prerequisites

**Must complete before any editor work starts.**

| Task | Effort | Notes |
|------|--------|-------|
| Verify CaseDefinition.yaml JSON Schema against current Java model | S | Schema may be stale after stages removal |
| Generate TypeScript types from verified schema | S | Use json-schema-to-typescript or similar |
| Spike: React Flow + Lit bridge proof of concept | S | Validate the 50-line bridge pattern works with Pages design tokens |

### Phase 1 — Foundation (graph-core + graph-renderer)

**The domain-agnostic platform. No CaseHub-specific code.**

```
Epic 1A: graph-core                    Epic 1B: graph-renderer
├── Graph model (nodes, edges, tree)   ├── React Flow Lit bridge
├── Stencil registry                   ├── ELK layout integration
├── Constraint validator               ├── Custom node rendering pipeline
├── Edit operations (add/remove/       ├── Pan/zoom/select interaction
│   replace with validation)           ├── Node selection → event emission
└── Persistence SPI                    └── Basic toolbar (zoom, reset layout)
```

**1A and 1B can run in parallel** — graph-core is the data model, graph-renderer is the visual layer. They integrate at the end when the renderer consumes the graph model.

### Phase 2 — Case Stencil (read-only viewer)

**First visual output. Render a real case definition.**

```
Epic 2: graph-stencil-case (viewer)
├── Case domain adapter: toGraph() — YAML → graph model
├── Structural stencils with SVG templates
│   ├── Compound (with completion semantics badge: all/any/M-of-N)
│   ├── Primitive (with executor type indicator)
│   ├── Milestone (diamond shape, lifecycle state)
│   ├── Goal (terminal node, SUCCESS/FAILURE variant)
│   └── SubCase (recursive, with input/output mapping indicator)
├── Containment rules registered with graph-core
├── Connection rules (bindings, triggers, criteria)
└── casehub-diagram Lit component (viewer mode only)
```

**Milestone:** render the `document-processing.yaml` example case as a visual graph.

### Phase 3 — Property Editing

**Click a node, edit its properties.**

```
Epic 3: Property editing
├── Property panel component (Lit, in casehub-diagram)
├── Schema-driven form generation from stencil PropertySchema
├── Bidirectional binding: panel edits → domain model → YAML
├── Validation feedback (red borders, error messages)
└── Persistence: save edited YAML via backend
```

### Phase 4 — Structural Editing

**Add, remove, replace nodes. Auto-layout recalculates.**

```
Epic 4A: Structural editing           Epic 4B: Persistence backends
├── Palette component (structural     ├── Git backend (GitHub URL read/write)
│   stencils as draggable/clickable)  ├── In-memory backend (playground)
├── Add node at point (validated      └── Electron file backend (later)
│   against containment rules)
├── Remove node (with dependency
│   check — warn if connected)
├── Replace node (swap type,
│   preserve connections where valid)
├── applyEdit() in domain adapter
└── YAML round-trip (edit → serialize
    → parse → verify unchanged)
```

**4A and 4B can run in parallel.**

### Phase 5 — SWF Drill-Down

**Expand a SWF Worker to see workflow steps.**

```
Epic 5: graph-stencil-swf
├── SWF domain adapter: workflow YAML → graph model
├── Workflow step stencils (call, switch, raise, catch, entry, exit)
├── Drill-down trigger (expand Primitive with SWF executor → sub-graph)
├── casehub:dispatch trace lines (workflow step → Case capability)
└── Collapse back to single Primitive node
```

### Phase 6 — Work Registry

**Runtime-discovered work stencils from marketplace.**

```
Epic 6: graph-work-registry
├── Marketplace YAML descriptor schema
├── URL-based discovery (fetch + parse + register)
├── Category index and palette integration
├── Work stencil → Primitive node binding
├── Custom property forms per work type
└── Marketplace configuration UI (manage URLs)
```

### Phase 7 — Runtime Overlay

**Live execution state projected onto the design-time graph.**

```
Epic 7: Runtime overlay
├── Runtime data source (WebSocket or polling from Case engine)
├── PlanItem state badges on nodes
├── Milestone progression indicators
├── Heatmap colouring (usage frequency)
├── Active compound highlighting
├── Blackboard data preview on hover
└── Design ↔ Runtime mode toggle in toolbar
```

## 5. Parallelisation Map

```
Phase 0 (prerequisites)
    │
    ▼
Phase 1A (graph-core) ──────┐
Phase 1B (graph-renderer) ──┤ parallel
    │                       │
    ▼                       ▼
    └───────┬───────────────┘
            │ integrate
            ▼
        Phase 2 (case stencil — viewer)
            │
            ▼
        Phase 3 (property editing)
            │
            ▼
    Phase 4A (structural editing) ──┐
    Phase 4B (persistence backends)─┤ parallel
            │                       │
            ▼                       ▼
            └───────┬───────────────┘
                    │
          ┌─────────┼──────────┐
          ▼         ▼          ▼
      Phase 5    Phase 6    Phase 7    ← all three parallel
      (SWF)      (registry) (runtime)
```

**Maximum parallelism points:**
- Phase 1: 2 agents (graph-core + graph-renderer)
- Phase 4: 2 agents (structural editing + persistence)
- Phase 5/6/7: 3 agents (SWF stencil + work registry + runtime overlay)

**Sequential gates:**
- Phase 0 must complete before Phase 1 (types + spike)
- Phase 2 requires both 1A and 1B (first visual integration)
- Phase 3 requires Phase 2 (need a rendered graph to select nodes)
- Phase 4 requires Phase 3 (property editing validates the domain adapter round-trip)

## 6. Epic Sizing

| Phase | Epic | Size | Complexity | Parallel? |
|-------|------|------|-----------|-----------|
| 0 | Schema verification | XS | Low | — |
| 0 | Type generation | XS | Low | — |
| 0 | React Flow + Lit bridge spike | S | Med | — |
| 1A | graph-core | M | Med | Yes (with 1B) |
| 1B | graph-renderer | M | High | Yes (with 1A) |
| 2 | Case stencil (viewer) | L | High | No |
| 3 | Property editing | M | Med | No |
| 4A | Structural editing | L | High | Yes (with 4B) |
| 4B | Persistence backends | S | Low | Yes (with 4A) |
| 5 | SWF drill-down | M | Med | Yes (with 6, 7) |
| 6 | Work registry | M | Med | Yes (with 5, 7) |
| 7 | Runtime overlay | M | High | Yes (with 5, 6) |

## 7. Open Questions

1. **Schema freshness** — the CaseDefinition.yaml JSON Schema must be verified against the current Java domain model before type generation. The schema may be stale.

2. **SWF visual language** — should the SWF drill-down render workflow steps in CaseHub's visual language (unified) or adopt SWF community visual conventions (familiar to SWF users)? Recommendation: CaseHub visual language for consistency, since SWF is always experienced through CaseHub.

3. **blocks-ui package structure** — new workspace packages (recommended) vs subdirectories in existing blocks-ui structure. Recommendation: new workspace packages for type-safe boundaries and independent consumption.

4. **@xyflow/lit timeline** — when to invest in a native Lit binding for xyflow. Not before Phase 4 is complete. Could be contributed upstream to the xyflow project.

5. **Lienzo port** — strategic option for full canvas control. Revisit after the React Flow approach proves out or hits limitations. 6-10 person-week investment. Provides canvas-level drawing API, scene graph, hit-testing, pan/zoom — but requires maintaining alone.

6. **fn() vs function()** — CaseHub YAML allows `fn()` as a shortcut for `function()`. SWF uses `call`. Confirm this is the only divergence from SWF YAML conventions.

## 8. Non-Goals

- Free-form drag-and-drop node positioning (deferred — auto-layout first)
- Collaborative multi-user editing (future consideration)
- Code generation from visual model (the YAML IS the model)
- Animation of execution playback (runtime overlay shows current state, not history)
- Mobile/touch-first editing (desktop-first)
