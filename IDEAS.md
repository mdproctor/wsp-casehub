# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-06-12 — casehub-ras: situational awareness and reactive case creation

**Priority:** high
**Status:** active

`casehub-ras` (Reticular Activating System) — a new repo that monitors multiple data streams and
detects scenarios that trigger case creation. The detection layer is fully pluggable via a `Ganglion`
SPI: implementations include `JavaSwitchGanglion` (simple pattern matching), `DroolsCepGanglion`
(complex event processing with sliding windows and event correlation), `BayesianGanglion` (weighted
signal accumulation), and `LlmGanglion` (narrative/ambiguous signal detection).

Architecture: multiple `Ganglion` implementations feed into a single RAS coordination layer. The RAS
aggregates detection results, handles composite events and detection chains, and decides when accumulated
signal crosses the threshold to create a case. Ganglia detect; the RAS decides. Multiple RAS instances
possible for independent detection contexts.

This addresses a real gap: CaseHub currently requires imperative `startCase()` calls. casehub-ras makes
case creation reactive and declarative — the system watches for conditions and creates cases automatically.

Connects to: `casehub-iot` `StateChangeEvent`, `casehub-desiredstate` `EventSource` SPI, `casehub-qhorus`
message streams as inputs. Drools CEP is a natural fit for the composite event chain requirement.

**Context:** Design discussion 2026-06-12. Terminology: RAS (Reticular Activating System) — chosen because
it precisely maps to "decides what's significant enough to elevate to awareness (a case)." Ganglion as the
internal detection unit — multiple ganglia within one RAS, natural biological metaphor for pluggable detectors.

**Promoted to:** casehubio/casehub-ras — spec at `docs/superpowers/specs/2026-06-12-casehub-ras-design.md`, epics #1–#5
**Status:** promoted

---

## 2026-06-12 — Desired-state management as research project and reference architecture

**Priority:** high
**Status:** active

Build `casehub-desiredstate` (generic runtime: planner, reconciliation loop, fault
policy, core SPIs) and `casehub-ops` (domain implementations: infrastructure
provisioning, compliance posture, IoT state, CaseHub agent topology) as a research
project and reference architecture — not a product commitment. The primary story is
an accountability-native desired-state layer sitting above Terraform and Ansible,
adding what they structurally lack: continuous reconciliation, tamper-evident audit
trail, first-class human governance gates, and trust-weighted agent routing. A
secondary path shows what's possible when CaseHub owns the full provisioning stack,
letting the community decide whether that gains traction.

The IBM/RHT angle is strong: Ansible and OpenShift are already in regulated
enterprises. CaseHub DSM as a governance layer above Ansible — every playbook
execution committed, tamper-evident, trust-weighted — is a story the RHT enterprise
field can sell without displacing the existing investment. The compliance posture
domain is the largest identified market gap: Vanta/Drata/Secureframe do point-in-time
audit prep; nobody has built continuous compliance posture as a desired-state control
plane (GRC market ~$50B, fastest-growing segment).

Self-referential: CaseHub manages its own agent topology via the DSM system. Strong
internal validation; CaseHub is the first customer of its own tool.

Two repos:
- `casehub-desiredstate` — generic domain-agnostic runtime (foundation tier)
- `casehub-ops` — domain implementations: infra provisioning (Terraform/Ansible
  adapters), compliance posture, IoT desired state, CaseHub agent topology
  (integration tier). "casehub-infrastructure" rejected — ambiguous, could later
  mean CaseHub's own deployment infrastructure.

**Context:** Extended brainstorm from "cases as long-lived service managers" →
desired-state + cases → accountability gap in Terraform/Ansible → IBM/RHT story
→ two-path research strategy. Research doc:
`docs/superpowers/research/2026-06-07-desired-state-management-research.md`

**Promoted to:**

---

## 2026-06-04 — ARC42STORIES as a continuous planning and delivery loop

**Priority:** medium
**Status:** active

ARC42STORIES should work in both directions: plan Chapters and roadmaps upfront to
sequence delivery, then revise them as work happens and reality diverges from the
plan. Currently the workflow only does the second half — syncing after the fact.
Adding the planning half means starting a new feature area by drafting Chapters first,
mapping existing open issues to them, and using the Chapter sequence as the delivery
order. The plan is guidance — developers choose which issues to work on and when,
the Chapter map is not a mandate.

