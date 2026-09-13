---
layout: post
title: "Why Every Data Table Gets Spanning Wrong"
date: 2026-07-19
type: phase-update
entry_type: note
subtype: diary
projects: [blocks-ui]
tags: [css-grid, virtual-scroll, data-table, architecture]
---

## Why Every Data Table Gets Spanning Wrong

The question that started this session looked straightforward: add row and column spanning to pages-table. It's a feature request that's been sitting since the component was built. Column spanning works — `grid-column: span N` inside a row's CSS Grid container is trivial. Row spanning is the problem.

The current rendering model creates each row as an independent `display: grid` container. Cells in row A have no layout relationship to cells in row B. That's the constraint, and it's structural — no amount of positioning trickery fixes it cleanly.

AG Grid's solution is `suppressRowTransform` — switch from `translateY` to `top` positioning, then let spanned cells overlay subsequent rows via z-index. Ant Design uses an `extraRender` callback to paint off-screen span origins as separate elements. Both approaches work. Both are hacks layered on top of the wrong abstraction.

## The grid was the answer all along

CSS Grid already knows how to span cells. `grid-row: 2 / span 3; grid-column: 2 / span 2` is a merged cell. The problem was never "how do we make cells span" — it was "why did we put each row in its own grid?"

The fix: one CSS Grid on the body container. `grid-template-rows: repeat(10000, 48px)` defines the full scrollable height as track metadata. Empty tracks are cheap — the browser doesn't allocate DOM for them. Cells go where they belong via explicit `grid-row` and `grid-column` placement. Row wrappers stay in the DOM with `display: contents` for ARIA semantics, but layout-wise they're invisible.

Virtual scrolling still works. Only visible cells are rendered, placed at their actual grid-row position. The browser handles scroll positioning natively — no `translateY` offset math. For spans that originate above the viewport, a boundary scan reads the suppressed cell's stored origin coordinates (O(1)) and extends the render window.

## The trade-off nobody mentions

`display: contents` means the row wrapper has no CSS box. `.row:hover` does nothing. `.row:focus { outline }` renders nothing. `::part(row)` can't match an invisible element. Row-level styling moves to per-cell application via JS state.

This sounds worse. It's actually more correct. When you hover a cell that spans four rows, which rows should highlight? The per-row model can't answer that — the row elements are independent. The per-cell model answers it explicitly: set a hover range, apply the class to every cell in that range.

## What shipped

The spec and implementation plan — not the code. pages-table lives in the pages repo, and the design review surfaced thirteen findings that all needed addressing before implementation: detail panel integration with paired auto-height tracks, focus management without CSS boxes, O(N) backward walks in the span map (fixed with stored origin coordinates), column-hiding semantics for colSpan, and six more.

The pages session has an epic with seven child issues, a build order, and a full implementation plan with TDD steps. blocks-ui's contribution was the design work — the architectural argument for why the rendering model needs to change, not just the spanning API that sits on top.

The single-grid model is the right foundation. It removes a constraint that every other grid library works around. That it also happens to make the code simpler — no `translateY`, no z-index stacking, no dual rendering paths — is the part that makes me think we should have done this from the start.
