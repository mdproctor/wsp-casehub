---
layout: post
title: "The Gaps Nobody Sees"
date: 2026-11-22
entry_type: note
subtype: diary
projects: [casehubio/neocortex, casehubio/eidos]
tags: [infrastructure, deserialization, cache-invalidation, consistency, cognitive-pipeline, testing]
series: cognitive-pipeline
---

# The Gaps Nobody Sees

*Continues from [Forty-Three Proofs That Personality Is Structural](2026-11-20-mdp01-thirty-six-proofs-that-personality-is-structural.md).*

The divergence tests proved every cognitive pipeline capability produces personality-dependent output. Forty-three tests, forty-eight capabilities, all green. But the pipeline can be correct at every stage and still produce wrong behaviour if the personality data never arrives, if the cache serves stale data, or if the orchestrators can't agree on how to identify an agent.

Seven XS issues. All invisible until you look. None would crash anything. All would silently produce subtly wrong behaviour — the kind where an agent responds with the wrong personality and nobody notices because the agent still responds.

## The facets that vanished

`AmpdFacet` is the PID-5 personality facet model: anxiousness, hostility, emotional lability, and others, each scored 0–100. The Java API works. The builder works. The comparator works. 512 eidos tests pass. But when a character's YAML descriptor includes `ampdFacets`, the `DispositionDeserializer` silently drops them — returns an empty list.

The deserializer handles every other field in `AgentDisposition`: the five disposition axes, delegation, disposition profiles, style profiles, MBTI type, enneagram type, situational profiles. Each has its own parsing block. `ampdFacets` was added in a later PR and the deserializer was never updated. The Java API accepted facets from code, the YAML path silently discarded them, and no test bridged the gap.

The fix is eight lines — parse the `ampdFacets` array, construct `AmpdFacet` records, pass them to the builder. The score validation in `AmpdFacet`'s constructor (`0–100`) propagates naturally through the deserializer. The interesting part isn't the fix. It's that the Java API and the YAML path could diverge without any symptom, test failure, or runtime error. The agent just has no personality facets. Its cognitive defaults derive from axes and profiles alone, missing the facet-level modulation that CAPS settling, Theory of Mind, and emotion regulation all depend on.

## The cache that never forgot

`EffectiveCognitiveDefaults` is the central cache for cognitive profile derivation. It takes a personality descriptor, derives hundreds of cognitive parameters, and caches the result. When the agent's facets shift through situational personality or lived experience, a composition version counter invalidates the cache and forces re-derivation.

But there's another mutation path. When an agent's eidos descriptor is updated — the YAML is reloaded, the disposition changes, the archetype shifts — the registry fires an `AgentDescriptorUpdated` CDI event. Nothing was listening. The cache kept serving the old cognitive defaults. The descriptor changed, the personality changed, and the pipeline kept running on stale data.

The fix is a CDI observer in `cognitive-eidos` that calls `invalidateBase()` on the affected agent's slot. The observer is four lines of wiring. The `invalidateBase()` method is two lines wrapping the existing `incrementBaseVersion()`. The gap existed because `EffectiveCognitiveDefaults` was designed for tick-driven invalidation (composition version changes on personality state transition) but not for descriptor-driven invalidation (the base personality definition itself changing). Both paths exist. Only one was wired.

## The keys that didn't match

Eight orchestrators in the cognition module maintain per-agent state in `ConcurrentHashMap` instances. Each needs a composite key from agent ID and tenant ID. Six of them use `agentId + ":" + tenantId`. `GoalProposalOrchestrator` uses `agentId + "|" + tenantId`. `InnerLifeOrchestrator` uses `tenancyId + ":" + agentId` — same separator, reversed order.

This works. Each orchestrator creates and reads its own keys. The maps are private. No cross-orchestrator lookup exists today. But `evictIdle()` — the method that cleans up cached state for agents that haven't been active — is being wired across orchestrators as a periodic maintenance operation. A generic eviction helper that iterates all orchestrator maps would need per-class key translation. More subtly: `InnerLifeOrchestrator` has both `agentKey(descriptor)` (which returns `tenancyId:agentId`) and `reportDropped(agentId, tenantId)` which constructs `tenantId + ":" + agentId`. If `tenancyId` and `tenantId` ever differ — and they can, through the descriptor's own `tenancyId()` accessor — the report would miss the state entry entirely.

The fix is mechanical: `|` to `:` in eight places, reversed field order in two. All thirteen `InnerLifeOrchestratorTest` assertions still pass.

## Why these gaps matter

A cognitive agent architecture that crashes on bad input is easy to debug. One that silently degrades is not. When personality facets disappear, the agent still talks. When cognitive defaults go stale, the agent still responds. When orchestrator keys don't match, nothing fails because nothing tries to match them.

The pattern across all three: a capability was added or modified in one layer without updating the adjacent layer. The YAML deserializer didn't track the API. The cache invalidation didn't track the mutation paths. The key format didn't track the convention. Each worked in isolation. Each failed as a system property.

Infrastructure testing for cognitive architectures is not about whether the pipeline produces correct output — the divergence tests cover that. It's about whether the data the pipeline receives is the data the author intended. The most dangerous bug is the one where the system works perfectly on the wrong input.
