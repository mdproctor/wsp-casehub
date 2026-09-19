---
layout: post
title: "Why Your AI Agent Fixates — And What Maslow Can Do About It"
date: 2026-09-19
entry_type: article
subtype: diary
projects: [casehubio/examples]
tags: [cognitive-architecture, maslow, needs-hierarchy, agent-design, wacky-manor]
series: issue-63-fix-cdi-config-beans
---

# Why Your AI Agent Fixates — And What Maslow Can Do About It

Penelope Pitstop helps everyone. That's her character — she has a `social-harmony` drive at 0.8, and every social interaction reinforces it. She mediates conflicts. She encourages the other characters. She reassures the Ant Hill Mob. And she does this all day, every day, while her personal goals pile up and her exploration of the mansion stalls completely.

This is what fixation looks like in an agent cognitive architecture. Give a character one strong drive and a reinforcement mechanism, and it will pursue that drive to the exclusion of everything else. The drive gets reinforced, which makes it stronger, which makes the agent pursue it more, which reinforces it further. A positive feedback loop with no brake.

Real people don't work like this. A person who spends all day socialising eventually feels the pressure of neglected obligations, or gets restless from lack of novelty, or starts worrying about something they've been ignoring. Neglected needs create counter-pressure. Abraham Maslow formalised this in 1943, and it turns out his core insight — that unmet needs generate increasing psychological urgency — maps directly onto the problem of agent drive fixation.

## What Maslow Got Right

Maslow's hierarchy of needs is one of those ideas that everyone vaguely knows and almost nobody applies precisely. The pyramid, in its original form:

<svg viewBox="0 0 760 440" xmlns="http://www.w3.org/2000/svg" style="max-width:760px">
  <rect width="760" height="440" fill="#1a1b26" rx="8"/>

  <!-- Pyramid tiers - bottom to top -->
  <polygon points="380,40 700,380 60,380" fill="none" stroke="#565f89" stroke-width="1.5"/>

  <!-- Tier fills -->
  <polygon points="120,380 640,380 610,340 150,340" fill="#f7768e" fill-opacity="0.15" stroke="#f7768e" stroke-width="1.5"/>
  <polygon points="150,340 610,340 540,280 220,280" fill="#ff9e64" fill-opacity="0.15" stroke="#ff9e64" stroke-width="1.5"/>
  <polygon points="220,280 540,280 480,220 280,220" fill="#e0af68" fill-opacity="0.15" stroke="#e0af68" stroke-width="1.5"/>
  <polygon points="280,220 480,220 430,160 330,160" fill="#9ece6a" fill-opacity="0.15" stroke="#9ece6a" stroke-width="1.5"/>
  <polygon points="330,160 430,160 380,100" fill="#7aa2f7" fill-opacity="0.15" stroke="#7aa2f7" stroke-width="1.5"/>

  <!-- Tier labels - inside pyramid -->
  <text x="380" y="368" text-anchor="middle" fill="#f7768e" font-family="sans-serif" font-size="14" font-weight="bold">SAFETY</text>
  <text x="380" y="318" text-anchor="middle" fill="#ff9e64" font-family="sans-serif" font-size="14" font-weight="bold">TASKS</text>
  <text x="380" y="258" text-anchor="middle" fill="#e0af68" font-family="sans-serif" font-size="14" font-weight="bold">SOCIAL</text>
  <text x="380" y="198" text-anchor="middle" fill="#9ece6a" font-family="sans-serif" font-size="14" font-weight="bold">SELF-EXPRESSION</text>
  <text x="380" y="138" text-anchor="middle" fill="#7aa2f7" font-family="sans-serif" font-size="14" font-weight="bold">UNDERSTANDING</text>

  <!-- Descriptions - right side -->
  <text x="660" y="368" text-anchor="start" fill="#565f89" font-family="sans-serif" font-size="11">avoid threats</text>
  <text x="570" y="318" text-anchor="start" fill="#565f89" font-family="sans-serif" font-size="11">duties, commitments</text>
  <text x="510" y="258" text-anchor="start" fill="#565f89" font-family="sans-serif" font-size="11">trust, belonging</text>
  <text x="455" y="198" text-anchor="start" fill="#565f89" font-family="sans-serif" font-size="11">identity, values</text>

  <!-- Decay rate arrow - left side -->
  <line x1="40" y1="140" x2="40" y2="370" stroke="#565f89" stroke-width="1.5" marker-end="url(#arrowDown)"/>
  <defs><marker id="arrowDown" viewBox="0 0 10 10" refX="5" refY="10" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 5 10 L 10 0" fill="none" stroke="#565f89" stroke-width="1.5"/></marker></defs>
  <text x="38" y="130" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">SLOW</text>
  <text x="38" y="395" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">FAST</text>
  <text x="38" y="260" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10" transform="rotate(-90, 38, 260)">DECAY RATE</text>

  <!-- Title -->
  <text x="380" y="425" text-anchor="middle" fill="#c0caf5" font-family="sans-serif" font-size="13">Agent Needs Pyramid — simulation-pragmatic tiers</text>
