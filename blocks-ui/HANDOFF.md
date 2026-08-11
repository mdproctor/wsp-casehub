# HANDOFF — casehub-blocks-ui

**Branch:** main (no active branch)
**Date:** 2026-08-11

## What landed

Worker function drill-down (#108). Workers in the case diagram now show a coloured function type badge (agent/flow/a2a/mcp/seq/ext) and the property panel has a function type selector with type-specific configuration forms. New `worker-function/` module in `graph-stencil-case` provides types, detection, defaults, and 6 form renderers (agent with nested provider selection, A2A with auth, MCP with stdio/HTTP transport switching, sequence with drag-reorder, unknown with raw JSON fallback, shared auth config). YAML editor gained `switchFunctionType`, `switchMcpTransport`, `switchModelProvider`. Pop-out prompt editor for agent systemPrompt uses native `<dialog>` to escape shadow DOM. 114 tests passing, build clean.

Design spec and decisions at `docs/specs/issue-108-worker-function-drill-down/`. Garden entry GE-20260811-f9c6f1 (nodeType DOM collision gotcha).

## What's left

- pages-table pagination buttons still use light backgrounds (upstream pages fix) · S · Low

## What's next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #110 | Conversation protocol viewer — convergence, epistemic status | L | High | Needs design |
| #111 | Orchestration monitor — execution lifecycle, audit chain | L | High | Needs design |
