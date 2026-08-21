# Scenario Controller UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/casehub-pages#341 — Scenario controller UI
**Issue group:** #341 (under epic casehubio/parent#408)

**Goal:** Build `<scenario-controller>` and `<scenario-narrative>` Lit
web components with a standalone `/scenario/remote` page, backed by
push wire state broadcasting and a REST outline endpoint.

**Architecture:** The orchestrator broadcasts `ScenarioState` on the
`scenario:state` push wire topic after every state mutation. Controller
components listen via `EventConnection.listen()` and receive events on
an injected `EventTarget`. Commands are sent via REST. A recursive
`OutlineNode` model projects the scenario hierarchy for navigation.

**Tech Stack:** Java 21 (Quarkus), TypeScript 5, Lit 3, Vitest, Jackson

## Global Constraints

- Pre-release platform — breaking changes acceptable
- IntelliJ MCP required for Java edits — open workspace before starting:
  `ide_open_workspace({ modules: [".../backend/scenario", ".../backend/scenario-runtime", ".../backend/push", ".../backend/push-runtime"] })`
- TypeScript edits use standard tools (pages is a TS monorepo without IntelliJ project files)
- All commits reference `Refs #341`
- `lit` must be added as explicit dependency to pages-aria `package.json`
- Jackson `@JsonTypeInfo`/`@JsonSubTypes` on `NarrativeContent` for polymorphic wire format

---

## Batch 1: Backend prerequisites and state broadcast

### Task 1: Jackson annotations on NarrativeContent + OutlineNode model

**Files:**
- Modify: `backend/scenario/src/main/java/io/casehub/pages/scenario/NarrativeContent.java`
- Create: `backend/scenario/src/main/java/io/casehub/pages/scenario/OutlineNode.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/NarrativeContentSerializationTest.java`
- Test: `backend/scenario/src/test/java/io/casehub/pages/scenario/OutlineNodeTest.java`

**Interfaces:**
- Produces: `NarrativeContent` with Jackson polymorphic annotations (wire format: `{"type":"inline","markdown":"..."}`)
- Produces: `OutlineNode(String label, String target, List<OutlineNode> children)` — recursive tree node

- [ ] **Step 1: Write failing test for NarrativeContent serialization**

```java
@Test
void inlineSerializesToTypeDiscriminator() throws Exception {
    var mapper = new ObjectMapper();
    NarrativeContent content = new NarrativeContent.Inline("Hello world");
    String json = mapper.writeValueAsString(content);
    var tree = mapper.readTree(json);
    assertEquals("inline", tree.get("type").asText());
    assertEquals("Hello world", tree.get("markdown").asText());
}

@Test
void templateSerializesWithType() throws Exception {
    var mapper = new ObjectMapper();
    NarrativeContent content = new NarrativeContent.Template("docs/intro.md", "overview", Map.of());
    String json = mapper.writeValueAsString(content);
    var tree = mapper.readTree(json);
    assertEquals("template", tree.get("type").asText());
    assertEquals("docs/intro.md", tree.get("path").asText());
}

@Test
void slideSerializesWithType() throws Exception {
    var mapper = new ObjectMapper();
    NarrativeContent content = new NarrativeContent.Slide("slide-3");
    String json = mapper.writeValueAsString(content);
    var tree = mapper.readTree(json);
    assertEquals("slide", tree.get("type").asText());
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl backend/scenario -Dtest=NarrativeContentSerializationTest -f backend/pom.xml`
Expected: FAIL — no `type` property in serialized output

- [ ] **Step 3: Add Jackson annotations to NarrativeContent**

Add to `NarrativeContent.java`:

```java
import com.fasterxml.jackson.annotation.JsonSubTypes;
import com.fasterxml.jackson.annotation.JsonTypeInfo;

@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = NarrativeContent.Inline.class, name = "inline"),
    @JsonSubTypes.Type(value = NarrativeContent.Template.class, name = "template"),
    @JsonSubTypes.Type(value = NarrativeContent.Slide.class, name = "slide"),
})
public sealed interface NarrativeContent {
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl backend/scenario -Dtest=NarrativeContentSerializationTest -f backend/pom.xml`
Expected: PASS

