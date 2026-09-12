---
title: "From YAML to Graph in One Session"
date: 2026-08-03
author: mdp
project: casehub-blocks-ui
tags: [diagram-editor, graph, yaml, stencils, lit, react-flow]
---

CaseHub case definitions are YAML. They describe bindings, workers, milestones, goals — the structural vocabulary of a case type. Until today, the only way to understand a case definition was to read the YAML. That changes with the visual diagram editor.

Three phases landed in a single session: schema verification (Phase 0), read-only viewer (Phase 2), and property editing (Phase 3). Eighteen commits, 64 tests, two new packages.

## The pipeline

The data flow is the interesting part. A CaseDefinition YAML string enters `toGraph()` and comes out the other end as a positioned, interactive graph:

```
YAML → parse → CaseAdapter.toGraph() → GraphModel → toReactFlowGraph() → ELK layout → React Flow canvas
```

`toGraph()` does the heavy lifting. It walks the parsed YAML, creates typed `GraphNode` instances for each element (workers, bindings, milestones, goals, subcases), and derives edges from string reference resolution — a binding with `capability: "ocr"` gets connected to the worker whose `capabilities` array includes `"ocr"`. Unresolvable capabilities produce annotation edges to synthetic external nodes. Not warnings — external workers are normal in CaseHub.

The edge derivation is where YAML becomes a graph. The schema defines relationships implicitly through string references; the adapter resolves those into explicit graph edges. A capability dispatch edge, a subcase spawn edge, a completion criteria edge — all derived, none stored.

## The stencil bridge

Stencils needed to be framework-agnostic — authored as lit-html templates (because blocks-ui is Lit), rendered inside React Flow (because the canvas is React). The bridge turned out to be ~20 lines: a React function component that creates a div ref, calls `litRender(template, div)` on mount and update, returns the div. Every stencil author writes a pure function `(data) => html\`...\`` and never imports React.

This matters because stencils will reappear in palette components, property panel previews, and potentially a future @xyflow/lit renderer. If they were React components, they'd be locked to React Flow. Lit-html templates go anywhere.

## Where YAML stays sovereign

The design spec made a decision early: YAML is the source of truth. There is no intermediate JSON graph model that accumulates state. Every property edit goes through `applyPropertyEdit(yaml, nodePath, field, value)` — the adapter parses a fresh `yaml.Document`, calls `setIn()`, returns the new YAML string via `toString()`. CST-preserving: change one property, the rest of the file keeps its formatting, comments, quoting style.

The undo model falls out of this naturally. Each edit pushes the previous YAML string onto a stack. Undo pops and re-parses. Every undo state is a complete, self-consistent YAML document. No command pattern, no graph-model-level undo — just strings.

The design review caught a subtlety: property edits don't change graph topology, so they shouldn't re-run ELK layout. The edit cycle is now synchronous — `applyPropertyEdit → toGraph → toReactFlowGraph → merge positions → update` — which also kills the async race condition from rapid edits overlapping in the layout pipeline.

## The schema verification surprise

Phase 0 was supposed to be the interesting part — verify the CaseDefinition JSON Schema against the Java model, find staleness from the stages removal. The surprise: the Java model classes are *generated from the same schema* via jsonschema2pojo. The schema is the single source of truth for both Java and TypeScript. There was nothing to verify except that the schema hadn't accumulated cruft. It hadn't. No stages remnants, no stale fields. The type generation was the real deliverable — `json-schema-to-typescript` producing complete interfaces from the 1230-line schema, with post-processing to handle `exactOptionalPropertyTypes` conflicts on index signatures.

## What's next

Phase 4A (structural editing — add/remove/replace nodes) and Phase 4B (persistence backends — Git, in-memory, Electron) can run in parallel. The adapter's `applyEdit()` method is the missing piece. Phase 5 (SWF drill-down) is blocked on `@openworkflowspec/sdk` availability.

The branch has the viewer, the property editor, the type generation pipeline, and the Lit→React bridge. The foundation is down. The hard part — structural editing with YAML round-trip, palette drag-and-drop, and conflict resolution — is next.