</svg>

The core insight isn't the pyramid shape or the specific tiers. It's that **neglected needs create increasing urgency**. A person who hasn't eaten in 12 hours finds it hard to concentrate on philosophy. Someone whose safety is threatened can't focus on self-expression. The urgency gradient is real and observable.

What Maslow got wrong — at least for simulated agents — is the strict blocking. In the original model, lower tiers must be substantially satisfied before higher tiers activate. This is too brittle for a game simulation. A character who feels slightly unsafe shouldn't be unable to pursue curiosity. The real world doesn't work that way either; Maslow himself softened the strict hierarchy in later work.

The other thing that doesn't translate: physiological needs. Agents don't eat, sleep, or breathe. The bottom of Maslow's pyramid is irrelevant. What agents do have are obligations (tasks they've committed to), social relationships (trust they've built or broken), a sense of identity (drives that define who they are), and curiosity (questions they haven't answered).

So I adapted the model. Five tiers, mapped to things that actually matter in a character simulation:

| Tier | What it represents | Decay rate | Why this rate |
|------|-------------------|-----------|---------------|
| **Safety** | Threats, self-preservation | 0.15 (fastest) | Threats demand immediate response |
| **Tasks** | Duties, commitments | 0.10 | Obligations accumulate quickly |
| **Social** | Relationships, belonging | 0.08 | Social bonds erode moderately |
| **Self-expression** | Identity, values, goals | 0.05 | Identity is more persistent |
| **Understanding** | Curiosity, sense-making | 0.03 (slowest) | Curiosity is patient |

The decay rates encode the urgency gradient. Safety decays five times faster than Understanding because a threat demands attention now, while an unanswered question can wait.

## How Satisfaction Flows Through Drives

The first design question was: when an event happens, which need tiers does it satisfy? The obvious approach is a direct mapping — `conflict_resolution → SAFETY + SOCIAL`. But this breaks immediately on character differences.

When Hooded Claw resolves a conflict, he's enacted a scheme. His SELF_EXPRESSION is satisfied. When Penelope resolves a conflict, she's restored harmony. Her SOCIAL need is satisfied. Same event, different psychological meaning. A direct event→tier mapping can't capture this.

The solution was to route satisfaction through the existing drive reinforcement layer:

<svg viewBox="0 0 760 340" xmlns="http://www.w3.org/2000/svg" style="max-width:760px">
  <rect width="760" height="340" fill="#1a1b26" rx="8"/>

  <!-- Event box -->
  <rect x="30" y="130" width="140" height="60" rx="6" fill="#1a1b26" stroke="#7aa2f7" stroke-width="1.5"/>
  <text x="100" y="155" text-anchor="middle" fill="#7aa2f7" font-family="sans-serif" font-size="12" font-weight="bold">conflict_resolution</text>
  <text x="100" y="172" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">pleasure: 0.6</text>

  <!-- Arrow to split -->
  <line x1="170" y1="160" x2="230" y2="160" stroke="#565f89" stroke-width="1.5" marker-end="url(#arrowR)"/>
  <defs><marker id="arrowR" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#565f89" stroke-width="1.5"/></marker></defs>

  <!-- Split point -->
  <circle cx="240" cy="160" r="6" fill="#565f89"/>

  <!-- CLAW PATH (top) -->
  <line x1="246" y1="156" x2="290" y2="80" stroke="#f7768e" stroke-width="1.5"/>
  <rect x="290" y="55" width="130" height="50" rx="6" fill="#1a1b26" stroke="#f7768e" stroke-width="1.5"/>
  <text x="355" y="75" text-anchor="middle" fill="#f7768e" font-family="sans-serif" font-size="11" font-weight="bold">Hooded Claw</text>
  <text x="355" y="92" text-anchor="middle" fill="#c0caf5" font-family="sans-serif" font-size="10">→ scheming drive</text>

  <line x1="420" y1="80" x2="490" y2="80" stroke="#f7768e" stroke-width="1.5" marker-end="url(#arrowR2)"/>
  <defs><marker id="arrowR2" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#f7768e" stroke-width="1.5"/></marker></defs>

  <rect x="490" y="55" width="140" height="50" rx="6" fill="#1a1b26" stroke="#f7768e" stroke-width="1.5"/>
  <text x="560" y="75" text-anchor="middle" fill="#f7768e" font-family="sans-serif" font-size="11" font-weight="bold">scheming → </text>
  <text x="560" y="92" text-anchor="middle" fill="#9ece6a" font-family="sans-serif" font-size="11" font-weight="bold">SELF_EXPRESSION</text>

  <line x1="630" y1="80" x2="690" y2="80" stroke="#9ece6a" stroke-width="1.5" marker-end="url(#arrowR3)"/>
  <defs><marker id="arrowR3" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#9ece6a" stroke-width="1.5"/></marker></defs>
  <text x="725" y="84" text-anchor="middle" fill="#9ece6a" font-family="sans-serif" font-size="11" font-weight="bold">↑ 0.5</text>

  <!-- PENELOPE PATH (bottom) -->
  <line x1="246" y1="164" x2="290" y2="240" stroke="#7aa2f7" stroke-width="1.5"/>
  <rect x="290" y="215" width="130" height="50" rx="6" fill="#1a1b26" stroke="#7aa2f7" stroke-width="1.5"/>
  <text x="355" y="235" text-anchor="middle" fill="#7aa2f7" font-family="sans-serif" font-size="11" font-weight="bold">Penelope</text>
  <text x="355" y="252" text-anchor="middle" fill="#c0caf5" font-family="sans-serif" font-size="10">→ social-harmony drive</text>

  <line x1="420" y1="240" x2="490" y2="240" stroke="#7aa2f7" stroke-width="1.5" marker-end="url(#arrowR4)"/>
  <defs><marker id="arrowR4" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#7aa2f7" stroke-width="1.5"/></marker></defs>

  <rect x="490" y="215" width="140" height="50" rx="6" fill="#1a1b26" stroke="#7aa2f7" stroke-width="1.5"/>
  <text x="560" y="235" text-anchor="middle" fill="#7aa2f7" font-family="sans-serif" font-size="11" font-weight="bold">social-harmony →</text>
  <text x="560" y="252" text-anchor="middle" fill="#e0af68" font-family="sans-serif" font-size="11" font-weight="bold">SOCIAL</text>

  <line x1="630" y1="240" x2="690" y2="240" stroke="#e0af68" stroke-width="1.5" marker-end="url(#arrowR5)"/>
  <defs><marker id="arrowR5" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#e0af68" stroke-width="1.5"/></marker></defs>
  <text x="725" y="244" text-anchor="middle" fill="#e0af68" font-family="sans-serif" font-size="11" font-weight="bold">↑ 0.5</text>

  <!-- Labels -->
  <text x="100" y="30" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">EVENT</text>
  <text x="355" y="30" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">DRIVE REINFORCEMENT</text>
  <text x="560" y="30" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">DRIVE → TIER MAPPING</text>
  <text x="725" y="30" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="10">SATISFACTION</text>

  <!-- Title -->
  <text x="380" y="325" text-anchor="middle" fill="#c0caf5" font-family="sans-serif" font-size="12">Same event, different satisfaction — routed through character drives</text>
</svg>

The key: per-character drive reinforcement mappings already exist. Each character declares which drives are reinforced by which event types. The needs pyramid adds one layer of indirection — a global lookup table mapping drive types to need tiers:

```java
public Map<String, Set<NeedTier>> tierMapping() {
    return Map.ofEntries(
        entry("scheming",          Set.of(SELF_EXPRESSION)),
        entry("self-preservation", Set.of(SAFETY)),
        entry("curiosity",        Set.of(UNDERSTANDING)),
        entry("social-harmony",   Set.of(SOCIAL)),
        entry("adventure",        Set.of(UNDERSTANDING, SOCIAL)),
        entry("gallantry",        Set.of(SOCIAL, TASKS)),
        entry("protection",       Set.of(SAFETY, SOCIAL)),
        entry("loyalty",          Set.of(SOCIAL, TASKS)),
        // ... 13 entries total
    );
}
```

