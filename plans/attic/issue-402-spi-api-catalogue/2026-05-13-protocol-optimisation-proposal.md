# Protocol Optimisation Proposal
**Date:** 2026-05-13  
**For:** Hortora Claude  
**From:** CaseHub parent session  
**Repo:** `casehubio/parent` at `/Users/mdproctor/claude/casehub/parent`

---

## Context — What You've Already Done

You (Hortora) completed two protocol audits this session:

- **Audit 1 & 2** — migrated ~22 generic Quarkus/Java entries from `docs/protocols/` to the canonical garden (`jvm` domain). CDI async boundaries, BroadcastProcessor backpressure, @ApplicationScoped test isolation, etc. These are now GE-IDs and no longer casehub-specific rules.
- The `docs/protocols/INDEX.md` is now three-tiered:
  1. Casehub-specific protocols (17 files)
  2. Garden references — generic entries now in the garden, listed for discoverability
  3. Casehub domain garden entries — casehub-specific gotchas by repo

This work established a clear framework: **generic Quarkus/Java knowledge → garden; casehub-specific rules → protocols**.

---

## The Decision Framework (for this proposal)

Before placing anything, apply this test:

| Question | If yes → |
|----------|----------|
| Is this principle true for any Java/Quarkus project, not just casehub? | Garden (`jvm` domain) or reusable protocol |
| Could it be written to be universal without losing its casehub value? | Write universally, use casehub as the worked example |
| Is it specific to casehub's module names, numbering, or architecture? | Casehub protocol in `docs/protocols/` |
| Is it a gotcha/technique discovered through pain, not a rule to follow upfront? | Garden entry (forage), not a protocol |

---

## Item 1 — Fix Stale Link in PLATFORM.md (definite, no judgement needed)

**The problem:** `docs/PLATFORM.md` contains an Implementation Protocols table (near the bottom, "Cross-Cutting Concerns → Implementation Protocols") that references `protocols/sql-type-portability.md`. That file no longer exists — it was migrated to the garden as `GE-20260512-2c2eff` during Audit 2.

**The fix:** Replace the table row:
```
| [SQL type portability](protocols/sql-type-portability.md) | `DOUBLE PRECISION` not `DOUBLE`; `SMALLINT` not `TINYINT` |
```
with a garden reference:
```
| SQL type portability (GE-20260512-2c2eff) | `DOUBLE PRECISION` not `DOUBLE`; `SMALLINT` not `TINYINT` — see garden |
```
Or remove the row if the garden reference in INDEX.md is sufficient discoverability.

**Commit:** Straightforward doc fix.

---

## Item 2 — Persistence Module Split Rule (needs your judgement on placement)

**Where it currently lives:** Inline in `docs/PLATFORM.md` Step 4 as a full paragraph — the only Step 4 bullet that isn't a one-liner. It reads:

> **Persistence module split rule:** JPA entity classes MUST live in a separate module from the domain model SPI. Any artifact that bundles JPA entities forces every downstream consumer to configure a datasource — including test modules that use in-memory repos. The correct split: `<name>-api` (domain POJOs + SPIs, zero JPA), `<name>` or `<name>-hibernate` (JPA entities + migrations). `casehub-work` is the canonical example: `casehub-work-api` is JPA-free; the runtime module has entities and is kept at arm's length. Violating this rule causes cascading datasource failures across all downstream test suites.

**The proposal:** Extract it from PLATFORM.md and replace with a one-line pointer, the way `module-tier-structure.md` and `auth-retrofit-readiness.md` are handled.

**Placement question — your call:**

The underlying principle ("JPA entities must not co-locate with domain SPIs") is **universally true for any multi-module Java/Quarkus project**. It is not casehub-specific. By the framework above, this is a candidate for the garden (`jvm` domain) rather than a casehub protocol.

However: it is closely related to `module-tier-structure.md` (which IS a casehub protocol). It could either:

