# Decisions — Cross-Service WorkItem Federation (#95)

## D1: Federation scope — full coordination protocol in casehub-work

**Choice:** Full WS-HumanTask-inspired coordination protocol owned by casehub-work
**Alternatives:**
- Primitives only (CloudEvents, client module, transport SPI) — respects layering but doesn't solve end-to-end; requires Qhorus/CaseHub assembly
- Primitives + projection module — complete CQRS solution but no formal protocol; ad-hoc coordination
**Rationale:** Pre-release platform — design for the RIGHT architecture, not the easiest. The coordination protocol defines the formal contract between Task Parents and Task Processors, enabling clean bidirectional lifecycle coupling. The projection module provides local inbox performance. The client module enables lightweight integration.
**Trade-offs:** Most upfront work. Adds ceremony (protocol versioning, registration endpoints) that may need revision when CaseHub/Qhorus stabilize. But: having the protocol defined NOW means CaseHub and Qhorus integrate against a stable contract rather than evolving ad-hoc.
**Sources:**
- [WS-HumanTask 1.1 spec](https://docs.oasis-open.org/bpel4people/ws-humantask-1.1-spec-cs-01.html) — Task Parent / Task Processor / Task Client separation, coordination protocol
- [Camunda 8.8 Unified Architecture](https://camunda.com/blog/2025/11/whats-different-orchestration-cluster-camunda-88/) — industry moved to single-cluster; federation is custom
- [CQRS pattern (Azure)](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) — single-writer + read projections
- [CloudEvents spec](https://cloudevents.io/) — standard event envelope
- docs/LAYERING.md — casehub-work boundary rule
- GE-20260805-a2aa1b — WorkItemStore SPI co-location with JPA (lightweight client needed)
**Exploration:** deep-analysis
**Status:** captured

## D2: Federation scenario — multi-service mesh with per-interaction asymmetry

**Choice:** Multi-service mesh with per-interaction Task Parent / Task Processor roles. Any service running casehub-work can be both creator and resolver. Protocol is asymmetric per-WorkItem (one owner) but symmetric in capability.
**Alternatives:**
- AI agent → human inbox only — artificially constrains to one-directional; platform architecture naturally produces any-to-any
- Hierarchical delegation — tree structure is a subset of mesh; designing for mesh doesn't prevent tree usage
**Rationale:** Traced from the platform's own consumers: CaseHub engine, Qhorus agents, claudony dashboard, and standalone apps all produce the same pattern — any service creates WorkItems that may need resolution in any other service. Per-interaction asymmetry (single owner per WorkItem) provides clean consistency semantics. System-level symmetry means no service is special.
**Trade-offs:** Mesh is more complex than one-directional. Service discovery matters. But the alternatives would require redesign when the next consumer arrives.
**Sources:**
- [A2A protocol spec](https://a2a-protocol.org/latest/specification/) — "fluid roles" (per-interaction asymmetry, system symmetry), now Linux Foundation standard with 150+ org support
- [WS-HumanTask 1.1](https://docs.oasis-open.org/bpel4people/ws-humantask-1.1-spec-cs-01.html) — Task Parent / Task Processor / Task Client separation
- CaseHub engine A2A module (`engine/a2a/`) — A2AClient, AgentCard already exist
- Qhorus messaging infrastructure — webhook, Kafka, WebSocket, PostgreSQL observers already exist
- docs/NORMATIVE-ALIGNMENT.md — WorkItem lifecycle ↔ Qhorus speech act mapping
- docs/LAYERING.md — ecosystem context (casehub-work below CaseHub, Qhorus, Quarkus-Flow)
**Depends on:** D1
**Exploration:** deep-analysis
**Status:** captured

## D3: Protocol relationship — casehub-work protocol independent, Qhorus as transport, A2A as discovery

**Choice:** casehub-work defines its own WorkItem-specific coordination protocol (CloudEvents-based). Qhorus channels are the primary transport. A2A compatibility is a separate adapter module.
**Alternatives:**
- Protocol IS an A2A profile/extension — A2A's 8 task states are too simple for WorkItem's 15+ lifecycle states; domain-specific operations (claim with OCC, delegate, escalate) don't map to A2A's generic messages
- Direct REST only (no protocol layer) — works for simple cases but no lifecycle coupling, no projection model, no subscription semantics
**Rationale:** casehub-work needs domain-specific protocol messages (claim, delegate, escalate, SLA breach, group completion) that A2A can't express. But A2A provides excellent discovery (Agent Cards) and the engine already has A2A infrastructure. Clean separation: casehub-work defines WHAT (protocol), Qhorus provides HOW (transport), A2A provides WHERE (discovery), CaseHub provides WHY (orchestration).
**Trade-offs:** Multiple layers to understand. But each layer has a clear owner and can evolve independently. A2A interop requires a separate adapter, adding a module.
**Sources:**
- [A2A protocol spec](https://a2a-protocol.org/latest/specification/) — 8 task states vs casehub-work's 15+; generic message model vs domain-specific operations
- [CloudEvents spec](https://cloudevents.io/) — CNCF graduated standard for event envelope
- Qhorus runtime — MessageService, MessageObserverDispatcher, multiple delivery backends
- CaseHub engine — A2AClient, AgentCard, A2AWorkerFunction
- WorkCloudEventAdapter, WorkCloudEventInboundAdapter — CloudEvents infrastructure already exists
**Depends on:** D1, D2
**Exploration:** deep-analysis
**Status:** captured
