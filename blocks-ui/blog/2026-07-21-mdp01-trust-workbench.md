---
layout: post
title: "trust-workbench — one tag instead of three"
date: 2026-07-21
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [trust, web-components, composition]
---

Every CaseHub app that uses trust-weighted routing ends up building the same composed view. The trust score panel shows per-capability scores. A routing history list shows who got assigned and why. A feedback feed shows what happened after the routing decision — whether the gate approved, what the trust delta was. Three components, the same event wiring, rebuilt independently in devtown, clinical, and AML.

`<trust-workbench>` pre-wires all of it. One tag, one endpoint, working trust visibility.

## The layout

The layout follows the split-workbench pattern used everywhere in blocks-ui. Left panel: trust-score-panel at the top for orientation — the actor's global score, per-capability breakdown, trend sparkline when data exists. Below it, a routing history list showing timestamped decisions with score bars and phase badges. Click a capability in the score panel and the list filters to just that capability's routing decisions. Click a routing decision and the right panel fills with the full rationale — selected candidate, alternatives table with trust scores and workload, policy summary — plus the downstream feedback entries showing gate decisions and trust score deltas.

## The wiring that apps kept getting wrong

The capability-to-filter wiring is where most apps got it wrong independently. trust-score-panel emits `trust:capability-selected` when you click a capability row. The workbench catches it, toggles the selection (click again to clear), and updates the routing history endpoint with a `?capability=` query parameter. This sounds straightforward, but each app that wired it manually had a slightly different version — some didn't toggle, some forgot to reset the detail pane, some didn't cancel in-flight fetches when the user clicked rapidly.

## Three tiers

Three consumption tiers, same pattern as case-explorer. Tier 1: drop in `<trust-workbench endpoint="/api" actor-id="worker-42">` and everything works. Tier 2: pass custom column renderers for the routing history list, or a `renderCandidate` callback for the rationale detail. Tier 3: compose the individual components directly with your own event wiring — which is what the apps were doing before, and what they can stop doing now.

## The gauge that earned nothing

One thing that came out of building the showcase: the trust-score-panel's SVG arc gauge was taking a third of the panel height for a single number. A 200-pixel speedometer might look good on a standalone demo page, but inside a split-workbench where every pixel of vertical space matters, it was actively hostile to the layout. I replaced it with a compact score header — the number, a level badge, and a thin progress bar. Same information in 40 pixels instead of 200. The capability table and trend sparkline fit naturally below it, and the routing history list gets enough room to show six rows without scrolling.

## What's not there yet

The backend endpoints for routing history don't exist yet — the routing-rationale spec noted the same gap. The workbench works today via inline data mode: pass a `routingHistory` array and a `routingDetailResolver` function, and it renders identically without touching the network. When the engine team builds the REST endpoints, switching from inline to endpoint mode is one property change.

devtown is the first consumer — it originated the request. clinical and AML have the same trust visibility patterns and can drop it in once their routing is wired. Any app using trust-weighted routing from the engine gets this for free.
