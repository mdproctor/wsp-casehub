---
title: "The form that lived in the wrong house"
date: 2026-07-18
tags: [blocks-ui, pages, schema-form, migration, platform-layering]
---

Schema-form was sitting in blocks-ui-core. It rendered JSON Schema as editable
forms — text inputs, dropdowns, checkboxes, dates, nested objects. Useful
component. Wrong repo.

The architecture is pages (infrastructure) → blocks-ui (domain-aware) → apps.
Schema-form doesn't know anything about cases, work items, or AML transactions.
It takes a schema, renders inputs. That's infrastructure — same tier as
pages-table, which we'd already migrated to pages months ago.

I'd known this for a while but hadn't acted on it. When we filed epic #81 for
schema-form enhancements (array editing, nested objects, validation), the
question surfaced: build #81 here and move later, or move first?

## The audit that should have come first

We moved the code. Then I looked at what pages already had.

Pages has six form input Web Components — `pages-text-input`, `pages-dropdown`,
`pages-checkbox`, `pages-date-picker`, `pages-number-input`, `pages-textarea`.
Each with proper a11y, theming, dataset binding. Plus type definitions, a base
class with submit-on-Enter, and three gallery examples.

The schema-form code I'd just moved renders its own raw `<input>` and `<select>`
elements. It duplicates every form input pages already provides, with less
functionality. The field-renderers.ts file is 130 lines of reimplemented wheel.

That gap analysis should have happened before the move, not after. The migration
was right — schema-form doesn't belong in blocks-ui. But copying it wholesale
into pages without checking what pages already has was sloppy.

## What schema-form actually adds

Once we separated the duplication from the genuine gaps, the picture clarified.
Pages has the individual field components. What it doesn't have:

- An orchestrator that takes a JSON Schema and renders a complete form
- A display mode (read-only formatted view — dates locale-formatted, booleans as Yes/No)
- Nested object rendering (recursive sub-forms with visual grouping)
- Array editing (add/remove for both primitive and structured items)
- Form-level data collection and submit with required-field validation
- A pluggable field registry for custom format renderers

These are the real value. The individual input rendering is implementation
detail — it happens to render the same HTML elements as pages' form inputs
because that's what forms are made of.

## The three-tab example

We built a gallery example that makes the tradeoffs visible. Three tabs, same
AML transaction data:

**Schema** — pass one JSON Schema object, get the complete form. Display/edit
toggle, nested Parties section with border-bar grouping, Tags array with
add/remove, LinkedTransactions object array. One component, one schema.

**HTML** — `type: html` escape hatch. Same visual result including nesting and
arrays, but every field hand-written as raw HTML. Full control, verbose.

**YAML** — pure DSL form inputs. Clean, dataset-bound, auto-save. But flat
fields only — no nesting, no arrays, no display mode, no datetime-local.

The comparison isn't about which is "better." It's about choosing the right
tool: schema-form for dynamic forms from API schemas, YAML for static
dashboard forms, HTML when you need layout control the DSL can't express.
