---
layout: post
title: "The Test That Was Missing"
date: 2026-11-11
entry_type: article
subtype: diary
projects: [casehubio/examples, casehubio/neocortex]
tags: [cognitive-architecture, testing, evaluation, pipeline-architecture, CAPS, integration-testing, agent-design]
---

# The Test That Was Missing

We had unit tests. Seven hundred of them, running in thirty seconds, covering every cognitive component in isolation — drives, appraisal, regulation, mood, MindMap seeding, CAPS settling. Each test stubs the pipeline, verifies its own component's logic, and moves on. We also had the 300-tick scenario run: the full production pipeline with seventeen characters across six rooms, taking four hours and producing transcripts that classify_emotions.py scores for drive coherence. Both are valuable.

Neither catches a broken pipeline.

The unit tests prove that `MoodOrchestrator` computes mood correctly when given the right inputs. They don't prove the inputs arrive. The 300-tick run proves the whole thing works — eventually, if you have four hours and are willing to read three hundred events looking for the moment something went wrong. Between these two extremes, there was nothing.

![The gap in the testing landscape](images/testing-gap-landscape.svg)

I discovered this gap the hard way. A previous session exposed several silent failures: `bridgeToMood` never firing because the mock participant wasn't registered. A `generate()` override with a signature mismatch that compiled fine against the stub but threw at runtime. An appraisal strategy that parsed JSON correctly in the unit test but choked on code fences in real LLM output. Each of these passed every unit test. Each broke the actual pipeline. Each was only found by running the full scenario and watching characters produce flat, emotionless responses.

## What the Smoke Test Does

The smoke scenario is a `@QuarkusTest` tagged `llm-eval` that sits in the gap. Three agents — Vincent Marsh (scheming antagonist), James Hartwell (protective protagonist), Clara Bellingham (social mediator) — run through the full cognitive pipeline for twenty-five ticks.

Full pipeline means full pipeline. `ManorCognitivePipelineBuilder` wires up real components: `CognitionCore.tick()` runs appraisal and regulation. Drive orchestration reads real MindMap state. The observation builder renders real beliefs, real norms, real cognitive sections derived from real situation classification. `SessionPipeline` sends real prompts and gets real responses. Actions resolve against real world state. Experience ingest writes real memories.

![One tick of the smoke scenario](images/smoke-scenario-pipeline-flow.svg)

Six scripted events — injected at specific ticks — span the emotional range the characters need to navigate. Arrival sets the social context. A torn letter discovered in the library triggers Vincent's scheming. James notices Vincent hiding it. A direct confrontation tests the conflict dynamics. A falling bookcase puts Clara in physical danger, activating James's protection drive. Resolution reveals the letter was a century-old document, not a current threat, and the tension recalibrates.

After the run, an LLM judge scores fifteen sampled events for drive coherence on a 1-5 scale. The test asserts: mean score at least 3.5, no individual event below 2.0.

## What the First Run Showed

The first run produced 137 events across 25 ticks, evenly distributed across the three characters. The judge scored fourteen events at 5.0 and one at 4.0. Mean: 4.93.

That's not just above threshold. That's the pipeline producing personality-coherent responses — characters with distinct drives behaving consistently across situations — without any of the mocking and stubbing that makes unit tests easy. The cognitive pipeline works end-to-end. Formation memories survive consolidation. Drives activate from MindMap state. Appraisal shapes what reaches the LLM. The characters feel like themselves.

More interesting than the score was what the test exposed about infrastructure.

## The Infrastructure Findings

Building the test required fighting CDI resolution failures that had been invisible. `InformationGapDetector` — a class used by `CuriosityDrive` — had no CDI scope annotation. `CuriosityDescriptionGenerationPhase`, an `@ApplicationScoped` consolidation phase, injected it directly. Without the annotation, Quarkus CDI validation fails at boot. This was masked because no test had previously booted the full CDI graph with all cognitive components active. The fix was a single `@Dependent` annotation, but the failure had been silently blocking all `@QuarkusTest` execution.

