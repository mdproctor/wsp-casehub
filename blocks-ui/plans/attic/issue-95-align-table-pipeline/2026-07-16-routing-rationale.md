# Routing Rationale Component Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #54 — feat: routing-rationale component
**Issue group:** #67, #54

**Goal:** Build a routing-rationale component that explains trust-weighted routing decisions — selected candidate, alternatives, scores, phases, policy.

**Architecture:** Web Component (`routing-rationale`) extending `DataSourceMixin(LiveRegionMixin(LitElement))`. Dual-data mode (property + endpoint). Alternatives rendered via pages-table with inline-styled column renderers. Score Header and Policy Summary render from `_rawData` side-channel.

**Tech Stack:** Lit 3, TypeScript, pages-table, pages-data (fromRows, columnId, ColumnType), blocks-ui-core (DataSourceMixin, createTypedFetchSource, emitPagesEvent), pages-primitives (LiveRegionMixin)

## Global Constraints

- Column renderers MUST use inline styles (not CSS classes) — shadow DOM scoping prevents parent styles from reaching pages-table's shadow DOM (#67)
- CSS tokens use `--pages-{semantic}-{scale}` numbered system (`-3` backgrounds, `-11` text, `-4` borders)
- Protocol PP-20260713-8ea1af: typed config + render callbacks, no slots for content
- `trustScore` is `null` for BOOTSTRAP candidates — renderers must handle NULL cells
- `@casehubio/pages-primitives` version `^0.2.2` for `LiveRegionMixin`

---

### Task 1: Package scaffold + types + empty component shell

**Files:**
- Create: `components/routing-rationale/package.json`
- Create: `components/routing-rationale/tsconfig.json`
- Create: `components/routing-rationale/tsconfig.build.json`
- Create: `components/routing-rationale/vitest.config.ts`
- Create: `components/routing-rationale/src/types.ts`
- Create: `components/routing-rationale/src/index.ts`
- Create: `components/routing-rationale/src/routing-rationale.ts`
- Create: `components/routing-rationale/src/routing-rationale.test.ts`

**Interfaces:**
- Produces: `RoutingRationaleData`, `CandidateScore`, `RoutingPolicySummary`, `RoutingCandidateSelectedDetail` types. `RoutingRationale` class with `data`, `scoreLabel`, `capabilityLabel`, `renderCandidate` properties. `RoutingRationaleTopics.CANDIDATE_SELECTED` constant.

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-routing-rationale",
  "version": "0.2.2",
  "description": "Trust-weighted routing decision explanation — score vs threshold, alternatives, phases",
  "repository": { "type": "git", "url": "https://github.com/casehubio/blocks-ui.git" },
  "publishConfig": { "registry": "https://npm.pkg.github.com" },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "@casehubio/pages-component": "^0.2.2",
    "@casehubio/pages-data": "^0.2.2",
    "@casehubio/pages-primitives": "^0.2.2",
    "@casehubio/pages-table": "^0.2.2",
    "lit": "^3.2.1"
  },
  "devDependencies": {
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  },
  "license": "Apache-2.0"
}
```

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
  "include": ["src"],
  "references": [{ "path": "../../packages/blocks-ui-core" }]
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

- [ ] **Step 4: Create vitest.config.ts**

Copy the pattern from similarity-panel's vitest.config.ts — same alias structure, same esbuild config, jsdom environment.

- [ ] **Step 5: Create types.ts**

```typescript
export interface RoutingRationaleData {
  readonly capabilityTag: string;
  readonly strategyId: string;
  readonly selected: CandidateScore;
  readonly alternatives: readonly CandidateScore[];
  readonly policy: RoutingPolicySummary;
}

export interface CandidateScore {
  readonly workerId: string;
  readonly trustScore: number | null;
  readonly workloadScore: number;
  readonly phase: 'BOOTSTRAP' | 'QUALIFIED' | 'BORDERLINE' | 'EXCLUDED_PHASE2B' | 'EXCLUDED_PHASE3';
  readonly observations: number;
  readonly finalScore: number;
  readonly exclusionReason?: string;
  readonly rationale?: string;
  readonly additionalScores?: Readonly<Record<string, number>>;
}

