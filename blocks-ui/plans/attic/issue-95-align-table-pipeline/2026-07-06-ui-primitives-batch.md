# UI Primitives Batch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #13 — Implement Approval Gate component
**Issue group:** #13, #12, #8, #16

**Goal:** Build three new Lit Web Components (sla-indicator, kpi-metric-row, approval-gate), two core utilities (SharedTimerController, blocks-confirm-dialog), and a cleanup pass across existing components from the Epic #4 review.

**Architecture:** Bottom-up by dependency: core utilities first, then standalone primitives (#8, #12), then the component that consumes them (#13), then cleanup (#16). Each new component follows the established Yarn workspace monorepo pattern with TypeScript project references. All components use the `pages-event` bus for cross-component communication with per-component topic constants.

**Tech Stack:** Lit 3, TypeScript 5.6+, Vitest, jsdom, @open-wc/testing, Yarn 4 workspaces

## Global Constraints

- All CSS values use `var(--blocks-*)` tokens with fallback values — no hardcoded pixels
- CustomEvents crossing shadow DOM: `bubbles: true` + `composed: true` (protocol: custom-event-shadow-dom)
- Reactive collections must be replaced, never mutated in place (protocol: lit-immutable-collections)
- `.js` extensions on all TypeScript import paths (ESM convention)
- `@property()` for public API, `@state()` with `_` prefix for internal state
- Subscribe in `connectedCallback`, unsubscribe in `disconnectedCallback`
- Every component implements `configure(props)` for programmatic setup
- `HTMLElementTagNameMap` augmentation on every custom element
- `@media (prefers-reduced-motion: reduce)` on all animations

---

### Task 1: SharedTimerController

Core utility in `packages/blocks-ui-core/src/timers/`. A single module-level `setInterval` at 1s serves all subscribers. Starts when first subscriber connects, stops when last disconnects. `visibilitychange` pause/resume handled once on the shared timer.

**Files:**
- Create: `packages/blocks-ui-core/src/timers/shared-timer-controller.ts`
- Create: `packages/blocks-ui-core/src/timers/shared-timer-controller.test.ts`
- Create: `packages/blocks-ui-core/src/timers/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts` — add `export * from './timers/index.js';`

**Interfaces:**
- Produces: `subscribe(callback: () => void): void`, `unsubscribe(callback: () => void): void` — used by sla-indicator (Task 4)

- [ ] **Step 1: Write the failing tests**

```typescript
// packages/blocks-ui-core/src/timers/shared-timer-controller.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { subscribe, unsubscribe } from './shared-timer-controller.js';

describe('SharedTimerController', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it('calls subscriber every second', () => {
    const cb = vi.fn();
    subscribe(cb);
    vi.advanceTimersByTime(3000);
    expect(cb).toHaveBeenCalledTimes(3);
    unsubscribe(cb);
  });

  it('does not tick when no subscribers', () => {
    const cb = vi.fn();
    subscribe(cb);
    unsubscribe(cb);
    vi.advanceTimersByTime(3000);
    expect(cb).toHaveBeenCalledTimes(0);
  });

  it('shares a single interval across multiple subscribers', () => {
    const cb1 = vi.fn();
    const cb2 = vi.fn();
    subscribe(cb1);
    subscribe(cb2);
    vi.advanceTimersByTime(1000);
    expect(cb1).toHaveBeenCalledTimes(1);
    expect(cb2).toHaveBeenCalledTimes(1);
    unsubscribe(cb1);
    vi.advanceTimersByTime(1000);
    expect(cb1).toHaveBeenCalledTimes(1);
    expect(cb2).toHaveBeenCalledTimes(2);
    unsubscribe(cb2);
  });

  it('stops interval when last subscriber unsubscribes', () => {
    const cb = vi.fn();
    subscribe(cb);
    vi.advanceTimersByTime(1000);
    unsubscribe(cb);
    vi.advanceTimersByTime(5000);
    expect(cb).toHaveBeenCalledTimes(1);
  });

  it('restarts interval when new subscriber arrives after all unsubscribed', () => {
    const cb1 = vi.fn();
    subscribe(cb1);
    vi.advanceTimersByTime(1000);
    unsubscribe(cb1);

    const cb2 = vi.fn();
    subscribe(cb2);
    vi.advanceTimersByTime(2000);
    expect(cb2).toHaveBeenCalledTimes(2);
    unsubscribe(cb2);
  });

  it('pauses when document is hidden', () => {
    const cb = vi.fn();
    subscribe(cb);
    Object.defineProperty(document, 'hidden', { value: true, configurable: true });
    document.dispatchEvent(new Event('visibilitychange'));
    vi.advanceTimersByTime(3000);
    expect(cb).toHaveBeenCalledTimes(0);

    Object.defineProperty(document, 'hidden', { value: false, configurable: true });
    document.dispatchEvent(new Event('visibilitychange'));
    vi.advanceTimersByTime(1000);
    expect(cb).toHaveBeenCalledTimes(1);
    unsubscribe(cb);
  });

  it('ignores duplicate subscribe calls', () => {
    const cb = vi.fn();
    subscribe(cb);
    subscribe(cb);
    vi.advanceTimersByTime(1000);
    expect(cb).toHaveBeenCalledTimes(1);
    unsubscribe(cb);
  });

  it('ignores unsubscribe for unknown callback', () => {
    const cb = vi.fn();
    expect(() => unsubscribe(cb)).not.toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run packages/blocks-ui-core/src/timers/shared-timer-controller.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Write implementation**

```typescript
// packages/blocks-ui-core/src/timers/shared-timer-controller.ts
const subscribers = new Set<() => void>();
let intervalId: ReturnType<typeof setInterval> | null = null;

function tick(): void {
  for (const cb of subscribers) cb();
}

function start(): void {
  if (intervalId !== null) return;
  intervalId = setInterval(tick, 1000);
}

function stop(): void {
  if (intervalId === null) return;
  clearInterval(intervalId);
  intervalId = null;
}

function handleVisibility(): void {
  if (subscribers.size === 0) return;
  if (document.hidden) {
    stop();
  } else {
    start();
  }
}

document.addEventListener('visibilitychange', handleVisibility);

export function subscribe(callback: () => void): void {
  if (subscribers.has(callback)) return;
  subscribers.add(callback);
  if (subscribers.size === 1 && !document.hidden) start();
}

export function unsubscribe(callback: () => void): void {
  subscribers.delete(callback);
  if (subscribers.size === 0) stop();
}
```

```typescript
// packages/blocks-ui-core/src/timers/index.ts
export { subscribe, unsubscribe } from './shared-timer-controller.js';
```

- [ ] **Step 4: Add export to core barrel**

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './timers/index.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run packages/blocks-ui-core/src/timers/shared-timer-controller.test.ts`
Expected: all 8 tests PASS

- [ ] **Step 6: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/timers/ packages/blocks-ui-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add SharedTimerController — single shared 1s interval for all sla-indicator instances  #8"
```

---

### Task 2: blocks-confirm-dialog

Lightweight confirmation dialog in `packages/blocks-ui-core/src/confirm-dialog/`. Reused by approval-gate (#13) and batch cancel fix (#16).

**Files:**
- Create: `packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.ts`
- Create: `packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.test.ts`
- Create: `packages/blocks-ui-core/src/confirm-dialog/index.ts`
- Modify: `packages/blocks-ui-core/src/index.ts` — add `export * from './confirm-dialog/index.js';`

**Interfaces:**
- Consumes: `FocusTrapMixin` from `../mixins/focus-trap.js`
- Produces: `<blocks-confirm-dialog>` element with properties `open`, `heading`, `message`, `confirmLabel`, `cancelLabel`, `confirmVariant`, `showReason`, `persistent`; events `confirm` (detail: `{ reason?: string }`), `cancel`

- [ ] **Step 1: Write the failing tests**

```typescript
// packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import './blocks-confirm-dialog.js';

type ConfirmDialog = HTMLElement & {
  open: boolean;
  heading: string;
  message: string;
  confirmLabel: string;
  cancelLabel: string;
  confirmVariant: 'success' | 'danger' | 'neutral';
  showReason: boolean;
  persistent: boolean;
  updateComplete: Promise<boolean>;
};

describe('blocks-confirm-dialog', () => {
  let el: ConfirmDialog;

  beforeEach(async () => {
    el = document.createElement('blocks-confirm-dialog') as ConfirmDialog;
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('is hidden when open is false', () => {
    expect(el.shadowRoot!.querySelector('.overlay')).toBeNull();
  });

  it('renders overlay and dialog when open is true', async () => {
    el.open = true;
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('.overlay')).toBeTruthy();
    expect(el.shadowRoot!.querySelector('[role="alertdialog"]')).toBeTruthy();
  });

  it('renders heading and message', async () => {
    el.heading = 'Delete item?';
    el.message = 'This cannot be undone.';
    el.open = true;
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Delete item?');
    expect(el.shadowRoot!.textContent).toContain('This cannot be undone.');
  });

  it('renders custom button labels', async () => {
    el.confirmLabel = 'Yes, delete';
    el.cancelLabel = 'Keep it';
    el.open = true;
    await el.updateComplete;
    const buttons = el.shadowRoot!.querySelectorAll('button');
    const labels = Array.from(buttons).map(b => b.textContent!.trim());
    expect(labels).toContain('Yes, delete');
    expect(labels).toContain('Keep it');
  });

  it('emits confirm event on confirm button click', async () => {
    el.open = true;
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('confirm', handler);
    el.shadowRoot!.querySelector<HTMLButtonElement>('.btn-confirm')!.click();
    expect(handler).toHaveBeenCalledTimes(1);
  });

  it('emits cancel event on cancel button click', async () => {
    el.open = true;
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('cancel', handler);
    el.shadowRoot!.querySelector<HTMLButtonElement>('.btn-cancel')!.click();
    expect(handler).toHaveBeenCalledTimes(1);
  });

  it('emits cancel on Escape key', async () => {
    el.open = true;
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('cancel', handler);
    document.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape' }));
    expect(handler).toHaveBeenCalledTimes(1);
  });

  it('emits cancel on overlay click when not persistent', async () => {
    el.open = true;
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('cancel', handler);
    el.shadowRoot!.querySelector<HTMLElement>('.overlay')!.click();
    expect(handler).toHaveBeenCalledTimes(1);
  });

  it('does NOT emit cancel on overlay click when persistent', async () => {
    el.persistent = true;
    el.open = true;
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('cancel', handler);
    el.shadowRoot!.querySelector<HTMLElement>('.overlay')!.click();
    expect(handler).not.toHaveBeenCalled();
  });

  it('still emits cancel on Escape when persistent', async () => {
    el.persistent = true;
    el.open = true;
    await el.updateComplete;
    const handler = vi.fn();
    el.addEventListener('cancel', handler);
    document.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape' }));
    expect(handler).toHaveBeenCalledTimes(1);
  });

  it('shows reason textarea when showReason is true', async () => {
    el.showReason = true;
    el.open = true;
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('textarea')).toBeTruthy();
  });

  it('includes reason in confirm detail', async () => {
    el.showReason = true;
    el.open = true;
    await el.updateComplete;
    const textarea = el.shadowRoot!.querySelector('textarea')!;
    textarea.value = 'Not ready yet';
    textarea.dispatchEvent(new Event('input'));
    const handler = vi.fn();
    el.addEventListener('confirm', handler);
    el.shadowRoot!.querySelector<HTMLButtonElement>('.btn-confirm')!.click();
    expect(handler.mock.calls[0][0].detail.reason).toBe('Not ready yet');
  });

  it('has aria-modal and aria-labelledby', async () => {
    el.open = true;
    await el.updateComplete;
    const dialog = el.shadowRoot!.querySelector('[role="alertdialog"]')!;
    expect(dialog.getAttribute('aria-modal')).toBe('true');
    expect(dialog.getAttribute('aria-labelledby')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Write implementation**

```typescript
// packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { FocusTrapMixin } from '../mixins/focus-trap.js';

@customElement('blocks-confirm-dialog')
export class BlocksConfirmDialog extends FocusTrapMixin(LitElement) {
  @property({ type: Boolean, reflect: true }) open = false;
  @property({ type: String }) heading = 'Confirm';
  @property({ type: String }) message = '';
  @property({ type: String }) confirmLabel = 'Confirm';
  @property({ type: String }) cancelLabel = 'Cancel';
  @property({ type: String }) confirmVariant: 'success' | 'danger' | 'neutral' = 'danger';
  @property({ type: Boolean }) showReason = false;
  @property({ type: Boolean }) persistent = false;

  @state() private _reason = '';

  private _boundEscape = this._handleEscape.bind(this);

  static override styles = css`
    :host { display: contents; }

    .overlay {
      position: fixed;
      inset: 0;
      background: oklch(0 0 0 / 0.5);
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 1000;
      animation: fade-in var(--blocks-duration-fast, 120ms) var(--blocks-ease-out);
    }

    .dialog {
      background: var(--blocks-neutral-1, #fff);
      border-radius: var(--blocks-radius-lg, 8px);
      box-shadow: var(--blocks-elevation-3, 0 8px 30px oklch(0 0 0 / 0.12));
      padding: var(--blocks-space-6, 24px);
      min-width: 320px;
      max-width: 480px;
      animation: scale-in var(--blocks-duration-normal, 200ms) var(--blocks-ease-out);
    }

    .heading {
      margin: 0 0 var(--blocks-space-2, 8px);
      font-size: var(--blocks-font-size-lg, 16px);
      font-weight: var(--blocks-font-weight-semibold, 600);
      color: var(--blocks-neutral-12, #111);
    }

    .message {
      color: var(--blocks-neutral-11, #666);
      font-size: var(--blocks-font-size-base, 14px);
      line-height: var(--blocks-line-height-relaxed, 1.6);
      margin-bottom: var(--blocks-space-5, 20px);
    }

    textarea {
      width: 100%;
      min-height: 60px;
      padding: var(--blocks-space-2, 8px);
      border: 1px solid var(--blocks-neutral-6, #ccc);
      border-radius: var(--blocks-radius-sm, 4px);
      font-family: inherit;
      font-size: var(--blocks-font-size-base, 14px);
      resize: vertical;
      margin-bottom: var(--blocks-space-4, 16px);
      background: var(--blocks-neutral-1, #fff);
      color: var(--blocks-neutral-12, #111);
    }

    textarea:focus {
      outline: 2px solid var(--blocks-accent-9, #2563eb);
      outline-offset: -1px;
      border-color: var(--blocks-accent-9, #2563eb);
    }

    .actions {
      display: flex;
      justify-content: flex-end;
      gap: var(--blocks-space-2, 8px);
    }

    button {
      padding: var(--blocks-space-1.5, 6px) var(--blocks-space-4, 16px);
      border-radius: var(--blocks-radius-sm, 4px);
      font-size: var(--blocks-font-size-base, 14px);
      font-weight: var(--blocks-font-weight-medium, 500);
      cursor: pointer;
      border: 1px solid transparent;
      transition: background var(--blocks-duration-fast, 120ms) var(--blocks-ease-out);
    }

    .btn-cancel {
      background: var(--blocks-neutral-3, #f5f5f5);
      color: var(--blocks-neutral-11, #666);
      border-color: var(--blocks-neutral-6, #ccc);
    }

    .btn-cancel:hover { background: var(--blocks-neutral-4, #e5e5e5); }

    .btn-confirm.variant-danger {
      background: var(--blocks-danger-9, #dc2626);
      color: #fff;
    }

    .btn-confirm.variant-danger:hover { background: var(--blocks-danger-10, #b91c1c); }

    .btn-confirm.variant-success {
      background: var(--blocks-success-9, #16a34a);
      color: #fff;
    }

    .btn-confirm.variant-success:hover { background: var(--blocks-success-10, #15803d); }

    .btn-confirm.variant-neutral {
      background: var(--blocks-neutral-9, #888);
      color: #fff;
    }

    .btn-confirm.variant-neutral:hover { background: var(--blocks-neutral-10, #666); }

    @keyframes fade-in { from { opacity: 0; } }
    @keyframes scale-in { from { transform: scale(0.95); opacity: 0; } }

    @media (prefers-reduced-motion: reduce) {
      .overlay, .dialog { animation: none; }
      button { transition: none; }
    }
  `;

  override render() {
    if (!this.open) return nothing;

    return html`
      <div class="overlay" @click=${this._handleOverlayClick}>
        <div
          class="dialog"
          role="alertdialog"
          aria-modal="true"
          aria-labelledby="confirm-heading"
          @click=${(e: Event) => e.stopPropagation()}
        >
          <h2 class="heading" id="confirm-heading">${this.heading}</h2>
          ${this.message ? html`<p class="message">${this.message}</p>` : nothing}
          ${this.showReason ? html`
            <textarea
              placeholder="Reason (optional)"
              .value=${this._reason}
              @input=${(e: Event) => { this._reason = (e.target as HTMLTextAreaElement).value; }}
            ></textarea>
          ` : nothing}
          <div class="actions">
            <button class="btn-cancel" @click=${this._handleCancel}>${this.cancelLabel}</button>
            <button class="btn-confirm variant-${this.confirmVariant}" @click=${this._handleConfirm}>${this.confirmLabel}</button>
          </div>
        </div>
      </div>
    `;
  }

  override updated(changed: Map<string, unknown>): void {
    if (changed.has('open')) {
      if (this.open) {
        document.addEventListener('keydown', this._boundEscape);
        const dialog = this.shadowRoot?.querySelector<HTMLElement>('.dialog');
        if (dialog) this.trapFocus(dialog);
      } else {
        document.removeEventListener('keydown', this._boundEscape);
        this.releaseFocus();
        this._reason = '';
      }
    }
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    document.removeEventListener('keydown', this._boundEscape);
  }

  private _handleOverlayClick(): void {
    if (!this.persistent) this._handleCancel();
  }

  private _handleConfirm(): void {
    const detail: { reason?: string } = {};
    if (this.showReason && this._reason) detail.reason = this._reason;
    this.dispatchEvent(new CustomEvent('confirm', { bubbles: true, composed: true, detail }));
  }

  private _handleCancel(): void {
    this.dispatchEvent(new CustomEvent('cancel', { bubbles: true, composed: true }));
  }

  private _handleEscape(e: KeyboardEvent): void {
    if (e.key === 'Escape') this._handleCancel();
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'blocks-confirm-dialog': BlocksConfirmDialog;
  }
}
```

```typescript
// packages/blocks-ui-core/src/confirm-dialog/index.ts
export { BlocksConfirmDialog } from './blocks-confirm-dialog.js';
```

- [ ] **Step 4: Add export to core barrel**

Add to `packages/blocks-ui-core/src/index.ts`:
```typescript
export * from './confirm-dialog/index.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.test.ts`
Expected: all 13 tests PASS

- [ ] **Step 6: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/confirm-dialog/ packages/blocks-ui-core/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add blocks-confirm-dialog — reusable confirmation with persistent mode and reason field  #13"
```

---

### Task 3: Schema-form Type Unification + Date/DateTime Fields + Edit Mode Tests

Unify `FieldSchema` and `SchemaObject` into a single type. Define `FieldRendererElement` interface. Add date/datetime format handling. Add comprehensive edit mode tests.

**Files:**
- Create: `packages/blocks-ui-core/src/schema-form/types.ts`
- Modify: `packages/blocks-ui-core/src/schema-form/field-renderers.ts` — import from `./types.js`, add date/datetime
- Modify: `packages/blocks-ui-core/src/schema-form/schema-form.ts` — import from `./types.js`, replace `as any` casts
- Modify: `packages/blocks-ui-core/src/schema-form/field-registry.ts` — use `FieldRendererElement`
- Modify: `packages/blocks-ui-core/src/schema-form/index.ts` — export new types
- Modify: `packages/blocks-ui-core/src/schema-form/schema-form.test.ts` — add edit mode tests

**Interfaces:**
- Produces: `FieldSchema`, `FieldRendererElement` types exported from `schema-form/index.ts`

- [ ] **Step 1: Create the unified types file**

```typescript
// packages/blocks-ui-core/src/schema-form/types.ts
export interface FieldSchema {
  readonly type?: string;
  readonly format?: string;
  readonly enum?: readonly string[];
  readonly maxLength?: number;
  readonly properties?: Readonly<Record<string, FieldSchema>>;
  readonly items?: FieldSchema;
  readonly required?: readonly string[];
}

export interface FieldRendererElement extends HTMLElement {
  value: unknown;
  schema: FieldSchema;
  mode: 'display' | 'edit';
}
```

- [ ] **Step 2: Update field-registry.ts to use FieldRendererElement**

Replace the entire `field-registry.ts`:

```typescript
// packages/blocks-ui-core/src/schema-form/field-registry.ts
import type { FieldRendererElement } from './types.js';

type FieldRendererConstructor = new () => FieldRendererElement;
const registry = new Map<string, FieldRendererConstructor>();

export function registerFieldRenderer(format: string, component: FieldRendererConstructor): void {
  registry.set(format, component);
}

export function getFieldRenderer(format: string): FieldRendererConstructor | undefined {
  return registry.get(format);
}

export function hasFieldRenderer(format: string): boolean {
  return registry.has(format);
}
```

- [ ] **Step 3: Update field-renderers.ts — use unified type, add date/datetime**

Replace the local `FieldSchema` interface import and add date/datetime handling:

```typescript
// packages/blocks-ui-core/src/schema-form/field-renderers.ts
import { html, type TemplateResult } from 'lit';
import type { FieldSchema } from './types.js';

const dateFormatter = new Intl.DateTimeFormat(undefined, { dateStyle: 'medium' });
const dateTimeFormatter = new Intl.DateTimeFormat(undefined, { dateStyle: 'medium', timeStyle: 'short' });

function formatDateValue(value: unknown, format: string): string {
  if (value === null || value === undefined || value === '') return '';
  const date = new Date(String(value));
  if (isNaN(date.getTime())) return String(value);
  return format === 'date-time' ? dateTimeFormatter.format(date) : dateFormatter.format(date);
}

function toDateInputValue(value: unknown): string {
  if (value === null || value === undefined || value === '') return '';
  const date = new Date(String(value));
  if (isNaN(date.getTime())) return '';
  return date.toISOString().split('T')[0]!;
}

function toDateTimeInputValue(value: unknown): string {
  if (value === null || value === undefined || value === '') return '';
  const date = new Date(String(value));
  if (isNaN(date.getTime())) return '';
  return date.toISOString().slice(0, 16);
}

export function renderDisplayField(
  key: string,
  schema: FieldSchema,
  value: unknown,
): TemplateResult {
  if (value === null || value === undefined) {
    return html`<div class="field"><span class="label">${key}</span><span class="value muted">—</span></div>`;
  }

  if (schema.format === 'date' || schema.format === 'date-time') {
    return html`<div class="field"><span class="label">${key}</span><span class="value">${formatDateValue(value, schema.format)}</span></div>`;
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

  if (schema.format === 'date') {
    return html`
      <div class="field">
        <label for="${key}">${key}</label>
        <input id="${key}" type="date" .value=${toDateInputValue(value)} @input=${(e: Event) => onChange(key, (e.target as HTMLInputElement).value)} />
      </div>`;
  }

  if (schema.format === 'date-time') {
    return html`
      <div class="field">
        <label for="${key}">${key}</label>
        <input id="${key}" type="datetime-local" .value=${toDateTimeInputValue(value)} @input=${(e: Event) => onChange(key, (e.target as HTMLInputElement).value)} />
      </div>`;
  }

  if (schema.type === 'string' && (schema.maxLength ?? 0) > 200) {
    return html`
      <div class="field">
        <label for="${key}">${key}</label>
        <textarea id="${key}" .value=${String(value ?? '')} @input=${(e: Event) => onChange(key, (e.target as HTMLTextAreaElement).value)}></textarea>
      </div>`;
  }

  return html`
    <div class="field">
      <label for="${key}">${key}</label>
      <input id="${key}" type="text" .value=${String(value ?? '')} @input=${(e: Event) => onChange(key, (e.target as HTMLInputElement).value)} />
    </div>`;
}

export type { FieldSchema };
```

- [ ] **Step 4: Update schema-form.ts — use unified types, remove `as any` casts**

Replace the local `SchemaObject` interface and fix the `as any` casts:

In `schema-form.ts`, replace the local `SchemaObject` interface import line and the three `as any` casts:

```typescript
// At the top, replace local SchemaObject with import from types
import type { FieldSchema, FieldRendererElement } from './types.js';

// Replace the SchemaObject interface declaration (lines ~6-14) entirely — it's now imported as FieldSchema

// Change all occurrences of SchemaObject to FieldSchema:
// @property({ type: Object }) schema: FieldSchema | null = null;

// Fix the three as any casts in the render method:
// Replace:
//   (el as any).value = dataSource[key];
//   (el as any).schema = fieldSchema;
//   (el as any).mode = this.mode;
// With:
//   const renderer = el as FieldRendererElement;
//   renderer.value = dataSource[key];
//   renderer.schema = fieldSchema;
//   renderer.mode = this.mode;
```

- [ ] **Step 5: Update schema-form/index.ts — export new types**

```typescript
// packages/blocks-ui-core/src/schema-form/index.ts
export { SchemaForm } from './schema-form.js';
export { registerFieldRenderer } from './field-registry.js';
export type { FieldSchema, FieldRendererElement } from './types.js';
```

- [ ] **Step 6: Write edit mode tests**

Append to `packages/blocks-ui-core/src/schema-form/schema-form.test.ts`:

```typescript
  describe('edit mode', () => {
    beforeEach(async () => {
      el.mode = 'edit';
      await el.updateComplete;
    });

    it('renders text input for string fields', () => {
      const input = el.shadowRoot!.querySelector<HTMLInputElement>('input[id="title"]');
      expect(input).toBeTruthy();
      expect(input!.type).toBe('text');
      expect(input!.value).toBe('Test Item');
    });

    it('renders number input for number fields', () => {
      const input = el.shadowRoot!.querySelector<HTMLInputElement>('input[id="count"]');
      expect(input).toBeTruthy();
      expect(input!.type).toBe('number');
    });

    it('renders checkbox for boolean fields', () => {
      const input = el.shadowRoot!.querySelector<HTMLInputElement>('input[type="checkbox"]');
      expect(input).toBeTruthy();
      expect(input!.checked).toBe(true);
    });

    it('renders select for enum fields', () => {
      const select = el.shadowRoot!.querySelector<HTMLSelectElement>('select[id="status"]');
      expect(select).toBeTruthy();
      expect(select!.value).toBe('open');
      const options = select!.querySelectorAll('option');
      expect(options.length).toBe(2);
    });

    it('emits schema-form-change on field edit', async () => {
      const handler = vi.fn();
      el.addEventListener('schema-form-change', handler);
      const input = el.shadowRoot!.querySelector<HTMLInputElement>('input[id="title"]')!;
      input.value = 'Updated';
      input.dispatchEvent(new Event('input'));
      expect(handler).toHaveBeenCalledTimes(1);
      expect(handler.mock.calls[0][0].detail.key).toBe('title');
      expect(handler.mock.calls[0][0].detail.value).toBe('Updated');
    });

    it('submit() returns data and emits schema-form-submit', () => {
      const handler = vi.fn();
      el.addEventListener('schema-form-submit', handler);
      const result = (el as any).submit();
      expect(result).toBeTruthy();
      expect(result.title).toBe('Test Item');
      expect(handler).toHaveBeenCalledTimes(1);
    });

    it('submit() returns null for missing required fields', async () => {
      el.data = { count: 42, active: true, status: 'open' };
      await el.updateComplete;
      const result = (el as any).submit();
      expect(result).toBeNull();
    });

    it('renders textarea for long strings', async () => {
      el.schema = {
        type: 'object',
        properties: { notes: { type: 'string', maxLength: 500 } },
      };
      el.data = { notes: 'Some long text' };
      await el.updateComplete;
      expect(el.shadowRoot!.querySelector('textarea')).toBeTruthy();
    });
  });

  describe('date fields', () => {
    const dateSchema = {
      type: 'object',
      properties: {
        born: { type: 'string', format: 'date' },
        created: { type: 'string', format: 'date-time' },
      },
    };

    it('renders formatted date in display mode', async () => {
      el.schema = dateSchema;
      el.data = { born: '2000-01-15', created: '2026-07-06T14:30:00Z' };
      el.mode = 'display';
      await el.updateComplete;
      const text = el.shadowRoot!.textContent!;
      expect(text).toContain('2000');
      expect(text).toContain('2026');
    });

    it('renders date input in edit mode', async () => {
      el.schema = dateSchema;
      el.data = { born: '2000-01-15', created: '2026-07-06T14:30:00Z' };
      el.mode = 'edit';
      await el.updateComplete;
      const dateInput = el.shadowRoot!.querySelector<HTMLInputElement>('input[type="date"]');
      expect(dateInput).toBeTruthy();
      expect(dateInput!.value).toBe('2000-01-15');
    });

    it('renders datetime-local input in edit mode', async () => {
      el.schema = dateSchema;
      el.data = { born: '2000-01-15', created: '2026-07-06T14:30:00Z' };
      el.mode = 'edit';
      await el.updateComplete;
      const dtInput = el.shadowRoot!.querySelector<HTMLInputElement>('input[type="datetime-local"]');
      expect(dtInput).toBeTruthy();
    });
  });
```

- [ ] **Step 7: Run all schema-form tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run packages/blocks-ui-core/src/schema-form/schema-form.test.ts`
Expected: all tests PASS (original 3 + new 10)

- [ ] **Step 8: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS — the `as any` casts are gone

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add packages/blocks-ui-core/src/schema-form/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: unify schema-form types, add date/datetime fields, add edit mode tests  #16"
```

---

### Task 4: sla-indicator Component

New component `<sla-indicator>` — countdown, breach state, escalation badge, threshold-based colour transitions.

**Files:**
- Create: `components/sla-indicator/package.json`
- Create: `components/sla-indicator/tsconfig.json`
- Create: `components/sla-indicator/tsconfig.build.json`
- Create: `components/sla-indicator/vitest.config.ts`
- Create: `components/sla-indicator/src/index.ts`
- Create: `components/sla-indicator/src/sla-indicator.ts`
- Create: `components/sla-indicator/src/sla-indicator.test.ts`
- Modify: `tsconfig.json` (root) — add reference

**Interfaces:**
- Consumes: `subscribe`/`unsubscribe` from `@casehubio/blocks-ui-core` (Task 1), `emitPagesEvent` from core
- Produces: `<sla-indicator>` element, `SlaIndicatorTopics.STATE_CHANGED` constant, `SlaIndicator` class — used by approval-gate (Task 6)

- [ ] **Step 1: Scaffold the component package**

```json
// components/sla-indicator/package.json
{
  "name": "@casehubio/blocks-ui-sla-indicator",
  "version": "0.1.0",
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
// components/sla-indicator/tsconfig.json
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
// components/sla-indicator/tsconfig.build.json
{ "extends": "./tsconfig.json", "exclude": ["src/**/*.test.ts"] }
```

```typescript
// components/sla-indicator/vitest.config.ts
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { environment: 'jsdom', globals: true } });
```

```typescript
// components/sla-indicator/src/index.ts
export { SlaIndicator, SlaIndicatorTopics } from './sla-indicator.js';
```

Add to root `tsconfig.json` references:
```json
{ "path": "components/sla-indicator" }
```

- [ ] **Step 2: Write the failing tests**

```typescript
// components/sla-indicator/src/sla-indicator.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import './sla-indicator.js';

type SlaIndicatorEl = HTMLElement & {
  deadline: string;
  slaWindow: number | null;
  warningThreshold: number;
  criticalThreshold: number;
  escalationStage: string | null;
  compact: boolean;
  updateComplete: Promise<boolean>;
  configure: (props: Record<string, unknown>) => void;
};

describe('sla-indicator', () => {
  let el: SlaIndicatorEl;

  beforeEach(async () => {
    vi.useFakeTimers();
    el = document.createElement('sla-indicator') as SlaIndicatorEl;
    document.body.appendChild(el);
  });

  afterEach(() => {
    el.remove();
    vi.useRealTimers();
  });

  it('renders countdown for a future deadline', async () => {
    el.deadline = new Date(Date.now() + 2 * 86400000 + 4 * 3600000).toISOString();
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('2d');
  });

  it('renders breach state for a past deadline', async () => {
    el.deadline = new Date(Date.now() - 3 * 3600000).toISOString();
    await el.updateComplete;
    expect(el.shadowRoot!.textContent!.toLowerCase()).toContain('breach');
  });

  it('transitions through warning state', async () => {
    el.slaWindow = 3600000;
    el.warningThreshold = 0.5;
    el.deadline = new Date(Date.now() + 1200000).toISOString();
    await el.updateComplete;
    const host = el.shadowRoot!.querySelector('.sla-indicator')!;
    expect(host.classList.contains('warning')).toBe(true);
  });

  it('transitions through critical state', async () => {
    el.slaWindow = 3600000;
    el.criticalThreshold = 0.1;
    el.deadline = new Date(Date.now() + 60000).toISOString();
    await el.updateComplete;
    const host = el.shadowRoot!.querySelector('.sla-indicator')!;
    expect(host.classList.contains('critical')).toBe(true);
  });

  it('uses absolute fallback thresholds when no slaWindow', async () => {
    el.deadline = new Date(Date.now() + 600000).toISOString();
    await el.updateComplete;
    const host = el.shadowRoot!.querySelector('.sla-indicator')!;
    expect(host.classList.contains('critical')).toBe(true);
  });

  it('renders escalation badge when escalationStage is set', async () => {
    el.deadline = new Date(Date.now() - 3600000).toISOString();
    el.escalationStage = 'L2';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('L2');
  });

  it('emits initial state event on connectedCallback', async () => {
    const handler = vi.fn();
    document.addEventListener('pages-event', handler);
    el.deadline = new Date(Date.now() + 86400000).toISOString();
    await el.updateComplete;
    const stateEvent = handler.mock.calls.find(
      (c: any) => c[0].detail.topic === 'sla.state-changed'
    );
    expect(stateEvent).toBeTruthy();
    document.removeEventListener('pages-event', handler);
  });

  it('emits state-changed on threshold crossing', async () => {
    el.slaWindow = 60000;
    el.warningThreshold = 0.5;
    el.deadline = new Date(Date.now() + 35000).toISOString();
    await el.updateComplete;

    const handler = vi.fn();
    document.addEventListener('pages-event', handler);

    vi.advanceTimersByTime(10000);
    await el.updateComplete;

    const events = handler.mock.calls
      .filter((c: any) => c[0].detail.topic === 'sla.state-changed')
      .map((c: any) => c[0].detail.payload.state);

    expect(events).toContain('warning');
    document.removeEventListener('pages-event', handler);
  });

  it('has role="timer" and aria-label', async () => {
    el.deadline = new Date(Date.now() + 7200000).toISOString();
    await el.updateComplete;
    const indicator = el.shadowRoot!.querySelector('[role="timer"]');
    expect(indicator).toBeTruthy();
    expect(indicator!.getAttribute('aria-label')).toBeTruthy();
  });

  it('shows tooltip with absolute datetime', async () => {
    el.deadline = '2026-07-08T14:30:00Z';
    await el.updateComplete;
    const indicator = el.shadowRoot!.querySelector('[title]');
    expect(indicator).toBeTruthy();
    expect(indicator!.getAttribute('title')).toContain('2026');
  });

  it('configure() sets multiple properties', async () => {
    el.configure({ deadline: '2026-07-10T00:00:00Z', compact: false, escalationStage: 'Manager' });
    await el.updateComplete;
    expect(el.compact).toBe(false);
    expect(el.escalationStage).toBe('Manager');
  });

  it('renders minutes+seconds when under 1 hour', async () => {
    el.deadline = new Date(Date.now() + 180000).toISOString();
    await el.updateComplete;
    const text = el.shadowRoot!.textContent!;
    expect(text).toMatch(/\d+m/);
  });

  it('updates countdown on timer tick', async () => {
    el.deadline = new Date(Date.now() + 120000).toISOString();
    await el.updateComplete;
    const before = el.shadowRoot!.textContent;
    vi.advanceTimersByTime(60000);
    await el.updateComplete;
    const after = el.shadowRoot!.textContent;
    expect(after).not.toBe(before);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn install && yarn vitest run components/sla-indicator/src/sla-indicator.test.ts`
Expected: FAIL — module not found

- [ ] **Step 4: Write implementation**

```typescript
// components/sla-indicator/src/sla-indicator.ts
import { LitElement, html, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { subscribe, unsubscribe, emitPagesEvent } from '@casehubio/blocks-ui-core';

export const SlaIndicatorTopics = {
  STATE_CHANGED: 'sla.state-changed',
} as const;

type SlaState = 'normal' | 'warning' | 'critical' | 'breached';

const ABSOLUTE_WARNING_MS = 3600000;
const ABSOLUTE_CRITICAL_MS = 900000;

function formatCountdown(ms: number): string {
  if (ms <= 0) return '';
  const totalSeconds = Math.floor(ms / 1000);
  const days = Math.floor(totalSeconds / 86400);
  const hours = Math.floor((totalSeconds % 86400) / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;

  if (days > 0) return `${days}d ${hours}h`;
  if (hours > 0) return `${hours}h ${minutes}m`;
  return `${minutes}m ${seconds}s`;
}

function formatBreach(ms: number): string {
  const elapsed = Math.abs(ms);
  const totalMinutes = Math.floor(elapsed / 60000);
  const hours = Math.floor(totalMinutes / 60);
  const days = Math.floor(hours / 24);

  if (days > 0) return `Breached ${days}d ago`;
  if (hours > 0) return `Breached ${hours}h ago`;
  return `Breached ${totalMinutes}m ago`;
}

function formatAriaLabel(ms: number): string {
  if (ms <= 0) {
    const elapsed = Math.abs(ms);
    const hours = Math.floor(elapsed / 3600000);
    const minutes = Math.floor((elapsed % 3600000) / 60000);
    if (hours > 0) return `Deadline breached ${hours} hours ${minutes} minutes ago`;
    return `Deadline breached ${minutes} minutes ago`;
  }
  const totalMinutes = Math.floor(ms / 60000);
  const days = Math.floor(totalMinutes / 1440);
  const hours = Math.floor((totalMinutes % 1440) / 60);
  const minutes = totalMinutes % 60;
  const parts: string[] = [];
  if (days > 0) parts.push(`${days} day${days !== 1 ? 's' : ''}`);
  if (hours > 0) parts.push(`${hours} hour${hours !== 1 ? 's' : ''}`);
  if (minutes > 0 || parts.length === 0) parts.push(`${minutes} minute${minutes !== 1 ? 's' : ''}`);
  return `${parts.join(' ')} remaining`;
}

@customElement('sla-indicator')
export class SlaIndicator extends LitElement {
  @property({ type: String }) deadline = '';
  @property({ type: Number, attribute: 'sla-window' }) slaWindow: number | null = null;
  @property({ type: Number, attribute: 'warning-threshold' }) warningThreshold = 0.25;
  @property({ type: Number, attribute: 'critical-threshold' }) criticalThreshold = 0.10;
  @property({ type: String, attribute: 'escalation-stage' }) escalationStage: string | null = null;
  @property({ type: Boolean }) compact = true;

  @state() private _remaining = 0;
  @state() private _state: SlaState = 'normal';

  private _lastEmittedState: SlaState | null = null;

  static override styles = css`
    :host { display: inline-flex; align-items: center; }

    .sla-indicator {
      display: inline-flex;
      align-items: center;
      gap: var(--blocks-space-1.5, 6px);
      font-family: var(--blocks-font-family, system-ui);
      font-variant-numeric: tabular-nums;
    }

    .countdown {
      font-weight: var(--blocks-font-weight-medium, 500);
    }

    .compact .countdown { font-size: var(--blocks-font-size-sm, 12px); }
    .expanded .countdown { font-size: var(--blocks-font-size-base, 14px); }

    .normal .countdown { color: var(--blocks-success-9, #16a34a); }
    .warning .countdown { color: var(--blocks-warning-9, #d97706); }
    .critical .countdown { color: var(--blocks-danger-9, #dc2626); }
    .breached .countdown {
      color: var(--blocks-danger-9, #dc2626);
      animation: pulse 2s ease-in-out infinite;
    }

    .escalation-badge {
      font-size: var(--blocks-font-size-xs, 11px);
      padding: 1px var(--blocks-space-1.5, 6px);
      border-radius: var(--blocks-radius-sm, 4px);
      background: var(--blocks-neutral-4, #e5e5e5);
      color: var(--blocks-danger-9, #dc2626);
      font-weight: var(--blocks-font-weight-medium, 500);
    }

    @keyframes pulse {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.6; }
    }

    @media (prefers-reduced-motion: reduce) {
      .breached .countdown { animation: none; }
    }
  `;

  private _tick = (): void => {
    this._update();
  };

  override connectedCallback(): void {
    super.connectedCallback();
    subscribe(this._tick);
    this._update();
  }

  override disconnectedCallback(): void {
    super.disconnectedCallback();
    unsubscribe(this._tick);
    this._lastEmittedState = null;
  }

  override willUpdate(changed: Map<string, unknown>): void {
    if (changed.has('deadline') || changed.has('slaWindow') ||
        changed.has('warningThreshold') || changed.has('criticalThreshold')) {
      this._lastEmittedState = null;
      this._update();
    }
  }

  private _update(): void {
    if (!this.deadline) return;
    const deadlineMs = new Date(this.deadline).getTime();
    this._remaining = deadlineMs - Date.now();
    this._state = this._computeState();

    if (this._state !== this._lastEmittedState) {
      this._lastEmittedState = this._state;
      emitPagesEvent(this, SlaIndicatorTopics.STATE_CHANGED, {
        state: this._state,
        deadline: this.deadline,
      });
    }
  }

  private _computeState(): SlaState {
    if (this._remaining <= 0) return 'breached';

    if (this.slaWindow !== null && this.slaWindow > 0) {
      const fraction = this._remaining / this.slaWindow;
      if (fraction <= this.criticalThreshold) return 'critical';
      if (fraction <= this.warningThreshold) return 'warning';
      return 'normal';
    }

    if (this._remaining <= ABSOLUTE_CRITICAL_MS) return 'critical';
    if (this._remaining <= ABSOLUTE_WARNING_MS) return 'warning';
    return 'normal';
  }

  override render() {
    if (!this.deadline) return nothing;
    const text = this._remaining > 0
      ? formatCountdown(this._remaining)
      : formatBreach(this._remaining);
    const modeClass = this.compact ? 'compact' : 'expanded';
    const tooltipDate = new Date(this.deadline);
    const tooltip = `Deadline: ${tooltipDate.toISOString().replace('T', ' ').replace(/\.\d+Z$/, ' UTC')}`;

    return html`
      <span
        class="sla-indicator ${this._state} ${modeClass}"
        role="timer"
        aria-label="${formatAriaLabel(this._remaining)}"
        title="${tooltip}"
      >
        <span class="countdown">${text}</span>
        ${this.escalationStage
          ? html`<span class="escalation-badge">${this.escalationStage}</span>`
          : nothing}
      </span>
    `;
  }

  configure(props: {
    deadline?: string;
    slaWindow?: number | null;
    warningThreshold?: number;
    criticalThreshold?: number;
    escalationStage?: string | null;
    compact?: boolean;
  }): void {
    if (props.deadline !== undefined) this.deadline = props.deadline;
    if (props.slaWindow !== undefined) this.slaWindow = props.slaWindow;
    if (props.warningThreshold !== undefined) this.warningThreshold = props.warningThreshold;
    if (props.criticalThreshold !== undefined) this.criticalThreshold = props.criticalThreshold;
    if (props.escalationStage !== undefined) this.escalationStage = props.escalationStage;
    if (props.compact !== undefined) this.compact = props.compact;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'sla-indicator': SlaIndicator;
  }
}
```

- [ ] **Step 5: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run components/sla-indicator/src/sla-indicator.test.ts`
Expected: all 13 tests PASS

- [ ] **Step 6: Typecheck and build**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/sla-indicator/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add sla-indicator component — countdown, breach, escalation, threshold transitions  #8"
```

---

### Task 5: kpi-metric-row Component

New component `<kpi-metric-row>` — responsive grid of metric cards with sparklines, trends, and status colours.

**Files:**
- Create: `components/kpi-metric-row/package.json`
- Create: `components/kpi-metric-row/tsconfig.json`
- Create: `components/kpi-metric-row/tsconfig.build.json`
- Create: `components/kpi-metric-row/vitest.config.ts`
- Create: `components/kpi-metric-row/src/index.ts`
- Create: `components/kpi-metric-row/src/kpi-metric-row.ts`
- Create: `components/kpi-metric-row/src/kpi-metric-row.test.ts`
- Modify: `tsconfig.json` (root) — add reference

**Interfaces:**
- Consumes: `emitPagesEvent`, `LiveRegionMixin` from `@casehubio/blocks-ui-core`
- Produces: `<kpi-metric-row>` element, `KpiMetricRowTopics.CARD_CLICKED` constant, `MetricDefinition` type

- [ ] **Step 1: Scaffold the component package**

Same pattern as Task 4 — `package.json`, `tsconfig.json`, `tsconfig.build.json`, `vitest.config.ts` identical except name is `@casehubio/blocks-ui-kpi-metric-row`.

```typescript
// components/kpi-metric-row/src/index.ts
export { KpiMetricRow, KpiMetricRowTopics, type MetricDefinition } from './kpi-metric-row.js';
```

Add `{ "path": "components/kpi-metric-row" }` to root `tsconfig.json` references.

- [ ] **Step 2: Write the failing tests**

```typescript
// components/kpi-metric-row/src/kpi-metric-row.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import type { MetricDefinition } from './kpi-metric-row.js';
import './kpi-metric-row.js';

type KpiMetricRowEl = HTMLElement & {
  metrics: MetricDefinition[];
  endpoint: string | null;
  columns: number | null;
  updateComplete: Promise<boolean>;
  configure: (props: Record<string, unknown>) => void;
  refresh: () => Promise<void>;
};

const sampleMetrics: MetricDefinition[] = [
  { key: 'cases', value: 142, label: 'Active Cases', unit: 'cases', status: 'normal' },
  { key: 'sla', value: '98.2', label: 'SLA Compliance', unit: '%', trend: { direction: 'up', delta: '+1.3%' }, status: 'normal' },
  { key: 'backlog', value: 23, label: 'Backlog', sparkline: [30, 28, 25, 27, 23], status: 'warning' },
];

describe('kpi-metric-row', () => {
  let el: KpiMetricRowEl;

  beforeEach(async () => {
    el = document.createElement('kpi-metric-row') as KpiMetricRowEl;
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => el.remove());

  it('renders empty state when no metrics', () => {
    expect(el.shadowRoot!.textContent).toContain('No metrics available');
  });

  it('renders metric cards', async () => {
    el.metrics = sampleMetrics;
    await el.updateComplete;
    const cards = el.shadowRoot!.querySelectorAll('[role="listitem"]');
    expect(cards.length).toBe(3);
  });

  it('renders value and label', async () => {
    el.metrics = [{ key: 'test', value: 42, label: 'Test Metric' }];
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('42');
    expect(el.shadowRoot!.textContent).toContain('Test Metric');
  });

  it('renders unit suffix', async () => {
    el.metrics = [{ key: 'test', value: 98, label: 'Score', unit: '%' }];
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('%');
  });

  it('renders trend arrow and delta', async () => {
    el.metrics = [{ key: 'test', value: 50, label: 'Metric', trend: { direction: 'up', delta: '+5' } }];
    await el.updateComplete;
    const text = el.shadowRoot!.textContent!;
    expect(text).toContain('+5');
  });

  it('renders sparkline as SVG', async () => {
    el.metrics = [{ key: 'test', value: 10, label: 'Metric', sparkline: [5, 10, 8, 12, 10] }];
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('svg')).toBeTruthy();
    expect(el.shadowRoot!.querySelector('polyline')).toBeTruthy();
  });

  it('renders status border', async () => {
    el.metrics = [{ key: 'test', value: 10, label: 'Metric', status: 'warning' }];
    await el.updateComplete;
    const card = el.shadowRoot!.querySelector('[role="listitem"]')!;
    expect(card.classList.contains('status-warning')).toBe(true);
  });

  it('emits pages-event on card click', async () => {
    el.metrics = [{ key: 'test', value: 42, label: 'Test' }];
    await el.updateComplete;
    const handler = vi.fn();
    document.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector<HTMLElement>('[role="listitem"]')!.click();
    const cardEvent = handler.mock.calls.find(
      (c: any) => c[0].detail.topic === 'kpi.card-clicked'
    );
    expect(cardEvent).toBeTruthy();
    expect(cardEvent![0].detail.payload.key).toBe('test');
    document.removeEventListener('pages-event', handler);
  });

  it('cards are keyboard accessible', async () => {
    el.metrics = [{ key: 'test', value: 42, label: 'Test' }];
    await el.updateComplete;
    const card = el.shadowRoot!.querySelector<HTMLElement>('[role="listitem"]')!;
    expect(card.getAttribute('tabindex')).toBe('0');
  });

  it('has aria-label on cards combining label, value, unit', async () => {
    el.metrics = [{ key: 'test', value: 42, label: 'Active', unit: 'cases' }];
    await el.updateComplete;
    const card = el.shadowRoot!.querySelector('[role="listitem"]')!;
    const label = card.getAttribute('aria-label')!;
    expect(label).toContain('Active');
    expect(label).toContain('42');
    expect(label).toContain('cases');
  });

  it('sparklines are aria-hidden', async () => {
    el.metrics = [{ key: 'test', value: 10, label: 'M', sparkline: [1, 2, 3] }];
    await el.updateComplete;
    const svg = el.shadowRoot!.querySelector('svg');
    expect(svg!.getAttribute('aria-hidden')).toBe('true');
  });

  it('configure() sets properties', async () => {
    el.configure({ metrics: sampleMetrics, columns: 2 });
    await el.updateComplete;
    expect(el.columns).toBe(2);
    expect(el.shadowRoot!.querySelectorAll('[role="listitem"]').length).toBe(3);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn install && yarn vitest run components/kpi-metric-row/src/kpi-metric-row.test.ts`
Expected: FAIL

- [ ] **Step 4: Write implementation**

```typescript
// components/kpi-metric-row/src/kpi-metric-row.ts
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { LiveRegionMixin, emitPagesEvent } from '@casehubio/blocks-ui-core';

export const KpiMetricRowTopics = {
  CARD_CLICKED: 'kpi.card-clicked',
} as const;

export interface MetricDefinition {
  readonly key: string;
  readonly value: number | string;
  readonly label: string;
  readonly unit?: string;
  readonly trend?: { readonly direction: 'up' | 'down' | 'stable'; readonly delta: string };
  readonly sparkline?: readonly number[];
  readonly status?: 'normal' | 'warning' | 'critical';
}

const TREND_ARROWS: Record<string, string> = { up: '▲', down: '▼', stable: '—' };

function renderSparkline(data: readonly number[]): TemplateResult {
  if (data.length < 2) return html``;
  const min = Math.min(...data);
  const max = Math.max(...data);
  const range = max - min || 1;
  const w = 48;
  const h = 20;
  const points = data.map((v, i) => {
    const x = (i / (data.length - 1)) * w;
    const y = h - ((v - min) / range) * h;
    return `${x},${y}`;
  }).join(' ');

  const polygonPoints = `0,${h} ${points} ${w},${h}`;

  return html`
    <svg width="${w}" height="${h}" viewBox="0 0 ${w} ${h}" aria-hidden="true" class="sparkline">
      <defs>
        <linearGradient id="spark-fill" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%" stop-color="currentColor" stop-opacity="0.2" />
          <stop offset="100%" stop-color="currentColor" stop-opacity="0.02" />
        </linearGradient>
      </defs>
      <polygon points="${polygonPoints}" fill="url(#spark-fill)" />
      <polyline points="${points}" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linejoin="round" />
    </svg>
  `;
}

@customElement('kpi-metric-row')
export class KpiMetricRow extends LiveRegionMixin(LitElement) {
  @property({ type: Array }) metrics: MetricDefinition[] = [];
  @property({ type: String }) endpoint: string | null = null;
  @property({ type: Number }) columns: number | null = null;

  @state() private _loading = false;
  @state() private _error: string | null = null;

  static override styles = css`
    :host { display: block; font-family: var(--blocks-font-family, system-ui); }

    .grid {
      display: grid;
      gap: var(--blocks-space-3, 12px);
    }

    .empty {
      text-align: center;
      padding: var(--blocks-space-6, 24px);
      color: var(--blocks-neutral-9, #888);
      font-size: var(--blocks-font-size-base, 14px);
    }

    .error {
      text-align: center;
      padding: var(--blocks-space-4, 16px);
      color: var(--blocks-danger-9, #dc2626);
      font-size: var(--blocks-font-size-sm, 12px);
    }

    .card {
      background: var(--blocks-neutral-2, #fafafa);
      border-radius: var(--blocks-radius-md, 6px);
      padding: var(--blocks-space-4, 16px);
      border-left: 3px solid transparent;
      cursor: pointer;
      transition: background var(--blocks-duration-fast, 120ms) var(--blocks-ease-out);
      outline: none;
    }

    .card:hover { background: var(--blocks-neutral-3, #f5f5f5); }
    .card:focus-visible { outline: 2px solid var(--blocks-accent-9, #2563eb); outline-offset: 2px; }

    .card.status-normal { border-left-color: var(--blocks-success-9, #16a34a); }
    .card.status-warning { border-left-color: var(--blocks-warning-9, #d97706); }
    .card.status-critical { border-left-color: var(--blocks-danger-9, #dc2626); }

    .value-row { display: flex; align-items: baseline; gap: var(--blocks-space-1, 4px); }

    .value {
      font-size: var(--blocks-font-size-2xl, 24px);
      font-weight: var(--blocks-font-weight-bold, 700);
      color: var(--blocks-neutral-12, #111);
      font-variant-numeric: tabular-nums;
    }

    .unit {
      font-size: var(--blocks-font-size-sm, 12px);
      color: var(--blocks-neutral-9, #888);
    }

    .label {
      font-size: var(--blocks-font-size-sm, 12px);
      color: var(--blocks-neutral-9, #888);
      margin-top: var(--blocks-space-1, 4px);
    }

    .trend {
      display: inline-flex;
      align-items: center;
      gap: var(--blocks-space-0.5, 2px);
      font-size: var(--blocks-font-size-xs, 11px);
      margin-top: var(--blocks-space-1, 4px);
    }

    .trend.up { color: var(--blocks-success-9, #16a34a); }
    .trend.down { color: var(--blocks-danger-9, #dc2626); }
    .trend.stable { color: var(--blocks-neutral-9, #888); }

    .sparkline { margin-top: var(--blocks-space-2, 8px); }
    .card.status-warning .sparkline { color: var(--blocks-warning-9, #d97706); }
    .card.status-critical .sparkline { color: var(--blocks-danger-9, #dc2626); }
    .card.status-normal .sparkline { color: var(--blocks-success-9, #16a34a); }
    .sparkline { color: var(--blocks-accent-9, #2563eb); }

    .skeleton-card {
      background: var(--blocks-neutral-3, #f5f5f5);
      border-radius: var(--blocks-radius-md, 6px);
      padding: var(--blocks-space-4, 16px);
      min-height: 80px;
      animation: shimmer 1.5s ease-in-out infinite;
    }

    @keyframes shimmer {
      0%, 100% { opacity: 1; }
      50% { opacity: 0.5; }
    }

    @media (prefers-reduced-motion: reduce) {
      .card { transition: none; }
      .skeleton-card { animation: none; }
    }
  `;

  override connectedCallback(): void {
    super.connectedCallback();
    if (this.endpoint) this._fetchMetrics();
  }

  override render() {
    const gridStyle = this.columns
      ? `grid-template-columns: repeat(${this.columns}, 1fr)`
      : 'grid-template-columns: repeat(auto-fill, minmax(160px, 1fr))';

    if (this._loading) {
      const count = this.columns ?? 3;
      return html`
        <div class="grid" role="list" style="${gridStyle}">
          ${Array.from({ length: count }, () => html`<div class="skeleton-card"></div>`)}
        </div>
      `;
    }

    if (this._error) {
      return html`<div class="error">${this._error}</div>`;
    }

    if (this.metrics.length === 0) {
      return html`<div class="empty">No metrics available</div>`;
    }

    return html`
      <div class="grid" role="list" style="${gridStyle}">
        ${this.metrics.map(m => this._renderCard(m))}
      </div>
    `;
  }

  private _renderCard(m: MetricDefinition): TemplateResult {
    const statusClass = m.status ? `status-${m.status}` : '';
    const ariaLabel = [m.label, String(m.value), m.unit, m.trend ? `${m.trend.direction} ${m.trend.delta}` : '']
      .filter(Boolean).join(' ');

    return html`
      <div
        class="card ${statusClass}"
        role="listitem"
        tabindex="0"
        aria-label="${ariaLabel}"
        @click=${() => this._handleCardClick(m)}
        @keydown=${(e: KeyboardEvent) => { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); this._handleCardClick(m); } }}
      >
        <div class="value-row">
          <span class="value">${m.value}</span>
          ${m.unit ? html`<span class="unit">${m.unit}</span>` : nothing}
        </div>
        <div class="label">${m.label}</div>
        ${m.trend ? html`
          <span class="trend ${m.trend.direction}">
            ${TREND_ARROWS[m.trend.direction]} ${m.trend.delta}
          </span>
        ` : nothing}
        ${m.sparkline ? renderSparkline(m.sparkline) : nothing}
      </div>
    `;
  }

  private _handleCardClick(m: MetricDefinition): void {
    this.announce(`Selected ${m.label}: ${m.value} ${m.unit ?? ''}`);
    emitPagesEvent(this, KpiMetricRowTopics.CARD_CLICKED, {
      key: m.key,
      value: m.value,
      label: m.label,
    });
  }

  private async _fetchMetrics(): Promise<void> {
    if (!this.endpoint) return;
    this._loading = true;
    this._error = null;
    try {
      const resp = await fetch(this.endpoint);
      if (!resp.ok) throw new Error(`HTTP ${resp.status}`);
      const data = await resp.json();
      this.metrics = data as MetricDefinition[];
    } catch (e) {
      this._error = `Failed to load metrics: ${(e as Error).message}`;
    } finally {
      this._loading = false;
    }
  }

  async refresh(): Promise<void> {
    await this._fetchMetrics();
  }

  configure(props: {
    metrics?: MetricDefinition[];
    endpoint?: string | null;
    columns?: number | null;
  }): void {
    if (props.metrics !== undefined) this.metrics = props.metrics;
    if (props.endpoint !== undefined) this.endpoint = props.endpoint;
    if (props.columns !== undefined) this.columns = props.columns;
  }
}

declare global {
  interface HTMLElementTagNameMap {
    'kpi-metric-row': KpiMetricRow;
  }
}
```

- [ ] **Step 5: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn install && yarn vitest run components/kpi-metric-row/src/kpi-metric-row.test.ts`
Expected: all 13 tests PASS

- [ ] **Step 6: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/kpi-metric-row/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add kpi-metric-row component — metric cards with sparklines, trends, status  #12"
```

---

### Task 6: approval-gate Component

New component `<approval-gate>` — structured decision point with quorum, evidence slots, confirmation dialog, and SLA integration.

**Files:**
- Create: `components/approval-gate/package.json`
- Create: `components/approval-gate/tsconfig.json`
- Create: `components/approval-gate/tsconfig.build.json`
- Create: `components/approval-gate/vitest.config.ts`
- Create: `components/approval-gate/src/index.ts`
- Create: `components/approval-gate/src/approval-gate.ts`
- Create: `components/approval-gate/src/approval-gate.test.ts`
- Modify: `tsconfig.json` (root) — add reference

**Interfaces:**
- Consumes: `<sla-indicator>` (Task 4), `<blocks-confirm-dialog>` (Task 2), `FocusTrapMixin`, `LiveRegionMixin`, `emitPagesEvent`, `WorkIdentity` from core
- Produces: `<approval-gate>` element, `ApprovalGateTopics` constants, types

- [ ] **Step 1: Scaffold the component package**

```json
// components/approval-gate/package.json
{
  "name": "@casehubio/blocks-ui-approval-gate",
  "version": "0.1.0",
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
    "@casehubio/blocks-ui-sla-indicator": "workspace:*",
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
// components/approval-gate/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/blocks-ui-core" },
    { "path": "../../components/sla-indicator" }
  ]
}
```

```typescript
// components/approval-gate/src/index.ts
export { ApprovalGate, ApprovalGateTopics, type OutcomeDefinition, type QuorumConfig, type VoterStatus, type GateDecision } from './approval-gate.js';
```

Add `{ "path": "components/approval-gate" }` to root `tsconfig.json` references.

- [ ] **Step 2: Write the failing tests**

```typescript
// components/approval-gate/src/approval-gate.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import type { WorkIdentity } from '@casehubio/blocks-ui-core';
import type { OutcomeDefinition, QuorumConfig } from './approval-gate.js';
import './approval-gate.js';

type ApprovalGateEl = HTMLElement & {
  gateId: string;
  endpoint: string;
  identity: WorkIdentity;
  prompt: string;
  contextText: string;
  outcomes: OutcomeDefinition[];
  quorum: QuorumConfig | null;
  deadline: string | null;
  slaWindow: number | null;
  history: Array<{ timestamp: string; actor: string; outcome: string }>;
  data: Record<string, unknown> | null;
  requireConfirmation: boolean;
  updateComplete: Promise<boolean>;
  configure: (props: Record<string, unknown>) => void;
};

const identity: WorkIdentity = { userId: 'user-1', displayName: 'Alice', groups: ['approvers'] };

describe('approval-gate', () => {
  let el: ApprovalGateEl;

  beforeEach(async () => {
    vi.useFakeTimers();
    el = document.createElement('approval-gate') as ApprovalGateEl;
    el.gateId = 'gate-001';
    el.endpoint = '/api/work-items';
    el.identity = identity;
    el.prompt = 'Approve PI authorisation for Trial X?';
    document.body.appendChild(el);
    await el.updateComplete;
  });

  afterEach(() => {
    el.remove();
    vi.useRealTimers();
  });

  it('renders the prompt text', () => {
    expect(el.shadowRoot!.textContent).toContain('Approve PI authorisation');
  });

  it('renders default approve/reject buttons', () => {
    const buttons = el.shadowRoot!.querySelectorAll('.action-btn');
    expect(buttons.length).toBe(2);
    const labels = Array.from(buttons).map(b => b.textContent!.trim());
    expect(labels).toContain('Approve');
    expect(labels).toContain('Reject');
  });

  it('renders custom outcomes', async () => {
    el.outcomes = [
      { key: 'file-sar', label: 'File SAR', variant: 'danger' },
      { key: 'close', label: 'Close', variant: 'neutral' },
    ];
    await el.updateComplete;
    const buttons = el.shadowRoot!.querySelectorAll('.action-btn');
    const labels = Array.from(buttons).map(b => b.textContent!.trim());
    expect(labels).toContain('File SAR');
    expect(labels).toContain('Close');
  });

  it('renders context text', async () => {
    el.contextText = 'This trial has 200 enrolled patients.';
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('200 enrolled patients');
  });

  it('renders inline key-value evidence when data is set', async () => {
    el.data = { risk: 'HIGH', category: 'compliance' };
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('risk');
    expect(el.shadowRoot!.textContent).toContain('HIGH');
  });

  it('renders sla-indicator when deadline is set', async () => {
    el.deadline = new Date(Date.now() + 86400000).toISOString();
    await el.updateComplete;
    expect(el.shadowRoot!.querySelector('sla-indicator')).toBeTruthy();
  });

  it('renders quorum progress bar', async () => {
    el.quorum = {
      required: 3,
      total: 5,
      voters: [
        { id: 'user-2', name: 'Bob', status: 'voted', outcome: 'approve' },
        { id: 'user-3', name: 'Charlie', status: 'pending' },
        { id: 'user-1', name: 'Alice', status: 'pending' },
        { id: 'user-4', name: 'Diana', status: 'voted', outcome: 'approve' },
        { id: 'user-5', name: 'Eve', status: 'pending' },
      ],
    };
    await el.updateComplete;
    const progressbar = el.shadowRoot!.querySelector('[role="progressbar"]');
    expect(progressbar).toBeTruthy();
    expect(progressbar!.getAttribute('aria-valuenow')).toBe('2');
    expect(progressbar!.getAttribute('aria-valuemax')).toBe('3');
  });

  it('shows already-decided state when current user has voted', async () => {
    el.quorum = {
      required: 2,
      total: 3,
      voters: [
        { id: 'user-1', name: 'Alice', status: 'voted', outcome: 'approve' },
        { id: 'user-2', name: 'Bob', status: 'pending' },
        { id: 'user-3', name: 'Charlie', status: 'pending' },
      ],
    };
    await el.updateComplete;
    const buttons = el.shadowRoot!.querySelectorAll<HTMLButtonElement>('.action-btn');
    for (const btn of buttons) {
      expect(btn.disabled).toBe(true);
    }
    expect(el.shadowRoot!.textContent).toContain('You voted');
  });

  it('opens confirmation dialog on outcome button click', async () => {
    el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
    await el.updateComplete;
    const dialog = el.shadowRoot!.querySelector('blocks-confirm-dialog');
    expect(dialog).toBeTruthy();
    expect((dialog as any).open).toBe(true);
  });

  it('skips confirmation when requireConfirmation is false', async () => {
    el.requireConfirmation = false;
    await el.updateComplete;
    const handler = vi.fn();
    globalThis.fetch = vi.fn().mockResolvedValue({ ok: true, json: () => ({}) });
    document.addEventListener('pages-event', handler);
    el.shadowRoot!.querySelector<HTMLButtonElement>('.action-btn')!.click();
    await el.updateComplete;
    await vi.runAllTimersAsync();
    const event = handler.mock.calls.find((c: any) => c[0].detail.topic === 'gate.decided');
    expect(event).toBeTruthy();
    document.removeEventListener('pages-event', handler);
  });

  it('renders history when provided', async () => {
    el.history = [{ timestamp: '2026-07-05T10:00:00Z', actor: 'Bob', outcome: 'approve' }];
    await el.updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Bob');
  });

  it('configure() sets properties', async () => {
    el.configure({ prompt: 'New prompt?', requireConfirmation: false });
    await el.updateComplete;
    expect(el.prompt).toBe('New prompt?');
    expect(el.requireConfirmation).toBe(false);
  });

  it('has aria-describedby on action buttons pointing to prompt', () => {
    const btn = el.shadowRoot!.querySelector('.action-btn');
    const describedBy = btn!.getAttribute('aria-describedby');
    expect(describedBy).toBeTruthy();
    expect(el.shadowRoot!.getElementById(describedBy!)).toBeTruthy();
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn install && yarn vitest run components/approval-gate/src/approval-gate.test.ts`
Expected: FAIL

- [ ] **Step 4: Write implementation**

The approval-gate component is the largest — approximately 350 lines. It composes `<sla-indicator>` and `<blocks-confirm-dialog>`. Key sections:

- Types and topic constants
- Properties with defaults
- Styles using `--blocks-*` tokens
- Layout rendering (header → context → evidence → quorum → history → actions → dialog)
- Evidence slot detection via `slotchange`
- Already-decided state detection via identity matching
- Confirmation dialog flow
- REST submission with error handling
- `configure()` method

The implementer should follow the spec at `docs/specs/2026-07-06-ui-primitives-batch-design.md` §3 precisely. The key patterns are already demonstrated in Tasks 4 and 5. The critical differences:

- Import and register `@casehubio/blocks-ui-sla-indicator` as a side-effect import
- Import `BlocksConfirmDialog` from core (triggers registration)
- Use `FocusTrapMixin` and `LiveRegionMixin` in the mixin chain
- Handle `slotchange` for evidence slot fallback to inline key-value renderer
- Match `identity.userId` against `quorum.voters[].id` for already-decided state
- PUT to `${endpoint}/workitems/${gateId}/complete` with `{ outcome, resolution? }` on confirm

- [ ] **Step 5: Run tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn vitest run components/approval-gate/src/approval-gate.test.ts`
Expected: all 14 tests PASS

- [ ] **Step 6: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/approval-gate/ tsconfig.json
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: add approval-gate component — decision point with quorum, evidence, SLA integration  #13"
```

---

### Task 7: Accessibility & Styling Cleanup

Quick fixes across existing components: container-type, aria-controls, hardcoded spacing, IntersectionObserver.

**Files:**
- Modify: `components/work-item-workbench/src/work-item-workbench.ts` — add `container-type: inline-size`
- Modify: `components/work-item-detail/src/work-item-detail.ts` — add tab aria-controls/labelledby linkage
- Modify: `components/work-item-inbox/src/inbox-filter-bar.ts` — replace hardcoded px with tokens
- Modify: `components/work-item-inbox/src/inbox-summary-bar.ts` — replace hardcoded px with tokens
- Modify: `components/work-item-inbox/src/queue-pill-bar.ts` — add IntersectionObserver gating

**Interfaces:**
- No new interfaces — these are fixes to existing components

- [ ] **Step 1: Fix workbench container-type**

In `work-item-workbench.ts`, add `container-type: inline-size;` to the `:host` CSS rule.

- [ ] **Step 2: Fix detail tabs aria-controls linkage**

In `work-item-detail.ts`:
- Add `id="tabpanel-activity"`, `id="tabpanel-relations"`, `id="tabpanel-notes"` to each tabpanel
- Add `aria-controls="tabpanel-activity"` etc. to each tab
- Add `aria-labelledby="tab-activity"` etc. to each tabpanel
- Add `id="tab-activity"` etc. to each tab

- [ ] **Step 3: Fix hardcoded spacing in filter-bar**

In `inbox-filter-bar.ts`, replace approximately 10 hardcoded values:
- `gap: 12px` → `gap: var(--blocks-space-3, 12px)`
- `gap: 8px` → `gap: var(--blocks-space-2, 8px)`
- `font-size: 12px` → `font-size: var(--blocks-font-size-sm, 12px)`
- `padding: 4px 12px` → `padding: var(--blocks-space-1, 4px) var(--blocks-space-3, 12px)`
- `border-radius: 12px` → `border-radius: var(--blocks-radius-md, 6px)` (or appropriate token)
- `height: 20px` → `height: var(--blocks-space-5, 20px)`

- [ ] **Step 4: Fix hardcoded spacing in summary-bar**

In `inbox-summary-bar.ts`, replace 3 hardcoded values:
- `gap: 6px` → `gap: var(--blocks-space-1.5, 6px)`
- `padding: 4px 12px` → `padding: var(--blocks-space-1, 4px) var(--blocks-space-3, 12px)`
- `font-size: 13px` → `font-size: var(--blocks-font-size-sm, 12px)` (snap to nearest token)

- [ ] **Step 5: Add IntersectionObserver to queue-pill-bar**

In `queue-pill-bar.ts`, replace the eager `_loadQueues()` call in `connectedCallback` with:

```typescript
private _observer: IntersectionObserver | null = null;

override connectedCallback(): void {
  super.connectedCallback();
  this._observer = new IntersectionObserver(
    (entries) => {
      if (entries[0]?.isIntersecting) {
        this._observer!.disconnect();
        this._observer = null;
        this._loadQueues();
        this._startPolling();
      }
    },
    { threshold: 0 }
  );
  this._observer.observe(this);
}

override disconnectedCallback(): void {
  super.disconnectedCallback();
  this._observer?.disconnect();
  this._observer = null;
}
```

- [ ] **Step 6: Run existing tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn test`
Expected: all existing tests still PASS

- [ ] **Step 7: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/work-item-workbench/ components/work-item-detail/ components/work-item-inbox/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "fix: accessibility and styling cleanup — container-type, aria-controls, spacing tokens, IntersectionObserver  #16"
```

---

### Task 8: LiveRegionMixin Integration + Batch Cancel Dialog

Wire `LiveRegionMixin` into existing components and replace `window.confirm()` with `<blocks-confirm-dialog>`.

**Files:**
- Modify: `components/work-item-inbox/src/work-item-inbox.ts` — add LiveRegionMixin, replace `confirm()` with dialog, add error banner for claim failures
- Modify: `components/work-item-detail/src/work-item-detail.ts` — add LiveRegionMixin with announcements

**Interfaces:**
- Consumes: `LiveRegionMixin` from core, `<blocks-confirm-dialog>` (Task 2)

- [ ] **Step 1: Add LiveRegionMixin to work-item-inbox**

In `work-item-inbox.ts`:
- Add `LiveRegionMixin` to the class chain: `extends LiveRegionMixin(KeyboardShortcutMixin(RovingTabindexMixin(LitElement)))`
- Add `this.announce()` calls at key action points:
  - After successful claim: `this.announce('Item claimed successfully')`
  - After claim failure: `this.announce('Failed to claim item', 'assertive')`
  - After batch complete: `this.announce('${count} items completed')`
  - After batch cancel: `this.announce('${count} items cancelled')`
  - SSE connected: `this.announce('Live updates connected')`
  - SSE error: `this.announce('Live updates disconnected', 'assertive')`

- [ ] **Step 2: Replace batch cancel confirm() with dialog**

In `work-item-inbox.ts`:
- Add `@state() private _showCancelDialog = false;`
- Add `@state() private _pendingCancelItems: string[] = [];`
- In the `_handleBatchCancel` method, replace `confirm(...)` with `this._showCancelDialog = true; this._pendingCancelItems = [...selectedIds];`
- Add `<blocks-confirm-dialog>` to the render tree:
  ```html
  <blocks-confirm-dialog
    .open=${this._showCancelDialog}
    heading="Cancel items?"
    message="This will cancel ${this._pendingCancelItems.length} selected item(s)."
    confirmLabel="Cancel items"
    cancelLabel="Keep"
    confirmVariant="danger"
    .showReason=${true}
    @confirm=${this._confirmBatchCancel}
    @cancel=${() => { this._showCancelDialog = false; }}
  ></blocks-confirm-dialog>
  ```
- Import `'@casehubio/blocks-ui-core'` confirm-dialog (side-effect import for registration)

- [ ] **Step 3: Add claim error banner**

In `work-item-inbox.ts`:
- Add `@state() private _claimError: string | null = null;`
- In the `claimItem()` catch block, replace console.log with:
  ```typescript
  this._claimError = 'Failed to claim item';
  this.announce('Failed to claim item', 'assertive');
  setTimeout(() => { this._claimError = null; }, 5000);
  ```
- Add error banner to render tree (above the item list):
  ```html
  ${this._claimError ? html`<div class="error-banner" role="alert">${this._claimError}</div>` : nothing}
  ```
- Add `.error-banner` CSS

- [ ] **Step 4: Add LiveRegionMixin to work-item-detail**

In `work-item-detail.ts`:
- Change `extends FocusTrapMixin(LitElement)` to `extends LiveRegionMixin(FocusTrapMixin(LitElement))`
- Add announcements:
  - After claim: `this.announce('Item claimed')`
  - After complete: `this.announce('Item completed')`
  - After escalate: `this.announce('Item escalated')`
  - After delegate: `this.announce('Item delegated')`
  - After add note: `this.announce('Note added')`
  - On error: `this.announce('Action failed: ' + error, 'assertive')`

- [ ] **Step 5: Run all tests**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn test`
Expected: all tests PASS

- [ ] **Step 6: Typecheck**

Run: `cd /Users/mdproctor/claude/casehub/blocks-ui && yarn typecheck`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/work-item-inbox/ components/work-item-detail/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat: LiveRegionMixin integration, batch cancel styled dialog, claim error handling  #16"
```

---

## File Structure Summary

### New files (32)
```
packages/blocks-ui-core/src/timers/shared-timer-controller.ts
packages/blocks-ui-core/src/timers/shared-timer-controller.test.ts
packages/blocks-ui-core/src/timers/index.ts
packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.ts
packages/blocks-ui-core/src/confirm-dialog/blocks-confirm-dialog.test.ts
packages/blocks-ui-core/src/confirm-dialog/index.ts
packages/blocks-ui-core/src/schema-form/types.ts
components/sla-indicator/package.json
components/sla-indicator/tsconfig.json
components/sla-indicator/tsconfig.build.json
components/sla-indicator/vitest.config.ts
components/sla-indicator/src/index.ts
components/sla-indicator/src/sla-indicator.ts
components/sla-indicator/src/sla-indicator.test.ts
components/kpi-metric-row/package.json
components/kpi-metric-row/tsconfig.json
components/kpi-metric-row/tsconfig.build.json
components/kpi-metric-row/vitest.config.ts
components/kpi-metric-row/src/index.ts
components/kpi-metric-row/src/kpi-metric-row.ts
components/kpi-metric-row/src/kpi-metric-row.test.ts
components/approval-gate/package.json
components/approval-gate/tsconfig.json
components/approval-gate/tsconfig.build.json
components/approval-gate/vitest.config.ts
components/approval-gate/src/index.ts
components/approval-gate/src/approval-gate.ts
components/approval-gate/src/approval-gate.test.ts
```

### Modified files (12)
```
packages/blocks-ui-core/src/index.ts
packages/blocks-ui-core/src/schema-form/field-renderers.ts
packages/blocks-ui-core/src/schema-form/schema-form.ts
packages/blocks-ui-core/src/schema-form/field-registry.ts
packages/blocks-ui-core/src/schema-form/index.ts
packages/blocks-ui-core/src/schema-form/schema-form.test.ts
components/work-item-workbench/src/work-item-workbench.ts
components/work-item-detail/src/work-item-detail.ts
components/work-item-inbox/src/inbox-filter-bar.ts
components/work-item-inbox/src/inbox-summary-bar.ts
components/work-item-inbox/src/queue-pill-bar.ts
components/work-item-inbox/src/work-item-inbox.ts
tsconfig.json
```

## Dependency Graph

```
Task 1 (SharedTimerController) ──→ Task 4 (sla-indicator) ──→ Task 6 (approval-gate)
Task 2 (confirm-dialog) ──────────────────────────────────────→ Task 6 (approval-gate)
Task 2 (confirm-dialog) ──────────────────────────────────────→ Task 8 (batch cancel)
Task 3 (schema-form) ── independent
Task 5 (kpi-metric-row) ── independent
Task 7 (a11y/styling) ── independent
```

Independent tasks (3, 5, 7) can run in parallel. Tasks 1→4→6 is the critical path.
