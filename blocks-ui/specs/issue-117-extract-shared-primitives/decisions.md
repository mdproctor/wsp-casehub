## D1: Scope — unify all formatTimestamp copies

**Choice:** Extract one `formatTimestamp` to blocks-ui-core and migrate all 10+ copies across 6 components. Full unification, not just the 3 listed in the issue.
**Alternatives:**
- Issue scope only (3 files) — leaves 7+ duplicates
- Core 3 + file issues — risks follow-up never happening
**Rationale:** Same approach as SSEManager elimination — one clean cut. The inconsistency across copies (some say "just now", others "now", different fallback formats) is worse than the duplication.
**Trade-offs:** Larger scope, touches more components
**Sources:** Exploration audit: 10-11 copies across debate-feed, point-detail, selection-threads, trust-workbench, notification-inbox, work-item-row, commitment-viz, timeline renderers, examples
**Exploration:** quick
**Status:** captured

## D2: Export form — functions + shared CSS

**Choice:** Pure functions exported as JS (`formatTimestamp`, `statusBorderColour`). CSS exported as Lit `css` tagged templates (`badgeStyles`, `selectedHighlightStyles`, `roundDividerStyles`, `entryCardStyles`). No new custom elements. Components import and compose in their own `static styles` and render methods.
**Alternatives:**
- Custom elements for everything — heavyweight, 4+ new elements for what's mostly CSS
- Single render-helpers module with inline styles — bakes styles into template output, breaks CSS custom property theming
**Rationale:** Matches existing blocks-ui-core pattern (types + utilities, no UI elements). CSS tagged templates compose cleanly with Lit's `static styles`. Functions are tree-shakeable. No shadow DOM overhead for primitives that are styling concerns.
**Trade-offs:** Consumers must import and spread CSS into their own styles — slightly more wiring than a custom element
**Sources:** blocks-ui-core current structure (types only, no rendering), component-customisation-pattern protocol
**Exploration:** quick
**Status:** captured

## D3: formatTimestamp API — options with unified default

**Choice:** `formatTimestamp(iso: string, options?: FormatTimestampOptions): string` with `style: 'conversational' | 'compact'` option. Default is `'conversational'` (just now / Xm ago / Xh ago / locale date+time) — the most common pattern across the 10+ copies. `'compact'` (Xm / Xh / Xd — no "ago" suffix) for notification-inbox and work-item-row.
**Alternatives:**
- Single canonical format — forces visual changes on compact consumers
- Format string template — over-engineered for 2 variants
**Rationale:** One well-chosen default covers 8 of 10 consumers. The 2 compact consumers pass `{ style: 'compact' }`. Extensible if new styles emerge without breaking existing callers.
**Trade-offs:** Options parameter adds API surface, but it's a single optional enum
**Sources:** Audit of all 10+ formatTimestamp variants, notification-inbox relativeTime, work-item-row relativeTime
**Exploration:** quick
**Status:** captured
