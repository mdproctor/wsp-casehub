# Channel Taxonomy Formalisation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `docs/CHANNELS.md` as a complete design reference and fix PLATFORM.md / qhorus deep-dive inconsistencies.

**Architecture:** Documentation-only. No code changes. The rewritten CHANNELS.md has 7 sections covering speech act foundation, 5 purpose categories (12 channel patterns total), 8 discriminator dimensions, purpose × semantic matrix, FIPA cross-reference, layered protocol stack, and academic lineage. Three companion docs get targeted line fixes.

**Tech Stack:** Markdown only.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-06-25-channel-taxonomy-design.md` — the authoritative source for all content
- Project repo: `/Users/mdproctor/claude/casehub/parent`
- All code facts verified against IntelliJ MCP during spec review — do not re-verify
- Commits reference #293
- CHANNELS.md target: complete rewrite (overwrite), not incremental edits

---

### Task 1: Rewrite docs/CHANNELS.md

**Files:**
- Overwrite: `docs/CHANNELS.md`

**Interfaces:**
- Consumes: spec at `docs/superpowers/specs/2026-06-25-channel-taxonomy-design.md`
- Produces: authoritative channel taxonomy document (referenced by PLATFORM.md, qhorus deep-dive, future channel implementors)

- [ ] **Step 1: Write the complete CHANNELS.md**

Overwrite `docs/CHANNELS.md` with the following content. This is the complete file — no sections are omitted.

```markdown
# CaseHub Channel Taxonomy

Channels are the communication primitive of the CaseHub platform. Every agent interaction,
human decision, domain event, and infrastructure signal flows through a Qhorus channel.
This document is the authoritative reference for the channel taxonomy — naming, classifying,
and providing design guidance for every channel pattern in the platform.

**Audiences:** Platform developers extending the channel system. Third-party application
builders choosing which channel patterns to use.

---

## 1. The Layered Protocol Stack

CaseHub separates agent communication concerns into three architectural layers.
Each layer depends only downward — no upward or lateral knowledge.

| Layer | Component | Owns |
|-------|-----------|------|
| **Transport + Semantics** | Qhorus | `MessageType` (speech acts), `ChannelSemantic` (data flow), `allowedTypes`/`deniedTypes` enforcement, `ChannelBackend` SPI, commitment lifecycle |
| **Orchestration + Layout** | Engine | `CaseChannelLayout` (topology declaration), `MeshParticipationStrategy` (engagement level), `CaseChannelProvider` (backend-agnostic operations), `QhorusMessageSignalBridge` (signal routing) |
| **Application + Domain** | Drafthouse, future apps | `DebateChannelBackend`, `ReviewerChannelBackend`, domain-specific projections, entry type systems |

**Where new channel patterns plug in:**

- New **coordination** pattern (e.g., consensus gate) → engine layer, uses Qhorus APPEND semantic
  (threshold logic in the backend, not the transport layer)
- New **deliberation** pattern (e.g., negotiation) → application layer as a `ChannelBackend`,
  follows the drafthouse structural pattern (backend + projection + state type)
- New **notification** adapter (e.g., Discord) → `connector-backend` submodule, implementing
  `HumanParticipatingChannelBackend` (not core Qhorus — the SPI is in qhorus-api, but the
  adapter plugs into the separately-deployable connector-backend module)

Sander et al. (2026) predict that the agent protocol ecosystem will converge toward
federated, layered protocol stacks rather than monolithic standards. CaseHub's internal
architecture demonstrates this principle at the intra-platform level.

---

## 2. Speech Act Foundation

Qhorus `MessageType` is a deliberate reduction of FIPA ACL's 22 communicative acts
(IEEE SC00037J, 2002) to 9 speech acts. The design rationale: concerns that FIPA
handled as speech acts belong in different layers of CaseHub's stack — infrastructure
(propagate), engine orchestration (cancel, request-when, subscribe), or error handling
(not-understood). The speech act layer is left focused on obligation-carrying discourse.

### 22→9 Mapping

**Merged into qhorus (14 FIPA acts → 9 MessageTypes):**

