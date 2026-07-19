# ChannelBackend Parallel COMMAND Routing

> **Issue:** openclaw#70
> **Repos:** casehub-qhorus (API change), casehub-engine (SPI + dispatch), casehub-openclaw (routing impl)
> **Date:** 2026-07-19

## Problem

When multiple agents are active simultaneously on the same case, COMMANDs
dispatched through the channel backend reach an arbitrary agent instead of the
intended one.

The routing signal exists in the data model — `MessageDispatch` and `Message`
both carry a `target` field — but it is dropped at two boundaries:

1. `CaseChannelProvider.postToChannel()` has no `target` parameter, so the
   engine cannot set it.
2. `OutboundMessage` has no `target` field, so `ChannelBackend.post()` is blind
   to routing intent even if `target` were set on the dispatch.

## Root Cause

A three-layer information gap between the engine (which knows the target) and
the backend (which needs it):

```
Engine (knows worker.name() = agentId)
  → CaseChannelProvider.postToChannel() — no target param
    → MessageDispatch.target — never populated
      → OutboundMessage — field absent
        → ChannelBackend.post() — routes arbitrarily
```

## Design

Fix the propagation gap. No new state, no new mappings, no new conventions.

### Layer 1: OutboundMessage — add `target` field

`OutboundMessage` (casehub-qhorus-api) gains a nullable `String target` as the
last record component:

```java
public record OutboundMessage(
        UUID messageId,
        String sender,
        MessageType type,
        String content,
        String correlationId,
        Long inReplyTo,
        ActorType senderActorType,
        List<ArtefactRef> artefactRefs,
        String target) {}
```

Four production construction sites pass `target` through:

| Site | File | Source |
|------|------|--------|
| Normal dispatch | `MessageService.dispatch()` | `dispatch.target()` |
| LAST_WRITE dispatch | `MessageService.dispatch()` | `dispatch.target()` |
| Remote delivery | `ChannelGateway.deliverRemote()` | `msg.target()` |
| Cursor-based delivery | `DeliveryBatchExecutor.toOutbound()` | `m.target()` |

Test construction sites (~30) pass `null` — target is irrelevant in those tests.

### Layer 2: CaseChannelProvider — add `target` parameter

The engine-api SPI gains a 7th parameter:

```java
void postToChannel(
    CaseChannel channel, String from, String content,
    MessageType type, String correlationId, String deadline,
    String target);
```

The existing 6-param signature becomes a default method delegating to the
7-param version with `target = null`. The existing 3-param convenience chains
through unchanged. Backward-compatible — existing implementations (Claudony,
no-ops, tests) override the 6-param method and inherit the delegation.

`ReactiveCaseChannelProvider` gets the same treatment.

### Layer 3: Engine dispatch — set target

`WorkerScheduleEventHandler.dispatchCommand()` passes
`target = worker.name()` (which equals the agentId by existing convention):

```java
caseChannelProvider.postToChannel(
    channel, "casehub-engine:orchestrator", serialize(command),
    MessageType.COMMAND, String.valueOf(eventLogId), deadline,
    worker.name());
```

### Layer 4: OpenClaw — route by target

**`OpenClawCaseChannelProvider.postToChannel()`** overrides the 7-param method
and sets `.target(target)` on the `MessageDispatch`.

**`OpenClawChannelBackend.post()`** reads `message.target()` for direct agent
lookup:

- `target` non-blank: validate agent is registered for this caseId via
  `registry.findCaseId(agentId)`. If not registered, WARN and ignore.
- `target` null/blank: fall back to `registry.findAgentId(caseId)` — pre-#70
  single-agent compat path.

The validation step prevents routing a COMMAND to an agent registered for a
different case if the target is stale.

## Convention Dependency

This design relies on `workerName == agentId` — case YAML worker names match
OpenClaw agent config identifiers. This convention is already enforced by
`WorkerProvisioner.terminate(workerId)` which calls
`registry.deregister(workerId)` keyed by agentId.

If the convention ever breaks, the fix is adding `workerId` to
`ProvisionResult` so the provisioner explicitly returns the resolved identity.
The engine would then track `provisionedWorkerId` per worker assignment and use
it as `target` instead of `worker.name()`.

## Alternatives Considered

### correlationId→agentId mapping

Provisioner or engine stores a per-COMMAND mapping. Rejected: the engine
generates correlationId (eventLogId) at dispatch time, after provisioning. Extra
hot-path state for every COMMAND. What about COMMANDs without correlationId?

### Per-agent channels

Each agent gets its own channel. Rejected: multiplies channels per case,
breaks the shared-channel coordination model where agents see each other's work,
major change to the normative 3-channel layout.

### Channel-name routing (openclaw-only)

Backend extracts worker name from channel name (`case-{uuid}/worker:{name}`).
Rejected: relies on naming convention, routing signal is implicit rather than
explicit, other backends can't use the mechanism, naming changes break routing
silently.

## Scope

| Repo | Changes |
|------|---------|
| casehub-qhorus | `OutboundMessage` + 4 production sites + ~30 test sites |
| casehub-engine-api | `CaseChannelProvider` + `ReactiveCaseChannelProvider` (backward-compat defaults) |
| casehub-engine | `WorkerScheduleEventHandler.dispatchCommand()` |
| casehub-openclaw | `OpenClawCaseChannelProvider` + `OpenClawChannelBackend` + reactive mirrors + tests |

## Testing

- **qhorus:** `OutboundMessage` carries target through fanOut, delivery batch, and remote delivery paths
- **engine:** `WorkerScheduleEventHandler` test verifies `postToChannel` receives `worker.name()` as target
- **openclaw:** `OpenClawChannelBackend` routes to correct agent when target is set, falls back when null, validates agent-case association, handles unknown target gracefully
