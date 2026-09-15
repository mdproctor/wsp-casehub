---
title: "Dependency Drift Archaeology"
date: 2026-09-14
entry_type: note
subtype: diary
author: mdp
projects: [casehub-soc]
tags: [cdi, quarkus, upstream-drift, crowdstrike, containment, wiremock]
publish: false
---

# Dependency Drift Archaeology

Started the session expecting a straightforward fix — issue #34, "upstream API breaks from dependency drift." The compilation fixes had already landed in an earlier session. What remained was getting `@QuarkusTest` to start. Should've been quick.

It wasn't.

## Three layers of CDI wreckage

The first error was `MemoryEmitter` — a CDI bean that upstream neocortex removed entirely, leaving engine's `CaseMemoryObserver` injecting a type that no longer exists. Excluded it. Next: `ScimAgentLookup` and `JwtVCValidator` from platform-identity — produced as `@ApplicationScoped` but with `final` fields and no no-args constructor, making them unproxyable.

Here's the part that cost real time: `quarkus.arc.exclude-types` doesn't work on producer-method outputs. The types are registered by `IdentityBeans` via `@Produces`, not by class scanning. Excluding the produced type is a no-op — you have to exclude the producer class itself. But that cascades: ledger's identity enrichers depend on the identity beans, so those need excluding too.

Three exclusions became six. Each one revealed the next dependency in the chain. Filed issues on engine, parent, and later qhorus when their SNAPSHOTs broke mid-session.

## Then the actual work

With `@QuarkusTest` running again (379 tests green), we moved to #52 — the CrowdStrike Falcon containment connector. This is the first real EDR integration for the containment pipeline. The infrastructure from #50 (HttpContainmentExecutor, ContainmentRequest/Response contract, SimulatedContainmentConnector) made the connector itself straightforward: a JAX-RS endpoint that translates the containment contract to CrowdStrike API calls.

The connector lives in-process — same Quarkus app, new package `io.casehub.soc.connector.crowdstrike`. No sidecar, no separate deployment. The HTTP contract is the integration boundary; extracting to a sidecar later is a mechanical refactor if credential isolation ever matters.

OAuth2 client credentials flow handles CrowdStrike authentication. Token cached in-memory with a 60-second expiry buffer. The CDI producer uses `@Singleton` — not `@ApplicationScoped` — because the OAuth2 client has final fields. Same unproxyable pattern we'd just spent an hour debugging in the identity module.

Twenty tests cover the connector: unit tests for action mapping and parameter validation, WireMock contract tests for the full HTTP round-trip including rate limiting, auth failures, and CrowdStrike API errors.

## The SNAPSHOT rug-pull

Late in the session, Maven refreshed the qhorus SNAPSHOTs. The new versions have their own compilation errors — constructor signature changes in `EvidentialChecker` and `StoredCommitmentAttestationPolicy` that weren't propagated. Thirty CDI deployment errors. Every `@QuarkusTest` in the project instantly broken.

The CrowdStrike code is clean — its own 20 tests pass. But the integration test re-enablement that #52 was supposed to deliver can't land until qhorus is fixed upstream.

## What's left

The queue has #53 (Palo Alto) and #54 (Okta/Azure AD) — mechanically identical to CrowdStrike, different external API. Those can start once qhorus is resolved. The connector pattern is established; the next two are translation work.
