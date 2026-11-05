---
layout: post
title: "The Gap Between Feeling and Acting"
date: 2026-11-04
entry_type: article
subtype: diary
projects: [casehubio/neocortex, casehubio/examples]
tags: [emotion-regulation, cognitive-architecture, gross-model, character-depth, design, implementation]
series: issue-590-emotion-regulation
---

# The Gap Between Feeling and Acting

Every AI agent that models emotion has the same problem. It can feel — appraise a situation, construct an emotional response, assign intensity and valence. What it cannot do is the thing humans do hundreds of times a day without noticing: regulate that feeling before acting on it.

A customer service agent that detects frustration and immediately escalates. A game NPC that appraises anger and instantly attacks. A conversational companion that senses anxiety and reflexively reassures. These agents are emotionally reactive. They confuse having a feeling with being controlled by it.

Real people are not like this. A person who feels anger at a colleague's remark might suppress it because the meeting isn't the place. Another person might reappraise — "they're under pressure, this isn't about me." A third might leave the room. Same anger, same intensity, three different outcomes. The gap between feeling and acting is where personality lives, where social intelligence operates, and where character depth comes from.

This is the single largest gap in most computational emotion architectures. Not more emotions, not better appraisal, not richer personality models. Regulation. The "I wanted to but held back" moment.

## Why This Matters Beyond Character Simulation

Emotion regulation is not a niche concern for character-driven games. It is fundamental to any system that models agents with internal states.

Multi-agent simulations produce unrealistic dynamics when every agent acts on every appraisal. Without regulation, high-intensity situations cascade — anger begets confrontation, confrontation begets more anger, until the simulation converges on extreme states. Real social groups stabilise because individuals regulate, and they regulate differently based on personality, context, and history.

Conversational AI systems need regulation to model appropriate emotional expression. A therapeutic chatbot that mirrors user distress without modulating its response fails at the core task. An agent that suppresses its own frustration signals when the conversation demands patience — that requires a regulation system, not just a sentiment filter.

Believable NPCs in interactive narrative need regulation to produce the dramatic tension that makes stories work. The villain who restrains their contempt. The ally who swallows fear to stay supportive. The rival who channels jealousy into competitive motivation rather than sabotage. These are regulation strategies, and without a principled model, they're hand-authored responses to specific triggers — they don't generalise, they don't adapt, and they break when the narrative goes somewhere the author didn't predict.

The field of affective computing has the models for this. They are well-formalised, empirically validated, and almost entirely ignored by the AI agent community.

## What the Literature Actually Says

I went into this expecting to find a simple model: feel something, dampen it, done. What I found is a surprisingly well-formalised field with computational models that map directly onto agent architectures.

James Gross's 1998 process model is the foundation. He identifies five families of regulation strategy, each intervening at a different point in the emotion generation sequence:

![Gross Process Model — five strategy families positioned along the emotion generation timeline](images/gross-process-model.svg)

The key insight is the antecedent/response distinction. Situation selection and modification happen before the emotion fully forms — you avoid the party because you know it will make you anxious. Attentional deployment, cognitive change, and response modulation happen after — you're already at the party, feeling the anxiety, and now you need to deal with it.

Gross's 2015 extended model adds a control loop: identify the discrepancy between what you feel and what you want to feel, select a strategy, implement it, monitor whether it's working, loop back if not. This is not a one-shot decision — it's continuous evaluation.

Then things get computational. Bosse, Pontier and Treur (2010) formalised the whole thing as differential equations — homeostatic baselines, persistence factors, adaptation rates. Regulation effectiveness decays with repeated use. An agent who keeps suppressing anger doesn't just feel suppressed anger — the suppression itself degrades over time, and the physiological cost accumulates. This is the kind of dynamic that produces emergent character behaviour — the moment when suppression finally fails.

Petter et al. (2025) added the strategy selection model: high-intensity emotions favour distraction (attentional deployment), low-intensity emotions favour reappraisal (cognitive change). Cognitive load gates the complex strategies — when you're overwhelmed, you can't reframe, you can only look away. This maps directly onto personality-driven regulation.

These are not theoretical sketches. They are quantitative models with published parameters. The gap is not in the science — it's in engineering adoption.

