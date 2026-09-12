---
layout: post
title: "Fetch Is Just Push"
date: 2026-07-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [data-pipeline, typeddataset, architecture]
series: issue-49-typeddataset-native
---

## The Type That Leaked

I'd been staring at the wrong end of the problem. blocks-ui's Lit components consume data through `DataSourceMixin`, which wraps `DataSourceAdapter`, which wraps `DataSourceController`. Every layer exposes `dataSet: unknown`. Every consumer casts: `this.dataSet as any`, `Array.isArray(data)`, `entries.dataSet as LedgerEntry[]`.

The fix looked like it was at the table — make `pages-data-table` accept `TypedDataSet`. But when we traced the chain back, the root was `DataReceiver.dataSet: unknown` in pages-component. `DataSourceController` already handles `TypedDataSet` internally — its `append`, `replace`, and `remove` handlers all cast to it. `SnapshotEvent.dataset` is typed `TypedDataSet`. The `unknown` declaration was just wrong — a looseness from the original implementation that everything downstream inherited.

The other half of the problem was `fetchSource`. The pages data system has a proper 4-stage extraction pipeline: parse → navigate/extract → tabulate → convert. Every native data source uses it — inline data, WebSocket push, SSE streams. But `fetchSource` bypasses the whole thing: `r.json()` → `sink.apply({ type: "snapshot", dataset: data as never })`. Raw JSON masquerading as TypedDataSet. No `TypedRow` accessors, no branded `ColumnId`s, no `CellValue` discriminated union.

## One Pipeline, Many Originators

The design conversation kept circling back to the same question: how should fetch-produced data and push-produced data relate to each other? I was initially thinking about two modes — a TypedDataSet path and a domain-object path. Then we realised they're the same thing.

A fetch is just a single-event push. The only difference between HTTP fetch, WebSocket push, SSE stream, inline YAML, and simulated data is the originator — the trigger that causes data to arrive. Once data arrives, the processing is identical. The pages resolver already does this: `provider.fetch()` → `extractDataSet()` → `manager.apply(id, { type: "snapshot", dataset })`. `fetchSource` should follow the same pattern.

This collapsed the design. The requestor declares the schema it expects (column names, types). The pipeline uses declared columns for tabulation; when absent, it infers from the data shape. One pipeline, one output type, many triggers.

## ColumnDef Dies

The prior data-table spec chose `ColumnDef<R>` with domain-typed extractors — `getValue: (row: WorkItemRootResponse) => row.item.title`. That reasoning was sound when fetchSource bypassed extraction: data arrived as `unknown`, and consumers needed extractors to project domain shapes into cells.

With unified pipeline, `getValue` becomes redundant. The extraction pipeline projects domain shapes into `CellValue` cells. The rendering contract splits into three independently-varying concerns: data schema (from `TypedDataSet.columns`), cell rendering (via `columnRenderers`), and presentation config (via `TableColumnConfig`). Each varies independently.

For in-memory domain data that doesn't come from an endpoint — like trust-score-panel building a capability table from its response — `fromRows()` absorbs the extraction role at construction time. Domain-typed extractors produce TypedDataSet, which the table renders uniformly.

## Where It Stands

Three of six layers are implemented in the pages repo (uncommitted, waiting for a pages session to commit):
- `ExtractionDef` type narrowed from `ExternalDataSetDef` so callers don't need a dummy `uuid`
- `fromRows()` factory in conversion.ts
- `SnapshotEvent.totalRows` for pagination metadata
- `DataReceiver.dataSet: TypedDataSet | undefined` — the root fix
- `SourceFactory` extended with `SourceFactoryOptions` (columns, dataPath, totalPath)

The table redesign — renaming `pages-data-table` to `pages-table`, replacing the API surface — is the remaining pages work. Filed as [casehub-pages#152](https://github.com/casehubio/casehub-pages/issues/152). After that lands, blocks-ui rewires `fetchSource` through the pipeline and migrates six consumers.

The adversarial design review caught gaps I'd missed: pagination metadata threading, client-sort mechanics with `TypedRow`, the activation system's dead `<pages-table>` element (old PagesTable was already removed — the rename actually *fixes* activation rather than just cleaning up a name).