| MessageType | FIPA acts absorbed | How |
|---|---|---|
| COMMAND | request, cfp | A call-for-proposal is a type of command |
| QUERY | query-if, query-ref | Merged — both ask for information |
| RESPONSE | inform, confirm, propose, inform-if, inform-ref | All deliver information/judgment |
| DECLINE | refuse, reject-proposal | Both reject an obligation or offer |
| DONE | accept-proposal | Both signal positive resolution of an exchange (see note below) |
| FAILURE | failure | Direct 1:1 |
| HANDOFF | proxy | Transfer obligation to another agent |
| STATUS | *(carved from inform)* | New — progress reports distinguished from terminal responses |
| EVENT | *(no FIPA equivalent)* | New — observer-only telemetry, not a speech act |

**DONE ← accept-proposal note:** In FIPA, accept-proposal occurs *before* work begins
(accepting an offer kicks off execution). DONE occurs *after* work completes. These are
at opposite lifecycle points. Both signal positive resolution, but CaseHub's negotiation
flow, when implemented (§3.2c), handles accept-proposal semantics at the channel pattern
level rather than the speech-act level.

**disconfirm note:** FIPA disconfirm is an informational act ("I know that proposition P
is false") — about truth, not refusal. CaseHub doesn't need a separate "that's false"
speech act; RESPONSE with disagreeing content covers the ground. disconfirm is not mapped
to DECLINE (which handles refusal); it is simply not needed as a distinct speech act.

**Dropped entirely (8 FIPA acts):**

| FIPA act | Why dropped | CaseHub layer that handles it |
|---|---|---|
| agree | Implicit — starting work signals acceptance | Engine (worker dispatch) |
| cancel | Case lifecycle, not agent discourse | Engine (case state machine) |
| disconfirm | Informational — "P is false"; RESPONSE covers disagreement | Not needed as distinct act |
| not-understood | Error handling, not communication | Qhorus (exceptions) |
| propagate | Infrastructure fan-out | Claudony (fleet relay) |
| request-when | Reactive triggers | Engine (CDI events, signal bridge) |
| request-whenever | Reactive triggers | Engine (CDI events, signal bridge) |
| subscribe | Structural membership, not negotiated | Engine (`CaseChannelLayout`) |

---

## 3. Channel Taxonomy

Five purpose categories. Each pattern carries discriminator values, recommended
`ChannelSemantic`, `MessageType` constraints, and FIPA cross-reference.

**Design decision: governance folded into coordination.** Earlier versions of this
document listed governance as a sixth purpose value. At the speech-act level, governance
IS coordination — the oversight channel uses the same obligation exchange pattern
(COMMAND → RESPONSE) as the work channel. The distinction is in participants
(human↔engine vs agent↔engine) and type constraints (`deniedTypes=EVENT`), not in the
fundamental communication pattern. Consensus (§3.1c) generalises oversight to M-of-N,
further confirming that governance is a coordination sub-pattern.

### 3.1 Coordination Channels — obligation exchange

Channels where one party commands and another responds with completion or refusal.
The fundamental pattern is obligation creation and fulfilment.

#### 3.1a Agent Mesh (work/observe/oversight)

The normative 3-channel layout for any agent participating in the CaseHub mesh.
Declared by `CaseChannelLayout` SPI (`io.casehub.api.spi.mesh`), `allowedTypes`
enforced at the Qhorus layer.

- **Implementations:** `NormativeChannelLayout` (3-channel), `SimpleLayout` (2-channel, no oversight)
- **Home:** `casehub-engine-api` (`io.casehub.api.spi.mesh`)
- **Semantic:** APPEND
- **FIPA:** Request (work, oversight), Subscribe (observe)

`MeshParticipationStrategy` is the companion SPI — ACTIVE, REACTIVE, or SILENT
engagement determines which channels surface in the agent's context.

| Channel | Purpose | allowedTypes | deniedTypes |
|---------|---------|-------------|-------------|
| `/work` | Obligation-carrying coordination | null (open) | null |
| `/observe` | Telemetry broadcast | EVENT only | null |
| `/oversight` | Human governance gate | null (open) | EVENT |

#### 3.1b Ad-hoc Engine Channels

Channels opened on-demand at specific engine lifecycle points, outside the
mesh layout topology.

- **"coordination"** — opened at case start for initial coordination
- **"worker:{name}"** — per-worker command dispatch channel
- **Home:** `casehub-engine` runtime (`CaseStartedEventHandler`, `WorkerScheduleEventHandler`)
- **Semantic:** APPEND
- **FIPA:** Request

#### 3.1c Consensus Gate (open — not yet implemented)

M-of-N independent votes collected until a threshold is met. No discourse —
each participant makes an independent judgment. Generalises oversight from
single-human to multi-party.

- **Semantic:** APPEND (votes accumulate; threshold evaluation in the backend/projection).
  Alternative: BARRIER for the N-of-N special case (unanimous — all must write).
  BARRIER is wrong for general M-of-N: it requires ALL contributors, not M-of-N.
  COLLECT clears after delivery, losing vote history. Threshold logic belongs in the
  `ConsensusChannelBackend`, not the transport semantic — `ChannelSemantic` describes
  data flow, not application-level decision logic.
- **Turn structure:** one-way per participant (vote), threshold-gated release
- **Lifecycle:** gated — opens, collects, releases on threshold, closes
- **Participants:** M-of-N (human, agent, or mixed)
- **FIPA:** Multi-party Propose (no direct FIPA equivalent for threshold semantics)
- **First likely consumer:** clinical DSMB approval, multi-agent oversight gates

**Probable ChannelBackend shape:**

```
ConsensusChannelBackend implements ChannelBackend
  - threshold: int (M required out of N contributors)
  - contributors: Set<String> (declared participants)
  - post(): validates contributor identity, records vote
  - Projects via ConsensusProjection → ConsensusState
    - votes: Map<String, Vote> (contributor → accept/reject + rationale)
    - threshold: int
    - met: boolean
    - outcome: PENDING | APPROVED | REJECTED

On threshold met → fires CDI event (ConsensusReachedEvent)
  - caseId, channelId, outcome, votes
  - Engine signal bridge routes to case context
```

### 3.2 Deliberation Channels — structured discourse

Channels for multi-participant reasoning toward a shared conclusion.
Not about task coordination — about reaching a conclusion through discourse.

**Shared structural pattern** (documented, not extracted as code): all deliberation
channels implement Qhorus `ChannelBackend` directly, use `ChannelProjection<S>` to
fold message history into a typed state, and are activated by `ChannelInitialisedEvent`
observation. Future deliberation channels follow this pattern with their own state types.

**Projection variant:** `RenderableProjection<S>` (extends `ChannelProjection<S>`)
adds `projectionName()` and `render()` — making the projection available as an MCP tool
via `ProjectionRegistry` and `project_channel`. `DebateChannelProjection` implements
`RenderableProjection<ReviewState>` (MCP-visible). `ReviewChannelProjection` implements
plain `ChannelProjection<ReviewState>` (renders via `ReviewConversationRenderer` directly,
not through the MCP tool path). Deliberation channels that need LLM-readable output via
MCP should implement `RenderableProjection`.

#### 3.2a Debate — multi-agent, multi-turn, document-scoped

- **Home:** `casehub-drafthouse`
- **Naming:** `drafthouse/debate/d-{UUID}`
- **Participants:** REV, IMP, SUPERVISOR, MODERATOR, SELECTOR roles
- **Turn structure:** round-based, explicit round numbers
- **Entry types:** RAISE, AGREE, COUNTER, DISPUTE, QUALIFY, FLAG_HUMAN, DECLINED,
  MEMO, SUB_TASK_REQUEST, SUB_TASK_FINDING, SUB_TASK_ERROR, RESTART_CONTEXT
- **Constraint enforcement:** protocol-layer (`DebateProtocol.META_SENTINEL`), not Qhorus `allowedTypes`
- **State:** folds into `ReviewState` via `DebateChannelProjection`
- **Semantic:** APPEND
- **FIPA:** No equivalent — closest ancestor is argumentation frameworks (Dung 1995)

#### 3.2b Review — single reviewer, reactive, document-scoped

- **Home:** `casehub-drafthouse`
- **Participants:** one human, one LLM reviewer (auto-responds via `ReviewerChannelBackend`)
- **Turn structure:** reactive — LLM auto-invoked on QUERY
- **State:** folds into `ReviewState` via `ReviewChannelProjection`
- **Semantic:** APPEND
- **FIPA:** Query (QUERY → analysis → RESPONSE, but with judgment semantics)

#### 3.2c Negotiation (open — not yet implemented)

Two or more agents reaching agreement through proposal/counter-proposal exchange.
Iterative — proposals are refined until convergence or deadlock.

- **Semantic:** APPEND. Alternative: COLLECT (fan-in proposals before evaluation).
- **Turn structure:** proposal/counter-proposal, iterative
- **Lifecycle:** session-scoped (open negotiation → agreement or deadlock → close)
- **Participants:** agent↔agent or agent↔human
- **FIPA:** Contract Net / Iterated Contract Net — structurally equivalent message flow

**Probable ChannelBackend shape:**

```
NegotiationChannelBackend implements ChannelBackend
  - post(): records proposal, counter-proposal, accept, reject
  - Projects via NegotiationProjection → NegotiationState
    - proposals: List<Proposal> (proposer, terms, round)
    - currentRound: int
    - status: OPEN | AGREED | DEADLOCKED | EXPIRED
    - agreement: Terms (null until AGREED)

On agreement → fires CDI event (NegotiationCompletedEvent)
  - participants, agreed terms, round count
  - Output feeds into coordination channels (task assignments)
```

#### 3.2d Planning (open — not yet implemented)

Structured decomposition of a goal into an ordered task graph through discourse.
A supervisor proposes a breakdown, participants critique or refine, the group
converges on a plan. Output feeds into coordination channels.

- **Semantic:** APPEND
- **Turn structure:** proposal/critique, iterative
- **Lifecycle:** session-scoped (goal stated → plan converged → close)
- **Participants:** supervisor + contributing agents
- **FIPA:** No single protocol — combines Contract Net (propose/critique)
  with Request-When (trigger on goal)

**Probable ChannelBackend shape:**

```
PlanningChannelBackend implements ChannelBackend
  - post(): records goal, decomposition, critique, refinement
  - Projects via PlanningProjection → PlanState
    - goal: String
    - tasks: List<PlannedTask> (description, dependencies, assignee, status)
    - critiques: List<Critique> (taskRef, agent, concern, resolution)
    - status: DECOMPOSING | UNDER_REVIEW | CONVERGED | ABANDONED

On convergence → fires CDI event (PlanConvergedEvent)
  - goal, finalised task graph
  - Engine creates WorkItems from the task graph
```

### 3.3 Signal Channels — case state mutation

One-way state-change notifications into running case instances.

- **Home:** Three entry points across two repos:
  - `CaseSignalSink` — SPI in `casehub-work-api`; called by casehub-work on SLA
    escalation; impl in `casehub-engine-work-adapter` calling `CaseHubRuntime.signal()`
  - `QhorusMessageSignalBridge` — in `casehub-engine` runtime; observes
    `@ObservesAsync MessageReceivedEvent` for commitment-resolving message types
  - Direct REST → `CaseHubRuntime.signal()` — in `casehub-engine` runtime
- **Direction:** external event → case context mutation
- **Turn structure:** one-way, no response expected
- **Semantic:** EPHEMERAL
- **FIPA:** Inform (but qhorus signals mutate case context directly — tighter
  coupling than FIPA's mentalistic semantics)

### 3.4 Notification Channels — external boundary

Channels bridging external systems into the platform (inbound) or the platform
out to external systems (outbound).

- **Home:** `casehub-connectors` (outbound delivery), `casehub-qhorus` `connector-backend`
  submodule (inbound bridge)
- **Semantic:** APPEND (persistent record). Alternative: EPHEMERAL (fire-and-forget).
- **FIPA:** No equivalent — FIPA assumed a closed agent ecosystem

**ChannelBackend SPI hierarchy matters here.** Notification channels implement
`HumanParticipatingChannelBackend` (extends `ChannelBackend`), not plain `ChannelBackend`.
This adds `normaliserFor(UUID channelId)` — per-channel inbound type inference that
converts raw human prose into typed `NormalisedMessage`. The qhorus backend SPI hierarchy:

```
ChannelBackend (base — backendId, actorType, open, post, close)
├── AgentChannelBackend (ActorType.AGENT; post() fatal on failure)
├── HumanObserverChannelBackend (ActorType.HUMAN; inbound capped to EVENT; unlimited per channel)
└── HumanParticipatingChannelBackend (ActorType.HUMAN; full speech acts; normaliserFor())
```

`ConnectorChannelBackend` and `SlackChannelBackend` both implement
`HumanParticipatingChannelBackend`. A new notification adapter (e.g., Discord)
plugs into the `connector-backend` submodule, implementing
`HumanParticipatingChannelBackend`.

Examples:
- Slack webhook → `InboundMessage` → `ConnectorChannelBackend` → case signal
- Watchdog alert → `WatchdogAlertEvent` → `ConnectorService.send()`
- CloudEvent from `casehub-iot` / `casehub-qhorus` / `casehub-connectors` → `casehub-ras`

### 3.5 Infrastructure Channels — platform plumbing

Channels used internally for platform coordination, not visible to domain agents.

- **Fleet relay (claudony):** cross-node SSE tick delivery via `FleetMessageRelayObserver`
- **Channel sync (claudony):** `POST /sync` registers `ClaudonyChannelBackend`;
  `POST /notify` relays cross-node ticks
- **Semantic:** APPEND or EPHEMERAL depending on relay pattern
- **FIPA:** No equivalent — deployment concerns outside FIPA's scope

---

## 4. Discriminator Dimensions

Eight dimensions for classifying any channel pattern:

| Dimension | Values |
|-----------|--------|
| **Purpose** | coordination · deliberation · signal · notification · infrastructure |
| **Semantic** | APPEND · COLLECT · BARRIER · EPHEMERAL · LAST_WRITE |
| **Participants** | agent↔agent · human↔agent · agent→broadcast · system · M-of-N |
| **Turn structure** | unstructured · round-based · reactive · one-way · proposal/counter |
| **Constraint enforcement** | Qhorus `allowedTypes` · Qhorus `deniedTypes` · protocol-layer · none |
| **Lifecycle** | long-lived (task) · session-scoped (debate) · ephemeral (signal) · gated (consensus) |
| **Initiation** | COMMAND on /work · MCP tool · CDI event · external webhook · engine lifecycle |
| **FIPA protocol** | Request · Contract Net · Propose · Subscribe · Query · — (no equivalent) |

---

## 5. Purpose × Semantic Matrix

Recommended `ChannelSemantic` for each channel pattern. **Bold** = primary
recommendation. (parentheses) = viable alternative.

| Channel Pattern | APPEND | COLLECT | BARRIER | EPHEMERAL | LAST_WRITE |
|----------------|--------|---------|---------|-----------|------------|
| 3.1a Agent Mesh (work) | **✓** | | | | |
| 3.1a Agent Mesh (observe) | **✓** | | | | |
| 3.1a Agent Mesh (oversight) | **✓** | | | | |
| 3.1b Ad-hoc Engine | **✓** | | | | |
| 3.1c Consensus Gate | **✓** | | (✓) | | |
| 3.2a Debate | **✓** | | | | |
| 3.2b Review | **✓** | | | | |
| 3.2c Negotiation | **✓** | (✓) | | | |
| 3.2d Planning | **✓** | | | | |
| 3.3 Signal | | | | **✓** | |
| 3.4 Notification | **✓** | | | (✓) | |
| 3.5 Infrastructure | **✓** | | | (✓) | |

APPEND dominates — most conversation patterns accumulate history, and threshold
logic belongs in the application layer (backend/projection), not the transport
semantic. BARRIER is the N-of-N special case (unanimous consensus only). COLLECT
is an alternative for fan-in patterns but clears after delivery. LAST_WRITE has
no current primary use but is available for future blackboard-cell patterns.

---

## 6. FIPA Interaction Protocol Cross-Reference

| Channel Pattern | FIPA Protocol | Mapping | Divergence |
|----------------|---------------|---------|------------|
| 3.1a Mesh /work | Request | COMMAND → agent → DONE/FAILURE/DECLINE | No explicit AGREE — acceptance implicit in starting work |
| 3.1a Mesh /observe | Subscribe | EVENT broadcast to observers | Membership is structural (layout), not negotiated |
| 3.1a Mesh /oversight | Request | COMMAND from engine → human → RESPONSE | FIPA doesn't distinguish human vs agent participants |
| 3.1b Ad-hoc Engine | Request | Engine → worker dispatch | Ephemeral — opened per worker, not pre-declared |
| 3.1c Consensus Gate | Propose (multi-party) | Independent accept/reject against threshold | No FIPA equivalent for M-of-N threshold semantics |
| 3.2a Debate | — | No FIPA equivalent | Closest ancestor: argumentation frameworks (Dung 1995) |
| 3.2b Review | Query | QUERY → analysis → RESPONSE | Reactive auto-invocation has no FIPA precedent |
| 3.2c Negotiation | Contract Net | CFP → propose → accept/reject → iterate | Agent-to-agent vs FIPA's manager-to-contractors |
| 3.2d Planning | — | No single FIPA protocol | Combines Contract Net + Request-When |
| 3.3 Signal | Inform | One-way state notification | Mutates case context directly — tighter than FIPA |
| 3.4 Notification | — | External boundary crossing | Outside FIPA's closed-ecosystem assumption |
| 3.5 Infrastructure | — | Platform plumbing | Outside FIPA's agent-layer scope |

CaseHub inherits from FIPA where patterns match (Request, Contract Net, Subscribe)
and diverges in three ways: (1) speech acts reduced to 9 from 22 — 14 merged, 8 dropped,
(2) purpose-based channel classification is a layer FIPA doesn't have,
(3) infrastructure and notification categories address deployment realities FIPA
never targeted.

---

## 7. Open Design Space

All previously open patterns have been classified within the taxonomy (§3):

- **Negotiation** → §3.2c (deliberation)
- **Consensus** → §3.1c (coordination)
- **Planning** → §3.2d (deliberation)

Two items from the original open design space were evaluated and dismissed as
cross-cutting concerns, not distinct channel patterns:

- **Audit trail** — a property of any APPEND-semantic channel when combined with
  ledger capture, not a distinct channel type
- **Broadcast** — implemented by casehub-ras via `@ObservesAsync CloudEvent`;
  the CDI event bus, not a Qhorus channel

This section remains as a landing zone for future channel patterns as they emerge.

---

## 8. Academic Lineage

The channel taxonomy draws on established multi-agent systems research:

- **FIPA ACL** (IEEE SC00037J, 2002) — 22 communicative acts based on speech act theory.
  Qhorus reduces to 9, relocating infrastructure, orchestration, and error-handling
  concerns to their appropriate stack layers.
- **Speech Act Theory** (Austin 1962, Searle 1969) — performatives as actions, not just
  information transfer. Qhorus `MessageType` values are performatives.
- **Contract Net Protocol** (Smith 1980) — task allocation via CFP/propose/accept. Maps
  directly to the negotiation channel pattern.
- **Argumentation Frameworks** (Dung 1995) — structured multi-party reasoning. Ancestor
  of the debate channel pattern.
- **Communication-Centric Survey** (Yan et al. 2025) — two-level analytical framework
  classifying multi-agent communication across system-level (architecture, goals, protocols)
  and system-internal (strategy, paradigm, objects, content) dimensions.
- **LLM Agent Communication Protocol Taxonomy** (Sander et al. 2026) — predicts federated
  layered protocol stacks rather than monolithic standards; CaseHub's internal architecture
  demonstrates this principle at the intra-platform level.

---

## References

- `docs/PLATFORM.md` — Capability Ownership table (channel SPIs, mesh primitives)
- `docs/LIFECYCLE.md` — state machine taxonomy (companion document)
- `docs/repos/casehub-qhorus.md` — qhorus deep-dive (normative layout detail)
- parent#93 — coordination channel extraction (CLOSED — shipped)
- parent#294 — Reusable Platform Primitives epic
- `casehub/garden: docs/protocols/casehub/qhorus-consumer-integration-pattern.md`
- [FIPA Communicative Act Library](http://www.fipa.org/specs/fipa00037/SC00037J.html)
- [Beyond Self-Talk: Communication-Centric Survey](https://arxiv.org/html/2502.14321v2)
- [Technical Taxonomy of LLM Agent Communication Protocols](https://arxiv.org/html/2606.19135)
```

- [ ] **Step 2: Commit CHANNELS.md**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/CHANNELS.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(#293): rewrite CHANNELS.md — complete channel taxonomy

Seven-section design reference covering speech act foundation (FIPA 22→9),
five purpose categories with 12 channel patterns, discriminator dimensions,
purpose × semantic matrix, FIPA cross-reference, layered protocol stack,
and academic lineage.

Refs #293"
```

---

### Task 2: Fix docs/PLATFORM.md inconsistencies

**Files:**
- Modify: `docs/PLATFORM.md` (lines 570, 571, 617)

**Interfaces:**
- Consumes: verified code facts from spec review (NormativeChannelLayout, MessageType enum)
- Produces: consistent PLATFORM.md (no contradictions with code or CHANNELS.md)

- [ ] **Step 1: Fix /work speech acts (line 570)**

Change line 570 from:
```
| `work` | Task assignment and completion (prescriptive) | COMMAND, RESPONSE, DONE, DECLINE, EXPIRED |
```
to:
```
| `work` | Task assignment and completion (prescriptive) | COMMAND, RESPONSE, DONE, DECLINE |
```

Rationale: EXPIRED is a `CommitmentState` / `WorkItem` status, not a `MessageType`. The `MessageType` enum has exactly 9 values and EXPIRED is not one of them. The primary speech acts on /work are COMMAND, RESPONSE, DONE, DECLINE (the obligation cycle). STATUS and HANDOFF also appear but are less common. The code sets `allowedTypes=null` (unrestricted), so this column describes primary usage, not enforcement.

- [ ] **Step 2: Fix /observe speech acts (line 571)**

Change line 571 from:
```
| `observe` | Passive monitoring and state sharing (descriptive) | EVENT, QUERY, STATUS |
```
to:
```
| `observe` | Passive monitoring and state sharing (descriptive) | EVENT |
```

Rationale: `NormativeChannelLayout` sets `allowedTypes = Set.of(MessageType.EVENT)` for /observe. QUERY and STATUS are hard-blocked. Confirmed by agent-mesh-primitives spec (PP-20260604-a7ad99).

- [ ] **Step 3: Remove stale CaseChannelLayout placement violation (line 617)**

Delete this row from the Known Placement Violations table:
```
| `CaseChannelLayout` — normative 3-channel constants | `claudony-casehub` | `casehub-engine-api` or `casehub-platform-api` | parent#93 |
```

Rationale: Extraction shipped in parent#93. The Capability Ownership table (line 394) already correctly shows `CaseChannelLayout` in `casehub-engine-api`.

- [ ] **Step 4: Commit PLATFORM.md fixes**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/PLATFORM.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(#293): fix PLATFORM.md channel type inconsistencies

- /work: remove EXPIRED (CommitmentState, not MessageType)
- /observe: EVENT only (code sets allowedTypes=Set.of(EVENT), not EVENT+QUERY+STATUS)
- Remove stale CaseChannelLayout placement violation (extraction shipped in #93)

Refs #293"
```

---

### Task 3: Fix docs/repos/casehub-qhorus.md inconsistencies

**Files:**
- Modify: `docs/repos/casehub-qhorus.md` (lines 201, 205, 206)

**Interfaces:**
- Consumes: verified code facts from spec review
- Produces: consistent qhorus deep-dive (no contradictions with code or CHANNELS.md)

- [ ] **Step 1: Fix section header (line 201)**

Change line 201 from:
```
The agent mesh framework defines a 3-channel normative layout implemented via `NormativeChannelLayout` (Claudony SPI) and enforced at channel creation:
```
to:
```
The agent mesh framework defines a 3-channel normative layout implemented via `NormativeChannelLayout` in `casehub-engine-api` (`io.casehub.api.spi.mesh`) and enforced at channel creation:
```

Rationale: `NormativeChannelLayout` was extracted from claudony to casehub-engine-api in parent#93.

- [ ] **Step 2: Fix /work allowedTypes (line 205)**

Change line 205 from:
```
| `/work` | Task assignment and completion (prescriptive) | `COMMAND, RESPONSE, DONE, DECLINE, EXPIRED` | null |
```
to:
```
| `/work` | Task assignment and completion (prescriptive) | null (all types permitted) | null |
```

Rationale: `NormativeChannelLayout` sets `allowedTypes=null` for /work (unrestricted). EXPIRED is not a `MessageType`.

- [ ] **Step 3: Fix /observe allowedTypes (line 206)**

Change line 206 from:
```
| `/observe` | Passive monitoring and state sharing (descriptive) | `EVENT, QUERY, STATUS` | null |
```
to:
```
| `/observe` | Passive monitoring and state sharing (descriptive) | `EVENT` | null |
```

Rationale: `NormativeChannelLayout` sets `allowedTypes = Set.of(MessageType.EVENT)` for /observe.

- [ ] **Step 4: Commit qhorus deep-dive fixes**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-qhorus.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(#293): fix qhorus deep-dive channel type inconsistencies

- NormativeChannelLayout: now in casehub-engine-api (extracted in #93)
- /work: allowedTypes is null (unrestricted), not explicit list with EXPIRED
- /observe: EVENT only, not EVENT+QUERY+STATUS

Refs #293"
```