**Option A — Garden entry:** Add as a `jvm` domain garden entry. The casehub example (casehub-work-api) becomes the illustration. PLATFORM.md Step 4 gets a garden reference instead of a protocol reference.

**Option B — Fold into module-tier-structure.md:** The persistence split is arguably already implied by the three-tier rule. Add a "Persistence Module Split" section to `module-tier-structure.md` rather than creating a new entry. PLATFORM.md keeps its existing `module-tier-structure.md` pointer.

**Option C — Standalone casehub protocol:** Keep it casehub-specific because the worked example (casehub-work-api vs casehub-engine dependency) is so concrete and useful that writing it universally loses the specificity.

Our instinct: **Option B** — it belongs in module-tier-structure.md as a concrete consequence of Tier 1 being JPA-free. But you understand your taxonomy better than we do.

**PLATFORM.md change (regardless of Option chosen):**  
Replace the paragraph with a single bullet:
```
- **Persistence module split:** JPA entities must not co-locate with domain SPIs — forces all consumers to configure a datasource. See [module-tier-structure.md](protocols/module-tier-structure.md).
```
(or equivalent garden reference if Option A)

---

## Item 3 — Ledger Subclass Extension Rules (new protocol, casehub-specific)

**Where it currently lives:** Inline in PLATFORM.md Step 4 as a single bullet:
```
- Ledger subclasses: JOINED inheritance, consumer-owned V1004+ migration, domain-agnostic leaf hash
```

**The proposal:** Extract into a new protocol file `ledger-subclass-extension.md` and replace the bullet with a one-line pointer.

**Placement:** Casehub-specific. The JOINED inheritance pattern is standard JPA, but:
- V1004+ Flyway numbering is casehub-ledger's specific allocation
- "Domain-agnostic leaf hash" is specific to casehub-ledger's Merkle Mountain Range structure
- The consumer-owned migration pattern is casehub-specific convention

This stays in `docs/protocols/` as a casehub protocol.

**Suggested protocol content:**
- When to use JOINED inheritance (and why not TABLE_PER_CLASS or SINGLE_TABLE)
- The V1004+ migration numbering rule and why (casehub-ledger owns V1000–V1003; consumers start at V1004)
- Domain-agnostic leaf hash requirement — the leaf hash must not encode domain data, only the entry's own fields
- The consumer-owned migration pattern — each consuming repo owns its own join table migration
- A checklist for adding a new ledger subclass

---

## What PLATFORM.md Looks Like After All Three Items

Step 4's pattern bullets become:
```
- Ledger subclasses: JOINED inheritance, consumer-owned V1004+ migration. See protocols/ledger-subclass-extension.md
- Persistence module split: JPA entities must not co-locate with domain SPIs. See protocols/module-tier-structure.md (or garden ref)
```

The Implementation Protocols table at the bottom drops the stale sql-type-portability row (or updates it to a garden reference).

PLATFORM.md becomes leaner in Step 4 without losing any rules — each bullet is now a one-line hook that RAG can surface, with full detail in the protocol or garden.

---

## Files to Create / Modify

| Action | File | Notes |
|--------|------|-------|
| Fix | `docs/PLATFORM.md` | Remove/update stale sql-type-portability link; replace persistence split paragraph with pointer; replace ledger subclass bullet with pointer |
| Create | `docs/protocols/ledger-subclass-extension.md` | New casehub protocol |
| Update or garden | `docs/protocols/module-tier-structure.md` or garden | Add persistence split content (your call on placement) |
| Update | `docs/protocols/INDEX.md` | Add ledger-subclass-extension.md entry |

---

## What We Are NOT Proposing

- Further extraction from PLATFORM.md beyond these three items. The remaining Step 4 bullets are already concise one-liners. The capability ownership table, boundary rules, and cross-cutting concerns tables are navigation/lookup content that should stay inline.
- Moving any of the 17 remaining casehub protocol files — that's your audit territory, not ours.
- Any changes to the garden structure or GE-ID taxonomy — that's yours.