Thirteen entries. That's it. This replaces what would have been a per-character event→tier mapping — 17 characters × 5 event types × 5 tiers = 425 potential entries. The per-character specificity is already handled by the drive reinforcement config. When Hooded Claw's `conflict_resolution` reinforces his `scheming` drive, the pyramid looks up `scheming → SELF_EXPRESSION` and bumps that tier. When Penelope's `conflict_resolution` reinforces `social-harmony`, the pyramid looks up `social-harmony → SOCIAL`. Different drives, different tiers, same 13-entry table.

Some drives map to multiple tiers. `adventure` satisfies both UNDERSTANDING (exploration) and SOCIAL (shared experiences). `protection` satisfies both SAFETY (neutralising threats) and SOCIAL (caring for others). These multi-tier mappings fall out naturally from the domain semantics — you don't need to configure them per character.

## Decay Toward Equilibrium, Not Zero

The first attempt at satisfaction decay was straightforward: multiply by a decay factor each consolidation cycle. `satisfaction *= (1 - decayRate)`. All tiers decay toward zero over time, creating increasing urgency for neglected needs.

This creates a problem I didn't anticipate until the design review caught it. With Safety decaying at 0.15 per cycle and consolidation running every 5 minutes, a character in a completely peaceful environment — no threats, no danger, nothing happening — reaches "critically neglected" Safety in about 50 minutes. The character starts reporting that they feel unsafe when the simulation has given them no reason to feel unsafe. The absence of threats should mean safety feels fine, not that safety is under critical pressure.

The fix: decay toward a resting level, not toward zero.

```java
double decayed = restingLevel + (satisfaction - restingLevel) * (1 - decayRate);
```

