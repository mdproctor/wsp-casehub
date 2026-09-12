## D1: Worker-to-CDI wiring pattern

**Choice:** Pass SocCbrRetrieveService as a constructor parameter through the existing descriptor pattern
**Alternatives:**
- Make descriptor a CDI bean — cleaner if more dependencies accumulate, but changes architecture for one worker
- Bypass descriptor — fragments worker registration across two locations
**Rationale:** Descriptor already takes ChatModel as a constructor parameter for LLM workers. Adding SocCbrRetrieveService is the same pattern — no architectural change.
**Trade-offs:** Descriptor constructor grows; if a third service is needed later, CDI conversion becomes worth it.
**Sources:** SocInvestigationCaseDescriptor.java, SocCaseHub.java, RuleIocEnrichmentWorker.java
**Exploration:** quick
**Status:** captured