export interface RoutingPolicySummary {
  readonly threshold: number;
  readonly borderlineMargin: number;
  readonly blendFactor: number;
  readonly minimumObservations: number;
  readonly qualityFloors: Readonly<Record<string, number>>;
  readonly cbrWeight: number;
  readonly bootstrapEscalationRequired: boolean;
}

export interface RoutingCandidateSelectedDetail {
  readonly workerId: string;
  readonly trustScore: number | null;
  readonly finalScore: number;
  readonly phase: CandidateScore['phase'];
}
```

- [ ] **Step 6: Create index.ts**

```typescript
export * from './types.js';
export { RoutingRationale, RoutingRationaleTopics } from './routing-rationale.js';
```

- [ ] **Step 7: Write failing test — empty component renders empty state**

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import './routing-rationale.js';
import type { RoutingRationaleData, CandidateScore, RoutingPolicySummary } from './types.js';

type RoutingRationaleEl = HTMLElement & {
  endpoint?: string;
  data: RoutingRationaleData | null;
  scoreLabel: string;
  capabilityLabel?: string;
  loading: boolean;
  updateComplete: Promise<boolean>;
};

const POLICY: RoutingPolicySummary = {
  threshold: 0.7, borderlineMargin: 0.1, blendFactor: 0.6,
  minimumObservations: 10, qualityFloors: {}, cbrWeight: 0, bootstrapEscalationRequired: false,
};

const SELECTED: CandidateScore = {
  workerId: 'agent-a', trustScore: 0.82, workloadScore: 0.8,
  phase: 'QUALIFIED', observations: 14, finalScore: 0.812,
};

const ALTERNATIVES: CandidateScore[] = [
  { workerId: 'agent-b', trustScore: 0.61, workloadScore: 0.9,
    phase: 'EXCLUDED_PHASE2B', observations: 12, finalScore: 0, exclusionReason: 'below threshold' },
  { workerId: 'agent-c', trustScore: null, workloadScore: 1.0,
    phase: 'BOOTSTRAP', observations: 3, finalScore: 1.0 },
];

const SAMPLE_DATA: RoutingRationaleData = {
  capabilityTag: 'code-review', strategyId: 'trust-weighted',
  selected: SELECTED, alternatives: ALTERNATIVES, policy: POLICY,
};

let originalFetch: typeof globalThis.fetch;

describe('routing-rationale', () => {
  let el: RoutingRationaleEl;

  beforeEach(() => {
    originalFetch = globalThis.fetch;
    el = document.createElement('routing-rationale') as RoutingRationaleEl;
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
    globalThis.fetch = originalFetch;
  });

  it('renders empty state when no data', async () => {
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('No routing data');
  });
});
```

- [ ] **Step 8: Run test to verify it fails**

Run: `cd components/routing-rationale && npx vitest run --reporter=verbose`
Expected: FAIL — module `./routing-rationale.js` not found or component not defined

- [ ] **Step 9: Create routing-rationale.ts — minimal shell**

```typescript
import { LitElement, html, css, type PropertyValues, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { DataSourceMixin, createTypedFetchSource, emitPagesEvent } from '@casehubio/blocks-ui-core';
import { LiveRegionMixin } from '@casehubio/pages-primitives';
import '@casehubio/pages-table';
import type { TableColumnConfig, ColumnRenderer } from '@casehubio/pages-table';
import type { SourceFactory } from '@casehubio/pages-component';
import { fromRows } from '@casehubio/pages-data/dist/dataset/conversion.js';
import { columnId, ColumnType } from '@casehubio/pages-data/dist/dataset/types.js';
import type { CellValue, ColumnId, TypedRow } from '@casehubio/pages-data/dist/dataset/types.js';
import type { RoutingRationaleData, CandidateScore, RoutingCandidateSelectedDetail } from './types.js';

export const RoutingRationaleTopics = {
  CANDIDATE_SELECTED: 'routing.candidate-selected',
} as const;

@customElement('routing-rationale')
export class RoutingRationale extends DataSourceMixin(LiveRegionMixin(LitElement)) {
  @property({ attribute: false }) data: RoutingRationaleData | null = null;
  @property({ type: String, attribute: 'score-label' }) scoreLabel = 'Trust Score';
  @property({ type: String, attribute: 'capability-label' }) capabilityLabel?: string;
  @property({ attribute: false }) renderCandidate?: (candidate: CandidateScore) => TemplateResult | undefined;

  @state() private _rawData: RoutingRationaleData | null = null;

  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui); }
    .empty { color: var(--pages-neutral-9, #888); font-style: italic; padding: var(--pages-space-4, 1rem); }
  `;

  override resolveEndpoint(): string | undefined {
    if (this.data) return undefined;
    return this.endpoint;
  }

  override createSourceFactory(): SourceFactory {
    return (url) => createTypedFetchSource<RoutingRationaleData>(url, (data, sink) => {
      this._rawData = data;
      const allCandidates = [data.selected, ...data.alternatives];
      const dataset = fromRows(allCandidates, CANDIDATE_COLUMNS(data.selected.workerId));
      sink.apply({ type: 'snapshot', dataset });
    });
  }

  override willUpdate(changed: PropertyValues): void {
    super.willUpdate(changed);
    if (changed.has('data') && this.data) {
      this._rawData = this.data;
      const allCandidates = [this.data.selected, ...this.data.alternatives];
      this.dataSet = fromRows(allCandidates, CANDIDATE_COLUMNS(this.data.selected.workerId));
    }
  }

  override render() {
    if (this.loading) return html`<div class="empty">Loading routing data...</div>`;
    if (this.error) return html`<div class="empty">Routing data unavailable</div>`;
    if (!this._rawData) return html`<div class="empty">No routing data</div>`;

    return html`<div>TODO</div>`;
  }
}

