# Research Brief: Personality Composition Paper Rewrite

> **Purpose:** Everything the next session needs to write the article from scratch.
> **Context:** This brief was produced at the end of a long session that built
> the experiment harness (issue #2), ran all tests, and attempted the article —
> but the first draft missed critical architecture. A code scan at the end of
> the session uncovered the full scope.

---

## Git Situation

**Slot 59 worktree:** `/Users/mdproctor/claude/casehub/worktrees/59/examples`
- Currently on branch `issue-9-narrator-wiring` (new work since our session)
- Issue #2 branch (`issue-2-staged-layer-comparison`) was already merged to
  main and stamped closed

**Main repo:** `/Users/mdproctor/claude/casehub/examples`
- On branch `feat/wacky-manor-poc`
- `main` is at `d798b62` (experiment harness merged)
- **UNCOMMITTED FILE:** `wacky-manor/docs/structured-personality-composition-in-llm-agents.md`
  — the incomplete first draft. This needs to be REPLACED, not patched.

**To land the rewritten article:**
1. Work from the main repo (not the slot worktree)
2. Create a new branch from `feat/wacky-manor-poc` or `main`
3. Delete the incomplete draft, write the new one
4. Merge to main, push to origin + upstream

---

## What the Article Must Cover

The user wants a white paper / research paper covering:
1. Problem statement and hypothesis
2. Psychology frameworks background (accessible to non-psychologists)
3. How frameworks map to AI agent disposition encoding
4. How frameworks compose to create richer agent identities
5. The compatibility matrix — which pairs reinforce, conflict, or are redundant
6. What was built (Eidos platform architecture)
7. How it was tested (the 4-layer experiment)
8. Results and evidence
9. Discussion — insights, gaps, weaknesses, next steps
10. Future research and applied AI directions
11. Forward-looking close (not a summary)

**Form:** Article/essay — structured essay with numbered hybrid headings.
**Voice:** Mark Proctor personal voice (`~/claude-workspace/writing-styles/mark-proctor-voice.md` + `structured-essay.md`).
**Style rules:** Load `write-content` skill. Anti-slop. No process narration. Each sentence earns its place.

---

## The Architecture the First Draft Missed

### 1. Five Disposition Axes (the shared coordinate system)

Every vocabulary projects onto these five axes — they are the normalized
representation all frameworks converge to:

| Axis | Terms (Conscientiousness vocab) | What it governs |
|---|---|---|
| SOCIAL_ORIENTATION | collaborative / independent / facilitative | How collaborative or independent |
| RULE_FOLLOWING | strict / principled / flexible | How rigidly rules are followed |
| RISK_APPETITE | conservative / measured / bold | How risk-tolerant |
| AUTONOMY | directed / semi-autonomous / autonomous | How self-directed |
| CONFLICT_MODE | competing / collaborating / compromising / avoiding / accommodating | How conflict is handled |

Source: `eidos/api/src/main/java/io/casehub/eidos/api/DispositionAxis.java`

### 2. Two Signal Channels

The composition architecture has TWO independent channels:

**Axis channel:** Framework → `axisExactMatch()` → Conscientiousness terms (axes 1-4) + Thomas-Kilmann terms (axis 5) → normalized disposition → rendered into prompt

**Prompt channel:** Slot vocabulary → slot label + description rendered directly into system prompt text (Belbin, SVO)

Jungian and DISC use the axis channel. Belbin uses the prompt channel. This is why Jungian+Belbin composition is non-redundant — they operate through different mechanisms.

### 3. Conscientiousness as Backbone

The Conscientiousness vocabulary is NOT "just another framework." It IS the axis system. Grounded in Big Five research. All other framework vocabularies project ONTO Conscientiousness terms. It defines the target terms for 4 of 5 axes. Thomas-Kilmann defines the 5th.

### 4. Eleven Frameworks Mapped (not four)

The first draft covered 4. The actual `personality-frameworks.md` maps 11:

| Framework | Vocabulary Role | Implemented? |
|---|---|---|
| **Jungian Cognitive Functions** | Disposition vocabulary — 8 functions with full axis projection | Yes (`JungianFunctionTerm`) |
| **MBTI Types** | Convenience shorthand — 16 types grounded via `specializes()` to Jungian stacks | Yes (`MbtiTypeTerm`) |
| **DISC** | Disposition vocabulary — 4 quadrants with full axis projection | Yes (`DiscTerm`) |
| **Thomas-Kilmann** | Conflict mode vocabulary — 5 modes | Yes (`ThomasKilmannTerm`) |
| **Belbin** | Slot vocabulary — 9 team roles (prompt channel only, no axisExactMatch) | Yes (`BelbinTerm`) |
| **SVO** | Slot vocabulary — 3 agent workflow roles (Coordinator/Performer/Evaluator) | Yes (`SvoTerm`) |
| **Big Five/OCEAN** | Reference only — Conscientiousness vocab IS Big Five-grounded | No (implicit) |
| **Margerison-McCann** | Reference only — redundant with Belbin | No |
| **Situational Leadership** | Reference only — conceptual framing for autonomy axis | No |
| **Kirton Adaption-Innovation** | Reference only — supports ruleFollowing + riskAppetite axes | No |
| **O*NET / SFIA** | Capabilities vocabulary source (occupational skills) | No (naming convention) |

### 5. Framework Compatibility Matrix

Critical section the first draft omitted. From `personality-frameworks.md` §6:

| Pair | Rating | Key Reasoning |
|---|---|---|
| Jungian + Belbin | **Additive** | Cognitive style + team role are orthogonal |
| Belbin + DISC | **Additive** | Role (slot) + behavioral style (disposition) are orthogonal |
| Belbin + Big Five | **Additive** | Role + stable trait are orthogonal |
| Big Five + Thomas-Kilmann | **Additive** | Personality trait + conflict strategy are different constructs |
| Jungian + DISC | **Redundant** | Both project onto same axes; Jungian is deeper |
| Jungian + Conscientiousness | **Redundant** | Jungian projects onto all Conscientiousness axes |
| DISC + Big Five | **Redundant** | DISC is a quadrant simplification of Big Five E×A |
| Belbin + Margerison-McCann | **Redundant** | Same conceptual territory, contradictory terminology |
| MBTI (human-assessed) + anything | **Inadvisable** | Poor test-retest reliability |
| MBTI (agent-specified) + Jungian | **Hierarchical** | Types emerge from function stacks via specializes() |

### 6. Four Named Combination Patterns

- **Belbin Profile** — Belbin slot + Conscientiousness disposition
- **Belbin + DISC Profile** — Belbin slot + DISC disposition
- **Occupational Profile** — O*NET/SFIA capabilities + Conscientiousness
- **Jungian Profile** — weighted function stack with auto-derived axes

### 7. The Jungian Rehabilitation Argument

MBTI is invalid for humans (~50% type-change on retest). But for LLM agents, personality is *specified* not *measured*. No measurement error because no measurement. MBTI types are supported via `MbtiTypeTerm`, grounded through Jungian cognitive functions via `specializes()`. The type label is emergent from the weighted function stack — not an injected identity.

### 8. Jungian Function Weight System

Functions have continuous weights [0.0-1.0]:
- Dominant: 0.31-1.00
- Auxiliary: 0.06-0.30
- Undifferentiated: 0-0.06
- Reinforcement delta: 0.06
- Decay factor: 0.20

Plus shadow functions, opposite functions, compatible auxiliaries — structural rules for personality evolution.

### 9. Why These Frameworks Were Included or Excluded

**The paper must explain the selection rationale, not just list frameworks.**

**Included (implemented as vocabularies):**

| Framework | Why included | Signal channel |
|---|---|---|
| **Jungian** | Deepest cognitive model — 8 functions with structural rules (shadow, auxiliary pairing, weight tiers). The JPAF paper (arXiv:2601.10025) demonstrates 100% MBTI alignment in LLMs. Subsumes DISC. | Axis (full 5-axis projection) |
| **MBTI** | Convenience layer — 16 types decompose into Jungian function stacks via `specializes()`. Resolves the "MBTI is invalid" objection: for agents, personality is specified not measured. | Axis (via Jungian) |
| **DISC** | Widely adopted workplace shorthand. Despite low scientific validity, its imprecision is bounded (correlates with Big Five E×A). Provides a simpler entry point than Jungian for users who think in 4 quadrants. Full axis projection. | Axis (full 5-axis projection) |
| **Thomas-Kilmann** | Only framework that models conflict behavior. Covers the 5th disposition axis (CONFLICT_MODE) that no other framework addresses. Essential for multi-agent disagreement scenarios. | Axis (CONFLICT_MODE only) |
| **Belbin** | Only framework that models team contribution roles. Operates through the prompt channel (slot label + description), making it orthogonal to axis-channel frameworks. The only way to answer "what does this agent contribute to a team?" | Prompt (slot vocabulary) |
| **SVO** | Lightweight agent workflow role model (Coordinator/Performer/Evaluator). Maps agent function in multi-agent orchestration. Platform-specific rather than psychology-derived. | Prompt (slot vocabulary) |

**Included as reference (not implemented):**

| Framework | Why reference only |
|---|---|
| **Big Five/OCEAN** | Not separate — it IS the Conscientiousness vocabulary. The axis system is Big Five-grounded by design. No separate vocabulary needed because the axis terms ARE Big Five terms. |
| **Situational Leadership** | Describes leader behavior, not agent traits. Used backwards (follower readiness → agent autonomy) as conceptual framing for the autonomy axis. Not precise enough for vocabulary terms. |
| **Kirton Adaption-Innovation** | Single-axis model (Adaptor↔Innovator) that maps onto existing ruleFollowing + riskAppetite axes. Confirms those axes from an independent theoretical basis but adds no new terms. |
| **O*NET / SFIA** | Occupational competence frameworks — capabilities vocabulary source (skill names), not personality. Orthogonal to disposition. |

**Excluded:**

| Framework | Why excluded |
|---|---|
| **Margerison-McCann** | Same conceptual territory as Belbin (team contribution roles) with incompatible terminology. Belbin has broader adoption and more research. Implementing both creates contradictory slot vocabulary. |
| **DISC + Jungian together** | Redundant — both project onto the same 5 axes. Jungian is strictly deeper (8 functions with structural rules vs 4 quadrants). Using both creates contradictory disposition encodings. |
| **MBTI (human-assessed)** | ~50% type-change on retest makes human-assessed types unreliable as vocabulary terms. Only agent-specified MBTI (via Jungian function stacks) is supported. |

### 10. Room for More Frameworks

**The architecture is explicitly extensible.** Adding a new framework means:
1. Create an enum implementing `VocabularyTerm`
2. Implement `axisExactMatch()` for axis-channel frameworks (projection onto Conscientiousness + Thomas-Kilmann)
3. Register via a `VocabularyRegistrar` CDI bean
4. The vocabulary registry discovers it automatically at startup

**Frameworks worth adding:**

| Framework | Why | What it would add | Effort |
|---|---|---|---|
| **Big Five (full)** | The gold standard in personality science. Currently only Conscientiousness is implemented. Adding Openness, Extraversion, Agreeableness, Neuroticism as explicit vocabularies would give agents the most scientifically validated personality encoding available. | 4 new vocabulary enums with axis projections. Would subsume DISC entirely. | Medium — mappings are well-documented |
| **Enneagram** | 9 personality types with growth/stress dynamics. Popular in coaching and personal development. The growth/stress paths map naturally to personality evolution (shadow function activation). | New disposition vocabulary with evolution rules. Orthogonal to Jungian (motivation vs cognition). | Medium — mappings to Big Five are published |
| **Strength Deployment Inventory (SDI)** | Relationship-focused: how people deploy their strengths in conflict vs non-conflict. Complementary to Thomas-Kilmann (which covers conflict style but not strength deployment). | Enhanced conflict-mode granularity. | Low — narrow scope, clear axis mapping |
| **VALUES frameworks (Schwartz, Rokeach)** | What agents value — not how they think or behave, but what they optimize for. Currently no axis covers terminal vs instrumental values. | New axis: VALUE_ORIENTATION. Would need a new DispositionAxis enum value. | High — new axis, cross-vocabulary projections needed |

**Frameworks NOT worth adding:**

| Framework | Why not |
|---|---|
| **StrengthsFinder / CliftonStrengths** | 34 talent themes — too granular, overlaps heavily with capabilities (O*NET). Talent ≠ personality. |
| **HEXACO** | Big Five variant adding Honesty-Humility. Valuable for humans but the 6th dimension adds little for LLM agents where honesty is a system constraint, not a personality trait. |
| **Dark Triad (Machiavellianism, Narcissism, Psychopathy)** | Measures pathological personality. Inappropriate for production agent design. Could be relevant for adversarial testing or red-teaming scenarios only. |

### 11. Interpretive Insights (from this session's analysis)

Key judgments the next session should carry forward when writing the paper:

**Why composition latency is additive, not redundant:** Each framework adds cognitive context to the system prompt. More context = more for the LLM to process before deciding. Jungian adds +2.3 turns, Belbin adds +1.0, composite adds +4.3 (close to the sum of +3.3, with a +1.0 interaction effect). The frameworks don't cancel each other out — they stack.

**Why briefing text dominates function activation:** The character briefings are rich — multiple sentences of voice guidance, specific speech patterns, named quirks. A character described as "theatrical villain who schemes elaborately" will produce Ne-like behavior (possibility explosions) regardless of whether the disposition says Te. The framework adds texture WITHIN the behavioral envelope defined by the briefing, but doesn't redefine the envelope.

**Why composite variance is a feature:** The composite layer's high variance (6 to 13 turns) means the agent sometimes deliberates extensively and sometimes acts quickly. A richer personality model makes behavior less predictable. For compliance/safety: use simpler encodings. For creative/emergent behavior: the variance is the point.

**Why Peter Perfect fails J/P:** The briefing text says "gallant, volunteering, tunnel vision on Penelope" — which reads as spontaneous and adaptive (P), overriding the ENFJ's J disposition. This is a briefing-framework tension, not a platform bug.

**Why MBTI alignment succeeds while function activation fails:** MBTI alignment evaluates the PROMPT (does it read as ENTJ?). Function activation evaluates the RESPONSE (does the agent act like Te?). The prompt correctly presents the Jungian profile, but the LLM's response is shaped more by the briefing than the disposition. The framework reaches the prompt; it doesn't control the response.

### 13. BehavioralExpectations

Derives operational governance from dispositions:
- `escalationExpected()` — does the autonomy level imply supervision?
- `delegationExpected()` — can the agent spawn sub-agents?
- `latencyBound()` — capability-driven latency expectations

Connects personality to platform operational behavior.

### 14. Nine Eval Judges (not two)

The eidos-eval module has a comprehensive evaluation framework:

| Judge | What it evaluates |
|---|---|
| MbtiAlignmentJudge | Does the rendered prompt present as the declared MBTI type? |
| FunctionActivationJudge | Does the prompt activate expected cognitive functions? |
| DispositionPresenceJudge | Are disposition terms present in the rendered prompt? |
| TraitExpressionJudge | Does the agent express expected personality traits? |
| BehavioralJudge | Does the agent's behavior match the disposition profile? |
| VocabularyExpressivenessJudge | Does the prompt vocabulary reflect the framework richness? |
| PromptJudge | General prompt quality assessment |
| ProximityJudge | How close is the agent's behavior to the expected profile? |
| PairContrastJudge | Do two agents with different profiles behave differently? |
| PersonalityEvolutionJudge | Does personality evolve correctly over interactions? |

### 15. Academic Citations

- JPAF paper: arXiv:2601.10025 — 100% MBTI alignment across GPT-4, Llama, Qwen
- Activation steering validation: arXiv:2607.20803 (July 2026) — confirms function-level personality control
- Jung, *Psychological Types*, 1921
- Belbin, *Team Roles at Work*, 1993
- Thomas & Kilmann, 1974 (from Blake-Mouton Managerial Grid)
- Marston, 1928 (DISC origin)
- Costa & McCrae, 1992 (NEO PI-R / Big Five definitive measurement)
- Rao & Georgeff, 1991 (BDI agent architecture)
- Kirton, 1976 (KAI)
- Hersey & Blanchard, 1969 (Situational Leadership)

---

## Experiment Results (for the results section)

### All 12 Autonomous Runs

| Layer | Run 1 | Run 2 | Run 3 | Avg Turns |
|---|---|---|---|---|
| BASELINE | POISONED (7) | POISONED (6) | POISONED (6) | 6.3 |
| JUNGIAN | POISONED (10) | POISONED (10) | POISONED (6) | 8.7 |
| BELBIN | POISONED (9) | POISONED (7) | POISONED (6) | 7.3 |
| COMPOSITE | POISONED (6) | POISONED (13) | POISONED (13) | 10.7 |

Key finding: composition adds latency additively (+2.3 Jungian, +1.0 Belbin, +4.3 composite). Composite has highest variance.

### MBTI Alignment (from PromptQualityTest — 38 min live LLM run)

8/10 profiles aligned. Peter Perfect consistently fails J/P. Hooded Claw fully aligned in composite.

### Function Activation

Flat across all layers. Fe always activates. Ni never activates (always classified as Ne). Te activates in baseline but NOT in Jungian/Composite. Character briefing dominates function activation.

### Existing Tests (all passed)

- 5 voice tests, 5 plot device tests, 3 interaction tests, 4 soliloquy tests, 1 live scenario — all pass
- 158 total tests (150 standard + 8 experiment) pass
- 30/31 llm-eval tests pass (1 PromptQualityTest NPE — null guard issue)

---

## Key Source Files

| File | What it contains |
|---|---|
| `eidos/docs/personality-frameworks.md` | The 946-line reference document — ALL framework mappings, compatibility matrix, combination patterns, anti-patterns |
| `eidos/api/src/main/java/.../DispositionAxis.java` | The 5 shared axes |
| `eidos/api/src/main/java/.../VocabularyTerm.java` | The term interface — exactMatch, axisExactMatch, impliesSupervision, defaultProfile |
| `eidos/api/src/main/java/.../AgentDisposition.java` | The structured disposition record |
| `eidos/api/src/main/java/.../BehavioralExpectations.java` | Disposition → operational governance |
| `eidos/vocab/src/main/java/.../JungianFunctionTerm.java` | 8 functions with full axis projections |
| `eidos/vocab/src/main/java/.../MbtiTypeTerm.java` | 16 types with defaultProfile() |
| `eidos/vocab/src/main/java/.../DiscTerm.java` | 4 quadrants with full axis projections |
| `eidos/vocab/src/main/java/.../ConscientiousnessTerm.java` | The axis backbone (Big Five-grounded) |
| `eidos/vocab/src/main/java/.../ThomasKilmannTerm.java` | 5 conflict modes |
| `eidos/vocab/src/main/java/.../BelbinTerm.java` | 9 team roles (prompt channel only) |
| `eidos/vocab/src/main/java/.../SvoTerm.java` | 3 agent workflow roles |
| `eidos/eval/src/main/java/.../` | All 9+ eval judges |
| `eidos/docs/specs/2026-07-28-jungian-personality-framework-design.md` | Jungian implementation spec |
| `eidos/docs/blog/2026-07-28-mdp03-rendering-personality.md` | Blog about rendering pipeline |
| `examples/wacky-manor/docs/POC-SPEC.md` | Wacky Manor scenario spec |
| `work/specs/.../2026-07-30-staged-layer-comparison-experiment-design.md` | Experiment design spec (adversarially reviewed) |
| `target/experiment-results/prompt-quality.json` | Full PromptQualityTest results (in main repo target/) |
| `target/experiment-results/*.json` | All 12 run transcripts (in main repo target/) |

---

## What Went Wrong With the First Draft

I wrote from session memory instead of scanning the codebase. The session had focused on the experiment harness (issue #2) which uses Jungian and Belbin only — so the article covered those two plus DISC and Thomas-Kilmann from the Phase 2.5b spec. I never read `personality-frameworks.md`, never checked what other vocabularies exist, never looked at the eval framework beyond the two judges I used.

The user caught it: "I'm sure we had more than that?" — and they were right. The system maps 11 frameworks, has 6 implemented vocabularies, a compatibility matrix, named combination patterns, anti-patterns, BDI lineage, academic citations, and a 9-judge evaluation framework. The first draft covered roughly 30% of the material.
