# HANDOFF — casehub-examples

## Last Session

**Issue #76 — Directive-minimal YAML rewrite (LANDED on main)**

Restructured wacky-manor character definitions into a typed content taxonomy. Every piece of content now has exactly one type and one structural home:

| Type | Home | Rendered in |
|---|---|---|
| Identity | `briefing` (1-2 sentences) | System prompt |
| Voice | `voice` section + genre templates | System prompt |
| Hard constraints | `constraints` (HARD) | System prompt |
| Tendencies | `tendencies` in social-config | Observation (first section) |
| Goals/Drives/Norms/Beliefs | social-config | Observation (dynamic) |

**Changes landed:**
- Stripped all 18 briefings to 1-2 sentence identity
- Added voice sections to 14 characters
- Stripped 3 role templates to voice-only
- Added `tendencies` field to SocialConfig with parsing + rendering
- Expanded social-config with tendencies + beliefs for all characters
- 8 integration tests validating the taxonomy
- Taxonomy audit: fixed 14 findings (duplicates, wrong-home, missing content)
- Fixed stale template args in baseline/belbin/jungian profiles
- Added PromptInspectionTest for eyeballing rendered prompts

**Live run results (partial — 25 events captured):**
Characters behave distinctively with stripped briefings. Voice, tendencies, constraints all working. Hooded Claw maintains dual persona, Dick Dastardly lies convincingly, Ant Hill Mob suspicious of Sneekly.

## Immediate Next Steps

### 1. Multi-repo rebase + rebuild (BLOCKER for scenario runs)

Quarkus dev server has CDI `Unsatisfied dependency` errors (EventLogRepository, CaseInstanceRepository) blocking startup. Fix:

```
rm -rf ~/.m2/repository/io/casehub/
```

Then rebuild in dependency order:

| Step | Repo | Location |
|---|---|---|
| 1 | platform | `/Users/mdproctor/claude/casehub/platform` (canonical) |
| 2 | eidos | `/Users/mdproctor/claude/casehub/eidos` (canonical) |
| 3 | engine | `/Users/mdproctor/claude/casehub/engine` (canonical) |
| 4 | blocks | `/Users/mdproctor/claude/casehub/slots/196/blocks` (slot) |
| 5 | neocortex | `/Users/mdproctor/claude/casehub/slots/196/neocortex` (slot) |
| 6 | examples | `/Users/mdproctor/claude/casehub/slots/196/examples` (slot) |

For each: `git fetch upstream && git rebase upstream/main` then `mvn install -DskipTests`

### 2. Comparison runs (after rebuild)

Retrieve old briefings from git (`f9cbe81`), run scenario, save transcript. Then run with new briefings, save transcript. Compare character behavior delta. Save BOTH transcripts to `docs/eval/`.

### 3. Iterative pare-back (#97 Phase 1)

After baseline comparison, progressively remove tendencies to find the minimum set.

## Issues Created This Session

- **casehubio/examples#97** — Epic: Emergent character behavior (14 sub-items, 4 phases)
- **casehubio/neocortex#398** — Memory seeding infrastructure
- **casehubio/neocortex#399** — Deductive goal formation
- **casehubio/neocortex#400** — Cognitive section calibration
- **casehubio/neocortex#401** — Standardised experience-to-behaviour schema
- **casehubio/neocortex#402** — CognitiveEmergenceTest framework

## Branch State

- `issue-076-directive-minimal-yaml-rewrite` — stamped closed, landed as ff5acaf
- `fix/076-taxonomy-cleanup` — merged to main as bca77d5
- `chore/076-prompt-inspection` — merged to main as 2abf0c3
- `run/old-briefings-baseline` — temp branch with stashed old YAML files, can be deleted

## Key Design Decisions (in workspace specs/)

- Content type taxonomy: 10 types, each with one home (spec + decisions.md)
- Tendencies = personality-stable behavioral patterns (always active), distinct from norms (contextual social rules)
- Templates carry voice only — behavioral content moved to social-config
- Emergence-first principle: prescribe less, let drives/goals/beliefs produce behavior
- Future vision: memory-seeded personality emergence, clinical framing, standardised experience schemas