- [ ] **Step 5: Write OutlineNode and test**

Create `OutlineNode.java`:

```java
package io.casehub.pages.scenario;

import java.util.List;

public record OutlineNode(String label, String target,
                          List<OutlineNode> children) {
    public OutlineNode(String label, List<OutlineNode> children) {
        this(label, null, children);
    }
    public OutlineNode(String label, String target) {
        this(label, target, List.of());
    }
}
```

Test that `OutlineNode` serializes correctly with Jackson (leaf node has empty children, non-leaf has null target).

- [ ] **Step 6: Run all tests, commit**

Run: `mvn test -pl backend/scenario -f backend/pom.xml`

```bash
git add backend/scenario/
git commit -m "feat(#341): Jackson annotations on NarrativeContent + OutlineNode model

Refs #341"
```

### Task 2: State broadcast and outline endpoint in ScenarioOrchestrator

**Files:**
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java`
- Modify: `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioControlResource.java`
- Test: `backend/scenario-runtime/src/test/java/io/casehub/pages/scenario/runtime/ScenarioOrchestratorBroadcastTest.java`

**Interfaces:**
- Consumes: `EventBroadcaster` (from push-runtime, already CDI-produced)
- Consumes: `OutlineNode` (from Task 1)
- Produces: `broadcaster.broadcast("scenario:state", state())` after each mutation
- Produces: `outline()` method returning `List<OutlineNode>`
- Produces: `GET /scenario/outline` REST endpoint
- Produces: `stop()` method clearing session and broadcasting idle state

- [ ] **Step 1: Write failing test for state broadcast**

```java
@Test
void startBroadcastsState() {
    // Given: orchestrator with mock EventBroadcaster
    var captured = new ArrayList<Object>();
    var broadcaster = new EventBroadcaster(/* ... */) {
        @Override
        public <T> long broadcast(String topic, T event) {
            if ("scenario:state".equals(topic)) captured.add(event);
            return 0;
        }
    };
    var orchestrator = new ScenarioOrchestrator(sender, broadcaster);

    // When: start a scenario
    orchestrator.start(SIMPLE_YAML);

    // Then: state was broadcast
    assertFalse(captured.isEmpty());
    var state = (ScenarioState) captured.getLast();
    assertNotNull(state.scenario());
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — `ScenarioOrchestrator` constructor doesn't accept `EventBroadcaster`

- [ ] **Step 3: Add EventBroadcaster to ScenarioOrchestrator**

Modify `ScenarioOrchestrator`:
1. Add `@Inject EventBroadcaster broadcaster` field
2. Update constructor to accept both `SessionSender` and `EventBroadcaster`
3. Add `private void broadcastState()` method calling `broadcaster.broadcast("scenario:state", state())`
4. Call `broadcastState()` at the end of: `start()`, `pause()`, `resume()`, `step()`, `speed()`, `onStepResult()`

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write failing test for stop()**

```java
@Test
void stopClearsSessionAndBroadcastsIdle() {
    orchestrator.start(SIMPLE_YAML);
    captured.clear();

    orchestrator.stop();

    var state = (ScenarioState) captured.getLast();
    assertNull(state.scenario());
    assertEquals(0.0, state.progress());
}
```

- [ ] **Step 6: Implement stop()**

```java
public void stop() {
    this.sessionId = null;
    this.scenario = null;
    this.allSteps = List.of();
    this.completedSteps.clear();
    this.paused = false;
    this.speed = 1.0;
    broadcastState();
}
```

Wire `ScenarioControlResource.stop()` to call `orchestrator.stop()`.

- [ ] **Step 7: Write failing test for outline()**

```java
@Test
void outlineReturnsHierarchicalTree() {
    orchestrator.start(CHAPTERS_YAML);
    var outline = orchestrator.outline();

    assertEquals(2, outline.size()); // 2 chapters
    assertEquals("Customer Reports Issue", outline.get(0).label());
    assertNull(outline.get(0).target()); // chapter — no target
    assertFalse(outline.get(0).children().isEmpty());
    // leaf step has target
    var step = outline.get(0).children().get(0).children().get(0);
    assertNotNull(step.target());
}
```

- [ ] **Step 8: Implement outline() and wire REST endpoint**

```java
// In ScenarioOrchestrator:
public List<OutlineNode> outline() {
    if (scenario == null) return List.of();
    return buildOutline(scenario);
}

private List<OutlineNode> buildOutline(HierarchicalScenario s) {
    if (s.chapters() != null) {
        return s.chapters().stream()
            .map(c -> new OutlineNode(c.label(),
                c.sections().stream()
                    .map(sec -> new OutlineNode(sec.label(),
                        sec.steps().stream()
                            .map(st -> new OutlineNode(st.label(), st.target()))
                            .toList()))
                    .toList()))
            .toList();
    }
    if (s.sections() != null) {
        return s.sections().stream()
            .map(sec -> new OutlineNode(sec.label(),
                sec.steps().stream()
                    .map(st -> new OutlineNode(st.label(), st.target()))
                    .toList()))
            .toList();
    }
    if (s.steps() != null) {
        return s.steps().stream()
            .map(st -> new OutlineNode(st.label(), st.target()))
            .toList();
    }
    return List.of();
}
```

Add to `ScenarioControlResource`:

```java
@GET
@Path("/outline")
public List<OutlineNode> outline() {
    var result = orchestrator.outline();
    if (result.isEmpty() && orchestrator.sessionId() == null) {
        throw new NotFoundException("No active scenario");
    }
    return result;
}
```

- [ ] **Step 9: Run all tests, commit**

Run: `mvn test -pl backend/scenario-runtime -f backend/pom.xml`

```bash
git add backend/scenario-runtime/ backend/scenario/
git commit -m "feat(#341): scenario:state broadcast, stop(), outline endpoint

Refs #341"
```

---

## Batch 2: Frontend — scenario-controller component

### Task 3: Add lit dependency to pages-aria and scaffold controller

**Files:**
- Modify: `packages/pages-aria/package.json` — add `lit` dependency
- Create: `packages/pages-aria/src/controller/index.ts`
- Create: `packages/pages-aria/src/controller/scenario-controller.ts`
- Create: `packages/pages-aria/src/controller/scenario-controller.test.ts`
- Modify: `packages/pages-aria/src/index.ts` — add controller export

**Interfaces:**
- Consumes: `EventConnection` from `@casehubio/pages-data`
- Produces: `<scenario-controller>` custom element with properties: `connection`, `eventTarget`, `baseUrl`

- [ ] **Step 1: Add lit dependency**

```bash
yarn workspace @casehubio/pages-aria add lit
```

- [ ] **Step 2: Write failing test for component registration**

```typescript
// scenario-controller.test.ts
import { describe, it, expect } from 'vitest';
import './scenario-controller.js';

describe('scenario-controller', () => {
  it('registers as custom element', () => {
    expect(customElements.get('scenario-controller')).toBeDefined();
  });

  it('renders empty state when no connection', async () => {
    const el = document.createElement('scenario-controller');
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.shadowRoot?.textContent).toContain('No connection configured');
    el.remove();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-aria run test -- --reporter=verbose`
Expected: FAIL — module not found

- [ ] **Step 4: Create the component skeleton**

```typescript
// scenario-controller.ts
import { LitElement, html, css } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { EventConnection } from '@casehubio/pages-data';

interface ControllerState {
  scenario: string | null;
  chapter: string | null;
  section: string | null;
  step: string | null;
  paused: boolean;
  speed: number;
  progress: number;
}

interface OutlineNode {
  label: string;
  target: string | null;
  children: OutlineNode[];
}

export class ScenarioController extends LitElement {
  static override styles = css`
    :host { display: block; font-family: var(--pages-font-family, system-ui, sans-serif); }
    .error { padding: var(--pages-space-4, 16px); color: var(--pages-danger-9, #dc2626); }
  `;

  @property({ attribute: false })
  connection?: EventConnection;

  @property({ attribute: false })
  eventTarget?: EventTarget;

  @property()
  baseUrl?: string;

  @state() private _state: ControllerState = {
    scenario: null, chapter: null, section: null,
    step: null, paused: false, speed: 1.0, progress: 0,
  };

  @state() private _outline: OutlineNode[] = [];
  @state() private _connectionStatus: string = 'disconnected';

  private _ownConnection?: EventConnection;
  private _ownEventTarget?: EventTarget;

  get restBase(): string {
    if (this.baseUrl) return this.baseUrl;
    return window.location.origin;
  }

  override render() {
    if (!this.connection && !this.baseUrl) {
      return html`<div class="error">No connection configured</div>`;
    }
    return html`<div>Controller placeholder</div>`;
  }
}

if (!customElements.get('scenario-controller')) {
  customElements.define('scenario-controller', ScenarioController);
}
```

Create `index.ts`:
```typescript
export { ScenarioController } from './scenario-controller.js';
```

Add to `packages/pages-aria/src/index.ts`:
```typescript
export { ScenarioController } from './controller/index.js';
```

- [ ] **Step 5: Run test to verify it passes**

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): scaffold scenario-controller component with lit dependency

Refs #341"
```

### Task 4: Push wire state listening and REST commands

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `EventConnection.listen(['scenario:state'])`, `EventConnection.send()`
- Consumes: `createEventConnection` from `@casehubio/pages-data`
- Produces: reactive `_state` updated from push wire events
- Produces: `sendCommand(path, body?)` for REST dispatch

- [ ] **Step 1: Write failing test for state update from push event**

```typescript
it('updates state from scenario:state event', async () => {
  const el = document.createElement('scenario-controller') as ScenarioController;
  const mockTarget = new EventTarget();
  const mockConnection = {
    listen: vi.fn().mockResolvedValue({ topics: ['scenario:state'] }),
    unlisten: vi.fn().mockResolvedValue(undefined),
    send: vi.fn(),
    close: vi.fn(),
    connected: true,
    status: 'connected' as const,
  };
  el.connection = mockConnection;
  el.eventTarget = mockTarget;
  document.body.appendChild(el);
  await el.updateComplete;

  // Simulate push wire event
  mockTarget.dispatchEvent(new CustomEvent('pages-event', {
    detail: {
      topic: 'scenario:state',
      payload: {
        scenario: 'test-demo', chapter: 'Ch1', section: 'S1',
        step: 'Step 1', paused: false, speed: 1.0, progress: 0.25,
        content: null, slides: null,
      },
    },
  }));
  await el.updateComplete;

  expect(el.shadowRoot?.textContent).toContain('Step 1');
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — component doesn't listen or update state

- [ ] **Step 3: Implement lifecycle and event handling**

Add to `ScenarioController`:

```typescript
private _eventHandler = (e: Event) => {
  const detail = (e as CustomEvent).detail as { topic?: string; payload?: unknown };
  if (detail?.topic !== 'scenario:state') return;
  const payload = detail.payload as ControllerState;
  const scenarioChanged = payload.scenario !== this._state.scenario;
  this._state = { ...payload };
  if (scenarioChanged && payload.scenario) {
    void this._fetchOutline();
  }
  if (!payload.scenario) {
    this._outline = [];
  }
};

override connectedCallback(): void {
  super.connectedCallback();
  const target = this._resolveEventTarget();
  const conn = this._resolveConnection();
  if (conn && target) {
    conn.listen(['scenario:state']);
    target.addEventListener('pages-event', this._eventHandler);
    if (conn.status) this._connectionStatus = conn.status;
    void this._fetchInitialState();
  }
}

override disconnectedCallback(): void {
  super.disconnectedCallback();
  const target = this._resolveEventTarget();
  const conn = this._resolveConnection();
  if (conn) conn.unlisten(['scenario:state']);
  if (target) target.removeEventListener('pages-event', this._eventHandler);
  if (this._ownConnection) {
    this._ownConnection.close();
    this._ownConnection = undefined;
  }
}

private _resolveConnection(): EventConnection | undefined {
  if (this.connection) return this.connection;
  if (this.baseUrl && !this._ownConnection) {
    const wsUrl = this.baseUrl.replace(/^http/, 'ws') + '/ws/push';
    this._ownEventTarget = new EventTarget();
    const { createEventConnection } = await import('@casehubio/pages-data');
    this._ownConnection = createEventConnection(wsUrl, {
      config: { eventTarget: this._ownEventTarget },
    });
  }
  return this._ownConnection;
}

private _resolveEventTarget(): EventTarget | undefined {
  return this.eventTarget ?? this._ownEventTarget;
}

private async _fetchInitialState(): Promise<void> {
  try {
    const resp = await fetch(`${this.restBase}/scenario/state`);
    if (resp.ok) {
      this._state = await resp.json();
      if (this._state.scenario) void this._fetchOutline();
    }
  } catch { /* connection not ready */ }
}

private async _fetchOutline(): Promise<void> {
  try {
    const resp = await fetch(`${this.restBase}/scenario/outline`);
    if (resp.ok) this._outline = await resp.json();
  } catch { /* ignore */ }
}

private async _sendCommand(path: string, body?: object): Promise<void> {
  await fetch(`${this.restBase}/scenario${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    ...(body ? { body: JSON.stringify(body) } : {}),
  });
}
```

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Write test for REST command dispatch**

```typescript
it('sends pause command via REST', async () => {
  global.fetch = vi.fn().mockResolvedValue({ ok: true, json: () => ({}) });
  // ... set up component with connection ...
  const pauseBtn = el.shadowRoot?.querySelector('[aria-label="Pause"]');
  pauseBtn?.click();
  expect(fetch).toHaveBeenCalledWith(
    expect.stringContaining('/scenario/pause'),
    expect.objectContaining({ method: 'POST' }),
  );
});
```

- [ ] **Step 6: Run tests, commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): push wire state listening and REST command dispatch

Refs #341"
```

### Task 5: Outline tree and transport controls rendering

**Files:**
- Modify: `packages/pages-aria/src/controller/scenario-controller.ts`
- Modify: `packages/pages-aria/src/controller/scenario-controller.test.ts`

**Interfaces:**
- Consumes: `_state`, `_outline`, `_sendCommand()` from Task 4
- Produces: fully rendered outline tree, transport bar, status bar

- [ ] **Step 1: Write failing test for outline rendering**

```typescript
it('renders outline tree with current position highlighted', async () => {
  // Set up component with mock connection
  // Set state to chapter 1, section 1, step 1
  // Set outline with 2 chapters, 2 sections each

  await el.updateComplete;

  const treeItems = el.shadowRoot?.querySelectorAll('[role="treeitem"]');
  expect(treeItems?.length).toBeGreaterThan(0);

  const current = el.shadowRoot?.querySelector('.current');
  expect(current?.textContent).toContain('Step 1');
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement outline tree rendering**

Add `_renderOutline()` method that walks `_outline` recursively:

```typescript
private _renderOutline(): TemplateResult {
  if (this._outline.length === 0) {
    return html`<div class="outline-empty">No scenario running</div>`;
  }
  return html`
    <div class="outline" role="tree" aria-label="Scenario outline">
      ${this._outline.map(node => this._renderNode(node, 0))}
    </div>
  `;
}

private _renderNode(node: OutlineNode, depth: number): TemplateResult {
  const isLeaf = node.children.length === 0;
  const isCurrent = isLeaf && node.label === this._state.step;
  const isCompleted = isLeaf && this._isCompleted(node.label);

  if (isLeaf) {
    return html`
      <div class="outline-step ${isCurrent ? 'current' : ''} ${isCompleted ? 'completed' : ''}"
           role="treeitem" tabindex="-1"
           style="padding-left: ${depth * 16 + 8}px"
           @click=${() => this._sendCommand('/run-to', { label: node.label })}>
        <span class="step-icon">${isCurrent ? '●' : isCompleted ? '✓' : '○'}</span>
        ${node.label}
      </div>
    `;
  }
  return html`
    <div class="outline-group" role="group">
      <div class="outline-heading" role="treeitem" tabindex="-1"
           style="padding-left: ${depth * 16}px"
           @click=${() => this._sendCommand('/run-to', { label: node.label })}>
        ${node.label}
      </div>
      ${node.children.map(child => this._renderNode(child, depth + 1))}
    </div>
  `;
}
```

- [ ] **Step 4: Write failing test for transport controls**

```typescript
it('toggles play/pause on button click', async () => {
  // Set state.paused = true
  await el.updateComplete;
  const playBtn = el.shadowRoot?.querySelector('[aria-label="Resume"]');
  expect(playBtn).toBeDefined();

  // Set state.paused = false
  await el.updateComplete;
  const pauseBtn = el.shadowRoot?.querySelector('[aria-label="Pause"]');
  expect(pauseBtn).toBeDefined();
});
```

- [ ] **Step 5: Implement transport controls**

Add `_renderTransport()` method:

```typescript
private _renderTransport(): TemplateResult {
  const hasScenario = !!this._state.scenario;
  return html`
    <div class="transport">
      <button aria-label=${this._state.paused ? 'Resume' : 'Pause'}
              ?disabled=${!hasScenario}
              @click=${() => this._sendCommand(this._state.paused ? '/resume' : '/pause')}>
        ${this._state.paused ? '▶' : '⏸'}
      </button>
      <button aria-label="Step" ?disabled=${!hasScenario}
              @click=${() => this._sendCommand('/step')}>⏩</button>
      <input type="range" min="-2" max="1" step="0.01"
             .value=${String(Math.log10(this._state.speed))}
             ?disabled=${!hasScenario}
             aria-label="Speed" aria-valuetext="${this._state.speed.toFixed(1)}x"
             @input=${this._onSpeedChange}>
      <span class="speed-label">${this._state.speed.toFixed(1)}x</span>
      <span class="progress">${Math.round(this._state.progress * 100)}%</span>
    </div>
  `;
}

private _speedDebounce: ReturnType<typeof setTimeout> | null = null;

private _onSpeedChange(e: Event): void {
  const logVal = parseFloat((e.target as HTMLInputElement).value);
  const speed = Math.pow(10, logVal);
  if (this._speedDebounce) clearTimeout(this._speedDebounce);
  this._speedDebounce = setTimeout(() => {
    void this._sendCommand('/speed', { speed: Math.round(speed * 100) / 100 });
  }, 250);
}
```

- [ ] **Step 6: Implement status bar and wire up render()**

```typescript
private _renderStatus(): TemplateResult {
  const breadcrumb = [this._state.chapter, this._state.section, this._state.step]
    .filter(Boolean).join(' → ');
  return html`
    <div class="status-bar">
      <span class="breadcrumb">${breadcrumb || 'Idle'}</span>
      <span class="connection-status ${this._connectionStatus}">
        ● ${this._connectionStatus}
      </span>
    </div>
  `;
}

override render() {
  if (!this.connection && !this.baseUrl) {
    return html`<div class="error">No connection configured</div>`;
  }
  return html`
    ${this._renderOutline()}
    ${this._renderTransport()}
    ${this._renderStatus()}
  `;
}
```

- [ ] **Step 7: Add CSS styles**

Add comprehensive styles using pages-ui-tokens CSS custom properties for
outline tree, transport bar, status bar, current/completed step highlighting,
connection status dots.

- [ ] **Step 8: Run all tests, commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): outline tree, transport controls, status bar rendering

Refs #341"
```

---

## Batch 3: Narrative component and standalone remote page

### Task 6: `<scenario-narrative>` component

**Files:**
- Create: `packages/pages-aria/src/controller/scenario-narrative.ts`
- Create: `packages/pages-aria/src/controller/scenario-narrative.test.ts`
- Modify: `packages/pages-aria/src/controller/index.ts` — add export
- Modify: `packages/pages-aria/src/index.ts` — add export

**Interfaces:**
- Consumes: Same `EventConnection` + `EventTarget` pattern as controller
- Produces: `<scenario-narrative>` custom element rendering NarrativeContent

- [ ] **Step 1: Write failing test for inline markdown rendering**

```typescript
it('renders inline markdown content', async () => {
  const el = document.createElement('scenario-narrative') as ScenarioNarrative;
  // ... set up with mock connection + eventTarget ...
  document.body.appendChild(el);

  // Simulate state event with inline content
  mockTarget.dispatchEvent(new CustomEvent('pages-event', {
    detail: {
      topic: 'scenario:state',
      payload: {
        scenario: 'test', content: { type: 'inline', markdown: '# Hello\n\nWorld' },
      },
    },
  }));
  await el.updateComplete;

  const rendered = el.shadowRoot?.querySelector('.narrative-content');
  expect(rendered?.innerHTML).toContain('<h1>');
  expect(rendered?.textContent).toContain('Hello');
  el.remove();
});
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Implement scenario-narrative**

```typescript
import { LitElement, html, css, nothing } from 'lit';
import { property, state } from 'lit/decorators.js';
import type { EventConnection } from '@casehubio/pages-data';
import { unsafeHTML } from 'lit/directives/unsafe-html.js';

interface NarrativeState {
  type: string;
  markdown?: string;
  path?: string;
  section?: string;
  ref?: unknown;
}

export class ScenarioNarrative extends LitElement {
  static override styles = css`
    :host { display: block; }
    .narrative-content {
      padding: var(--pages-space-4, 16px);
      max-width: 680px;
      line-height: 1.6;
      font-size: var(--pages-font-size-base, 14px);
      font-family: var(--pages-font-family, system-ui, sans-serif);
    }
    .narrative-content h1 { font-size: 1.5em; margin: 0.5em 0; }
    .narrative-content h2 { font-size: 1.25em; margin: 0.5em 0; }
    .narrative-content p { margin: 0.5em 0; }
    .narrative-content code { background: var(--pages-neutral-3, #f5f5f5); padding: 2px 4px; border-radius: 3px; }
  `;

  @property({ attribute: false }) connection?: EventConnection;
  @property({ attribute: false }) eventTarget?: EventTarget;
  @property() baseUrl?: string;

  @state() private _content: NarrativeState | null = null;
  private _templateCache = new Map<string, string>();
  // ... same lifecycle pattern as controller (listen, unlisten, event handler)

  override render() {
    if (!this._content) return nothing;
    return html`<div class="narrative-content">${this._renderContent()}</div>`;
  }

  private _renderContent() {
    if (!this._content) return nothing;
    switch (this._content.type) {
      case 'inline':
        return unsafeHTML(this._parseMarkdown(this._content.markdown ?? ''));
      case 'template':
        return html`<div class="template-loading">Loading...</div>`;
      case 'slide':
        return html`<div class="slide-ref">Slide: ${String(this._content.ref)}</div>`;
      default:
        return nothing;
    }
  }

  private _parseMarkdown(md: string): string {
    // Minimal markdown: headings, paragraphs, bold, italic, code, lists
    return md
      .replace(/^### (.+)$/gm, '<h3>$1</h3>')
      .replace(/^## (.+)$/gm, '<h2>$1</h2>')
      .replace(/^# (.+)$/gm, '<h1>$1</h1>')
      .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
      .replace(/\*(.+?)\*/g, '<em>$1</em>')
      .replace(/`(.+?)`/g, '<code>$1</code>')
      .replace(/^- (.+)$/gm, '<li>$1</li>')
      .replace(/(<li>.*<\/li>)/s, '<ul>$1</ul>')
      .replace(/\n\n/g, '</p><p>')
      .replace(/^(?!<[hulo])(.+)$/gm, '<p>$1</p>');
  }
}

if (!customElements.get('scenario-narrative')) {
  customElements.define('scenario-narrative', ScenarioNarrative);
}
```

- [ ] **Step 4: Write test for empty content (renders nothing)**

```typescript
it('renders nothing when content is null', async () => {
  // Simulate state with content: null
  await el.updateComplete;
  expect(el.shadowRoot?.querySelector('.narrative-content')).toBeNull();
});
```

- [ ] **Step 5: Run all tests, commit**

```bash
git add packages/pages-aria/
git commit -m "feat(#341): scenario-narrative component with markdown rendering

Refs #341"
```

### Task 7: Standalone remote page and bundle

**Files:**
- Create: `backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/remote.html`
- Create: `packages/pages-aria/src/controller/standalone.ts` — bundle entry point
- Modify: `packages/pages-aria/package.json` — add build:controller script

**Interfaces:**
- Consumes: `ScenarioController` from Task 3-5
- Produces: `/scenario/remote` page served by Quarkus
- Produces: `/scenario/controller.js` standalone bundle

- [ ] **Step 1: Create standalone entry point**

```typescript
// standalone.ts — bundle entry for remote page
import './scenario-controller.js';
import './scenario-narrative.js';
```

- [ ] **Step 2: Add build script to package.json**

Add to `packages/pages-aria/package.json` scripts:

```json
"build:controller": "esbuild src/controller/standalone.ts --bundle --format=esm --outfile=dist/controller.js --external:nothing"
```

Or use the existing build pipeline if pages-aria has one.

- [ ] **Step 3: Create remote.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Scenario Remote</title>
  <style>
    body { margin: 0; font-family: system-ui, sans-serif;
           background: var(--pages-neutral-1, #fafafa); }
    scenario-controller { display: block; width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <scenario-controller id="ctrl"></scenario-controller>
  <script type="module">
    import '/scenario/controller.js';
    document.getElementById('ctrl').baseUrl = window.location.origin;
  </script>
</body>
</html>
```

- [ ] **Step 4: Build and verify bundle**

Run: `yarn workspace @casehubio/pages-aria run build:controller`
Verify: `dist/controller.js` exists and is loadable.

- [ ] **Step 5: Copy bundle to META-INF/resources**

Add a build step or Maven resource filter that copies
`packages/pages-aria/dist/controller.js` to
`backend/scenario-runtime/src/main/resources/META-INF/resources/scenario/controller.js`.

- [ ] **Step 6: Commit**

```bash
git add packages/pages-aria/ backend/scenario-runtime/src/main/resources/
git commit -m "feat(#341): standalone /scenario/remote page with bundled controller

Refs #341"
```

---

## Batch 4: Integration verification

### Task 8: E2E integration test

**Files:**
- Test: `packages/pages-aria/src/controller/scenario-controller.integration.test.ts`

**Interfaces:**
- Consumes: all previous tasks
- Produces: verified end-to-end flow

- [ ] **Step 1: Write integration test**

Test the full flow: create a mock WebSocket server that sends
`scenario:state` events, verify the controller updates its outline
and state display, verify REST commands are sent on button clicks.

This can use Vitest with a mock WebSocket (via `vitest-websocket-mock`
or a simple mock) rather than full Playwright, since the component
tests can verify DOM rendering without a real browser.

- [ ] **Step 2: Run the full test suite**

```bash
yarn workspace @casehubio/pages-aria run test
```

- [ ] **Step 3: Run the Java backend tests**

```bash
mvn test -pl backend/scenario,backend/scenario-runtime -f backend/pom.xml
```

- [ ] **Step 4: Build everything**

```bash
yarn build:packages
```

- [ ] **Step 5: Commit final state**

```bash
git add .
git commit -m "feat(#341): scenario controller UI — complete implementation

Closes casehubio/casehub-pages#341
Refs casehubio/parent#408"
```

---

## References

- [2026-08-21-scenario-controller-ui-design.md] — design spec this plan implements
- [2026-08-20-distributed-executor-protocol-design.md] — protocol spec (§5 Controller API, §7 Controller UI)
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioOrchestrator.java` — orchestrator (no broadcast currently)
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioControlResource.java` — REST endpoints
- `backend/scenario/src/main/java/io/casehub/pages/scenario/NarrativeContent.java` — sealed interface needing Jackson annotations
- `packages/pages-data/src/dataset/external/sources/event-connection.ts` — push wire client
- `packages/pages-aria/src/server/scenario-handler.ts` — browser executor pattern (connection + eventTarget)
- `packages/pages-ui-components/src/button/pages-button.ts` — existing Lit component pattern
- [GE-20260818-c61c29] — topicSource adapter technique
- [GE-20260816-e89cda] — composable Lit reactive controllers technique
- casehubio/casehub-pages#341 — focal issue
- casehubio/parent#408 — parent epic
