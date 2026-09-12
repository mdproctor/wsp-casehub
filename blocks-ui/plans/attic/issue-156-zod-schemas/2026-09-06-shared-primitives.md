# Shared Rendering Primitives Extraction — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #117 — Extract shared debate/conversation rendering primitives to blocks-ui-core
**Issue group:** #117

**Goal:** Extract 6 duplicated rendering primitives into `blocks-ui-core/src/rendering/` and migrate all 10+ consumers to use the shared versions.

**Architecture:** New `rendering/` module in blocks-ui-core exports pure functions (`formatTimestamp`, `statusBorderColour`) and Lit CSS tagged templates (`entryCardStyles`, `badgeStyles`, `roundDividerStyles`, `selectedHighlightStyles`). Consumers import and compose via Lit's `static styles` array and direct function calls. No new custom elements.

**Tech Stack:** Lit 3, TypeScript, vitest, Lit `css` tagged templates

## Global Constraints

- No capability loss — all existing rendering preserved after migration
- No forced migration — shared primitives are additive (issue constraint)
- Render callbacks preserved per PP-20260713-8ea1af
- CSS uses `--pages-*` custom properties with fallback hex values
- `DebateStreamEntry` / `ConversationEntry` type unification is out of scope

---

## Batch 1: Create shared primitives in blocks-ui-core

### Task 1: formatTimestamp function + statusBorderColour

**Files:**
- Create: `packages/blocks-ui-core/src/rendering/format-timestamp.ts`
- Create: `packages/blocks-ui-core/src/rendering/status-colours.ts`
- Create: `packages/blocks-ui-core/src/rendering/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`
- Test: `packages/blocks-ui-core/src/rendering/format-timestamp.test.ts`
- Test: `packages/blocks-ui-core/src/rendering/status-colours.test.ts`

**Interfaces:**
- Produces: `formatTimestamp(iso: string, options?: { style?: 'conversational' | 'compact' }): string`, `statusBorderColour(category: EntryCategory): string`, `EntryCategory` type — consumed by all subsequent tasks

- [ ] **Step 1: Write failing tests for formatTimestamp**

```typescript
import { describe, it, expect } from 'vitest';
import { formatTimestamp } from './format-timestamp.js';

describe('formatTimestamp', () => {
  const now = Date.now();
  const iso = (offsetMs: number) => new Date(now - offsetMs).toISOString();

  describe('conversational (default)', () => {
    it('returns "just now" for < 1 minute', () => {
      expect(formatTimestamp(iso(30_000))).toBe('just now');
    });
    it('returns "Xm ago" for < 60 minutes', () => {
      expect(formatTimestamp(iso(5 * 60_000))).toBe('5m ago');
    });
    it('returns "Xh ago" for < 24 hours', () => {
      expect(formatTimestamp(iso(3 * 3600_000))).toBe('3h ago');
    });
    it('returns locale date+time for >= 24 hours', () => {
      const result = formatTimestamp(iso(48 * 3600_000));
      expect(result).not.toContain('ago');
      expect(result.length).toBeGreaterThan(5);
    });
  });

  describe('compact', () => {
    it('returns "now" for < 1 minute', () => {
      expect(formatTimestamp(iso(30_000), { style: 'compact' })).toBe('now');
    });
    it('returns "Xm" for < 60 minutes', () => {
      expect(formatTimestamp(iso(5 * 60_000), { style: 'compact' })).toBe('5m');
    });
    it('returns "Xh" for < 24 hours', () => {
      expect(formatTimestamp(iso(3 * 3600_000), { style: 'compact' })).toBe('3h');
    });
    it('returns "Xd" for >= 24 hours', () => {
      expect(formatTimestamp(iso(3 * 86400_000), { style: 'compact' })).toBe('3d');
    });
  });

  it('returns empty string for empty input', () => {
    expect(formatTimestamp('')).toBe('');
  });

  it('returns empty string for undefined-like input', () => {
    expect(formatTimestamp(undefined as any)).toBe('');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd packages/blocks-ui-core && npx vitest run --reporter=verbose`
Expected: FAIL — module not found

- [ ] **Step 3: Implement formatTimestamp**

