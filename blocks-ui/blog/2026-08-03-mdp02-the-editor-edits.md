---
title: "The Editor Edits"
date: 2026-08-03
author: mdp
project: casehub-blocks-ui
tags: [diagram-editor, structural-editing, persistence, yaml, cst]
---

The earlier session gave us a viewer. This one gave us an editor.

Phase 4 adds the operations that make the diagram interactive: add nodes from a palette, delete with dependency warnings, switch a binding's target type between capability, subCase, and humanTask. Save to GitHub. Undo everything.

## What I was trying to achieve: structural editing without losing the YAML

The interesting constraint is that YAML is the source of truth — not the graph model. graph-core provides `addNode` and `removeNode` on `GraphModel`, but we never persist the graph. We persist YAML. So every structural edit has to mutate the YAML string directly, then re-derive the graph from scratch.

This is the same pattern Phase 3 established with `applyPropertyEdit` — parse the YAML with the `yaml` npm CST-preserving parser, mutate the document, emit the new string. Phase 4 extends it with `addElement`, `removeElement`, and `switchBindingTarget`. All three are pure functions: YAML string in, YAML string out.

The distinction from property edits matters at the rendering layer. Property edits don't change graph topology — you can skip the ELK layout and reuse existing node positions. Structural edits add or remove nodes, which means the layout has to recompute. That async ELK call creates a race condition Phase 3 never had to deal with.

## The render guard

Click the palette twice quickly. Two `addElement` calls fire. The first triggers `computeElkLayout` (async). The second triggers another `computeElkLayout` before the first finishes. The second render completes with intermediate state — one node, not two.

The fix is a `_renderInProgress` flag. While an ELK layout is in flight, new YAML mutations still happen synchronously (the YAML string is always current), but the render is deferred. When the in-flight render completes, it checks whether the YAML changed while it was running and triggers one more render if so. The final rendered state always matches the current YAML.

This also forced a change to undo/redo. Phase 3's undo called `_updateWithoutLayout` — fast, synchronous, position-preserving. But if you undo a structural edit (which changed topology), you need the full layout pass. Since the undo stack doesn't track whether each entry was a property edit or a structural edit, we switched undo/redo to always use `_fullRender`. The occasional unnecessary re-layout on a property-edit undo is cheap compared to the visual corruption of skipping re-layout on a structural-edit undo.

## Persistence is a property, not a mode

The `PersistenceBackend` SPI already existed in graph-core — `read(uri)` and `write(uri, yaml, expectedVersion)` with optimistic concurrency. `InMemoryBackend` was already there for playground mode. We added `GitHubBackend`, which maps to the GitHub Contents API: file SHA as the version token, base64 encoding, 409 on conflict.

What makes this clean is that persistence is a property on `<casehub-diagram>`, not a mode. Set `backend` and `uri` — the component loads, tracks dirty state, saves on Ctrl+S. Don't set them — the component works identically as a playground editor, no save button, no dirty indicator. The toolbar hides itself when there's no backend.

Dirty tracking uses string comparison: `_currentYaml !== _savedYaml`. This handles the undo-past-save-point edge case correctly — if you save, then undo twice, the current YAML differs from what was saved, so the dirty indicator comes back.

## What it is now

The diagram editor has four phases landed: types from the schema, a viewer that renders case definitions as interactive graphs, property editing with schema-driven forms, and structural editing with persistence. The remaining phases — SWF drill-down, work registry, runtime overlay — are independent tracks that build on this foundation without modifying it.

The palette has four buttons. The toolbar has one. The canvas fills the space between them. That's enough to author a case definition from scratch in a browser.
