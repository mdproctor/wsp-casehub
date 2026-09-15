---
layout: post
title: "Three connectors, one contract"
date: 2026-09-14
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [containment, connectors, crowdstrike, paloalto, okta, graph, quarkus]
series: issue-34-fix-upstream-api-breaks
---

The previous session fixed the upstream CDI drift and built the first real connector — CrowdStrike Falcon for host isolation. This session took the same contract and stamped it across two more external APIs: Palo Alto PAN-OS for network containment and an Okta/Microsoft Graph dual-provider connector for identity operations.

The interesting part wasn't the replication — it was how each API forced the pattern to bend.

## Where the pattern held

All three connectors follow the same shape: JAX-RS endpoint receives a `ContainmentRequest`, translates it to vendor-specific API calls, returns a `ContainmentResponse`. Each has an auth client, a health check, and a CDI config producer. The `HttpContainmentExecutor` routes by action type — it doesn't know or care which vendor handles the call. From the SOC engine's perspective, `block.ip` and `isolate.host` go through the same pipeline. The connector is an implementation detail behind a config property.

## Where it bent

**PAN-OS has a two-phase commit.** CrowdStrike's API is fire-and-forget — POST, get a response, done. PAN-OS stages changes in a candidate configuration, then you commit. The commit is async — it returns a job ID and you poll until it finishes. That meant the Palo Alto connector needed a commit-polling loop with configurable interval and timeout, and the endpoint timeout had to be set higher than the commit timeout to avoid the executor killing the HTTP call while the firewall is still committing. A subtle ordering constraint that would silently produce false "unreachable" errors if violated.

The other PAN-OS difference: XML, not JSON. We used string templates rather than a proper XML library — the payloads are small and fixed-shape, and javax.xml DOM would have been verbose overkill for four XPath patterns. Regex-based response parsing, which felt dirty until I counted: exactly six patterns, all simple, all tested.

**The identity connector needed a strategy pattern.** CrowdStrike and Palo Alto each target one vendor. The identity connector needed to support both Okta and Microsoft Graph because real deployments are split roughly evenly. A config property (`casehub.soc.identity.provider=okta` or `graph`) selects the active provider at startup. Internally, an `IdentityProvider` interface with three methods — `disableAccount`, `revokeSessions`, `rotateApiKey` — dispatches to either `OktaIdentityProvider` (SSWS token auth, dead simple) or `GraphIdentityProvider` (OAuth2 client credentials with token caching, same pattern as CrowdStrike).

The action semantics mapped cleanly — both APIs have direct equivalents for account disable, session revocation, and credential rotation. The abstraction holds because the differences are in auth mechanism and URL structure, not in what the operations mean.

## What bit us

Two things surfaced during the code review that were present in all three connectors from the start.

First: every HTTP call was creating `WebClient.create(Vertx.vertx())` inline. Each `Vertx.vertx()` spins up a full event loop group — threads, file resolver, timers — and none of them ever get closed. In a test it's invisible. In production, you'd watch the thread count climb until the process runs out of file descriptors. The fix: inject the Quarkus-managed `Vertx` instance through the CDI producers and create one `WebClient` per connector lifecycle.

Second: SmallRye Config's string converter treats empty string as null. We had `@ConfigProperty(name = "casehub.soc.crowdstrike.client-id", defaultValue = "")` on CDI producer method parameters — reasonable-looking default for optional credentials. It works on field injection. It fails on producer parameters because the converter null-coerces the empty string before the default substitution logic runs, and Quarkus deployment validation rejects non-Optional parameters with null values. The fix is `Optional<String>` for all credential properties in producer methods. The error message names the config property but not the producer, so debugging requires tracing from config key to the producer that references it.

## The contract as a design tool

Nine action types, three vendors, one `ContainmentResponse` record. The contract is doing the work the architecture promised — each connector can be built, tested, and deployed independently. A fourth connector (Zscaler, Cortex XDR, whatever comes next) would follow the same pattern in a few hours. The connector itself is the simple part; the external API's authentication model and commit semantics are where the real design decisions live.
