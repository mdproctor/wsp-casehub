# Subscription Editor & Notification Preferences Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #33 — feat: subscription editor component
**Issue group:** #33, #34

**Goal:** Schema-driven subscription editor, notification preferences (channel/mute/snooze), and GDPR form migration — all using `pages-schema-form`.

**Architecture:** Extend `pages-schema-form` with four capabilities (oneOf labeled enums, time format, readOnly, recursive validation), then build five new components and migrate one existing form. All forms delegate rendering to `<pages-schema-form>` and own only schema construction, change orchestration, and API interaction.

**Tech Stack:** Lit 3, TypeScript, `@casehubio/pages-form`, `@casehubio/pages-data`, Vitest

## Global Constraints

- All components use `--pages-*` CSS custom properties — no hardcoded colours
- Component customisation via typed config properties + render callbacks, never slots for content (PP-20260713-8ea1af)
- Events use `pages-event` CustomEvent with `{ bubbles: true, composed: true }`
- Pre-release: breaking changes to types and APIs are acceptable
- Cross-repo: pages-form changes committed to casehub-pages main; blocks-ui changes on `issue-33-subscription-editor` branch

---

### Task 1: pages-form Schema Improvements

**Repo:** `casehub-pages` (`/Users/mdproctor/claude/casehub/pages`)

**Files:**
- Modify: `packages/pages-form/src/types.ts`
- Modify: `packages/pages-form/src/field-renderers.ts`
- Modify: `packages/pages-form/src/validation.ts`
- Modify: `packages/pages-form/src/pages-schema-form.test.ts`

**Interfaces:**
- Produces: `FieldSchema.oneOf`, `FieldSchema.readOnly` — consumed by all subsequent tasks
- Produces: `format: 'time'` handling — consumed by Task 4 (channel-preferences)
- Produces: recursive `validateField` — consumed by Task 3 (subscription-editor template validation)

- [ ] **Step 1: Write failing tests for oneOf**

Add to `pages-schema-form.test.ts`:

```typescript
describe('oneOf labeled enums', () => {
  const oneOfSchema = {
    type: 'object',
    properties: {
      severity: {
        type: 'string',
        title: 'Severity',
        oneOf: [
          { const: 'INFO', title: 'Information' },
          { const: 'WARNING', title: 'Warning' },
          { const: 'URGENT', title: 'Urgent' },
        ],
      },
    },
    required: ['severity'],
  };

  it('renders select with title labels and const values in edit mode', async () => {
    el.schema = oneOfSchema;
    el.data = { severity: 'WARNING' };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const select = el.shadowRoot!.querySelector<HTMLSelectElement>('select[id="severity"]');
    expect(select).toBeTruthy();
    expect(select!.value).toBe('WARNING');
    const options = Array.from(select!.querySelectorAll('option:not([disabled])'));
    expect(options.map(o => o.textContent?.trim())).toEqual(['Information', 'Warning', 'Urgent']);
    expect(options.map(o => (o as HTMLOptionElement).value)).toEqual(['INFO', 'WARNING', 'URGENT']);
  });

  it('renders disabled placeholder when value is empty', async () => {
    el.schema = oneOfSchema;
    el.data = { severity: '' };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const select = el.shadowRoot!.querySelector<HTMLSelectElement>('select[id="severity"]');
    const placeholder = select!.querySelector('option[disabled]');
    expect(placeholder).toBeTruthy();
    expect(placeholder!.textContent?.trim()).toContain('Select');
    expect(placeholder!.selected).toBe(true);
  });

  it('displays title instead of const in display mode', async () => {
    el.schema = oneOfSchema;
    el.data = { severity: 'URGENT' };
    el.mode = 'display';
    await (el as any).updateComplete;
    expect(el.shadowRoot!.textContent).toContain('Urgent');
    expect(el.shadowRoot!.textContent).not.toContain('URGENT');
  });

  it('takes priority over enum when both present', async () => {
    el.schema = {
      type: 'object',
      properties: {
        status: {
          type: 'string',
          enum: ['A', 'B'],
          oneOf: [
            { const: 'X', title: 'Option X' },
            { const: 'Y', title: 'Option Y' },
          ],
        },
      },
    };
    el.data = { status: 'X' };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const options = el.shadowRoot!.querySelectorAll('select option:not([disabled])');
    expect(options.length).toBe(2);
    expect(options[0]!.textContent?.trim()).toBe('Option X');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages/packages/pages-form test`
Expected: FAIL — oneOf not handled, renders as text input

- [ ] **Step 3: Implement oneOf in types.ts**

Add `oneOf` and `readOnly` to `FieldSchema` in `types.ts`:

```typescript
export interface FieldSchema {
  readonly type?: string;
  readonly format?: string;
  readonly title?: string;
  readonly description?: string;
  readonly placeholder?: string;
  readonly enum?: readonly string[];
  readonly oneOf?: readonly { readonly const: string; readonly title: string }[];
  readonly readOnly?: boolean;
  readonly pattern?: string;
  readonly minimum?: number;
  readonly maximum?: number;
  readonly minLength?: number;
  readonly maxLength?: number;
  readonly properties?: Readonly<Record<string, FieldSchema>>;
  readonly items?: FieldSchema;
  readonly required?: readonly string[];
}
```

- [ ] **Step 4: Implement oneOf rendering in field-renderers.ts**

In `renderEditField`, add oneOf handling BEFORE the existing `schema.enum` check:

```typescript
if (schema.oneOf) {
  const currentMatch = schema.oneOf.some(o => o.const === value);
  return html`
    <div class="field">
      <label for="${key}">${schema.title ?? key}</label>
      <select id="${key}" @change=${(e: Event) => onChange(key, (e.target as HTMLSelectElement).value)} @blur=${() => onBlur?.(key)}>
        ${!currentMatch ? html`<option value="" disabled selected>Select ${schema.title ?? key}...</option>` : ''}
        ${schema.oneOf.map(opt => html`<option value=${opt.const} ?selected=${value === opt.const}>${opt.title}</option>`)}
      </select>
      ${schema.description ? html`<span class="description">${schema.description}</span>` : ''}
      ${error ? html`<span class="error">${error}</span>` : ''}
    </div>`;
}
```

In `renderDisplayField`, add oneOf handling at the top (before null check):

```typescript
if (schema.oneOf) {
  const match = schema.oneOf.find(o => o.const === value);
  const displayValue = match ? match.title : (value ?? '—');
  return html`<div class="field"><span class="label">${schema.title ?? key}</span><span class="value${value == null ? ' muted' : ''}">${displayValue}</span>${schema.description ? html`<span class="description">${schema.description}</span>` : ''}</div>`;
}
```

- [ ] **Step 5: Run oneOf tests to verify they pass**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages/packages/pages-form test`
Expected: oneOf tests PASS, existing tests PASS

- [ ] **Step 6: Write failing tests for format: 'time' and readOnly**

Add to `pages-schema-form.test.ts`:

```typescript
describe('format: time', () => {
  const timeSchema = {
    type: 'object',
    properties: {
      start: { type: 'string', format: 'time', title: 'Start Time' },
    },
  };

  it('renders time input in edit mode', async () => {
    el.schema = timeSchema;
    el.data = { start: '09:30' };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const input = el.shadowRoot!.querySelector<HTMLInputElement>('input[type="time"]');
    expect(input).toBeTruthy();
    expect(input!.value).toBe('09:30');
  });

  it('displays time value in display mode', async () => {
    el.schema = timeSchema;
    el.data = { start: '14:00' };
    el.mode = 'display';
    await (el as any).updateComplete;
    expect(el.shadowRoot!.textContent).toContain('14:00');
  });
});

describe('readOnly', () => {
  const readOnlySchema = {
    type: 'object',
    properties: {
      id: { type: 'string', title: 'ID', readOnly: true },
      name: { type: 'string', title: 'Name' },
    },
    required: ['id', 'name'],
  };

  it('renders readonly attribute in edit mode', async () => {
    el.schema = readOnlySchema;
    el.data = { id: 'abc-123', name: 'Test' };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const idInput = el.shadowRoot!.querySelector<HTMLInputElement>('input[id="id"]');
    expect(idInput!.readOnly).toBe(true);
  });

  it('skips readOnly fields during validation', async () => {
    el.schema = readOnlySchema;
    el.data = { id: '', name: 'Test' };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const result = (el as any).submit();
    expect(result).toBeTruthy();
  });
});
```

- [ ] **Step 7: Implement format: 'time' in field-renderers.ts**

In `renderEditField`, add after the `format === 'date-time'` case:

```typescript
if (schema.format === 'time') {
  return html`
    <div class="field">
      <label for="${key}">${schema.title ?? key}</label>
      <input id="${key}" type="time" .value=${String(value ?? '')} ${schema.readOnly ? html`readonly` : ''} @input=${(e: Event) => onChange(key, (e.target as HTMLInputElement).value)} @blur=${() => onBlur?.(key)} />
      ${schema.description ? html`<span class="description">${schema.description}</span>` : ''}
      ${error ? html`<span class="error">${error}</span>` : ''}
    </div>`;
}
```

In `renderDisplayField`, add after the `format === 'date-time'` case:

```typescript
if (schema.format === 'time') {
  return html`<div class="field"><span class="label">${schema.title ?? key}</span><span class="value">${value ?? '—'}</span>${schema.description ? html`<span class="description">${schema.description}</span>` : ''}</div>`;
}
```

- [ ] **Step 8: Implement readOnly in field-renderers.ts**

For string inputs (the default text input case at the bottom of `renderEditField`), add `readonly` when `schema.readOnly` is true:

```typescript
return html`
  <div class="field">
    <label for="${key}">${schema.title ?? key}</label>
    <input id="${key}" type="text" placeholder=${schema.placeholder ?? ''} .value=${String(value ?? '')} ?readonly=${schema.readOnly} @input=${(e: Event) => onChange(key, (e.target as HTMLInputElement).value)} @blur=${() => onBlur?.(key)} />
    ${schema.description ? html`<span class="description">${schema.description}</span>` : ''}
    ${error ? html`<span class="error">${error}</span>` : ''}
  </div>`;
