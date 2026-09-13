---
layout: post
title: "The Mansion Learns to Sleep"
date: 2026-09-13
entry_type: note
subtype: diary
projects: [casehubio/examples]
tags: [wacky-manor, neocortex, cognitive, consolidation, social-cognition]
series: issue-52-social-cognition-layer
---

# The Mansion Learns to Sleep

Wacky Manor's characters have operated on a stateless loop since day one: observe, think, act. No persistent beliefs, no social drives, no memory that outlives the current tick. The neocortex platform already had the infrastructure — CognitiveProfile for querying knowledge graphs, CognitiveDerivationEngine for deriving personality-driven cognitive defaults, ConsolidationScheduler for memory consolidation — but nothing consumed it. This branch wires it up.

The architecture is a three-tier memory model. Tier 1 is working memory: rebuilt every tick, never persisted. Tier 2 is an episodic buffer: recent events accumulating in-process, waiting for a reason to graduate. Tier 3 is the neocortex mindmap: consolidated knowledge written only during sleep. The key constraint is that the mindmap is never written during the high-frequency tick loop — consolidation bridges Tier 2 to Tier 3 during a deliberate pause.

That pause is now a game mechanic. Every fifty ticks, night falls. The narrator broadcasts "Night falls. The characters rest and reflect on the day's events..." and the orchestrator calls `consolidateNow(tenantId)`. One call, tenant-scoped — all characters' memories consolidate together through AccessFrequencyPhase, MergeDetectionPhase, CommunitySummaryPhase, and CuriosityRefreshPhase. Then dawn breaks and the tick loop resumes with characters carrying evolved understanding.

The interesting design choice was where personality enters. CognitiveDerivationEngine derives social cognition defaults from Eidos personality descriptors — trust formation rate, conflict interpretation mode. A character with high trust formation gains and loses trust faster. A character whose conflict interpretation is REPAIR takes less damage from theft than one who DISENGAGE. These modulate the base trust weights per action type, so the same event (stealing an item) has different cognitive impact depending on who witnesses it.

Each character now carries a `CharacterCognition` object that composes all of this: experience recall, trust event buffering, importance scoring, and cognitive section rendering. The observation the LLM sees includes four new sections — drives, principles, beliefs, and contextual norms — sorted by intensity or priority. The LLM reasons about "You believe Penelope is naive and trusts too easily" and "Your drive to scheme is at 90%" explicitly, which makes character behaviour transparent to observers watching the simulation.

The upstream dependencies shaped the implementation boundary. `consolidateNow` on ConsolidationScheduler landed (neocortex#323, now closed), so the sleep mechanic uses the real API. Cognitive node type registration (neocortex#322) is still open — trust events compute weights but don't persist them to the mindmap yet. The current `recordTrustEvent` validates the weight calculation against personality-modulated rates; actual Tier 2 to Tier 3 graduation waits for typed nodes. The social config is hardcoded in Java rather than parsed from Eidos extensionData — a pragmatic choice that keeps the config functionally equivalent without requiring an eidos schema extension.

What this opens up: characters that remember across sleep cycles, form persistent beliefs from experience, and develop trust relationships that compound over time. The cognitive stack is wired but the persistent layer is thin — once neocortex#322 lands, the Thing trait facades (`node.as(Belieflike.class)`) give typed access to consolidated knowledge, and the observation sections can switch from static initial beliefs to dynamically queried mindmap state.