const CANDIDATE_COLUMNS = (_selectedId: string) => [];

declare global {
  interface HTMLElementTagNameMap {
    'routing-rationale': RoutingRationale;
  }
}
```

- [ ] **Step 10: Run test to verify it passes**

Run: `cd components/routing-rationale && npx vitest run --reporter=verbose`
Expected: PASS

- [ ] **Step 11: Install dependencies and verify typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn install && cd components/routing-rationale && npx tsc --noEmit`

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/routing-rationale/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(routing-rationale): package scaffold, types, and empty shell (#54)"
```

---

### Task 2: Dataset conversion + alternatives table

**Files:**
- Modify: `components/routing-rationale/src/routing-rationale.ts`
- Modify: `components/routing-rationale/src/routing-rationale.test.ts`

**Interfaces:**
- Consumes: `RoutingRationaleData`, `CandidateScore` from types.ts. `fromRows`, `columnId`, `ColumnType` from pages-data.
- Produces: `CANDIDATE_COLUMNS` column definition array, column renderers map, pages-table rendering in `render()`.

- [ ] **Step 1: Write failing test — pages-table renders with data**

Add to the describe block in routing-rationale.test.ts:

```typescript
  it('renders pages-table when data is provided', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    expect(table).toBeTruthy();
    await (table as unknown as { updateComplete: Promise<boolean> }).updateComplete;
    const rows = table.shadowRoot!.querySelectorAll('.row[role="row"]:not(.header)');
    expect(rows.length).toBe(3);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — pages-table not rendered (render returns `<div>TODO</div>`)

- [ ] **Step 3: Implement CANDIDATE_COLUMNS and table rendering**

Replace the placeholder `CANDIDATE_COLUMNS` with the real column definitions:

```typescript
const WORKER_COL = columnId('workerId');
const TRUST_COL = columnId('trustScore');
const WORKLOAD_COL = columnId('workloadScore');
const PHASE_COL = columnId('phase');
const OBS_COL = columnId('observations');
const FINAL_COL = columnId('finalScore');
const STATUS_COL = columnId('status');

const CANDIDATE_COLUMNS = (selectedId: string) => [
  { id: WORKER_COL, name: 'Worker', type: ColumnType.TEXT, getValue: (c: CandidateScore) => c.workerId },
  { id: TRUST_COL, name: 'Trust', type: ColumnType.NUMBER, getValue: (c: CandidateScore) => c.trustScore },
  { id: WORKLOAD_COL, name: 'Workload', type: ColumnType.NUMBER, getValue: (c: CandidateScore) => c.workloadScore },
  { id: PHASE_COL, name: 'Phase', type: ColumnType.TEXT, getValue: (c: CandidateScore) => c.phase },
  { id: OBS_COL, name: 'Observations', type: ColumnType.NUMBER, getValue: (c: CandidateScore) => c.observations },
  { id: FINAL_COL, name: 'Final Score', type: ColumnType.NUMBER, getValue: (c: CandidateScore) => c.finalScore },
  { id: STATUS_COL, name: 'Status', type: ColumnType.TEXT, getValue: (c: CandidateScore) => c.workerId === selectedId ? 'Selected' : c.exclusionReason ?? 'Eligible' },
];

const TABLE_CONFIG: readonly TableColumnConfig[] = [
  { id: WORKER_COL, sortable: true },
  { id: TRUST_COL, sortable: true },
  { id: WORKLOAD_COL, sortable: true },
  { id: PHASE_COL, sortable: true },
  { id: OBS_COL, sortable: true },
  { id: FINAL_COL, sortable: true },
  { id: STATUS_COL, sortable: false },
];
```

Update render to include the table:

```typescript
override render() {
    if (this.loading) return html`<div class="empty">Loading routing data...</div>`;
    if (this.error) return html`<div class="empty">Routing data unavailable</div>`;
    if (!this._rawData || !this.dataSet) return html`<div class="empty">No routing data</div>`;

    return html`
      <pages-table
        .dataSet=${this.dataSet}
        .columnConfig=${TABLE_CONFIG}
        .columnRenderers=${this._columnRenderers}
        @row-activate=${this._handleRowActivate}
      ></pages-table>
    `;
  }
```

Add the `_handleRowActivate` method and empty `_columnRenderers`:

```typescript
  private _columnRenderers: ReadonlyMap<ColumnId, ColumnRenderer> = new Map();

  private _handleRowActivate(e: CustomEvent) {
    const row = e.detail.row as TypedRow;
    const detail: RoutingCandidateSelectedDetail = {
      workerId: row.text(WORKER_COL),
      trustScore: (() => { const c = row.cell(TRUST_COL); return c.type === 'NULL' ? null : (c as { value: number }).value; })(),
      finalScore: row.number(FINAL_COL),
      phase: row.text(PHASE_COL) as CandidateScore['phase'],
    };
    emitPagesEvent(this, RoutingRationaleTopics.CANDIDATE_SELECTED, detail);
  }
```

- [ ] **Step 4: Run test to verify it passes**

Expected: PASS — pages-table renders with 3 rows

- [ ] **Step 5: Write failing test — status column shows correct values**

```typescript
  it('status column shows Selected for winner and exclusion reasons for others', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    await (table as unknown as { updateComplete: Promise<boolean> }).updateComplete;
    const cells = table.shadowRoot!.querySelectorAll('[role="gridcell"]');
    const cellTexts = Array.from(cells).map(c => c.textContent!.trim());
    expect(cellTexts).toContain('Selected');
    expect(cellTexts).toContain('below threshold');
    expect(cellTexts).toContain('Eligible');
  });
```

- [ ] **Step 6: Run test to verify it passes**

Expected: PASS — status column values come from `CANDIDATE_COLUMNS` getValue

- [ ] **Step 7: Write failing test — null trustScore renders as NULL cell**

```typescript
  it('null trustScore produces NULL cell (not zero)', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    await (table as unknown as { updateComplete: Promise<boolean> }).updateComplete;
    const cells = table.shadowRoot!.querySelectorAll('[role="gridcell"]');
    const cellTexts = Array.from(cells).map(c => c.textContent!.trim());
    expect(cellTexts).not.toContain('0%');
  });
```

- [ ] **Step 8: Run test — verify passes (fromRows with null getValue passes null through)**

- [ ] **Step 9: Write failing test — emits candidate-selected event**

```typescript
  it('emits routing.candidate-selected on row activation', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const handler = vi.fn();
    document.addEventListener('pages-event', handler);
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    table.dispatchEvent(new CustomEvent('row-activate', {
      bubbles: true, composed: true,
      detail: {
        row: {
          text: (col: unknown) => { const id = String(col); return id.includes('workerId') ? 'agent-a' : id.includes('phase') ? 'QUALIFIED' : ''; },
          number: (col: unknown) => String(col).includes('finalScore') ? 0.812 : 0,
          cell: (col: unknown) => String(col).includes('trustScore') ? { type: 'NUMBER' as const, value: 0.82 } : { type: 'NULL' as const },
        },
      },
    }));
    const event = handler.mock.calls.find((c: unknown[]) => (c[0] as CustomEvent).detail.topic === 'routing.candidate-selected');
    expect(event).toBeTruthy();
    document.removeEventListener('pages-event', handler);
  });
```

- [ ] **Step 10: Run test — verify passes**

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/routing-rationale/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(routing-rationale): dataset conversion, alternatives table, event emission (#54)"
```

---

### Task 3: Column renderers (inline-styled score bars, phase badges, status)

**Files:**
- Modify: `components/routing-rationale/src/routing-rationale.ts`
- Modify: `components/routing-rationale/src/routing-rationale.test.ts`

**Interfaces:**
- Consumes: `TRUST_COL`, `WORKLOAD_COL`, `PHASE_COL`, `FINAL_COL`, `STATUS_COL` column IDs. `_rawData.policy.threshold`, `_rawData.policy.borderlineMargin` for threshold markers.
- Produces: `_columnRenderers` Map with inline-styled renderers for all 7 columns.

- [ ] **Step 1: Write failing test — trust column uses inline-styled bar**

```typescript
  it('trust column renderer uses inline styles with threshold marker', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    await (table as unknown as { updateComplete: Promise<boolean> }).updateComplete;
    const cells = table.shadowRoot!.querySelectorAll('[role="gridcell"]');
    const cellContents = Array.from(cells).map(c => c.innerHTML);
    const trustCell = cellContents.find(h => h.includes('82%'));
    expect(trustCell).toBeDefined();
    expect(trustCell).toContain('style=');
    expect(trustCell).not.toMatch(/class="[^"]*bar/);
  });
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `_columnRenderers` is empty, trust column shows raw number

- [ ] **Step 3: Implement column renderers**

Build the `_columnRenderers` Map. Each renderer uses inline styles. The trust column includes a threshold marker. The phase column uses phase-specific colours. The status column uses Selected/Eligible/exclusion badges.

Key: renderers must access `this._rawData` for threshold/margin values. Since `_columnRenderers` is a class field, it can't reference `this`. Instead, build the map in a getter or rebuild it in `willUpdate` when `_rawData` changes:

```typescript
  @state() private _renderers: ReadonlyMap<ColumnId, ColumnRenderer> = new Map();

  override willUpdate(changed: PropertyValues): void {
    super.willUpdate(changed);
    if (changed.has('data') && this.data) {
      this._rawData = this.data;
      const allCandidates = [this.data.selected, ...this.data.alternatives];
      this.dataSet = fromRows(allCandidates, CANDIDATE_COLUMNS(this.data.selected.workerId));
    }
    if (this._rawData && (changed.has('data') || changed.has('_rawData'))) {
      this._renderers = buildColumnRenderers(this._rawData.policy, this._rawData.selected.workerId, this.renderCandidate);
    }
  }
```

Then implement `buildColumnRenderers(policy, selectedId, renderCandidate)` as a module-level function returning `ReadonlyMap<ColumnId, ColumnRenderer>`. Each renderer:

- **TRUST_COL**: bar 0–1 with threshold marker line and borderline margin band. NULL → "—".
- **WORKLOAD_COL**: simple bar 0–1.
- **PHASE_COL**: badge with phase-specific background/text from the Phase Badge Styles table.
- **FINAL_COL**: bar 0–1, no threshold marker.
- **STATUS_COL**: "Selected" green badge, "Eligible" neutral badge, or exclusion reason text.
- **WORKER_COL**: if `renderCandidate` provided, attempt it; fall back to plain text.

All use inline styles. Example for trust bar:

```typescript
[TRUST_COL, (cell: CellValue) => {
  if (cell.type === 'NULL') return html`<span style="color: var(--pages-neutral-9, #888);">—</span>`;
  const value = (cell as { value: number }).value;
  const pct = Math.round(value * 100);
  const threshPct = Math.round(policy.threshold * 100);
  const marginPct = Math.round(policy.borderlineMargin * 100);
  return html`
    <div style="display: flex; align-items: center; gap: 0.5rem; position: relative;" role="img" aria-label="${scoreLabel} ${pct}% of 100%, threshold ${threshPct}%">
      <div style="flex: 1; height: 8px; background: var(--pages-neutral-4, #e5e5e5); border-radius: 4px; overflow: hidden; position: relative;">
        <div style="position: absolute; left: ${threshPct - marginPct}%; width: ${marginPct * 2}%; height: 100%; background: var(--pages-warning-3, #fff3cd); opacity: 0.5;" aria-hidden="true"></div>
        <div style="height: 100%; width: ${pct}%; background: var(--pages-accent-9, #3b82f6); position: relative; z-index: 1;"></div>
        <div style="position: absolute; left: ${threshPct}%; top: -2px; width: 2px; height: 12px; background: var(--pages-neutral-11, #333);" aria-hidden="true"></div>
      </div>
      <span style="font-weight: 600; min-width: 35px; font-size: 13px;">${pct}%</span>
    </div>
  `;
}],
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write failing test — phase badge uses correct colours**

```typescript
  it('phase badge renders with correct inline colours per phase', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    await (table as unknown as { updateComplete: Promise<boolean> }).updateComplete;
    const cells = table.shadowRoot!.querySelectorAll('[role="gridcell"]');
    const cellContents = Array.from(cells).map(c => c.innerHTML);
    const qualifiedBadge = cellContents.find(h => h.includes('QUALIFIED'));
    expect(qualifiedBadge).toContain('--pages-success-3');
    const excludedBadge = cellContents.find(h => h.includes('EXCLUDED'));
    expect(excludedBadge).toContain('--pages-danger-3');
    const bootstrapBadge = cellContents.find(h => h.includes('BOOTSTRAP'));
    expect(bootstrapBadge).toContain('--pages-neutral-3');
  });