<svg viewBox="0 0 760 360" xmlns="http://www.w3.org/2000/svg" style="max-width:760px">
  <rect width="760" height="360" fill="#1a1b26" rx="8"/>

  <!-- Grid -->
  <line x1="80" y1="30" x2="80" y2="300" stroke="#24283b" stroke-width="1"/>
  <line x1="80" y1="300" x2="720" y2="300" stroke="#24283b" stroke-width="1"/>
  <!-- Horizontal grid lines -->
  <line x1="80" y1="60" x2="720" y2="60" stroke="#24283b" stroke-width="0.5" stroke-dasharray="4,4"/>
  <line x1="80" y1="120" x2="720" y2="120" stroke="#24283b" stroke-width="0.5" stroke-dasharray="4,4"/>
  <line x1="80" y1="180" x2="720" y2="180" stroke="#24283b" stroke-width="0.5" stroke-dasharray="4,4"/>
  <line x1="80" y1="240" x2="720" y2="240" stroke="#24283b" stroke-width="0.5" stroke-dasharray="4,4"/>

  <!-- Y axis labels -->
  <text x="70" y="64" text-anchor="end" fill="#565f89" font-family="sans-serif" font-size="10">1.0</text>
  <text x="70" y="124" text-anchor="end" fill="#565f89" font-family="sans-serif" font-size="10">0.8</text>
  <text x="70" y="184" text-anchor="end" fill="#565f89" font-family="sans-serif" font-size="10">0.6</text>
  <text x="70" y="244" text-anchor="end" fill="#565f89" font-family="sans-serif" font-size="10">0.4</text>
  <text x="70" y="304" text-anchor="end" fill="#565f89" font-family="sans-serif" font-size="10">0.2</text>

  <!-- X axis label -->
  <text x="400" y="330" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="11">Consolidation cycles →</text>

  <!-- Resting level lines (dashed) -->
  <line x1="80" y1="156" x2="720" y2="156" stroke="#f7768e" stroke-width="1" stroke-dasharray="6,4" opacity="0.4"/>
  <text x="725" y="152" text-anchor="start" fill="#f7768e" font-family="sans-serif" font-size="9" opacity="0.6">rest: 0.6</text>

  <line x1="80" y1="252" x2="720" y2="252" stroke="#ff9e64" stroke-width="1" stroke-dasharray="6,4" opacity="0.4"/>
  <text x="725" y="248" text-anchor="start" fill="#ff9e64" font-family="sans-serif" font-size="9" opacity="0.6">rest: 0.3</text>

  <line x1="80" y1="228" x2="720" y2="228" stroke="#7aa2f7" stroke-width="1" stroke-dasharray="6,4" opacity="0.4"/>
  <text x="725" y="224" text-anchor="start" fill="#7aa2f7" font-family="sans-serif" font-size="9" opacity="0.6">rest: 0.4</text>

  <!-- Starting point: all at 0.5 -->
  <!-- Safety (starts 0.5, rises to 0.6) - RED -->
  <polyline points="80,180 140,172 200,165 260,160 320,158 380,157 440,156 500,156 560,156 620,156" fill="none" stroke="#f7768e" stroke-width="2"/>

  <!-- Negative event drops Safety at cycle 8 -->
  <polyline points="620,156 630,240" fill="none" stroke="#f7768e" stroke-width="2" stroke-dasharray="4,2"/>
  <circle cx="630" cy="240" r="4" fill="#f7768e"/>
  <text x="640" y="238" fill="#f7768e" font-family="sans-serif" font-size="9">threat!</text>

  <!-- Recovery after threat -->
  <polyline points="630,240 660,220 690,200 720,185" fill="none" stroke="#f7768e" stroke-width="2"/>

  <!-- Tasks (starts 0.5, falls to 0.3) - ORANGE -->
  <polyline points="80,180 140,190 200,200 260,210 280,215 320,225 380,235 440,242 500,247 560,250 620,251 680,251 720,252" fill="none" stroke="#ff9e64" stroke-width="2"/>

  <!-- Understanding (starts 0.5, slowly falls to 0.4) - BLUE -->
  <polyline points="80,180 140,181 200,183 260,186 320,190 380,196 440,202 500,208 560,214 620,218 680,222 720,225" fill="none" stroke="#7aa2f7" stroke-width="2"/>

  <!-- Legend -->
  <line x1="100" y1="345" x2="130" y2="345" stroke="#f7768e" stroke-width="2"/>
  <text x="135" y="349" fill="#f7768e" font-family="sans-serif" font-size="10">Safety (rest: 0.6)</text>

  <line x1="280" y1="345" x2="310" y2="345" stroke="#ff9e64" stroke-width="2"/>
  <text x="315" y="349" fill="#ff9e64" font-family="sans-serif" font-size="10">Tasks (rest: 0.3)</text>

  <line x1="460" y1="345" x2="490" y2="345" stroke="#7aa2f7" stroke-width="2"/>
  <text x="495" y="349" fill="#7aa2f7" font-family="sans-serif" font-size="10">Understanding (rest: 0.4)</text>
</svg>

Each tier has a resting level — the satisfaction an agent settles at when nothing is happening. Safety rests at 0.6 because "no news is good news": the absence of threats should feel safe. Tasks rests at 0.3 because obligations accumulate even without events — a character who's been doing nothing should feel the pressure of unfinished duties. Social, Self-expression, and Understanding rest at 0.4 — moderate background pressure.

The formula is elegantly symmetric. When satisfaction is above the resting level (you just had a great social interaction), it decays downward — the warm glow fades. When satisfaction is below the resting level (your cover was just blown), it recovers upward — the immediate crisis passes. Both directions use the same formula. The resting level is the equilibrium point, and events perturb it in both directions.

This bidirectional model is the piece I almost missed. The first design only handled positive events — satisfaction could go up from good experiences, and decay provided the only downward pressure. A design reviewer pointed out the concrete failure: Hooded Claw's identity gets exposed (a `trust_change` event with negative pleasure), but because negative events didn't decrease satisfaction, his Safety stays at resting level 0.6. The character whose cover was just blown feels safe. That's wrong.

Now negative events actively push satisfaction below resting level. A threat drops Safety to 0.2, and the character genuinely feels unsafe. The decay-toward-resting model then provides the recovery path — the crisis fades over subsequent cycles, and Safety climbs back toward 0.6. The character feels the danger, then gradually calms down.

## The Constraint Loop

Here's where the pyramid earns its place architecturally. It doesn't just track satisfaction — it constrains drive adaptation.

