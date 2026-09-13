---
title: Schema-Driven Notification Forms
date: 2026-07-21
tags: [blocks-ui, pages-form, notification, subscription, preferences]
---

# Schema-Driven Notification Forms

The notification-inbox package needed a subscription editor and a preferences panel. Both are form-heavy — event type pickers, constraint builders, delivery channel toggles, mute rules, snooze timers. Hand-coding each form meant duplicating field rendering, validation, and error display across every component.

Instead, we extended `pages-schema-form` with four capabilities and built every form on top of it.

## What Changed in pages-form

Three rendering additions and one validation fix:

**Labeled enums via `oneOf`.** The existing `enum` field treats value and label as identical — fine for `['open', 'closed']`, unusable for `['EQ', 'NEQ', 'CONTAINS']` where the user needs to see "Equals", "Not equals", "Contains". `oneOf` follows the JSON Schema spec: `[{ const: 'EQ', title: 'Equals' }, ...]`. When the current value doesn't match any const (create mode), a disabled placeholder option renders automatically — no accidental submission with the first option selected.

**`format: 'time'`** for HH:mm inputs. Quiet hours need time pickers; the existing date and date-time formats didn't cover this.

**`readOnly`** for non-editable fields. Validation skips them entirely — they pass through without blocking submit.

**Recursive validation.** The original `validateField` only checked top-level properties. A nested `template` object with empty required fields passed validation and the first error came back from the server. Now validation recurses into nested objects and array items, returning compound errors like `template: titlePattern: Required`.

## The Subscription Editor

The most complex form. An event type dropdown populated from `GET /subscriptions/event-types`, where selecting an event type rebuilds the constraint field options, the `entityIdField` select, and the `actorIdField` select — all from the selected type's `EventFieldDescriptor[]`. The component builds the `FieldSchema` object, the form renders it, and when the event type changes the component rebuilds and sets a new schema.

## Notification Preferences

Three independent components composed by a container:

- **Channel preferences** — per-channel delivery settings with an immediate/digest toggle. Digest mode shows schedule configuration (interval, daily, weekly) and a groupBy selector. The `deliveryMode` is a UI concept — it maps to `digestSchedule: null` (immediate) or a configured schedule object on save.
- **Mute list** — a `pages-table` of active rules with an inline `pages-schema-form` for adding new ones. Scope-conditional: selecting CATEGORY hides the entityType field.
- **Snooze control** — two render paths based on state. Not snoozed shows a date-time form; snoozed shows a cancel button.

## The GDPR Migration

The existing `gdpr-erasure-action` had a hand-coded form with 40 lines of input/select HTML and 20 lines of field CSS. After migration: a `_buildSchema()` method that returns a 15-line `FieldSchema` object, and a `<pages-schema-form>` element in the template. The three-phase flow (input → confirm dialog → receipt) is unchanged — only the input phase rendering moved to schema-form. Net result: −48 lines.

## Numbers

- 4 schema improvements in pages-form (65 tests)
- 5 new components in notification-inbox (36 tests)
- 1 migrated form in gdpr-erasure-action (11 tests updated)
- 177 tests total, all passing
