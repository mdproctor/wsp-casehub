# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-29
**Branch:** issue-156-computed-mode-quality (examples), main (neocortex)
**Era:** 4 complete → Era 5 ready to start

## Status

Era 4 computed-mode quality work is **complete**. All aggregate metrics exceed V2 prose baseline. A ceiling was identified on reactive drives — requires #477 relational schemas (Era 5) to close.

**DO NOT sync neocortex with canonical main yet.** Slot 211 landed #477 relational schemas on neocortex main. Merge to this slot only when Era 5 begins.

## Results

| Metric | V2 (prose) | 300-tick clean | Delta |
|--------|-----------|---------------|-------|
| Quality | 3.88 | **3.92** | +0.04 |
| Register | 3.35 | **3.74** | +0.39 |
| Differentiation | 3.67 | **3.74** | +0.07 |
| Trigger rate | — | **87%** | — |
| Cartoon contamination | 27+ hits | **0** | — |
| Metacognitive leaks | 27+ hits | **0** | — |

### Reactive drive ceiling (what Era 5 must address)

| Drive | V2 Q | Now Q | V2 R | Now R | Issue |
|-------|------|-------|------|-------|-------|
| HC gloating | 4.00 | 2.40 | 4.50 | 3.20 | No relational trigger |
| Peter protection | 4.60 | 3.80 | 4.00 | 3.80 | Needs threat context |
| DD greed | 4.33 | 3.80 | 3.83 | 3.60 | No situational trigger |
| Peter gallantry | 4.00 | 3.67 | 3.40 | 3.00 | Needs relational context |

Targeted probes confirm the code WORKS: HC gloating 4.33, Peter gallantry 5.00 in ideal scenarios. The gap is in complex, long-context conditions where the relational trigger connection is missing.

## What was done this session

### Fixes (all committed, all tested, 648 fast tests pass)

