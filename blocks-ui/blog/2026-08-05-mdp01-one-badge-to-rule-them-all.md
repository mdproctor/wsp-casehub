---
layout: post
title: "One Badge to Rule Them All"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [status-badge, registry-pattern, web-components, consolidation]
---

Three components in blocks-ui rendered status badges. Each had its own `_statusColors` record — a private `Record<string, string>` mapping state names to inline CSS. Each used the same `--pages-*` CSS custom properties. Each produced the same rounded pill. None shared any code.

The diagram overlay was a fourth implementation — hardcoded `TASK_STATUS_DECORATIONS` and `MILESTONE_STATUS_DECORATIONS` maps that produced `NodeDecoration` objects with raw hex colours instead of CSS variables. Same visual language, different rendering pipeline, completely disconnected from the pill code.

I wanted to fix this before it got worse. The engine defines at least ten distinct status enums — CaseStatus, TaskStatus, WorkItemStatus, WorkStatus, MilestoneLifecycleStatus, OutcomeKind, GroupStatus, SlaStatus, NodeState, and whatever the session manager uses. Each time a component needs to render one, someone was going to copy-paste another `_statusColors` block and hope they picked the right colour for REJECTED.

The design insight was that most states share semantics across domains. COMPLETED is green everywhere. FAULTED is red everywhere. PENDING is grey everywhere. The differences are in the edges — `case:WAITING` means something specific (blocked on external input), `task:DELEGATED` gets a border in the diagram but `workitem:DELEGATED` just gets an info-blue pill. A registry with cross-domain defaults and per-domain overrides handles both the common case and the exceptions cleanly.

The `STATUS_REGISTRY` is a static `Map<string, StatusDescriptor>` keyed by `${domain}:${state}`. Lookup checks `domain:state` first, then `*:state` for cross-domain defaults, then a neutral fallback. New domains get sensible rendering for every shared state name without registering anything — COMPLETED, PENDING, RUNNING, FAULTED, CANCELLED, SUSPENDED all just work. The only things you register are the states that need domain-specific treatment.

```typescript
const d = lookupStatus('case', 'WAITING');
// → { category: 'warning', icon: '⏳' }

const d2 = lookupStatus('case', 'COMPLETED');
// → falls through to *:COMPLETED → { category: 'success', icon: '✓' }
```

The `<status-badge>` component consumes this — pass `domain` and `state`, get a pill. The diagram overlay uses `toDecoration()`, which maps the same descriptors to `NodeDecoration` objects with vibrant hex colours via a separate `BADGE_COLORS` record. Two visual contexts, one source of truth for what each state means.

The review caught something we'd missed: the work-item-inbox doesn't render engine `WorkStatus` (8 states) — it renders `WorkItemStatus`, a 12-state UI-side composite that adds ASSIGNED, IN_PROGRESS, and ESCALATED. Building the `workitem:` domain from the wrong enum would have broken the retrofit silently. The fix was a separate `workitem` domain in the registry with the correct state set.

Retrofitting took three components off their private colour maps. `commitment-state-pill` now delegates to `<status-badge domain="commitment">` internally — deprecated but not removed, so existing consumers don't break. The diagram toolbar gained a case-level status badge that renders `CaseRuntimeState.caseStatus` directly — no `NodeDecoration` needed since the case itself isn't a node in the plan graph.

The extensibility matters more than the cleanup. Epics #110 and #111 (conversation protocol viewer, orchestration monitor) will each bring their own status enums — convergence states, execution states, agent results. With the registry, each domain registers its handful of unique states and inherits sensible defaults for the rest. Without it, each would have meant another `_statusColors` record in another component, diverging one copy at a time.
