---
title: The Controller Was the Point
date: 2026-07-10
entry_type: note
subtype: diary
tags: [blocks-ui, datasource, architecture, lit]
project: casehub-blocks-ui
issue: "#44"
---

I started #44 thinking the migration was mechanical — swap one mixin for another, keep the same fetchData pattern, done. The issue even said "one-line mixin swap per component." I'd read the DataSourceController code, understood the state machine, and figured blocks-ui just needed a thin Lit wrapper around it.

Then I got corrected. Hard.

The whole point of DataSourceController is that components *stop reimplementing fetch lifecycle*. Every blocks-ui component that used DataEndpointMixin had its own fetchData method — its own loading flag, its own error clearing, its own abort controller, its own connect/disconnect. About 80% of each component was identical state machine boilerplate with slight variations and inconsistent edge-case handling. I'd been treating that as a design constraint ("they all do domain-specific fetch logic") when it was the problem being solved.

Once that clicked, the design fell into place. Two layers:

**DataSourceAdapter** — a Lit ReactiveController wrapping DataSourceController. Same pattern as the EventStreamController already in blocks-ui-core. Components create one per data concern, get lifecycle for free via `host.addController(this)`. Audit-trail-viewer, which fetches entries and verification in parallel, just creates two adapters. No special API — "have two" is the composition model.

**DataSourceMixin** — optional sugar for the common case. Provides `endpoint` as an HTML attribute, bridges `loading`/`error`/`dataSet` to the controller, and handles `configure()` batching for hosted mode. Three of four components use it. The fourth (audit-trail-viewer) uses the adapter directly.

The adapter is the reusable unit. The mixin is convenience. That distinction matters because it determines which components can opt out without losing lifecycle management.

We also wrote `fetchSource` — a raw JSON DataSource for domain components. Pages' `restSource` delivers TypedDataSet via extractDataSet, which is right for charts and data-table pipelines. Blocks-ui components work with domain-typed JSON (LedgerEntry[], CaseEvent[], TrustScoreResponse). fetchSource fills that gap: fetch, parse JSON, deliver through the DataSource interface. Fifteen lines, abort-safe, dynamic headers via function for refresh-safe state.

The design review caught real issues. The mixin originally had no DataReceiver setters — hosted pipeline push was broken. The `_configuring` guard was in willUpdate, which didn't cover subclass syncEndpoint calls. Import paths used `dist/` instead of barrel exports. The `as any` type assertion was too wide — `as never` is narrower and visible to strict checks. Sixteen issues total, thirteen verified with spec changes, three accepted as intentional design choices. The spec improved substantially.

A bonus fix: the case-timeline and audit-trail-viewer had been dumping raw JSON.stringify into their "Details" expandable sections. We replaced that with `renderPropertyTree` — a generic recursive renderer that handles objects as definition lists, arrays as comma-separated values or numbered items, booleans as Yes/No, nulls as dashes. Works for any JSON shape. Both components now show clean property trees instead of monospace JSON blobs.

Net result: -200 lines across the repo. The state machine boilerplate is gone. Components set an endpoint and receive data. Domain logic (URL construction, auth headers, response parsing) stays but moves to configuration — sourceFactory options, resolveEndpoint overrides, configure batches.

Next: #45 — trust-score-panel trend sparkline with simulated data source. The DataSource pipeline includes `simulated()` and `ScenarioController` for exactly this: generating synthetic data for demos without a backend. The trend section currently shows a placeholder. Time to use the infrastructure we just migrated to.