```

- [ ] **Step 6: Run test — verify passes**

- [ ] **Step 7: Write failing test — null trustScore shows dash, not zero bar**

```typescript
  it('null trustScore renders dash instead of zero-width bar', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const table = el.shadowRoot!.querySelector('pages-table') as HTMLElement;
    await (table as unknown as { updateComplete: Promise<boolean> }).updateComplete;
    const cells = table.shadowRoot!.querySelectorAll('[role="gridcell"]');
    const trustCells = Array.from(cells).filter((_, i) => i % 7 === 1);
    const bootstrapTrust = trustCells[2];
    expect(bootstrapTrust?.textContent?.trim()).toBe('—');
  });
```

- [ ] **Step 8: Run test — verify passes**

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/routing-rationale/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(routing-rationale): inline-styled column renderers — bars, badges, status (#54)"
```

---

### Task 4: Score Header + Component Title + Policy Summary

**Files:**
- Modify: `components/routing-rationale/src/routing-rationale.ts`
- Modify: `components/routing-rationale/src/routing-rationale.test.ts`

**Interfaces:**
- Consumes: `_rawData` (RoutingRationaleData), `scoreLabel`, `capabilityLabel`, `renderCandidate`.
- Produces: `_renderTitle()`, `_renderScoreHeader()`, `_renderPolicySummary()` methods.

