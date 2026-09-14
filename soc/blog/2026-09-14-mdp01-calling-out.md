---
title: "Calling Out"
date: 2026-09-14
entry_type: note
subtype: diary
author: mdp
projects: [casehub-soc]
tags: [containment, connectors, http, spi, design-review]
publish: false
---

# Calling Out

The SOC containment pipeline has been executing actions into a void.
`LoggingContainmentExecutor` logs "CONTAINMENT EXECUTED" and returns
success — the audit trail records an isolation that never happened, a
credential revocation that touched nothing. The DORA timeline measures
how fast we can write a log line. Today we built the infrastructure to
call out to real systems.

## The design problem

The ContainmentExecutor SPI from epic #40 is synchronous:
`execute(actionType, params, context) → ContainmentResult`. The CaseHub
workers repo has a full HTTP dispatch infrastructure — endpoint resolution,
retry, circuit-breaking — but its `WorkerExecutionManager.submit()` is
fire-and-forget async with callback-based completion. The two models are
incompatible.

The initial design proposed injecting `HttpEndpointResolver` from the
workers-http module to reuse its config system. Claude's post-spec review
caught the problem: `HttpEndpointResolver.initialize()` is package-private,
called by `HttpWorkerRuntime` during a lifecycle boot sequence that SOC
doesn't participate in. Injecting the resolver from SOC gives you a bean
with an empty endpoint map — CDI resolves it without complaint, and the
failure is silent at runtime.

The fix was simple: SOC owns its own `ContainmentEndpointResolver` — thirty
lines of config parsing, no cross-module lifecycle coupling. The workers repo
pattern was the reference, not the dependency.

## Two-layer routing

The config design uses two layers. Layer 1 maps action types to endpoint
tags (`casehub.soc.containment.routing.isolate-host=crowdstrike`). Layer 2
maps tags to URLs (`casehub.soc.containment.endpoints.crowdstrike.url=...`).
Different deployments can swap integrations per action type without touching
code — one SOC uses CrowdStrike for host isolation, another uses SentinelOne,
same binary.

Action types that have no routing entry fall through to the logging behaviour.
This means connectors can be adopted incrementally: configure what you have,
the rest keeps working as before.

## What's in the box

`HttpContainmentExecutor` displaces `LoggingContainmentExecutor` via CDI
displacement — it's `@ApplicationScoped`, the logging executor is `@DefaultBean`.
When a containment action fires, the executor resolves the endpoint tag, POSTs a
`ContainmentRequest` to the connector, and maps the `ContainmentResponse` back to
a `ContainmentResult`. Seven failure modes are handled: success, 429, 4xx, 5xx,
connection timeout, malformed response, unresolved endpoint tag.

A `SimulatedContainmentConnector` JAX-RS endpoint at `/sim/containment` implements
the same API contract. All action types route to it in dev/test config. It returns
realistic responses with simulated latency and external IDs — enough to exercise
the full pipeline without a CrowdStrike account.

We also added a retryable failure path to `RuleContainmentExecutionWorker`. The
worker previously mapped all executor results unconditionally as successes.
Now when `ContainmentResult.success()` is false, the worker records
`executed=false` with the error reason, letting the SLA breach policy
handle escalation.

## What's next

The integration test is written but disabled — a pre-existing `MemoryEmitter`
CDI issue (#34) breaks all `@QuarkusTest` tests in the module. Once that's
resolved, the full pipeline verification (alert → triage → gate → HTTP
containment → ledger chain) can run.

Three real connector batches follow: CrowdStrike Falcon for host isolation,
Palo Alto for network segmentation, Okta for credential revocation. Each is a
separate sidecar service implementing the containment API contract. The architecture
is proven; what remains is the external API integration work — and the question of
whether to build those connectors in this repo or extract them.
