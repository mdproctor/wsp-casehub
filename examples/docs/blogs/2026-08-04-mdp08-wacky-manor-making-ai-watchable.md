---
layout: post
title: "Wacky Manor: Making the AI Watchable"
date: 2026-08-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub]
tags: [eidos, ui, multi-agent, wacky-races, pause-play, character-profiles]
series: issue-27-pause-play-profiles
---

*Part of a series on [#27 — Pause/play and speed controls](https://github.com/casehubio/examples/issues/27). Previous: [The Mansion Runs](2026-07-27-mdp02-wacky-manor-the-mansion-runs.md).*

The previous entry ended with 17 characters acting simultaneously. The problem was immediate: nobody could read it. Dialogue from Penelope, Sneekly, the Ant Hill Mob, Dastardly, and twelve others flooded the chat panels faster than any human could follow. The entertainment value — the whole reason for building this — was lost in the noise.

So the task was clear. Not more AI. Better viewing.

**Pause and speed** turned out to be cleaner than I expected. A shared volatile `paused` flag on `WorldState`, checked by each character's agent loop before it makes an LLM call. When paused, no tokens burn — characters spin-wait on a 200ms sleep until you unpause. Speed is the same idea: a `speedMultiplier` on WorldState that scales each character's think delay at call time, leaving the base delay immutable on `CharacterState`. The frontend gets `0.5x | 1x | 2x | 4x` buttons in the toolbar. Click 4x when Lazy Luke is being lazy, 0.5x when the Hooded Claw starts scheming.

The WebSocket protocol gained a `control` event type — `paused`, `resumed`, or `speed` — broadcast to all connected clients. This means if you open a second browser tab, it stays in sync. A fix I had to add partway through: when a browser connects mid-scenario, `ManorEventBus.addListener()` now sends the current scenario status and control state on connect, not just the character snapshot. Without that, a reconnecting browser sits on "idle" with no transport controls visible, even though the scenario is running.

**The character profile panel** was the more interesting piece. Click any character sprite in the manor view and a slide-out panel appears showing the Eidos personality data that drives their behaviour: Belbin team role, Jungian function weights as a bar chart, capabilities with tags, public goals and constraints, and the full briefing text. Private goals — like the Hooded Claw's "eliminate Penelope" — are filtered server-side before the JSON reaches the browser.

I built this as a `CharacterProfileDTO` that projects from `AgentDescriptor` rather than sending the raw descriptor to the frontend. The raw descriptor carries operational fields like `provider`, `modelFamily`, `weightsFingerprint` — none of which the audience needs to see, and some of which you definitely don't want public. The vocabulary registry resolves the slot label ("Shaper" from the raw "shaper" term), and the disposition profile carries through as weighted Jungian function entries for the bar chart.

The profile panel is the educational payoff of the Eidos platform. You see a cartoon Penelope on screen, click her, and the panel reveals the structured composition that makes her act that way: ESFJ-aligned function weights, "riddle-solving" and "charm" capabilities, a "trusts everyone" constraint, and a briefing that establishes her Southern drawl and obliviousness to danger. The machinery becomes visible.

**Twelve new SVG sprites** brought the remaining characters to life visually — Muttley as a snickering brown dog, Pat Pending in a lab coat with goggles, the Slag Brothers as stocky cavemen in brown and grey, Big Gruesome as a large friendly purple monster. Each is about 20 lines of inline SVG at the same scale as the original five. The manor view now has 17 distinctive figures instead of 12 grey circles.

While wiring the profile endpoint, Claude found that `CharacterProfile` and `CharacterProfileLoader` — the original Phase 0 code for loading descriptors — had zero callers. The plan document from Phase 1 explicitly said "legacy, can be deleted." They'd survived three phases. We deleted them.

We also wired eidos's `BriefingCoherenceValidator` into the prompt renderer. It was already running inside `EidosSystemPromptRenderer` and computing coherence reports, but `ScenarioOrchestrator.renderPrompt()` was calling `.content()` and discarding the rest. Now it logs any TENSION or CONFLICT findings — cases where the briefing text contradicts the disposition profile. A briefing that says "outgoing" for a character whose dominant Jungian function is introverted will get flagged.

**The runtime surprise** was that 17 simultaneous `claude -p` subprocesses exhaust system memory. Each character loop spawns a Claude CLI process for every LLM call — that's 17 concurrent Node.js runtimes competing with the Quarkus JVM. An `LlmSmokeTest` proves the full pipeline works in isolation (descriptor renders, LLM responds with dialogue JSON), but the dev mode process collapses under the concurrency. The pragmatic fix: a `manor.scenario.active-characters` config property that limits which characters spawn LLM threads. The `%dev` profile runs the core five — Penelope, Sneekly, Ant Hill Mob, Dastardly, and Peter — while tests use all seventeen.

This surfaced a design question I hadn't considered: the game doesn't need all 17 characters running at once. It needs 4–5 characters with enough plot devices between them to create tension. The poison-in-the-tea scenario works because it gives the Hooded Claw a goal that conflicts with everyone else's. Twelve extra characters wandering the halls don't add tension — they add load. The next step is probably episodic scenarios: small casts, focused plot devices, each act self-contained but world state carrying forward.
