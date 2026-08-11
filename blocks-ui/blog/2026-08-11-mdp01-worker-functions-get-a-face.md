---
layout: post
title: "Worker Functions Get a Face"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [web-components, diagram, worker, property-panel, lit]
---

Workers in the case diagram were opaque — a box with a name and a capabilities list, regardless of whether the thing behind it was an LLM agent, an MCP tool server, an A2A remote, or a simple step sequence. The engine knows the difference (its `WorkerFunctionProvider` SPI dispatches to the right implementation based on which YAML key is present), but the diagram had no way to show or edit that distinction.

The design tension here is between open and closed. The engine treats function types as an open set — any module can register a new provider for a new YAML key, and `additionalProperties: true` on the Worker schema means the schema never needs to change. The UI can't work that way. Each function type has its own configuration shape: agent needs a prompt editor and a provider-specific model form; MCP needs a transport selector (stdio vs HTTP); A2A needs an endpoint and auth. These aren't schema-drivable — they're discriminated unions with type-specific UX.

We went with custom render sections per function type, with a fallback to raw JSON for anything the UI doesn't recognise. The property form already had a precedent for this pattern — the binding target-type selector switches between capability, subCase, and humanTask forms using the same detect-and-dispatch approach. The worker function type selector is the same idea, one level deeper.

The agent form is the most complex: two nested discriminated unions (function type → agent → model provider). Each provider has the same core fields (modelName, temperature, maxTokens) but the selector needs to clear the old provider block and create a new one with defaults. The YAML editor already had `switchBindingTarget` doing exactly this kind of atomic key-swap — `switchFunctionType`, `switchMcpTransport`, and `switchModelProvider` follow the same CST-preserving pattern.

One thing that caught us: the pop-out prompt editor for systemPrompt. The property panel is a shadow DOM component inside a 300px sidebar. An overlay rendered in the panel's shadow root can't visually cover sibling elements outside its DOM tree. Native `<dialog>` with `showModal()` solves this — the browser's top layer escapes all stacking contexts. The parent diagram component owns the dialog and coordinates via events from the properties panel.

The spec review flagged something worth noting: `sequence` is an explicit property on the generated Worker interface (not just `additionalProperties`), which means `renderPropertyForm` would render it as a textarea AND the function section would render the drag-reorder editor. We filter function-type keys from the schema before passing it to the generic form — same keys that `switchFunctionType` removes when switching types.

The worker stencil now shows a coloured pill badge — agent in purple, flow in blue, a2a in teal, mcp in orange. Static type labels, not the StatusBadge component (which is for dynamic runtime state). At a glance you can see the function composition of a case definition without selecting any node.
