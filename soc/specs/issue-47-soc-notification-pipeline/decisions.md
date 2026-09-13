## D1: SOC notification event types

**Choice:** Eight event types — four SOC-specific (bridged by SocNotificationBridge) + four SYSTEM subscriptions on existing platform events (already bridged by WorkItemSubscriptionBridge)
**Event type enumeration:**
- SOC-specific (registered and dispatched by SocNotificationBridge):
  1. `io.casehub.soc.incident.created` — new incident enters TRIAGING (previousStatus is null)
  2. `io.casehub.soc.incident.escalated` — incident reaches terminal ESCALATED state
  3. `io.casehub.soc.incident.resolved` — incident reaches terminal RESOLVED state
  4. `io.casehub.soc.incident.sla_breached` — SLA breach for SOC incident work item
- Platform events (bridged by WorkItemSubscriptionBridge, SOC seeds SYSTEM subscriptions only):
  5. `io.casehub.work.workitem.created` — containment approval request created (notifies SOC manager before claim)
  6. `io.casehub.work.workitem.assigned` — analyst assigned to containment task
  7. `io.casehub.work.workitem.escalated` — work item escalated via SLA breach decision
  8. `io.casehub.work.workitem.completed` — containment task completed
**Alternatives:**
- Six flat event types, all SOC-specific bridges — redundant with platform bridges, risk of double notifications
- Full incident status change notifications (every transition) — alert fatigue from rapid automated transitions
**Rationale:** Split status changes into significant transitions only (created, escalated, resolved). Reuse platform bridges for work item events to avoid duplication. `workitem.created` required for containment approval requests — SOC manager needs out-of-band notification (Slack/email/webhook) at creation time, not just the qhorus PROPOSE message visible only in web UI. SYSTEM subscription on `workitem.created` filtered for containment approval work items avoids duplicate bridge work. SLA breach handled by SOC bridge because SlaBreachEvent is not currently bridged by the platform and notification templates/targeting are SOC-specific. FALSE_POSITIVE terminal state not separately notified — treated as implicit resolution.
**Trade-offs:** No pre-breach SLA warning (requires timer mechanism, deferred). No intermediate status change notifications (analysts must check dashboard for TRIAGING→INVESTIGATING transitions).
**Sources:** EventTypeRegistry, EventTypeDescriptor, WorkItemSubscriptionBridge, SocIncidentStatusChangedEvent, SlaBreachEvent, SocActionType.candidateGroups()
**Exploration:** quick
**Status:** revised (R1-01: corrected phantom sources. R1-06: enumerated all event types with namespace. R2-01: added workitem.created for containment approval — 8 total)

## D2: Subscription scope for defaults

**Choice:** SYSTEM scope for all default subscriptions
**Alternatives:**
- USER scope (bootstrapped per-user on first login) — allows individual deletion, creates compliance gaps
- Mixed (SYSTEM for compliance-critical, USER for operational) — inconsistent UX, marginal benefit
**Rationale:** DORA Art. 17, SOC2 CC7.3 require demonstrable notification chains for critical incidents. SYSTEM scope prevents silent deletion while users retain mute/snooze escape valves. Consistent scope avoids confusion.
**Trade-offs:** Admin overhead to manage system subscriptions. Users cannot fully opt out — only mute/snooze.
**Matching mechanism:** `SubscriptionStore.findAllEnabled()` returns all enabled subscriptions regardless of scope. The platform notification matching pipeline uses this to evaluate both SYSTEM and USER subscriptions against each event — they are naturally additive with no custom matching logic required.
**Sources:** DORA Article 17, SOC2 CC7.3, SubscriptionStore.findAllEnabled(), SubscriptionScope enum (USER, SYSTEM)
**Exploration:** deep-analysis
**Status:** revised (R1-12: clarified OR-disjunction mechanism — matching uses findAllEnabled(), not scope-filtered queries)

## D3: Subscription seeding mechanism

**Choice:** CDI @Startup bean with idempotent upsert via content comparison
**Alternatives:**
- Flyway SQL migration — disconnected from event POJO field names, template patterns rot silently on refactoring
- Java Flyway migration — Quarkus classpath challenges, over-engineered for this use case
- Template version field — does not exist in platform API (Subscription, SubscriptionInput, SubscriptionUpdate have no version field)
**Rationale:** Template patterns reference event POJO field names. Keeping templates in Java alongside the event records means refactoring catches breakage. Idempotency via lookup: `SubscriptionStore.find(SubscriptionQuery(ownerId="SYSTEM", tenancyId, scope=SYSTEM))`, filter by eventType in results, then compare template fields (titlePattern, bodyPattern, severity, category, actionUrlPattern). If no match exists, create via `SubscriptionStore.store()`. If match exists but template differs, update via `SubscriptionStore.update()`. SYSTEM subscriptions are not user-editable, so overwriting is always safe.
**Trade-offs:** Runs every startup (fast — one query per tenant, field comparison per event type). No external version tracking needed. Content comparison is slightly more expensive than a version check but avoids assumptions about nonexistent API fields.
**Sources:** NoOpEventTypeRegistry (@DefaultBean displacement pattern), EventTypeRegistry.register(), SubscriptionStore.find(SubscriptionQuery), SubscriptionStore.update(), SubscriptionInput, SubscriptionUpdate, NotificationTemplate
**Exploration:** deep-analysis
**Status:** revised (R1-02: replaced templateVersion-based idempotency with content comparison — templateVersion field does not exist in platform API)

