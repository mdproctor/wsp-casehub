# Scenario Controller UI — Visual Transport Controls and Outline

> **Issue:** casehubio/casehub-pages#341
> **Epic:** casehubio/parent#408 — Cross-Platform Scenario Engine
> **Date:** 2026-08-21
> **Status:** Draft

## 1. Overview

The scenario engine (issues #418, #332, #340, #342) delivers orchestration,
distributed executor dispatch, and narrative content. The missing piece is a
controller UI that lets a demo operator see the scenario structure, control
execution pace, and optionally view narrative content — from the same device
or a separate one (presenter remote).

This spec covers:
1. Backend additions — `scenario:state` push wire broadcast, `GET /scenario/outline`
2. `<scenario-controller>` — Lit web component with outline tree and transport controls
3. `<scenario-narrative>` — Lit web component for narrative content rendering
4. Standalone `/scenario/remote` page

### 1.1 Constraints

- **Single-session orchestrator:** one scenario at a time per Pages instance
  (ScenarioOrchestrator is `@ApplicationScoped` with volatile fields). The
  controller UI doesn't multi-tenant — it shows whatever's running.
- **REST + push wire only:** no GraphQL or MCP endpoints for the controller.
  Those can be added later if a consumer needs them.
- **Pre-release:** breaking changes to public API are acceptable.

## 2. Backend Additions

### 2.1 State broadcast — `scenario:state` topic

The `ScenarioOrchestrator` currently serves state via `GET /scenario/state`
and broadcasts `executor-control` messages to executors, but does not push
state changes to listening controllers. Add state broadcasting.

**What changes in `ScenarioOrchestrator`:**

1. Inject `EventBroadcaster` (from push-runtime).
2. After every state-mutating operation (`start`, `pause`, `resume`, `step`,
   `speed`, `onStepResult`), call:
   ```java
   broadcaster.broadcast("scenario:state", state());
   ```
3. The `EventBroadcaster.broadcast()` serialises `ScenarioState` as the
   event payload and delivers it to all sessions listening on `scenario:state`.

**Wire format — `op: "event"` on topic `scenario:state`:**

```json
{
  "op": "event",
  "topic": "scenario:state",
  "payload": {
    "scenario": "help-desk-demo",
    "chapter": "Customer Reports Issue",
    "section": "Customer sends message",
    "step": "Submit support request",
    "paused": false,
    "speed": 1.0,
    "progress": 0.33,
    "content": {"type": "inline", "markdown": "The customer fills out..."},
    "slides": null
  },
  "seq": 42
}
```

The `content` field is polymorphic — serialised as the `NarrativeContent`
sealed interface (Inline, Template, Slide). The controller and narrative
components handle each variant.

**When to broadcast:**
- `start()` — initial state after dispatch
- `pause()` — paused=true
- `resume()` — paused=false
- `step()` — after the step executes (paused=true)
- `speed()` — new speed value
- `onStepResult()` — position advances, progress updates
- `stop()` — idle state (null scenario)

### 2.2 Outline endpoint — `GET /scenario/outline`

Add to `ScenarioControlResource`:

```java
@GET
@Path("/outline")
public ScenarioOutline outline() {
    return orchestrator.outline();
}
```

**Response model:**

```java
public record ScenarioOutline(
    String scenario,
    List<OutlineChapter> chapters,
    List<OutlineSection> sections,
    List<OutlineStep> steps
) {
    // Mirrors HierarchicalScenario's three mutually exclusive
    // top-level structures: chapters, sections, or flat steps
}

public record OutlineChapter(String label, List<OutlineSection> sections) {}
public record OutlineSection(String label, List<OutlineStep> steps) {}
public record OutlineStep(String label, String target) {}
```

The outline is a static, label-only tree for the controller's navigation.
No commands, no data, no narrative content — just the hierarchy and labels
that the "run to" feature needs.

Returns `404` if no scenario is active.

## 3. `<scenario-controller>` Web Component

### 3.1 Package and registration

Lives in `packages/pages-aria/src/controller/`.

```
packages/pages-aria/src/controller/
  scenario-controller.ts     — LitElement
  scenario-controller.test.ts
  index.ts                   — export + customElements.define
```

Registered as `<scenario-controller>`. Exported from `pages-aria` index.

### 3.2 Properties

```typescript
@property({ attribute: false })
connection?: EventConnection;        // Embedded mode — shared connection

@property()
baseUrl?: string;                    // Remote mode — e.g. "http://localhost:8080"
```

**Mode resolution:**
- If `connection` is set → embedded mode. Use the provided connection for
  listening. REST base URL is `baseUrl` if provided, otherwise
  `window.location.origin` (same-origin assumption for embedded use).
- If only `baseUrl` is set → remote mode. Create an internal
  `EventConnection` to `${baseUrl.replace(/^http/, 'ws')}/ws/push`.
  REST base is `baseUrl`.
- If neither → render an error state ("No connection configured").

### 3.3 Internal state

```typescript
interface ControllerState {
  scenario: string | null;
  chapter: string | null;
  section: string | null;
  step: string | null;
  paused: boolean;
  speed: number;
  progress: number;
}

interface Outline {
  scenario: string;
  chapters?: Array<{ label: string; sections: Array<{ label: string; steps: Array<{ label: string; target: string }> }> }>;
  sections?: Array<{ label: string; steps: Array<{ label: string; target: string }> }>;
  steps?: Array<{ label: string; target: string }>;
}
```

All state is reactive (`@state()` decorator) — changes trigger re-render.

### 3.4 Lifecycle

**`connectedCallback()`:**
1. If remote mode → create internal `EventConnection` to `${baseUrl}/ws/push`
2. Listen on `scenario:state` via connection
3. Register event handler for `pages-event` with topic `scenario:state`
4. Fetch initial state: `GET ${restBase}/scenario/state`
5. If state has an active scenario → fetch outline: `GET ${restBase}/scenario/outline`

**`disconnectedCallback()`:**
1. Unlisten `scenario:state`
2. If remote mode → close internal connection

**On `scenario:state` event received:**
1. Update internal state from payload
2. If `scenario` changed (new scenario started) → re-fetch outline
3. If `scenario` is null → clear outline (scenario ended)

### 3.5 Rendering — three sections

**Outline panel (top):**

A collapsible tree showing the chapter/section/step hierarchy. The current
position is highlighted. Each label is clickable — clicking triggers a
"run to" command for that label.

```
▾ Customer Reports Issue          ← chapter (bold)
  ▾ Customer sends message        ← section
    ● Submit support request      ← step (current — highlighted)
    ○ System classifies ticket    ← step (pending)
  ▸ Specialist resolves           ← section (collapsed)
▸ Reporting                       ← chapter (collapsed)
```

Visual states for steps:
- `●` filled circle — current step
- `✓` check — completed (label is before current position in the flat list)
- `○` empty circle — pending

Clicking a label calls `POST ${restBase}/scenario/run-to` with that label.

Outline is empty when no scenario is active — show a "No scenario running"
placeholder.

**Transport bar (middle):**

```
[⏮] [⏪] [⏸/▶] [⏩] [⏭]   ●────────○ 1.0x   33%
 │     │     │      │    │    └─ speed slider    └─ progress
 │     │     │      │    └─ run-to next chapter
 │     │     └──────└─── step / skip section
 │     └─ run-to prev section (disabled if at start)
 └─ restart (disabled — no rewind support)
```

Core controls:
- **Play/Pause toggle** — `POST /scenario/resume` or `POST /scenario/pause`
- **Step** — `POST /scenario/step` (advance one step, then pause)
- **Speed slider** — range input, 0.1x to 10x, sends `POST /scenario/speed`
  with debounce (250ms). Display current speed as label.

All buttons disabled when no scenario is active.

**Status bar (bottom):**

Single line showing the current position breadcrumb and connection status:

```
Customer Reports Issue → Customer sends message → Submit support request   ● Connected
```

Connection status reflects `EventConnection.status`:
- `● Connected` (green dot)
- `● Reconnecting` (amber dot, pulsing)
- `● Disconnected` (red dot)

### 3.6 REST command dispatch

All commands are sent via `fetch()`:

```typescript
private async sendCommand(path: string, body?: object): Promise<void> {
  const url = `${this.restBase}/scenario${path}`;
  const resp = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    ...(body ? { body: JSON.stringify(body) } : {}),
  });
  if (!resp.ok) {
    // Handle error — update status bar with error message
  }
}
```

The REST response returns `ScenarioState` — but we don't use it for state
updates (the push wire broadcast delivers the canonical state to all
controllers simultaneously). The response is only used for error detection.

### 3.7 Styling

Uses CSS custom properties from `pages-ui-tokens` for consistency:
- `--pages-neutral-*` for text and backgrounds
- `--pages-accent-*` for the current position highlight
- `--pages-success-*` for completed steps
- `--pages-font-*` for typography
- `--pages-space-*` for spacing
- `--pages-radius-*` for border radius
- `--pages-duration-*` for transition timing

Shadow DOM with `:host` sizing to fill its container. The host can control
dimensions via CSS.

## 4. `<scenario-narrative>` Web Component

### 4.1 Package and registration

Lives in `packages/pages-aria/src/controller/`.

```
packages/pages-aria/src/controller/
  scenario-narrative.ts
  scenario-narrative.test.ts
```

Registered as `<scenario-narrative>`.

### 4.2 Properties

```typescript
@property({ attribute: false })
connection?: EventConnection;

@property()
baseUrl?: string;
```

Same connection/mode pattern as `<scenario-controller>`.

### 4.3 Rendering

Listens on `scenario:state` and renders the `content` field based on its type:

**`Inline` (markdown):**
Render markdown to HTML. Use a lightweight renderer — either import an
existing one (marked) or a minimal implementation for the subset needed
(headings, paragraphs, bold, italic, lists, code blocks). The rendered
HTML is set via `innerHTML` on a container with scoped styles.

**`Template` (path + section extraction):**
Fetch the template file from `${restBase}/${path}`, extract the named
section (if specified), and render the resulting markdown. Cache fetched
templates — the same file may be referenced by multiple steps.

**`Slide` (reveal.js reference):**
Render a reference or embed. Details TBD based on the reveal.js integration
— for now, show the slide reference as a link or placeholder.

**No content:**
Show nothing (empty component). The narrative panel is optional — if no
step has narrative content, it takes no space.

### 4.4 Styling

Clean reading typography. Light background, max-width for readability.
CSS custom properties from `pages-ui-tokens`.

## 5. Standalone Remote Page

### 5.1 Location

A static HTML file at:
```
backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/remote.html
```

Served by Quarkus at `/scenario/remote` (or `/scenario/remote.html`).

### 5.2 Content

Minimal HTML that loads the `<scenario-controller>` component in remote
mode:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Scenario Remote</title>
  <style>
    body { margin: 0; font-family: system-ui, sans-serif; }
    scenario-controller {
      display: block; width: 100vw; height: 100vh;
    }
  </style>
</head>
<body>
  <scenario-controller baseUrl=""></scenario-controller>
  <script type="module" src="/scenario/controller.js"></script>
</body>
</html>
```

The `baseUrl=""` means "same origin" — the remote page is served by the
same backend that runs the orchestrator. The script bundle
(`controller.js`) is the pages-aria controller entry point, built as
an ESM bundle for standalone loading.

### 5.3 Bundle

A separate webpack/esbuild entry point in pages-aria that bundles:
- `scenario-controller.ts` + its Lit dependencies
- Custom element registration

This produces a standalone JS file that can be loaded independently of
the full pages-runtime bundle. The file is placed in
`META-INF/resources/scenario/controller.js` during the build.

## 6. Data Flow

```
                    ┌─────────────────────────┐
                    │  ScenarioOrchestrator    │
                    │  (Java backend)          │
                    ├─────────────────────────┤
      REST ←────── │ /scenario/state          │
      REST ←────── │ /scenario/outline        │
      REST ←────── │ /scenario/pause|resume|… │
                    │                          │
                    │  EventBroadcaster        │
                    │    .broadcast(           │
                    │      "scenario:state",   │
                    │       state())           │
                    └──────────┬──────────────┘
                               │ push wire (WebSocket)
                               │ op: "event"
                               │ topic: "scenario:state"
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                  │
   ┌─────────▼──────┐  ┌──────▼────────┐  ┌─────▼──────────┐
   │ <scenario-     │  │ <scenario-    │  │ <scenario-     │
   │  controller>   │  │  narrative>   │  │  controller>   │
   │ (embedded)     │  │ (embedded)    │  │ (remote/phone) │
   │                │  │               │  │                │
   │ shared         │  │ shared        │  │ own            │
   │ EventConnection│  │ EventConnection│  │ EventConnection│
   └────────────────┘  └───────────────┘  └────────────────┘
```

Multiple controllers and narrative components can listen simultaneously.
The push wire delivers the same state to all of them. Commands are sent
via REST — the response state is not used for rendering (the push wire
broadcast is the canonical source).

## 7. Testing Strategy

### 7.1 Backend (Java)

- **ScenarioOrchestrator broadcast test:** verify `EventBroadcaster.broadcast`
  is called after each state mutation (start, pause, resume, step, speed,
  onStepResult) with correct `ScenarioState` payload.
- **Outline endpoint test:** `GET /scenario/outline` returns correct tree
  structure for chapters/sections/steps scenarios. Returns 404 when no
  scenario is active.

### 7.2 Frontend (TypeScript)

- **`<scenario-controller>` unit tests:**
  - Renders empty state when no scenario is active
  - Updates state from mock `scenario:state` events
  - Highlights current position in outline tree
  - Sends correct REST commands on button clicks
  - Handles connection status changes
  - Embedded vs remote mode connection wiring
- **`<scenario-narrative>` unit tests:**
  - Renders inline markdown content
  - Renders nothing when content is null
  - Handles content type switching

### 7.3 Integration / E2E

- **Playwright MCP:** Start a scenario via REST, verify the controller
  shows the outline, advance steps, verify position updates in real-time
  via the push wire connection.

## References

- [distributed-executor-protocol-design.md](2026-08-20-distributed-executor-protocol-design.md) — §5 Controller API, §7 Controller UI
- ScenarioOrchestrator.java — current orchestrator (no broadcast)
- ScenarioControlResource.java — existing REST endpoints
- ScenarioState.java — state record with NarrativeContent
- EventConnection.ts — push wire client with `listen()` API
- scenario-handler.ts — browser executor (dispatch-sequence/control handling)
- PagesButton.ts — existing Lit component pattern
- [GE-20260818-c61c29] — topicSource adapter for push wire → DataSource bridge
- [GE-20260816-e89cda] — composable Lit reactive controllers (evaluated, deferred for single-consumer case)
- [GE-20260812-5cd146] — EventConnection drops non-event messages (addressed — dispatch-sequence/control now handled)
- [GE-20260818-78bf96] — subscribe vs listen protocol distinction
