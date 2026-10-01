# HANDOFF — casehub-examples

## Last Session

Two phases of work:

**1. Pre-test hygiene (branch closed, landed on main):**
- #71 — Removed ManorTrustEvents (superseded by CDI trust events)
- #92 — Closed as already done (voice descriptions)
- #91 — Removed CognitiveBudget (attention-driven gating replaces it)
- #40 — Refactored ManorEvent from 12-param record to sealed hierarchy (Action/Dialogue/Aside/Narrator)

**2. Cross-repo cognition migration (on main, no branch):**
- All 3 repos rebased against canonical local main
- Neocortex: deleted 5 stale CbrCase duplicate files from incomplete rename
- Blocks: removed CDI producers for missing neocortex classes (SocialNormDetector, NarrativeGoalEscalationPolicy, LlmCrossAxisGoalEnricher)
- Examples: migrated 26 files from `blocks.agentic.social` → `neocortex.cognition` packages
- Deleted 5 obsolete InMemory store classes — replaced with neocortex Memory classes backed by InMemoryCbrRecordStore
- Fixed `contribute()` → `render()` API change (CognitionPromptRenderer)
- Fixed Quarkus integration tests — CDI bean producers, connector exclusion, engine-testing dependency
- **555 tests pass, 0 failures**

Closed #96, #71, #64 (Phase C epic — all 6 children done).

## Immediate Next Step

**#76 — Directive-minimal YAML rewrite.** The pre-test hygiene was specifically to prepare for this. Sequence:
1. Write behavioral test baselines for current directive-heavy descriptors
2. Strip verbose directives from descriptors-composite.yaml
3. Verify behavior doesn't regress — the cognitive system (drives, beliefs, trust, personality) should produce character behavior without hand-holding

## Pre-existing Issues

- `PersonalityCompositionVerificationTest` — 5 tests @Disabled (pre-existing)
- Blocks `engine-adapter` modules don't compile (engine repo not in slot) — not needed by examples

## References

| Artifact | Path |
|----------|------|
| Diary entry | wsp-casehub/blog/2026-10-01-mdp01-clearing-the-runway.md |
| Next issue | casehubio/examples#76 |
| Blocks migration issue | casehubio/blocks#328 (social-jpa modules — deleted from blocks, function served by neocortex CBR) |