- [ ] **Step 1: Write failing test — component title displays capability**

```typescript
  it('displays capability tag as component title', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('code-review');
  });

  it('capabilityLabel overrides capabilityTag in title', async () => {
    el.capabilityLabel = 'Code Review';
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Code Review');
    expect(el.shadowRoot!.textContent).not.toContain('code-review');
  });
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement _renderTitle()**

```typescript
  private _renderTitle() {
    const name = this.capabilityLabel ?? this._rawData?.capabilityTag;
    if (!name) return '';
    return html`<div style="font-size: 14px; font-weight: 600; color: var(--pages-neutral-12, #171717); padding: var(--pages-space-3, 12px) 0;">Routing: ${name}</div>`;
  }
```

Add to render(): `${this._renderTitle()}`

- [ ] **Step 4: Run test — verify passes**

- [ ] **Step 5: Write failing test — score header shows trust bar and final score bar**

```typescript
  it('score header shows selected candidate trust score and final score', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const header = el.shadowRoot!.querySelector('[data-section="score-header"]');
    expect(header).toBeTruthy();
    expect(header!.textContent).toContain('82%');
    expect(header!.textContent).toContain('agent-a');
    expect(header!.textContent).toContain('QUALIFIED');
  });
```

- [ ] **Step 6: Run test to verify it fails**

- [ ] **Step 7: Implement _renderScoreHeader()**

Renders selected candidate info: worker ID (or `renderCandidate` output), trust score bar with threshold + margin, final score bar, phase badge, observation count, workload score, additional scores, and optional rationale text. For BOOTSTRAP, show "No trust data — availability routing" instead of trust bar.

Add to render(): `${this._renderScoreHeader()}`

- [ ] **Step 8: Run test — verify passes**

- [ ] **Step 9: Write failing test — BOOTSTRAP candidate omits trust bar**

```typescript
  it('bootstrap selected candidate shows no-trust-data message', async () => {
    const bootstrapData: RoutingRationaleData = {
      ...SAMPLE_DATA,
      selected: { workerId: 'agent-c', trustScore: null, workloadScore: 1.0, phase: 'BOOTSTRAP', observations: 3, finalScore: 1.0 },
      alternatives: [SELECTED],
    };
    el.data = bootstrapData;
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('No trust data');
  });
