---
layout: post
title: "Designing a visual builder for YAML-first applications"
date: 2026-09-12
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [visual-builder, yaml, cst, facade, agentic-ai, design]
series: issue-428-visual-yaml-builder
---

CaseHub is YAML-first. Pages, case definitions, agentic patterns, org structures, cognitive profiles, work templates — all authored as YAML documents. The CodeMirror editor with schema-driven autocomplete handles syntax, but it doesn't solve the harder problems: discoverability, structural composition, and the blank-page problem. A new user staring at an empty `.page.yaml` has no idea what's available or how to begin.

I wanted a visual builder that works as a complete authoring environment — users who never open the YAML pane aren't second-class citizens — while creating a natural gradient toward text fluency. Not a teaching tool that forces YAML exposure, and not a visual abstraction that hides it. A collapsible pane that starts closed, highlights the YAML range when you click a component in the tree, and lets you slide from fully visual to fully textual at your own pace.

The bigger picture is more interesting than the page builder alone. CaseHub applications span six authoring layers: Eidos (org topology and agent profiles), Engine (case definitions with capabilities, workers, bindings), Blocks (agentic patterns — supervisor, parallel, loop, voting, debate, HTN — plus cognition and world models), Work (human task templates, SLA), Neocortex (cognitive profiles, CBR memory, RAG), and Pages (the UI). A complete agentic AI application is a graph of YAML documents across all six, with typed cross-references between them. A capability name in an engine binding must resolve to a declared capability. A worker's agent ID must resolve to an Eidos profile. An agent's cognition config derives from its Jungian disposition profile.

This is where CaseHub can genuinely differentiate: type-safe cross-document references with rename refactoring across files. Most platforms treat cross-file references as strings. Rename a capability and you silently break every binding that references it. The project graph validates all references and propagates renames — the same type safety single-file languages take for granted, applied to multi-document YAML projects.

## The CST facade

The key architectural decision was choosing YAML's Concrete Syntax Tree as the single source of truth. Most visual editors build a typed semantic model, edit it, and serialize back to YAML — losing comments, whitespace, and the author's formatting on every save. The CST facade holds a live `Document` object from the `yaml` library and wraps it with typed node classes. A `ComponentNode` holds its CST path and delegates reads and writes directly to `Document.getIn()` and `Document.setIn()`. No second model, no divergence, no lost formatting.

Undo is string snapshots. Each visual mutation pushes `Document.toString()` onto a stack. Undo re-parses the snapshot. At page-document scale (5–50KB), `parseDocument()` completes in under a millisecond. The simplest correct approach.

The existing `yaml-edits.ts` primitives in pages-lsp operate as `(yaml: string) → string` — parse, mutate, serialize on every call. They stay unchanged for LSP callers. The facade bypasses them entirely, mutating the live Document directly.

## The platform audit

I ran parallel explorations across all six CaseHub layers to map what a visual builder needs to cover beyond pages. The findings shaped the architecture:

- **Blocks** splits each agentic application into four YAML files — pattern, cognition, world, pipeline — with recursive nesting in the pattern file (composed agents contain their own nested patterns)
- **Eidos** has weighted disposition vectors across five axes plus Jungian cognitive profiles. Raw number entry won't work; this needs sliders with sensible defaults
- **Engine** has a capability → worker → binding triad where missing any one silently breaks the chain. The builder must validate the complete triad
- **blocks-ui** already has LSP format registrations, Zod schemas, and symbol extractors for case-definition, org, HTN, and SWF documents — the Phase 2 facade work has a foundation

The CST facade pattern extends naturally. Each document type gets its own facade class (`CaseDocument`, `OrgDocument`, `PatternDocument`) sharing the same CST mutation substrate.

## First implementation

The `pages-document` package is the foundation — a new package in the casehub-pages monorepo with `PageDocument`, `PageNode`, `RowNode`, `ColumnNode`, `DatasetNode`, `ComponentNode`, and `NavTreeNode`. Container types (tabs, sidebar, split, form-scope) hold children in type-specific YAML keys that differ per container — a static descriptor registry maps each type to its child structure. A Zod-to-FieldSchema converter bridges the Zod schemas in `componentSchemaRegistry` to the `FieldSchema` interface that `PagesPropertyPalette` consumes.

58 tests across three files. Round-trip fidelity, structural mutations, layout modes, component properties, container children, move/duplicate operations, undo/redo, and the Zod conversion. The facade is solid.

Three batches remain: the outline tree and component catalog, the palette and property panel integration, and runtime edit-mode wiring with Playwright e2e tests. The facade does the hard work; the UI components are rendering and event wiring.

The part I'm most curious about is how the project-level graph will feel — seeing an entire agentic AI application's topology, with typed edges between org structures, case definitions, coordination patterns, and cognitive profiles. That's genuinely new territory.