Without the constraint, a dominant drive enters a runaway feedback loop. Penelope's `social-harmony` gets reinforced every time she helps someone. The reinforcement makes the drive stronger. The stronger drive means she's more likely to help. More helping means more reinforcement. Nothing stops this.

The pyramid adds a brake: when a tier becomes saturated, the learning rate for its associated drives drops toward zero.

```java
double effectiveLR = config.learningRate() * (1 - avgSatisfaction);
```

<svg viewBox="0 0 760 300" xmlns="http://www.w3.org/2000/svg" style="max-width:760px">
  <rect width="760" height="300" fill="#1a1b26" rx="8"/>

  <!-- Circular flow -->
  <!-- Box 1: Drive Reinforced -->
  <rect x="275" y="20" width="210" height="40" rx="6" fill="#1a1b26" stroke="#7aa2f7" stroke-width="1.5"/>
  <text x="380" y="45" text-anchor="middle" fill="#7aa2f7" font-family="sans-serif" font-size="12" font-weight="bold">Drive Reinforced</text>

  <!-- Arrow down-right -->
  <path d="M 485 40 Q 580 40 620 80" fill="none" stroke="#9ece6a" stroke-width="1.5" marker-end="url(#arrowG)"/>
  <defs><marker id="arrowG" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#9ece6a" stroke-width="1.5"/></marker></defs>

  <!-- Box 2: Tier Satisfaction ↑ -->
  <rect x="550" y="80" width="180" height="40" rx="6" fill="#1a1b26" stroke="#9ece6a" stroke-width="1.5"/>
  <text x="640" y="105" text-anchor="middle" fill="#9ece6a" font-family="sans-serif" font-size="12" font-weight="bold">Tier Satisfaction ↑</text>

  <!-- Arrow down -->
  <path d="M 640 120 Q 640 150 640 150" fill="none" stroke="#e0af68" stroke-width="1.5" marker-end="url(#arrowY)"/>
  <defs><marker id="arrowY" viewBox="0 0 10 10" refX="5" refY="10" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 5 10 L 10 0" fill="none" stroke="#e0af68" stroke-width="1.5"/></marker></defs>

  <!-- Box 3: Learning Rate ↓ -->
  <rect x="550" y="160" width="180" height="40" rx="6" fill="#1a1b26" stroke="#e0af68" stroke-width="1.5"/>
  <text x="640" y="185" text-anchor="middle" fill="#e0af68" font-family="sans-serif" font-size="12" font-weight="bold">Learning Rate ↓</text>

  <!-- Arrow down-left -->
  <path d="M 550 180 Q 460 220 380 230" fill="none" stroke="#ff9e64" stroke-width="1.5" marker-end="url(#arrowO)"/>
  <defs><marker id="arrowO" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#ff9e64" stroke-width="1.5"/></marker></defs>

  <!-- Box 4: Drive Stabilises -->
  <rect x="245" y="210" width="270" height="40" rx="6" fill="#1a1b26" stroke="#ff9e64" stroke-width="1.5"/>
  <text x="380" y="235" text-anchor="middle" fill="#ff9e64" font-family="sans-serif" font-size="12" font-weight="bold">Drive Stabilises → Others Respond</text>

  <!-- Arrow left then up -->
  <path d="M 245 230 Q 160 230 120 180" fill="none" stroke="#f7768e" stroke-width="1.5" marker-end="url(#arrowP)"/>
  <defs><marker id="arrowP" viewBox="0 0 10 10" refX="5" refY="0" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 10 L 5 0 L 10 10" fill="none" stroke="#f7768e" stroke-width="1.5"/></marker></defs>

  <!-- Box 5: Neglected Tiers Decay -->
  <rect x="30" y="110" width="180" height="50" rx="6" fill="#1a1b26" stroke="#f7768e" stroke-width="1.5"/>
  <text x="120" y="133" text-anchor="middle" fill="#f7768e" font-family="sans-serif" font-size="12" font-weight="bold">Neglected Tiers</text>
  <text x="120" y="150" text-anchor="middle" fill="#f7768e" font-family="sans-serif" font-size="12" font-weight="bold">Decay → LR Recovers</text>

  <!-- Arrow up-right back to start -->
  <path d="M 120 110 Q 120 40 275 40" fill="none" stroke="#7aa2f7" stroke-width="1.5" marker-end="url(#arrowB)"/>
  <defs><marker id="arrowB" viewBox="0 0 10 10" refX="10" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M 0 0 L 10 5 L 0 10" fill="none" stroke="#7aa2f7" stroke-width="1.5"/></marker></defs>

  <!-- Center label -->
  <text x="380" y="150" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="11">self-regulating</text>
  <text x="380" y="165" text-anchor="middle" fill="#565f89" font-family="sans-serif" font-size="11">feedback loop</text>

  <!-- Title -->
  <text x="380" y="285" text-anchor="middle" fill="#c0caf5" font-family="sans-serif" font-size="12">The saturation constraint prevents drive monopolisation</text>
