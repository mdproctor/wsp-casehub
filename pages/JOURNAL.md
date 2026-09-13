# Design Journal — issue-428-visual-yaml-builder

## 2026-09-12 — Design + foundation implementation

**Design phase:** Brainstormed the visual YAML builder across two scopes:
file-level (editing individual .page.yaml files) and project-level (the
full agentic AI application graph spanning eidos/engine/blocks/work/
neocortex/pages). Explored existing tools (Backstage Template Builder,
Google Blockly, Form.io, Kube Composer) for prior art. Key decisions:
hybrid progressive disclosure (full visual capability with natural YAML
slide), outline tree + property panel (Figma layers model), CST-backed
typed facade (format-preserving mutations via live Document object),
contextual component palette, and type-safe cross-document references
as a first-class architectural concern.

**Audit of CaseHub agentic AI YAML:** Explored all six authoring layers
with parallel agents — Eidos (org/agent profiles), Engine (case definitions
with capabilities/workers/bindings), Blocks (pattern/cognition/world/
pipeline YAML), Work (templates/progress/SLA), Neocortex (cognitive
profiles/CBR/RAG), and Pages (UI composition). Mapped cross-document
reference types and identified complexity areas requiring special builder
support.

**Spec reviewed:** 3-round Standard review (27 issues, 24 verified).
Major additions: container child descriptor registry, detailed DatasetNode,
RowNode/ColumnNode interfaces, mode-aware undo with two independent stacks,
FieldSchema bridge (not ZodType) for property palette, separate
pages-document package, hosting integration detail, testing strategy.

**Implementation started (Batch 1 complete):**
- Task 1: `pages-document` package with PageDocument, PageNode, RowNode,
  ColumnNode, DatasetNode, ComponentNode, NavTreeNode — 41 tests
- Task 2: Container descriptors, Zod→FieldSchema conversion, move/duplicate
  operations — 58 tests total across 3 files

**Key discovery:** `yaml` library's `Document.setIn()` with plain JS arrays
silently fails `isSeq()` when creating new top-level keys — must use
`doc.createNode([...])` explicitly (captured as GE-20260912-7742a0).
