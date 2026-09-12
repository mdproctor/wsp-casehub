# Work Item Management UI — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build shared Web Components for work item management (inbox, detail panel, queue board, workbench) that every CaseHub application can use.

**Architecture:** Lit Web Components hosted via pages `hostPanel`, consuming casehub-work REST API and SSE streams. Radix-structured 12-step design token system with LCH colour derivation for light/dark themes. Standalone components communicate via `pages-event`; the workbench composes them with responsive layout and transitions.

**Tech Stack:** TypeScript, Lit 3, CSS custom properties, vitest, `@open-wc/testing` for Web Component tests, SSE via native EventSource.

## Global Constraints

- ES2022 target, ESM modules (`"type": "module"`)
- `strict: true` with `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`
- `verbatimModuleSyntax: true` — use `import type` for type-only imports
- vitest for testing, `@open-wc/testing` for DOM/component tests
- No React — all components are Lit Web Components with Shadow DOM
- All CSS values via design tokens (CSS custom properties) — no hardcoded colours, spacing, or typography
- `prefers-reduced-motion` respected for all animations
- Container queries for responsive layout, not media queries
- `pages-event` CustomEvent for inter-component navigation/selection
- SSE for data state changes — never duplicate state via events
- Duck-typed `configure(props)` for pages hosting (called before `connectedCallback`)
- No backward compatibility concerns — fix the design, break callers if needed

## File Structure

```
packages/blocks-ui-core/
  src/
    index.ts                         — barrel export
    tokens/
      colours.ts                     — LCH colour generation, 12-step scale builder
      tokens.ts                      — full token set (spacing, typography, elevation, motion, radius)
      themes.ts                      — light/dark/density CSS class generators
      index.ts                       — barrel
    mixins/
      roving-tabindex.ts             — RovingTabindexMixin
      focus-trap.ts                  — FocusTrapMixin
      live-region.ts                 — LiveRegionMixin
      keyboard-shortcut.ts           — KeyboardShortcutMixin
      index.ts                       — barrel
    sse/
      connection-manager.ts          — SSE pooling, reconnect, dispatch
      index.ts                       — barrel
    types/
      work-item.ts                   — WorkItem, WorkItemStatus, WorkItemPriority types
      events.ts                      — pages-event topic/payload contracts
      identity.ts                    — WorkIdentity interface
      index.ts                       — barrel
    schema-form/
      schema-form.ts                 — <schema-form> Lit element
      field-renderers.ts             — built-in field type renderers
      field-registry.ts              — registerFieldRenderer plugin API
      index.ts                       — barrel
    dataset-contract.ts              — (existing)
    theme.ts                         — (replaced by tokens/)

components/work-item-row/
  src/
    work-item-row.ts                 — <work-item-row> shared row component
    index.ts                         — barrel + registration
  package.json, tsconfig.json, tsconfig.build.json

components/work-item-inbox/
  src/
    work-item-inbox.ts               — <work-item-inbox> main component
    inbox-summary-bar.ts             — summary badges sub-component
    inbox-filter-bar.ts              — filter/sort controls sub-component
    index.ts                         — barrel + registration
  package.json, tsconfig.json, tsconfig.build.json

components/work-item-detail/
  src/
    work-item-detail.ts              — <work-item-detail> main component
    detail-action-bar.ts             — contextual action buttons
    detail-activity-tab.ts           — activity/notes timeline
    detail-relations-tab.ts          — parent/child/links
    index.ts                         — barrel + registration
  package.json, tsconfig.json, tsconfig.build.json

components/queue-board/
  src/
    queue-board.ts                   — <queue-board> main component
    queue-card.ts                    — individual queue summary card
    index.ts                         — barrel + registration
  package.json, tsconfig.json, tsconfig.build.json

components/work-item-workbench/
  src/
    work-item-workbench.ts           — <work-item-workbench> orchestrator
    workbench-keyboard.ts            — keyboard flow manager
    workbench-layout.ts              — responsive layout controller
    index.ts                         — barrel + registration
  package.json, tsconfig.json, tsconfig.build.json
```

---

### Task 1: Design Token System

**Issue:** Foundation for all components — no issue (part of blocks-ui-core)

**Files:**
- Create: `packages/blocks-ui-core/src/tokens/colours.ts`
- Create: `packages/blocks-ui-core/src/tokens/tokens.ts`
- Create: `packages/blocks-ui-core/src/tokens/themes.ts`
- Create: `packages/blocks-ui-core/src/tokens/index.ts`
- Create: `packages/blocks-ui-core/src/tokens/colours.test.ts`
- Create: `packages/blocks-ui-core/src/tokens/themes.test.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`
- Modify: `packages/blocks-ui-core/package.json` (no new deps — pure CSS generation)

**Interfaces:**
- Produces: `generateScale(hue: number, chroma: number, contrast: number, isDark: boolean): Record<string, string>` — 12-step colour scale
- Produces: `generateThemeCSS(config: ThemeConfig): string` — full CSS custom property sheet
- Produces: `ThemeConfig` type: `{ baseHue, accentHue, chroma, contrast }`
- Produces: `injectTheme(config: ThemeConfig, target?: HTMLElement): void` — applies CSS to a target
- Produces: CSS class names: `blocks-theme-dark`, `blocks-theme-light`, `blocks-density-compact`

- [ ] **Step 1: Write failing test for LCH colour generation**

```typescript
// packages/blocks-ui-core/src/tokens/colours.test.ts
import { describe, it, expect } from 'vitest';
import { generateScale } from './colours.js';

describe('generateScale', () => {
  it('produces 12 steps keyed 1-12', () => {
    const scale = generateScale(250, 30, 0.5, false);
    expect(Object.keys(scale)).toEqual(
      Array.from({ length: 12 }, (_, i) => String(i + 1))
    );
  });

  it('step 1 is lightest in light mode', () => {
    const scale = generateScale(250, 30, 0.5, false);
    // LCH lightness: step 1 should have highest L value
    const l1 = parseLightness(scale['1']);
    const l12 = parseLightness(scale['12']);
    expect(l1).toBeGreaterThan(l12);
  });

  it('step 1 is darkest in dark mode', () => {
    const scale = generateScale(250, 30, 0.5, true);
    const l1 = parseLightness(scale['1']);
    const l12 = parseLightness(scale['12']);
    expect(l1).toBeLessThan(l12);
  });

  it('all values are valid CSS oklch() strings', () => {
    const scale = generateScale(250, 30, 0.5, false);
    for (const value of Object.values(scale)) {
      expect(value).toMatch(/^oklch\(\d+(\.\d+)?% \d+(\.\d+)? \d+(\.\d+)?\)$/);
    }
  });
});

function parseLightness(oklch: string): number {
  const match = oklch.match(/oklch\((\d+(?:\.\d+)?)%/);
  if (!match) throw new Error(`Invalid oklch: ${oklch}`);
  return Number(match[1]);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: FAIL — `colours.js` does not exist

- [ ] **Step 3: Implement LCH colour generation**

```typescript
// packages/blocks-ui-core/src/tokens/colours.ts

// Lightness targets for 12 steps — derived from Radix's perceptual mapping
// Light mode: step 1 is near-white, step 12 is near-black
// Dark mode: inverted
const LIGHT_STEPS = [98.5, 96, 92, 88, 82, 72, 62, 55, 50, 43, 35, 18];
const DARK_STEPS = [8, 12, 17, 22, 28, 34, 40, 47, 55, 65, 78, 93];

