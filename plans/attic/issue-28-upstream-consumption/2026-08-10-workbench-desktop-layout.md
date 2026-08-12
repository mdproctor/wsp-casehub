# Workbench Desktop Layout Migration — Implementation Plan (Slice 1)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #10 — qhorus-chat-ui: Migrate workbench to split/dockBar layout
**Issue group:** #28, #29, #30, #31, #5, #10

**Goal:** Replace the desktop layout in qhorus-workbench with pages-runtime
`dockWorkbench()`, giving resizable split panes, dock strip, exclusive panel
switching, and LayoutStore persistence.

**Architecture:** The workbench stays as a Lit element but delegates desktop
layout to pages-runtime. `renderComponent()` renders a `dockWorkbench()` tree
into the shadow root. Panel components are mounted imperatively and receive
data state via property propagation in `updated()`.

**Tech Stack:** TypeScript, Lit 3, pages-component/pages-ui/pages-runtime
(SNAPSHOT), vitest + jsdom

## Global Constraints

- Pages packages consumed via Maven SNAPSHOT + vite aliases (ADR-0001)
- All styling uses `--pages-*` design tokens exclusively
- Tablet and phone layouts are NOT in scope — they keep existing CSS for now
- Pages changes target the source repo at `/Users/mdproctor/claude/casehub/pages`
- Chat-app changes target `/Users/mdproctor/claude/casehub/slots/101/chat-app`
- IntelliJ MCP required for all code navigation and structural editing

## Scope Note

This plan covers **Slice 1 (Desktop) only**. Slices 2 (Tablet) and 3 (Phone)
will be planned separately after Slice 1 validates the pages-runtime integration.

---

### Task 1: Pages — Shadow Root Rendering

**Files:**
- Modify: `packages/pages-component/src/renderer/render.ts`
- Test: `packages/pages-component/src/renderer/render.test.ts`

**Interfaces:**
- Produces: `renderComponent(target: HTMLElement | ShadowRoot, component, options)` — widened target type

- [ ] **Step 1: Write the failing test**

In `packages/pages-component/src/renderer/render.test.ts`, add a test that
renders a component tree into a `ShadowRoot`:

```typescript
import { describe, it, expect } from 'vitest';
import { renderComponent } from './render.js';

describe('renderComponent into ShadowRoot', () => {
  it('renders a component tree into a shadow root', () => {
    const host = document.createElement('div');
    const shadow = host.attachShadow({ mode: 'open' });
    const tree = {
      id: 'root',
      type: 'container',
      props: {},
      slots: {
        default: [{
          id: 'child',
          type: 'text',
          props: { content: 'hello' },
          slots: {},
        }],
      },
    };

    renderComponent(shadow, tree);

    const rendered = shadow.querySelector('[data-component-id="root"]');
    expect(rendered).toBeTruthy();
    expect(rendered!.querySelector('[data-component-id="child"]')).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd packages/pages-component && npx vitest run src/renderer/render.test.ts -t "shadow root"`
Expected: TypeScript error — `ShadowRoot` is not assignable to `HTMLElement`

- [ ] **Step 3: Widen the target type**

Use `ide_replace_text_in_file` to change the `renderComponent` signature in
`packages/pages-component/src/renderer/render.ts`:

Change:
```typescript
export function renderComponent(
  target: HTMLElement,
```
To:
```typescript
export function renderComponent(
  target: HTMLElement | ShadowRoot,
```

Also update the internal `renderNode` function's `parent` parameter if it
constrains to `HTMLElement`. The key call is `parent.appendChild(el)` which
works on both `HTMLElement` and `ShadowRoot` since both implement `Node`.

The `target.innerHTML = ""` call also works on `ShadowRoot` (it has
`innerHTML` as a property).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd packages/pages-component && npx vitest run src/renderer/render.test.ts -t "shadow root"`
Expected: PASS

- [ ] **Step 5: Run full pages-component test suite**

Run: `cd packages/pages-component && npx vitest run`
Expected: All existing tests still pass

- [ ] **Step 6: Commit**

```
git -C /Users/mdproctor/claude/casehub/pages add packages/pages-component/src/renderer/render.ts packages/pages-component/src/renderer/render.test.ts
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: support ShadowRoot target in renderComponent()

