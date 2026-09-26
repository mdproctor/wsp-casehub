# HANDOFF — casehub-examples

## Last Session

Completed the cognitive architecture wiring audit (#77) — all 6 issues closed. Fixed the Hibernate JPA entity indexing failure (#90). Verified the cognitive architecture end-to-end with a live scenario run. Filed and refined #89 (redefine briefing as voice card).

### What was done

- **#77 audit queue** (6/6 complete):
  - #87 (classloader visibility) — was done prior session, closed
  - #88 (upstream API sync) — was done prior session, closed
  - #86 (NeedTierMappingProvider @DefaultBean) — added `@Produces @DefaultBean` in blocks `BlocksBeans`, commit 78289913 in slot blocks
  - #85, #83, #82, #81 — verified already resolved by prior work, closed with evidence

- **#90** (Hibernate JPA entity indexing failure):
  - Root cause: stale `casehub-ledger` jar in `~/.m2`. Ledger's `.mvn/maven.config` points `maven.repo.local` to `/Users/mdproctor/claude/casehub/worktrees/30/.m2` — so `mvn install` from ledger repo doesn't write to shared `~/.m2/repository`
  - Fix: rebuilt ledger runtime+deployment with `-Dmaven.repo.local=~/.m2/repository`
  - Removed `quarkus.hibernate-orm.packages=io.casehub.eidos` workaround (no longer needed)
  - Updated `SocialCognitionIntegrationTest` — drives now flow through CognitionCore, not SocialConfig
  - All tests green: SocialCognitionIntegrationTest (5/5), CharacterCognitionTest (18/18), CognitiveActivationTest (10/10), BackendFactoryDiscoveryTest (3/3)

- **#89** filed and refined: "Redefine briefing as voice card — cognitive systems carry behavior"

- **End-to-end scenario verification**: 51 events, characters with distinct personalities, internal reasoning (asides), social interactions, puzzle-solving. Quarkus starts clean with all extensions.

- **Eidos personality rendering investigation**: eidos has a rich rendering pipeline (`assembleMarkdownCognitiveProfile`) that resolves Jungian codes to full descriptions. Wacky-manor bypasses it — uses blocks' `PersonalityPromptSection` which renders raw codes. No behavioral impact today since briefings dominate.

### Key insight from this session

The scenario output looks good but is almost entirely the LLM following briefing instructions. The cognitive systems (drives, memory, strategy, mood, mental models) are wired but haven't been proven to produce emergent behavior. The briefings are behavioral scripts that should be thinned to voice cards (speech patterns, accent, quirks) with behavior coming from neocortex. This is #89.

## Immediate Next Step

**#89 — design phase.** This is XL/High and needs a spec before implementation. The design must answer:
1. What stays in the briefing (voice card) vs what comes from cognitive state
2. How the single composition pipeline works across eidos, blocks, and neocortex
3. How to verify emergence — run with voice-only briefs and measure whether cognitive systems produce distinct behavior

Before starting #89, push the blocks @DefaultBean commit upstream (78289913 in slot blocks, not yet pushed).

## Critical: .m2 State

The slot's `.m2` is a **symlink** to `~/.m2/repository`. Key gotcha:

**Ledger's `.mvn/maven.config`** points `maven.repo.local` to a worktree-specific path. When rebuilding ledger from source, always pass `-Dmaven.repo.local=~/.m2/repository` or the jar won't reach the shared .m2. This was the root cause of the Hibernate failure.

All other repos install to `~/.m2/repository` normally.

### repos installed from source to `~/.m2/repository`:
- `platform` (agent-api, agent-claude, agent-router, agent-gate, agent-config, credentials, llm-config, identity + deps)
- `engine` (from slot 204 — full install including runtime)
- `blocks` (blocks + deps) — includes @DefaultBean for NeedTierMappingProvider (commit 78289913, not pushed upstream)
- `neocortex` (memory-api, memory, memory-core, memory-cbr-inmem, mindmap-*, cognitive-index)
- `eidos` (runtime, core, api, vocab, persistence-memory, eval + deployment)
- `ledger` (runtime + deployment rebuilt Sep 25 with `-Dmaven.repo.local=~/.m2/repository`)
- `qhorus` (runtime, api, persistence-memory)
- `work` (progress-api)

## Cross-Slot Dependencies

| Issue | Slot | Repo | Interaction with #89 | Status |
|-------|------|------|---------------------|--------|
| blocks#304 (CognitiveAttentionMediator) | 203 | blocks | Adds attention prompt section to `CognitionCore.promptSections()`. #89 redesigns the composition pipeline — must absorb #304's section. No conflict if #304 uses standard `promptSections()` pattern. | In progress — commented on #304 with guidance |

**As #89 progresses:** After #304 lands on blocks main, rebase slot 196 blocks to pick it up. The unified pipeline spec must include the attention section as a registered participant. If #89's design changes how sections register (e.g., moves from list-based to SPI-based), update #304's comment with the new pattern.

## Known Issues

1. **Eidos eval excluded** — `io.casehub.eidos.eval.**` in exclude-types prevents eval judge beans from loading; eval tests won't run until this is resolved
2. **Cross-repo API mismatches at HEAD** — qhorus references removed ledger types; engine references removed ledger field (`domainData`)
3. **Neocortex rebase conflicts** — slot neocortex branch has conflicts with main, skipped during both work-end cycles
4. **Blocks @DefaultBean not pushed upstream** — commit 78289913 in slot blocks main, needs `git push` to origin

## References

| Artifact | Path |
|----------|------|
| Phase D spec | specs/phase-d-cognitive-activation/2026-09-20-phase-d-cognitive-activation-design.md |
| Phase D plan | plans/2026-09-20-phase-d-cognitive-activation.md |
| Audit report | audits/2026-09-19-cognitive-architecture-audit.md |
| Prompt pipeline issue | casehubio/examples#89 |