```

- [ ] **Step 10: Run test — verify passes**

- [ ] **Step 11: Write failing test — policy summary shows all fields**

```typescript
  it('policy summary shows threshold, margin, blend, and min observations', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const summary = el.shadowRoot!.querySelector('[data-section="policy-summary"]');
    expect(summary).toBeTruthy();
    expect(summary!.textContent).toContain('0.70');
    expect(summary!.textContent).toContain('0.10');
    expect(summary!.textContent).toContain('60%');
    expect(summary!.textContent).toContain('10');
    expect(summary!.textContent).toContain('trust-weighted');
  });
```

- [ ] **Step 12: Run test to verify it fails**

- [ ] **Step 13: Implement _renderPolicySummary()**

Renders strategy ID, threshold, margin, blend factor, min observations, CBR weight. Shows quality floors line when non-empty. Shows bootstrap escalation badge when true.

Add to render(): `${this._renderPolicySummary()}`

- [ ] **Step 14: Run test — verify passes**

- [ ] **Step 15: Write failing test — quality floors display when present**

```typescript
  it('shows quality floors when present', async () => {
    const dataWithFloors = {
      ...SAMPLE_DATA,
      policy: { ...POLICY, qualityFloors: { accuracy: 0.8, completeness: 0.7 } },
    };
    el.data = dataWithFloors;
    await el.updateComplete;
    const summary = el.shadowRoot!.querySelector('[data-section="policy-summary"]');
    expect(summary!.textContent).toContain('accuracy');
    expect(summary!.textContent).toContain('0.80');
  });
