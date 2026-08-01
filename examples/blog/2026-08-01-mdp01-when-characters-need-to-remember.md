---
layout: post
title: "Wacky Manor — When Characters Need to Remember"
date: 2026-08-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-examples]
tags: [llm, multi-agent, observation, context-management, casehub-blocks]
---

# Wacky Manor — When Characters Need to Remember

**Date:** 2026-08-01
**Type:** phase-update

---

## What I was trying to solve: LLM agents that forget everything they've seen

Wacky Manor is a multi-agent LLM simulation where cartoon characters navigate a mansion, scheme against each other, and interact with objects. Five characters, three rooms, each running their own async loop on virtual threads, each calling an LLM every turn to decide what to do next.

The problem is simple to state: what does each character know?

Until now, each character's observation included the last five raw events from whatever room they're standing in. Walk into the Kitchen, see the last five things that happened there. Walk back to the Entrance Hall — same thing, different room. No memory of what you saw before. No awareness that you left Penelope talking to Peter in the Ballroom. No sense that the Hooded Claw was fussing suspiciously with the tea service ten turns ago.

For a cartoon villain whose entire plot depends on remembering where the poison is and who's watching, this is a problem.

## What makes this harder than it looks: the context scaling trap

The obvious fix is "give them everything" — append every event to the prompt. But LLM prompts have a budget, and more context doesn't mean better context. Dump 200 events into a prompt and the LLM drowns in noise. The character can't distinguish between "Penelope walked into the Kitchen 30 turns ago" (stale) and "Sneekly just picked up the rat poison" (critical).

This is the same problem every LLM-powered system faces at scale. Customer service bots that lose track of what the customer already told them. Code assistants that forget the architectural decision made three files ago. Medical copilots that can't hold a patient's full history in context. The constraint is universal: LLMs need bounded, relevant context, and the world generates unbounded, noisy events.

The question isn't whether to batch and filter — it's how, and what you lose when you do.

## Two tiers of compaction and a service that owns them both

I wanted to solve two things at once: deduplication (characters re-reading the same events every turn) and volume scaling (too many events for a prompt to hold). The design uses casehub-blocks' `ObservationAccumulator` — a buffer-and-render primitive that collects events and drains them on demand through a pluggable renderer.

The central piece is `ObservationService` — it sits between the game loop and the character agents. Every world event (actions, dialogue, asides) flows through it. The service routes each event to the characters who can see it, based on room presence. Asides — villain monologues, private thoughts — route only to the speaker. Movement events route to observers in the departure room, because they saw the person leave.

Each character gets a per-room accumulator. When they move rooms, the old room's accumulator becomes a "remembered" partition. On the next drain, it's compacted, cached, and rendered as a "Remembered" section in the character's observation. The cache is the retention — no re-rendering, no progressive re-compression.

Compaction itself runs in two tiers. Mechanical compaction is deterministic and runs every drain: "Penelope moved to Kitchen" followed by "Penelope moved to Ballroom" — the Kitchen entry is gone. Same character taking the same object twice, same cabinet interacted with twice — later event wins. Duplicate dialogue is deduplicated. All of this operates on structured event metadata (action type, target, character ID), never on parsing narrative text.

When mechanical compaction isn't enough — a long remembered-room buffer with 20 dialogue lines — an LLM summariser fires via `AgentProvider`. It compresses conversation sequences into summaries: "Dastardly and Peter had a brief exchange where Dastardly gave misleading directions about the treasure room." The summariser catches exceptions and falls back to the mechanically-compacted text, so a timeout or API error never means data loss.

## What a character sees now

Here's the Hooded Claw's observation after moving from the Entrance Hall to the Kitchen:

```
== Recent Activity ==
- [2s ago] Sneekly: What have we here...
- [5s ago] Sneekly walked to the Kitchen.

== Remembered ==
Entrance Hall (30s ago): Penelope greeted the guests. Peter volunteered
to explore the dark corridor. Dastardly gave Muttley a fake medal.
```

The Recent Activity is verbatim — current room, last few events, timestamped. The Remembered section is compacted. Three rooms' worth of history, compressed to what matters. The Hooded Claw knows Dastardly gave Muttley the medal (relevant if he wants the brass key). He knows Peter went toward the corridor (opportunity to scheme while Peter is gone). He doesn't need to re-read 30 lines of dialogue to know this.

## Why this isn't just a cartoon game

The patterns here apply directly to any system where LLM agents operate over time in a shared environment:

**Multi-agent coordination.** When multiple agents share a world and act asynchronously, each agent needs a personalised view of what happened. Broadcasting everything to everyone doesn't scale — agents need visibility filtering (who can see what) and temporal compaction (what's still relevant).

**Cross-session memory.** The remembered-room mechanism is a miniature version of the problem every long-running LLM application faces: how do you carry forward context from earlier interactions without overwhelming the prompt? The answer is the same at every scale — accumulate, compact, tier. Verbatim for recent, grouped for moderate history, summarised for deep history.

**Graceful degradation.** The two-tier model means the system never fails hard. If the LLM summariser is down, mechanical compaction still runs. If even that produces too much, the renderer caps at a configurable line budget. The observation is always bounded, always populated, always degraded gracefully rather than catastrophically.

**Event-driven architecture over raw queries.** The shift from `world.recentEvents(roomId, 5)` to `observationService.drain(characterId, now)` is the shift from poll to push. Events flow through the service as they happen. The service routes, buffers, and compacts. The consumer just drains when ready. This is the same architectural pattern as Kafka consumers with compacted topics — latest-value-per-key semantics, but for LLM context windows instead of data streams.

## The pattern that wants to be a platform primitive

Looking at what we built, something became clear: `ObservationService` has nothing mansion-specific in its bones. Strip the cartoon names away and it's a generic `PartitionedObservationService<E, K>` — route events to per-observer partitioned accumulators with visibility filtering and drain-once memory retention. The domain-specific parts (room presence checks, aside privacy, mechanical compaction rules) are policy decisions that plug in through interfaces, not structural features of the service itself.

This is the kind of thing that belongs in casehub-blocks, not in an example application. A monitoring system needs per-device observation windows with compacted history when attention shifts. A customer service platform needs per-conversation-thread accumulation with cross-thread memory. Any multi-agent LLM deployment with contextual partitioning needs exactly what the Hooded Claw needs — except the partition key is "device" or "thread" instead of "room."

I've filed casehubio/blocks#76 to track the promotion. The key design question is the `VisibilityPolicy` interface — it determines how much routing logic lives in blocks versus the application. Get that boundary right and the whole partitioned observation pattern becomes a one-import solution for any multi-agent context management problem.

Phase 2.7 takes the next step: a live LLM narrator consuming the same accumulated event stream, generating running commentary for the audience. The observation pipeline we built here is the narrator's input source — and if blocks#76 lands first, the narrator will be the second consumer of the platform primitive, not the second copy of the pattern.