```typescript
// format-timestamp.ts
export type TimestampStyle = 'conversational' | 'compact';

export interface FormatTimestampOptions {
  style?: TimestampStyle;
}

export function formatTimestamp(iso: string, options?: FormatTimestampOptions): string {
  if (!iso) return '';
  const style = options?.style ?? 'conversational';
  const elapsed = Date.now() - new Date(iso).getTime();
  const minutes = Math.floor(elapsed / 60_000);
  const hours = Math.floor(elapsed / 3_600_000);
  const days = Math.floor(elapsed / 86_400_000);

  if (style === 'compact') {
    if (minutes < 1) return 'now';
    if (minutes < 60) return `${minutes}m`;
    if (hours < 24) return `${hours}h`;
    return `${days}d`;
  }

  if (minutes < 1) return 'just now';
  if (minutes < 60) return `${minutes}m ago`;
  if (hours < 24) return `${hours}h ago`;
  const d = new Date(iso);
  return d.toLocaleDateString(undefined, { month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit' });
}
```

- [ ] **Step 4: Write failing tests for statusBorderColour**

```typescript
import { describe, it, expect } from 'vitest';
import { statusBorderColour } from './status-colours.js';

describe('statusBorderColour', () => {
  it('returns neutral colour', () => {
    expect(statusBorderColour('neutral')).toContain('--pages-neutral-12');
  });
  it('returns success colour', () => {
    expect(statusBorderColour('success')).toContain('--pages-success-9');
  });
  it('returns warning colour', () => {
    expect(statusBorderColour('warning')).toContain('--pages-warning-9');
  });
  it('returns error colour', () => {
    expect(statusBorderColour('error')).toContain('--pages-error-9');
  });
  it('returns accent colour', () => {
    expect(statusBorderColour('accent')).toContain('--pages-accent-9');
  });
  it('returns neutral for unknown category', () => {
    expect(statusBorderColour('unknown' as any)).toContain('--pages-neutral-12');
  });
});
```

- [ ] **Step 5: Implement statusBorderColour**

```typescript
// status-colours.ts
export type EntryCategory = 'neutral' | 'success' | 'warning' | 'error' | 'accent';

const COLOURS: Record<EntryCategory, string> = {
  neutral: 'var(--pages-neutral-12, #111)',
  success: 'var(--pages-success-9, #16a34a)',
  warning: 'var(--pages-warning-9, #d97706)',
  error: 'var(--pages-error-9, #dc2626)',
  accent: 'var(--pages-accent-9, #6366f1)',
};

export function statusBorderColour(category: EntryCategory): string {
  return COLOURS[category] ?? COLOURS.neutral;
}
```

- [ ] **Step 6: Create rendering/index.ts barrel + update blocks-ui-core index**

```typescript
// rendering/index.ts
export { formatTimestamp } from './format-timestamp.js';
export type { TimestampStyle, FormatTimestampOptions } from './format-timestamp.js';
export { statusBorderColour } from './status-colours.js';
export type { EntryCategory } from './status-colours.js';
```

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './rendering/index.js';
```

- [ ] **Step 7: Run all tests**

Run: `cd packages/blocks-ui-core && npx vitest run --reporter=verbose`
Expected: All PASS

- [ ] **Step 8: Commit**

```bash
git add packages/blocks-ui-core/src/rendering/
git commit -m "feat(blocks-ui-core): add formatTimestamp and statusBorderColour rendering utilities Refs #117"
```

### Task 2: Shared CSS tagged templates

**Files:**
- Create: `packages/blocks-ui-core/src/rendering/entry-card-styles.ts`
- Create: `packages/blocks-ui-core/src/rendering/badge-styles.ts`
- Create: `packages/blocks-ui-core/src/rendering/round-divider-styles.ts`
- Create: `packages/blocks-ui-core/src/rendering/selected-highlight-styles.ts`
- Modify: `packages/blocks-ui-core/src/rendering/index.ts`

**Interfaces:**
- Produces: `entryCardStyles`, `badgeStyles`, `roundDividerStyles`, `selectedHighlightStyles` (all `CSSResult`) — consumed by Tasks 3-5

- [ ] **Step 1: Create entry-card-styles.ts**

```typescript
import { css } from 'lit';

