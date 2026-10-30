# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), main (neocortex)
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Status

Barrett instruction iterated from v3 → v5. **Full-transcript structural comparison** against golden prose revealed 5 gaps; 4 fixed in instruction, 1 requires pipeline-level dynamic drive activation.

**Key finding:** When gloating drive (0.7 intensity) is present in the prompt, the model activates it even under threat — no instruction can suppress it. Removing the gloating drive from the threat scenario prompt produces near-golden output. **The fix is pipeline-level dynamic drive activation based on situational context, not instruction tuning.**

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

## Next Step — Continue the Event Queue

The Barrett v5 instruction works for non-threat scenarios. Don't block on dynamic drive activation — continue the event queue with events that don't require drive suppression.

**Do this now:**

1. **HC in library (event 2, aside).** HC alone, territorial claim, inscription discovery. No threat → gloating is fine. Create smoke test, generate thinking via subagent, do full-transcript comparison against golden prose (lines 20-25 of `golden-prose-20261030/transcript.json`). Check structure (bold anchors, multi-paragraph), drive blending, own words, performance cost (mask drops when alone). This tests instruction generalisation to a different room.

2. **Different character — DD entrance hall (event 10, dialogue).** Reginald Foxworth arriving. Different drives (recognition, acquisitiveness), different personality (impulsive, reactive). No bold anchors in golden prose — DD experiences as continuous flow. This tests whether the v5 instruction's "you MAY bold" correctly allows DD to skip them.

3. **After those two pass:** file an issue for dynamic drive activation in neocortex (`CognitionObservationBuilder.renderInternalState()` — modulate presented drives based on situational context). That work can be a separate branch.

### Event queue

1. ~~HC aside, entrance hall (event 1)~~ — Q=4.25 R=4.50 D=4.0
2. ~~HC dialogue, entrance hall (event 0)~~ — Q=4.50 R=4.0 D=4.25
3. ~~HC confrontation, entrance hall (event 5)~~ — near-golden with v5c
4. **HC aside, library (event 2)** ← do this next
5. **DD dialogue, entrance hall (event 10)** ← then this
6. File issue for dynamic drive activation (neocortex)
7. Full 300-tick eval after 5+ events at golden standard

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
