---
layout: post
title: "Three Repos, One Component Library"
date: 2026-07-13
type: phase-update
entry_type: note
subtype: diary
projects: [blocks-ui]
tags: [channel-activity, connectors, component-promotion]
---

The channel-activity component started as a two-line stub. The real implementation lived in two other places — claudony had a hand-rolled channel panel with SSE, polling, and cursor management, and connectors had a full Lit-based chat UI with threading, reactions, emoji, member presence, and the qhorus speech-act model.

Epic #39 recommended promoting claudony's code. The audit reversed that. Claudony's channel-panel predated the pages data pipeline entirely — it hand-rolled its own EventSource with poll fallback, managed cursors in sessionStorage, and rendered a flat message list. Connectors had the richer architecture: sender grouping within two-minute windows, threaded replies via `inReplyTo`, commitment state tracking, artefact reference chips, markdown content rendering, keyboard navigation, and full `pages-event` integration. The data model wasn't close.

The design question shifted from "what do we port from claudony?" to "what do we port from connectors, and what extension points does claudony need?" The answer: everything from connectors' primitives comes across as shared components, claudony customises through typed config properties and render callbacks — the same pattern audit-trail-viewer uses with `renderEntryPayload`. We formalised this as a protocol: config properties first, optional render callbacks with defaults second, factory method overrides for cross-cutting behaviour third. Slots reserved for layout shells only.

The fit-gap analysis surfaced five things connectors didn't have. A message type selector — claudony lets humans pick the speech-act type (COMMAND, QUERY, STATUS), connectors just sent untyped text. Terminal and event message styling — dimming DONE/FAILURE/DECLINE/HANDOFF messages and italicising EVENT messages. Auto-scroll that respects whether the user has scrolled up to read history. Error feedback on the send input. And stale cursor detection — a timestamp-based prompt asking whether to catch up from where you left off or reload full history, emitting events for the host to handle.

The other thing that fell out of the audit: connectors can't keep its UI code. Work, engine, and other apps need connectors for the chat SPI and backend — pulling in a webui with npm dependencies and a vite build is the wrong coupling. All UI leaves connectors. The chat primitives go to blocks-ui, the workbench shell gets extracted as a standalone module, connectors becomes pure Java.

Two pages issues came out of this too. The pages `EventConnection` already receives gap information from the server on reconnect but throws it away — the reconnect listen is fire-and-forget. And cursor persistence is in-memory only, lost on page reload. Both are real issues for any real-time component, not just channel feeds. Filed as casehub-pages#174 and #175.

The implementation was eight components, ten source files, twelve test files, all renamed from `qhorus-*` to `channel-*` with event topics moved from `chat:*` to `channel:*` and `emitChatEvent` replaced by `emitPagesEvent` from blocks-ui-core. The extension points — `formatSender` for claudony's worker-name stripping, `renderContextHeader` for the case header above the feed, `showTypeSelector` with `allowedTypes`/`deniedTypes` filtering — all follow the protocol.

The interesting architectural point: the design review caught a real contradiction in the spec. The stale cursor section originally had the component fetching data directly, but the architecture said components receive data via Lit properties. The fix was event-driven — the component detects staleness, shows the prompt, and emits `channel:cursor-catchup` or `channel:cursor-reload` for the host to handle. The host fetches, the component renders. Same pattern as everything else.
