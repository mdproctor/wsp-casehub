# Rich Org Diagram — Design Spec

**Issue:** casehubio/blocks-ui#157
**Date:** 2026-09-08
**Status:** Draft
**Builds on:** casehubio/blocks-ui#155 (base org diagram — adapter, stencils, archetype detection, layout, editing)

---

## Overview

Enrich the existing org-diagram component with full information density: rich agent cards showing descriptor properties (slot, disposition, capabilities, supervision targets, escalation paths, backup, attestation), kind-differentiated unit containers with capability pills, labeled relationship lines with scope badges, internal sub-panels (escalation chain, attestation grants, legend), and interactive features (collapsible units, hover tooltips, relationship highlighting).

The reference target is `casehubio/eidos/docs/diagrams/gastown-org-structure.svg` — this spec matches or exceeds its information density in an interactive, editable form.

---

## Data Model

### Agent Descriptor Integration (D1, D2)

The component receives two data sources via a single API surface:

| Property | Type | Source | Editable |
|----------|------|--------|----------|
| `yaml` / `src` | `string` | Org YAML (units, members, relationships) | Yes — via DiagramBaseMixin pipeline |
| `agents` | `Record<string, AgentDescriptor>` | Agent descriptors keyed by agentId | No — read-only enrichment |
| `kindColors` | `Record<string, {start: string, end: string}>` | Kind-to-gradient color overrides | No — optional configuration |

```typescript
interface AgentDescriptor {
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  goals?: AgentGoal[];
  constraints?: AgentConstraint[];
  briefing?: string;
}

interface DispositionAxes {
  autonomy?: string;
  ruleFollowing?: string;
  socialOrient?: string;
  riskAppetite?: string;
  conflictMode?: string;
}
```

### Typed Node Data Interfaces

All node properties are typed — no untyped `Record<string, unknown>` access in stencils:

```typescript
interface OrgAgentNodeData {
  agentId: string;
  role?: string;
  roleVocabulary?: string;
  unitId: string;
  label: string;
  // From descriptor (D1)
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  // From derived computation (D3)
  supervisionTargets?: string[];
  escalationChain?: string[];
  backupAgents?: { agentId: string; scope?: string; direction: 'backs' | 'backed-by' }[];
  attestationGrants?: { targetAgentId: string; scope?: string; dimensions: string[]; signalTypes?: string[] }[];
  // From color resolution
  unitKind?: string;
  unitColorStart?: string;
  unitColorEnd?: string;
}

interface OrgUnitNodeData {
  unitId: string;
  name: string;
  kind?: string;
  kindVocabulary?: string;
  label: string;
  memberCount: number;
  capabilities: AgentCapability[];
  goals: AgentGoal[];
  constraints: AgentConstraint[];
  // From color resolution
  kindColorStart?: string;
  kindColorEnd?: string;
}
```

### Adapter Decomposition — Three Pure Functions

The adapter is decomposed into three independent pure functions chained by the org-diagram component. This keeps `toOrgGraph` as a pure YAML→GraphModel function, makes each step independently testable, and respects the protocol's "definition data parsed from YAML" semantics.

```typescript
// Step 1: Pure YAML → GraphModel (unchanged contract)
export function toOrgGraph(yaml: string): OrgAdapterResult;

// Step 2: Merge descriptor data into agent node properties
export function enrichWithDescriptors(
  model: GraphModel,
  agents: Readonly<Record<string, AgentDescriptor>>,
): GraphModel;

// Step 3: Compute derived data from relationship graph
export function computeDerivedData(
  model: GraphModel,
): DerivedOrgData;

interface DerivedOrgData {
  model: GraphModel;  // nodes enriched with supervision/escalation/backup/attestation
  escalationChains: readonly { path: string[]; terminal: string }[];
  attestationSummary: readonly {
    source: string; target: string;
    scope?: string; dimensions: string[];
    signalTypes?: string[];
  }[];
}
```

The org-diagram component chains them:
```typescript
const base = toOrgGraph(yaml);
const enriched = this.agents
  ? enrichWithDescriptors(base.model, this.agents)
  : base.model;
const derived = computeDerivedData(enriched);
```

`toOrgGraph` stays pure — same YAML always produces the same graph. `enrichWithDescriptors` adds descriptor data. `computeDerivedData` walks the relationship graph for escalation chains, supervision targets, backup agents, and attestation grants.

### Additional Adapter Functions

