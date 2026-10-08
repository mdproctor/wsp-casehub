# Design Journal — issue-115-drive-expression-gap

## 2026-10-08 — Root-to-tip propagation analysis

### What we set out to do
Close the drive expression gap: secondary/performative drives (gloating, gallantry, proving-worth) collapse without prescriptive text. Started with PP proving-worth as lowest-hanging fruit.

### What we discovered
The drive expression gap is a **propagation problem**, not an LLM capability or data problem. The cognitive pipeline has rich personality data (AMPD, memories, drives, CAPS attractors) but five dead ends prevent it from reaching the LLM:

1. `BehavioralPromptSection` disabled — `behavioralEnabled = false` in `CognitionConfig.all()`. 50 CAPS attractor nodes produced by consolidation, never rendered.
2. `renderFormationMemories()` never called — method exists in CharacterCognition, zero callers.
3. `seedCapsState()` never called in production — CAPS engine runs without disposition data.
4. `seedPeople()` never called in production — relationship evolution has no substrate.
5. No drive rendering path — drives seeded as MindMap nodes but never surfaced in prompt.

Of 14 personality data types, only 5 reach the LLM. The richest ones all hit dead ends.

### Experimental evidence
Micro tests on PP proving-worth:
- Direct trigger (competitor praised): **5/5** — LLM capability confirmed
- Ambiguous scenario (alone in library): **3/5** — only 1 sentence carries signal
- + drives as text: **4/5** — naming the drive helps
- + enriched memories: **3/5** — memories alone don't activate
- + drives + memories: **5/5** — both needed

**Critical constraint:** drive cues must be somatic/felt, not prescriptive. Every change must be tested for scripting vs emergence.

### Key architectural finding
The neocortex source is at `/Users/mdproctor/claude/casehub/slots/196/neocortex`. Key files:
- `BehavioralPromptSection.java` at `cognition/src/main/java/.../cognition/prompt/`
- `DriveAdaptationPhase.java` at `cognition/src/main/java/.../cognition/drive/`
- `CognitionCore.java` at `cognition/src/main/java/.../cognition/core/`
- `BehavioralSynthesisPhase.java` at `mindmap-intelligence/src/main/java/.../consolidation/`

### Decisions
- Start with wiring fixes (Layer 1) before inference enrichment (Layer 2)
- Each propagation boundary needs an isolated unit test before integration
- TDD approach: red test per boundary, fix, verify emergence not scripting
