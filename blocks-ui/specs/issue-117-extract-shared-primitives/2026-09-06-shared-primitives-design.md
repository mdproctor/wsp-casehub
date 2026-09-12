# Extract Shared Debate/Conversation Rendering Primitives — Design Spec

**Issue:** casehubio/blocks-ui#117
**Date:** 2026-09-06
**Branch:** issue-117-extract-shared-primitives

## Summary

Extract 6 duplicated rendering primitives from debate-feed, conversation-viewer, review-tracker, and other components into `blocks-ui-core/src/rendering/`. Pure functions exported as JS, CSS exported as Lit `css` tagged templates. Additionally unify all 10+ `formatTimestamp` copies across the codebase into one configurable function. No new custom elements. No capability loss.

## New module in blocks-ui-core

```
packages/blocks-ui-core/src/rendering/
├── index.ts
├── format-timestamp.ts
├── status-colours.ts
├── entry-card-styles.ts
├── badge-styles.ts
├── round-divider-styles.ts
└── selected-highlight-styles.ts
```

Exported from `blocks-ui-core`'s main `index.ts` via `export * from './rendering/index.js'`.

## API

### formatTimestamp

```typescript
export type TimestampStyle = 'conversational' | 'compact';

export interface FormatTimestampOptions {
  style?: TimestampStyle;
}

export function formatTimestamp(iso: string, options?: FormatTimestampOptions): string;
```

| Style | < 1 min | < 60 min | < 24 h | >= 24 h |
|-------|---------|----------|--------|---------|
| `conversational` (default) | "just now" | "Xm ago" | "Xh ago" | locale date + time |
| `compact` | "now" | "Xm" | "Xh" | "Xd" |

Handles undefined/empty input by returning `''`.

### statusBorderColour

```typescript
export type EntryCategory = 'neutral' | 'success' | 'warning' | 'error' | 'accent';

export function statusBorderColour(category: EntryCategory): string;
```

Returns CSS values using `--pages-*` custom properties with fallback hex values. Unifies the 4 different encoding mechanisms currently in use (CSS classes in debate-feed/review-tracker, JS maps in point-list/point-detail).

| Category | CSS value |
|----------|-----------|
| `neutral` | `var(--pages-neutral-12, #111)` |
| `success` | `var(--pages-success-9, #16a34a)` |
| `warning` | `var(--pages-warning-9, #d97706)` |
| `error` | `var(--pages-error-9, #dc2626)` |
| `accent` | `var(--pages-accent-9, #6366f1)` |

### Shared CSS

All exported as Lit `CSSResult` tagged templates. Components spread them into `static styles`:

```typescript
import { entryCardStyles, badgeStyles } from '@casehubio/blocks-ui-core';

static override styles = [entryCardStyles, badgeStyles, css`/* overrides */`];
```

#### entryCardStyles

```css
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
```

Normalises the trivial differences: padding (was 10/12 vs 8/10 — unified to 8/12), border-radius (was 3px vs 4px — unified to `--pages-radius-sm`).

#### badgeStyles

```css
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
  font-family: SFMono-Regular, Consolas, monospace;
}
```

Normalises padding (was 2px/6px vs 1px/5px — unified to 1px/6px). Includes all 3 priority variants from debate-feed.

#### roundDividerStyles

```css
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
```

Normalises margin (was 20/0/12 vs 12/0/4 — unified to 16/0/8).

#### selectedHighlightStyles

```css
.selected {
  outline: 2px solid var(--pages-accent-9, #6366f1);
  outline-offset: -2px;
  background: rgba(99, 102, 241, 0.08);
}
:host *:hover:not(.selected) {
  border-color: var(--pages-accent-9, #6366f1);
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}
```

Exact duplicate across review-tracker and point-list — no normalisation needed.

## Migration map

### formatTimestamp (10+ copies → 1)

| Component | Current function | Migration |
|-----------|-----------------|-----------|
| `debate-feed.ts` | `_formatTimestamp` | `import { formatTimestamp } from '@casehubio/blocks-ui-core'` — default style |
| `blocks-point-detail.ts` | `_formatTimestamp` | Same — default style |
| `selection-threads.ts` | `_formatTimestamp` | Same — default style, handle optional input |
| `trust-workbench/columns.ts` | `formatTimestamp` | Same — default style |
| `notification-inbox.ts` | `relativeTime` | `formatTimestamp(iso, { style: 'compact' })` |
| `work-item-row.ts` | `relativeTime` | `formatTimestamp(iso, { style: 'compact' })` |
| `commitment-transition-badge.ts` | `_formatRelativeTime` | `formatTimestamp(iso)` — default style |
| `blocks-timeline/horizontal.ts` | `formatTimestamp` | `import { formatTimestamp } from '@casehubio/blocks-ui-core'` |
| `blocks-timeline/compact.ts` | `formatTimestamp` | Same |
| `blocks-timeline/vertical.ts` | `formatTimestamp` | Same |
| `examples/data-table-page.ts` | `relativeTime` | `formatTimestamp(iso, { style: 'compact' })` |

### Shared CSS (6 primitives)