```

Apply the same `?readonly=${schema.readOnly}` to number, date, date-time, time, and textarea inputs.

- [ ] **Step 9: Implement readOnly in validation.ts**

In `validateField`, add at the top (after the function signature, before required check):

```typescript
if (schema.readOnly) return null;
```

- [ ] **Step 10: Run time and readOnly tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages/packages/pages-form test`
Expected: all new tests PASS, existing tests PASS

- [ ] **Step 11: Write failing tests for recursive validation**

Add to `pages-schema-form.test.ts`:

```typescript
describe('recursive validation', () => {
  it('validates required fields in nested objects', async () => {
    el.schema = {
      type: 'object',
      properties: {
        template: {
          type: 'object',
          properties: {
            title: { type: 'string' },
            category: { type: 'string' },
          },
          required: ['title', 'category'],
        },
      },
    };
    el.data = { template: { title: '', category: '' } };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const result = (el as any).submit();
    expect(result).toBeNull();
    await (el as any).updateComplete;
    const errors = el.shadowRoot!.querySelectorAll('.error');
    expect(errors.length).toBeGreaterThanOrEqual(2);
  });

  it('validates required fields in array items', async () => {
    el.schema = {
      type: 'object',
      properties: {
        items: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              name: { type: 'string' },
              value: { type: 'string' },
            },
            required: ['name', 'value'],
          },
        },
      },
    };
    el.data = { items: [{ name: '', value: '' }] };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const result = (el as any).submit();
    expect(result).toBeNull();
  });

  it('passes when nested required fields are present', async () => {
    el.schema = {
      type: 'object',
      properties: {
        template: {
          type: 'object',
          properties: {
            title: { type: 'string' },
          },
          required: ['title'],
        },
      },
    };
    el.data = { template: { title: 'Hello' } };
    el.mode = 'edit';
    await (el as any).updateComplete;
    const result = (el as any).submit();
    expect(result).toBeTruthy();
  });
});
```

- [ ] **Step 12: Implement recursive validation in validation.ts**

Extend `validateField` to handle nested objects and arrays:

```typescript
export function validateField(
  _key: string,
  schema: FieldSchema,
  value: unknown,
  required: boolean,
): string | null {
  if (schema.readOnly) return null;

  if (required && (value === null || value === undefined || value === '')) {
    return 'Required';
  }

  if (value === null || value === undefined || value === '') return null;

  // oneOf validation
  if (schema.oneOf && typeof value === 'string') {
    if (!schema.oneOf.some(o => o.const === value)) {
      return 'Invalid selection';
    }
  }

  if (typeof value === 'string') {
    if (schema.pattern != null) {
      const re = new RegExp(schema.pattern);
      if (!re.test(value)) return 'Invalid format';
    }
    if (schema.minLength != null && value.length < schema.minLength) {
      return `Must be at least ${schema.minLength} characters`;
    }
    if (schema.maxLength != null && value.length > schema.maxLength) {
      return `Must be at most ${schema.maxLength} characters`;
    }
  }

  if (typeof value === 'number') {
    if (schema.minimum != null && value < schema.minimum) {
      return `Must be at least ${schema.minimum}`;
    }
    if (schema.maximum != null && value > schema.maximum) {
      return `Must be at most ${schema.maximum}`;
    }
  }

  // Recursive: nested objects
  if (schema.type === 'object' && schema.properties && typeof value === 'object' && value !== null) {
    const obj = value as Record<string, unknown>;
    const requiredSet = new Set(schema.required ?? []);
    for (const [k, subSchema] of Object.entries(schema.properties)) {
      const error = validateField(k, subSchema, obj[k], requiredSet.has(k));
      if (error) return `${k}: ${error}`;
    }
  }

  // Recursive: arrays
  if (schema.type === 'array' && schema.items && Array.isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      const item = value[i];
      if (schema.items.type === 'object' && schema.items.properties && typeof item === 'object' && item !== null) {
        const obj = item as Record<string, unknown>;
        const requiredSet = new Set(schema.items.required ?? []);
        for (const [k, subSchema] of Object.entries(schema.items.properties)) {
          const error = validateField(k, subSchema, obj[k], requiredSet.has(k));
          if (error) return `Item ${i + 1}: ${k}: ${error}`;
        }
      }
    }
  }

  return null;
}
```