## D4: Default target resolution strategy

**Choice:** Mixed targeting — EVENT_FIELD for role-variable events, GROUP for fixed-audience events, ENTITY_WATCHERS for incident-scoped status changes
**Alternatives:**
- Hardcoded GROUP targets for all events — breaks when SocActionType candidateGroups vary (WIPE_ENDPOINT needs ciso+soc-manager, NETWORK_SEGMENTATION needs soc-manager+network-ops)
- EVENT_FIELD only — requires all events to carry target fields, unnecessary for fixed-audience events like incident.created
- ENTITY_WATCHERS only — requires watcher infrastructure on all entities, inappropriate for events with no entity context (SLA breach)
**Rationale:** Containment approval and work item assignment targets vary by action type and case — EVENT_FIELD uses runtime values from the enriched notification event. Incident created goes to fixed analyst groups (tier1-analyst) — GROUP is appropriate. Incident escalated/resolved target users who have interacted with the incident — ENTITY_WATCHERS delivers to case watchers without hardcoding recipients. SLA breach targets the assigned group — EVENT_FIELD resolves from the work item context.
**Event POJO enrichment:** The bridge creates notification-specific event records implementing SubscribableEvent (type() + tenancyId()). SocIncidentStatusChangedEvent is a lean domain event (caseId, tenancyId, previousStatus, newStatus, occurredAt) — it does not carry assignedAnalystId or candidateGroups. The bridge enriches by loading case context at dispatch time. CandidateSetStrategy.evaluate(CandidateSetContext) returns Set<String> which must be joined to a comma-separated string for EVENT_FIELD resolution, following WorkItemLifecycleEvent's pattern where candidateGroups is a pre-resolved String field.
**Trade-offs:** Bridge has case context lookup responsibility. Three targeting strategies add complexity but each is the right tool for its event type. ENTITY_WATCHERS requires the platform watcher mechanism to be active for SOC incidents.
**Sources:** TargetType enum (USER, GROUP, EVENT_FIELD, ENTITY_WATCHERS), NotificationTarget record, SocActionType.candidateGroups(), CandidateSetStrategy interface, WorkItemLifecycleEvent (candidateGroups as String), SocIncidentStatusChangedEvent
**Exploration:** quick
**Status:** revised (R1-03: acknowledged event POJO enrichment requirement — bridge creates SubscribableEvent wrapper. R1-08: added ENTITY_WATCHERS for incident status change events)

## D5: Bridge code structure

**Choice:** Single `SocNotificationBridge` class handling all four SOC-specific event types
**Alternatives:**
- One bridge class per event type (SocIncidentCreatedBridge, SocIncidentEscalatedBridge, etc.) — four files for what amounts to a switch on status transition
**Rationale:** The four SOC events come from two CDI sources: SocIncidentStatusChangedEvent (covers incident created/escalated/resolved, fired via `fireAsync()` by SocIncidentStatusObserver) and SlaBreachEvent (covers SLA breached, fired synchronously by casehub-work runtime within the SLA timer transaction). A single bridge with three methods keeps all notification logic co-located: onStartup(@Observes StartupEvent) for type registration, onIncidentStatusChanged(@ObservesAsync SocIncidentStatusChangedEvent) for async-fired incident events, and onSlaBreach(@Observes(during=AFTER_SUCCESS) SlaBreachEvent) for synchronous SLA breach events. This follows the WorkItemSubscriptionBridge pattern — a single @ApplicationScoped class handling 24 event types with two methods.
**Observer annotations:** `@ObservesAsync` is mandatory for `SocIncidentStatusChangedEvent` because `SocIncidentStatusObserver` fires it via `Event.fireAsync()` — `@Observes` would silently never trigger (per protocol engine-worker-event-observer-async). `@Observes(during=AFTER_SUCCESS)` for `SlaBreachEvent` matches `WorkItemSubscriptionBridge`'s pattern for synchronous transactional events.
**Trade-offs:** Larger single class. Test coverage requires testing all paths in one class rather than isolated per-event tests.
**Sources:** SocIncidentStatusChangedEvent, SlaBreachEvent, WorkItemSubscriptionBridge (single-class notification bridge precedent), EventTypeRegistry, DataSourceRegistry, SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH
**Exploration:** quick
**Status:** revised (R1-04: corrected precedent from SocContainmentCommitmentBridge to WorkItemSubscriptionBridge. R1-05: corrected CDI source from SlaBreachPolicy to SlaBreachEvent. R2-02: specified correct observer annotations — @ObservesAsync for fireAsync-dispatched events, @Observes(during=AFTER_SUCCESS) for synchronous transactional events)

## D6: Filter strategy for subscription matching

