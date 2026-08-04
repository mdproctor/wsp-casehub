# Pause/Play/Speed Controls + Character Profile Panel

**Issues:** #27, #28
**Date:** 2026-08-04
**Branch:** issue-27-pause-play-profiles

## Overview

Two UI features for wacky-manor: transport controls (pause/play/speed) to make
17-character scenarios readable, and a character profile panel that reveals the
Eidos personality graph behind each character.

## 1. Server-Side Pause and Speed

### WorldState changes

Add two volatile fields to `WorldState`:

```java
private volatile boolean paused = false;
private volatile double speedMultiplier = 1.0;
```

Accessors: `isPaused()`, `setPaused(boolean)`, `speedMultiplier()`,
`setSpeedMultiplier(double)`. Speed clamped to `[0.25, 8.0]`.

### CharacterAgentLoop pause gate

Before each LLM call in `CharacterAgentLoop.run()`, check `world.isPaused()`.
If paused, spin-wait with 200ms sleep intervals until unpaused or scenario
completes. No LLM tokens are burned while paused.

The think delay becomes:
```java
long delay = (long)(character.thinkDelayMs() / world.speedMultiplier());
Thread.sleep(delay);
```

`thinkDelayMs` on `CharacterState` stays `final` (base delay). The multiplier
is applied at call time from `WorldState`.

## 2. REST Endpoints

New endpoints on `ManorResource`:

| Method | Path | Body | Effect |
|--------|------|------|--------|
| POST | `/manor/pause` | — | `world.setPaused(true)`, broadcast control event |
| POST | `/manor/resume` | — | `world.setPaused(false)`, broadcast control event |
| POST | `/manor/speed` | `?rate=2.0` | `world.setSpeedMultiplier(rate)`, broadcast control event |
| GET | `/manor/characters/{id}/profile` | — | Return `CharacterProfileDTO` as JSON |

The profile endpoint reads from `AgentRegistry.findById(id, TENANCY_ID)` and
projects a `CharacterProfileDTO` — not the raw `AgentDescriptor`, which contains
operational fields (`provider`, `modelFamily`, `weightsFingerprint`, `jurisdiction`,
`dataHandlingPolicy`) that have no business reaching the frontend.

Guard: pause/resume/speed return 404 if no active scenario.

### CharacterProfileDTO

```java
public record CharacterProfileDTO(
    String agentId,
    String name,
    String slot,           // raw slot value e.g. "shaper"
    String slotLabel,      // resolved from VocabularyRegistry e.g. "Shaper"
    String mbtiType,       // derived from dispositionProfile via EidosRenderPipeline.deriveMbtiType()
    String enneagramType,  // from disposition config
    List<FunctionWeight> dispositionProfile,  // Jungian function weights for bar chart
    List<CapabilityDTO> capabilities,
    List<GoalDTO> goals,          // PUBLIC visibility only
    List<ConstraintDTO> constraints,  // PUBLIC visibility only
    String briefing
) {}
```

`slotLabel` is resolved by looking up the `slot` term in the vocabulary registry
using the descriptor's `slotVocabulary` URI. `mbtiType` is derived from the
disposition profile's Jungian function weights (same logic as the render pipeline).
`enneagramType` comes from the `AgentDisposition` enum config.

## 3. WebSocket Protocol Extension

New `ManorWebSocketEvent` factory method:

```java
public static ManorWebSocketEvent control(String status, double speedMultiplier) {
    // type="control", status="paused"|"resumed"|"speed", speedMultiplier=current value
}
```

Three status values:
- `"paused"` — scenario paused
- `"resumed"` — scenario resumed
- `"speed"` — speed changed without pause/resume state change

Frontend `ManorEvent` type union gains:

```typescript
| { type: 'control'; status: 'paused' | 'resumed' | 'speed'; speedMultiplier: number }
```

Control events broadcast to all connected clients so multi-client state stays
in sync. The `speedMultiplier` field is added to `ManorWebSocketEvent` as a
nullable `Double` (only present on `control` events).

## 4. Frontend Transport Controls

### Toolbar additions (manor-app.ts)

New controls in the `.controls` div, between the status badge and start button:

- **Pause/Play toggle**: `⏸ Pause` / `▶ Play` button. Calls `POST /manor/pause`
  or `/manor/resume`. Disabled when scenario is idle or completed.
- **Speed buttons**: `0.5x | 1x | 2x | 4x` inline button group. Active speed
  highlighted. Calls `POST /manor/speed?rate=N`. Disabled when idle/completed.

State tracked via `@state() paused: boolean` and `@state() speed: number`.
Updated from WebSocket `control` events (authoritative) and optimistically
on button press.

## 5. Character Profile Panel

### Click target (manor-view.ts)

Add `@click` handler to each character `<g>` group in the SVG. On click,
dispatch a `character-selected` CustomEvent with the character ID. The
`manor-app` listens for this event.

### Profile component (character-profile.ts)

New Lit component `<character-profile>`. Takes a `characterId` property.

On `characterId` change:
1. Fetch `GET /manor/characters/{characterId}/profile`
2. Render a slide-out panel showing:
   - Character name and Belbin role (slot label resolved from vocabulary)
   - MBTI type (derived from Jungian function weights) and Enneagram type
   - Jungian function stack as SVG bar chart (8 bars, weighted from dispositionProfile)
   - Capabilities (name + tags)
   - Goals (PUBLIC visibility only, with priority)
   - Constraints (PUBLIC visibility only, with severity)
   - Briefing text