```typescript
// Resolve kind colors into node properties
export function resolveKindColors(
  model: GraphModel,
  kindColors?: Readonly<Record<string, { start: string; end: string }>>,
): GraphModel;

// Compute node sizes based on populated content rows
export function computeNodeSizes(
  model: GraphModel,
): ReadonlyMap<string, { width: number; height: number }>;

// Filter collapsed units — removes agent nodes AND their edges
export function applyCollapsedUnits(
  model: GraphModel,
  yamlPaths: ReadonlyMap<string, readonly (string | number)[]>,
  collapsedUnits: ReadonlySet<string>,
): { model: GraphModel; yamlPaths: ReadonlyMap<string, readonly (string | number)[]> };
// Edges where source or target is inside a collapsed unit are removed entirely.
// Cross-unit edges where only one end is collapsed are also removed (not re-routed).
```

The full pipeline in the org-diagram component:
```typescript
const base = toOrgGraph(yaml);
let model = this.agents ? enrichWithDescriptors(base.model, this.agents) : base.model;
const derived = computeDerivedData(model);
model = resolveKindColors(derived.model, this.kindColors);
const { model: layoutModel, yamlPaths } = this._collapsedUnits.size > 0
  ? applyCollapsedUnits(model, base.yamlPaths, this._collapsedUnits)
  : { model, yamlPaths: base.yamlPaths };
const nodeSizes = computeNodeSizes(layoutModel);
```

### Derived Relationship Data (D3)

`computeDerivedData` walks the relationship graph and stores per-agent derived data in `OrgAgentNodeData`:

| Derived property | Computation |
|-----------------|-------------|
| `supervisionTargets` | Outgoing SUPERVISES edges from this agent → `string[]` of target agentIds |
| `escalationChain` | Walk ESCALATES_TO from this agent to terminal → `string[]` ordered chain |
| `backupAgents` | BACKS_UP edges involving this agent → `{agentId, scope?, direction}[]` |
| `attestationGrants` | From ANY relationship kind with attestation field → `{targetAgentId, scope?, dimensions, signalTypes?}[]` |

D3 is independent of D1 — derived data comes from the relationship graph (edges), not descriptor data.

**Attestation grants:** Derived from ALL relationship kinds that carry an `attestation` field, not just SUPERVISES. The `AgentRelationship` type allows any kind to have attestation.

**Cycle detection:** Escalation chain walking tracks visited nodes. If a cycle is detected, the chain is truncated at the revisit point and the `terminal` field is set to the cycle entry node with a `(cycle)` marker. The escalation chain panel renders cycles distinctly.

**Stencil typing:** Node properties are accessed via a typed cast: `const data = node.properties as OrgAgentNodeData`. The cast is the enforcement mechanism — `GraphNode.properties` is `Record<string, unknown>` in graph-core.

### Color Resolution

`resolveKindColors` stores resolved colors in node properties so stencils remain pure renderers:

