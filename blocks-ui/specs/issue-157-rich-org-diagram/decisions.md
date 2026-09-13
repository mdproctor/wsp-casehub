## D1: Agent descriptor data flow

**Choice:** Merge descriptor data into GraphNode.properties via a separate enrichment step. `toOrgGraph(yaml)` stays a pure YAML→GraphModel function. A second function `enrichWithDescriptors(model, agents)` merges descriptor data into agent node properties. This preserves the adapter's purity (same YAML always produces same graph) and respects the protocol's "definition data parsed from YAML" semantics — descriptor data is added explicitly *after* adaptation.
**Alternatives:**
- Pass via NodeDecoration — violates node-decoration-runtime-boundary protocol; descriptors are static identity, not runtime overlays.
- Renderer context/lookup — requires changing the stencil render signature across graph-renderer infrastructure for one consumer's need.
- Merge inside toOrgGraph — breaks adapter purity; same YAML would produce different graphs depending on external state.
**Rationale:** Descriptor data belongs in properties (it's definition data, not decoration), but the adapter should not be the merge point (it breaks purity and the protocol's "parsed from YAML" language). A separate enrichment step makes the two-source nature explicit and testable in isolation.
**Trade-offs:** Two function calls instead of one. Trivial — the org-diagram component chains them.
**Sources:** node-decoration-runtime-boundary protocol (PP-20260903-c9a82e), org-adapter.ts, issue #157 comment (OrgDiagramData shape)
**Exploration:** quick
**Status:** revised (R1-01)

## D2: Component API — YAML + agents property

**Choice:** Keep YAML for org structure (`yaml`/`src` props via DiagramBaseMixin), add a new `agents` property (`Record<string, AgentDescriptor>`) for read-only descriptor enrichment. The component chains `toOrgGraph` → `enrichWithDescriptors` → `computeDerivedData`. Descriptors are not editable from this component.
**Alternatives:**
- New `data` property accepting `OrgDiagramData` — bypasses YAML entirely, loses DiagramBaseMixin editing pipeline (undo/redo, CST-preserving edits, persistence). Would require building a parallel editing pipeline and blurs ownership boundary.
**Rationale:** The split mirrors domain ownership: org structure is editable here (OrgRegistry), agent identity is read-only context (AgentRegistry). YAML pipeline stays intact for structural editing. Descriptor enrichment layers on top without affecting the editing contract.
**Trade-offs:** Consumers must split a joined REST response into YAML + agents map. Trivial orchestration in the workbench or app layer.
**Sources:** DiagramBaseMixin (pages-diagram-core), issue #157 comment (OrgDiagramData shape), D1
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D3: Derived relationship data as separate pure function

**Choice:** A separate pure function `computeDerivedData(model)` walks the relationship graph to compute per-agent supervision targets, escalation chains, backup agents, and attestation grants. Results stored in GraphNode.properties. Independent of descriptor data — operates on the edge list only.
**Alternatives:**
- Compute in stencil renderer — mixes graph traversal with rendering. Stencil render function only sees its own node + decoration, would need the full edge list.
- Merge into toOrgGraph — adds unrelated concerns (graph traversal) to YAML parsing.
**Rationale:** Separate pure function keeps each step focused and independently testable. The function needs the full edge list, which the adapter already produces. The stencil stays a pure renderer of pre-computed properties.
**Trade-offs:** Additional function call in the pipeline. Trivial overhead.
**Sources:** org-adapter.ts (existing agentIndex, resolveAgentNode)
**Exploration:** quick
**Status:** revised (R1-05, R1-12 — removed false D1 dependency)

## D4: Escalation chain, attestation, and legend as internal sub-panels

**Choice:** Internal Lit elements within `components/org-diagram/src/panels/` — `OrgEscalationChainPanel`, `OrgAttestationPanel`, `OrgLegend`. Each accepts data via properties and emits events for interaction. Exported from the org-diagram package for testing and composition. Promoted to standalone packages when a second consumer materialises.
**Alternatives:**
- Standalone packages immediately — violates promotion pipeline principle (ARC42STORIES §5). No second consumer exists. Creates unnecessary maintenance overhead (3 package.json, 3 tsconfig, 3 registry entries).
- Rendered as synthetic graph nodes — fighting ELK layout for what are annotations, not graph entities.
**Rationale:** Follows the established promotion pipeline: born inside the consumer, promoted when reuse materialises. Channel-activity has 8 sub-components in one package — precedent is clear.
**Trade-offs:** If a second consumer needs the panels before promotion, they'd import from org-diagram package rather than a dedicated package. Acceptable — promotion is a quick extraction when needed.
**Sources:** ARC42STORIES.MD §5 (promotion pipeline), channel-activity sub-components, reference SVG panels
**Exploration:** quick
**Status:** revised (R1-02)

## D5: Pre-compute agent card heights with typed data