| Component | Imports | Removes |
|-----------|---------|---------|
| `debate-feed.ts` | `entryCardStyles`, `badgeStyles`, `roundDividerStyles` | ~80 lines of duplicated CSS, `statusBorderColour` for left borders |
| `blocks-point-detail.ts` | `entryCardStyles`, `badgeStyles` | ~30 lines CSS, `ENTRY_BORDER_COLOURS` map |
| `blocks-point-list.ts` | `badgeStyles`, `roundDividerStyles`, `selectedHighlightStyles` | ~40 lines CSS, `BORDER_COLOURS` map |
| `review-tracker.ts` | `selectedHighlightStyles` | ~6 lines CSS |

### statusBorderColour function

| Component | Current mechanism | Migration |
|-----------|------------------|-----------|
| `debate-feed.ts` | CSS classes (`.entry-raise`, `.entry-agree`) | `statusBorderColour(category)` in inline style |
| `review-tracker.ts` | CSS classes (`.status-open`, `.status-resolved`) | Same |
| `blocks-point-list.ts` | JS map `BORDER_COLOURS` keyed by category | Replace map with `statusBorderColour()` call |
| `blocks-point-detail.ts` | JS map `ENTRY_BORDER_COLOURS` keyed by type | Replace map with `statusBorderColour()` call |

Components need a mapping from their domain-specific entry type/status to `EntryCategory`. This mapping stays in the component — the shared function just provides the colour for a category.

## Testing

| Test | What it verifies |
|------|-----------------|
| `formatTimestamp` conversational style | "just now", "Xm ago", "Xh ago", locale date for all time ranges |
| `formatTimestamp` compact style | "now", "Xm", "Xh", "Xd" for all time ranges |
| `formatTimestamp` empty/undefined input | Returns `''` |
| `statusBorderColour` all categories | Returns correct CSS custom property string |
| `statusBorderColour` unknown category | Returns neutral fallback |
| All migrated component tests | Existing tests pass unchanged |

## Constraints

- No capability loss — all existing rendering preserved
- No forced migration — shared primitives are additive; issue says "no forced migration"
- Render callbacks preserved per PP-20260713-8ea1af
- `DebateStreamEntry` and `ConversationEntry` type unification is out of scope (noted in issue as future step)

## File changes

| File | Change |
|------|--------|
| `packages/blocks-ui-core/src/rendering/index.ts` | New — barrel exports |
| `packages/blocks-ui-core/src/rendering/format-timestamp.ts` | New — unified formatTimestamp |
| `packages/blocks-ui-core/src/rendering/status-colours.ts` | New — statusBorderColour |
| `packages/blocks-ui-core/src/rendering/entry-card-styles.ts` | New — Lit CSS |
| `packages/blocks-ui-core/src/rendering/badge-styles.ts` | New — Lit CSS |
| `packages/blocks-ui-core/src/rendering/round-divider-styles.ts` | New — Lit CSS |
| `packages/blocks-ui-core/src/rendering/selected-highlight-styles.ts` | New — Lit CSS |
| `packages/blocks-ui-core/src/index.ts` | Add `export * from './rendering/index.js'` |
| `components/document-workbench/src/debate-feed.ts` | Remove duplicated CSS + function, import shared |
| `components/conversation-viewer/src/blocks-point-detail.ts` | Remove duplicated CSS + function + map, import shared |
| `components/conversation-viewer/src/blocks-point-list.ts` | Remove duplicated CSS + map, import shared |
| `components/document-workbench/src/review-tracker.ts` | Remove duplicated CSS, import shared |
| `components/document-workbench/src/selection-threads.ts` | Remove function, import shared |
| `components/notification-inbox/src/notification-inbox.ts` | Remove `relativeTime`, import `formatTimestamp` compact |
| `components/work-item-row/src/work-item-row.ts` | Remove `relativeTime`, import `formatTimestamp` compact |
| `components/commitment-viz/src/commitment-transition-badge.ts` | Remove function, import shared |
| `components/trust-workbench/src/columns.ts` | Remove function, import shared |
| `components/blocks-timeline/src/renderers/horizontal.ts` | Remove function, import shared |
| `components/blocks-timeline/src/renderers/compact.ts` | Remove function, import shared |
| `components/blocks-timeline/src/renderers/vertical.ts` | Remove function, import shared |
| `examples/src/pages/data-table-page.ts` | Remove `relativeTime`, import shared |

## Design decisions

See `decisions.md` — D1 (full unification), D2 (functions + shared CSS), D3 (options with unified default).

## References

- [Issue #117](https://github.com/casehubio/blocks-ui/issues/117) — original issue
- [component-customisation-pattern protocol (PP-20260713-8ea1af)](/Users/mdproctor/claude/casehub/blocks-ui/docs/protocols/blocks-ui/component-customisation-pattern.md) — render callbacks preserved
- [debate-feed.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench/src/debate-feed.ts) — entry card, badges, round dividers, formatTimestamp, status colours
- [blocks-point-detail.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/conversation-viewer/src/blocks-point-detail.ts) — entry card, badges, formatTimestamp, ENTRY_BORDER_COLOURS
- [blocks-point-list.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/conversation-viewer/src/blocks-point-list.ts) — badges, round dividers, selected highlight, BORDER_COLOURS
- [review-tracker.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/document-workbench/src/review-tracker.ts) — selected highlight, status colours
- [notification-inbox.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/notification-inbox/src/notification-inbox.ts) — relativeTime (compact variant)
- [work-item-row.ts](/Users/mdproctor/claude/casehub/blocks-ui/components/work-item-row/src/work-item-row.ts) — relativeTime (compact variant)
- Duplication audit: 10-11 formatTimestamp copies, ~340 total duplicated LoC across 6 primitives
