---
layout: post
title: "When Your Character's Personality Works — Except When It Doesn't"
date: 2026-08-05
type: phase-update
entry_type: note
subtype: diary
projects: [wacky-manor]
tags: [eidos, personality, calibration, experiment]
---

The Eidos personality framework has a disposition encoding pipeline that actually works. Give it Peter Perfect's ENFJ profile — dominant Fe, auxiliary Ni — and the rendered system prompt reads as an ENFJ. Seven or eight times out of ten, an independent LLM judge classifies the prompt's personality correctly across all four MBTI dimensions.

The problem surfaces when you add the character briefing.

Peter Perfect's briefing describes him as "gallant, volunteering, tunnel vision on Penelope." That language reads as Perceiving — spontaneous, reactive, chasing what catches his attention. His disposition says Judging — structured, planful, methodical. The LLM sees both signals in the same system prompt and has to reconcile them. For feeling and sensing functions, the briefing reinforces the disposition. For thinking and intuition, the briefing overwhelms it.

This is a calibration problem, not an architecture problem. The distinction matters. An architecture problem means the machinery is wrong — you need to redesign. A calibration problem means the machinery is sound but the inputs are fighting each other. Fix the inputs, or amplify the signal you want to win.

We built three tools to address this.

## The Coherence Judge

A `BriefingCoherenceJudge` that takes a character's raw briefing text and their Jungian function stack, sends both to an LLM judge, and gets back per-function coherence scores plus a list of specific textual tensions. Peter Perfect's briefing would flag a HIGH tension on the J/P dimension — "volunteering" and "tunnel vision" contradict the structured planning his ENFJ expects.

The judge self-guards against non-Jungian vocabularies. Pass it a baseline descriptor with simple axes like `socialOrient: collaborative` and it returns a sentinel — no wasted API call, no misleading result. It lives in `casehub-eidos-eval` alongside the existing `MbtiAlignmentJudge` and `FunctionActivationJudge`, following the same CDI pattern.

## The Experiment Matrix

The existing experiment runs 5 characters across 4 disposition layers (baseline, Jungian MBTI, Belbin team roles, composite). We extended it with a briefing richness dimension: empty (no briefing at all), name-only ("You are an agent named The Hooded Claw"), name-plus-role ("You are The Hooded Claw, a villain and secret nemesis"), and rich (the full current briefing). That gives a 4×4 matrix — 16 cells, each measured with function activation TAA.

The matrix answers the question the existing experiment couldn't: does the disposition framework contribute signal independently, or is the rich briefing doing all the work? If TAA improves as briefing decreases, the framework works but the briefing drowns it out. If TAA stays flat, the encoding itself is too weak.

## Three Strengthening Mechanisms

For the case where the framework works but needs amplification, we built three mechanisms to test:

**Format constraints** append a function-aligned response structure to the system prompt. A Te-dominant character gets "Structure your responses as numbered action plans with explicit criteria for success." An Fe-dominant character gets "Frame your responses around how actions affect the group."

**Observation directives** inject a cognitive approach section into the observation context — the per-turn situational text the character receives. "As a systematic strategic thinker, prioritise structured execution over creative exploration."

**Schema reinforcement** parameterises the thinking field description in the response JSON schema. Instead of generic "your internal reasoning," a Te-dominant character gets "(describe your systematic analysis — what options exist, which is optimal, why)."

Each mechanism is measured independently and in combination. The experiment will show which moves TAA the most, whether they're additive, and whether different functions respond to different mechanisms.

The infrastructure is built. The experiment matrix is ready to run. What we don't know yet — and what the data will tell us — is whether Peter Perfect needs a rewritten briefing, a stronger disposition signal, or both.
