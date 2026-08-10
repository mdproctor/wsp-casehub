---
layout: post
title: "The Overlay That Wouldn't Stretch"
date: 2026-08-10
entry_type: note
subtype: diary
projects: [casehub-pages]
tags: [floating-workspace, dockview, css, runtime-integration]
---

The floating workspace landed in casehub-pages today — the last three implementation tasks (runtime integration, examples gallery, event contract), plus two runtime bugs that only surfaced once we drove the actual app in a browser.

The component itself is straightforward in concept: floating frames with tabbed content hovering over a main workspace area, backed by Dockview for the drag-and-resize mechanics. An engine/backend split keeps the state management pure and the rendering pluggable. Centre content renders synchronously; the Dockview backend lazy-loads. The spec had been through adversarial review; the implementation plan was detailed. Wiring it into the pages runtime — activation callback, frame event handlers, layout persistence through dock-rearrange teardowns — went cleanly.

Then we opened the examples gallery.

Two problems, both invisible to unit tests. First: every floating panel showed "No content." No console errors, no warnings, just blank panels with Dockview chrome rendered correctly around nothing. The content factory was being called, but it never received the tab configuration. Dockview v7's `createComponent` callback receives `{id, name}` — not the panel params. Those arrive later, via the `init()` lifecycle method. The fix was mechanical once diagnosed — defer content creation from construction to `init(params.params.tabConfig)` — but the silent failure mode made it harder to find than it should have been. Nothing in the dockview-core README mentions this.

Second: the floating frames could only be dragged horizontally. Vertical drag was constrained to about 85 pixels — the height of the markdown paragraph in the centre, not the height of the dock-workbench centre area. A `position:absolute;inset:0` overlay resolves against its containing block's dimensions. If that containing block is a flex child with implicit height, `inset:0` means "fill the content height," not "fill the available space." The fix: make the floating-workspace element stretch explicitly with `height:100%;flex:1;min-height:0`.

Both bugs share a trait: they pass every unit test and only manifest when the component is rendered inside a real dock-workbench layout with real content. The test harness uses happy-dom, which doesn't load Dockview at all — the backend init fails silently and the tests exercise a different code path. The Playwright e2e tests are the actual safety net here.

The content factory wiring is the piece I find most interesting architecturally. Dockview owns the panel lifecycle — it creates panels, calls your content factory, manages disposal. But pages owns the rendering pipeline — `renderComponent` with activation callbacks, context managers, data binding. The bridge between them is a single function that takes a `FrameTabConfig` (which carries a `content: Component`) and returns an `HTMLElement` with the pages renderer wired up inside it. That function runs inside Dockview's `init()` — after the panel exists in the DOM but before the user sees anything. Stateful content (terminals, WebSocket connections) will need the same bridge but with session lifecycle independent of panel lifecycle. The spec already accounts for this with the Terminal/Renderer separation pattern — `dispose()` releases the renderer, not the session.

The floating workspace is content-agnostic by design. Any pages component can be tab content. This is what makes it a platform primitive rather than a trellis feature — trellis becomes a consumer that provides a terminal content factory and holds its own backend reference for Electron-specific features via `unwrap()`.