```

- [ ] **Step 16: Run test — verify passes**

- [ ] **Step 17: Write failing test — additional scores in score header**

```typescript
  it('shows additional scores in score header when present', async () => {
    const dataWithExtra = {
      ...SAMPLE_DATA,
      selected: { ...SELECTED, additionalScores: { semanticScore: 0.78 } },
    };
    el.data = dataWithExtra;
    await el.updateComplete;
    const header = el.shadowRoot!.querySelector('[data-section="score-header"]');
    expect(header!.textContent).toContain('Semantic');
    expect(header!.textContent).toContain('0.78');
  });
```

- [ ] **Step 18: Run test — verify passes**

- [ ] **Step 19: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/routing-rationale/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(routing-rationale): score header, component title, policy summary (#54)"
```

---

### Task 5: Dual-mode lifecycle + loading/error states + accessibility + scoreLabel

**Files:**
- Modify: `components/routing-rationale/src/routing-rationale.ts`
- Modify: `components/routing-rationale/src/routing-rationale.test.ts`

**Interfaces:**
- Consumes: DataSourceMixin lifecycle (`resolveEndpoint`, `createSourceFactory`, `willUpdate`), LiveRegionMixin.
- Produces: Complete dual-mode data lifecycle, loading/error state rendering, ARIA labels on all visual elements.

- [ ] **Step 1: Write failing test — data property suppresses fetch**

```typescript
  it('suppresses fetch when data prop is set', async () => {
    const mockFetch = vi.fn();
    globalThis.fetch = mockFetch as unknown as typeof fetch;
    el.data = SAMPLE_DATA;
    el.endpoint = 'http://test.local/api/routing';
    await el.updateComplete;
    expect(mockFetch).not.toHaveBeenCalled();
  });
```

- [ ] **Step 2: Run test — verify passes** (already implemented in Task 1)

- [ ] **Step 3: Write failing test — fetches from endpoint**

