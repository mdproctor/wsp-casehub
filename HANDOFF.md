# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-31
**Branch:** main (all repos merged and pushed)
**Issue:** #175 — CLOSED. Pipeline validated.

## Status

**Barrett v5 pipeline validated.** Micro-tests (8 scenarios, 2 characters, 3 forms) score all drives Q≥4 and all 8 cognitive dimensions 4-5 on Opus CLI classifier. Zero simulation contamination.

**300-tick eval completed** (336 events, saved at `docs/eval/era5-300tick-20261031/`). Reveals two architectural gaps micro-tests can't catch:

| Weakness | Evidence | Cause |
|----------|---------|-------|
| **Echo loss** | Body vocab 75%→51% over 300 ticks | Barrett instruction buried under growing context |
| **P22 arrival spike** | Analytical narration 45%→75% at tick 50 | P22 descriptions add cognitive content, model shifts to analysis |
| **No body vocab** | 44% of events have zero body words | Combined effect of echo loss + P22 quality |
| **Analytical narration** | 53% of events use "I need to", "I should" | Barrett instruction not reinforced at generation time |

**Direction: architecture, not tuning.** More 300-tick runs are pointless — they'll show the same curve. The investment is in context management and P22 quality.

## Orientation — Start Here

Read these in order:

1. **Barrett v5 instruction** — `CharacterCognition.java` lines 237-258. This is what's in the codebase now. Key additions from v3: dialogue/aside form distinction, blend drives, own words (palette not script), anti-conclusion-about-others, performance cost, drive suppression.

