# Channel Activity Wrapper Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #137 — feat(channel-activity): add \<blocks-channel-activity\> convenience wrapper
**Issue group:** #137

**Goal:** Add a `<blocks-channel-activity>` convenience wrapper that composes channel-activity sub-components into a default chat layout with controller lifecycle, event routing, and tabbed sidebar.

**Architecture:** The wrapper extends `KeyboardShortcutMixin(LiveRegionMixin(LitElement))` and renders `blocks-split-workbench` internally. It creates domain controllers from a host-provided `PushController`, routes all 18 channel event topics to controller `handleEvent()` methods, and coordinates sibling sub-component state (channel selection, topic/view-mode changes, message selection). A collapsible tabbed sidebar provides Members, Tasks, Artifacts, and Links panels with lazy creation.

**Tech Stack:** Lit 3, TypeScript, pages-event coordination, pages-primitives a11y mixins

## Global Constraints

- All custom elements use `blocks-` prefix
- ARIA is mandatory — role, aria-label, aria-expanded, tablist/tab/tabpanel for sidebar
- Protocol PP-20260713-8ea1af: slots for layout shells only, content via typed properties + callbacks
- `--pages-*` CSS custom properties from pages-ui-tokens
- Tests use vitest with happy-dom

---

## Batch 1: Core wrapper with controller lifecycle and event routing

### Task 1: Component shell with split-workbench composition

**Files:**
- Create: `components/channel-activity/src/blocks-channel-activity.ts`
- Create: `components/channel-activity/src/blocks-channel-activity.test.ts`
- Modify: `components/channel-activity/src/index.ts`

**Interfaces:**
- Consumes: `blocks-split-workbench` (slot="list", slot="detail", selection-topic attribute), `blocks-channel-feed`, `blocks-channel-nav`, `blocks-channel-input`, `blocks-channel-topic-bar` — all existing custom elements
- Produces: `BlocksChannelActivityElement` class, `<blocks-channel-activity>` custom element with properties: `selectionTopic: string`, `channelNavLayout: 'sidebar' | 'dropdown'`, `sidebarOpen: boolean`, `currentActorId?: string`

- [ ] **Step 1: Write failing test — component renders sub-components**

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import './blocks-channel-activity.js';