Refs casehubio/chat-app#10"
```

---

### Task 2: Pages — Exclusive Dock Mode

**Files:**
- Modify: `packages/pages-ui/src/dsl/builders.ts` (DockSideConfig, dockWorkbench)
- Modify: `packages/pages-component/src/model/component-props.ts` (DockBarProps)
- Modify: `packages/pages-runtime/src/site.ts` (dock-toggle handler)
- Test: `packages/pages-ui/src/dsl/builders.test.ts`
- Test: `packages/pages-runtime/src/site.test.ts` (if exists)

**Interfaces:**
- Consumes: `DockSideConfig`, `DockWorkbenchConfig` from Task 1's repo
- Produces: `DockSideConfig.exclusive?: boolean` — when true, only one panel
  per side can be open. Toggling a new panel closes the previous one.
  `pages-dock-toggle` event handler respects exclusive mode.

- [ ] **Step 1: Explore DockSideConfig**

Use `ide_find_class` or `ide_search_text` to find the `DockSideConfig`
interface definition. Determine its current shape and where `exclusive`
should be added.

Also explore the `pages-dock-toggle` event handler in `site.ts` to understand
how dock toggles currently work — what state they modify and how panel
visibility is managed.

- [ ] **Step 2: Write the failing test for exclusive dock tree**

In `packages/pages-ui/src/dsl/builders.test.ts`:

```typescript
describe('dockWorkbench exclusive mode', () => {
  it('passes exclusive flag through to dock-bar props', () => {
    const tree = dockWorkbench({
      centre: { id: 'c', type: 'text', props: { content: 'centre' }, slots: {} },
      left: {
        exclusive: true,
        panels: [
          { panelId: 'a', icon: 'A', label: 'Panel A', defaultOpen: true },
          { panelId: 'b', icon: 'B', label: 'Panel B', defaultOpen: false },
        ],
      },
    });

    // Find the dock-bar component in the tree
    const dockBar = findComponentByType(tree, 'dock-bar');
    expect(dockBar).toBeTruthy();
    expect(dockBar!.props.exclusive).toBe(true);
  });
});
```

Write a `findComponentByType` helper that walks the component tree recursively.

- [ ] **Step 3: Run test to verify it fails**

Run: `cd packages/pages-ui && npx vitest run src/dsl/builders.test.ts -t "exclusive"`
Expected: FAIL — `exclusive` not in DockSideConfig or not passed through

- [ ] **Step 4: Add exclusive to DockSideConfig**

Use `ide_search_text` to find `DockSideConfig` in `builders.ts`. Add
`exclusive?: boolean` to the interface. Then update `normalizeConfig()` or
`buildTreeFromZones()` to pass the flag through to the dock-bar component's
props.

```typescript
export interface DockSideConfig {
  // ... existing fields
  readonly exclusive?: boolean;
}
```

In the tree-building logic, when creating the dock-bar component for a side,
set `props.exclusive = sideConfig.exclusive ?? false`.

- [ ] **Step 5: Run test to verify it passes**

Run: `cd packages/pages-ui && npx vitest run src/dsl/builders.test.ts -t "exclusive"`
Expected: PASS

- [ ] **Step 6: Implement exclusive toggle behavior**

In `packages/pages-runtime/src/site.ts`, find the `pages-dock-toggle` event
handler. When the dock-bar has `exclusive: true` in its props:
- On toggle-open: close any other currently-open panel in the same dock group
  before opening the new one
- On toggle-close: just close the panel (no replacement)

The handler likely uses `display: none`/`display: ''` to show/hide panels. For
exclusive mode, iterate all panel containers in the dock group and set
`display: none`, then set the target panel to `display: ''`.

Write a test for this behavior if `site.test.ts` exists, or add one to the
builders test as an integration test.

- [ ] **Step 7: Run full test suite**

Run: `cd packages/pages-ui && npx vitest run`
Run: `cd packages/pages-runtime && npx vitest run`
Expected: All tests pass

- [ ] **Step 8: Commit**

```
git -C /Users/mdproctor/claude/casehub/pages add -A
git -C /Users/mdproctor/claude/casehub/pages commit -m "feat: exclusive dock mode for dockWorkbench side panels

