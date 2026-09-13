---
layout: post
title: "Three Bugs Where There Was One"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [validation, web-components, testing]
---

## Three Bugs Where There Was One

The sla-indicator invalid date finding looked straightforward in the issue: set `deadline` to a non-ISO string, get NaN in the countdown text. A one-line guard in `_update()`, move on.

I traced the actual execution path and found three failures, not one. JavaScript's NaN comparison semantics — `NaN <= 0` is false, `NaN > 0` is also false — meant every threshold check in `_computeState()` silently fell through. The component returned `'normal'` for garbage input: green state, breach text. Then `render()` called `new Date("garbage").toISOString()`, which throws `RangeError`. A validation gap at the property boundary cascaded into incoherent internal state, incoherent visual output, and a render crash.

The fix needed to be more than a guard. We added a `@state() _deadlineValid` flag, computed once in `_update()` and checked in `render()`. Invalid deadlines reset `_remaining` and `_state` to coherent values, emit no events, and render nothing. A `console.warn` fires on the valid-to-invalid transition so misconfiguration is observable during development.

The warn initially looked simple — add it inside the NaN guard. Claude's adversarial reviewer caught the problem: `SharedTimerController` calls `_update()` at 1Hz. An invalid deadline would log the same warning every second. The fix: guard the warn with `if (this._deadlineValid)` — it fires exactly once per transition, then the flag prevents repetition on subsequent ticks.

The other two components — approval-gate and kpi-metric-row — needed test coverage, not production changes. The approval-gate error path (`_submitDecision` catch block) was already correct but entirely untested. The kpi-metric-row endpoint mode had never been exercised at all: no test for the loading skeleton, no test for fetch errors, no test for `refresh()`. One test documented that changing `endpoint` after mount silently does nothing — the component only fetches in `connectedCallback`. That's a reactive gap worth fixing separately.

The skeleton loading test needed a non-resolving promise (`new Promise(() => {})`) to hold `_loading = true` long enough for assertion. A resolving mock would complete the loading-to-loaded transition within a single Lit update cycle, making the skeleton unobservable. Small detail, easy to get wrong.
