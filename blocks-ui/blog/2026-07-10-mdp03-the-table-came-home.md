---
title: "The Table Came Home"
date: 2026-07-10
projects:
  - casehub-blocks-ui
  - casehub-pages
branch: issue-48-migrate-data-table-to-pages
tags: [architecture, migration, layering, web-components]
---

Moved `pages-data-table` and four a11y mixins from blocks-ui back to pages where they belong. The layering violation was clear: generic UI primitives living in a domain-component repo. A table that renders rows and columns has no domain awareness. Neither does `LiveRegionMixin`. They belong in pages alongside tokens, the event system, and the component base.

The migration itself was mechanical — import path changes, package.json updates, vitest config standardisation. The interesting part was what the migration exposed.

The design review caught that the a11y mixins should go to `pages-primitives` (a new package from the July 5 spec), not `pages-component`. The separation matters: `pages-component` stays framework-agnostic with zero `lit` dependency. `pages-primitives` is where Lit-coupled code lives. Four rounds, fifteen issues, all resolved — the adversarial process earned its cost.

The real discovery came when I looked at the pages examples. The runtime was still rendering tables using a stale `PagesTable.js` in `pages-viz/dist/` — source deleted sessions ago, compiled output never cleaned. Two overlapping table implementations is unacceptable. But replacing `PagesTable` with `pages-data-table` in the runtime isn't a simple tag swap. The old table speaks `TypedDataSet` — branded column IDs, discriminated cell value unions, typed row accessors. The new table takes `rows: unknown[]`. That's a type-safety downgrade.

This is the deeper problem. When Lit components were built in blocks-ui, they weren't based on `TypedDataSet`. Every component re-derives structure the platform already has. The migration fixed where the code lives, but the data contract gap remains. Filed #49 to address it — make `pages-data-table` consume `TypedDataSet` natively, rename to `pages-table`, replace the stale `PagesTable`, and migrate the examples. The table examples don't move until the data contract is right.
