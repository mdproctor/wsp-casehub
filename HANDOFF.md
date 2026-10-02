# HANDOFF — casehub-examples

## Last Session

**Branch: `fix/098-engine-package-rename`** (4 commits, on examples repo)

### What was done

**1. Multi-repo rebase + .m2 rebuild**
- Rebased all 8 repos against upstream/main: platform, eidos, engine, blocks, neocortex, examples, qhorus, work
- Cleared `~/.m2/repository/io/casehub/` and rebuilt in dependency order
- Correct build order: platform → eidos → engine (skip test compile) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → examples/wacky-manor
- Blocks has cross-repo API mismatch in engine-adapter-core (package rename) — excluded from build, not needed by wacky-manor

**2. Engine package rename fix (#98)**
- Engine renamed `io.casehub.engine.internal` → `io.casehub.engine.runtime`
- Fixed wacky-manor CDI exclusion in `application.properties` (line 18)
- Fixed test import in `TestCdiBeans.java`
- This was the root cause of the 57 CDI deployment errors

**3. Comparison scenarios — old vs new briefings**
- Ran both scenarios (~10 min each, ~330 events each)
- Old briefings (verbose, from f9cbe81): `docs/eval/old-briefings-20261002/transcript.json` — 327 events
- New briefings (directive-minimal): `docs/eval/new-briefings-20261002/transcript.json` — 333 events
- Result: behavioral parity confirmed. Characters maintain voice and behavioral patterns with stripped briefings

**4. Generic character ablation test (#99)**
- Created `GENERIC` profile with renamed characters (Clara Bellingham, Vincent Marsh, Brixton Boys, Reginald Foxworth, James Hartwell)
- Same taxonomy structure, no pop-culture references, regional voice archetypes
- Files: `descriptors-generic.yaml`, `social-config-generic.yaml`, `dramatic-ensemble-style` template
- Made social-config loader profile-aware (`loadForProfile` method)
- Ran generic scenario: `docs/eval/generic-briefings-20261002/transcript.json` — 306 events
- Awaiting comparison analysis vs Wacky Races run

### Commits on branch

| SHA | Message |
|-----|---------|
| 6e20db4 | fix(#98): update engine.internal→engine.runtime package references |
| e67a133 | chore(#98): save old vs new briefing comparison transcripts |
| 3b0bb14 | feat(#99): add generic character ablation test profile |
| efc1e20 | chore(#99): save generic character ablation transcript — 306 events |

## Issues Created This Session

- **casehubio/examples#98** — Fix wacky-manor build: engine.internal→engine.runtime package rename
- **casehubio/examples#99** — Generic character ablation test — isolate taxonomy contribution vs model prior knowledge

## Build Notes

**Correct build order (updated):**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → examples/wacky-manor
```

**Known cross-repo issues:**
- Engine test code references stale neocortex CBR APIs — skip test compile
- Blocks engine-adapter-core references old `io.casehub.engine.internal.executor` package — exclude from build
- Blocks blocks-core has `MemoryDomain` type mismatch with neocortex — exclude from build (blocks main module still builds)

## Immediate Next Steps

### 1. Analyze ablation results
Review generic vs Wacky Races transcript comparison. Key question: is the behavioral delta small (taxonomy drives behavior) or large (model priors do the work)?

### 2. Iterative pare-back (#97 Phase 1)
After ablation analysis, progressively remove tendencies to find the minimum set that maintains character distinctiveness.

### 3. Land the branch
Once analysis is complete, squash and land `fix/098-engine-package-rename` on main.
