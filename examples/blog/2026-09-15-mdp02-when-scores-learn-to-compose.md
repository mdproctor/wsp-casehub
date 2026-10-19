---
title: "When scores learn to compose"
date: 2026-09-15
author: mdp
entry_type: note
subtype: diary
series: issue-58-content-scorer-refactoring
projects:
  - casehubio/examples
  - casehubio/neocortex
  - casehubio/blocks
tags: [scoring, consolidation, cognitive-architecture, social-cognition]
---

The consolidation phase has been graduating all episodic events equally since it landed. Every trust event, every idle observation, every conflict resolution — all weighted the same when the mansion sleeps. That's wrong. A character resolving a tense standoff should matter more than noticing the curtains are dusty.

The problem isn't that we lack scoring heuristics. Blocks already has `SurpriseScorer` and `ArousalScorer` — they measure feature diversity and emotional intensity. But they operate on `ScoredCbrCase` via `ConfidenceScorer`, and consolidation operates on `Memory` via `GraduationScorer`. Different interfaces, different input types. The scoring logic is right there in the codebase, locked behind the wrong type boundary.

## The extraction insight

Both scorers are doing the same thing: reading text and metadata from a domain object, then computing a number. SurpriseScorer counts feature diversity. ArousalScorer matches keywords against a high-arousal word set. Neither actually needs the full `ScoredCbrCase` — they need the text, the metadata, and a timestamp. That's the signal, and it can be extracted from any domain type.

`ScoreableContent(text, metadata, timestamp)` is the intermediary. A Memory projects onto it. A ScoredCbrCase projects onto it. The scorers operate on the projection, not the domain type. Three new types in neocortex-memory-api: `ScoreableContent`, `ContentScorer` (`@FunctionalInterface`), and `WeightedContentScorer` for composition.

The dual-interface approach turned out clean. `SurpriseScorer` implements both `ContentScorer` and `ConfidenceScorer`. The `ContentScorer.score(ScoreableContent)` method is the real logic. The `ConfidenceScorer.score(ScoredCbrCase, Instant)` method extracts `ScoreableContent` from the CBR case and delegates. Zero changes in `MemoryHygieneOrchestrator`, `CompositeConfidenceScorer`, the YAML pipeline — none of the existing consumers know anything changed.

## The graduation formula

`ManorGraduationScorer` composes three dimensions:

- **Arousal** (0.3 weight) — keyword matching against high-intensity vocabulary
- **Action importance** (0.4 weight) — game-specific event scoring: conflict resolution ranks highest, idle ranks lowest
- **Confidence** (0.3 weight) — the memory's own confidence value

The 0.4 weight on action importance is deliberate. In a character-driven game, *what happened* matters more than *how intensely it was described*. A calmly narrated trust breach should outrank an exclamatory observation about the weather.

## Person nodes bring the social pipeline to life

The SocialComparison integration from the previous branch was complete but inert — `CognitiveProfile.compare()` found no entity nodes because none existed. We extended `SocialConfig` with relationship data (initial PAD values per observer-target pair) and added `ManorCognitiveSeeder.seedPeople()`, which creates a shared "people" subgraph with one Entitylike node per character and perspectival overlays per observer. Each overlay carries initial pleasure/arousal/dominance from the character's relationship config.

Hooded Claw's perception of Penelope: pleasure -0.3, arousal 0.8, dominance 0.9. Peter Perfect's perception of Penelope: pleasure 0.8, arousal 0.5, dominance 0.6. When the mansion sleeps, characters now consolidate differently based on who they interacted with and how important those interactions were. And when they wake, the social awareness pipeline has divergent PAD profiles to compare — the observation "Peter seems to enjoy your interactions more than you do" can finally emerge from real data.

## What this opens up

Consolidation is now a game mechanic, not a batch job. Worthy events graduate to the knowledge graph during sleep; unremarkable ones decay. The scoring formula is tunable per game via `ContentScorer.composite()` — a horror-themed mansion would weight arousal higher; a diplomacy scenario would weight action importance even more.

The person nodes are static seeds today. The next step is updating PAD values as interactions accumulate — trust events, dialogue outcomes, conflict resolution should shift each character's perception of the others over time. The seeded values are the starting position; the game writes the rest.
