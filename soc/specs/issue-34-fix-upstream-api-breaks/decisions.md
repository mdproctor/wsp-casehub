## D1: CrowdStrike connector deployment model

**Choice:** In-process JAX-RS endpoint within SOC's Quarkus application
**Alternatives:**
- Separate Maven module/sidecar — credential and failure isolation, but adds deployment complexity unnecessary for pre-release
- Direct ContainmentExecutor impl — skips the HTTP contract, breaks the established routing architecture
**Rationale:** Follows the SimulatedContainmentConnector pattern exactly. Zero deployment complexity. The HTTP contract is the integration boundary — extracting to a sidecar later is a mechanical refactor.
**Trade-offs:** SOC's classpath picks up CrowdStrike client dependencies; SOC process needs CrowdStrike API credentials at runtime.
**Sources:** SimulatedContainmentConnector.java:19, HttpContainmentExecutor.java:25, issue-50 spec (Batch 2)
**Exploration:** quick
**Status:** captured

## D2: PAN-OS commit model

**Choice:** Auto-commit per action — each connector call stages the config change and commits synchronously, polling the commit job to completion before returning the ContainmentResponse
**Alternatives:**
- Stage only, external commit — faster per-call but caller can't tell if the change is live; requires separate orchestration
- Configurable per action — maximum flexibility but harder to reason about (mixed live/staged state)
**Rationale:** Containment actions must be confirmed effective. A staged-but-uncommitted change provides no containment. The 5-15s commit time is acceptable — containment is not latency-sensitive, and the caller needs a definitive success/failure signal.
**Trade-offs:** Each action incurs full commit latency. Rapid sequential actions (multiple IPs blocked) each wait for their own commit rather than batching.
**Sources:** PAN-OS XML API documentation, CrowdStrike connector pattern (atomic per-call)
**Exploration:** quick
**Status:** captured

## D3: PAN-OS direct vs Panorama

**Choice:** Target PAN-OS directly (individual firewall), not Panorama
**Alternatives:**
- Panorama only — centralized management, more enterprise-grade, but requires Panorama deployment
- Abstract both — support both via config, but API differences are subtle and hard to abstract
**Rationale:** Simpler API surface, works without Panorama. Most SOC deployments start with individual firewalls. Panorama support is a future config switch (same API structure, different base URL and device-group scoping) — not a redesign.
**Trade-offs:** Multi-firewall environments need individual connector instances per firewall, or later Panorama support.
**Sources:** PAN-OS API reference, issue #53 description ("PAN-OS API or Panorama API")
**Exploration:** quick
**Status:** captured

## D4: XML handling approach

**Choice:** String templates with parameter substitution
**Alternatives:**
- javax.xml DOM — type-safe but verbose for small, fixed-shape payloads
- JAXB — most heavyweight, adds coupling and boilerplate for 3-4 fixed XML shapes
**Rationale:** PAN-OS API payloads are small and well-defined. Templates are cleaner and more readable than DOM or JAXB for this use case. No new dependencies.
**Trade-offs:** No compile-time XML validation. Malformed templates produce runtime errors.
**Sources:** CrowdStrike connector (used Jackson for JSON — analogous simplicity)
**Exploration:** quick
**Status:** captured

## D5: Pre-configured infrastructure model

**Choice:** Assume pre-configured address groups and URL categories — connector adds entries to existing groups, does not create security rules
**Alternatives:**
- Create standalone rules per action — self-contained but creates rule sprawl, harder to manage
- Auto-create groups if missing — most robust but mixes infrastructure provisioning with operational blocking
**Rationale:** Standard firewall operational pattern. The firewall admin pre-configures a `casehub-blocked-ips` address group and `casehub-blocked-domains` URL category, referenced by existing deny rules. Connector just adds entries. Avoids rule sprawl, one-time setup.
**Trade-offs:** Requires one-time firewall configuration before the connector works. Connector will fail if groups don't exist — need clear error messages.
**Sources:** PAN-OS operational best practices
**Exploration:** quick
**Status:** captured

## D6: Identity connector — dual provider abstraction

**Choice:** Abstract both Okta and Microsoft Graph behind a single connector with config-driven provider selection (`casehub.soc.identity.provider=okta` or `graph`)
**Alternatives:**
- Okta only — simpler, but limits enterprise reach; Graph support would need a separate connector later
- Per-request provider — allows multi-tenant routing but adds complexity to every request
**Rationale:** The three containment actions (disable account, revoke sessions, rotate API key) map cleanly between both APIs — the differences are auth mechanism and URL structure, not semantics. A strategy pattern internally dispatches to the configured provider. One provider per deployment.
**Trade-offs:** Two auth implementations (SSWS token for Okta, OAuth2 for Graph). Testing requires WireMock scenarios for both providers. If the APIs diverge in future, the abstraction may leak.
**Sources:** Okta Users API reference, Microsoft Graph API reference, CrowdStrike/Palo Alto connector patterns
**Exploration:** quick
**Status:** captured

## D7: Identity connector — config-driven provider selection

**Choice:** Single config property selects the active provider at startup
**Alternatives:**
- Per-request parameter — more flexible but more complex
- Action-to-provider config map — maximum flexibility but unusual
**Rationale:** Matches how CrowdStrike and Palo Alto connectors work — one deployment, one provider. Simple, deterministic.
**Trade-offs:** Can't mix providers in a single deployment. Multi-IdP environments need multiple connector deployments.
**Depends on:** D6 (dual provider abstraction)
**Sources:** CrowdStrike/Palo Alto config patterns
**Exploration:** quick
**Status:** captured