export const entryCardStyles = css`
  .entry-card {
    padding: 8px 12px;
    border: 1px solid var(--pages-neutral-4, #d4d4d4);
    border-left-width: 3px;
    border-radius: var(--pages-radius-sm, 4px);
    background: var(--pages-neutral-1, #fafafa);
  }
  .entry-header {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 11px;
    color: var(--pages-neutral-9, #525252);
    margin-bottom: 4px;
  }
  .entry-agent { font-weight: 600; color: var(--pages-neutral-12, #111); }
  .entry-type { text-transform: uppercase; font-size: 9px; letter-spacing: 0.3px; }
  .entry-timestamp { margin-left: auto; font-size: 10px; }
  .entry-content {
    color: var(--pages-neutral-12, #111);
    font-size: 13px;
    line-height: 1.5;
    white-space: pre-wrap;
    word-wrap: break-word;
  }
`;
```

- [ ] **Step 2: Create badge-styles.ts**

```typescript
import { css } from 'lit';

export const badgeStyles = css`
  .badge {
    display: inline-block;
    padding: 1px 6px;
    border-radius: 2px;
    font-size: 9px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.3px;
  }
  .badge-priority { background: var(--pages-neutral-3, #e5e5e5); color: var(--pages-neutral-8, #9ca3af); }
  .badge-priority-high { background: var(--pages-error-2, #fee2e2); color: var(--pages-error-9, #dc2626); }
  .badge-priority-medium { background: var(--pages-warning-2, #fef3c7); color: var(--pages-warning-9, #d97706); }
  .badge-scope { background: var(--pages-accent-2, #e0e7ff); color: var(--pages-accent-9, #6366f1); border: 1px solid var(--pages-accent-9, #6366f1); }
  .badge-location {
    background: var(--pages-neutral-2, #f5f5f5);
    color: var(--pages-neutral-11, #333);
    border: 1px solid var(--pages-neutral-5, #d4d4d4);
    font-family: SFMono-Regular, Consolas, 'Liberation Mono', Menlo, monospace;
  }
`;
```

- [ ] **Step 3: Create round-divider-styles.ts**

```typescript
import { css } from 'lit';

export const roundDividerStyles = css`
  .round-divider {
    margin: 16px 0 8px;
    padding: 4px 10px;
    border-bottom: 1px solid var(--pages-neutral-5, #d4d4d4);
    color: var(--pages-neutral-8, #9ca3af);
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
  .round-divider:first-child { margin-top: 0; }
`;
```

- [ ] **Step 4: Create selected-highlight-styles.ts**

```typescript
import { css } from 'lit';

export const selectedHighlightStyles = css`
  .selected {
    outline: 2px solid var(--pages-accent-9, #6366f1);
    outline-offset: -2px;
    background: rgba(99, 102, 241, 0.08);
  }
`;
```

- [ ] **Step 5: Update rendering/index.ts with CSS exports**

Add to the barrel:
```typescript
export { entryCardStyles } from './entry-card-styles.js';
export { badgeStyles } from './badge-styles.js';
export { roundDividerStyles } from './round-divider-styles.js';
export { selectedHighlightStyles } from './selected-highlight-styles.js';
```

- [ ] **Step 6: Verify typecheck passes**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 7: Commit**

```bash
git add packages/blocks-ui-core/src/rendering/
git commit -m "feat(blocks-ui-core): add shared CSS tagged templates for entry card, badges, dividers, highlights Refs #117"
```

---

## Batch 2: Migrate debate/conversation components (CSS + formatTimestamp)

### Task 3: Migrate debate-feed + review-tracker + selection-threads

**Files:**
- Modify: `components/document-workbench/src/debate-feed.ts` — remove ~80 lines duplicated CSS + `_formatTimestamp`, import shared
- Modify: `components/document-workbench/src/review-tracker.ts` — remove selected highlight CSS, import shared
- Modify: `components/document-workbench/src/selection-threads.ts` — remove `_formatTimestamp`, import shared

**Interfaces:**
- Consumes: `formatTimestamp`, `entryCardStyles`, `badgeStyles`, `roundDividerStyles`, `selectedHighlightStyles`, `statusBorderColour` from Task 1/2

- [ ] **Step 1: Migrate debate-feed.ts**

1. Add import: `import { formatTimestamp, statusBorderColour, entryCardStyles, badgeStyles, roundDividerStyles } from '@casehubio/blocks-ui-core';`
2. Replace `static styles = css\`...\`` with `static styles = [entryCardStyles, badgeStyles, roundDividerStyles, css\`/* component-specific only */\`];`
3. Remove the duplicated `.entry`, `.entry-header`, `.entry-content`, `.badge`, `.round-divider` CSS rules (~80 lines)
4. Remove the `_formatTimestamp` method
5. Replace calls: `this._formatTimestamp(ts)` → `formatTimestamp(ts)`
6. Replace entry-type CSS classes (`.entry-raise`, `.entry-agree`) with inline `style="border-left-color: ${statusBorderColour(category)}"` on entry cards
7. Keep component-specific CSS (layout, scrolling, input area)