### Profile response TypeScript type

```typescript
export interface CharacterProfileResponse {
  agentId: string;
  name: string;
  slot: string;
  slotLabel: string;
  mbtiType: string | null;
  enneagramType: string | null;
  dispositionProfile: Array<{ term: string; weight: number }>;
  capabilities: Array<{ name: string; tags: string[] }>;
  goals: Array<{ name: string; description: string; priority: string }>;
  constraints: Array<{ name: string; description: string; severity: string }>;
  briefing: string | null;
}
```

### Panel position

Absolutely positioned overlay anchored to the right side of the manor view
section. Width ~280px, dark theme matching the existing UI palette.

### Dismiss

Click same character again, click outside the panel, or click a different
character (which opens that character's profile instead).

## 6. Character Sprites

12 new inline SVG sprites added to the `renderCharacterAtOrigin` switch in
`manor-view.ts`. Same scale/style as the existing 5 sprites (~20 lines each).

| Character | Visual identity | Colour |
|-----------|----------------|--------|
| muttley | Small snickering dog, brown | `#8B6914` |
| pat-pending | Lab coat, goggles on head | `#2E8B57` |
| sergeant-blast | Military cap, olive drab | `#556B2F` |
| private-meekly | Small timid figure, olive drab | `#6B8E23` |
| lazy-luke | Lanky hillbilly, straw hat | `#DAA520` |
| blubber-bear | Large round bear, brown | `#8B4513` |
| rock-slag | Stocky caveman, brown fur | `#A0522D` |
| gravel-slag | Stocky caveman, grey fur | `#708090` |
| rufus-ruffcut | Lumberjack, flannel, beard | `#B22222` |
| sawtooth | Beaver, flat tail | `#D2691E` |
| big-gruesome | Large friendly purple monster | `#9370DB` |
| little-gruesome | Tiny winged green creature | `#32CD32` |

`CHARACTER_COLORS` and `CHARACTER_SHORT_NAMES` in `types.ts` updated with
entries for all 12 new characters.

## 7. File Changes

### Java (server)
- `WorldState.java` — add `paused`, `speedMultiplier` fields + accessors
- `CharacterAgentLoop.java` — pause gate before LLM call, speed multiplier on delay
- `ManorResource.java` — 4 new endpoints (pause, resume, speed, profile)
- `ManorWebSocketEvent.java` — add `control()` factory, add `speedMultiplier` field
- `CharacterProfileDTO.java` — new record projecting AgentDescriptor for frontend
- `ManorEventBus.java` — no changes (broadcasts already generic)

### Dead code cleanup
- Delete `CharacterProfile.java` — legacy Phase 0, zero callers
- Delete `CharacterProfileLoader.java` — legacy Phase 0, zero callers

### TypeScript (frontend)
- `types.ts` — add `control` event type, `CharacterProfileResponse` type, 12 new `CHARACTER_COLORS` + `CHARACTER_SHORT_NAMES` entries
- `manor-app.ts` — pause/play/speed controls in toolbar, profile panel wiring
- `manor-view.ts` — click handlers on characters, 12 new SVG sprites
- `character-profile.ts` — new component for the slide-out profile panel

## 8. Testing

### Java
- `WorldState` pause/speed: unit tests for flag behavior, clamping
- `ManorResource` endpoints: `@QuarkusTest` for pause/resume/speed/profile
- `CharacterAgentLoop` pause gate: test that loop blocks when paused
- `CharacterProfileDTO`: unit test for projection from AgentDescriptor

### TypeScript
- `layout.test.ts` extensions if layout changes
- Manual browser testing for UI interactions (click, dismiss, speed buttons)

## 9. Garden Context

Relevant entries from work-start:
- GE-20260522-1bc491 — SSE double-frame (bare JSON, no manual prefix)
- GE-20260617-36c6b5 — @Blocking for polling loops
- GE-20260804-8b0fd6 — BroadcastProcessor for CDI async events
- GE-20260617-c2ceb3 — @RunOnVirtualThread for SSE scaling

These apply if we add SSE endpoints later. Current design uses WebSocket
(already established) for all real-time communication.

## 10. Review Findings

Light design review (coherence + structure + robustness + cross-cutting).
Structure and robustness clean. Coherence and cross-cutting surfaced findings,
triaged below:

| Finding | Verdict | Resolution |
|---------|---------|------------|
| MBTI/Enneagram not on AgentDescriptor | Valid | §2: CharacterProfileDTO derives from disposition + vocab registry |
| `slotLabel` doesn't exist | Valid | §2: resolved from VocabularyRegistry |
| Control event can't represent speed-only | Valid | §3: added `"speed"` status variant |
| Full AgentDescriptor exposes operational fields | Valid | §2: project CharacterProfileDTO instead |
| CHARACTER_COLORS not updated | Valid | §6: colour column added, types.ts update noted |
| Profile TypeScript types unspecified | Valid | §5: CharacterProfileResponse interface defined |
| CharacterProfileLoader conflict | False alarm | Dead code — both files deleted |
| WorldState concurrency concern | Non-issue | Two independent volatile fields, no compound invariant |
| AgentRegistry CDI availability | Non-issue | Already injected in ScenarioOrchestrator, verified working |
