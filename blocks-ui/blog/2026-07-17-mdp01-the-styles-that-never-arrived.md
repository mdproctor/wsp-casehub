---
layout: post
title: "The Styles That Never Arrived"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [blocks-ui]
tags: [shadow-dom, css-scoping, web-components, routing-rationale, trust-routing]
---

The column renderers in similarity-panel and compliance-summary weren't broken. The HTML was rendering. The elements existed. The tests passed. Everything about the code was correct — property binding, Map lookup, CellValue handling. The renderers executed, returned their template results, and pages-table placed them in the DOM exactly where they belonged.

The cells just looked like raw text.

I spent time tracing through property binding timing, checking whether `ColumnId`'s branded type caused Map lookup mismatches, comparing the pattern against work-item-inbox (identical code, identical "bug"). The investigation kept pointing at renderer execution — and that was the wrong trail entirely.

The root cause is shadow DOM CSS scoping. When similarity-panel defines `.bar-fill` in its `static styles` and returns `<div class="bar-fill">` from a column renderer callback, that HTML ends up inside pages-table's shadow DOM. The parent's stylesheet doesn't follow it across that boundary. The progress bar div exists — it has zero height, zero background, zero visual presence. In a browser, that's indistinguishable from "renderer not working."

jsdom made it worse. The tests checked for element existence (`.querySelector('.bar-fill')`), found the elements, and passed. Shadow DOM CSS scoping is invisible in jsdom — there's no CSS rendering engine to enforce it.

The fix is straightforward: column renderer output must use inline styles, not CSS classes. Inline styles travel with the element regardless of which shadow root it lands in. We converted all three components (similarity-panel, compliance-summary, and work-item-inbox — same bug, unnoticed) and removed the dead CSS class definitions.

This generalises. Any callback-based rendering pattern where the returned HTML lands in a different component's shadow DOM has the same problem. We formalised it as an addition to the component customisation protocol (PP-20260713-8ea1af): render callbacks must use inline styles.

With the renderer fix landed, we built the routing-rationale component — a trust-weighted assignment explanation panel. The data shape maps to engine's `TrustCandidateClassifier`: per-candidate phase classification, trust and workload scores, blended final scores, and the `TrustRoutingPolicy` parameters that govern the selection. The score header shows the selected candidate's trust score against the threshold, with the borderline margin as a shaded band. The alternatives table uses pages-table with inline-styled column renderers — phase badges colour-coded by maturity stage, score bars with threshold markers, status badges distinguishing "Selected", "Eligible", and specific exclusion reasons.

The data contract accommodates both `TrustWeightedAgentStrategy` and `SemanticAgentRoutingStrategy` through a strategy-agnostic `additionalScores` field — the design review caught that gap. No REST endpoint exists yet in engine (flagged as a gap in devtown's UI requirements), so the component supports the same dual-data mode as similarity-panel: property for when the app already has the data, endpoint for when the API is built.

The shadow DOM finding is worth internalising. When a web component accepts render callbacks, the contract is implicit: the caller provides HTML, the callee renders it in its own shadow root. That's a style boundary the caller can't see. Inline styles are the only portable format for cross-shadow content.
