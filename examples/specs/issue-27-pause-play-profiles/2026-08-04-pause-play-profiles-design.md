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
| GET | `/manor/characters/{id}/profile` | — | Return `AgentDescriptor` as JSON |

The profile endpoint reads from `AgentRegistry.findById(id, TENANCY_ID)` and
returns the full descriptor. The frontend filters `PRIVATE` visibility items.

Guard: pause/resume/speed return 404 if no active scenario.

## 3. WebSocket Protocol Extension

New `ManorWebSocketEvent` factory method:

```java
public static ManorWebSocketEvent control(String status, double speedMultiplier) {
    // type="control", status="paused"|"resumed", speedMultiplier=current value
}
```

Frontend `ManorEvent` type union gains:

```typescript
| { type: 'control'; status: 'paused' | 'resumed'; speedMultiplier: number }
```

Control events broadcast to all connected clients so multi-client state stays
in sync.

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
   - Character name and Belbin role (slot + slotLabel)
   - MBTI type and Enneagram type
   - Jungian function stack as SVG bar chart (8 bars, weighted)
   - Capabilities (name + tags)
   - Goals (PUBLIC visibility only, with priority)
   - Constraints (PUBLIC visibility only, with severity)
   - Briefing text

### Panel position

Absolutely positioned overlay anchored to the right side of the manor view
section. Width ~280px, dark theme matching the existing UI palette.

### Dismiss

Click same character again, click outside the panel, or click a different
character (which opens that character's profile instead).

## 6. Character Sprites

12 new inline SVG sprites added to the `renderCharacterAtOrigin` switch in
`manor-view.ts`. Same scale/style as the existing 5 sprites (~20 lines each).

| Character | Visual identity |
|-----------|----------------|
| muttley | Small snickering dog, brown |
| pat-pending | Lab coat, goggles on head |
| sergeant-blast | Military cap, olive drab |
| private-meekly | Small timid figure, olive drab |
| lazy-luke | Lanky hillbilly, straw hat |
| blubber-bear | Large round bear, brown |
| rock-slag | Stocky caveman, brown fur |
| gravel-slag | Stocky caveman, grey fur |
| rufus-ruffcut | Lumberjack, flannel, beard |
| sawtooth | Beaver, flat tail |
| big-gruesome | Large friendly purple monster |
| little-gruesome | Tiny winged green creature |

## 7. File Changes

### Java (server)
- `WorldState.java` — add `paused`, `speedMultiplier` fields + accessors
- `CharacterAgentLoop.java` — pause gate before LLM call, speed multiplier on delay
- `ManorResource.java` — 4 new endpoints (pause, resume, speed, profile)
- `ManorWebSocketEvent.java` — add `control()` factory, add `speedMultiplier` field
- `ManorEventBus.java` — no changes (broadcasts already generic)

### TypeScript (frontend)
- `types.ts` — add `control` event type, add profile response types
- `manor-app.ts` — pause/play/speed controls in toolbar, profile panel wiring
- `manor-view.ts` — click handlers on characters, 12 new SVG sprites
- `character-profile.ts` — new component for the slide-out profile panel

## 8. Testing

### Java
- `WorldState` pause/speed: unit tests for flag behavior, clamping
- `ManorResource` endpoints: `@QuarkusTest` for pause/resume/speed/profile
- `CharacterAgentLoop` pause gate: test that loop blocks when paused

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
