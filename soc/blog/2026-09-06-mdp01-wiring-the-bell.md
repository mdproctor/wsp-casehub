---
layout: post
title: "Wiring the bell"
date: 2026-09-06
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [notifications, platform-integration, subscription-engine, CDI]
---

The SOC dashboard shows incidents. The analyst workbench shows work items. SSE pushes updates in real time. But if a SOC manager isn't staring at the screen when a P1 drops, they don't know about it until they check. For a 24/7 operation where DORA mandates response timelines, that gap is a compliance risk.

The platform already has a full notification subsystem — subscription engine, dispatch pipeline, template resolution, delivery channels, digest batching, suppression, quiet hours. The blocks-ui repo has a complete notification inbox component suite. None of it was wired to SOC.

So the job was narrow: define what SOC events mean in notification terms, bridge them to the platform DataSource, and seed the default subscriptions that make the bell ring.

## What we built

Three production files. That's the entire feature.

`SocNotificationEvents` in api/ defines eight event type constants — four for incident lifecycle (created, escalated, resolved, SLA breached) and four for work item events (created, assigned, escalated, completed). Each comes with `EventTypeDescriptor` factories that register filterable fields (severity, tactic, assigneeId) so the subscription editor knows what to offer.

`SocNotificationBridge` observes three CDI event sources and publishes `SubscribableEvent` records to the notification DataSource. The interesting part is how it handles work item filtering. `WorkItemLifecycleEvent` carries a `callerRef` in `case:{uuid}/pi:{uuid}` format — but no case definition name. You can't tell from the event alone whether a work item belongs to an SOC incident or an AML investigation. The bridge maintains a `ConcurrentHashMap` of active SOC case IDs, populated by the incident status observer, and checks incoming work item events against it. Simple, in-memory, and correct enough for v1 — with a documented edge case around restarts.

`SocNotificationSeeder` creates eight SYSTEM-scope subscriptions per tenant at startup. SYSTEM scope means analysts can mute or snooze but can't delete the subscription — the compliance chain stays intact. Idempotent via content comparison: if the template hasn't changed, no update. If it has, the seeder overwrites. No version field needed.

## The design question that mattered

SYSTEM vs USER scope for default subscriptions. USER scope gives analysts full control — including the ability to silently delete a containment approval notification. For a SOC under DORA and SOC2, that's a compliance gap you can't audit your way out of. SYSTEM scope with mute/snooze escape valves gives individuals enough flexibility without letting anyone break the notification chain.

The decision review surfaced a second important correction: the original plan filtered platform work item events by `caseType`. That field doesn't exist on `WorkItemLifecycleEvent`. The reviewer caught it, and we pivoted to SOC-bridged work item events with callerRef-based case tracking instead of trying to filter platform-wide events by a field that isn't there.

## What this opens up

The subscription engine evaluates MVEL and JQ filter expressions. Users can create their own subscriptions — "notify me only for P1 incidents involving credential access" — using the fields SOC registered. The blocks-ui subscription editor renders this automatically from the `EventFieldDescriptor` declarations.

Digest batching and quiet hours are platform-provided and already wired. A SOC manager can configure "batch P3/P4 notifications into a daily digest at 08:00" without SOC code changes. The plumbing is there; the knobs are exposed.

The piece that isn't in scope yet: delivery channel configuration. The platform has Slack, email, and webhook connectors. SOC needs to register which channels are available and wire the delivery adapters. That's the next layer — taking notifications from in-app to out-of-band.
