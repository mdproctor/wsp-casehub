---
layout: post
title: "The Pipeline Was Already There"
date: 2026-07-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
series: issue-95-align-table-pipeline
---

# CaseHub Blocks UI — The Pipeline Was Already There

**Date:** 2026-07-25
**Type:** phase-update

---

## What I was trying to achieve: aligning ten components with an API they predated

The pages pipeline — `DataSource → SourceConnector → DataSourceController` — handles fetch lifecycle, loading/error state, and dataset delivery for both dynamic and static table usage. blocks-ui's table components were written before this API existed. They manually called `fetch()`, built `TypedDataSet` via `fromRows()` on every render, and managed their own loading state. The pipeline was right there; they just weren't using it.

## What we found: four distinct bypass patterns

The ten components fell into tiers by how far they'd drifted from the pipeline.

Five components — compliance-summary, similarity-panel, routing-rationale, trust-score-panel, and the table in trust-workbench — already used `DataSourceMixin` for the endpoint path but bypassed it for inline data. They'd call `fromRows()` in `willUpdate` or the render template, setting the mixin's property directly instead of flowing data through the controller. The fix was small: let the controller manage the dataset whether data arrives via endpoint or property.

The more interesting cases were the three that avoided the pipeline entirely. audit-trail-viewer created two `DataSourceAdapter` instances but pushed *empty* datasets to the sink — using the adapter purely for loading/error state tracking while keeping the real data in component state. preferences-editor extended plain `LitElement` with its own `_loading`/`_error`/`_dataSet` fields, duplicating what the adapter provides. work-item-inbox had the most complex lifecycle — manual `fetch()`, SSE via `SSEManager`, `fromRows()` on every render cycle, manual filter application — and was tightly coupled enough that a full adapter adoption would have meant restructuring the SSE lifecycle.

## The fix that mattered most was the smallest

routing-rationale had a genuine bug: the endpoint path stored `_rawData` and built the dataset through the pipeline, but never built the column renderers. Only the property path — the `willUpdate` branch — called `buildColumnRenderers()`. In the endpoint path, the table rendered with an empty renderer map. No custom trust score bars, no phase badges, no status pills. One line added to the sourceFactory callback fixed it.

trust-score-panel had the most wasteful pattern. The sourceFactory transformed the API response into capability score rows and pushed a dataset through the pipeline — correct. Then `_renderCapabilityTable()` ignored that dataset entirely, built a *new* one from the raw response data, created fresh renderers and config objects, all on every render cycle. Hoisting the renderers to stable class members and binding the table to `this.dataSet` from the pipeline eliminated the duplication.

## What we left alone

work-item-inbox already had `client-sort` wired. Its data lifecycle — parallel fetch of items and summary, SSE mutations on `this.items`, queue scope swapping the data source, bulk operations — is complex enough that forcing a `DataSourceAdapter` would have meant restructuring the SSE handlers and the three-tab filtering chain. The pragmatic change: extract `fromRows` from the render template into a `@state()` property recomputed when data or filters change. The render path binds to the precomputed dataset instead of rebuilding it.

Six of the ten components had sortable columns in their `TABLE_CONFIG` but no `client-sort` attribute on the table element — clicking sort headers did nothing. Adding the attribute was trivial but the gap is worth noting: it's the kind of thing that works in the pipeline's framework hosting mode (where sort events bubble to the framework) but silently fails in standalone mode. The alignment work exposed it.

The pipeline alignment is a first step. More alignment work follows — there are related issues for prefix naming (#94) and native element migration (#93). Each one narrows the gap between how pages intends components to work and how blocks-ui actually uses them.
