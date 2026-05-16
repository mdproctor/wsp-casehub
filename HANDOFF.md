# Session Handover — 2026-05-15

## Engine session (earlier today — preserved from previous handover)

**engine#258 and #261 complete.** `ContextDiffStrategy` selection refactored to config-driven `@Produces @DefaultBean` producer. Config: `casehub.engine.diff-strategy=none|top-level|json-patch` (default `none`). Protocol updated. Open: engine#255 (HumanTaskTarget template), #252 (SubCaseCompletionService), #254 (Java 21 migration), #253 (hibernate-reactive dep). Blog: `blog/2026-05-15-mdp01-config-over-cdi.md`.

---

## Parent/tooling session (this session)

### What happened

1. **ARCHITECTURE.md** — `docs/ARCHITECTURE.md` created for casehub/parent: 17 patterns, rationale, dependency rule. PLATFORM.md pointer added. Blog article published.

2. **Workflow audit** — 20 gaps found in epic/workspace/routing skills (`mdproctor/cc-praxis#71`). Nine fixed (trivial + easy). Eleven hard items remain (#87–91).

3. **cc-praxis skill fixes** — workspace-init routing template, handover journal-entry default, java-update-design mode detection, adr routing cascade, write-blog git-C, epic close plan archiving + publish-blog offer + spec scoping.

4. **Routing standardisation** — all 10 casehub workspace CLAUDE.md files now consistent: `adr/specs → project`, `blog → workspace → mdproctor.github.io`, `plans/design/snapshots → workspace`. Three wrong-direction moves required recovery; quarkmind and cccli force-pushed.

5. **24 blog entries published** across all repos. reactive-blackboard-control-shell moved from `_articles/` to `_notes/`.

6. **casehub-flow rename** — pending Treble confirmation. Current thinking: `casehub-app`.

### Open items

- `cc-praxis#87–91` — 5 hard workflow issues (DESIGN.md section hashing, atomic branch switching, HANDOFF.md to main, pause/resume)
- casehub-flow → `casehub-app` rename (confirm with Treble first)
- `plans/2026-05-13-workspace-init-routing-fix.md` — skill fix instructions for cc-praxis
- `plans/2026-05-13-skill-routing-improvements.md` — write-blog routing improvement instructions

### Next

Pick up `cc-praxis#88` (atomic branch switching helper) or start the casehub-flow rename once Treble confirms.

### References

- Blog: `blog/2026-05-15-mdp01-workflow-audit-routing-repair.md`
- cc-praxis epic: `mdproctor/cc-praxis#71`
- Plans: `plans/2026-05-13-workspace-init-routing-fix.md`