- [ ] **Step 13: Run all tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages/packages/pages-form test`
Expected: ALL tests PASS

- [ ] **Step 14: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-form/src/types.ts packages/pages-form/src/field-renderers.ts packages/pages-form/src/validation.ts packages/pages-form/src/pages-schema-form.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat(pages-form): oneOf labeled enums, time format, readOnly, recursive validation

Extends FieldSchema for blocks-ui subscription editor and notification
preferences (#33, #34). Adds oneOf with disabled placeholder, format
time, readOnly skip, and recursive nested/array validation."
```

---

### Task 2: blocks-ui Type Updates, Event Topics, Dependencies

**Repo:** `casehub-blocks-ui` (`/Users/mdproctor/claude/casehub/worktrees/12/blocks-ui`)

**Files:**
- Modify: `components/notification-inbox/src/types.ts`
- Modify: `components/notification-inbox/src/events.ts`
- Modify: `components/notification-inbox/package.json`
- Modify: `components/gdpr-erasure-action/package.json`

**Interfaces:**
- Produces: `DigestScheduleWeeklyAt`, extended `DigestSchedule`, `DigestGroupBy`, `QuietHoursAction`, `ENTITY_WATCHERS` in `TargetType` — consumed by Tasks 3–7
- Produces: 5 new event topics in `NotificationEventTopics` — consumed by Tasks 3–7
- Produces: `@casehubio/pages-form` dependency — consumed by Tasks 3–8

- [ ] **Step 1: Update types.ts — add missing backend types**

Add `DigestScheduleWeeklyAt` to the digest schedule union:

```typescript
export type DigestSchedule = DigestScheduleInterval | DigestScheduleDailyAt | DigestScheduleWeeklyAt;
```

Add the new interface after `DigestScheduleDailyAt`:

```typescript
export interface DigestScheduleWeeklyAt {
  readonly type: 'weekly_at';
  readonly day: string; // DayOfWeek: MONDAY..SUNDAY
  readonly time: string; // HH:mm
  readonly timezone: string;
}
```

Add `ENTITY_WATCHERS` to `TargetType`:

```typescript
export type TargetType = 'USER' | 'GROUP' | 'EVENT_FIELD' | 'ENTITY_WATCHERS';
```

Add new types after `DeliveryChannelDescriptor`:

```typescript
export type DigestGroupBy = 'FLAT' | 'CATEGORY' | 'ENTITY';

export type QuietHoursAction = 'SUPPRESS' | 'BUFFER_FOR_DIGEST';
```

Update `QuietHours` to include action:

```typescript
export interface QuietHours {
  readonly start: string;
  readonly end: string;
  readonly timezone: string;
  readonly action?: QuietHoursAction;
}
```

Update `ChannelPreference` to include groupBy:

```typescript
export interface ChannelPreference {
  readonly enabled: boolean;
  readonly minSeverity: NotificationSeverity;
  readonly digestSchedule: DigestSchedule | null;
  readonly groupBy?: DigestGroupBy;
}
```

- [ ] **Step 2: Update events.ts — add new topics**

Add to `NotificationEventTopics`:

```typescript
export const NotificationEventTopics = {
  SELECTED: 'notification.selected',
  DISMISSED: 'notification.dismissed',
  MUTED: 'notification.muted',
  SUBSCRIPTION_CREATED: 'subscription.created',
  SUBSCRIPTION_DELETED: 'subscription.deleted',
  PREFERENCE_UPDATED: 'preference.updated',
  MUTE_CREATED: 'mute.created',
  MUTE_DELETED: 'mute.deleted',
  SNOOZE_ACTIVATED: 'snooze.activated',
  SNOOZE_CANCELLED: 'snooze.cancelled',
} as const;
```

- [ ] **Step 3: Add pages-form dependency to notification-inbox**

In `components/notification-inbox/package.json`, add to `dependencies`:

```json
"@casehubio/pages-form": "^0.1.0"
```

- [ ] **Step 4: Add pages-form dependency to gdpr-erasure-action**

In `components/gdpr-erasure-action/package.json`, add to `dependencies`:

```json
"@casehubio/pages-form": "^0.1.0"
```

- [ ] **Step 5: Run yarn install and typecheck**

```bash
yarn --cwd /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui install
yarn --cwd /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui typecheck
```