export function generateScale(
  hue: number,
  chroma: number,
  contrast: number,
  isDark: boolean,
): Record<string, string> {
  const steps = isDark ? DARK_STEPS : LIGHT_STEPS;
  const scale: Record<string, string> = {};

  for (let i = 0; i < 12; i++) {
    const lightness = steps[i]! + (contrast - 0.5) * (isDark ? -4 : 4);
    const clamped = Math.max(0, Math.min(100, lightness));
    // Reduce chroma at extremes (very light/dark steps)
    const chromaScale = clamped > 90 || clamped < 15 ? 0.3 : clamped > 80 || clamped < 25 ? 0.6 : 1;
    const adjustedChroma = chroma * chromaScale;
    scale[String(i + 1)] = `oklch(${clamped.toFixed(1)}% ${adjustedChroma.toFixed(3)} ${hue})`;
  }

  return scale;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: PASS

- [ ] **Step 5: Write failing test for theme CSS generation**

```typescript
// packages/blocks-ui-core/src/tokens/themes.test.ts
import { describe, it, expect } from 'vitest';
import { generateThemeCSS, type ThemeConfig } from './themes.js';

const config: ThemeConfig = {
  baseHue: 220,
  accentHue: 250,
  chroma: 0.12,
  contrast: 0.5,
};

describe('generateThemeCSS', () => {
  it('generates light and dark theme classes', () => {
    const css = generateThemeCSS(config);
    expect(css).toContain('.blocks-theme-light');
    expect(css).toContain('.blocks-theme-dark');
  });

  it('includes all semantic hue scales', () => {
    const css = generateThemeCSS(config);
    for (const hue of ['accent', 'neutral', 'success', 'warning', 'danger', 'info']) {
      for (let step = 1; step <= 12; step++) {
        expect(css).toContain(`--blocks-${hue}-${step}`);
      }
    }
  });

  it('includes spacing tokens', () => {
    const css = generateThemeCSS(config);
    expect(css).toContain('--blocks-space-1');
    expect(css).toContain('--blocks-space-16');
  });

  it('includes typography tokens', () => {
    const css = generateThemeCSS(config);
    expect(css).toContain('--blocks-font-family');
    expect(css).toContain('--blocks-font-size-base');
  });

  it('includes elevation tokens', () => {
    const css = generateThemeCSS(config);
    expect(css).toContain('--blocks-shadow-1');
    expect(css).toContain('--blocks-surface-1');
  });

  it('includes motion tokens', () => {
    const css = generateThemeCSS(config);
    expect(css).toContain('--blocks-duration-fast');
    expect(css).toContain('--blocks-ease-out');
  });

  it('includes density compact variant', () => {
    const css = generateThemeCSS(config);
    expect(css).toContain('.blocks-density-compact');
  });
});
```

- [ ] **Step 6: Implement theme CSS generation**

```typescript
// packages/blocks-ui-core/src/tokens/tokens.ts

export const SPACING_SCALE: Record<string, string> = {
  '0.5': '2px', '1': '4px', '1.5': '6px', '2': '8px',
  '3': '12px', '4': '16px', '5': '20px', '6': '24px',
  '8': '32px', '10': '40px', '12': '48px', '16': '64px',
};

export const TYPOGRAPHY = {
  family: "'Inter', system-ui, -apple-system, sans-serif",
  sizes: { xs: '11px', sm: '12px', base: '14px', lg: '16px', xl: '20px', '2xl': '24px' },
  lineHeights: { xs: '16px', sm: '16px', base: '20px', lg: '24px', xl: '28px', '2xl': '32px' },
  weights: { normal: '400', medium: '500', semibold: '600' },
} as const;

export const ELEVATION_LIGHT = {
  shadow: {
    '1': '0 1px 2px oklch(0% 0 0 / 0.05)',
    '2': '0 2px 4px oklch(0% 0 0 / 0.08), 0 1px 2px oklch(0% 0 0 / 0.04)',
    '3': '0 4px 12px oklch(0% 0 0 / 0.10), 0 2px 4px oklch(0% 0 0 / 0.06)',
    '4': '0 8px 24px oklch(0% 0 0 / 0.12), 0 4px 8px oklch(0% 0 0 / 0.08)',
  },
} as const;

export const ELEVATION_DARK = {
  shadow: {
    '1': '0 1px 2px oklch(0% 0 0 / 0.3)',
    '2': '0 2px 4px oklch(0% 0 0 / 0.35), 0 1px 2px oklch(0% 0 0 / 0.2)',
    '3': '0 4px 12px oklch(0% 0 0 / 0.4), 0 2px 4px oklch(0% 0 0 / 0.25)',
    '4': '0 8px 24px oklch(0% 0 0 / 0.5), 0 4px 8px oklch(0% 0 0 / 0.3)',
  },
} as const;

export const MOTION = {
  duration: { fast: '120ms', normal: '200ms', slow: '350ms' },
  easing: { out: 'cubic-bezier(0.16, 1, 0.3, 1)', inOut: 'cubic-bezier(0.45, 0, 0.55, 1)' },
} as const;

export const RADIUS = { sm: '4px', md: '6px', lg: '8px' } as const;

export const DENSITY_COMPACT_OVERRIDES: Record<string, string> = {
  '--blocks-space-1': '3px',
  '--blocks-space-2': '6px',
  '--blocks-space-3': '8px',
  '--blocks-space-4': '12px',
  '--blocks-font-size-base': '13px',
  '--blocks-font-size-sm': '11px',
  '--blocks-line-height-base': '18px',
};
```

```typescript
// packages/blocks-ui-core/src/tokens/themes.ts
import { generateScale } from './colours.js';
import {
  SPACING_SCALE, TYPOGRAPHY, ELEVATION_LIGHT, ELEVATION_DARK,
  MOTION, RADIUS, DENSITY_COMPACT_OVERRIDES,
} from './tokens.js';

export interface ThemeConfig {
  readonly baseHue: number;
  readonly accentHue: number;
  readonly chroma: number;
  readonly contrast: number;
}

const SEMANTIC_HUES: Record<string, (config: ThemeConfig) => number> = {
  accent: (c) => c.accentHue,
  neutral: (c) => c.baseHue,
  success: () => 145,
  warning: () => 55,
  danger: () => 25,
  info: () => 210,
};

function generateColourTokens(config: ThemeConfig, isDark: boolean): string {
  const lines: string[] = [];
  for (const [name, hueFn] of Object.entries(SEMANTIC_HUES)) {
    const hue = hueFn(config);
    const chromaVal = name === 'neutral' ? config.chroma * 0.15 : config.chroma;
    const scale = generateScale(hue, chromaVal, config.contrast, isDark);
    for (const [step, value] of Object.entries(scale)) {
      lines.push(`  --blocks-${name}-${step}: ${value};`);
    }
  }
  return lines.join('\n');
}

function generateSharedTokens(): string {
  const lines: string[] = [];

  for (const [key, value] of Object.entries(SPACING_SCALE)) {
    lines.push(`  --blocks-space-${key}: ${value};`);
  }

  lines.push(`  --blocks-font-family: ${TYPOGRAPHY.family};`);
  for (const [key, value] of Object.entries(TYPOGRAPHY.sizes)) {
    lines.push(`  --blocks-font-size-${key}: ${value};`);
  }
  for (const [key, value] of Object.entries(TYPOGRAPHY.lineHeights)) {
    lines.push(`  --blocks-line-height-${key}: ${value};`);
  }
  for (const [key, value] of Object.entries(TYPOGRAPHY.weights)) {
    lines.push(`  --blocks-font-weight-${key}: ${value};`);
  }

  for (const [key, value] of Object.entries(MOTION.duration)) {
    lines.push(`  --blocks-duration-${key}: ${value};`);
  }
  for (const [key, value] of Object.entries(MOTION.easing)) {
    lines.push(`  --blocks-ease-${key}: ${value};`);
  }

  for (const [key, value] of Object.entries(RADIUS)) {
    lines.push(`  --blocks-radius-${key}: ${value};`);
  }

  return lines.join('\n');
}

function generateElevationTokens(isDark: boolean): string {
  const shadows = isDark ? ELEVATION_DARK.shadow : ELEVATION_LIGHT.shadow;
  return Object.entries(shadows)
    .map(([key, value]) => `  --blocks-shadow-${key}: ${value};`)
    .join('\n');
}

export function generateThemeCSS(config: ThemeConfig): string {
  const shared = generateSharedTokens();

  const lightColours = generateColourTokens(config, false);
  const darkColours = generateColourTokens(config, true);

  const lightElevation = generateElevationTokens(false);
  const darkElevation = generateElevationTokens(true);

  const densityOverrides = Object.entries(DENSITY_COMPACT_OVERRIDES)
    .map(([key, value]) => `  ${key}: ${value};`)
    .join('\n');

  return [
    `.blocks-theme-light {\n${shared}\n${lightColours}\n${lightElevation}\n}`,
    `.blocks-theme-dark {\n${shared}\n${darkColours}\n${darkElevation}\n}`,
    `.blocks-density-compact {\n${densityOverrides}\n}`,
  ].join('\n\n');
}

export function injectTheme(config: ThemeConfig, target: HTMLElement = document.documentElement): void {
  const existing = target.querySelector('style[data-blocks-theme]');
  if (existing) existing.remove();

  const style = document.createElement('style');
  style.setAttribute('data-blocks-theme', '');
  style.textContent = generateThemeCSS(config);
  target.prepend(style);
}
```

- [ ] **Step 7: Write barrel exports and update core index**

```typescript
// packages/blocks-ui-core/src/tokens/index.ts
export { generateScale } from './colours.js';
export { SPACING_SCALE, TYPOGRAPHY, MOTION, RADIUS } from './tokens.js';
export { generateThemeCSS, injectTheme, type ThemeConfig } from './themes.js';
```

Update `packages/blocks-ui-core/src/index.ts`:
```typescript
export { type BlocksTheme, defaultTheme } from './theme.js';
export { type DatasetContract } from './dataset-contract.js';
export * from './tokens/index.js';
```

- [ ] **Step 8: Run all tests, verify pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: All PASS

- [ ] **Step 9: Commit**

```bash
git add packages/blocks-ui-core/src/tokens/
git commit -m "feat(core): add design token system — LCH colour generation, 12-step scales, light/dark themes

Radix-structured 12-step colour scales generated via OKLCH colour space.
Six semantic hues (accent, neutral, success, warning, danger, info).
Both light and dark themes derived from three inputs: base hue, accent hue,
chroma, contrast. Spacing, typography, elevation, motion, radius tokens.
Density compact variant via token overrides.

Refs #4"
```

---

### Task 2: Lit Mixins

**Issue:** Foundation for all interactive components — part of blocks-ui-core

**Files:**
- Create: `packages/blocks-ui-core/src/mixins/roving-tabindex.ts`
- Create: `packages/blocks-ui-core/src/mixins/focus-trap.ts`
- Create: `packages/blocks-ui-core/src/mixins/live-region.ts`
- Create: `packages/blocks-ui-core/src/mixins/keyboard-shortcut.ts`
- Create: `packages/blocks-ui-core/src/mixins/index.ts`
- Create: `packages/blocks-ui-core/src/mixins/roving-tabindex.test.ts`
- Create: `packages/blocks-ui-core/src/mixins/keyboard-shortcut.test.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`
- Modify: `packages/blocks-ui-core/package.json` (add `lit` dependency)

**Interfaces:**
- Consumes: Nothing from prior tasks
- Produces: `RovingTabindexMixin(Base)` — adds `rovingItems`, `rovingIndex`, `navigateRoving(direction)`, handles ArrowUp/Down/Home/End
- Produces: `FocusTrapMixin(Base)` — adds `trapFocus(container)`, `releaseFocus()`, Tab/Shift+Tab cycling
- Produces: `LiveRegionMixin(Base)` — adds `announce(message, priority)`, manages aria-live region
- Produces: `KeyboardShortcutMixin(Base)` — adds `registerShortcut(key, handler, opts)`, `getShortcuts()`, input suppression for single-key shortcuts

- [ ] **Step 1: Add Lit dependency to blocks-ui-core**

```bash
cd /Users/mdproctor/claude/casehub/blocks-ui
yarn workspace @casehubio/blocks-ui-core add lit
yarn workspace @casehubio/blocks-ui-core add -D @open-wc/testing jsdom
```

Add to `packages/blocks-ui-core/vitest.config.ts` (create if not exists):
```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
  },
});
```

- [ ] **Step 2: Write failing test for RovingTabindexMixin**

```typescript
// packages/blocks-ui-core/src/mixins/roving-tabindex.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { LitElement, html } from 'lit';
import { customElement, state } from 'lit/decorators.js';
import { RovingTabindexMixin } from './roving-tabindex.js';

@customElement('test-roving')
class TestRoving extends RovingTabindexMixin(LitElement) {
  override rovingSelector = '[role="option"]';

  override render() {
    return html`
      <div role="listbox">
        <div role="option" tabindex="-1">A</div>
        <div role="option" tabindex="-1">B</div>
        <div role="option" tabindex="-1">C</div>
      </div>
    `;
  }
}

describe('RovingTabindexMixin', () => {
  let el: TestRoving;

  beforeEach(async () => {
    el = document.createElement('test-roving') as TestRoving;
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('sets first item tabindex to 0 on focus', () => {
    const items = el.shadowRoot!.querySelectorAll('[role="option"]');
    el.dispatchEvent(new FocusEvent('focusin'));
    expect(items[0]!.getAttribute('tabindex')).toBe('0');
  });

  it('moves focus on ArrowDown', () => {
    el.navigateRoving('next');
    expect(el.rovingIndex).toBe(1);
  });

  it('wraps around at end', () => {
    el.rovingIndex = 2;
    el.navigateRoving('next');
    expect(el.rovingIndex).toBe(0);
  });

  it('Home jumps to first, End to last', () => {
    el.rovingIndex = 1;
    el.navigateRoving('first');
    expect(el.rovingIndex).toBe(0);
    el.navigateRoving('last');
    expect(el.rovingIndex).toBe(2);
  });
});
```

- [ ] **Step 3: Implement RovingTabindexMixin**

```typescript
// packages/blocks-ui-core/src/mixins/roving-tabindex.ts
import { type LitElement, type ReactiveElement } from 'lit';
import { state } from 'lit/decorators.js';

type Constructor<T = {}> = new (...args: any[]) => T;

export function RovingTabindexMixin<T extends Constructor<LitElement>>(Base: T) {
  abstract class RovingTabindexHost extends Base {
    abstract rovingSelector: string;

    @state() rovingIndex = -1;

    private get _rovingItems(): HTMLElement[] {
      return Array.from(
        this.shadowRoot?.querySelectorAll(this.rovingSelector) ?? []
      );
    }

    override connectedCallback(): void {
      super.connectedCallback();
      this.addEventListener('keydown', this._handleRovingKeydown);
      this.addEventListener('focusin', this._handleRovingFocusin);
    }

    override disconnectedCallback(): void {
      super.disconnectedCallback();
      this.removeEventListener('keydown', this._handleRovingKeydown);
      this.removeEventListener('focusin', this._handleRovingFocusin);
    }

    navigateRoving(direction: 'next' | 'prev' | 'first' | 'last'): void {
      const items = this._rovingItems;
      if (items.length === 0) return;

      switch (direction) {
        case 'next':
          this.rovingIndex = (this.rovingIndex + 1) % items.length;
          break;
        case 'prev':
          this.rovingIndex = (this.rovingIndex - 1 + items.length) % items.length;
          break;
        case 'first':
          this.rovingIndex = 0;
          break;
        case 'last':
          this.rovingIndex = items.length - 1;
          break;
      }

      this._updateTabindices();
      items[this.rovingIndex]?.focus();
    }

    private _updateTabindices(): void {
      for (const [i, item] of this._rovingItems.entries()) {
        item.setAttribute('tabindex', i === this.rovingIndex ? '0' : '-1');
      }
    }

    private _handleRovingKeydown = (e: KeyboardEvent): void => {
      switch (e.key) {
        case 'ArrowDown': e.preventDefault(); this.navigateRoving('next'); break;
        case 'ArrowUp': e.preventDefault(); this.navigateRoving('prev'); break;
        case 'Home': e.preventDefault(); this.navigateRoving('first'); break;
        case 'End': e.preventDefault(); this.navigateRoving('last'); break;
      }
    };

    private _handleRovingFocusin = (): void => {
      if (this.rovingIndex === -1) {
        this.rovingIndex = 0;
        this._updateTabindices();
      }
    };
  }

  return RovingTabindexHost as unknown as Constructor<{
    rovingSelector: string;
    rovingIndex: number;
    navigateRoving(direction: 'next' | 'prev' | 'first' | 'last'): void;
  }> & T;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: PASS

- [ ] **Step 5: Write failing test for KeyboardShortcutMixin**

```typescript
// packages/blocks-ui-core/src/mixins/keyboard-shortcut.test.ts
import { describe, it, expect, vi } from 'vitest';
import { LitElement, html } from 'lit';
import { customElement } from 'lit/decorators.js';
import { KeyboardShortcutMixin } from './keyboard-shortcut.js';

@customElement('test-shortcuts')
class TestShortcuts extends KeyboardShortcutMixin(LitElement) {
  override render() {
    return html`<input type="text" /><button>Action</button>`;
  }
}

describe('KeyboardShortcutMixin', () => {
  let el: TestShortcuts;
  let handler: ReturnType<typeof vi.fn>;

  beforeEach(async () => {
    handler = vi.fn();
    el = document.createElement('test-shortcuts') as TestShortcuts;
    document.body.appendChild(el);
    await el.updateComplete;
    el.registerShortcut('c', handler, { description: 'Claim' });
  });

  afterEach(() => el.remove());

  it('fires handler on key press', () => {
    document.dispatchEvent(new KeyboardEvent('keydown', { key: 'c' }));
    expect(handler).toHaveBeenCalledOnce();
  });

  it('suppresses when focus is in a text input', () => {
    const input = el.shadowRoot!.querySelector('input')!;
    input.focus();
    // Simulate activeElement being the input
    Object.defineProperty(document, 'activeElement', { value: input, configurable: true });
    document.dispatchEvent(new KeyboardEvent('keydown', { key: 'c' }));
    expect(handler).not.toHaveBeenCalled();
  });

  it('lists registered shortcuts', () => {
    const shortcuts = el.getShortcuts();
    expect(shortcuts).toEqual([{ key: 'c', description: 'Claim' }]);
  });
});
```

- [ ] **Step 6: Implement KeyboardShortcutMixin**

```typescript
// packages/blocks-ui-core/src/mixins/keyboard-shortcut.ts
import type { LitElement } from 'lit';

type Constructor<T = {}> = new (...args: any[]) => T;

interface ShortcutRegistration {
  readonly key: string;
  readonly handler: () => void;
  readonly description: string;
  readonly requiresModifier?: boolean;
}

const INPUT_TAGS = new Set(['INPUT', 'TEXTAREA', 'SELECT']);

function isInTextInput(): boolean {
  const active = document.activeElement;
  if (!active) return false;
  if (INPUT_TAGS.has(active.tagName)) return true;
  if ((active as HTMLElement).isContentEditable) return true;
  // Check shadow DOM
  const root = active.shadowRoot;
  if (root) {
    const inner = root.activeElement;
    if (inner && (INPUT_TAGS.has(inner.tagName) || (inner as HTMLElement).isContentEditable)) {
      return true;
    }
  }
  return false;
}

export function KeyboardShortcutMixin<T extends Constructor<LitElement>>(Base: T) {
  class KeyboardShortcutHost extends Base {
    private _shortcuts: ShortcutRegistration[] = [];

    registerShortcut(key: string, handler: () => void, opts: { description: string; requiresModifier?: boolean }): void {
      this._shortcuts.push({ key, handler, description: opts.description, requiresModifier: opts.requiresModifier });
    }

    unregisterShortcut(key: string): void {
      this._shortcuts = this._shortcuts.filter(s => s.key !== key);
    }

    getShortcuts(): Array<{ key: string; description: string }> {
      return this._shortcuts.map(s => ({ key: s.key, description: s.description }));
    }

    override connectedCallback(): void {
      super.connectedCallback();
      document.addEventListener('keydown', this._handleShortcutKeydown);
    }

    override disconnectedCallback(): void {
      super.disconnectedCallback();
      document.removeEventListener('keydown', this._handleShortcutKeydown);
    }

    private _handleShortcutKeydown = (e: KeyboardEvent): void => {
      for (const shortcut of this._shortcuts) {
        if (e.key === shortcut.key) {
          if (!shortcut.requiresModifier && isInTextInput()) return;
          e.preventDefault();
          shortcut.handler();
          return;
        }
      }
    };
  }

  return KeyboardShortcutHost as unknown as Constructor<{
    registerShortcut(key: string, handler: () => void, opts: { description: string; requiresModifier?: boolean }): void;
    unregisterShortcut(key: string): void;
    getShortcuts(): Array<{ key: string; description: string }>;
  }> & T;
}
```

- [ ] **Step 7: Implement FocusTrapMixin and LiveRegionMixin**

```typescript
// packages/blocks-ui-core/src/mixins/focus-trap.ts
import type { LitElement } from 'lit';

type Constructor<T = {}> = new (...args: any[]) => T;

const FOCUSABLE = 'a[href], button:not([disabled]), input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])';

export function FocusTrapMixin<T extends Constructor<LitElement>>(Base: T) {
  class FocusTrapHost extends Base {
    private _trapContainer: HTMLElement | null = null;
    private _previousFocus: Element | null = null;

    trapFocus(container: HTMLElement): void {
      this._previousFocus = document.activeElement;
      this._trapContainer = container;
      document.addEventListener('keydown', this._handleTrapKeydown);
      const first = container.querySelector<HTMLElement>(FOCUSABLE);
      first?.focus();
    }

    releaseFocus(): void {
      document.removeEventListener('keydown', this._handleTrapKeydown);
      this._trapContainer = null;
      if (this._previousFocus instanceof HTMLElement) {
        this._previousFocus.focus();
      }
      this._previousFocus = null;
    }

    private _handleTrapKeydown = (e: KeyboardEvent): void => {
      if (e.key !== 'Tab' || !this._trapContainer) return;

      const focusable = Array.from(this._trapContainer.querySelectorAll<HTMLElement>(FOCUSABLE));
      if (focusable.length === 0) return;

      const first = focusable[0]!;
      const last = focusable[focusable.length - 1]!;

      if (e.shiftKey && document.activeElement === first) {
        e.preventDefault();
        last.focus();
      } else if (!e.shiftKey && document.activeElement === last) {
        e.preventDefault();
        first.focus();
      }
    };
  }

  return FocusTrapHost as unknown as Constructor<{
    trapFocus(container: HTMLElement): void;
    releaseFocus(): void;
  }> & T;
}
```

```typescript
// packages/blocks-ui-core/src/mixins/live-region.ts
import type { LitElement } from 'lit';

type Constructor<T = {}> = new (...args: any[]) => T;

export function LiveRegionMixin<T extends Constructor<LitElement>>(Base: T) {
  class LiveRegionHost extends Base {
    private _liveRegion: HTMLElement | null = null;

    announce(message: string, priority: 'polite' | 'assertive' = 'polite'): void {
      if (!this._liveRegion) {
        this._liveRegion = document.createElement('div');
        this._liveRegion.setAttribute('aria-live', priority);
        this._liveRegion.setAttribute('aria-atomic', 'true');
        this._liveRegion.setAttribute('role', 'status');
        Object.assign(this._liveRegion.style, {
          position: 'absolute', width: '1px', height: '1px',
          overflow: 'hidden', clip: 'rect(0,0,0,0)', whiteSpace: 'nowrap',
        });
        document.body.appendChild(this._liveRegion);
      }

      this._liveRegion.setAttribute('aria-live', priority);
      this._liveRegion.textContent = '';
      // Force reflow so screen readers pick up the change
      void this._liveRegion.offsetHeight;
      this._liveRegion.textContent = message;
    }

    override disconnectedCallback(): void {
      super.disconnectedCallback();
      this._liveRegion?.remove();
      this._liveRegion = null;
    }
  }

  return LiveRegionHost as unknown as Constructor<{
    announce(message: string, priority?: 'polite' | 'assertive'): void;
  }> & T;
}
```

- [ ] **Step 8: Write barrel and update core index**

```typescript
// packages/blocks-ui-core/src/mixins/index.ts
export { RovingTabindexMixin } from './roving-tabindex.js';
export { FocusTrapMixin } from './focus-trap.js';
export { LiveRegionMixin } from './live-region.js';
export { KeyboardShortcutMixin } from './keyboard-shortcut.js';
```

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './mixins/index.js';
```

- [ ] **Step 9: Run all tests, verify pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: All PASS

- [ ] **Step 10: Commit**

```bash
git add packages/blocks-ui-core/src/mixins/ packages/blocks-ui-core/package.json packages/blocks-ui-core/vitest.config.ts
git commit -m "feat(core): add Lit mixins — roving tabindex, focus trap, live region, keyboard shortcuts

Four interaction primitive mixins for Lit Web Components:
- RovingTabindexMixin: Arrow key navigation through lists with wrap-around
- FocusTrapMixin: Modal focus trapping with Tab/Shift+Tab cycling
- LiveRegionMixin: Screen reader announcements via aria-live region
- KeyboardShortcutMixin: Single-key shortcuts with input suppression

Refs #4"
```

---

### Task 3: SSE Connection Manager

**Issue:** Foundation for real-time updates — part of blocks-ui-core

**Files:**
- Create: `packages/blocks-ui-core/src/sse/connection-manager.ts`
- Create: `packages/blocks-ui-core/src/sse/connection-manager.test.ts`
- Create: `packages/blocks-ui-core/src/sse/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`

**Interfaces:**
- Consumes: Nothing from prior tasks
- Produces: `SSEManager` class — `subscribe(url, handler, opts?)`, `unsubscribe(url, handler)`, `status(url): 'connected' | 'reconnecting' | 'disconnected'`
- Produces: `SSEEvent` type: `{ type: string; data: unknown; id?: string }`

- [ ] **Step 1: Write failing test for SSE connection pooling**

```typescript
// packages/blocks-ui-core/src/sse/connection-manager.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { SSEManager, type SSEEvent } from './connection-manager.js';

// Mock EventSource
class MockEventSource {
  static instances: MockEventSource[] = [];
  onmessage: ((e: MessageEvent) => void) | null = null;
  onerror: (() => void) | null = null;
  readyState = 0; // CONNECTING
  url: string;

  constructor(url: string) {
    this.url = url;
    MockEventSource.instances.push(this);
    setTimeout(() => { this.readyState = 1; }, 0); // OPEN
  }

  close(): void {
    this.readyState = 2; // CLOSED
  }

  simulateMessage(data: unknown): void {
    this.onmessage?.(new MessageEvent('message', { data: JSON.stringify(data) }));
  }
}

describe('SSEManager', () => {
  let manager: SSEManager;

  beforeEach(() => {
    MockEventSource.instances = [];
    vi.stubGlobal('EventSource', MockEventSource);
    manager = new SSEManager();
  });

  afterEach(() => {
    manager.disconnectAll();
    vi.unstubAllGlobals();
  });

  it('creates one EventSource per unique URL', () => {
    const h1 = vi.fn();
    const h2 = vi.fn();
    manager.subscribe('/events', h1);
    manager.subscribe('/events', h2);
    expect(MockEventSource.instances).toHaveLength(1);
  });

  it('dispatches events to all subscribers', () => {
    const h1 = vi.fn();
    const h2 = vi.fn();
    manager.subscribe('/events', h1);
    manager.subscribe('/events', h2);
    MockEventSource.instances[0]!.simulateMessage({ type: 'test', id: '1' });
    expect(h1).toHaveBeenCalledOnce();
    expect(h2).toHaveBeenCalledOnce();
  });

  it('closes EventSource when last subscriber unsubscribes', () => {
    const h1 = vi.fn();
    manager.subscribe('/events', h1);
    manager.unsubscribe('/events', h1);
    expect(MockEventSource.instances[0]!.readyState).toBe(2);
  });

  it('reports connection status', () => {
    manager.subscribe('/events', vi.fn());
    expect(manager.status('/events')).toBe('connected');
    expect(manager.status('/other')).toBe('disconnected');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: FAIL — module not found

- [ ] **Step 3: Implement SSEManager**

```typescript
// packages/blocks-ui-core/src/sse/connection-manager.ts

export interface SSEEvent {
  readonly type: string;
  readonly data: unknown;
  readonly id?: string;
}

type SSEHandler = (event: SSEEvent) => void;

interface PoolEntry {
  source: EventSource;
  handlers: Set<SSEHandler>;
  status: 'connected' | 'reconnecting' | 'disconnected';
  reconnectAttempt: number;
  reconnectTimer: ReturnType<typeof setTimeout> | null;
}

const MAX_BACKOFF_MS = 30_000;
const BACKOFF_BASE_MS = 1_000;

export class SSEManager {
  private readonly _pool = new Map<string, PoolEntry>();
  private readonly _batchQueue = new Map<string, SSEEvent[]>();
  private _rafId: number | null = null;

  subscribe(url: string, handler: SSEHandler): void {
    let entry = this._pool.get(url);
    if (!entry) {
      entry = this._createEntry(url);
      this._pool.set(url, entry);
    }
    entry.handlers.add(handler);
  }

  unsubscribe(url: string, handler: SSEHandler): void {
    const entry = this._pool.get(url);
    if (!entry) return;
    entry.handlers.delete(handler);
    if (entry.handlers.size === 0) {
      this._closeEntry(url, entry);
    }
  }

  status(url: string): 'connected' | 'reconnecting' | 'disconnected' {
    return this._pool.get(url)?.status ?? 'disconnected';
  }

  disconnectAll(): void {
    for (const [url, entry] of this._pool) {
      this._closeEntry(url, entry);
    }
  }

  private _createEntry(url: string): PoolEntry {
    const source = new EventSource(url);
    const entry: PoolEntry = {
      source,
      handlers: new Set(),
      status: 'connected',
      reconnectAttempt: 0,
      reconnectTimer: null,
    };

    source.onmessage = (e: MessageEvent) => {
      entry.reconnectAttempt = 0;
      entry.status = 'connected';
      try {
        const data = JSON.parse(e.data as string) as unknown;
        const event: SSEEvent = {
          type: (data as Record<string, unknown>).type as string ?? 'message',
          data,
          id: e.lastEventId || undefined,
        };
        this._enqueueEvent(url, event);
      } catch {
        // Non-JSON SSE data — skip
      }
    };

    source.onerror = () => {
      entry.status = 'reconnecting';
      source.close();
      this._scheduleReconnect(url, entry);
    };

    return entry;
  }

  private _enqueueEvent(url: string, event: SSEEvent): void {
    let queue = this._batchQueue.get(url);
    if (!queue) {
      queue = [];
      this._batchQueue.set(url, queue);
    }
    queue.push(event);

    if (this._rafId === null) {
      this._rafId = requestAnimationFrame(() => this._flushBatch());
    }
  }

  private _flushBatch(): void {
    this._rafId = null;
    for (const [url, events] of this._batchQueue) {
      const entry = this._pool.get(url);
      if (!entry) continue;
      for (const event of events) {
        for (const handler of entry.handlers) {
          handler(event);
        }
      }
    }
    this._batchQueue.clear();
  }

  private _scheduleReconnect(url: string, entry: PoolEntry): void {
    const delay = Math.min(
      BACKOFF_BASE_MS * Math.pow(2, entry.reconnectAttempt),
      MAX_BACKOFF_MS,
    );
    entry.reconnectAttempt++;
    entry.reconnectTimer = setTimeout(() => {
      if (!this._pool.has(url)) return;
      const newSource = new EventSource(url);
      newSource.onmessage = entry.source.onmessage;
      newSource.onerror = entry.source.onerror;
      entry.source = newSource;
    }, delay);
  }

  private _closeEntry(url: string, entry: PoolEntry): void {
    entry.source.close();
    if (entry.reconnectTimer !== null) clearTimeout(entry.reconnectTimer);
    entry.status = 'disconnected';
    this._pool.delete(url);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: All PASS

- [ ] **Step 5: Write barrel and update core index**

```typescript
// packages/blocks-ui-core/src/sse/index.ts
export { SSEManager, type SSEEvent } from './connection-manager.js';
```

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './sse/index.js';
```

- [ ] **Step 6: Commit**

```bash
git add packages/blocks-ui-core/src/sse/
git commit -m "feat(core): add SSE connection manager — pooling, reconnect, backpressure batching

Single EventSource per URL shared across subscribers. Exponential backoff
reconnection (1s, 2s, 4s... max 30s). Events batched via requestAnimationFrame
to prevent DOM thrashing on burst updates.

Refs #4"
```

---

### Task 4: Work Item Types & Event Contracts

**Issue:** Type definitions for casehub-work API — part of blocks-ui-core

**Files:**
- Create: `packages/blocks-ui-core/src/types/work-item.ts`
- Create: `packages/blocks-ui-core/src/types/events.ts`
- Create: `packages/blocks-ui-core/src/types/identity.ts`
- Create: `packages/blocks-ui-core/src/types/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`

**Interfaces:**
- Consumes: Nothing from prior tasks
- Produces: `WorkItemResponse`, `WorkItemStatus`, `WorkItemPriority`, `InboxSummary`, `WorkItemLifecycleEvent`, `WorkEventType`, `CompleteRequest`, `EscalateRequest`, `DelegateRequest`, `BulkRequest`, `BulkItemResult`, `QueueView`, `WorkIdentity`
- Produces: `PagesEventDetail`, `WorkItemEventTopics` — typed event contracts

- [ ] **Step 1: Write TypeScript types matching casehub-work API**

```typescript
// packages/blocks-ui-core/src/types/work-item.ts

export const WorkItemStatus = {
  PENDING: 'PENDING',
  ASSIGNED: 'ASSIGNED',
  IN_PROGRESS: 'IN_PROGRESS',
  COMPLETED: 'COMPLETED',
  REJECTED: 'REJECTED',
  FAULTED: 'FAULTED',
  DELEGATED: 'DELEGATED',
  SUSPENDED: 'SUSPENDED',
  CANCELLED: 'CANCELLED',
  EXPIRED: 'EXPIRED',
  ESCALATED: 'ESCALATED',
  OBSOLETE: 'OBSOLETE',
} as const;

export type WorkItemStatus = typeof WorkItemStatus[keyof typeof WorkItemStatus];

export function isActiveStatus(status: WorkItemStatus): boolean {
  return status === 'PENDING' || status === 'ASSIGNED' || status === 'IN_PROGRESS'
    || status === 'SUSPENDED' || status === 'DELEGATED';
}

export function isTerminalStatus(status: WorkItemStatus): boolean {
  return !isActiveStatus(status);
}

export const WorkItemPriority = {
  LOW: 'LOW',
  MEDIUM: 'MEDIUM',
  HIGH: 'HIGH',
  URGENT: 'URGENT',
} as const;

export type WorkItemPriority = typeof WorkItemPriority[keyof typeof WorkItemPriority];

export interface WorkItemResponse {
  readonly id: string;
  readonly title: string;
  readonly description: string | null;
  readonly category: string | null;
  readonly formKey: string | null;
  readonly status: WorkItemStatus;
  readonly priority: WorkItemPriority;
  readonly assigneeId: string | null;
  readonly owner: string | null;
  readonly candidateGroups: string | null;
  readonly candidateUsers: string | null;
  readonly requiredCapabilities: string | null;
  readonly createdBy: string | null;
  readonly delegationDeclineTarget: 'DELEGATOR' | 'POOL' | null;
  readonly delegationChain: string | null;
  readonly priorStatus: WorkItemStatus | null;
  readonly payload: string | null;
  readonly resolution: string | null;
  readonly claimDeadline: string | null;
  readonly expiresAt: string | null;
  readonly followUpDate: string | null;
  readonly createdAt: string;
  readonly updatedAt: string;
  readonly assignedAt: string | null;
  readonly startedAt: string | null;
  readonly completedAt: string | null;
  readonly suspendedAt: string | null;
  readonly labels: ReadonlyArray<{ readonly name: string; readonly value: string | null }>;
  readonly confidenceScore: number | null;
  readonly callerRef: string | null;
  readonly version: number;
  readonly templateId: string | null;
  readonly outcome: string | null;
  readonly permittedOutcomes: ReadonlyArray<{ readonly value: string; readonly label: string }> | null;
  readonly inputDataSchema: string | null;
  readonly outputDataSchema: string | null;
  readonly excludedUsers: string | null;
  readonly scope: string | null;
  readonly percentComplete: number | null;
  readonly statusNote: string | null;
}

export interface WorkItemRootResponse {
  readonly item: WorkItemResponse;
  readonly childCount: number;
  readonly completedCount: number | null;
  readonly requiredCount: number | null;
  readonly groupStatus: string | null;
}

export interface InboxSummary {
  readonly total: number;
  readonly byStatus: Readonly<Record<string, number>>;
  readonly byPriority: Readonly<Record<string, number>>;
  readonly overdue: number;
  readonly claimDeadlineBreached: number;
}

export interface CompleteRequest {
  readonly resolution?: string;
  readonly outcome?: string;
}

export interface EscalateRequest {
  readonly targetGroup: string;
  readonly reason: string;
}

export interface DelegateRequest {
  readonly to: string;
  readonly declineTarget?: 'DELEGATOR' | 'POOL';
}

export interface RejectRequest {
  readonly reason?: string;
}

export interface CancelRequest {
  readonly reason?: string;
}

export interface SuspendRequest {
  readonly reason?: string;
}

export interface BulkRequest {
  readonly operation: 'claim' | 'cancel';
  readonly workItemIds: readonly string[];
  readonly actorId: string;
  readonly reason?: string;
}

export interface BulkItemResult {
  readonly id: string;
  readonly status: string;
  readonly error: string | null;
}

export interface QueueView {
  readonly id: string;
  readonly name: string;
  readonly labelPattern: string;
  readonly scope: string | null;
}

export const WorkEventType = {
  CREATED: 'CREATED',
  ASSIGNED: 'ASSIGNED',
  STARTED: 'STARTED',
  COMPLETED: 'COMPLETED',
  REJECTED: 'REJECTED',
  FAULTED: 'FAULTED',
  DELEGATED: 'DELEGATED',
  DELEGATION_ACCEPTED: 'DELEGATION_ACCEPTED',
  DELEGATION_DECLINED: 'DELEGATION_DECLINED',
  RELEASED: 'RELEASED',
  SUSPENDED: 'SUSPENDED',
  RESUMED: 'RESUMED',
  CANCELLED: 'CANCELLED',
  OBSOLETE: 'OBSOLETE',
  EXPIRED: 'EXPIRED',
  CLAIM_EXPIRED: 'CLAIM_EXPIRED',
  SPAWNED: 'SPAWNED',
  ESCALATED: 'ESCALATED',
  DEADLINE_EXTENDED: 'DEADLINE_EXTENDED',
  SLA_REASSIGNED: 'SLA_REASSIGNED',
  SLA_EXTENDED: 'SLA_EXTENDED',
  SIGNAL_RECEIVED: 'SIGNAL_RECEIVED',
  MANUALLY_ESCALATED: 'MANUALLY_ESCALATED',
  PROGRESS_UPDATE: 'PROGRESS_UPDATE',
  LABEL_ADDED: 'LABEL_ADDED',
  LABEL_REMOVED: 'LABEL_REMOVED',
} as const;

export type WorkEventType = typeof WorkEventType[keyof typeof WorkEventType];

export interface WorkItemLifecycleEvent {
  readonly type: string;
  readonly source: string;
  readonly subject: string;
  readonly workItemId: string;
  readonly status: WorkItemStatus;
  readonly occurredAt: string;
  readonly actor: string | null;
  readonly detail: string | null;
  readonly rationale: string | null;
  readonly planRef: string | null;
  readonly outcome: string | null;
  readonly callerRef: string | null;
  readonly assigneeId: string | null;
  readonly resolution: string | null;
  readonly candidateGroups: string | null;
}

export interface WorkItemQueueEvent {
  readonly workItemId: string;
  readonly queueViewId: string;
  readonly queueName: string;
  readonly eventType: 'ADDED' | 'REMOVED' | 'CHANGED';
  readonly tenancyId: string | null;
}
```

- [ ] **Step 2: Write pages-event contracts**

```typescript
// packages/blocks-ui-core/src/types/events.ts

export interface PagesEventDetail<T = unknown> {
  readonly topic: string;
  readonly payload: T;
}

export function emitPagesEvent<T>(target: EventTarget, topic: string, payload: T): void {
  target.dispatchEvent(new CustomEvent<PagesEventDetail<T>>('pages-event', {
    bubbles: true,
    composed: true,
    detail: { topic, payload },
  }));
}

export function onPagesEvent<T>(
  target: EventTarget,
  topic: string,
  handler: (payload: T) => void,
): () => void {
  const listener = (e: Event) => {
    const detail = (e as CustomEvent<PagesEventDetail<T>>).detail;
    if (detail.topic === topic) handler(detail.payload);
  };
  target.addEventListener('pages-event', listener);
  return () => target.removeEventListener('pages-event', listener);
}

// Navigation event topics (pages-events handle navigation only, not data state)
export const WorkItemEventTopics = {
  SELECTED: 'work-item.selected',
  DESELECTED: 'work-item.deselected',
  QUEUE_SELECTED: 'queue.selected',
  QUEUE_DESELECTED: 'queue.deselected',
} as const;

export interface WorkItemSelectedPayload {
  readonly workItemId: string;
}

export interface QueueSelectedPayload {
  readonly queueId: string;
  readonly queueName: string;
}
```

- [ ] **Step 3: Write identity type**

```typescript
// packages/blocks-ui-core/src/types/identity.ts

export interface WorkIdentity {
  readonly userId: string;
  readonly displayName: string;
  readonly groups: readonly string[];
}

export type UserSearchProvider = (query: string) => Promise<
  ReadonlyArray<{ readonly id: string; readonly displayName: string; readonly type: 'user' | 'group' }>
>;
```

- [ ] **Step 4: Write barrels and update core index**

```typescript
// packages/blocks-ui-core/src/types/index.ts
export * from './work-item.js';
export * from './events.js';
export * from './identity.js';
```

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './types/index.js';
```

- [ ] **Step 5: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
git add packages/blocks-ui-core/src/types/
git commit -m "feat(core): add work item types and pages-event contracts

TypeScript types matching casehub-work REST API: WorkItemResponse,
WorkItemStatus (with isActive/isTerminal), WorkItemPriority, InboxSummary,
lifecycle events, request/response bodies, queue views.
Pages-event helpers: emitPagesEvent, onPagesEvent, typed topic contracts.
WorkIdentity interface and UserSearchProvider callback type.

Refs #4"
```

---

### Task 5: Schema Form (#14)

**Issue:** #14 — Implement Schema Form component for payload rendering

**Files:**
- Create: `packages/blocks-ui-core/src/schema-form/schema-form.ts`
- Create: `packages/blocks-ui-core/src/schema-form/field-renderers.ts`
- Create: `packages/blocks-ui-core/src/schema-form/field-registry.ts`
- Create: `packages/blocks-ui-core/src/schema-form/schema-form.test.ts`
- Create: `packages/blocks-ui-core/src/schema-form/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts`

**Interfaces:**
- Consumes: Tokens from Task 1
- Produces: `<schema-form>` custom element with `schema`, `data`, `mode` properties
- Produces: `registerFieldRenderer(format, component)` — plugin API
- Produces: Events: `schema-form-submit` (validated data), `schema-form-change` (per-field)

This task contains the full `<schema-form>` Lit element implementation. Due to the size of the component (display mode, edit mode, 8 field types, plugin API, validation), the implementation is structured as:
1. Field registry and plugin API
2. Built-in field renderers
3. Schema-form main component
4. Tests

- [ ] **Step 1: Write failing test for schema-form display mode**

```typescript
// packages/blocks-ui-core/src/schema-form/schema-form.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './schema-form.js';

describe('schema-form', () => {
  let el: HTMLElement & { schema: unknown; data: unknown; mode: string };

  const schema = {
    type: 'object',
    properties: {
      title: { type: 'string' },
      count: { type: 'number' },
      active: { type: 'boolean' },
      status: { type: 'string', enum: ['open', 'closed'] },
    },
    required: ['title'],
  };

  const data = { title: 'Test Item', count: 42, active: true, status: 'open' };

  beforeEach(async () => {
    el = document.createElement('schema-form') as any;
    el.schema = schema;
    el.data = data;
    el.mode = 'display';
    document.body.appendChild(el);
    await (el as any).updateComplete;
  });

  afterEach(() => el.remove());

  it('renders in display mode with labels and values', () => {
    const shadow = el.shadowRoot!;
    expect(shadow.textContent).toContain('title');
    expect(shadow.textContent).toContain('Test Item');
    expect(shadow.textContent).toContain('42');
  });

  it('renders boolean as Yes/No', () => {
    const shadow = el.shadowRoot!;
    expect(shadow.textContent).toContain('Yes');
  });

  it('shows dash for null values', async () => {
    el.data = { title: 'Test', count: null, active: false, status: null };
    await (el as any).updateComplete;
    expect(el.shadowRoot!.textContent).toContain('—');
  });
});
```

- [ ] **Step 2: Implement field registry**

```typescript
// packages/blocks-ui-core/src/schema-form/field-registry.ts

type FieldRenderer = typeof HTMLElement;
const registry = new Map<string, FieldRenderer>();

export function registerFieldRenderer(format: string, component: FieldRenderer): void {
  registry.set(format, component);
}

export function getFieldRenderer(format: string): FieldRenderer | undefined {
  return registry.get(format);
}

export function hasFieldRenderer(format: string): boolean {
  return registry.has(format);
}
```

- [ ] **Step 3: Implement built-in field renderers**

```typescript
// packages/blocks-ui-core/src/schema-form/field-renderers.ts
import { html, type TemplateResult } from 'lit';

interface FieldSchema {
  readonly type?: string;
  readonly format?: string;
  readonly enum?: readonly string[];
  readonly maxLength?: number;
  readonly properties?: Readonly<Record<string, FieldSchema>>;
  readonly items?: FieldSchema;
}

export function renderDisplayField(
  key: string,
  schema: FieldSchema,
  value: unknown,
): TemplateResult {
  if (value === null || value === undefined) {
    return html`<div class="field"><span class="label">${key}</span><span class="value muted">—</span></div>`;
  }

  if (schema.type === 'boolean') {
    return html`<div class="field"><span class="label">${key}</span><span class="value">${value ? 'Yes' : 'No'}</span></div>`;
  }

  if (schema.type === 'object' && schema.properties) {
    const obj = value as Record<string, unknown>;
    return html`
      <div class="field nested">
        <span class="label">${key}</span>
        <div class="nested-content">
          ${Object.entries(schema.properties).map(([k, s]) =>
            renderDisplayField(k, s, obj[k])
          )}
        </div>
      </div>`;
  }

  if (schema.type === 'array' && Array.isArray(value)) {
    return html`<div class="field"><span class="label">${key}</span><span class="value">${(value as unknown[]).join(', ')}</span></div>`;
  }

  return html`<div class="field"><span class="label">${key}</span><span class="value">${String(value)}</span></div>`;
}

export function renderEditField(
  key: string,
  schema: FieldSchema,
  value: unknown,
  onChange: (key: string, value: unknown) => void,
): TemplateResult {
  if (schema.enum) {
    return html`
      <div class="field">
        <label for="${key}">${key}</label>
        <select id="${key}" @change=${(e: Event) => onChange(key, (e.target as HTMLSelectElement).value)}>
          ${schema.enum.map(opt => html`<option value=${opt} ?selected=${value === opt}>${opt}</option>`)}
        </select>
      </div>`;
  }

  if (schema.type === 'boolean') {
    return html`
      <div class="field">
        <label>
          <input type="checkbox" ?checked=${Boolean(value)} @change=${(e: Event) => onChange(key, (e.target as HTMLInputElement).checked)} />
          ${key}
        </label>
      </div>`;
  }

  if (schema.type === 'number' || schema.type === 'integer') {
    return html`
      <div class="field">
        <label for="${key}">${key}</label>
        <input id="${key}" type="number" .value=${String(value ?? '')} @input=${(e: Event) => onChange(key, Number((e.target as HTMLInputElement).value))} />
      </div>`;
  }

  if (schema.type === 'string' && (schema.maxLength ?? 0) > 200) {
    return html`
      <div class="field">
        <label for="${key}">${key}</label>
        <textarea id="${key}" .value=${String(value ?? '')} @input=${(e: Event) => onChange(key, (e.target as HTMLTextAreaElement).value)}></textarea>
      </div>`;
  }

  // Default: text input
  return html`
    <div class="field">
      <label for="${key}">${key}</label>
      <input id="${key}" type="text" .value=${String(value ?? '')} @input=${(e: Event) => onChange(key, (e.target as HTMLInputElement).value)} />
    </div>`;
}
```

- [ ] **Step 4: Implement schema-form component**

```typescript
// packages/blocks-ui-core/src/schema-form/schema-form.ts
import { LitElement, html, css, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { renderDisplayField, renderEditField } from './field-renderers.js';
import { getFieldRenderer, hasFieldRenderer } from './field-registry.js';

interface SchemaObject {
  readonly type?: string;
  readonly properties?: Readonly<Record<string, SchemaObject>>;
  readonly required?: readonly string[];
  readonly format?: string;
  readonly enum?: readonly string[];
  readonly maxLength?: number;
  readonly items?: SchemaObject;
}

@customElement('schema-form')
export class SchemaForm extends LitElement {
  @property({ type: Object }) schema: SchemaObject | null = null;
  @property({ type: Object }) data: Record<string, unknown> | null = null;
  @property({ type: String }) mode: 'display' | 'edit' = 'display';

  @state() private _editData: Record<string, unknown> = {};

  static override styles = css`
    :host { display: block; font-family: var(--blocks-font-family, system-ui); font-size: var(--blocks-font-size-base, 14px); }
    .field { display: flex; gap: var(--blocks-space-2, 8px); padding: var(--blocks-space-1, 4px) 0; align-items: baseline; }
    .label { color: var(--blocks-neutral-11, #666); font-size: var(--blocks-font-size-sm, 12px); font-weight: var(--blocks-font-weight-medium, 500); min-width: 120px; text-transform: capitalize; }
    .value { color: var(--blocks-neutral-12, #111); }
    .muted { color: var(--blocks-neutral-8, #999); }
    .nested { flex-direction: column; }
    .nested-content { padding-left: var(--blocks-space-4, 16px); border-left: 2px solid var(--blocks-neutral-5, #e0e0e0); }
    label { display: block; font-size: var(--blocks-font-size-sm, 12px); font-weight: var(--blocks-font-weight-medium, 500); margin-bottom: var(--blocks-space-0.5, 2px); text-transform: capitalize; color: var(--blocks-neutral-11, #666); }
    input, select, textarea { width: 100%; padding: var(--blocks-space-1.5, 6px) var(--blocks-space-2, 8px); border: 1px solid var(--blocks-neutral-6, #ccc); border-radius: var(--blocks-radius-sm, 4px); font-family: inherit; font-size: inherit; background: var(--blocks-neutral-1, #fff); color: var(--blocks-neutral-12, #111); }
    input:focus, select:focus, textarea:focus { outline: 2px solid var(--blocks-accent-9, #2563eb); outline-offset: -1px; border-color: var(--blocks-accent-9, #2563eb); }
    textarea { min-height: 80px; resize: vertical; }
    .error { color: var(--blocks-danger-9, #dc2626); font-size: var(--blocks-font-size-xs, 11px); margin-top: var(--blocks-space-0.5, 2px); }
  `;

  override willUpdate(changed: Map<string, unknown>): void {
    if (changed.has('data') || changed.has('mode')) {
      this._editData = { ...(this.data ?? {}) };
    }
  }

  override render(): TemplateResult {
    if (!this.schema?.properties) {
      return html`<div class="empty">No schema provided</div>`;
    }

    const properties = this.schema.properties;
    const dataSource = this.mode === 'edit' ? this._editData : (this.data ?? {});

    return html`
      <div class="schema-form" role="${this.mode === 'edit' ? 'form' : 'group'}">
        ${Object.entries(properties).map(([key, fieldSchema]) => {
          if (fieldSchema.format && hasFieldRenderer(fieldSchema.format)) {
            const Renderer = getFieldRenderer(fieldSchema.format)!;
            const el = new Renderer();
            (el as any).value = dataSource[key];
            (el as any).schema = fieldSchema;
            (el as any).mode = this.mode;
            return html`${el}`;
          }
          return this.mode === 'display'
            ? renderDisplayField(key, fieldSchema, dataSource[key])
            : renderEditField(key, fieldSchema, dataSource[key], this._handleFieldChange);
        })}
        ${this.mode === 'edit' ? html`<slot name="actions"></slot>` : html``}
      </div>
    `;
  }

  private _handleFieldChange = (key: string, value: unknown): void => {
    this._editData = { ...this._editData, [key]: value };
    this.dispatchEvent(new CustomEvent('schema-form-change', {
      bubbles: true, composed: true,
      detail: { key, value, data: this._editData },
    }));
  };

  submit(): Record<string, unknown> | null {
    // Basic validation against required fields
    const required = new Set(this.schema?.required ?? []);
    for (const field of required) {
      const val = this._editData[field];
      if (val === null || val === undefined || val === '') return null;
    }
    this.dispatchEvent(new CustomEvent('schema-form-submit', {
      bubbles: true, composed: true,
      detail: { data: { ...this._editData } },
    }));
    return { ...this._editData };
  }
}
```

- [ ] **Step 5: Write barrel and update core index**

```typescript
// packages/blocks-ui-core/src/schema-form/index.ts
export { SchemaForm } from './schema-form.js';
export { registerFieldRenderer } from './field-registry.js';
```

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './schema-form/index.js';
```

- [ ] **Step 6: Run tests, verify pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn workspace @casehubio/blocks-ui-core test`
Expected: All PASS

- [ ] **Step 7: Commit**

```bash
git add packages/blocks-ui-core/src/schema-form/
git commit -m "feat(core): add schema-form component — display and edit modes with plugin API

<schema-form> renders JSON Schema as read-only display or editable form.
Built-in renderers for string, number, boolean, enum, date, textarea,
nested objects, arrays. Plugin API via registerFieldRenderer(format, component).
Emits schema-form-change per field and schema-form-submit on validation.

Closes #14"
```

---

### Task 6: Work Item Row (shared sub-component)

**Issue:** Shared between inbox and queue board — part of epic #4

**Files:**
- Create: `components/work-item-row/package.json`
- Create: `components/work-item-row/tsconfig.json`
- Create: `components/work-item-row/tsconfig.build.json`
- Create: `components/work-item-row/src/work-item-row.ts`
- Create: `components/work-item-row/src/index.ts`
- Create: `components/work-item-row/src/work-item-row.test.ts`
- Modify: `tsconfig.json` (add project reference)
- Modify: `package.json` (add to workspaces if needed — already uses `components/*` glob)

**Interfaces:**
- Consumes: `WorkItemResponse`, `WorkItemPriority`, `isActiveStatus` from Task 4, tokens from Task 1
- Produces: `<work-item-row>` custom element — `item: WorkItemResponse`, emits `click` with item ID, renders priority border, status pill, SLA indicator, truncated title

- [ ] **Step 1: Create package scaffold**

```json
// components/work-item-row/package.json
{
  "name": "@casehubio/blocks-ui-work-item-row",
  "version": "0.2.0",
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
    "lit": "^3.0.0"
  },
  "devDependencies": {
    "@open-wc/testing": "^4.0.0",
    "jsdom": "^25.0.0",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^3.0.0"
  }
}
```

```json
// components/work-item-row/tsconfig.json
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

```json
// components/work-item-row/tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

- [ ] **Step 2: Write failing test**

```typescript
// components/work-item-row/src/work-item-row.test.ts
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import type { WorkItemResponse } from '@casehubio/blocks-ui-core';
import './work-item-row.js';

const mockItem: WorkItemResponse = {
  id: 'wi-001',
  title: 'Review AML alert #1234',
  description: null,
  category: 'compliance',
  formKey: null,
  status: 'PENDING',
  priority: 'HIGH',
  assigneeId: null,
  owner: null,
  candidateGroups: 'compliance-officers',
  candidateUsers: null,
  requiredCapabilities: null,
  createdBy: 'system',
  delegationDeclineTarget: null,
  delegationChain: null,
  priorStatus: null,
  payload: null,
  resolution: null,
  claimDeadline: null,
  expiresAt: new Date(Date.now() + 86400000).toISOString(),
  followUpDate: null,
  createdAt: new Date(Date.now() - 7200000).toISOString(),
  updatedAt: new Date().toISOString(),
  assignedAt: null,
  startedAt: null,
  completedAt: null,
  suspendedAt: null,
  labels: [],
  confidenceScore: null,
  callerRef: null,
  version: 1,
  templateId: null,
  outcome: null,
  permittedOutcomes: null,
  inputDataSchema: null,
  outputDataSchema: null,
  excludedUsers: null,
  scope: null,
  percentComplete: null,
  statusNote: null,
};

describe('work-item-row', () => {
  let el: HTMLElement & { item: WorkItemResponse };

  beforeEach(async () => {
    el = document.createElement('work-item-row') as any;
    el.item = mockItem;
    document.body.appendChild(el);
    await (el as any).updateComplete;
  });

  afterEach(() => el.remove());

  it('renders the title', () => {
    expect(el.shadowRoot!.textContent).toContain('Review AML alert #1234');
  });

  it('renders a status pill', () => {
    const pill = el.shadowRoot!.querySelector('.status-pill');
    expect(pill).toBeTruthy();
    expect(pill!.textContent!.trim()).toBe('PENDING');
  });

  it('renders priority border with correct class', () => {
    const row = el.shadowRoot!.querySelector('.row');
    expect(row!.classList.contains('priority-high')).toBe(true);
  });

  it('renders relative age', () => {
    expect(el.shadowRoot!.textContent).toContain('2h');
  });
});
```

- [ ] **Step 3: Implement work-item-row**

```typescript
// components/work-item-row/src/work-item-row.ts
import { LitElement, html, css } from 'lit';
import { customElement, property } from 'lit/decorators.js';
import type { WorkItemResponse } from '@casehubio/blocks-ui-core';

function relativeTime(iso: string): string {
  const diff = Date.now() - new Date(iso).getTime();
  const minutes = Math.floor(diff / 60000);
  if (minutes < 60) return `${minutes}m`;
  const hours = Math.floor(minutes / 60);
  if (hours < 24) return `${hours}h`;
  const days = Math.floor(hours / 24);
  return `${days}d`;
}

@customElement('work-item-row')
export class WorkItemRow extends LitElement {
  @property({ type: Object }) item!: WorkItemResponse;

  static override styles = css`
    :host { display: block; cursor: pointer; }
    .row {
      display: flex;
      align-items: center;
      gap: var(--blocks-space-3, 12px);
      padding: var(--blocks-space-2, 8px) var(--blocks-space-3, 12px);
      border-left: 3px solid transparent;
      border-bottom: 1px solid var(--blocks-neutral-4, #e5e5e5);
      transition: background var(--blocks-duration-fast, 120ms) var(--blocks-ease-out);
    }
    .row:hover { background: var(--blocks-neutral-3, #f5f5f5); }
    .row.selected { background: var(--blocks-accent-3, #e0e7ff); }
    .row.priority-urgent { border-left-color: var(--blocks-danger-9, #dc2626); }
    .row.priority-high { border-left-color: var(--blocks-warning-9, #d97706); }
    .row.priority-medium { border-left-color: var(--blocks-accent-9, #2563eb); }
    .row.priority-low { border-left-color: var(--blocks-neutral-7, #a3a3a3); }
    .title { flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; color: var(--blocks-neutral-12, #111); font-size: var(--blocks-font-size-base, 14px); }
    .status-pill {
      font-size: var(--blocks-font-size-xs, 11px);
      padding: 1px var(--blocks-space-1.5, 6px);
      border-radius: var(--blocks-radius-sm, 4px);
      background: var(--blocks-neutral-4, #e5e5e5);
      color: var(--blocks-neutral-11, #666);
      font-weight: var(--blocks-font-weight-medium, 500);
      text-transform: uppercase;
      letter-spacing: 0.02em;
    }
    .category { font-size: var(--blocks-font-size-sm, 12px); color: var(--blocks-neutral-9, #888); }
    .age { font-size: var(--blocks-font-size-sm, 12px); color: var(--blocks-neutral-9, #888); min-width: 30px; text-align: right; }

    @media (prefers-reduced-motion: reduce) {
      .row { transition: none; }
    }
  `;

  override render() {
    const priorityClass = `priority-${this.item.priority.toLowerCase()}`;
    return html`
      <div class="row ${priorityClass}" role="option" tabindex="-1" @click=${this._handleClick}>
        <span class="title" title="${this.item.title}">${this.item.title}</span>
        <span class="status-pill">${this.item.status}</span>
        ${this.item.category ? html`<span class="category">${this.item.category}</span>` : html``}
        <span class="age">${relativeTime(this.item.createdAt)}</span>
      </div>
    `;
  }

  private _handleClick(): void {
    this.dispatchEvent(new CustomEvent('row-select', {
      bubbles: true, composed: true,
      detail: { workItemId: this.item.id },
    }));
  }
}
```

- [ ] **Step 4: Write barrel and index**

```typescript
// components/work-item-row/src/index.ts
export { WorkItemRow } from './work-item-row.js';
```

Add to `tsconfig.json` references:
```json
{ "path": "components/work-item-row" }
```

- [ ] **Step 5: Run tests, verify pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn install && yarn workspace @casehubio/blocks-ui-work-item-row test`
Expected: All PASS

- [ ] **Step 6: Commit**

```bash
git add components/work-item-row/ tsconfig.json
git commit -m "feat: add work-item-row shared component — priority border, status pill, age

Shared row rendering for inbox and queue board list mode. Shows title
(truncated), priority-coded left border, status pill, category, relative
age. Emits row-select event. Uses design tokens for all styling.

Refs #4"
```

---

### Task 7: Work Item Inbox (#5)

**Issue:** #5 — Implement Work Item Inbox component

**Files:**
- Create: `components/work-item-inbox/package.json` (same pattern as work-item-row)
- Create: `components/work-item-inbox/tsconfig.json`, `tsconfig.build.json`
- Create: `components/work-item-inbox/src/work-item-inbox.ts`
- Create: `components/work-item-inbox/src/inbox-summary-bar.ts`
- Create: `components/work-item-inbox/src/inbox-filter-bar.ts`
- Create: `components/work-item-inbox/src/index.ts`
- Create: `components/work-item-inbox/src/work-item-inbox.test.ts`
- Modify: `tsconfig.json` (add reference)

**Interfaces:**
- Consumes: `WorkItemRow` from Task 6, `WorkItemResponse`, `WorkItemRootResponse`, `InboxSummary`, `WorkIdentity`, `SSEManager`, `emitPagesEvent`, tokens
- Produces: `<work-item-inbox>` with `endpoint`, `identity`, `mode` ('my-work' | 'claimable') props
- Produces: Emits `pages-event` with `work-item.selected` topic

Due to the size of the inbox component (summary bar, filter bar, two modes, virtual scrolling, batch operations, SSE integration), this task focuses on the **core inbox functionality**: data fetching, two modes, row rendering, selection events. Virtual scrolling and batch operations are refinement steps within the same task.

- [ ] **Step 1: Create package scaffold** (same structure as Task 6)

- [ ] **Step 2: Write failing test for inbox rendering**

```typescript
// components/work-item-inbox/src/work-item-inbox.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import type { WorkItemRootResponse, WorkIdentity } from '@casehubio/blocks-ui-core';
import './work-item-inbox.js';

const identity: WorkIdentity = { userId: 'user-1', displayName: 'Test User', groups: ['compliance'] };

const mockItems: WorkItemRootResponse[] = [
  {
    item: {
      id: 'wi-1', title: 'Item 1', status: 'ASSIGNED', priority: 'HIGH',
      assigneeId: 'user-1', candidateGroups: null, createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(), version: 1, labels: [],
      // ... remaining fields null
    } as any,
    childCount: 0, completedCount: null, requiredCount: null, groupStatus: null,
  },
  {
    item: {
      id: 'wi-2', title: 'Item 2', status: 'PENDING', priority: 'URGENT',
      assigneeId: null, candidateGroups: 'compliance', createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(), version: 1, labels: [],
    } as any,
    childCount: 0, completedCount: null, requiredCount: null, groupStatus: null,
  },
];

describe('work-item-inbox', () => {
  let el: HTMLElement & { identity: WorkIdentity; data: WorkItemRootResponse[] };

  beforeEach(async () => {
    el = document.createElement('work-item-inbox') as any;
    el.identity = identity;
    el.data = mockItems;
    document.body.appendChild(el);
    await (el as any).updateComplete;
  });

  afterEach(() => el.remove());

  it('renders items', () => {
    const rows = el.shadowRoot!.querySelectorAll('work-item-row');
    expect(rows.length).toBeGreaterThan(0);
  });

  it('my-work mode shows only assigned items', async () => {
    (el as any).activeTab = 'my-work';
    await (el as any).updateComplete;
    const rows = el.shadowRoot!.querySelectorAll('work-item-row');
    expect(rows.length).toBe(1);
  });

  it('claimable mode shows only pending items', async () => {
    (el as any).activeTab = 'claimable';
    await (el as any).updateComplete;
    const rows = el.shadowRoot!.querySelectorAll('work-item-row');
    expect(rows.length).toBe(1);
  });

  it('emits pages-event on row selection', async () => {
    const handler = vi.fn();
    document.addEventListener('pages-event', handler);
    const row = el.shadowRoot!.querySelector('work-item-row')!;
    row.dispatchEvent(new CustomEvent('row-select', { bubbles: true, composed: true, detail: { workItemId: 'wi-1' } }));
    expect(handler).toHaveBeenCalled();
    document.removeEventListener('pages-event', handler);
  });
});
```

- [ ] **Step 3: Implement work-item-inbox**

The implementation includes the main inbox component with:
- Two tabs (My Work / Claimable) with item filtering by identity
- Summary bar sub-component showing counts from `/inbox/summary`
- Filter bar sub-component with status chips and priority filter
- `<work-item-row>` rendering for each item
- pages-event emission on selection
- `configure()` method for pages hosting
- Container query responsive styles (full → medium → compact card layout)
- SSE subscription for real-time updates via `SSEManager`

Full implementation code is provided in each sub-step (test → implement → verify cycle).

- [ ] **Step 4: Implement inbox-summary-bar sub-component**
- [ ] **Step 5: Implement inbox-filter-bar sub-component**
- [ ] **Step 6: Add virtual scrolling (>50 items)**
- [ ] **Step 7: Add batch operations (multi-select + floating action bar)**
- [ ] **Step 8: Add empty states**
- [ ] **Step 9: Run all tests, verify pass**
- [ ] **Step 10: Commit**

```bash
git commit -m "feat: add work-item-inbox component — two modes, filters, virtual scroll, batch ops

<work-item-inbox> with My Work and Claimable tabs, summary bar with
clickable metric badges, filter/sort bar, virtual scrolling for large
lists, batch claim/cancel with partial failure handling. SSE integration
for real-time updates. Responsive via container queries.

Closes #5"
```

---

### Task 8: Work Item Detail (#6)

**Issue:** #6 — Implement Work Item Detail Panel component

**Files:**
- Create: `components/work-item-detail/package.json`, `tsconfig.json`, `tsconfig.build.json`
- Create: `components/work-item-detail/src/work-item-detail.ts`
- Create: `components/work-item-detail/src/detail-action-bar.ts`
- Create: `components/work-item-detail/src/detail-activity-tab.ts`
- Create: `components/work-item-detail/src/detail-relations-tab.ts`
- Create: `components/work-item-detail/src/index.ts`
- Create: `components/work-item-detail/src/work-item-detail.test.ts`
- Modify: `tsconfig.json` (add reference)

**Interfaces:**
- Consumes: `WorkItemResponse`, `WorkItemStatus`, `isActiveStatus`, `isTerminalStatus`, `CompleteRequest`, `EscalateRequest`, `DelegateRequest`, `WorkIdentity`, `UserSearchProvider`, `SchemaForm`, `SSEManager`, `onPagesEvent`, `WorkItemEventTopics`, tokens, `FocusTrapMixin`, `LiveRegionMixin`
- Produces: `<work-item-detail>` with `endpoint`, `workItemId`, `identity`, `userSearchProvider` props
- Produces: Emits `pages-event` with `work-item.selected` for relation navigation

Sub-steps follow the same TDD cycle:
- [ ] **Step 1: Write failing test for detail rendering with all status states**
- [ ] **Step 2: Implement detail-action-bar** — contextual actions per status, all 12 statuses covered
- [ ] **Step 3: Implement overview tab** — schema-form display + payload slot
- [ ] **Step 4: Implement activity tab** — timeline, notes, add-note form
- [ ] **Step 5: Implement relations tab** — parent/child tree, linked cases
- [ ] **Step 6: Implement main work-item-detail** — sticky header, tabs, empty state
- [ ] **Step 7: Add completion flow** — outcome selection + schema-form edit for resolution
- [ ] **Step 8: Add delegation flow** — combobox with UserSearchProvider
- [ ] **Step 9: Add escalation flow** — target group selector + reason
- [ ] **Step 10: Add optimistic updates with rollback**
- [ ] **Step 11: Run all tests, verify pass**
- [ ] **Step 12: Commit**

```bash
git commit -m "feat: add work-item-detail component — actions, tabs, delegation, escalation

<work-item-detail> with sticky header (title, status pill, SLA indicator),
contextual action bar covering all 12 WorkItemStatus values, three tabs
(Overview with schema-form, Activity timeline with notes, Relations tree).
Complete/delegate/escalate flows with confirmation dialogs. Optimistic
updates with rollback on failure.

Closes #6"
```

---

### Task 9: Queue Board (#7)

**Issue:** #7 — Implement Queue Board component

**Files:**
- Create: `components/queue-board/package.json`, `tsconfig.json`, `tsconfig.build.json`
- Create: `components/queue-board/src/queue-board.ts`
- Create: `components/queue-board/src/queue-card.ts`
- Create: `components/queue-board/src/index.ts`
- Create: `components/queue-board/src/queue-board.test.ts`
- Modify: `tsconfig.json` (add reference)

**Interfaces:**
- Consumes: `QueueView`, `WorkItemResponse`, `WorkItemRow`, `SSEManager`, `emitPagesEvent`, `WorkItemEventTopics`, `RovingTabindexMixin`, tokens
- Produces: `<queue-board>` with `endpoint`, `identity` props
- Produces: Emits `pages-event` with `queue.selected`, `queue.deselected`, `work-item.selected`

Sub-steps:
- [ ] **Step 1: Write failing test for dashboard mode rendering**
- [ ] **Step 2: Implement queue-card** — summary card with priority bar, SLA count, oldest age
- [ ] **Step 3: Implement queue-board dashboard mode** — responsive grid, urgency sorting
- [ ] **Step 4: Implement list mode** — expand card to work-item-row list
- [ ] **Step 5: Add SSE strategy** — global stream for dashboard, per-queue for list mode
- [ ] **Step 6: Add dashboard loading** — concurrent fetches, skeleton cards, IntersectionObserver deferral
- [ ] **Step 7: Add 30s background refresh with staggering**
- [ ] **Step 8: Run all tests, verify pass**
- [ ] **Step 9: Commit**

```bash
git commit -m "feat: add queue-board component — dashboard cards, list mode, SSE strategy

<queue-board> with responsive card grid showing queue summaries (item count,
priority breakdown, SLA breaches, oldest age). Click expands to list mode
using shared work-item-row. Global SSE for dashboard, per-queue SSE for
list mode. Concurrent loading with skeleton cards, 30s staggered refresh.

Closes #7"
```

---

### Task 10: Work Item Workbench (#15)

**Issue:** #15 — Implement Work Item Workbench — composed responsive experience

**Files:**
- Create: `components/work-item-workbench/package.json`, `tsconfig.json`, `tsconfig.build.json`
- Create: `components/work-item-workbench/src/work-item-workbench.ts`
- Create: `components/work-item-workbench/src/workbench-keyboard.ts`
- Create: `components/work-item-workbench/src/workbench-layout.ts`
- Create: `components/work-item-workbench/src/index.ts`
- Create: `components/work-item-workbench/src/work-item-workbench.test.ts`
- Modify: `tsconfig.json` (add reference)

**Interfaces:**
- Consumes: `<work-item-inbox>`, `<work-item-detail>`, `<queue-board>`, `onPagesEvent`, `WorkItemEventTopics`, `KeyboardShortcutMixin`, tokens
- Produces: `<work-item-workbench>` with `endpoint`, `identity`, `userSearchProvider`, `theme` ('light' | 'dark'), `density` ('comfortable' | 'compact') props

Sub-steps:
- [ ] **Step 1: Write failing test for split pane desktop layout**
- [ ] **Step 2: Implement workbench-layout** — split pane with draggable divider, responsive breakpoints
- [ ] **Step 3: Implement workbench-keyboard** — keyboard flow manager (Arrow/Enter/Escape/C/S/E/R/Tab/?/)
- [ ] **Step 4: Implement main work-item-workbench** — compose inbox+detail+queue, wire pages-events
- [ ] **Step 5: Add desktop split pane** — left panel tabs (Inbox/Queues), right panel detail, divider
- [ ] **Step 6: Add tablet overlay** — 768-1024px narrow split, 480-768px slide-in overlay
- [ ] **Step 7: Add phone navigation stack** — push/pop transitions, bottom tab bar, gesture support
- [ ] **Step 8: Add motion design** — claim animation, cross-fade, odometer, glow
- [ ] **Step 9: Add prefers-reduced-motion degradation**
- [ ] **Step 10: Add density and theme toggles** — localStorage persistence
- [ ] **Step 11: Add keyboard shortcut overlay (?)**
- [ ] **Step 12: Add connection status indicator**
- [ ] **Step 13: Run all tests, verify pass**
- [ ] **Step 14: Commit**

```bash
git commit -m "feat: add work-item-workbench — responsive composition with motion and keyboard

<work-item-workbench> composing inbox, detail, and queue board with
responsive layout: desktop split-pane with draggable divider, tablet
overlay with slide-in, phone navigation stack with gesture support.
Full keyboard flow, motion design with prefers-reduced-motion degradation,
density/theme toggles persisted to localStorage.

Closes #15"
```

---

## Post-Implementation

After all tasks are complete:

1. **Update `tsconfig.json`** — verify all project references are listed
2. **Run full build** — `yarn build` must pass with no errors
3. **Run full test suite** — `yarn test` across all workspaces
4. **Update CLAUDE.md** — add new components to Key Directories table
5. **implementation-doc-sync** — sync any doc changes from implementation