- [ ] **Step 2: Migrate review-tracker.ts**

1. Add import: `import { selectedHighlightStyles, statusBorderColour } from '@casehubio/blocks-ui-core';`
2. Add `selectedHighlightStyles` to static styles array
3. Remove duplicated `.point-item.selected` and hover CSS (~6 lines)
4. Replace status border colour CSS classes with `statusBorderColour()` calls

- [ ] **Step 3: Migrate selection-threads.ts**

1. Add import: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`
2. Remove `_formatTimestamp` method
3. Replace calls: `this._formatTimestamp(ts)` → `formatTimestamp(ts)`

- [ ] **Step 4: Run document-workbench tests**

Run: `cd components/document-workbench && npx vitest run --reporter=verbose`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
git add components/document-workbench/src/
git commit -m "refactor(document-workbench): use shared rendering primitives from blocks-ui-core Refs #117"
```

### Task 4: Migrate conversation-viewer components

**Files:**
- Modify: `components/conversation-viewer/src/blocks-point-detail.ts` — remove entry card CSS + `_formatTimestamp` + `ENTRY_BORDER_COLOURS`, import shared
- Modify: `components/conversation-viewer/src/blocks-point-list.ts` — remove badge/divider/highlight CSS + `BORDER_COLOURS`, import shared

**Interfaces:**
- Consumes: `formatTimestamp`, `entryCardStyles`, `badgeStyles`, `roundDividerStyles`, `selectedHighlightStyles`, `statusBorderColour` from Task 1/2

- [ ] **Step 1: Migrate blocks-point-detail.ts**

1. Add import: `import { formatTimestamp, statusBorderColour, entryCardStyles, badgeStyles } from '@casehubio/blocks-ui-core';`
2. Add `entryCardStyles, badgeStyles` to static styles array
3. Remove duplicated CSS (~30 lines)
4. Remove `_formatTimestamp` method
5. Remove `ENTRY_BORDER_COLOURS` map
6. Replace: `ENTRY_BORDER_COLOURS[type]` → `statusBorderColour(this._categoryFromType(type))`
7. Add helper: `_categoryFromType(type: string): EntryCategory` mapping entry types to categories (RAISE→neutral, AGREE→success, COUNTER→warning, DISPUTE→error, QUALIFY→accent)

- [ ] **Step 2: Migrate blocks-point-list.ts**

1. Add import: `import { statusBorderColour, badgeStyles, roundDividerStyles, selectedHighlightStyles } from '@casehubio/blocks-ui-core';`
2. Add shared styles to static styles array
3. Remove duplicated CSS (~40 lines)
4. Remove `BORDER_COLOURS` map
5. Replace border colour lookups with `statusBorderColour()` calls

- [ ] **Step 3: Run conversation-viewer tests**

Run: `cd components/conversation-viewer && npx vitest run --reporter=verbose`
Expected: All tests PASS

- [ ] **Step 4: Commit**

```bash
git add components/conversation-viewer/src/
git commit -m "refactor(conversation-viewer): use shared rendering primitives from blocks-ui-core Refs #117"
```

---

## Batch 3: Migrate remaining formatTimestamp consumers + cleanup

### Task 5: Migrate all remaining formatTimestamp copies

**Files:**
- Modify: `components/notification-inbox/src/notification-inbox.ts` — remove `relativeTime`, use `formatTimestamp(iso, { style: 'compact' })`
- Modify: `components/work-item-row/src/work-item-row.ts` — remove `relativeTime`, use `formatTimestamp(iso, { style: 'compact' })`
- Modify: `components/commitment-viz/src/commitment-transition-badge.ts` — remove `_formatRelativeTime`, use `formatTimestamp`
- Modify: `components/trust-workbench/src/columns.ts` — remove local `formatTimestamp`, import from core
- Modify: `components/blocks-timeline/src/renderers/horizontal.ts` — remove local `formatTimestamp`, import from core
- Modify: `components/blocks-timeline/src/renderers/compact.ts` — remove local `formatTimestamp`, import from core
- Modify: `components/blocks-timeline/src/renderers/vertical.ts` — remove local `formatTimestamp`, import from core
- Modify: `examples/src/pages/data-table-page.ts` — remove `relativeTime`, import from core