**Context:** Currently ARC42STORIES is updated reactively via update-design. The format
already supports forward planning (Journeys, Chapters, layer impact, sequencing). Worth
formalising as a planning step in the skill workflow — applicable to any project using
these skills, not just casehub. Came up during README writing for arc42stories-readme.md.

**Promoted to:**

---

## 2026-06-04 — Standardise build status badges in all repo READMEs

**Priority:** low
**Status:** promoted

Some casehub repos show build status in their README, others don't. All should
have a consistent build badge so repo health is visible at a glance from GitHub.
Could be enforced via a project-health check or a template in the workspace-init skill.

**Context:** Noticed during session work on arc42stories-readme.md — inconsistency
across the ecosystem. Quick fix per repo but worth doing systematically rather than
ad hoc.

**Promoted to:** Done 2026-06-12 — CI + PR badges added to all 19 casehub repos via gh api; quarkmind ci.yml created.

---

## 2026-05-25 — Shared base doc for PLATFORM.md and APPLICATIONS.md

**Priority:** low
**Status:** active

PLATFORM.md and APPLICATIONS.md both contain cross-cutting convention sections
(Flyway migration paths, dependency scoping, naming rules) that have to be kept
in sync manually. A shared `CONVENTIONS.md` or `COMMON.md` that both docs
reference or incorporate would make these sections single-source-of-truth.
PLATFORM.md and APPLICATIONS.md would then extend it with their tier-specific content.

**Context:** Noticed while adding a Flyway migration path protocol — the persistence
section in PLATFORM.md would need updating, and the same convention applies to
apps in APPLICATIONS.md. Two places, same rule.

**Promoted to:**

---

## 2026-05-10 — Layer 2 tutorial framing aligned across aml and clinical

**Priority:** high
**Status:** active

Both aml and clinical Layer 2 should open with the same framing: *"the foundation tracks the deadline; the domain sets the policy."* Mechanism is identical (`claimDeadline` on `WorkItem`); regulatory context differs (30-day FinCEN flat SLA for aml; 1h/24h/7d by CTCAE grade for clinical). Both tutorials should be visibly consistent here.

Open question shared by both: does SLA expiry escalation create a **second WorkItem** or just mark the entity? Resolve from `casehub-work` deep-dive before either Layer 2 is designed. Both sessions must use the same answer.

**Context:** aml and clinical compared Layer 2 SLA approaches mid-brainstorm before either had a plan. Clinical confirmed the teaching structure is parallel. Framing phrase agreed collaboratively.

**Promoted to:**

---

## 2026-07-07 — Cross-repo project manager: coordination service for the casehub fleet

**Priority:** high
**Status:** active — blocked by claudony resurrection and devtown UI

### Problem

CaseHub development spans ~27 active repos, each with its own Claude Code session in a
separate tmux pane. Today there is no unified view of what's working where, what's paused,
what's blocked, or what just landed. Cross-cutting concerns (interface changes, convention
updates, dependency bumps) require manual tmux-switching and sequencing. Dependencies between
issues across repos are tracked in prose inside issue bodies — no structured graph, no
automated blocking enforcement, no priority visualization.

### Three capabilities

**1. Fleet visibility (integrates with work-start)**
- What work is active in each repo right now (branch, issue, status)
- What work is paused (saved branch state, waiting for return)
- What recently completed (merged to main, closed issues)
- Surfaced during work-start in any repo session — show ecosystem-level blockers
  and cross-cutting concerns affecting this repo, full dashboard on demand

**2. Dependency tracking and priority visualization**
- Structured cross-repo dependency graph: "this task blocks these tasks in other repos",
  "this task requires changes in N repos before it can proceed"
- Priority visualization — not just a flat list but a graph showing blocking chains,
  critical paths, and where effort unblocks the most downstream work
- Better than the current prose-in-issue-body approach — queryable, renderable,
  maintainable

**3. Cross-cutting concern automation**
When a change in one repo needs propagation to others, the coordinator orchestrates the
full lifecycle:

*Phase 1 — Quiesce the fleet:* Three pause strategies for target repo sessions, chosen
per-repo based on urgency:
- **Force-stop:** send escape to the tmux pane, interrupt immediately
- **Wait-for-prompt:** monitor until the session is idle at a prompt, then intervene
- **Wait-for-work-end:** let the current work complete fully before intervening

*Phase 2 — Save state:* For each target repo, if on a branch, ensure everything is
committed and the branch is in a clean paused state. Record the branch name and work
context for later restoration.

*Phase 3 — Apply cross-cutting change:* Create a branch for the cross-cutting work.
The coordinator operates in hybrid mode:
- **Direct edits** for repos where the change is mechanical (version bumps, renames,
  signature updates, import fixes) — the coordinator Claude does this itself
- **Delegated edits** for repos where the change requires domain judgment — the
  coordinator unpauses that repo's Claude session with a specific instruction,
  waits for completion, then re-pauses

*Phase 4 — Land and restore:* Once all repos have the change applied, merge to main
and push across the fleet. Then restore each repo to its previous branch and optionally
unpause paused work.

*Bonus — Chained PRs:* Optionally create linked PRs across repos instead of direct
merges, preserving review gates. The chain would respect dependency order
(platform → foundation modules → application modules).

### Architecture direction

**Not a standalone tool — a claudony capability.** Built on the casehub platform itself,
running as an agent within claudony. This means:

- **Persistence:** SQLite database (or casehub ledger) for work state, dependency graph,
  fleet status, cross-cutting concern queue
- **Orchestration:** tmux session management via `send-keys`, `capture-pane`, pane
  targeting by repo name
- **Intelligence:** Claude invocation via prompt/MCP only where judgment is needed —
  most coordination is mechanical state machine work
- **Visualization:** could use casehub-pages for web dashboards, or terminal TUI for
  in-session quick checks, or both

The service is reactive — it watches for events (session state changes, issue updates,
branch merges) and acts, rather than being invoked on demand.

### Relationship to devtown

Complementary, not overlapping. DevTown models software development workflows as casehub
cases (PR review, merge decisions, CI orchestration). This project manager operates at the
fleet/portfolio level — coordinating across repos, managing cross-cutting concerns, tracking
inter-repo dependencies. DevTown is "how one repo does its dev work"; this is "how the
ecosystem moves together."

Could share infrastructure: both are casehub domain applications, both use the ledger for
audit trail, both interact with GitHub. The dependency graph from this tool could feed into
devtown's case creation (e.g., "cross-cutting concern X generated cases in repos A, B, C").

### Eat-your-own-dog-food angle

CaseHub managing its own development as a casehub domain application. Cases for epics and
cross-cutting concerns, work items for per-repo tasks, the ledger for state transitions,
claudony agents coordinating repo sessions, channels for inter-session communication. The
project manager becomes both a product capability and internal tooling — validated by daily
use before being offered to users.

### Prerequisites

1. **Claudony running** — the coordination service lives here
2. **DevTown UI** — need the web visualization layer that this will also use
3. **tmux session naming convention** — each repo session needs a predictable pane name
4. **work-start/work-end state persistence** — sessions need to write their state
   somewhere the coordinator can read (today HANDOFF.md is per-workspace, not machine-readable)

### Open questions

- How does the dependency graph sync with GitHub issues? Bidirectional (issues update the
  graph, graph updates issues) or one-way (graph is source of truth, issues are rendered views)?
- What's the right granularity for the coordinator's direct-edit capability? Should it be
  limited to changes that can be validated by compilation, or can it make semantic changes
  in repos it understands well?
- Should cross-cutting branches follow a naming convention (e.g., `xcut/description`) to
  distinguish them from regular feature branches?
- How does this interact with the incremental build system (`build-all-decision.sh`)? A
  cross-cutting change that touches many repos needs a coordinated build verification.

**Context:** Brainstorming session 2026-07-07. Started as "project manager task" and
evolved through clarifying questions into a claudony capability design. Decided to
capture as idea rather than full spec — prerequisites (claudony, devtown UI) need to
land first.

**Promoted to:**