Expected: PASS — type additions are backward compatible

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui add components/notification-inbox/src/types.ts components/notification-inbox/src/events.ts components/notification-inbox/package.json components/gdpr-erasure-action/package.json yarn.lock
git -C /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui commit -m "feat(#33,#34): add missing backend types, event topics, pages-form dependency

Aligns frontend types with backend API: DigestScheduleWeeklyAt,
ENTITY_WATCHERS target, DigestGroupBy, QuietHoursAction. Adds five
event topics for preferences/mute/snooze. Wires pages-form dependency."
```

---

### Task 3: Subscription Editor

**Repo:** `casehub-blocks-ui`

**Files:**
- Create: `components/notification-inbox/src/subscription-editor.ts`
- Create: `components/notification-inbox/src/subscription-editor.test.ts`
- Modify: `components/notification-inbox/src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi.getEventTypes()`, `.createSubscription()`, `.updateSubscription()` from `api.ts`
- Consumes: `Subscription`, `SubscriptionInput`, `SubscriptionUpdate`, `EventTypeDescriptor`, `Constraint`, `NotificationTarget`, `NotificationTemplate` from `types.ts`
- Consumes: `FieldSchema` with `oneOf` from `@casehubio/pages-form`
- Consumes: `emitNotificationEvent`, `NotificationEventTopics.SUBSCRIPTION_CREATED` from `events.ts`
- Produces: `<subscription-editor>` custom element — consumed by `subscription-list.ts` (already referencing it)

- [ ] **Step 1: Write failing tests**

Create `subscription-editor.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import { NotificationApi } from './api.js';
import './subscription-editor.js';
import type { SubscriptionEditor } from './subscription-editor.js';
import type { Subscription, EventTypeDescriptor, SubscriptionInput } from './types.js';

async function createElement(): Promise<SubscriptionEditor> {
  const el = document.createElement('subscription-editor') as SubscriptionEditor;
  document.body.appendChild(el);
  await el.updateComplete;
  return el;
}

function waitUntil(condition: () => boolean, message = '', timeout = 1000): Promise<void> {
  return new Promise((resolve, reject) => {
    const start = Date.now();
    const check = () => {
      if (condition()) resolve();
      else if (Date.now() - start > timeout) reject(new Error(message || 'Timeout'));
      else setTimeout(check, 10);
    };
    check();
  });
}

const mockEventTypes: EventTypeDescriptor[] = [
  {
    eventType: 'issue.updated',
    displayName: 'Issue Updated',
    description: 'Fires when an issue is updated',
    fields: [
      { name: 'issueId', type: 'string', description: 'Issue identifier' },
      { name: 'assigneeId', type: 'string', description: 'Assigned user' },
      { name: 'status', type: 'string', description: 'Issue status' },
    ],
  },
  {
    eventType: 'pr.created',
    displayName: 'Pull Request Created',
    description: 'Fires when a PR is created',
    fields: [
      { name: 'prId', type: 'string', description: 'PR identifier' },
      { name: 'author', type: 'string', description: 'PR author' },
    ],
  },
];

const mockSubscription: Subscription = {
  id: 'sub-1',
  ownerId: 'user-1',
  tenancyId: 'tenant-1',
  name: 'My Sub',
  eventType: 'issue.updated',
  constraints: [{ field: 'assigneeId', op: 'EQ', value: '$me' }],
  targets: [{ type: 'USER', id: 'user-1' }],
  includeActor: false,
  template: {
    titlePattern: 'Issue ${issueId} updated',
    bodyPattern: null,
    severity: 'INFO',
    category: 'issues',
    actionUrlPattern: '/issues/${issueId}',
    entityType: 'issue',
    entityIdField: 'issueId',
    actorIdField: 'assigneeId',
  },
  enabled: true,
  createdAt: '2026-07-01T10:00:00Z',
  updatedAt: '2026-07-01T10:00:00Z',
};

function createMockApi(overrides: Partial<Record<keyof NotificationApi, unknown>> = {}) {
  return {
    getEventTypes: async () => mockEventTypes,
    createSubscription: async (input: SubscriptionInput) => ({
      id: 'sub-new',
      ...input,
      createdAt: '2026-07-21T10:00:00Z',
      updatedAt: '2026-07-21T10:00:00Z',
    }),
    updateSubscription: async (id: string, update: unknown) => ({
      ...mockSubscription,
      ...update,
      id,
    }),
    ...overrides,
  } as unknown as NotificationApi;
}