2. **Golden prose target** — `docs/eval/golden-prose-20261030/transcript.json`. The full standard. Key structural patterns:
   - HC asides: bold anchors + multi-paragraph (HC-specific — DD/PP never use bold anchors)
   - HC dialogues: single dense paragraph, no bold anchors, ~200-230 words
   - DD/PP everything: continuous flow, single paragraph, no bold anchors
   - Bold anchors are CHARACTER-SPECIFIC (HC's compartmentalizing mind), not event-type-specific

3. **The v5c confrontation** — `docs/eval/barrett-v5-confrontation-no-gloating/transcript.json`. This is near-golden quality: "Hands want to close. Hands want to become fists. Hands stay open, soft, Pemberton's hands." Achieved by removing gloating drive from the prompt.

4. **All smoke tests** — `PipelineToRenderingTest.java`, methods `goldenScenario*`. Three tests: HC aside (event 1), HC dialogue (event 0), HC confrontation (event 5). All render the full prompt and dump it for inspection. ~5s each.

## What This Session Did

### Barrett Instruction Iteration (v3 → v5)

| Version | What Changed | What Improved | What Remained |
|---------|-------------|---------------|---------------|
| **v3** (previous session) | Scene-discovery over drive-catalog | Q/R/D ≥ 4.0 on classifier | Bold anchors in dialogue, P22 echo, cataloguing, no performance cost |
| **v4** | Renamed ASIDE→THINKING, dialogue/aside distinction, anti-catalog, anti-echo, anti-conclusion example | Structure fixed, own words, drive blending | Conclusions about others, gloating under threat, no performance cost |
| **v5** | Anti-conclusion-about-others rule, performance cost instruction, drive suppression clause | Performance cost shown ("muscles holding the smile"), some conclusions eliminated | Gloating still appears under threat, occasional analytical moments |
| **v5c** (experiment) | Removed gloating drive from prompt entirely | Near-golden output — pure survival + performance cost | Proves fix is pipeline-level, not instruction-level |

### Smoke Tests Created

- `goldenScenario_hcDialogue_entranceHall_fullPromptDump` — event 0, all guests present
- `goldenScenario_hcConfrontation_entranceHall_fullPromptDump` — event 5, only Brixton Boys, threat scenario

### Eval Transcripts Saved

- `barrett-v3-dialogue/` — v3 instruction, dialogue event 0
- `barrett-v3-confrontation/` — v3 instruction, confrontation event 5
- `barrett-v5-confrontation-no-gloating/` — v5c experiment, gloating drive removed

## What This Session Built

### Comprehensive cognitive evaluation instrument
`classify_emotions.py` now scores 4 per-drive dimensions + 8 cross-cutting cognitive dimensions (channel coverage, attachment coloring, theory of mind, formation echo, performance cost, person-specific shifts, drive blending, spatial grounding). CLI backend (`--backend cli`) routes through Opus via `drive-classifier` agent.

### 8 micro-test scenarios validated (all Q≥4 on Opus)
HC library solo, HC drawing room (all agents), HC dialogue (all agents), DD entrance dialogue, DD gallery wounded, DD kitchen gloating. Plus clean pipeline-only tests proving no coaching needed.

### Pipeline fixes
- Deleted 17 hand-authored drive descriptions from social-config-generic.yaml (forces P22 dependency)
- Cache drift detection in PlaybookOrchestrator (warns when stale cache skips P22)
- CLASSIFY_MODEL env var override for classifier model selection

### Issues closed: #175, #174, #165, #164, #156, #121, #100

## Next Step — Architecture Investment

**Do NOT run more 300-tick evals.** The degradation curve is measured. Fix the architecture.

### Priority 1: Context reset with three-tier memory
- **examples#183** — per-character context reset synced with consolidation
- **neocortex#737** — three-tier memory: structured brief + RAG chat logs + sub-agent deep dive
- Fix: periodic context clear → summariser produces structured brief → fresh Barrett instruction at top of short context → body vocabulary stays at 75%

### Priority 2: Short-run reproduction tests
- **examples#182** — integration test using real pipeline (not hand-seeded), one consolidation tick, classify output
- Reproduce weaknesses in 5-10 ticks from a primed scenario state
- Use saved consolidation cache to skip 50-tick warmup

### Priority 3: P22 description quality
- **neocortex#735** — formation echo in dialogue-form descriptions
- **neocortex#734** — dynamic drive activation (suppress under threat)

### All open issues from this session
| Issue | Repo | Title |
|:---:|------|-------|
| #177 | examples | Comprehensive cognitive instrument as CI gate |
| #178 | examples | Two-tier classification (Haiku→Opus) |
| #179 | examples | DD formation echo weakness |
| #180 | examples | Consolidation cache drift detection |
| #182 | examples | Integration test: P22 quality after consolidation |
| #183 | examples | Context reset with async summariser |
| #734 | neocortex | Dynamic drive activation |
| #735 | neocortex | Formation echo in dialogue-form |
| #737 | neocortex | Three-tier context memory architecture |

### Session technique note

Use **subagents** for generation and comparison work. This session pulled too much generated prose into the main context. The pattern should be:
- Main context: instruction design, structural analysis, decision-making
- Subagents: generate thinking, compare against golden prose, report quality scores
- Fork: when doing side-research that doesn't need to stay in context

## What v5 Instruction Fixed (from v3)

| Gap | v3 Problem | v5 Fix |
|-----|-----------|--------|
| Dialogue structure | Bold anchors in dialogue events | "Speaking: one dense paragraph. No anchors, no breaks." |
| Drive cataloguing | One paragraph per drive | "Blend drives. One moment — not three paragraphs." |
| P22 echo | "warmth pools behind sternum" verbatim | "Own words. The drive descriptions are a palette. Do not echo them back." |
| Conclusions about others | "they don't know whose house this is" | "Never state what others think, know, or fail to see." |
| Performance cost | No Pemberton/Vincent split shown | "When performing a face... notice the cost." |
| Drive suppression | All drives always activate | "Not every drive activates. If a territory is quiet, it is simply not there." — works for instruction-level suppression but not enough for presented drives |

## Known Gaps Still Open

- **Gloating under threat** — instruction can't suppress a drive the pipeline presents as active. Needs pipeline fix.
- **Occasional analytical moments** — "which means they rehearsed this" type phrases still appear. Diminishing returns on instruction rules.
- **Formation memory echoes** — golden prose weaves age references; generated output sometimes omits them. May need formation memory section to be more prominent in the prompt.
- **Bold anchor quality** — HC's golden prose anchors on FEELINGS ("Behind my sternum, the private knowledge settles") not just events. Generated anchors tend toward events only.

## Key Files

| What | Where |
|------|-------|
| **Barrett v5 instruction** | `examples/.../agent/CharacterCognition.java` lines 237-258 |
| **Smoke tests** | `examples/.../agent/PipelineToRenderingTest.java` → `goldenScenario*` |
| **Golden prose** | `examples/wacky-manor/docs/eval/golden-prose-20261030/transcript.json` |
| **v5c result (near-golden)** | `examples/wacky-manor/docs/eval/barrett-v5-confrontation-no-gloating/transcript.json` |
| **Classifier** | `examples/wacky-manor/docs/eval/classify_emotions.py` |

## Eval History

| Run | Events | Q | R | D | Notes |
|-----|--------|:-:|:-:|:-:|-------|
| **Golden v12 (target)** | **51** | **4.25** | **4.20** | **4.20** | All per-drive targets ≥4.0 |
| Barrett v3 (aside) | 1 | 4.25 | 4.50 | 4.00 | Scene-discovery instruction |
| Barrett v3 (dialogue) | 1 | 4.50 | 4.00 | 4.25 | Generalises to dialogue |
| Barrett v3 (confrontation) | 1 | 4.50 | 4.50 | 4.50 | SP Q=5 under threat |
| Barrett v5c (no gloat) | 1 | — | — | — | Near-golden structural quality |

## Constraints

- Do NOT remove cognitive sections — they represent abilities
- Isolation principle — one change at a time, measure, keep or revert
- Delete consolidation cache before eval runs
- Install neocortex before running examples
- Use subagents for generation/comparison work — keep main context clean
