# HANDOFF — casehub-examples

## Last Session

**Landed on main:** `cfc6152` (#98) + `090d7c2` (#99)

### What was done

1. Multi-repo rebase + .m2 rebuild (all 8 repos)
2. Engine package rename fix (#98) — `engine.internal` → `engine.runtime` in CDI exclusions and test imports
3. Old vs new briefing comparison runs
4. Generic character ablation test (#99) — proves taxonomy drives behavior, not model priors
5. Style-directive ablation — "lean into colloquialisms" beats prescribed catchphrases for 4/5 archetypes

### Key findings

- **Taxonomy is load-bearing.** Generic characters (no pop-culture refs) exhibit identical behavioral patterns to Wacky Races counterparts.
- **Prescribe structure, not content.** Style-directive ("lean into Southern colloquialisms") produces richer voice than prescribed catchphrases.
- **Exception: non-regional archetypes.** Theatrical villain didn't spontaneously generate an evil laugh — keep 1-2 prescribed anchors for archetypes without strong regional grounding.

## Baseline Transcripts — DO NOT DELETE

These transcripts are the comparison baselines for the #97 iterative pare-back. Every future ablation run compares against these. They live in `wacky-manor/docs/eval/` on main:

| Transcript | Events | What it tests | Profile |
|---|---|---|---|
| `old-briefings-20261002/transcript.json` | 327 | Verbose briefings (pre-rewrite) — behavioral ceiling | BASELINE |
| `new-briefings-20261002/transcript.json` | 333 | Directive-minimal (post-rewrite) — parity confirmation | BASELINE |
| `generic-briefings-20261002/transcript.json` | 306 | Generic chars + prescribed catchphrases — taxonomy validation | GENERIC |
| `generic-colloquial-20261002/transcript.json` | 310 | Generic chars + style-directive only — emergence validation | GENERIC |
| `directive-minimal-20261002/partial-events.json` | 25 | Partial earlier run (kept for historical reference) | COMPOSITE |

**How to run a comparison:** start quarkus:dev with the desired profile, POST `/manor/start`, poll `/manor/events`, save transcript alongside these baselines. Compare character voice distinctiveness and behavioral patterns.

```bash
# Run with GENERIC profile
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml quarkus:dev -s .mvn/slot-settings.xml -Dquarkus.http.port=8180 -Dmanor.scenario.profile=generic -Dmaven.test.skip=true

# Start scenario and poll
curl -4 -s -X POST http://127.0.0.1:8180/manor/start
curl -4 -s http://127.0.0.1:8180/manor/events
```

## Immediate Next Steps

### 1. Iterative pare-back (#97 Phase 1)

Using the generic colloquial transcript as the baseline, progressively strip tendencies from `descriptors-generic.yaml` and `social-config-generic.yaml` to find the minimum taxonomy:

1. Remove tendencies one at a time
2. Re-run generic scenario (~10 min, ~300 events)
3. Compare against `generic-colloquial-20261002/transcript.json` for behavioral drift
4. Find the floor — smallest set that maintains distinct characters

**Villain laugh investigation:** Can `theatrical-menace: 0.9` drive + tendency "you express triumph through dramatic vocalisation" produce an evil laugh without prescribing one? Test with Vincent Marsh.

### 2. Phase 2 — Memory-seeded personality emergence

After pare-back establishes the minimum taxonomy, feed into neocortex memory seeding (#398–#402).

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

**Known cross-repo issues (as of 2026-10-02):**
- Engine test code references stale neocortex CBR APIs — skip test compile
- Blocks engine-adapter-core references old `engine.internal.executor` package — exclude from build
- Blocks blocks-core has `MemoryDomain` type mismatch with neocortex — exclude from build

## Issues Created This Session

- **casehubio/examples#98** — CLOSED — engine.internal→engine.runtime package rename fix
- **casehubio/examples#99** — CLOSED — generic character ablation test

## Branch State

- `fix/098-engine-package-rename` — landed as cfc6152 + 090d7c2 on main
- `backup/pre-squash-fix/098-engine-package-rename-20261002` — pre-squash backup (9 commits)