describe('subscription-editor', () => {
  afterEach(() => {
    document.querySelectorAll('subscription-editor').forEach(el => el.remove());
  });

  it('fetches event types on connect and renders form', async () => {
    const el = await createElement();
    el.api = createMockApi();
    el.endpoint = 'http://localhost/api';
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => !el.loading, 'event types loaded');
    const form = el.shadowRoot!.querySelector('pages-schema-form');
    expect(form).toBeTruthy();
  });

  it('populates event type dropdown from API', async () => {
    const el = await createElement();
    el.api = createMockApi();
    el.endpoint = 'http://localhost/api';
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => !el.loading);
    const form = el.shadowRoot!.querySelector('pages-schema-form') as any;
    expect(form.schema.properties.eventType.oneOf).toHaveLength(2);
    expect(form.schema.properties.eventType.oneOf[0].const).toBe('issue.updated');
    expect(form.schema.properties.eventType.oneOf[0].title).toBe('Issue Updated');
  });

  it('rebuilds constraint field options when event type changes', async () => {
    const el = await createElement();
    el.api = createMockApi();
    el.endpoint = 'http://localhost/api';
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => !el.loading);
    el.handleFormChange({ detail: { key: 'eventType', value: 'issue.updated', data: { eventType: 'issue.updated' } } } as CustomEvent);
    await el.updateComplete;
    const form = el.shadowRoot!.querySelector('pages-schema-form') as any;
    const constraintFieldSchema = form.schema.properties.constraints.items.properties.field;
    expect(constraintFieldSchema.oneOf).toHaveLength(3);
    expect(constraintFieldSchema.oneOf[0].const).toBe('issueId');
  });

  it('pre-populates form in edit mode', async () => {
    const el = await createElement();
    el.api = createMockApi();
    el.endpoint = 'http://localhost/api';
    el.subscription = mockSubscription;
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => !el.loading);
    const form = el.shadowRoot!.querySelector('pages-schema-form') as any;
    expect(form.data.name).toBe('My Sub');
    expect(form.data.eventType).toBe('issue.updated');
  });

  it('calls createSubscription on save in create mode', async () => {
    let createCalled = false;
    const el = await createElement();
    el.api = createMockApi({
      createSubscription: async (input: SubscriptionInput) => {
        createCalled = true;
        expect(input.name).toBe('Test');
        expect(input.eventType).toBe('issue.updated');
        return { id: 'new', ...input, createdAt: '', updatedAt: '' };
      },
    });
    el.endpoint = 'http://localhost/api';
    el.identity = { userId: 'user-1', tenancyId: 'tenant-1' };
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => !el.loading);
    el.handleFormChange({ detail: { key: 'name', value: 'Test', data: { name: 'Test', eventType: 'issue.updated', constraints: [], targets: [{ type: 'USER', id: 'user-1' }], includeActor: false, template: { titlePattern: 'T', bodyPattern: null, severity: 'INFO', category: 'test', actionUrlPattern: null, entityType: 'issue', entityIdField: 'issueId', actorIdField: 'assigneeId' } } } } as CustomEvent);
    await el.save();
    expect(createCalled).toBe(true);
  });

  it('emits save event after successful create', async () => {
    const el = await createElement();
    el.api = createMockApi();
    el.endpoint = 'http://localhost/api';
    el.identity = { userId: 'user-1', tenancyId: 'tenant-1' };
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => !el.loading);
    let saveFired = false;
    el.addEventListener('save', () => { saveFired = true; });
    el.formData = { name: 'Test', eventType: 'issue.updated', constraints: [], targets: [{ type: 'USER', id: 'user-1' }], includeActor: false, template: { titlePattern: 'T', bodyPattern: null, severity: 'INFO', category: 'c', actionUrlPattern: null, entityType: 'e', entityIdField: 'f', actorIdField: 'a' } };
    await el.save();
    expect(saveFired).toBe(true);
  });

  it('emits cancel event on cancel', async () => {
    const el = await createElement();
    el.api = createMockApi();
    el.endpoint = 'http://localhost/api';
    await el.connectedCallback();
    await el.updateComplete;
    let cancelFired = false;
    el.addEventListener('cancel', () => { cancelFired = true; });
    el.cancel();
    expect(cancelFired).toBe(true);
  });

  it('shows error on API failure', async () => {
    const el = await createElement();
    el.api = createMockApi({
      getEventTypes: async () => { throw new Error('Network error'); },
    });
    el.endpoint = 'http://localhost/api';
    await el.connectedCallback();
    await el.updateComplete;
    await waitUntil(() => el.error != null, 'error set');
    expect(el.shadowRoot!.textContent).toContain('Network error');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui/components/notification-inbox test`
Expected: FAIL — subscription-editor module not found

- [ ] **Step 3: Implement subscription-editor.ts**

Create `components/notification-inbox/src/subscription-editor.ts` with the full component. Key structure:

- `@customElement('subscription-editor')` extending `LitElement`
- Properties: `endpoint`, `identity`, `subscription`, `api`
- State: `loading`, `error`, `eventTypes`, `formData`, `schema`
- `connectedCallback`: fetch event types, build initial schema
- `buildSchema(eventType?)`: constructs `FieldSchema` from event types + domain types
- `handleFormChange(e)`: listens for event type changes, rebuilds schema
- `save()`: validates via form `submit()`, maps to API input, calls create/update
- `cancel()`: dispatches cancel event
- `render()`: loading/error/form states

The schema construction builds the structure described in the spec's §2 Schema Structure, using `oneOf` for event type, constraint operators, target types, and severity.

- [ ] **Step 4: Add export to index.ts**

Add to `components/notification-inbox/src/index.ts`:

```typescript
export { SubscriptionEditor } from './subscription-editor.js';
```

- [ ] **Step 5: Run tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui/components/notification-inbox test`
Expected: ALL tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui add components/notification-inbox/src/subscription-editor.ts components/notification-inbox/src/subscription-editor.test.ts components/notification-inbox/src/index.ts
git -C /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui commit -m "feat(#33): subscription editor — schema-driven form with dynamic field rebuild

Event type picker from API, constraint builder with per-event-type
field options, target selector, template config with guided field
selects. Uses pages-schema-form for rendering."
```

---

### Task 4: Channel Preferences

**Repo:** `casehub-blocks-ui`

**Files:**
- Create: `components/notification-inbox/src/channel-preferences.ts`
- Create: `components/notification-inbox/src/channel-preferences.test.ts`
- Modify: `components/notification-inbox/src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi.getChannels()`, `.getPreferences()`, `.updatePreferences()` from `api.ts`
- Consumes: `DeliveryChannelDescriptor`, `NotificationPreferences`, `NotificationPreferenceUpdate`, `ChannelPreference`, `DigestSchedule`, `QuietHours` from `types.ts`
- Consumes: `FieldSchema` with `oneOf`, `format: 'time'` from `@casehubio/pages-form`
- Produces: `<channel-preferences>` custom element

- [ ] **Step 1: Write failing tests**

Create `channel-preferences.test.ts` with tests for:
- Fetches channels + preferences on connect, builds per-channel schema
- `deliveryMode` defaults from `descriptor.defaultDigestSchedule` (null → IMMEDIATE, non-null → DIGEST)
- `deliveryMode` IMMEDIATE hides digestSchedule/groupBy; DIGEST shows them
- IMMEDIATE maps to `digestSchedule: null` in API update
- `weekly_at` schedule shows `dayOfWeek + time + timezone`
- Quiet hours renders time inputs + action select
- Clear quiet hours button sets `clearQuietHours: true`
- Emits `preference.updated` event on save
- Error/loading states

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement channel-preferences.ts**

Component structure:
- Fetches channels and preferences on connect
- Builds one schema section per channel with deliveryMode toggle
- Rebuilds sub-schema on deliveryMode change (IMMEDIATE hides digest fields)
- Rebuilds digest sub-schema on schedule type change
- Save maps UI model to `NotificationPreferenceUpdate`
- Quiet hours section with clear button

- [ ] **Step 4: Add export, run tests, commit**

---

### Task 5: Mute List

**Repo:** `casehub-blocks-ui`

**Files:**
- Create: `components/notification-inbox/src/mute-list.ts`
- Create: `components/notification-inbox/src/mute-list.test.ts`
- Modify: `components/notification-inbox/src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi.listMuteRules()`, `.addMuteRule()`, `.removeMuteRule()` from `api.ts`
- Consumes: `MuteRule`, `MuteRuleInput` from `types.ts`
- Produces: `<mute-list>` custom element

- [ ] **Step 1: Write failing tests**

Create `mute-list.test.ts` with tests for:
- Fetches and renders rules in pages-table
- Add form shows inline schema-form with scope/scopeId/entityType/expiresAt
- Scope change to CATEGORY hides entityType; ENTITY shows it
- Submit calls `addMuteRule()` with correct input (userId/tenancyId from identity)
- Remove shows confirm dialog, calls `removeMuteRule()` on confirm
- Emits `mute.created` and `mute.deleted` events
- Error/loading states

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement mute-list.ts**

Component structure:
- `<pages-table>` for existing rules (same pattern as subscription-list.ts)
- "Add Mute Rule" button toggles inline `<pages-schema-form>`
- Schema rebuilds on scope change (ENTITY shows entityType, CATEGORY hides it)
- `<blocks-confirm-dialog>` for remove confirmation

- [ ] **Step 4: Add export, run tests, commit**

---

### Task 6: Snooze Control

**Repo:** `casehub-blocks-ui`

**Files:**
- Create: `components/notification-inbox/src/snooze-control.ts`
- Create: `components/notification-inbox/src/snooze-control.test.ts`
- Modify: `components/notification-inbox/src/index.ts`

**Interfaces:**
- Consumes: `NotificationApi.getSnooze()`, `.activateSnooze()`, `.cancelSnooze()` from `api.ts`
- Consumes: `Snooze` from `types.ts`
- Produces: `<snooze-control>` custom element

- [ ] **Step 1: Write failing tests**

Create `snooze-control.test.ts` with tests for:
- Shows activate form (date-time input) when not snoozed
- Shows "Snoozed until {time}" with cancel button when snoozed
- Activate calls `activateSnooze()` with date-time value
- Cancel calls `cancelSnooze()`
- State transitions: not snoozed → activate → snoozed → cancel → not snoozed
- Emits `snooze.activated` and `snooze.cancelled` events
- Error/loading states

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement snooze-control.ts**

Simplest component — two render paths based on snooze state.

- [ ] **Step 4: Add export, run tests, commit**

---

### Task 7: Notification Preferences Container

**Repo:** `casehub-blocks-ui`

**Files:**
- Create: `components/notification-inbox/src/notification-preferences.ts`
- Create: `components/notification-inbox/src/notification-preferences.test.ts`
- Modify: `components/notification-inbox/src/index.ts`

**Interfaces:**
- Consumes: `<channel-preferences>`, `<mute-list>`, `<snooze-control>` from Tasks 4–6
- Produces: `<notification-preferences>` custom element

- [ ] **Step 1: Write failing tests**

Create `notification-preferences.test.ts` with tests for:
- Renders all three child components (channel-preferences, mute-list, snooze-control)
- Passes endpoint and identity to children
- Three stacked sections with headings

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Implement notification-preferences.ts**

Pure layout container — three `<section>` elements with headings, each rendering a child component with `.endpoint` and `.identity` passed through.

- [ ] **Step 4: Add export, run tests, commit**

---

### Task 8: GDPR Erasure Action Migration

**Repo:** `casehub-blocks-ui`

**Files:**
- Modify: `components/gdpr-erasure-action/src/gdpr-erasure-action.ts`
- Modify: `components/gdpr-erasure-action/src/gdpr-erasure-action.test.ts`

**Interfaces:**
- Consumes: `FieldSchema` with `oneOf` from `@casehubio/pages-form`
- Produces: Updated `<gdpr-erasure-action>` — same public API, form rendering via schema-form

- [ ] **Step 1: Write/update failing tests**

Update `gdpr-erasure-action.test.ts`:
- Form renders via `<pages-schema-form>` (check for the element in shadow DOM)
- Subject ID field is rendered as text input
- Reason field renders as select with options from `reasonOptions`
- Schema rebuilds when `reasonOptions` property changes
- Form submit still triggers confirmation flow
- Three-phase flow (input → confirm → receipt) still works

- [ ] **Step 2: Run tests to verify they fail**

- [ ] **Step 3: Migrate form to schema-form**

Replace the hand-coded form HTML with:

```typescript
import '@casehubio/pages-form';
import type { FieldSchema } from '@casehubio/pages-form';
```

Build schema in a method:

```typescript
private _buildSchema(): FieldSchema {
  return {
    type: 'object',
    properties: {
      subjectId: {
        type: 'string',
        title: `${this.subjectLabel} ID`,
        placeholder: `Enter ${this.subjectLabel.toLowerCase()} ID`,
      },
      reason: {
        type: 'string',
        title: 'Erasure Reason',
        oneOf: this.reasonOptions.map(opt => ({ const: opt, title: opt })),
      },
    },
    required: ['subjectId', 'reason'],
  };
}
```

Rebuild schema in `willUpdate` when `reasonOptions` changes.

Replace the form HTML in `render()` with `<pages-schema-form>`, keeping the warning banner above it and the button group below (via the `actions` slot).

Remove the hand-coded form CSS (`.form-field`, `label`, `input`, `select` rules) — `pages-schema-form` owns the field styling. Keep `.form-container`, `.button-group`, `.warning`, `.error-text`, `.receipt-*`, `.btn-*` styles.

- [ ] **Step 4: Run tests**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui/components/gdpr-erasure-action test`
Expected: ALL tests PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui add components/gdpr-erasure-action/src/gdpr-erasure-action.ts components/gdpr-erasure-action/src/gdpr-erasure-action.test.ts
git -C /Users/mdproctor/claude/casehub/worktrees/12/blocks-ui commit -m "refactor(#34): migrate gdpr-erasure-action form to pages-schema-form

Replaces hand-coded form HTML with schema-driven rendering. Schema
rebuilds on reasonOptions change. Three-phase flow unchanged."
```