## The Design Choice That Matters Most

We had a fork in the road. The Bosse equations are fully specified — you could implement regulation as pure computation, the way we already do drive intensities. Mechanical. Deterministic. Testable.

But we've been burned before. Mechanical keyword filtering for an earlier system produced output that was technically correct and completely unconvincing. The difference between "anger intensity reduced by 0.35 via response modulation" and a character whose jaw tightens as they force their expression neutral — that difference is the whole point.

So the decision was LLM-first for quality, with the literature providing structured prompt context rather than direct computation. The Bosse model tells the LLM about homeostatic baselines and adaptation. Petter's model tells it about intensity-driven strategy selection. Personality traits tell it about regulation tendencies — high neuroticism means more suppression, high openness means more reappraisal. The LLM integrates all of this and produces both the strategy choice and the embodied narrative.

If empirical testing shows the LLM's strategy selections become formulaic — if it always picks reappraisal for low-intensity anger regardless of personality — that's the signal to mechanise that specific decision. Quality first. Mechanise what's proven.

## Architecture: Regulation as a Decision Layer

The architectural insight came from pushing back on an initial instinct to split environmental strategies (situation selection, modification) from internal strategies (distraction, reappraisal, suppression). They seemed like different integration points — one affects goals, the other affects drive rendering.

But all five strategies share the same trigger. An agent appraises their current emotional state, detects a discrepancy from what they want to feel, and decides to regulate. The decision happens at one point. What differs is where the implementation targets — internal state or the environment.

Regulation is a decision system, not an execution system.

![Regulation pipeline — unified decision layer with distributed execution to drive rendering and goal pipeline](images/regulation-pipeline.svg)

One `EmotionRegulationTickParticipant` in the DERIVED phase selects from all five strategies via an LLM call. Internal strategies (distraction, reappraisal, suppression) flow to `renderInternalState()`, where they modify how drives appear in the agent's prompt — a suppressed gloating drive gets text like "the triumph is there but you hold it back, your jaw tightens with the effort." Environmental strategies (situation selection, modification) flow to the goal pipeline as `DriveGoalProposal` instances, where they become avoidance constraints or situation-change goals.

Same participant. Same LLM call. Same data model. Two downstream consumers. The architecture stays clean because the abstraction is right: decide here, execute there.

## What This Actually Produces

Here is the concrete difference it makes:

![Reactive agent vs. regulated agent — same situation, different behaviour paths](images/reactive-vs-regulated.svg)

A character with high neuroticism encounters Muttley undermining them in front of the group. Appraisal fires — anger at 0.85 intensity, well above the regulation threshold. The regulation participant activates.

The LLM receives: the anger emotion, the character's personality profile (high neuroticism, low openness), the intensity-strategy guidance from Petter (high intensity → distraction), and the last three ticks of regulation history (the character has been suppressing frustration for two ticks already — effectiveness is declining).

It selects response modulation — suppression. The character's "dominance" drive gets regulation-coloured text: the somatic channel shows physical cost (tension, clenched hands), the temporal channel shows duration pressure (this can't last), and the action tendency shifts from confrontation to controlled withdrawal.

Compare that with a high-openness character in the same situation. Same anger. But this character reappraises — cognitive change. The anger is reframed as competitive motivation. The "scheming" drive shifts from hostile to strategic. The narrative feels different because the personality is different, and the LLM has the structured context to produce that difference.

This is what mechanical computation cannot give you. The specific somatic texture of holding back. The difference between someone who suppresses because they can't do anything else and someone who reappraises because they see a better frame. These aren't different parameters to the same equation — they're fundamentally different experiences that need different language.

## The Silent Failure That Wasn't a Failure

The system was complete. The wiring was correct — I proved that with mechanical tests that stubbed the LLM and ran the full tick pipeline: appraisal participant fires, regulation participant reads the result, generation phase produces a strategy. Four tests, all passing. The pipeline worked perfectly with stubbed data.

But when we ran it with actual LLM calls, regulation never activated. The appraisal participant was called. The LLM returned a response. And the result was zero emotions, null narrative. Every time.