A separate dependency issue: `CognitionCoreFactory` references `EntityIdentityCache` from the `entity-token-cdi` module, but that module is declared `<optional>true</optional>` in the cognition POM. Every downstream consumer must independently discover and add it. The fix for wacky-manor was explicit — add the dependency. The proper fix is making it non-optional in neocortex.

The most consequential finding was in `LlmSituationClassifier`. Every call to `classify()` opens a new LLM session with a random UUID:

```java
var sessionId = "classifier-" + java.util.UUID.randomUUID();
var session = pipeline.openSession(sessionId, init);
```

Three agents, twenty-five ticks, two classification calls per agent per tick. That's a hundred and fifty cold session opens, each paying the full system prompt cost for the CAPS topology. The test ran forty-four minutes. Most of that time was spent repeating the same system prompt to the same model for the same agent. A sticky session per agent — `"classifier-" + agentId` — would amortize the system prompt and likely cut the runtime by more than half.

This isn't just a classifier problem. A full audit of every LLM interaction point revealed something worse.

## The Cross-Agent Contamination Bug

The cognitive pipeline has sixteen components that make LLM calls: appraisal, regulation, inner life, mental model, user model, strategy learning, narrative summarisation, mood appraisal, BDI extraction, and more. We audited all sixteen.

Fourteen of them used static session IDs. `"emotion-regulation"`. `"inner-life"`. `"mood-appraisal"`. `SessionPipeline.openSession` uses `computeIfAbsent` — if a session with that ID already exists, it returns the existing one. Which means when Agent B calls `openSession("emotion-regulation")`, it gets Agent A's session. Agent A's regulation narrative — the specific coping strategy it selected for shame, the appraisal it made of the confrontation — is sitting in the context window when Agent B processes its own entirely different emotional situation.

This is not a performance problem. This is a correctness bug. Every multi-agent scenario run to date has had cross-agent cognitive contamination. Vincent Marsh's scheming appraisal is in the context when Clara Bellingham's social-harmony regulation runs. James Hartwell's protection drive narrative is visible when Vincent's dominance drive is being evaluated.

The fix is mechanical: append `agentId` to every session ID. `"emotion-regulation"` becomes `"emotion-regulation-" + agentId`. Ten of sixteen components had `agentId` available at the call site and were fixed immediately. Six need an API change to thread `agentId` through the appraisal chain.

The 4.93 score from the smoke test was produced under contaminated conditions. A clean run with per-agent sessions will establish the true baseline — the score could go up (removing noise from cross-agent pollution) or down (the contamination may have been accidentally helpful). Either way, the measurement will finally be honest.

![Contaminated vs isolated session flow](images/session-contamination-flow.svg)

## What This Means for Multi-Agent Cognitive Architecture

If you're building multi-agent systems with shared LLM infrastructure, audit your session lifecycle. The pattern is common: a component opens a session with a descriptive name, uses it, and moves on. When only one agent runs, this is fine. When multiple agents share the pipeline, every descriptive name becomes a shared context window. Agent isolation requires agent-scoped session IDs — not just for performance, but for cognitive integrity.

The smoke test didn't just find a testing gap. It found a fundamental violation of agent cognitive isolation that had been invisible because no prior test ran multiple agents through the real pipeline at a scale where the contamination accumulated enough to matter. Unit tests use stubs. The 300-tick run uses per-agent sessions in `PlaybookOrchestrator` for character responses but not for the cognitive components that feed into those responses.

The forty-four minute runtime is a feature, not a bug. It's long enough to see propagation effects that unit tests can't reveal: how drive intensities drift over consecutive ticks, whether memory accumulation changes the observation builder's output, whether regulation strategies compound correctly across emotional episodes. These are the dynamics that matter for believable cognitive agents, and they only manifest at scale.

Twenty-five ticks. Three characters. One gap, closed. One contamination bug, found.
