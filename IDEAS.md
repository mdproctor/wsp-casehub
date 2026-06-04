# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

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
**Status:** active

Some casehub repos show build status in their README, others don't. All should
have a consistent build badge so repo health is visible at a glance from GitHub.
Could be enforced via a project-health check or a template in the workspace-init skill.

**Context:** Noticed during session work on arc42stories-readme.md — inconsistency
across the ecosystem. Quick fix per repo but worth doing systematically rather than
ad hoc.

**Promoted to:**

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
