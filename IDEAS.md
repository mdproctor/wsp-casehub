# Idea Log

Undecided possibilities — things worth remembering but not yet decided.
Promote to an ADR when ready to decide; discard when no longer relevant.

---

## 2026-05-10 — Layer 2 tutorial framing aligned across aml and clinical

**Priority:** high
**Status:** active

Both aml and clinical Layer 2 should open with the same framing: *"the foundation tracks the deadline; the domain sets the policy."* Mechanism is identical (`claimDeadline` on `WorkItem`); regulatory context differs (30-day FinCEN flat SLA for aml; 1h/24h/7d by CTCAE grade for clinical). Both tutorials should be visibly consistent here.

Open question shared by both: does SLA expiry escalation create a **second WorkItem** or just mark the entity? Resolve from `casehub-work` deep-dive before either Layer 2 is designed. Both sessions must use the same answer.

**Context:** aml and clinical compared Layer 2 SLA approaches mid-brainstorm before either had a plan. Clinical confirmed the teaching structure is parallel. Framing phrase agreed collaboratively.

**Promoted to:**