describe('blocks-channel-activity', () => {
  let el: HTMLElement;

  beforeEach(() => {
    document.body.innerHTML = '';
    el = document.createElement('blocks-channel-activity');
    document.body.appendChild(el);
  });

  it('renders split-workbench with nav and feed', async () => {
    await (el as any).updateComplete;
    const sr = el.shadowRoot!;
    const workbench = sr.querySelector('blocks-split-workbench');
    expect(workbench).toBeTruthy();
    expect(workbench!.getAttribute('selection-topic')).toBe('channel');
    const nav = sr.querySelector('blocks-channel-nav');
    expect(nav).toBeTruthy();
    const feed = sr.querySelector('blocks-channel-feed');
    expect(feed).toBeTruthy();
    const input = sr.querySelector('blocks-channel-input');
    expect(input).toBeTruthy();
    const topicBar = sr.querySelector('blocks-channel-topic-bar');
    expect(topicBar).toBeTruthy();
  });

  it('has role region and aria-label', async () => {
    await (el as any).updateComplete;
    expect(el.getAttribute('role')).toBe('region');
    expect(el.getAttribute('aria-label')).toBe('Channel activity');
  });

  it('passes selectionTopic to split-workbench', async () => {
    (el as any).selectionTopic = 'my-channel';
    await (el as any).updateComplete;
    const wb = el.shadowRoot!.querySelector('blocks-split-workbench');
    expect(wb!.getAttribute('selection-topic')).toBe('my-channel');
  });

  it('passes channelNavLayout to channel-nav', async () => {
    (el as any).channelNavLayout = 'dropdown';
    await (el as any).updateComplete;
    const nav = el.shadowRoot!.querySelector('blocks-channel-nav') as any;
    expect(nav.layout).toBe('dropdown');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test -- --reporter verbose 2>&1 | tail -20`
Expected: FAIL — `blocks-channel-activity` not defined

- [ ] **Step 3: Implement component shell**

```typescript
import { LitElement, html, css, nothing, type TemplateResult } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { LiveRegionMixin, KeyboardShortcutMixin } from '@casehubio/pages-primitives/a11y';
import { onPagesEvent } from '@casehubio/pages-component';
import type { PushController } from './push-controller.js';
import type { MessagingConfig } from './messaging-controller.js';
import type { ReactionConfig } from './reaction-controller.js';
import { ChannelStateController } from './channel-state-controller.js';
import { MessagingController } from './messaging-controller.js';
import { MembershipController } from './membership-controller.js';
import { ReactionController } from './reaction-controller.js';
import { CommitmentController } from './commitment-controller.js';
import type { QhorusChannel, QhorusMessage, QhorusTopic, ChannelMember, PresenceState, Reaction, ActorType, ArtefactRef } from './types.js';
import type { CommitmentRecord } from '@casehubio/blocks-ui-core';
import type { ChannelTree } from './channel-state-controller.js';
import './channel-feed.js';
import './channel-nav.js';
import './channel-input.js';
import './channel-topic-bar.js';
import '@casehubio/blocks-ui-split-workbench';

const Base = KeyboardShortcutMixin(LiveRegionMixin(LitElement));

@customElement('blocks-channel-activity')
export class BlocksChannelActivityElement extends Base {
  @property({ type: String, attribute: 'selection-topic' }) selectionTopic = 'channel';
  @property({ type: String }) channelNavLayout: 'sidebar' | 'dropdown' = 'sidebar';
  @property({ type: Boolean }) sidebarOpen = false;
  @property({ type: String }) currentActorId?: string;

  // Controller-mode inputs
  @property({ attribute: false }) pushController?: PushController;
  @property({ attribute: false }) messagingConfig?: MessagingConfig;
  @property({ attribute: false }) reactionConfig?: ReactionConfig;

  // Inline-mode inputs
  @property({ attribute: false }) channels: QhorusChannel[] = [];
  @property({ attribute: false }) messages: QhorusMessage[] = [];
  @property({ attribute: false }) members: ChannelMember[] = [];
  @property({ attribute: false }) presence: PresenceState[] = [];
  @property({ attribute: false }) reactions: Reaction[] = [];
  @property({ attribute: false }) commitments: Map<string, CommitmentRecord> = new Map();
  @property({ attribute: false }) channelTree?: ChannelTree;
  @property({ attribute: false }) topics: QhorusTopic[] = [];

  // Pass-through: feed
  @property({ type: Boolean }) autoScroll = true;
  @property({ type: Number }) staleCursorMinutes = 30;
  @property({ type: Boolean }) terminalDimming = true;
  @property({ type: Boolean }) eventStyling = true;
  @property({ type: String }) viewMode: 'flat' | 'threaded' | 'topics' = 'flat';
  @property({ attribute: false }) selectedMessageId?: string;
  @property({ attribute: false }) messageHighlights: Record<string, string> = {};
  @property({ attribute: false }) renderContextHeader?: () => TemplateResult;
  @property({ attribute: false }) renderContent?: (message: QhorusMessage) => TemplateResult | undefined;
  @property({ attribute: false }) formatSender?: (sender: string, actorType: ActorType) => string;

  // Pass-through: input
  @property({ type: Boolean }) showTypeSelector = false;
  @property({ type: Boolean }) showTopicSelector = false;
  @property({ attribute: false }) messageTypes?: import('./types.js').MessageType[];
  @property({ attribute: false }) allowedTypes?: import('./types.js').MessageType[];
  @property({ attribute: false }) deniedTypes?: import('./types.js').MessageType[];
  @property({ attribute: false }) renderError?: (error: string) => TemplateResult;

  // Pass-through: nav
  @property({ type: Boolean }) showCreate = true;
  @property({ type: Boolean }) showDelete = true;

  // Pass-through: sidebar
  @property({ attribute: false }) resolveArtifact?: (ref: ArtefactRef) => Promise<unknown>;

  override connectedCallback() {
    super.connectedCallback();
    this.setAttribute('role', 'region');
    this.setAttribute('aria-label', 'Channel activity');
  }

  static override styles = css`
    :host { display: flex; flex-direction: column; height: 100%; }
    .detail-area { display: flex; height: 100%; }
    .main-column { flex: 1; display: flex; flex-direction: column; min-width: 0; }
    .sidebar { width: 280px; border-left: 1px solid var(--pages-border-color, #e0e0e0); display: flex; flex-direction: column; }
    .sidebar[hidden] { display: none; }
    .sidebar-tabs { display: flex; border-bottom: 1px solid var(--pages-border-color, #e0e0e0); }
    .sidebar-tab { flex: 1; padding: 8px 4px; text-align: center; cursor: pointer; background: none; border: none; font-size: 12px; color: var(--pages-neutral-11, #555); }
    .sidebar-tab[aria-selected="true"] { color: var(--pages-accent-color, #1a73e8); border-bottom: 2px solid var(--pages-accent-color, #1a73e8); }
    .sidebar-content { flex: 1; overflow-y: auto; }
    .toggle-btn { padding: 4px 8px; border-radius: 4px; border: 1px solid var(--pages-border-color, #ccc); background: var(--pages-surface-color, #fff); cursor: pointer; font-size: 12px; color: var(--pages-text-color, #333); }
  `;

  override render() {
    return html`
      <blocks-split-workbench selection-topic=${this.selectionTopic}>
        <span slot="header">
          <button class="toggle-btn"
            aria-expanded=${this.sidebarOpen}
            aria-controls="sidebar"
            @click=${() => { this.sidebarOpen = !this.sidebarOpen; this.announce(this.sidebarOpen ? 'Sidebar opened' : 'Sidebar closed'); }}>
            ☰ Panels
          </button>
        </span>
        <blocks-channel-nav slot="list"
          .channels=${this._navChannels}
          .channelTree=${this._navChannelTree}
          .layout=${this.channelNavLayout}
          .selectedChannelId=${this._selectedChannelId}
          .showCreate=${this.showCreate}
          .showDelete=${this.showDelete}>
        </blocks-channel-nav>
        <div slot="detail" class="detail-area">
          <div class="main-column">
            <blocks-channel-topic-bar
              .topics=${this._feedTopics}
              .selectedTopicId=${this._selectedTopicId}
              .viewMode=${this.viewMode}>
            </blocks-channel-topic-bar>
            <blocks-channel-feed
              .messages=${this._feedMessages}
              .reactions=${this._feedReactions}
              .channelId=${this._selectedChannelId}
              .channelName=${this._selectedChannelName}
              .currentActorId=${this.currentActorId}
              .autoScroll=${this.autoScroll}
              .staleCursorMinutes=${this.staleCursorMinutes}
              .terminalDimming=${this.terminalDimming}
              .eventStyling=${this.eventStyling}
              .viewMode=${this.viewMode}
              .topics=${this._feedTopics}
              .selectedMessageId=${this.selectedMessageId}
              .messageHighlights=${this.messageHighlights}
              .renderContextHeader=${this.renderContextHeader}
              .renderContent=${this.renderContent}
              .formatSender=${this.formatSender}>
            </blocks-channel-feed>
            <blocks-channel-input
              .channelId=${this._selectedChannelId}
              .replyTo=${this._replyTo}
              .showTypeSelector=${this.showTypeSelector}
              .showTopicSelector=${this.showTopicSelector}
              .topics=${this._feedTopics}
              .topicId=${this._selectedTopicId ?? ''}
              .messageTypes=${this.messageTypes ?? []}
              .allowedTypes=${this.allowedTypes}
              .deniedTypes=${this.deniedTypes}
              .renderError=${this.renderError}>
            </blocks-channel-input>
          </div>
          ${this._renderSidebar()}
        </div>
      </blocks-split-workbench>
    `;
  }

  // --- Internal state ---
  @state() private _selectedChannelId = '';
  @state() private _selectedChannelName?: string;
  @state() private _selectedTopicId: string | null = null;
  @state() private _replyTo?: { messageId: string; senderName: string };
  @state() private _activeSidebarTab = 'members';
  private _sidebarPanels = new Map<string, HTMLElement>();

  // Controller instances (created internally)
  private _channels?: ChannelStateController;
  private _messaging?: MessagingController;
  private _membership?: MembershipController;
  private _reactions?: ReactionController;
  private _commitmentCtrl?: CommitmentController;
  private _unsubs: (() => void)[] = [];

  // --- Data accessors (controller or inline) ---
  private get _navChannels(): QhorusChannel[] {
    return this._channels?.channels ?? this.channels;
  }
  private get _navChannelTree(): ChannelTree | undefined {
    return this._channels?.channelTree ?? this.channelTree;
  }
  private get _feedMessages(): QhorusMessage[] {
    return this._channels?.messages ?? this.messages;
  }
  private get _feedReactions(): Reaction[] {
    return this._reactions?.filteredReactions() ?? this.reactions;
  }
  private get _feedTopics(): QhorusTopic[] {
    return this._channels?.topics ?? this.topics;
  }

  // Sidebar render — stub for Task 3
  private _renderSidebar(): TemplateResult | typeof nothing {
    return nothing;
  }
}
```

- [ ] **Step 4: Add export to index.ts**

Append to `components/channel-activity/src/index.ts`:
```typescript
export { BlocksChannelActivityElement } from './blocks-channel-activity.js';
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test -- --reporter verbose 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/blocks-channel-activity.ts components/channel-activity/src/blocks-channel-activity.test.ts components/channel-activity/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): add blocks-channel-activity shell with split-workbench composition Refs #137"
```

### Task 2: Controller lifecycle and event routing

**Files:**
- Modify: `components/channel-activity/src/blocks-channel-activity.ts`
- Modify: `components/channel-activity/src/blocks-channel-activity.test.ts`

**Interfaces:**
- Consumes: `PushController`, `ChannelStateController(host, push)`, `MessagingController(host, channels, config)`, `MembershipController(host, push, channels)`, `ReactionController(host, push, channels, config)`, `CommitmentController(host, push, channels)`, `ChannelEventTopics` (18 topics), `onPagesEvent(target, topic, handler)`
- Produces: Controller creation on `pushController` set, teardown on disconnect, event routing for all 18 topics, internal coordination (channel/topic/message/artifact selection)

- [ ] **Step 1: Write failing tests — controller lifecycle**

Add to test file:
```typescript
import { PushController } from './push-controller.js';

describe('controller lifecycle', () => {
  it('creates domain controllers when pushController is set', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    const push = new PushController(el);
    el.pushController = push;
    await el.updateComplete;
    expect(el._channels).toBeTruthy();
    expect(el._messaging).toBeUndefined(); // no messagingConfig
    expect(el._membership).toBeTruthy();
    expect(el._commitmentCtrl).toBeTruthy();
    document.body.removeChild(el);
  });

  it('creates MessagingController when messagingConfig is provided', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    const push = new PushController(el);
    el.pushController = push;
    el.messagingConfig = { restBase: '/api' };
    await el.updateComplete;
    expect(el._messaging).toBeTruthy();
    document.body.removeChild(el);
  });

  it('tears down controllers on disconnect', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    const push = new PushController(el);
    el.pushController = push;
    await el.updateComplete;
    document.body.removeChild(el);
    expect(el._channels).toBeUndefined();
  });
});
```

- [ ] **Step 2: Write failing tests — event routing and internal coordination**

```typescript
import { emitPagesEvent } from '@casehubio/pages-component';