</svg>

At 90% tier satisfaction, the effective learning rate is 10% of normal. The drive barely strengthens, even with strong positive reinforcement. Meanwhile, other drives — whose tiers are decaying from neglect — have full learning rates and respond normally to events. The system self-balances.

The formula is a single multiplication. That's intentional. The constraint needed to be computationally trivial because it runs inside the consolidation loop for every drive for every character. A complex constraint function would be architecturally clean but impractical at game-loop frequency.

One subtlety: the constraint uses current-cycle satisfaction, not stale data from the previous cycle. I originally designed the needs tracking as a separate consolidation phase running after drive adaptation, but this meant the constraint would always be one cycle behind — the drive would strengthen, then satisfaction would update, and the constraint would only kick in next cycle. A design reviewer pointed out that the satisfaction computation already has all the data it needs from the drive adaptation step. Integrating them into one phase eliminates the delay.

## Making Characters Feel Their Needs

Tracking satisfaction is pointless if the agent never perceives it. The needs pyramid renders satisfaction as qualitative prose in the agent's observation pipeline:

```
## Inner Needs

Your sense of safety feels critically neglected — recent events have left you uneasy.
Your task commitments feel adequate.
Your social bonds feel well-met.
Your self-expression feels fulfilled — you've been true to yourself lately.
Your curiosity feels neglected — there's much you don't understand yet.
```

The choice to render as prose rather than numbers was deliberate. `Safety: 0.15` is a game stat. "Your sense of safety feels critically neglected" is something a character can internalise and reference in dialogue. The LLM responds to qualitative descriptions more naturally — it generates richer, more character-appropriate responses when it reads "you feel uneasy" than when it reads `0.15`.

The rendering uses five bands:

```java
private static String bandLabel(double satisfaction) {
    if (satisfaction < 0.2) return "critically neglected";
    if (satisfaction < 0.4) return "neglected";
    if (satisfaction < 0.6) return "adequate";
    if (satisfaction < 0.8) return "well-met";
    return "fulfilled";
}
```

There's a filtering step that matters: the prompt section only renders tiers reachable through the character's drives. Hooded Claw has no drives mapping to TASKS (no `gallantry`, `proving-worth`, or `loyalty`). Without filtering, he'd see "Your task commitments feel neglected" in every prompt — character-breaking for a villain who has zero interest in obligations. With filtering, he only sees Safety, Self-expression, and the other tiers his drives can address.

Characters with no drives at all (Lazy Luke, Muttley) have the entire "Inner Needs" section suppressed. They don't participate in the cognitive simulation, and they shouldn't see needs they can't affect.

## What This Opens Up

The pyramid is infrastructure, not the end product. It exposes satisfaction levels that Goal Prioritization consumes. The next piece — wiring satisfaction into `GoalRevisionStrategy` — is where the pyramid's effect becomes visible in character behaviour.

The architecture bridges two systems that were previously disconnected. Drive adaptation (#66) determines *what the character wants to do* — drives strengthen from positive experiences. The needs pyramid determines *what the character should prioritise* — neglected needs create urgency. Goal prioritization (#69) synthesises both: an LLM reads the character's current drives, their need satisfaction levels, and their active goals, then reorders goals to address the most urgent gaps.

A character with high `social-harmony` drive but saturated SOCIAL satisfaction will still form social goals, but they'll rank below unsatisfied TASKS goals. The drive says "I want to help." The pyramid says "I've helped enough for now; my obligations are piling up." The goal system mediates.

That's the point of all this. Not to build a faithful Maslow simulation. To build agents whose behaviour feels like it comes from a real person — someone who balances competing internal pressures, not someone who fixates on the first thing that worked.
