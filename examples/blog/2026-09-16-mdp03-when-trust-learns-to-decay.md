---
layout: post
title: "When Trust Learns to Decay"
date: 2026-09-16
entry_type: note
subtype: diary
projects: [casehubio/examples, casehubio/blocks, casehubio/neocortex]
tags: [trust, ledger, bayesian-beta, consolidation, cognitive-agents]
series: issue-65-trust-evolution
---

# When Trust Learns to Decay

Until today, Penelope Pitstop's view of the Hooded Claw was frozen at scenario start. He could steal from her every turn and she'd keep the same PAD-seeded perception she had at the beginning. The person-entity overlays from Phase B gave characters perspectival awareness — each character sees every other through their own emotional lens — but the lens never adjusted.

The missing piece was trust evolution: gameplay actions should feed back into those overlay nodes, updating trust scores during consolidation so that characters' perceptions change based on experience.

## The ledger insight

I started by assuming we'd build a custom trust computation in wacky-manor — count positive and negative events, apply weights, write to overlay nodes. Claude pointed out that `TrustScoreComputer` already exists in the ledger module with a Bayesian Beta model, temporal decay, and credibility weighting. I'd been looking at it as enterprise audit infrastructure; I hadn't considered using it for game character trust.

The key realisation: `casehub-ledger-memory` provides in-memory stores backed by `ConcurrentHashMap`. Same Bayesian Beta scoring, same `DecayFunction` with verdict-aware asymmetry, zero database overhead. Wacky-manor already had the dependency in its POM — it just wasn't being used for trust.

## Per-relationship from per-actor

`TrustScoreComputer.compute()` is a per-actor model: given all of an actor's decisions and all attestations on those decisions, it produces a single trust score. But characters need per-relationship trust — Penelope trusts the Hooded Claw differently than Muttley does.

The solution is almost embarrassingly simple: filter the attestation map by `attestorId` before passing it to the computer. Same decisions, different attestation views, different scores. The computer doesn't care about attestor identity; it just aggregates whatever attestations you give it. We get per-relationship Bayesian Beta scoring without modifying the computation engine at all.

## Everything belongs in blocks

My initial design put the trust evolution framework in wacky-manor. But as I walked through the components with Claude, each one turned out to be reusable:

The action-to-verdict mapping is a configuration table, not application logic. The consolidation phase that bridges ledger scoring to overlay nodes works for any cognitive agent using the mindmap. The CDI event pattern for recording trust-relevant actions follows the same `fireAsync`/`@ObservesAsync` convention neocortex already uses for extraction events.

The result: wacky-manor writes zero trust-specific Java. A `trust-evolution.yaml` maps STEAL to FLAGGED, GIVE to SOUND, PULL_ASIDE to SOUND — and that's the entire app-specific contribution. The framework classes split across blocks-core (pure computation records), blocks (CDI wiring), and neocortex-mindmap-intelligence (the consolidation phase itself).

## What the design review caught

Claude ran an adversarial spec review — three rounds. The most significant catch: `AgentTrustProvider`, the existing trust SPI, takes a single `agentId` parameter. There's no observer parameter. Per-relationship trust can't flow through it. Rather than evolving the SPI (which would break all existing callers for a concept they don't need), the spec bypasses it entirely — per-relationship trust reads directly from overlay node properties. The global trust SPI keeps serving its existing consumers unchanged.

The review also caught that `CharacterCognition` is a POJO, not a CDI bean — it can't fire CDI events. The orchestrator (which is `@ApplicationScoped`) has to fire the event itself, not delegate to `CharacterCognition.recordTrustEvent()`. That method turned out to be a no-op anyway — computing a trust weight and discarding the return value.

## The Bayesian prior as design

One detail I particularly like: the evidence gate. With no attestation history, the Bayesian Beta prior gives `alpha=1, beta=1, score=0.5`. But 0.5 maps to `TrustLevel.MODERATE` in the observation rendering — which would tell a character "you have mixed feelings about Peter" when they've never interacted with Peter at all. The spec adds `alpha + beta <= 2 → TrustLevel.UNKNOWN` — skip rendering entirely. No evidence is not an opinion.

## What opens up

Phase C has five more issues after trust evolution — drive adaptation, belief revision, needs pyramid, goal prioritisation, relationship stages. Each one now has a template for how cognitive feedback loops wire through the platform: YAML-configured event classification, ledger-backed scoring, consolidation-phase writeback to the mindmap, observation rendering from overlay properties.

The follow-up I'm most interested in is personality-driven trust interpretation. Right now all characters evaluate the same action identically — STEAL is FLAGGED at 0.9 confidence for everyone. But `CognitiveDerivationEngine` already produces per-character `trustFormationRate` and `conflictInterpretation` from Eidos descriptors. Applying those as confidence modifiers during consolidation (not at recording time) keeps ledger records personality-neutral while letting personality shape perception. The architecture is ready for it; the old `ManorTrustEvents` weight model already had this — we just need to bring it forward into the Bayesian framework.