**Choice:** Declare filterable fields via EventFieldDescriptor; SYSTEM subscriptions use no filters (catch-all for compliance); USER subscriptions may add severity/status/tactic filters
**Alternatives:**
- No filter support — all subscribers receive all events regardless of severity or tactic; users must rely on mute/snooze
- Hardcoded filter presets — predefined filter combinations (P1-only, my-assignments-only); less flexible, more discoverable
**Rationale:** Issue #47 DoD requires "SOC analysts can subscribe to incident notifications by severity, tactic, or assignment." The platform supports List<ExpressionEvaluator> filters on SubscriptionInput. The bridge declares filterable fields when registering event types via EventTypeDescriptor: severity, incidentStatus, tactic, assigneeId, candidateGroups. Users create subscriptions with filter expressions (e.g., severity == 'P1'). SYSTEM subscriptions remain unfiltered for compliance — every significant incident generates a SYSTEM notification regardless of individual preferences.
**Trade-offs:** Filter expression authoring requires UI support in blocks-notification-inbox. Complex filters may require MVEL/JQ expression builder UI. Filter fields must be present on the enriched notification event POJO (see D4 enrichment).
**Sources:** EventFieldDescriptor, ExpressionEvaluator, SubscriptionInput.filters(), issue #47 Definition of Done
**Exploration:** implicit (surfaced by reviewer R1-07)
**Status:** captured

## D7: Multi-tenant seeding lifecycle

**Choice:** Startup seeding for all known tenants + tenant creation observer for new tenants
**Alternatives:**
- Startup-only seeding — new tenants created after startup have no SYSTEM subscriptions until next restart; compliance gap
- Lazy seeding on first event per tenant — first event for a new tenant may be lost or delayed if subscription doesn't exist yet; race condition
- Global "all tenants" scope — does not exist in current platform API (SubscriptionInput requires non-null tenancyId via Objects.requireNonNull)
**Rationale:** SubscriptionStore.store(SubscriptionInput) requires tenancyId (non-null). SYSTEM subscriptions must exist per-tenant. The seeding bean runs at @Startup, iterating all known tenants from the tenancy registry. A CDI observer on tenant creation events seeds SYSTEM subscriptions for new tenants immediately, closing the lifecycle gap.
**Trade-offs:** Requires a tenancy enumeration mechanism at startup. Tenant creation observer adds a runtime dependency on tenant lifecycle events. If the tenancy registry is unavailable at startup, seeding fails — needs graceful degradation.
**Sources:** SubscriptionInput (requireNonNull on tenancyId), SubscriptionStore.store(), SubscriptionQuery (requireNonNull on tenancyId)
**Exploration:** implicit (surfaced by reviewer R1-09)
**Status:** captured

## D8: Dispatch mechanism

**Choice:** Push events via DataSourceRegistry.resolveSource(NOTIFICATION_DATASOURCE_PATH, tenancyId) → ds.add(event)
**Alternatives:**
- Direct SubscriptionStore query + manual notification creation — bypasses platform matching pipeline, duplicates matching/template/delivery logic
- CDI event only (no DataSource push) — notification pipeline never sees the event; no subscription matching occurs
**Rationale:** This is the platform's standard notification dispatch contract, as implemented by WorkItemSubscriptionBridge. Events pushed to the notification DataSource are matched against all enabled subscriptions by the platform notification runtime (casehub-platform-notifications). The bridge creates a SubscribableEvent-implementing POJO with type() and tenancyId(), then pushes it. The platform handles subscription matching, template rendering, and delivery. Uses Instance<DataSourceRegistry> with isUnsatisfied() guard for graceful degradation when the notifications module is absent.
**Trade-offs:** Depends on casehub-platform-notifications being on the classpath. Without it, notifications silently degrade (logged warning, no crash).
**Sources:** DataSourceRegistry.resolveSource(Path, String), SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH, WorkItemSubscriptionBridge.onWorkItemEvent(), SubscribableEvent interface (type(), tenancyId())
**Exploration:** implicit (surfaced by reviewer R1-10)
**Status:** captured

## D9: Event type naming convention

**Choice:** Follow platform convention: io.casehub.soc.incident.<eventtype>
**Alternatives:**
- Flat names (incident_created, sla_breached) — no namespace, collision risk across apps
- Reverse domain only (io.casehub.soc.created) — loses entity context (incident vs case vs alert)
**Rationale:** WorkItemSubscriptionBridge registers event types as io.casehub.work.workitem.<eventtype> (e.g., io.casehub.work.workitem.created). SOC follows the same pattern: io.casehub.soc.incident.created, io.casehub.soc.incident.escalated, io.casehub.soc.incident.resolved, io.casehub.soc.incident.sla_breached. This convention is consistent across the platform, supports UI display grouping in blocks-notification-inbox, and avoids cross-app collisions.
**Trade-offs:** Longer event type strings. Convention must be documented and enforced manually (no compile-time check).
**Sources:** WorkItemSubscriptionBridge event type registration pattern, EventTypeDescriptor.eventType()
**Exploration:** implicit (surfaced by reviewer R1-11)
**Status:** captured
