---
layout: post
title: "The SSE Question Had the Wrong Premise"
date: 2026-07-09
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [datasource, sse, architecture, design-review, pages]
---

The pages team delivered casehub-pages#145 — the DataSource pipeline unification — and asked blocks-ui to confirm three design questions before they close it out. I spent the session reviewing both the spec and the implementation against first principles, and the most interesting finding was that one of their questions was based on a wrong assumption.

## The question that wasn't

Pages asked: "Does any blocks-ui component need raw SSE access beyond 'give me live data from this URL'?"

The question assumes SSE is a mode of data-fetching that the new DataSourceController should absorb. I traced every SSE usage in blocks-ui and the code tells a different story. Two completely independent patterns exist, and they don't overlap.

The DataEndpointMixin consumers — list-pane, trust-score-panel, case-timeline, audit-trail-viewer — use the mixin purely for REST fetch. None of them call `sseUrl()` or `handleSSEEvent()`. The mixin's SSE integration is dead code.

The components that actually use SSE — work-item-inbox, notification-bell, notification-inbox — all manage SSEManager directly. They need domain-specific event routing: work-item-inbox routes events by `WorkEventType` to appears/disappears/updated handlers with entry animations. Notification-bell subscribes to named events (`notification`, `notification-updated`, `unread-count`) and updates a badge count. These aren't "give me live data" — they're event interpreters with domain logic per event type.

The DataSource abstraction delivers dataset snapshots via `sink.apply()`. That doesn't fit event-driven state machines. The answer: blocks-ui does need raw SSE, but it's orthogonal to the controller. Components will compose DataSourceMixin *and* SSEManager, not choose one.

## What pages shipped

The implementation is clean. All six deliverables landed as specified: DataSourceController (a framework-agnostic state machine implementing VizTarget), DataReceiver with `loading: boolean`, VizTarget moved to pages-component, standalone source factories (restSource no longer needs ResolverContext — it calls `fetch` directly), EventStream `onReconnect` callback, and SSEManager explicitly left unchanged.

The `restSource` fix matters most architecturally. Previously, a standalone component would need to construct the entire pages runtime — DataSetManager, providerFactory, presetRegistry, capabilities — just to fetch JSON from a URL. Now it's `fetch → extractDataSet → sink.apply`. The pipeline concern no longer leaks into the source abstraction.

One deferred item: `createSourceFromUrl` returns a no-op. The convenience `controller.endpoint = "/api/items"` auto-routing needs a cross-package import strategy. Pipeline mode works fully, and `controller.source = restSource(...)` works for standalone. Only the URL-scheme auto-detection is deferred — a follow-on, not a gap.

## The branch archaeology

The other half of this session was housekeeping. Four project branches had their work on main but were never stamped with the closure commit. Eleven closed workspace branches — one had a blog entry that never reached workspace main, two had plans stranded on their branches. Six blog entries across the workspace had never been published to either destination.

IntelliJ also needed attention — both blocks-ui and pages opened with only "External Libraries" visible and no source tree. The root cause: TypeScript projects opened via MCP don't get the `.iml` and `modules.xml` files that Maven/Gradle projects get automatically. A `WEB_MODULE` content root definition fixed both.

The kind of drift that accumulates quietly across sessions. Worth catching before it compounds.