**Choice:** `computeNodeSizes(model)` pre-computes node heights based on populated content rows using typed `OrgAgentNodeData`. Long values are truncated with ellipsis (no text wrapping), keeping the height formula deterministic. A height sync test structurally couples sizing with stencil rendering.
**Alternatives:**
- Fixed height with scroll overflow — wastes space on sparse agents, clips dense ones.
- Two-pass render — doubles rendering work, adds pipeline complexity.
**Rationale:** Deterministic, no double-render. Typed interface (R1-03) prevents property access errors. Truncation avoids font-metric complexity. Sync test catches drift in CI.
**Trade-offs:** Truncated long values may hide information. Acceptable — the tooltip and property panel show full detail.
**Sources:** Reference SVG (variable-height agent cards), ElkLayoutOptions.nodeSizes
**Exploration:** quick
**Status:** revised (R1-03, R1-06 — added typed data, truncation, sync test)

## D6: Unit kind color mapping — built-in palette with override

**Choice:** Ship a default palette mapping common eidos kinds to gradient pairs (supervision-hierarchy → purple, rig → blue, department → teal, holarchy → indigo). Unknown kinds auto-assigned from unused palette slots. Component exposes `kindColors` property for consumer overrides.
**Alternatives:**
- Hash-derived colors — no manual mapping but no semantic association. "rig" could end up red.
- Fully configurable, no defaults — tedious for the common case.
**Rationale:** Semantic defaults serve the primary consumer (eidos). Override property handles non-eidos vocabularies. Hash fallback for unknown kinds is deterministic.
**Trade-offs:** Default palette is eidos-specific. Override property handles other vocabularies.
**Sources:** Reference SVG (headerOversight purple, headerRig blue gradients)
**Exploration:** quick
**Status:** captured

## D7: Edge labels via post-processing — pure function

**Choice:** Pure function `applyOrgEdgeLabels(edges): edges` post-processes ReactFlow edges. Sets native `label`, `labelStyle`, `labelBgStyle`, `labelBgPadding`, `labelBgBorderRadius` properties based on edge data. Separate from selection highlighting (D10).
**Alternatives:**
- Extend graph-renderer mapping — cross-repo change for org-specific rendering logic. Wrong layer.
- Custom ReactFlow edge components — available as a fallback if text-only labels prove insufficient.
**Rationale:** ReactFlow natively supports edge labels with background styling. Org-specific label logic stays in graph-stencil-org. Separated from highlighting (R1-11) for single-responsibility.
**Trade-offs:** Text-only labels (no React element pills). Sufficient for the reference SVG's actual requirements. Custom edge component is the fallback if richer rendering is needed.
**Sources:** ReactFlow Edge API (label, labelBgStyle), ReactFlowApp.tsx:234, reference SVG scope badges
**Exploration:** quick
**Status:** revised (R1-11 — separated from highlighting)

## D8: Collapsible units via adapter filtering with re-layout

**Choice:** Clicking a unit header toggles a `collapsedUnits: Set<string>` on the component. `applyCollapsedUnits(model, yamlPaths, collapsedUnits)` omits agent nodes and their edges, produces compact unit height (~40px). ELK re-layouts with the reduced graph.
**Alternatives:**
- CSS-only hide — ELK doesn't know about the collapse, wastes space.
**Rationale:** ELK needs accurate sizes. Filtering is a natural adapter operation.
**Trade-offs:** Re-layout on collapse/expand. ELK is fast for org-sized graphs. Nodes may jump positions — transition animation is a future enhancement if needed.
**Sources:** org-adapter.ts (node generation loop), ElkLayoutOptions.nodeSizes
**Exploration:** quick
**Status:** captured

## D9: Hover tooltip for quick scanning + property panel for full detail

**Choice:** Both hover and selection. Mouseover shows a lightweight rendered tooltip with key properties — read-only, no forms, with `role="tooltip"` and `aria-describedby`. Click/selection opens the full property panel. 150ms dismiss delay prevents flicker on dense graphs.
**Alternatives:**
- Property panel only — slow for scanning. User explicitly requested hover.
- Rich hover only — competes with the property panel.
**Rationale:** Different interaction modes: hover for overview, click for depth. Tooltip is a subset of stencil content plus goals/constraints.
**Trade-offs:** Two rendering paths (tooltip + panel). Tooltip is simpler — rendered-only, no forms.
**Sources:** Reference SVG, issue #157 §7
**Exploration:** quick
**Status:** revised (R1-08 — added ARIA)

## D10: Relationship highlighting — separate pure function

**Choice:** Pure function `applySelectionHighlight(edges, nodes, selectedAgentId?)` sets `className: 'org-edge-highlighted'` on connected edges, `style: { opacity: 0.15 }` on others. Separate from edge label enrichment (D7). The org-diagram chains them: `applyOrgEdgeLabels` then `applySelectionHighlight`.
**Alternatives:**
- Combined with D7 — mixes labeling and highlighting concerns in one function.
- Custom edge components — over-engineered for opacity toggling.
**Rationale:** Single-responsibility: labels and highlighting are independent concerns, independently testable. CSS class toggling on ReactFlow edge properties.
**Trade-offs:** Two post-processing passes over the edge array. Trivial.
**Sources:** ReactFlow Edge className/style properties, issue #157 §7
**Exploration:** quick
**Status:** revised (R1-11 — separated from D7)