describe('event routing', () => {
  it('routes channel:selected to internal state', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    const push = new PushController(el);
    el.pushController = push;
    el.messagingConfig = { restBase: '/api' };
    await el.updateComplete;

    // Simulate channel data
    push.applyOp({ op: 'snapshot', dataset: 'channels', rows: [['ch1', 'General', '', '', '', '', 0]] });
    await el.updateComplete;

    emitPagesEvent(document, 'channel:selected', { channelId: 'ch1' });
    await el.updateComplete;
    expect(el._selectedChannelId).toBe('ch1');
    document.body.removeChild(el);
  });

  it('routes channel:view-mode to viewMode state', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    emitPagesEvent(document, 'channel:view-mode', { mode: 'threaded' });
    await el.updateComplete;
    expect(el.viewMode).toBe('threaded');
    document.body.removeChild(el);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test -- --reporter verbose 2>&1 | tail -30`
Expected: FAIL

- [ ] **Step 4: Implement controller lifecycle**

Add to `blocks-channel-activity.ts` — `updated()` method:

```typescript
override updated(changed: Map<string, unknown>) {
  super.updated(changed);
  if (changed.has('pushController') || changed.has('messagingConfig') || changed.has('reactionConfig')) {
    this._teardownControllers();
    this._setupControllers();
  }
}

private _setupControllers() {
  const push = this.pushController;
  if (!push) return;
  this._channels = new ChannelStateController(this, push);
  this._membership = new MembershipController(this, push, this._channels);
  this._commitmentCtrl = new CommitmentController(this, push, this._channels);
  if (this.messagingConfig) {
    this._messaging = new MessagingController(this, this._channels, this.messagingConfig);
  }
  if (this.reactionConfig) {
    this._reactions = new ReactionController(this, push, this._channels, this.reactionConfig);
  }
  this._wireEventRouting();
}

private _teardownControllers() {
  for (const unsub of this._unsubs) unsub();
  this._unsubs = [];
  this._channels = undefined;
  this._messaging = undefined;
  this._membership = undefined;
  this._reactions = undefined;
  this._commitmentCtrl = undefined;
}

override disconnectedCallback() {
  this._teardownControllers();
  super.disconnectedCallback();
}
```

- [ ] **Step 5: Implement event routing and internal coordination**

```typescript
private _wireEventRouting() {
  // Controller event routing
  if (this._channels) {
    this._unsubs.push(
      onPagesEvent(document, ChannelEventTopics.SELECT_CHANNEL, (p: any) => {
        this._channels!.handleEvent(ChannelEventTopics.SELECT_CHANNEL, p);
        this._selectedChannelId = p.channelId;
        this._selectedChannelName = this._channels!.channels.find(c => c.id === p.channelId)?.name;
        this._selectedTopicId = null;
        this._replyTo = undefined;
        this.announce(`Switched to ${this._selectedChannelName ?? p.channelId}`);
      }),
      onPagesEvent(document, ChannelEventTopics.SELECT_TOPIC, (p: any) => {
        this._channels!.handleEvent(ChannelEventTopics.SELECT_TOPIC, p);
        this._selectedTopicId = p.topicId;
      }),
      onPagesEvent(document, ChannelEventTopics.VIEW_MODE, (p: any) => {
        this._channels!.handleEvent(ChannelEventTopics.VIEW_MODE, p);
        this.viewMode = p.mode;
      }),
      onPagesEvent(document, ChannelEventTopics.CREATE_CHANNEL, p => this._channels!.handleEvent(ChannelEventTopics.CREATE_CHANNEL, p)),
      onPagesEvent(document, ChannelEventTopics.DELETE_CHANNEL, p => this._channels!.handleEvent(ChannelEventTopics.DELETE_CHANNEL, p)),
    );
  }
  if (this._messaging) {
    this._unsubs.push(
      onPagesEvent(document, ChannelEventTopics.SEND_MESSAGE, p => this._messaging!.handleEvent(ChannelEventTopics.SEND_MESSAGE, p)),
      onPagesEvent(document, ChannelEventTopics.MESSAGE_SELECTED, (p: any) => {
        this._messaging!.handleEvent(ChannelEventTopics.MESSAGE_SELECTED, p);
        this._replyTo = this._messaging!.replyTo;
        this.selectedMessageId = p.message.id;
      }),
      onPagesEvent(document, ChannelEventTopics.CURSOR_CATCHUP, (p: any) => {
        this._messaging!.handleEvent(ChannelEventTopics.CURSOR_CATCHUP, p);
        this.announce('Catching up on messages');
      }),
      onPagesEvent(document, ChannelEventTopics.CURSOR_RELOAD, (p: any) => {
        this._messaging!.handleEvent(ChannelEventTopics.CURSOR_RELOAD, p);
        this.announce('Reloading messages');
      }),
      onPagesEvent(document, ChannelEventTopics.CREATE_TOPIC, p => this._messaging!.handleEvent(ChannelEventTopics.CREATE_TOPIC, p)),
      onPagesEvent(document, ChannelEventTopics.RESOLVE_TOPIC, p => this._messaging!.handleEvent(ChannelEventTopics.RESOLVE_TOPIC, p)),
      onPagesEvent(document, ChannelEventTopics.REOPEN_TOPIC, p => this._messaging!.handleEvent(ChannelEventTopics.REOPEN_TOPIC, p)),
      onPagesEvent(document, ChannelEventTopics.ARCHIVE_TOPIC, p => this._messaging!.handleEvent(ChannelEventTopics.ARCHIVE_TOPIC, p)),
      onPagesEvent(document, ChannelEventTopics.RENAME_TOPIC, p => this._messaging!.handleEvent(ChannelEventTopics.RENAME_TOPIC, p)),
      onPagesEvent(document, ChannelEventTopics.MERGE_TOPIC, p => this._messaging!.handleEvent(ChannelEventTopics.MERGE_TOPIC, p)),
    );
  }
  if (this._reactions) {
    this._unsubs.push(
      onPagesEvent(document, ChannelEventTopics.REACT, p => this._reactions!.handleEvent(ChannelEventTopics.REACT, p)),
      onPagesEvent(document, ChannelEventTopics.UNREACT, p => this._reactions!.handleEvent(ChannelEventTopics.UNREACT, p)),
    );
  }
  if (this._commitmentCtrl) {
    this._unsubs.push(
      onPagesEvent(document, ChannelEventTopics.MESSAGE_SELECTED, p => this._commitmentCtrl!.handleEvent(ChannelEventTopics.MESSAGE_SELECTED, p)),
    );
  }
  // Artifact selection (no controller — direct state update)
  this._unsubs.push(
    onPagesEvent(document, ChannelEventTopics.ARTEFACT_SELECTED, (p: any) => {
      this._selectedArtefactRef = p;
    }),
  );
}
```

Add `_selectedArtefactRef` state:
```typescript
@state() private _selectedArtefactRef?: ArtefactRef;
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test -- --reporter verbose 2>&1 | tail -30`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/blocks-channel-activity.ts components/channel-activity/src/blocks-channel-activity.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): controller lifecycle and event routing for all 18 topics Refs #137"
```

## Batch 2: Tabbed sidebar, inline mode, configure, keyboard

### Task 3: Tabbed sidebar with lazy panel creation and data wiring

**Files:**
- Modify: `components/channel-activity/src/blocks-channel-activity.ts`
- Modify: `components/channel-activity/src/blocks-channel-activity.test.ts`

**Interfaces:**
- Consumes: `blocks-channel-member-panel` (`.members`, `.presence`), `blocks-channel-task-panel` (`.messages`, `.commitments`, `.selectedMessageId`), `blocks-channel-correlation-panel` (`.messages`, `.commitments`, `.selectedMessageId`), `blocks-channel-artifact-panel` (`.selectedArtefactRef`, `.resolveArtifact`)
- Produces: Collapsible sidebar with 4 tabs, lazy panel creation, data wiring from controllers or inline props

- [ ] **Step 1: Write failing tests — sidebar rendering and tab switching**

```typescript
describe('tabbed sidebar', () => {
  it('sidebar is hidden by default', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    const sidebar = el.shadowRoot!.querySelector('.sidebar');
    expect(sidebar!.hidden).toBe(true);
    document.body.removeChild(el);
  });

  it('sidebar toggle shows sidebar', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    const toggle = el.shadowRoot!.querySelector('.toggle-btn') as HTMLElement;
    toggle.click();
    await el.updateComplete;
    const sidebar = el.shadowRoot!.querySelector('.sidebar');
    expect(sidebar!.hidden).toBe(false);
    expect(toggle.getAttribute('aria-expanded')).toBe('true');
    document.body.removeChild(el);
  });

  it('lazily creates panel elements on tab activation', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    el.sidebarOpen = true;
    document.body.appendChild(el);
    await el.updateComplete;
    const memberPanel = el.shadowRoot!.querySelector('blocks-channel-member-panel');
    expect(memberPanel).toBeTruthy();
    // Switch to tasks tab
    const tabs = el.shadowRoot!.querySelectorAll('.sidebar-tab');
    (tabs[1] as HTMLElement).click();
    await el.updateComplete;
    const taskPanel = el.shadowRoot!.querySelector('blocks-channel-task-panel');
    expect(taskPanel).toBeTruthy();
    document.body.removeChild(el);
  });

  it('sidebar tabs have correct ARIA', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    el.sidebarOpen = true;
    document.body.appendChild(el);
    await el.updateComplete;
    const tablist = el.shadowRoot!.querySelector('[role="tablist"]');
    expect(tablist).toBeTruthy();
    const tabs = el.shadowRoot!.querySelectorAll('[role="tab"]');
    expect(tabs.length).toBe(4);
    const selectedTab = el.shadowRoot!.querySelector('[aria-selected="true"]');
    expect(selectedTab).toBeTruthy();
    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAIL — sidebar stub returns `nothing`

- [ ] **Step 3: Implement sidebar rendering**

Replace `_renderSidebar()` stub with:

```typescript
private static readonly SIDEBAR_TABS = [
  { id: 'members', label: 'Members', tagName: 'blocks-channel-member-panel' },
  { id: 'tasks', label: 'Tasks', tagName: 'blocks-channel-task-panel' },
  { id: 'artifacts', label: 'Artifacts', tagName: 'blocks-channel-artifact-panel' },
  { id: 'links', label: 'Links', tagName: 'blocks-channel-correlation-panel' },
] as const;

private _renderSidebar(): TemplateResult | typeof nothing {
  return html`
    <div class="sidebar" id="sidebar" ?hidden=${!this.sidebarOpen}>
      <div class="sidebar-tabs" role="tablist" aria-label="Panel tabs">
        ${BlocksChannelActivityElement.SIDEBAR_TABS.map(tab => html`
          <button class="sidebar-tab" role="tab"
            aria-selected=${this._activeSidebarTab === tab.id}
            aria-controls="sidebar-panel-${tab.id}"
            @click=${() => { this._activeSidebarTab = tab.id; this.announce(`${tab.label} tab`); }}>
            ${tab.label}
          </button>
        `)}
      </div>
      <div class="sidebar-content" role="tabpanel"
        id="sidebar-panel-${this._activeSidebarTab}"
        aria-labelledby="sidebar-tab-${this._activeSidebarTab}">
        ${this._renderSidebarPanel()}
      </div>
    </div>
  `;
}

private _renderSidebarPanel(): TemplateResult {
  const tab = BlocksChannelActivityElement.SIDEBAR_TABS.find(t => t.id === this._activeSidebarTab);
  if (!tab) return html``;

  let panel = this._sidebarPanels.get(tab.id);
  if (!panel) {
    panel = document.createElement(tab.tagName);
    this._sidebarPanels.set(tab.id, panel);
  }

  // Wire data to panel
  this._wireSidebarPanel(tab.id, panel);

  return html`${panel}`;
}

private _wireSidebarPanel(tabId: string, panel: HTMLElement) {
  const p = panel as any;
  switch (tabId) {
    case 'members':
      p.members = this._membership?.filteredMembers() ?? this.members;
      p.presence = this._membership?.presence ?? this.presence;
      break;
    case 'tasks':
    case 'links':
      p.messages = this._feedMessages;
      p.commitments = this._commitmentCtrl?.commitments ?? this.commitments;
      p.selectedMessageId = this.selectedMessageId;
      break;
    case 'artifacts':
      p.selectedArtefactRef = this._selectedArtefactRef;
      p.resolveArtifact = this.resolveArtifact;
      break;
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test -- --reporter verbose 2>&1 | tail -30`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/blocks-channel-activity.ts components/channel-activity/src/blocks-channel-activity.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): tabbed sidebar with lazy panel creation and data wiring Refs #137"
```

### Task 4: configure(), keyboard shortcuts, and inline data mode tests

**Files:**
- Modify: `components/channel-activity/src/blocks-channel-activity.ts`
- Modify: `components/channel-activity/src/blocks-channel-activity.test.ts`

**Interfaces:**
- Consumes: existing component from Tasks 1-3
- Produces: `configure()` method, keyboard shortcuts (`?`, `Escape`, `m`), inline data mode tests

- [ ] **Step 1: Write failing tests**

```typescript
describe('configure()', () => {
  it('sets properties from config object', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    el.configure({ selectionTopic: 'custom', channelNavLayout: 'dropdown', currentActorId: 'user1' });
    await el.updateComplete;
    expect(el.selectionTopic).toBe('custom');
    expect(el.channelNavLayout).toBe('dropdown');
    expect(el.currentActorId).toBe('user1');
    document.body.removeChild(el);
  });
});

describe('inline data mode', () => {
  it('passes channels and messages to sub-components without controllers', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    el.channels = [{ id: 'ch1', name: 'General' }];
    el.messages = [{ id: 'm1', channelId: 'ch1', content: 'hello', sender: 'alice' }];
    document.body.appendChild(el);
    await el.updateComplete;
    const nav = el.shadowRoot!.querySelector('blocks-channel-nav') as any;
    expect(nav.channels.length).toBe(1);
    const feed = el.shadowRoot!.querySelector('blocks-channel-feed') as any;
    expect(feed.messages.length).toBe(1);
    document.body.removeChild(el);
  });
});

describe('keyboard shortcuts', () => {
  it('m toggles sidebar', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    document.body.appendChild(el);
    await el.updateComplete;
    expect(el.sidebarOpen).toBe(false);
    el.dispatchEvent(new KeyboardEvent('keydown', { key: 'm' }));
    await el.updateComplete;
    expect(el.sidebarOpen).toBe(true);
    document.body.removeChild(el);
  });

  it('Escape closes sidebar when open', async () => {
    const el = document.createElement('blocks-channel-activity') as any;
    el.sidebarOpen = true;
    document.body.appendChild(el);
    await el.updateComplete;
    el.dispatchEvent(new KeyboardEvent('keydown', { key: 'Escape' }));
    await el.updateComplete;
    expect(el.sidebarOpen).toBe(false);
    document.body.removeChild(el);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Expected: FAIL — `configure` not defined, keyboard handlers not wired

- [ ] **Step 3: Implement configure() and keyboard shortcuts**

Add to `blocks-channel-activity.ts`:

```typescript
configure(props: Record<string, unknown>) {
  Object.assign(this, props);
  this.requestUpdate();
}
```

Add keyboard handler in `connectedCallback()`:

```typescript
this.addEventListener('keydown', this._handleKeydown);
```

Add cleanup in `disconnectedCallback()`:

```typescript
this.removeEventListener('keydown', this._handleKeydown);
```

Add handler:

```typescript
private _handleKeydown = (e: KeyboardEvent) => {
  if (e.key === 'm' && !e.ctrlKey && !e.metaKey && !e.altKey) {
    e.preventDefault();
    this.sidebarOpen = !this.sidebarOpen;
    this.announce(this.sidebarOpen ? 'Sidebar opened' : 'Sidebar closed');
  }
  if (e.key === 'Escape') {
    if (this.sidebarOpen) {
      this.sidebarOpen = false;
      this.announce('Sidebar closed');
    }
  }
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test -- --reporter verbose 2>&1 | tail -30`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `GH_PACKAGES_TOKEN="" yarn workspace @casehubio/blocks-ui-channel-activity test 2>&1 | tail -10`
Expected: All tests PASS (existing + new)

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/blocks-channel-activity.ts components/channel-activity/src/blocks-channel-activity.test.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): configure(), keyboard shortcuts, inline data mode tests Refs #137"
```

## References

- specs/issue-137-channel-activity-wrapper/2026-08-26-channel-activity-wrapper-design.md — design spec
- components/channel-activity/src/channel-state-controller.ts — ChannelStateController constructor, channelTree getter
- components/channel-activity/src/messaging-controller.ts — MessagingController constructor, handleEvent routing
- components/channel-activity/src/membership-controller.ts — MembershipController constructor, filteredMembers()
- components/channel-activity/src/reaction-controller.ts — ReactionController constructor, filteredReactions()
- components/channel-activity/src/commitment-controller.ts — CommitmentController constructor, commitments Map
- components/channel-activity/src/events.ts — ChannelEventTopics (18 topics), payload interfaces
- components/channel-activity/src/channel-feed.ts:23-43 — feed property declarations
- components/channel-activity/src/channel-nav.ts:11-17 — nav property declarations
- components/channel-activity/src/channel-input.ts:9-20 — input property declarations
- components/channel-activity/src/channel-topic-bar.ts:9-12 — topic-bar property declarations
- components/channel-activity/src/push-controller.ts — PushController (transport-agnostic)
- docs/protocols/blocks-ui/component-customisation-pattern.md (PP-20260713-8ea1af)
- casehubio/blocks-ui#137