When DockSideConfig.exclusive is true, only one panel per side can be
open at a time. Toggling a new panel closes the previous one.

Refs casehubio/chat-app#10"
```

---

### Task 3: Pages — Verify Split + Dock in Shadow DOM

**Files:**
- Test: `packages/pages-component/src/renderer/shadow-dom.integration.test.ts` (create)
- Modify: `packages/pages-component/src/renderer/interactive.ts` (if fixes needed)
- Modify: `packages/pages-runtime/src/activation.ts` (if fixes needed)

**Interfaces:**
- Consumes: `renderComponent()` with ShadowRoot target (Task 1)
- Produces: Verified that split handles, dock toggles, and host-panel mounting
  work inside a shadow root

- [ ] **Step 1: Write integration test**

Create `packages/pages-component/src/renderer/shadow-dom.integration.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { renderComponent } from './render.js';
import { wireSplit } from './interactive.js';

describe('shadow DOM integration', () => {
  it('split handles render inside shadow root', () => {
    const host = document.createElement('div');
    const shadow = host.attachShadow({ mode: 'open' });

    const tree = {
      id: 'split-1',
      type: 'split',
      props: { direction: 'horizontal', ratio: [1, 2] },
      slots: {
        '0': [{ id: 'left', type: 'text', props: { content: 'L' }, slots: {} }],
        '1': [{ id: 'right', type: 'text', props: { content: 'R' }, slots: {} }],
      },
    };

    renderComponent(shadow, tree);

    const splitEl = shadow.querySelector('[data-component-type="split"]');
    expect(splitEl).toBeTruthy();

    // Verify split handle was created
    const handle = shadow.querySelector('[data-split-handle]');
    expect(handle).toBeTruthy();
  });

  it('dock-bar buttons render inside shadow root', () => {
    const host = document.createElement('div');
    const shadow = host.attachShadow({ mode: 'open' });

    const tree = {
      id: 'dock-1',
      type: 'dock-bar',
      props: {
        orientation: 'vertical',
        items: [
          { icon: 'A', label: 'Panel A', panelId: 'a', defaultOpen: true },
        ],
      },
      slots: {},
    };

    renderComponent(shadow, tree);

    const dockEl = shadow.querySelector('[data-component-type="dock-bar"]');
    expect(dockEl).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run integration tests**

Run: `cd packages/pages-component && npx vitest run src/renderer/shadow-dom.integration.test.ts`

If tests pass: shadow DOM integration works out of the box. Proceed to commit.

If tests fail: diagnose the issue. Common problems:
- `querySelector` calls that walk up to `document` instead of the shadow root
- Event listener attachment on `document` instead of the shadow root's host
- `getRootNode()` calls that expect `Document` but get `ShadowRoot`

Fix any issues found in `interactive.ts` or `activation.ts`.

- [ ] **Step 3: Run full test suite**

Run: `cd packages/pages-component && npx vitest run`
Expected: All tests pass

- [ ] **Step 4: Commit**

```
git -C /Users/mdproctor/claude/casehub/pages add -A
git -C /Users/mdproctor/claude/casehub/pages commit -m "test: verify split + dock rendering in shadow DOM

Integration tests confirm split handles and dock-bar buttons render
correctly when renderComponent() targets a ShadowRoot.

Refs casehubio/chat-app#10"
```

---

### Task 4: Pages — Build and Install SNAPSHOT

**Files:** None modified — build step only

**Interfaces:**
- Consumes: Tasks 1-3 (all pages changes committed)
- Produces: Updated SNAPSHOT in `~/.m2` for chat-app consumption

- [ ] **Step 1: Build pages packages**

```bash
cd /Users/mdproctor/claude/casehub/pages && yarn build
```

- [ ] **Step 2: Install SNAPSHOT to local Maven**

```bash
cd /Users/mdproctor/claude/casehub/pages && yarn build && mvn install
```

Verify the SNAPSHOT JARs are updated in `~/.m2/repository/io/casehub/casehub-pages-npm/`.

- [ ] **Step 3: Re-unpack in chat-app**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /Users/mdproctor/claude/casehub/slots/101/chat-app/pom.xml initialize
```

This refreshes `.casehub-packages` with the updated pages SNAPSHOT containing
shadow root support and exclusive dock mode.

---

### Task 5: Chat-app — Desktop Tree Builder and Shell Integration

**Files:**
- Modify: `src/main/webui/src/workbench/qhorus-workbench.ts`
- Test: `src/main/webui/src/workbench/qhorus-workbench.test.ts`

**Interfaces:**
- Consumes: `dockWorkbench()` from `@casehubio/pages-ui`, `renderComponent()`
  from `@casehubio/pages-component`, `createLocalLayoutStore()` from
  `@casehubio/pages-runtime`, `DockItem` and `LayoutState` from
  `@casehubio/pages-component`
- Produces: Desktop layout rendered via pages-runtime with resizable splits,
  exclusive dock, and LayoutStore persistence. Tablet/phone unchanged.

- [ ] **Step 1: Write test for desktop tree structure**

In `src/main/webui/src/workbench/qhorus-workbench.test.ts`, add tests for the
new tree builder. Since `dockWorkbench()` returns a pure data structure, test
the tree shape without DOM rendering:

```typescript
describe('desktop tree builder', () => {
  it('produces a dockWorkbench tree with exclusive left and right zones', () => {
    // Import the tree builder function (will be extracted as a testable function)
    const tree = buildDesktopTree();

    expect(tree).toBeTruthy();
    expect(tree.type).toBe('columns'); // dockWorkbench wraps in columns
    // Verify the tree contains dock-bar, split, and zone components
  });
});
```

The exact assertions depend on the tree shape produced by `dockWorkbench()`.
Build the test after exploring the actual output in Step 3.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd src/main/webui && node_modules/.bin/vitest run src/workbench/qhorus-workbench.test.ts -t "desktop tree"`
Expected: FAIL — `buildDesktopTree` not defined

- [ ] **Step 3: Implement buildDesktopTree()**

In `qhorus-workbench.ts`, add the tree builder as a module-level function
(not a method — keeps it testable):

```typescript
import { dockWorkbench } from '@casehubio/pages-ui';
import { renderComponent } from '@casehubio/pages-component';
import type { Component } from '@casehubio/pages-component';

function buildDesktopTree(chatContent: Component): Component {
  return dockWorkbench({
    centre: chatContent,
    left: {
      exclusive: true,
      panels: [
        { panelId: 'nav', icon: '💬', label: 'Channels', defaultOpen: true },
        { panelId: 'tasks', icon: '📋', label: 'Tasks', defaultOpen: false },
      ],
    },
    right: {
      exclusive: true,
      panels: [
        { panelId: 'members', icon: '👥', label: 'Members', defaultOpen: true },
        { panelId: 'correlation', icon: '🔗', label: 'Correlation', defaultOpen: false },
        { panelId: 'artifacts', icon: '📎', label: 'Artifacts', defaultOpen: false },
      ],
    },
  });
}

export { buildDesktopTree }; // for testing
```

The `chatContent` parameter is a placeholder `Component` node that represents
the centre zone (channel-feed + input + topic-bar). Build it as a simple
container with a known `id` so the shell can mount Lit elements into it.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd src/main/webui && node_modules/.bin/vitest run src/workbench/qhorus-workbench.test.ts -t "desktop tree"`
Expected: PASS

- [ ] **Step 5: Integrate into workbench render cycle**

Modify `QhorusWorkbenchElement._renderDesktop()` to use the tree builder
and `renderComponent()`:

```typescript
private _renderDesktop() {
  const chatContent: Component = {
    id: 'chat-centre',
    type: 'container',
    props: {},
    slots: {},
  };

  const tree = buildDesktopTree(chatContent);
  const shadow = this.renderRoot as ShadowRoot;

  // Clear previous render
  shadow.innerHTML = '';

  // Render the pages tree
  renderComponent(shadow, tree);

  // Mount Lit panel components into zones
  this._mountPanels(shadow);

  // Mount chat content into centre
  this._mountChatContent(shadow);

  // Return nothing — Lit's render() is bypassed for desktop
  return nothing;
}
```

- [ ] **Step 6: Implement panel mounting**

Add `_mountPanels()` and `_mountChatContent()` methods:

```typescript
private _panelRefs = new Map<string, HTMLElement>();

private _mountPanels(shadow: ShadowRoot) {
  const panelConfigs: Record<string, { tag: string; propSetter: (el: any) => void }> = {
    nav: {
      tag: 'div', // nav content rendered via _renderNav()
      propSetter: () => {},
    },
    tasks: {
      tag: 'qhorus-task-panel',
      propSetter: (el: QhorusTaskPanelElement) => {
        el.messages = this._filteredMessages();
        el.commitments = this._commitments;
        el.selectedMessageId = this._selectedMessageId;
      },
    },
    members: {
      tag: 'channel-member-panel',
      propSetter: (el: ChannelMemberPanelElement) => {
        el.members = this._filteredMembers();
        el.presence = this._presence;
      },
    },
    correlation: {
      tag: 'qhorus-correlation-panel',
      propSetter: (el: QhorusCorrelationPanelElement) => {
        el.messages = this._filteredMessages();
        el.commitments = this._commitments;
        el.selectedMessageId = this._selectedMessageId;
      },
    },
    artifacts: {
      tag: 'qhorus-artifact-panel',
      propSetter: (el: QhorusArtifactPanelElement) => {
        el.selectedArtefactRef = this._selectedArtefactRef;
      },
    },
  };

  for (const [panelId, config] of Object.entries(panelConfigs)) {
    const zone = shadow.querySelector(`[data-panel-id="${panelId}"]`);
    if (!zone) continue;
    const el = document.createElement(config.tag);
    config.propSetter(el);
    zone.appendChild(el);
    this._panelRefs.set(panelId, el);
  }
}
```

Note: The exact `data-panel-id` selector depends on how `dockWorkbench()`
marks panel zones. Explore the rendered DOM to determine the correct
attribute. It may be `data-component-id` matching the `panelId`.

- [ ] **Step 7: Implement property propagation in updated()**

Add an `updated()` override that syncs data state to mounted panel elements:

```typescript
override updated(changed: Map<string, unknown>) {
  super.updated(changed);
  if (this._mode !== 'desktop') return;

  const nav = this._panelRefs.get('nav');
  if (nav) { /* update nav content */ }

  const tasks = this._panelRefs.get('tasks') as QhorusTaskPanelElement | undefined;
  if (tasks) {
    tasks.messages = this._filteredMessages();
    tasks.commitments = this._commitments;
    tasks.selectedMessageId = this._selectedMessageId;
  }

  const members = this._panelRefs.get('members') as ChannelMemberPanelElement | undefined;
  if (members) {
    members.members = this._filteredMembers();
    members.presence = this._presence;
  }

  const corr = this._panelRefs.get('correlation') as QhorusCorrelationPanelElement | undefined;
  if (corr) {
    corr.messages = this._filteredMessages();
    corr.commitments = this._commitments;
    corr.selectedMessageId = this._selectedMessageId;
  }

  const arts = this._panelRefs.get('artifacts') as QhorusArtifactPanelElement | undefined;
  if (arts) {
    arts.selectedArtefactRef = this._selectedArtefactRef;
  }
}
```

- [ ] **Step 8: Wire LayoutStore**

Replace `_dockState` with LayoutStore-backed state:

```typescript
private _layoutStore = createLocalLayoutStore('qhorus-workbench:');

override async connectedCallback() {
  super.connectedCallback();
  // ... existing setup
  const saved = await this._layoutStore.load('workbench');
  if (saved) {
    this._layoutState = saved;
  }
}
```

Listen for `pages-split-resize` and `pages-dock-toggle` events on the shadow
root to persist state changes:

```typescript
this.renderRoot.addEventListener('pages-split-resize', (e: CustomEvent) => {
  this._layoutState = {
    ...this._layoutState,
    splits: { ...this._layoutState.splits, [e.detail.componentId]: e.detail.ratios },
  };
  this._layoutStore.save('workbench', this._layoutState);
});

this.renderRoot.addEventListener('pages-dock-toggle', (e: CustomEvent) => {
  this._layoutState = {
    ...this._layoutState,
    docks: { ...this._layoutState.docks, [e.detail.panelId]: e.detail.open },
  };
  this._layoutStore.save('workbench', this._layoutState);
});
```

- [ ] **Step 9: Remove desktop layout CSS**

Remove these CSS rules from `static styles`:
- `.nav-panel` (width: 240px, flex-shrink: 0, border)
- `.main-panel` (flex: 1, display: flex)
- `.member-panel` (width: 220px, flex-shrink: 0, border)
- `.dock-strip` (display: flex, width: 48px, etc.)
- `.dock-btn` and `.dock-btn:hover`, `.dock-btn.active`
- `channel-feed` and `channel-input` flex rules (if handled by pages)

Keep:
- `:host` with minimal `display: block; height: 100%;`
- `.connection-banner` styles (not layout-related)
- Tablet and phone CSS (unchanged in Slice 1)
- `@keyframes spin` for connection spinner

- [ ] **Step 10: Remove _renderDockStrip()**

Delete `_renderDockStrip()` method — the dock strip is now rendered by the
pages `dockBar()` component in the tree. Delete `DOCK_ITEMS` static array
(moved into `buildDesktopTree()`).

- [ ] **Step 11: Update _toggleDock() for desktop mode**

The desktop branch of `_toggleDock()` can be simplified — the dock-bar
component emits `pages-dock-toggle` events which are handled by the
LayoutStore listener (Step 8). The workbench only needs to catch the event
for app-level side effects (e.g., auto-opening artifacts panel on artefact
selection).

- [ ] **Step 12: Write integration test**

Add a test that verifies the full desktop render cycle:

```typescript
describe('desktop layout via pages-runtime', () => {
  it('renders dock strip, left zone, centre, and right zone', async () => {
    const el = document.createElement('qhorus-workbench') as QhorusWorkbenchElement;
    // Force desktop mode
    // ... set up media query mock for desktop
    document.body.appendChild(el);
    await el.updateComplete;

    const shadow = el.shadowRoot!;
    expect(shadow.querySelector('[data-component-type="dock-bar"]')).toBeTruthy();
    expect(shadow.querySelector('[data-component-type="split"]')).toBeTruthy();
  });
});
```

- [ ] **Step 13: Run full test suite**

Run: `cd src/main/webui && node_modules/.bin/vitest run`
Expected: All tests pass (desktop tests updated, tablet/phone tests unchanged)

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f /path/to/chat-app/pom.xml test -Dquarkus.http.test-port=0`
Expected: All Java tests pass (backend unchanged)

- [ ] **Step 14: Visual verification**

Start the dev server and verify in a browser:
- Dock strip appears on far left with 5 panel buttons + theme toggle
- Nav panel opens in left zone (default open)
- Members panel opens in right zone (default open)
- Clicking Tasks button in dock strip replaces nav with tasks (exclusive)
- Clicking Tasks again collapses the left zone
- Split handles between zones are draggable
- Resizing persists across page reload (LayoutStore)
- Theme toggle still works
- Connection banner still appears on disconnect

- [ ] **Step 15: Commit**

```
git -C /path/to/chat-app add src/main/webui/src/workbench/qhorus-workbench.ts src/main/webui/src/workbench/qhorus-workbench.test.ts
git -C /path/to/chat-app commit -m "feat(#10): migrate desktop layout to pages-runtime dockWorkbench

Desktop layout now uses pages-runtime renderComponent() with
dockWorkbench() tree. Resizable split panes replace fixed-width panels.
Exclusive dock mode — one panel per side. LayoutStore persists split
ratios and dock state.

Tablet and phone layouts unchanged (Slices 2 and 3).

Refs #10"
```

---

## Follow-Up Plans (Not in Scope)

**Slice 2 — Tablet:** Likely requires no pages changes. Build a tablet-specific
tree with a single exclusive sidebar zone. Replace tab switcher with dock strip.
Plan after Slice 1 validates the integration.

**Slice 3 — Phone:** Requires a new pages drawer primitive with swipe gesture
support. Needs its own brainstorming cycle for the drawer design. Plan after
Slice 2.

**Prerequisite — pages-ui-tokens#297:** The workbench imports `injectTheme`,
`applyThemeMode`, `DEFAULT_THEME` from `@casehubio/pages-ui-tokens`, but these
aren't exported from the package's public API. Filed as casehub-pages#297.
Must be resolved before or during this work — either by adding the export in
pages-ui-tokens or by importing from the specific module path
(`@casehubio/pages-ui-tokens/dist/themes.js`).
