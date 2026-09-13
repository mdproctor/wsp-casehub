---
layout: post
title: "The Pipeline Closes"
date: 2026-07-11
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [data-pipeline, typeddataset, implementation]
series: issue-49-typeddataset-native
---

*Part of a series on [#49 — TypedDataSet native](https://github.com/casehubio/blocks-ui/issues/49). Previous: [Fetch Is Just Push](2026-07-11-mdp01-fetch-is-just-push.md).*

## The `as never` Dies

The single worst line in blocks-ui was in `fetchSource`:

```typescript
sink.apply({ type: "snapshot", dataset: data as never });
```

Raw JSON from `r.json()`, cast to bypass the type system entirely, shoved into a `SnapshotEvent` that claims to carry `TypedDataSet`. No `TypedRow` accessors. No branded `ColumnId`s. Every downstream consumer forced to cast back: `this.dataSet as any`, `Array.isArray(data)`, `entries.dataSet as LedgerEntry[]`.

We replaced it with the extraction pipeline that pages already had — `extractDataSet` takes the raw JSON, tabulates it into columns and rows (inferring schema from object keys when columns aren't declared, using explicit `ExternalColumnDef` when they are), and produces real `TypedDataSet` with working `cell()`, `number()`, `text()`, `date()` accessors. The pages session had already done the heavy lifting: `ExtractionDef` type narrowing, `fromRows()` factory, `SnapshotEvent.totalRows`, the `DataReceiver.dataSet: TypedDataSet | undefined` contract fix, and the full table redesign.

On the blocks-ui side, `fetchSource` gained `columns`, `dataPath`, and `totalPath` options. `DataSourceAdapter` and `DataSourceMixin` propagate the correct type. `createSourceFactory` passes the options through.

## list-pane — First Consumer

list-pane was the proving ground. Before: it received `unknown` from `DataSourceMixin`, did `const data = this.dataSet as any`, checked `Array.isArray(data)` or `data.items`, stored rows in private state, and built `ColumnDef` arrays with `getValue` extractors. After: it passes `this.dataSet` directly to `<pages-table>`. The `_rows`, `_totalRows`, `_lastDataSet` state variables and the `willUpdate` extraction logic are gone. The component shrank from 125 lines to 100.

One surprise: the extraction pipeline throws `EMPTY_RESULT` on empty arrays. An endpoint returning `[]` produces `sink.error` instead of an empty `TypedDataSet`. This is intentional for YAML data sources (where empty data signals misconfiguration) but wrong for REST APIs where zero results is normal. Filed as a garden entry — the upstream fix is straightforward but lives in pages-data.

## Five Consumers Left

trust-score-panel, audit-trail-viewer, work-item-inbox, notification-inbox, and subscription-list all follow the same migration pattern: replace `ColumnDef` with `columnConfig` + `columnRenderers`, replace `<pages-data-table>` with `<pages-table>`, delete the manual extraction from `unknown`. trust-score-panel and audit-trail-viewer have existing bugs (wrong property names on ColumnDef, wrong element property) that get fixed as part of the migration.
