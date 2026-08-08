# Room Scaling — 6 to 10 Rooms

**Issue:** casehubio/examples#34
**Date:** 2026-08-08

## Goal

Expand the mansion from 6 to 10 rooms with multi-floor layout. The rooms
are stage dressing for character autonomy — objects create interaction
opportunities, personalities do the rest. Success: characters spread across
rooms, interact in entertaining ways, and the observation accumulator handles
the additional partitions without degradation.

A per-tick log line showing characters-per-room and interaction-vs-idle
ratio validates spatial distribution and interaction rate (issue #34's
measurement requirements) without a metrics framework.

## Layout

Multi-floor layout. Two stairway connections (tower and gallery) provide
route options between floors, though the attic is only reachable via tower.

```
  Attic (3)                        [attic]
                                      |
  Upper (2)                  [tower]   [gallery]
                               |           |
  Ground (1) [ballroom]-[kitchen]-[entrance-hall]-[library]-[laboratory]
                                       |
                                 [conservatory]
                                                       |
  Below (0)                                        [cellar]
```

**Adjacency list (authoritative — diagram is illustrative):**

| Room | Adjacent to |
|------|------------|
| entrance-hall | kitchen, library, gallery, conservatory |
| kitchen | entrance-hall, ballroom |
| ballroom | kitchen |
| library | entrance-hall, laboratory |
| laboratory | library, cellar, tower |
| cellar | laboratory |
| tower | laboratory, attic |
| attic | tower |
| gallery | entrance-hall, conservatory |
| conservatory | entrance-hall, gallery |

Rooms gain a `floor` integer property (0-3). The `Room` Java record
needs a `floor` field added. Used by the UI for vertical rendering.
Adjacency stays as string lists — floor is for presentation only.

**Floor assignments:** entrance-hall, kitchen, ballroom, library,
conservatory = 1. Laboratory = 1. Cellar = 0. Tower, gallery = 2.
Attic = 3.

## New Rooms

### Tower (floor 2)

Adjacent: laboratory, attic

A windswept circular room at the top of a crumbling stone staircase.
A brass telescope points out the only unbroken window. A heavy iron-bound
chest sits against the wall.

Objects:
- **Telescope** — interactable, flavour only. Characters who look through
  it get a description of the moonlit grounds. No cross-room observation
  enrichment (that would require new trigger/observation mechanics).
- **Iron Chest** — interactable. Yields: gold-doubloon (flavour item).
  No prerequisite item required.

### Gallery (floor 2)

Adjacent: entrance-hall, conservatory

A long hall lined with portraits of the Doily family ancestors. Their
eyes seem to follow you. Three suits of armour stand at attention along
the far wall.

Objects:
- **Suit of Armour** — interactable. Characters can hide inside.
  Hooded Claw would use this for ambush; Ant Hill Mob for stakeout.
- **Portraits** — description mentions the eyes follow you. Spooky
  flavour — characters with paranoid or suspicious personalities
  will comment on them.

### Conservatory (floor 1)

Adjacent: entrance-hall, gallery

A glass-walled Victorian greenhouse, humid and overgrown. Exotic plants
climb the iron framework.

Objects:
- **Exotic Plants** — description mentions some look poisonous.
  visibleTo: [hooded-claw] for the poisonous ones. A second scheme
  opportunity — different object, same character motivation.
- **Ornate Fountain** — non-interactable flavour. Bubbling water,
  peaceful atmosphere. Draws characters seeking calm (Penelope,
  Lazy Luke).

### Attic (floor 3)

Adjacent: tower

A dusty storage room under the eaves. Old steamer trunks, a dressmaker's
dummy, and a rocking chair that moves on its own when nobody's watching.

Objects:
- **Steamer Trunk** — interactable. Contains: disguise-kit (portable).
  Hooded Claw or Dastardly would want this. Multiple characters competing
  for one useful item.
- **Rocking Chair** — non-interactable but described as "gently rocking
  despite no breeze." Spooky flavour — characters will react based on
  personality (Penelope: scared, Gruesomes: charmed, Ant Hill Mob:
  suspicious).
- **Dressmaker's Dummy** — non-interactable. Looks like a person in
  the dark. Characters entering will react to it.

## Character Starting Positions

Spread 18 characters across 8 rooms. Tower and attic empty — destinations.

| Room | Characters | Count |
|------|-----------|-------|
| entrance-hall | Penelope, Hooded Claw, Peter Perfect | 3 |
| kitchen | Ant Hill Mob, Muttley | 2 |
| ballroom | Lazy Luke, Blubber Bear | 2 |
| library | Rock Slag, Gravel Slag | 2 |
| laboratory | Rufus Ruffcut, Sawtooth, Pat Pending | 3 |
| cellar | Big Gruesome, Little Gruesome, Red Max | 3 |
| gallery | Dick Dastardly | 1 |
| conservatory | Sergeant Blast, Private Meekly | 2 |
| tower | — | 0 |
| attic | — | 0 |

## Interaction Magnets

Objects designed to draw multiple characters for conflicting reasons.
All interactions are emergent — driven by object descriptions in the
LLM observation prompt, not new coded interaction types.

- **Telescope** — Dastardly wants intelligence, Pat Pending wants to
  study it, Peter Perfect wants to spot danger for Penelope
- **Suit of Armour** — Hooded Claw hides inside, Ant Hill Mob
  investigates suspicious clanking
- **Steamer Trunk (disguise kit)** — Hooded Claw and Dastardly both
  want disguises, for completely different schemes
- **Poisonous Plants** — Hooded Claw's backup scheme if tea poisoning
  fails. Same character motivation, different room, natural escalation

## UI Changes

`manor-view` SVG renders rooms by floor — 4 rows instead of 1 horizontal
strip. Each floor is a row, rooms positioned by adjacency within the row.
Stairway connections shown as vertical lines between floors. Room positions
and floor layout derived from room YAML data (floor property + adjacency),
not hardcoded in the component.

## What's NOT in scope

- New triggers or scenes — characters interact autonomously
- Fear meters or elimination mechanics — future issue if needed
- New character descriptors — existing personalities are sufficient
- Narrative scripting — rooms create opportunities, characters drive plot
- Cross-room observation mechanics (telescope seeing other rooms, etc.)
- New coded interaction types — all new object behaviors are prompt-driven

## Implementation

1. Add `floor` field to `Room` Java record and `MansionLoader` parser
2. Add 4 rooms to `rooms.yaml` with floor property on all 10 rooms
3. Update adjacency lists for existing rooms (entrance-hall gains
   gallery + conservatory; laboratory gains tower)
4. Update `characters.yaml` starting positions
5. Add per-tick log line: room distribution + interaction/idle ratio
6. Update `manor-view` SVG for multi-floor rendering
7. Run a 10-room, 18-character autonomous scenario and watch
