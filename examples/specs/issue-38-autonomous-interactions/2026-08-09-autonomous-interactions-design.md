# Autonomous Interaction Chains — Design

**Issue:** casehubio/examples#38
**Date:** 2026-08-09
**Status:** Aspirational — starting points, not committed scope

## Core Principle

**Capabilities gate what you observe. Personality drives what you do about it.**

Characters see the same world. The engine filters observation *richness*
based on Eidos capability tags — a perceptive character sees more detail
about what's happening, not different objects. The LLM decides what to do
based on personality, goals, and what it observes. No plot-specific
instructions in descriptors.

## Validated Hypothesis

Removed `visibleTo: [hooded-claw]` from the poison object. All characters
now see the poison. `AutonomousDiscoveryTest` validates that personality
alone drives differentiated reactions to the same scenario:

- **Hooded Claw:** schemes to use poison against Penelope (villain goal)
- **Peter Perfect:** protective concern (gallant personality)
- **Penelope Pitstop:** completely ignores the poison (oblivious to danger — the authentic Penelope response, better than "concern")
- **Ant Hill Mob:** suspicious, on alert (protective instinct)
- **Dick Dastardly:** schemes for personal gain (different motivation than HC)

No `visibleTo` needed. Adding a new dangerous object to any room should
produce interesting character behavior WITHOUT descriptor changes.

## Source Material Patterns

Research into Perils of Penelope Pitstop and Wacky Races reveals
recurring patterns that map directly to autonomous agent behavior.

### The Rube Goldberg Trap Chain (Perils of Penelope Pitstop)

Every episode follows: HC builds elaborate trap → **monologues about how
it works** (LLMs are naturally good at this) → trap has a built-in delay →
during the delay, either Penelope escapes herself OR the Ant Hill Mob
bumbles in and accidentally saves her. The Mob's "help" often makes things
worse — Penelope sometimes has to save THEM.

### Character Flaw Dynamics (Wacky Races)

The characters' flaws drive the chains, not the objects:

| Character | Flaw | Consequence |
|-----------|------|-------------|
| Hooded Claw | Monologues about plans | Gives others time to intervene |
| Dastardly | Schemes when he should act directly | Self-sabotage ("We can't win fairly!" — 3 feet from the goal) |
| Muttley | Takes instructions literally | "Do something!" → tap dances. "Give me a hand!" → applauds |
| Peter Perfect | Gallantry overrides judgment | Stops to help Penelope, his car falls apart when he boasts |
| Penelope | Oblivious to danger | Trusts Sneekly completely, doesn't notice poison |
| Ant Hill Mob | Bumbling protectors | Rescue attempts create new problems |
| Slag Brothers | Won't hit Penelope | One stops the other with a club |
| Gruesomes | Accidentally terrifying | Think they're being hospitable |

## Proposed Interaction Chains

These are aspirational — explore, don't commit.

### Chain 1 — Hooded Claw's Escalating Schemes

Three stages, each foiled differently:

1. **Poison tea** (already implemented) — foiled by Ant Hill Mob knocking cup
2. **Poisonous plants** (conservatory, if/when room is added) — foiled by Pat Pending identifying the plant
3. **Laboratory machine trap** — elaborate Rube Goldberg device, foiled by Muttley "helping" Dastardly investigate and accidentally triggering it on HC

Each follows: discover tool → monologue → set trap → delay → foiled by accidental interference.

### Chain 2 — Dastardly's Treasure Race

Blubber Bear has a treasure map → Dastardly sees it → steals a copy →
gives wrong directions to Peter → Peter gallantly follows (to "scout ahead
for Penelope") → gets lost → Penelope finds the REAL path by asking the
Gruesomes → Dastardly arrives last. Self-sabotage chain.

### Chain 3 — The Gruesome Tea Party

Gruesomes prepare a "welcoming reception" in the cellar. Characters react
by personality: Penelope is polite but terrified, Sergeant Blast treats it
as hostile territory, Slag Brothers start redecorating with clubs, Mob
stakes it out. Gruesomes are confused why guests are screaming.

### Chain 4 — Peter's Gallantry Loop

Peter tries to protect Penelope from every perceived danger. Each attempt
backfires: breaks a door trying to open it, crashes into furniture chasing
a bat, his back gives out carrying her. Penelope calmly walks around the
mess. She's resourceful when not grabbed by the Hooded Claw.

### Chain 5 — The Machine Problem

Lab machine requires 3 operators. Comedy of miscommunication getting 3
Wacky Races characters to cooperate. When it finally works by accident,
reveals something useful.

## Game Frame — Will Reading

All characters are at Doily Manor for the reading of Horace Doily's will.
Condition: survive the night to inherit. The Hooded Claw (as executor
Sneekly) knows Penelope gets everything and wants her eliminated before
dawn. Three acts: Evening → Midnight → Dawn. Fixed outcome — Penelope
survives. The entertainment is the journey.

This is briefing context for every character — one line in the descriptor,
not a game mechanic. Characters act autonomously within this frame.

## Capability-Gated Observations

Same event, filtered by observer's capabilities:

- **Character WITH perception:** "Sneekly is standing unusually close to Penelope's tea cup"
- **Character WITHOUT perception:** "Sneekly is by the tea service"

Implementation: ObservationBuilder checks the observing character's Eidos
capability tags. Characters with `perception`/`suspicion` tags get richer
event descriptions. No dice rolls — binary capability tag matching.

## Concealable Actions

Certain actions (STEAL, USE in some contexts) can be concealed:

- **Actor has deception capability** → observers without perception don't
  see the action in their next observation. Victim always knows.
- **Actor lacks deception** → everyone in the room sees it (Muttley just
  grabs and runs)
- **Observer has perception capability** → sees it regardless of concealment

This creates natural information asymmetry driven by character capabilities.
HC steals poison while Mob is present: HC has deception → concealed. But
Mob has suspicion → they notice. Peter standing right there? Oblivious.

## STEAL Action Type

Like TAKE but targets another character's inventory. Engine validates:
target in same room, item exists in their inventory. Concealable (see
above). Victim's next observation includes the theft — autonomous reaction.

A character without deception capabilities wouldn't attempt to steal in
a crowded room — not because the engine blocks it, but because the LLM
knows its character is a terrible liar and chooses a different approach.

Infinite loops (steal → steal back → steal again) are unlikely with good
descriptors — real characters don't do the same thing twice. If it happens
in practice, a "you already tried this recently" line in the observation
nudges variety. Try without guardrails first.

## Rooms Serve Plot

Rooms are added when a plot mechanism requires a location, not scaled
speculatively. The existing 6 rooms serve the current interaction chains.
New rooms (gallery, conservatory, tower, attic) are added as part of
specific chain implementations if the chain needs a location that doesn't
exist.

Issue #34 (room scaling to 10) was closed based on this principle.

## Testing Strategy

Same pattern as Phase 0 LLM eval tests:

- **Autonomous discovery:** same scenario, different characters, verify personality-appropriate reactions (validated — AutonomousDiscoveryTest)
- **Capability-gated observation:** same event, two capability profiles, verify observation richness difference
- **Concealment:** verify non-perceptive observers miss concealed actions
- **Steal interaction:** verify victim reacts in-character
- **Broad goal pursuit:** verify characters discover and use world tools without explicit instructions

## Platform Opportunity

Issue #39 captures the hypothesis that capability-driven observation
filtering may generalise to Eidos as a platform SPI. Validate the pattern
in wacky-manor first (#38), then decide whether to extract.