1. **#165** — Formation memories ungated in computed mode
2. **#164** — GENERIC agentId leak (6 sites + ExchangeRunner)
3. **Drive type labels** — `GLOATING (0.7):` prepended to generated descriptions (neocortex)
4. **Immersion overhaul** — section headers, action results, thinking instruction
5. **ID rewrite at import** — cartoon IDs eliminated from runtime in GENERIC mode
6. **EXAMINE affordance** — INTERACT → EXAMINE in both ActionDescriptor and entity Affordance
7. **ActionType.verb()** — enum name never reaches LLM-visible text
8. **ExchangeRunner** — PULL_ASIDE events now use display names
9. **Non-active characters** — marked inactive, don't appear in observations
10. **Auto-pause on API failure** — health-check recovery every 30s (#172)

### Key finding: the hybrid

Label + degree + somatic prose is the working combination:
- Label alone → simulation awareness (Era 1 P13, Era 4 Oct 6-9)
- Somatic alone → quality 3.76, drives lack conceptual anchor
- **Hybrid → quality 3.99, zero metacognitive leaks**

The somatic text is protective — it prevents the label from triggering metacognitive reasoning.

### Issues filed

| # | Repo | Title |
|---|------|-------|
| #167 | examples | Eval: metacognitive awareness check |
| #168 | examples | Eval: action-format awareness check |
| #169 | examples | Rename INTERACT affordance label (DONE) |
| #170 | examples | Character name tokenization |
| #172 | examples | Resilient scenario execution |
| #173 | examples | Session auto-reconnect after eviction |
| neocortex #721 | neocortex | Unified entity linking (design) |
| neocortex #722 | neocortex | Performance optimization |

### Tests created

| Test | Type | What it checks |
|------|------|---------------|
| `ActionResolverTest` (2 tests) | Unit, fast | INTERACT results are experiential |
| `WorldStateTest` (2 tests) | Unit, fast | ID remapping works |
| `MetacognitiveAwarenessEvalTest` | llm-eval | Simulation awareness in LLM responses |
| `DriveQualityEvalTest` (5 tests) | llm-eval | Targeted drive quality probes |

## Eval data — where everything lives

All eval data is under `wacky-manor/docs/eval/`:

### Transcripts and results

| Directory | Events | Profile | Mode | Notes |
|-----------|--------|---------|------|-------|
| `generic-colloquial-20261002/` | 310 | GENERIC | Briefing | B0 baseline |
| `emotional-core-baseline1-generic-20261004/` | 336 | GENERIC | Briefing | B1 baseline |
| `full-cognitive-run-2026-10-26/` | 303 | GENERIC | Briefing | **V2 baseline** (prose) |
| `computed-engram-run-20261028/` | 48 | GENERIC | Computed | Pre-fix computed (broken) |
| `computed-fixes-run-20261028/` | 103 | GENERIC | Computed | Fix1 — formation memories + names |
| `label-fix-run-20261028/` | 30 | GENERIC | Computed | Label fix — quality 3.99 |
| `barrett-v3-run-20261028/` | 45 | GENERIC | Computed | Barrett V3 (regressed, reverted) |
| `300-tick-opus-baseline-20261028/` | 2300 | GENERIC | Computed | Contaminated (ExchangeRunner leak) |
| `300-tick-clean-opus-20261029/` | 851 | GENERIC | Computed | **Clean baseline** — quality 3.92, register 3.74 |

### Eval result files

Each directory contains:
- `transcript.json` — raw events from the scenario run
- `full-eval.json` or `full-eval-v2.json` — Haiku-judged eval (quality, register, differentiation, trigger rate, trends, flags)
- Some have `transcript-for-eval.json` — ID-remapped version for eval compatibility

### Scorecard

`scorecard.json` — cross-baseline comparison table (B0, B1, V2, Pre-fix, Fix1)

### Eval script

`classify_emotions.py` — the Haiku eval instrument. Three modes: `flat`, `relevance`, `full` (default).
```bash
python3 classify_emotions.py <transcript_dir> --mode full --limit 30 --output <dir>/full-eval.json
```

**Note:** When using ID-rewritten transcripts (generic IDs like `clara-bellingham`), the eval script needs cartoon IDs to match against `social-config-generic.yaml` drive definitions. Use `transcript-for-eval.json` (ID-remapped) or remap before running.

## Research documents

| Doc | Repo | What it covers |
|-----|------|---------------|
| `era4-measurement-methodology-research.md` | neocortex | Entry 4: full session writeup — immersion, hybrid, ceiling |
| `era5-relational-cognition-research.md` | neocortex | Opening doc — #477 hypothesis, measurement plan |

## Blog

`blog/2026-10-29-mdp01-the-label-the-body-and-the-wall.md` — diary entry covering the hybrid finding, immersion audit, reactive drive wall, relational answer.

## Key references

| What | Where |
|------|-------|
| Barrett instruction | `CharacterCognition.java` lines 208-248 (`!briefingMode` block) |
| Drive label rendering | `CognitionObservationBuilder.java` line 139 (neocortex) |
| ID rewrite | `PlaybookOrchestrator.java` lines 188-220 |
| ActionType.verb() | `ActionType.java` |
| Entity slug | `WorldState.slug()` + `WorldState.resolveCharacter()` |
| Auto-pause | `PlaybookOrchestrator.java` after query phase |
| Eval script | `wacky-manor/docs/eval/classify_emotions.py` |
| Scorecard | `wacky-manor/docs/eval/scorecard.json` |
| Consolidation cache | `~/claude/casehub/consolidation-cache/consolidation-cache-generic.json` |

## Next session: Era 5

1. Merge #477 relational schemas from slot 211's neocortex main
2. Delete consolidation cache (schemas change consolidation output)
3. Run 30-event eval — compare reactive drive quality
4. If reactive drives improve → 300-tick validation
5. If not → add explicit trigger conditions alongside relational context