The trap was double exception suppression. `LlmAppraisalStrategy.appraise()` wraps its LLM call in a try-catch that returns `AppraisalResult.empty()` on any exception. Then `CognitionCore.safeRun()` wraps the entire participant tick in another try-catch that logs and continues. Two layers of "fail silently and carry on." The system never crashed, never logged an error at a level anyone would notice, never showed any sign of distress. It just quietly produced nothing.

Claude and I built a reproducer — three tests comparing the same backend through three different paths: fire-and-forget `invoke()`, session pipeline `openSession()`, and bare `backend.openSession()`. All three returned valid text. The session layer was not the problem.

The response from `openSession()` started with ` ```json `. Markdown code fences. The Claude CLI wraps JSON responses in code fences when running as a session — a formatting choice that's invisible to the caller until the caller tries to parse the raw response as JSON.

The appraisal strategy passed the raw response directly to Jackson: `MAPPER.readTree(response)`. Jackson hit the backtick prefix and threw `JsonProcessingException`. Caught silently. Empty result stored. Regulation saw empty emotions. Returned early.

The regulation generation phase — the other LLM consumer in the same pipeline — already handled this. Its `parseResponse()` extracted JSON between `{` and `}` before parsing. One component was resilient to formatting variation. The other wasn't. A one-line fix.

The lesson is not "always extract JSON." The lesson is that `catch (Exception e) { return empty(); }` is the worst debugging pattern in a multi-component pipeline. The system appears healthy. The data flows. Nothing crashes. And the component that silently produces nothing looks identical to a component that correctly determined there's nothing to produce. You can stare at the architecture diagram for hours and never find it, because the architecture is correct. The bug is in parsing, not wiring — but the exception handling makes them indistinguishable from the outside.

## The Activation Gate That Wasn't in the Paper

Not every emotion needs regulation. A character who regulates joy at seeing a friend is pathological, not nuanced. The system needs to fire selectively.

The first implementation used what I thought was a Bosse et al. threshold — a binary gate at 0.4 discrepancy from baseline. If your anger is below 0.7 intensity, regulation doesn't fire. Above it, regulation engages fully. On or off.

Then I actually read Bosse. Their model has no binary gate. Regulation effort is continuous and proportional: `v_n = α × d / d_max`. Small discrepancy from your emotional baseline? Small regulation effort — a gentle cognitive reframe, barely noticeable. Large discrepancy? Full-effort regulation — active suppression, deliberate distraction, the strategies that have visible cost.

This matters practically, not just theoretically. The EmotionMapper that converts appraisal dimensions into emotion intensities uses a three-factor product for anger: `relevance × |conduciveness| × controllability`. An empirical survey across all possible dimension combinations showed that even extreme scenarios — betrayal, mortal danger — produce anger intensities of 0.13 to 0.24 when the appraisal LLM returns moderate conduciveness values (which it does, correctly, because the SEC model interprets conduciveness as "how much does this advance or block your goals" — a betrayal already happened, the discovery doesn't change the damage).

The binary gate at 0.7 made regulation structurally unreachable for anger. It was dead code wearing the citation of a paper that never recommended it.

![Regulation effort: binary gate vs Bosse proportional model — the binary gate misses the entire operating range of typical LLM appraisals](images/bosse-proportional-regulation.svg)

The replacement is the actual Bosse formula. Regulation effort scales linearly with emotional displacement. A noise floor at 0.01 prevents LLM calls when emotions are literally at baseline — a cost optimisation, not a theoretical parameter. The `α` factor (willingness to regulate) is personality-dependent: high neuroticism means higher α, meaning the same emotional discrepancy produces more regulation effort. That's a parameter with published empirical basis, not a magic number.

The lesson generalises beyond this specific system. When you adopt a model from the literature, adopt the model — not a simplified version that looks plausible. Read the equations, not just the abstract. A binary threshold is easier to implement than a proportional formula, and it's wrong in a way that's invisible until you run the empirical survey and discover your system never activates.

## Regulation History: Continuity Across Ticks

A regulation system without memory is a regulation system without consequences. If a character suppresses anger on tick 5 and the system has no memory of that on tick 6, it cannot model suppression fatigue, strategy switching, or the dramatic moment when regulation fails.

Every regulation result is persisted to the MindMap as a cognitive node — strategy, target emotions, effectiveness, narrative. The next regulation tick loads the last five results and includes them in the LLM prompt. The LLM can see that this character has been suppressing for three consecutive ticks with declining effectiveness, and choose a different strategy — or produce a narrative where the suppression finally breaks.

This is where the emergent behaviour lives. Not in the architecture diagrams or the strategy enums, but in the temporal continuity that lets regulation evolve within a scene.

## What the Community Should Take From This

The affective computing literature has been producing well-formalised, empirically validated models of emotion regulation for over twenty-five years. Gross 1998, Bosse 2010, Petter 2025 — these are not speculative. They are quantitative, they are computational, and they produce measurably different agent behaviour when implemented.

The AI agent community has largely ignored this work. Multi-agent systems model beliefs, desires, intentions, plans. Some model emotions. Almost none model what happens between feeling an emotion and acting on it. The result is agents that are sophisticated reasoners and emotional toddlers — they feel, therefore they do.

Three specific things that any agent system with emotional modelling should consider:

**1. Regulation is not suppression.** Five strategy families, each with different cognitive demands, personality correlations, and downstream effects. Reducing regulation to "dampen the intensity" discards most of the model and most of the behavioural richness.

**2. Strategy selection is personality-dependent.** Neurotic individuals favour suppression. Open individuals favour reappraisal. Conscientious individuals favour situation selection. If your agents have personality models, the regulation literature tells you exactly how those personalities should modulate emotional expression — and the mappings are empirically validated, not intuitive.

**3. Regulation has temporal dynamics.** Effectiveness decays with repeated use. Suppression accumulates physiological cost. Strategy switching happens when the current strategy stops working. These dynamics produce the emergent moments that make agent behaviour surprising and interesting — but only if regulation has memory and continuity.

The implementation demonstrated here is specific to the CaseHub cognitive architecture, but the principles are general. Any system that appraises emotions can add a regulation phase. Any system that renders agent state into prompts or behaviour can modify that rendering based on regulation results. Any system that tracks agent goals can create regulation-motivated goals for environmental strategies.

The science exists. The computational models exist. The engineering is straightforward. The gap is adoption.

## The Intensity Problem: When Your System Works But Nobody Can Tell

The micro-scenarios are verified. Betrayal routed through shame. Mortal danger through fear. Public humiliation through reproach. Each pathway fires regulation and selects an appropriate strategy — cognitive change for moderate anger, attentional deployment for high fear, response modulation for public shame. The mechanical pipeline tests prove the wiring. The LLM eval tests prove the strategy selection.

Then we ran the real scenario. Three hundred events. Five characters. A manor full of secrets, schemes, and interpersonal tension. Regulation enabled.

The classification report came back: sparse drive triggering, most characters at zero triggered events. No visible difference between the regulation-enabled run and the baseline. Regulation was firing — the tick tests proved that — but the EmotionMapper was producing intensities so low that the classifier couldn't distinguish regulated behaviour from unregulated behaviour in the dialogue output.

The culprit is the intensity formula. The EmotionMapper uses different factor counts for different emotions. Anger is a three-factor product: `relevance × |conduciveness| × controllability`. Shame and reproach are two-factor: `relevance × (1 - standards)`. With the same LLM appraisal dimensions, shame produces double the intensity of anger.

![Emotion intensity by mapping formula — the three-factor anger product systematically underestimates compared to two-factor shame and reproach](images/intensity-compression.svg)

With a typical moderate appraisal — relevance 0.7, conduciveness -0.6, controllability 0.55 — anger computes to 0.23. That's below the 0.3 baseline. No discrepancy. No regulation effort. The character should be angry about a betrayal, but the formula says they barely register it. Meanwhile, shame from the same appraisal hits 0.46. Same situation, same LLM assessment, but the two-factor product reaches the operating range where regulation matters and the three-factor product doesn't.

This is not a bug in regulation. It's a bug in the emotion-to-intensity mapping that predates the regulation system. But regulation makes it visible, because regulation is the first consumer that actually cares whether emotions cross a meaningful intensity threshold. Drive rendering will show "anger" regardless of intensity. Regulation needs intensity to exceed the baseline before it has anything to regulate.

The fix is in the EmotionMapper — either a weighted combination instead of a product, or the OCC model's max-based formula. But the deeper lesson applies to anyone building on appraisal-based emotion models: if you multiply three normalised values, the result converges toward zero. Every factor you add compresses the range. Two factors give you useful intensity variation. Three factors give you a system that never activates.

## What's Next

The intensity mapping needs fixing before regulation becomes visible in full scenario runs. Once emotions reach the operating range consistently, the scenario itself needs engineering — not scripted triggers, but a format change.

The current scenario is open-ended exploration. Characters wander rooms, examine objects, occasionally talk. Emotional confrontation is possible but not structural. The plan is to adopt a Traitors-style game format: day phases where characters move through spatial rooms and make discoveries, night phases where the villain acts, and roundtable discussions where every character must publicly accuse or defend. The format guarantees emotional confrontation at regular intervals — you can't avoid the roundtable, and you can't stay silent during it.

The spatial model matters too. Right now a character is "in the library" — a flat location. Moving to x-positions within rooms creates walk-past triggers (you hear the beam creak as you pass under it) and examine triggers (you open the safe and find the documents). One dimension is enough — the room is a corridor, and every character who walks from one end to the other must pass through every trigger zone. No spatial luck, no near-misses.

But the regulation system itself has four missing pieces that change the quality of what it produces:

**Suppression fatigue.** Bosse models physiological cost accumulation from repeated regulation. A character who suppresses anger for ten ticks should experience degrading regulatory capacity — not just declining effectiveness per the LLM's judgment, but a mechanical reduction in their ability to regulate at all. The dramatic moment this produces: suppression finally fails. The held-back emotion breaks through. That's emergent from the fatigue model, not scripted.

**Automatic regulation.** Most regulation decisions are predictable from personality. High neuroticism → suppression. High openness → reappraisal. These habitual responses don't need an LLM call. Handling the predictable cases mechanically reserves LLM reasoning for the complex situations — novel contexts, conflicting personality signals, the moments where the interesting choices happen. It also cuts per-tick latency significantly.

**Proactive regulation.** All current regulation is reactive — it fires after appraisal detects an emotional discrepancy. But situation selection is fundamentally proactive: "I know that room will make me angry, so I avoid it." A character who was humiliated in the library last round should feel reluctance to enter the library this round — not because appraisal fires (they haven't entered yet), but because emotional memory drives avoidance. This connects regulation to the spatial model — the avoidance has somewhere to go.

**The monitoring loop.** Gross 2015 adds continuous re-evaluation: is the current strategy working? If a character has been suppressing for three ticks with declining effectiveness (0.6 → 0.4 → 0.2), the system should recognise suppression is failing and switch — to cognitive change, to attentional deployment, to something that costs less. Combined with suppression fatigue, this produces the escalation dynamics that make regulation feel alive rather than mechanical.

Social regulation remains the most dramatically interesting extension. A character who confides in a friend to manage anxiety. A character who deliberately provokes someone to manage their own boredom. A character who withdraws from interaction to cool down. These interpersonal strategies require the intrapersonal system to exist first — you need to know what you're feeling before you can decide to seek comfort for it.

Eventually, the stable patterns get encoded as CAPS nodes — the attractor dynamics that make characters settle into personality-consistent regulation styles without per-tick LLM calls. That's the consolidation step: observe what the LLM does, validate it empirically, then mechanise the parts that become formulaic. Quality first. Mechanise what's proven.

---

*References: Gross, J.J. (1998). The emerging field of emotion regulation. Review of General Psychology, 2(3), 271-299. Gross, J.J. (2015). Emotion regulation: Current status and future prospects. Psychological Inquiry, 26(1), 1-26. Bosse, T., Pontier, M. & Treur, J. (2010). A computational model based on Gross' emotion regulation theory. Cognitive Systems Research, 11(3), 211-230. Petter, J. et al. (2025). Emotion regulation, fast or slow. Emotion, 25(5), 1273-1292.*
