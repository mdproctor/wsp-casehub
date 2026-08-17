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