**Interfaces:**
- Consumes: `formatTimestamp` from Task 1

- [ ] **Step 1: Migrate notification-inbox.ts**

1. Add import: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`
2. Remove the `relativeTime` function definition
3. Replace all calls: `relativeTime(iso)` → `formatTimestamp(iso, { style: 'compact' })`

- [ ] **Step 2: Migrate work-item-row.ts**

1. Add import: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`
2. Remove the `relativeTime` function definition
3. Replace all calls: `relativeTime(iso)` → `formatTimestamp(iso, { style: 'compact' })`

- [ ] **Step 3: Migrate commitment-transition-badge.ts**

1. Add import: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`
2. Remove `_formatRelativeTime` method
3. Replace calls: `this._formatRelativeTime(iso)` → `formatTimestamp(iso)`

- [ ] **Step 4: Migrate trust-workbench/columns.ts**

1. Replace local `formatTimestamp` import/definition with: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`

- [ ] **Step 5: Migrate timeline renderers (3 files)**

For each of `horizontal.ts`, `compact.ts`, `vertical.ts`:
1. Remove local `formatTimestamp` function
2. Add import: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`

- [ ] **Step 6: Migrate examples/data-table-page.ts**

1. Replace local `relativeTime` with: `import { formatTimestamp } from '@casehubio/blocks-ui-core';`
2. Replace calls: `relativeTime(iso)` → `formatTimestamp(iso, { style: 'compact' })`

- [ ] **Step 7: Run tests for all migrated components**

```bash
cd components/notification-inbox && npx vitest run 2>&1 | tail -3
cd ../work-item-row && npx vitest run 2>&1 | tail -3
cd ../commitment-viz && npx vitest run 2>&1 | tail -3
cd ../blocks-timeline && npx vitest run 2>&1 | tail -3
```

Expected: All PASS

- [ ] **Step 8: Verify zero remaining local formatTimestamp/relativeTime copies**

Search with IntelliJ: `ide_search_text` for `_formatTimestamp\b|function formatTimestamp|function relativeTime|_formatRelativeTime` (regex) in `components/` — should return zero matches (only the shared version in `packages/blocks-ui-core/`).

- [ ] **Step 9: Commit**

```bash
git add components/ examples/
git commit -m "refactor: migrate all formatTimestamp/relativeTime copies to blocks-ui-core shared utility Refs #117"
```

### Task 6: Update CLAUDE.md + docs

**Files:**
- Modify: `CLAUDE.md` — update blocks-ui-core description to mention rendering utilities
- Modify: `ARC42STORIES.MD` — update §5 if blocks-ui-core description needs it

- [ ] **Step 1: Update CLAUDE.md**

Add to the `packages/blocks-ui-core/` description in Key Directories:
```
, rendering utilities (formatTimestamp, statusBorderColour, shared CSS: entryCardStyles, badgeStyles, roundDividerStyles, selectedHighlightStyles)
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update blocks-ui-core description with rendering utilities Refs #117"
```

## References

- [2026-09-06-shared-primitives-design.md](/Users/mdproctor/claude/public/casehub/blocks-ui/specs/issue-117-extract-shared-primitives/2026-09-06-shared-primitives-design.md) — design spec
- [decisions.md](/Users/mdproctor/claude/public/casehub/blocks-ui/specs/issue-117-extract-shared-primitives/decisions.md) — D1-D3
- [debate-feed.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench/src/debate-feed.ts) — primary source of duplicated CSS
- [blocks-point-detail.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/conversation-viewer/src/blocks-point-detail.ts) — entry card + formatTimestamp duplicate
- [blocks-point-list.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/conversation-viewer/src/blocks-point-list.ts) — badges + highlight duplicate
- [component-customisation-pattern (PP-20260713-8ea1af)](/Users/mdproctor/claude/casehub/blocks-ui/docs/protocols/blocks-ui/component-customisation-pattern.md) — render callbacks preserved
- [GitHub #117](https://github.com/casehubio/blocks-ui/issues/117) — focal issue