```typescript
  it('fetches from endpoint when no data prop', async () => {
    const mockFetch = vi.fn().mockResolvedValue(
      new Response(JSON.stringify(SAMPLE_DATA), { status: 200 })
    );
    globalThis.fetch = mockFetch as unknown as typeof fetch;
    el.endpoint = 'http://test.local/api/routing';
    await el.updateComplete;
    await vi.waitFor(() => expect(mockFetch).toHaveBeenCalled());
  });
```

- [ ] **Step 4: Run test — verify passes**

- [ ] **Step 5: Write failing test — loading state**

```typescript
  it('shows loading state during fetch', async () => {
    globalThis.fetch = vi.fn(() => new Promise(() => {})) as unknown as typeof fetch;
    el.endpoint = 'http://test.local/api/routing';
    await el.updateComplete;
    await vi.waitFor(() => expect(el.loading).toBe(true));
    expect(el.shadowRoot!.textContent).toContain('Loading');
  });
```

- [ ] **Step 6: Run test — verify passes**

- [ ] **Step 7: Write failing test — error state**

```typescript
  it('shows error state on fetch failure', async () => {
    globalThis.fetch = vi.fn().mockResolvedValue(new Response('', { status: 500 })) as unknown as typeof fetch;
    el.endpoint = 'http://test.local/api/routing';
    await el.updateComplete;
    await vi.waitFor(() => expect(el.shadowRoot!.textContent).toContain('unavailable'));
  });
```

- [ ] **Step 8: Run test — verify passes**

- [ ] **Step 9: Write failing test — scoreLabel on trust column header**

```typescript
  it('scoreLabel appears on trust score bar label', async () => {
    el.scoreLabel = 'Capability Score';
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Capability Score');
  });
```

- [ ] **Step 10: Run test — verify passes**

- [ ] **Step 11: Write failing test — ARIA on score bars**

```typescript
  it('score bars have role="img" with descriptive aria-label', async () => {
    el.data = SAMPLE_DATA;
    await el.updateComplete;
    const imgs = el.shadowRoot!.querySelectorAll('[role="img"]');
    expect(imgs.length).toBeGreaterThan(0);
    const labels = Array.from(imgs).map(i => i.getAttribute('aria-label'));
    expect(labels.some(l => l?.includes('82%'))).toBe(true);
    expect(labels.some(l => l?.includes('threshold'))).toBe(true);
  });
```

- [ ] **Step 12: Run test — verify passes**

- [ ] **Step 13: Run full test suite for routing-rationale**

Run: `cd components/routing-rationale && npx vitest run --reporter=verbose`
Expected: ALL PASS

- [ ] **Step 14: Typecheck**

Run: `cd components/routing-rationale && npx tsc --noEmit`
Expected: Clean

- [ ] **Step 15: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/routing-rationale/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(routing-rationale): dual-mode lifecycle, loading/error, accessibility, scoreLabel (#54)"
```

---

### Task 6: CLAUDE.md update + final verification

**Files:**
- Modify: `CLAUDE.md` (add routing-rationale to Key Directories table)

**Interfaces:**
- Consumes: Complete routing-rationale component from Tasks 1–5.

- [ ] **Step 1: Add routing-rationale to CLAUDE.md Key Directories**

Add after the `grouped-data-view` entry:

```
| `components/routing-rationale/` | Routing rationale — trust-weighted assignment explanation: score vs threshold, alternatives table, phase badges, policy summary. DataSourceMixin, inline-styled column renderers, renderCandidate callback. |
```

- [ ] **Step 2: Run all three affected test suites**

```bash
cd /Users/mdproctor/claude/casehub/blocks-ui/components/similarity-panel && npx vitest run
cd /Users/mdproctor/claude/casehub/blocks-ui/components/compliance-summary && npx vitest run
cd /Users/mdproctor/claude/casehub/blocks-ui/components/routing-rationale && npx vitest run
```

Expected: ALL PASS

- [ ] **Step 3: Typecheck all modified components**

```bash
cd /Users/mdproctor/claude/casehub/blocks-ui
npx tsc --noEmit -p components/similarity-panel/tsconfig.json
npx tsc --noEmit -p components/compliance-summary/tsconfig.json
npx tsc --noEmit -p components/routing-rationale/tsconfig.json
```

Expected: Clean

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "docs: add routing-rationale to CLAUDE.md key directories (#54)"
```
