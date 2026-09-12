## D1: Access control for alert injection endpoint

**Choice:** Role-gated in all profiles — `@RolesAllowed("soc-demo-admin")`
**Alternatives:**
- Dev profile only (`@IfBuildProfile("dev")`) — simplest but unavailable in staged demos and training environments
- Both profile + role — defense-in-depth but over-engineered for a demo tool
**Rationale:** Staged demos and training environments need the endpoint too, not just local dev. Role gating prevents accidental use without restricting deployment topology.
**Trade-offs:** Fake alerts can be injected in production by anyone with the role. Acceptable risk — the role name makes intent clear.
**Sources:** `AlertToCaseIntegrationTest.java` (proven injection pattern), `SiemAlertGanglion.java` (CloudEvent extensions), `ras-situations.yaml` (situation definitions)
**Exploration:** quick
**Status:** captured