- **Unit nodes** get `kindColorStart` and `kindColorEnd` (from `kindColors` param → default palette → auto-assignment fallback)
- **Agent nodes** get `unitKind`, `unitColorStart`, `unitColorEnd` (from parent unit's resolved colors)

---

## Rich Agent Card Stencil (org-agent)

Replace the current minimal card with a multi-row rendered card matching the reference SVG density.

### Card Structure

```
┌──────────────────────────────────────────┐
│ ● agent-id                    ROLE BADGE │  ← header (tinted bg)
├──────────────────────────────────────────┤
│ SLOT     worker                          │
│ CAPS     full-stack-code-work            │
│ DISPOSITION  [autonomy: semi-auto]       │
│              [rules: measured]           │
│ SUPERVISES   polecat-1, polecat-2        │
│ ESCALATES    → witness-α → deacon → boot │
│ BACKUP       polecat-2 (scope: code-a…)  │
│ ATTESTATION  [LATENCY] [ATTEST_RATE]     │
│              signals: COMPLIANT, VIOLATED│
└──────────────────────────────────────────┘
```

### Row Rendering Rules

Each row is conditionally rendered based on data availability. Long values are truncated with ellipsis — no text wrapping. This keeps the height formula deterministic.

| Row | Condition | Label style | Value style |
|-----|-----------|-------------|-------------|
| SLOT | `slot` present | Gray 9px bold | Dark 9px |
| CAPS | `capabilities` non-empty | Gray 9px bold | Dark 9px, comma-separated, truncated |
| DISPOSITION | Any axis populated | Gray 9px bold | Color-coded pills (see below) |
| SUPERVISES | `supervisionTargets` non-empty | Gray 9px bold | Dark 9px, comma-separated agentIds, truncated |
| ESCALATES | `escalationChain` non-empty | Gray 9px bold | Red 9px, `→` separated chain, truncated |
| BACKUP | `backupAgents` non-empty | Gray 9px bold | Dark 9px, with scope in parens |
| ATTESTATION | `attestationGrants` non-empty | Gray 9px bold | Purple dimension pills + signal text |

### Header

- Colored circle indicator: matches unit kind color (purple for oversight, blue for rig — from D6 palette)
- Agent ID: bold 12px, colored to match unit kind
- Role badge: uppercase 8px, right-aligned, unit kind color

### Disposition Axis Pills

| Axis | Background | Text color |
|------|------------|------------|
| `autonomy` | `#fed7d7` | `#c53030` |
| `ruleFollowing` | `#fefcbf` | `#975a16` |
| `socialOrient` | `#c6f6d5` | `#276749` |
| `riskAppetite` | `#c6f6d5` | `#276749` |
| `conflictMode` | `#ebf8ff` | `#2b6cb0` |

Format: `axis-short-name: value` in a rounded pill (3px radius, 14px height).

Short names defined as a constant:

```typescript
const DISPOSITION_SHORT_NAMES: Record<keyof DispositionAxes, string> = {
  autonomy: 'autonomy',
  ruleFollowing: 'rules',
  socialOrient: 'social',
  riskAppetite: 'risk',
  conflictMode: 'conflict',
};
```

### Node Sizing (D5)

`computeNodeSizes` pre-computes each agent node's height based on populated rows:

```typescript
const ROW_HEIGHT = 16;
const HEADER_HEIGHT = 28;
const DISPOSITION_ROW_HEIGHT = 18;
const ATTESTATION_HEIGHT = 32;
const PADDING = 12;
const AGENT_WIDTH = 280;

function computeAgentHeight(data: OrgAgentNodeData): number {
  let h = HEADER_HEIGHT + PADDING;
  if (data.slot) h += ROW_HEIGHT;
  if (data.capabilities?.length) h += ROW_HEIGHT;
  if (data.disposition && Object.keys(data.disposition).length > 0) {
    h += DISPOSITION_ROW_HEIGHT;
  }
  if (data.supervisionTargets?.length) h += ROW_HEIGHT;
  if (data.escalationChain?.length) h += ROW_HEIGHT;
  if (data.backupAgents?.length) h += ROW_HEIGHT;
  if (data.attestationGrants?.length) h += ATTESTATION_HEIGHT;
  return h;
}
```

The `nodeSizes` map is passed to `computeElkLayout()` so ELK allocates correct space.

**Unit sizing:** `computeNodeSizes` also computes unit container heights: header (30px) + capability pills row (20px if capabilities present) + padding for ELK child layout. Unit width is determined by ELK based on children.

**Height sync test:** A test renders nodes with known data, applies the same row-counting logic used by the stencil, and asserts it matches `computeAgentHeight`. This structurally couples sizing and rendering — drift fails in CI.

---

## Rich Unit Container Stencil (org-unit)

### Header Gradient by Kind (D6)

Default palette:

| Kind | Gradient start | Gradient end |
|------|---------------|--------------|
| `supervision-hierarchy` | `#553c9a` | `#6b46c1` |
| `rig` | `#2c5282` | `#3182ce` |
| `department` | `#276749` | `#38a169` |
| `holarchy` | `#3730a3` | `#4f46e5` |
| `team` | `#9c4221` | `#dd6b20` |

Unknown kinds: auto-assigned from unused slots in rotation order. The component's `kindColors` property (`Record<string, {start: string, end: string}>`) overrides any default.

The kind color is also used for:
- Agent card header tint (lighter version)
- Agent card circle indicator fill
- Agent card border stroke
- Agent ID text color

### Capability Pills

Below the header, render unit capabilities as pill badges:

```
[CAP: full-stack-code-work] [CAP: code-review]
```

Style: `#ebf8ff` background, `#90cdf4` border, `#2b6cb0` text, 8px font, 4px radius.

### Collapsible (D8)

Clicking the unit header toggles collapse. The component maintains a `_collapsedUnits: Set<string>` state. On toggle, re-runs the pipeline with `applyCollapsedUnits` and re-layouts.

Collapsed state: header bar only (~40px), no agents visible, member count badge shows how many are hidden.

**ARIA:** Unit header gets `aria-expanded="true|false"` reflecting collapse state. Screen readers announce the toggle.

---

## Edge Labels (D7) and Selection Highlighting (D10)

Two separate pure functions — label enrichment and selection highlighting are independent concerns:

### `applyOrgEdgeLabels(edges: Edge[]): Edge[]`

Sets ReactFlow label properties based on edge data:

| Edge type | Label | Background | Border | Text color |
|-----------|-------|------------|--------|------------|
| `org-supervises` | `scope: {capabilityName}` when scoped, none otherwise | `#e9d8fd` | `#d6bcfa` | `#553c9a` |
| `org-delegates-to` | `"DELEGATES_TO"` always | `#e6fffa` | `#81e6d9` | `#2c7a7b` |
| `org-escalates-to` | none (line style is distinctive enough) | — | — | — |
| `org-reports-to` | none | — | — | — |
| `org-backs-up` | `"BACKS_UP"` + scope if present | `#ebf8ff` | `#90cdf4` | `#2b6cb0` |
| `org-extended` | `extendedKind` value | `#f3e8ff` | `#c4b5fd` | `#6d28d9` |

ReactFlow properties set per edge: `label`, `labelStyle: { fontSize: 8, fontWeight: 600 }`, `labelBgStyle: { fill, stroke }`, `labelBgPadding: [3, 6]`, `labelBgBorderRadius: 3`.

### `applySelectionHighlight(edges: Edge[], selectedNodeId?: string): Edge[]`

When `selectedNodeId` is provided (format: `agent:<unitId>:<agentId>`), edges whose `source` or `target` matches the node ID get `className: 'org-edge-highlighted'`. All other edges get `style: { opacity: 0.15 }`. Clearing selection restores all edges to full opacity. Uses node IDs throughout — no agentId extraction needed.

### Lightweight Selection Update Path

Selection changes must NOT trigger `_fullRender` (which runs ELK layout). Instead, the component caches the post-`toReactFlowGraph` edges as `_baseEdges`. On selection change, it re-applies the two edge functions to the cached edges:

```typescript
// In _fullRender (after toReactFlowGraph):
this._baseEdges = rfEdges;
this._updateEdgeStyles();

// On selection change (lightweight — no layout):
private _updateEdgeStyles(): void {
  let edges = applyOrgEdgeLabels(this._baseEdges);
  edges = applySelectionHighlight(edges, this._selectedNodeId || undefined);
  (this as any)._edges = edges;
}
```

This ensures agent clicks update edge highlighting instantly without re-parsing YAML or re-running ELK.

---

## Internal Sub-Panels (D4)

These start as internal modules within `components/org-diagram/src/panels/`. Each is a Lit element exported from the org-diagram package. When a second consumer needs them, they get promoted to standalone packages — following the established promotion pipeline.

### `OrgEscalationChainPanel` (internal element)

**File:** `components/org-diagram/src/panels/escalation-chain-panel.ts`

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `chains` | `{path: string[], terminal: string}[]` | Escalation chains from adapter |
| `highlightAgent` | `string \| undefined` | Agent to highlight in the chain |

**Events:**
- `agent-click` — `CustomEvent<{agentId: string}>` when an agent name is clicked

**Rendering:**
Red-tinted panel (`#fff5f5` background, `#feb2b2` border). Title: "ESCALATION CHAIN" bold red. Each chain as a horizontal sequence: agent names in dark text, red `→` arrows between them, `(terminal)` gray text after the final agent. The highlighted agent (if any) gets a bold + underline treatment.

**ARIA:** `role="region"`, `aria-label="Escalation chains"`. Each chain rendered as an ordered list (`role="list"` + `role="listitem"`).

### `OrgAttestationPanel` (internal element)

**File:** `components/org-diagram/src/panels/attestation-panel.ts`

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `grants` | `{source: string, target: string, scope?: string, dimensions: string[], signalTypes?: string[]}[]` | Attestation grants from adapter |
| `highlightAgent` | `string \| undefined` | Agent to highlight |

**Events:**
- `agent-click` — `CustomEvent<{agentId: string}>`

**Rendering:**
Purple-tinted panel (`#faf5ff` background, `#d6bcfa` border). Title: "ATTESTATION GRANTS (source → target)" bold purple. For each grant: scope line, dimension pills (purple `#e9d8fd`), signal type pills (green `#c6f6d5` for COMPLIANT, red `#fed7d7` for VIOLATED), store target text.

**ARIA:** `role="region"`, `aria-label="Attestation grants"`.

### `OrgLegend` (internal element)

**File:** `components/org-diagram/src/panels/org-legend.ts`

**Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `kindColors` | `Record<string, {start: string, end: string}>` | Kind-to-color mapping (rendered as swatches) |
| `showDisposition` | `boolean` | Whether to show disposition axis section (default true) |
| `showAttestation` | `boolean` | Whether to show attestation section (default true) |

**Sections:**
1. **Relationships** — line samples for SUPERVISES, ESCALATES_TO, BACKS_UP, DELEGATES_TO with matching stroke styles
2. **Unit Kinds** — color swatches with kind names (from `kindColors`)
3. **Agent Properties** — dot indicators (colored per unit kind)
4. **Disposition Axes** — color swatches for each axis with labels
5. **Scope & Attestation** — pill samples: scoped supervision, attestation grant, unit capability
6. **Eidos Model Layers** — brief text description

**ARIA:** `role="region"`, `aria-label="Organization diagram legend"`.

---

## Hover Tooltip (D9)

### Implementation

The org-diagram component renders a floating tooltip on agent node mouseover. The tooltip shows a rendered subset of agent properties — same visual language as the card (pills, badges, text rows) but includes additional detail (goals, constraints) not shown in the card.

**Tooltip content (rendered, read-only):**
- Agent ID + role badge (header)
- Slot
- Capabilities
- Disposition pills
- Goals (name + priority)
- Constraints (name + severity)
- Briefing (truncated to 2 lines)

**Positioning:** Above the hovered node, clamped to viewport. Dismisses on mouseout with a 150ms delay to prevent flicker when moving between adjacent cards.

**ARIA:** The tooltip element has `role="tooltip"` and a unique ID. The triggering node references it via `aria-describedby`. The tooltip content is announced to screen readers when it appears.

**Relationship to property panel:** The tooltip is a quick preview for scanning. The property panel (on click/selection) shows everything plus editing forms for editable fields (agentId, role via the org YAML).

---

## Component Composition

The org-diagram composes the panels below the canvas:

```
┌─────────────────────────────────────────────────────────┐
│ Toolbar                                                 │
├──────┬──────────────────────────────┬───────────────────┤
│      │                              │                   │
│ Pal  │    Graph Canvas              │  Properties       │
│      │                              │                   │
├──────┴──────────────────────────────┴───────────────────┤
│ ┌─── Escalation Chain ──┐ ┌── Attestation Grants ─────┐ │
│ │ polecat-1 → witness…  │ │ deacon → witnesses        │ │
│ │ polecat-2 → witness…  │ │ scope: rig-monitoring     │ │
│ └───────────────────────┘ └───────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│ Legend                                                   │
└─────────────────────────────────────────────────────────┘
```

The panels receive data from `computeDerivedData` results (`escalationChains`, `attestationSummary`). The legend receives the resolved `kindColors`. Agent click events from panels trigger node selection in the canvas.

---

## AgentDescriptor Type (new in graph-stencil-org)

```typescript
export interface AgentDescriptor {
  slot?: string;
  capabilities?: AgentCapability[];
  disposition?: Partial<DispositionAxes>;
  goals?: AgentGoal[];
  constraints?: AgentConstraint[];
  briefing?: string;
}

export interface DispositionAxes {
  autonomy?: string;
  ruleFollowing?: string;
  socialOrient?: string;
  riskAppetite?: string;
  conflictMode?: string;
}
```

This type is defined in `graph-stencil-org/src/types.ts` alongside the existing org types. `AgentCapability`, `AgentGoal`, and `AgentConstraint` are already defined there.

---

## Testing

### graph-stencil-org tests

- **toOrgGraph purity:** Same YAML always produces same GraphModel regardless of external state
- **enrichWithDescriptors:** Merge agents map → verify agent nodes have descriptor properties (slot, disposition, capabilities). Partial map → verify missing descriptors produce basic nodes.
- **computeDerivedData:**
  - Supervision targets: verify each agent's `supervisionTargets` matches outgoing SUPERVISES edges
  - Escalation chains: verify full chain walk (e.g. polecat-1 → witness-alpha → deacon → boot)
  - Backup agents: verify bidirectional backup with scope
  - Attestation grants: verify grants derived from SUPERVISES edges with attestation
- **resolveKindColors:** Default palette applied. Custom override applied. Unknown kinds auto-assigned.
- **computeNodeSizes:** Agent with all fields → max height. Agent with only role → min height. Height sync test: row-counting logic matches stencil rendering.
- **applyCollapsedUnits:** Collapsed unit omits agent nodes, produces compact unit height.
- **applyOrgEdgeLabels:** Scoped SUPERVISES → label with scope badge. Unscoped SUPERVISES → no label. BACKS_UP → label with scope.
- **applySelectionHighlight:** Selected agent → connected edges highlighted, others dimmed. No selection → all edges full opacity.

### org-diagram tests

- **ARIA:** Verify role, aria-label on host, canvas, toolbar, tooltip. Verify `aria-expanded` on collapsible units. Verify tooltip has `role="tooltip"`.
- **Agents property:** Set `agents` map → verify rich card rendering (disposition pills visible)
- **Hover tooltip:** Mouseover agent node → verify tooltip appears with goals/constraints
- **Collapsible units:** Click unit header → verify `aria-expanded` toggles, member count shown, agents hidden
- **Selection highlighting:** Click agent → verify connected edges highlighted, others dimmed
- **Kind colors:** Set custom kindColors → verify unit header uses override color

### Internal panel tests

- **Escalation chain panel:** Pass chains → verify rendered paths with arrows and terminal markers. Click agent name → verify event emitted. Verify ARIA list structure.
- **Attestation panel:** Pass grants → verify dimension pills and signal pills rendered. Verify scope text.
- **Legend:** Pass kindColors → verify swatches rendered. Toggle showDisposition → verify section hidden.
- **ARIA:** All three panels: verify role and aria-label.

---

## References

- `casehubio/eidos/docs/diagrams/gastown-org-structure.svg` — reference SVG target
- `packages/graph-stencil-org/src/adapter/org-adapter.ts` — existing adapter (toOrgGraph)
- `packages/graph-stencil-org/src/stencils/org-agent.ts` — current minimal agent stencil
- `packages/graph-stencil-org/src/stencils/org-unit.ts` — current minimal unit stencil
- `packages/graph-stencil-org/src/types.ts` — existing org domain types
- `components/org-diagram/src/blocks-org-diagram.ts` — existing diagram component
- `docs/protocols/blocks-ui/node-decoration-runtime-boundary.md` — descriptor data is definition, not decoration
- `docs/protocols/blocks-ui/component-registry-props.md` — new components need registry entries
- `docs/protocols/blocks-ui/stencil-package-isolation.md` — panels are separate components, not stencil cross-imports
- `docs/specs/issue-155-org-diagram/2026-09-05-org-diagram-design.md` — base org diagram spec
- `docs/specs/issue-155-org-diagram/decisions.md` — 15 prior decisions (D1-D15)
- ReactFlow Edge API — label, labelBgStyle, labelBgPadding, labelBgBorderRadius
- `casehubio/blocks-ui#157` — issue with full requirements
- `casehubio/blocks-ui#157` comment — OrgDiagramData shape guidance

---

## BlocksComponentRegistry — OrgDiagram

The existing `blocks-org-diagram` component is not yet registered in `BlocksComponentRegistry`. This spec adds new properties (`agents`, `kindColors`). Must add:

```typescript
export interface OrgDiagramProps {
  yaml?: string;
  src?: string;
  agents?: Record<string, AgentDescriptor>;
  kindColors?: Record<string, { start: string; end: string }>;
  layoutStrategy?: OrgLayoutStrategy | 'auto';
  selectionTopic?: string;
  readonly?: boolean;
}
```

Add to registry in `packages/blocks-ui-schema/src/registry.ts`. Run `yarn workspace @casehubio/blocks-ui-schema run generate`.

---

## Deferred Items

These issue requirements are not addressed by this spec and should be filed as follow-up issues:

| Requirement | Issue §  | Reason |
|-------------|----------|--------|
| Draggable agent repositioning within units | §7 | Requires position persistence, container bounds constraints, and interaction with re-layout. Separate concern from information density. |
| Double-click inline editing | §7 | Current editing is via property panel + YAML pane. Inline editing is a UX enhancement beyond the information density scope. |
