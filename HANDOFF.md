# HANDOFF — casehub

**Date:** 2026-08-04
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

Delivered the API catalogue infrastructure (#402) — machine-generated markdown API docs using jmarkdoc (Java) and TypeDoc (TypeScript), aggregated via existing subtree sync, with a cross-repo SPI overlay showing implementation matrix across repos.

Key design evolution: started from #359's per-SPI hand-curated chunks, pivoted to doclet-generated full public API reference. The "use existing tools, don't build custom" principle held — jmarkdoc for Java (requires JDK 25+ to run, analyzes any JDK source), TypeDoc for TS. Cross-repo correlation is the one custom script.

## Immediate Next Step

Roll out jmarkdoc to remaining Java repos — add doc generation to each repo's CI. Start with work, eidos, qhorus (high-value API surfaces). Same pattern as engine: `java -jar jmarkdoc.jar api/src/main/java docs/guides/api/`.

## What's Left

- **TypeDoc pilot for pages** — blocked on `GH_PACKAGES_TOKEN` for yarn install. Run in a session with the token, or in CI. · S · Low
- **Roll out jmarkdoc to remaining Java repos** (work, eidos, qhorus, ledger, worker, platform, blocks, neocortex, ras, desiredstate, connectors, ops, iot) — mechanical, same pattern. · M · Low
- **Roll out TypeDoc to blocks-ui** — after pages pilot proves the config. · S · Low
- **Per-repo CI workflows** — add generation step to each repo's existing CI. Template from parent's generate-api-catalogue.yml. · M · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #359 | Remaining: audit tooling + implementation pattern catalogue | L | Med | API catalogue done; audit tooling and pattern catalogue still open |

## References

- #402 closed — `df406f91` on main
- Spec: `docs/specs/issue-402-spi-api-catalogue/2026-08-03-api-catalogue-design.md`
- Blog: `blog/2026-08-04-mdp01-the-mechanical-layer.md`
- Garden: GE-20260804-7469da, GE-20260804-c1cf5c, GE-20260804-09c7dc (jmarkdoc entries)
