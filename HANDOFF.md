# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-27
**Branch:** main (all work landed)

## What was done this session

### Issue #154 — Thinking-field quality eval (CLOSED)

Built a multi-dimensional eval instrument measuring character expression quality across three LLM-scored dimensions (quality, differentiation, register) plus mechanical analysis (temporal windowing, co-occurrence, proportional flagging).

### The BASELINE0 Reappraisal

**The key finding this session:** B0's 3.93 flat mean was report-mode expression. Re-scoring B0 with the full eval revealed register 1.0-1.7 across all drives — characters stating conclusions, never discovering feelings. PP gallantry in B0: quality 2.40, register 1.0. The target we chased for a month was never as good as we thought.

### Three-Way Comparison (B0 → B1 → V2)

30-event full eval on each:

| Drive | Metric | B0 (briefed) | B1 (L1 echo) | V2 (full pipeline) |
|---|---|:-:|:-:|:-:|
| HC scheming | quality / register | 4.50 / 1.5 | 5.00 / 3.8 | 5.00 / 4.0 |
| PP gallantry | quality / register | 2.40 / 1.0 | 3.00 / 1.2 | 4.00 / 3.4 |
| AHM protection | quality / register | 3.83 / 1.7 | 4.75 / 3.8 | 4.50 / 3.7 |
| DD gloating | quality / register | 3.50 / 1.5 | 4.00 / 2.5 | 2.40 / 2.8 |
| DD recognition | quality / register | 3.83 / 1.0 | 3.88 / 1.1 | 3.33 / 1.5 |

### Eval instrument (`classify_emotions.py`)

Redesigned with `full` mode (now default):
- **Three dimensions**: quality, differentiation, register (discovery vs reporting)
- **Tighter relevance gate**: "would removing this drive change behavior?" — dropped false triggers
- **Event type split**: per-drive scores broken out by dialogue/aside/thinking
- **Co-occurrence matrix**: quality of each drive when both are active simultaneously
- **Temporal windowing**: 10 segments with trend detection
- **Proportional flagging**: sparse, high-variance, degrading, volatile, never-triggers

Backward compatible — `--mode flat` and `--mode relevance` unchanged.

### Cross-repo: Era 4 journal (neocortex)

Two entries in `neocortex/docs/research/era4-measurement-methodology-research.md`:
- Entry 1: thinking-field eval instrument, historical baseline, attribution analysis
- Entry 2: B0 reappraisal, three-way comparison, five improvement targets, iterative methodology

### Blog

`2026-10-27-mdp02-measuring-what-the-architecture-produces.md` — covers the month's trajectory, B0 reappraisal, and what's next.

### Issue #155 created

Configurable thinking-field mode (V2 verbose vs V3 condensed). Code change pending IntelliJ.

## Five Improvement Targets (next session focus)

| # | Target | Quality | Register | Diff | What to check |
|---|--------|:-------:|:--------:|:----:|---------------|
| 1 | DD gloating | **2.40** | 2.8 | 2.4 | Formation memories — does DD have gloating-forming experiences? |
| 2 | DD recognition | 3.33 | **1.5** | 2.8 | Barrett 4-channel description — too cognitive? |
| 3 | PP social-harmony | **2.83** ↓ | 2.8 | 3.7 | Why degrading within runs? Context filling? |
| 4 | HC dominance | **3.17** | **2.5** | 3.0 | Needs more somatic grounding in drive description |
| 5 | PP proving-worth | 3.40 | **2.8** | **2.8** | Discrimination text vs gallantry — distinctive enough? |

### Iteration methodology

Short cycles, not 300-tick runs:
1. Check cognitive content for target drive (formation memories, drive description, discrimination texts)
2. Make a targeted change
3. Run short scenario (3-10 ticks)
4. Score with `classify_emotions.py <dir> --mode full --limit 30`
5. Compare to baseline (B0/B1/V2 30-event results in each run's `full-eval-*.json`)

~10 minutes per iteration cycle.

## Repo state

- **examples:** main, clean. Pushed to origin.
- **neocortex:** main, clean. Era 4 Entry 2 committed.
- **wsp-casehub:** blog updated, handoff updated.
- **Server:** wacky-manor may still be running on port 8180. Check before starting.

## Key references

- Era 4 journal: `neocortex/docs/research/era4-measurement-methodology-research.md`
- Eval script: `wacky-manor/docs/eval/classify_emotions.py` (use `--mode full`)
- Eval README: `wacky-manor/docs/eval/README.md`
- B0 full eval: `wacky-manor/docs/eval/generic-colloquial-20261002/full-eval-test.json`
- B1 full eval: `wacky-manor/docs/eval/emotional-core-baseline1-generic-20261004/full-eval-v2.json`
- V2 full eval: `wacky-manor/docs/eval/full-cognitive-run-2026-10-26/full-eval-v2.json`
- Social config (drives): `wacky-manor/src/main/resources/META-INF/eidos/social-config-generic.yaml`
- #155: configurable thinking-field mode (V2/V3)
