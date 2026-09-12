# Channel Activity Promotion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #39 — epic: claudony component migration — channel panel, worker panel
**Issue group:** #39

**Goal:** Promote the connectors chat-demo qhorus UI primitives into
blocks-ui as shared platform components with extension points for
domain-specific customisation.

**Architecture:** Move 8 components, types, events, and markdown helper
from `connectors/chat-demo/src/main/webui/src/qhorus/` into
`blocks-ui/components/channel-activity/`. Rename tag prefix from
`qhorus-*` to `channel-*`. Replace local `emitChatEvent` with
`emitPagesEvent` from blocks-ui-core. Add extension points (typed config
properties, render callbacks) per protocol PP-20260713-8ea1af. Close 5
gaps (type selector, terminal/event styling, auto-scroll, error feedback,
stale cursor detection).

**Tech Stack:** TypeScript, Lit 3, marked, dompurify, emoji-picker-element,
vitest, blocks-ui-core, pages-component

## Global Constraints

- Tag names: `channel-*` (not `qhorus-*`)
- Event topics: `channel:*` (not `chat:*`)
- Event emission: `emitPagesEvent` from `@casehubio/blocks-ui-core` (not local `emitChatEvent`)
- Types retain `Qhorus` prefix (`QhorusMessage`, `QhorusChannel`)
- CSS custom properties: `--pages-*` from `pages-ui-tokens`
- Extension pattern: typed config + render callbacks + factory overrides (protocol PP-20260713-8ea1af)
- No slots for content customisation (layout shells only)
- Pre-release: breaking changes cost nothing

## Source Mapping

| Connectors path (under `chat-demo/src/main/webui/src/qhorus/`) | blocks-ui target (under `components/channel-activity/src/`) |
|---|---|
| `types.ts` | `types.ts` |
| `events.ts` | `events.ts` |
| `markdown.ts` | `markdown.ts` |
| `primitives/qhorus-message.ts` | `channel-message.ts` |
| `primitives/qhorus-message-input.ts` | `channel-input.ts` |
| `primitives/qhorus-emoji-picker.ts` | `channel-emoji-picker.ts` |
| `primitives/qhorus-reaction-bar.ts` | `channel-reaction-bar.ts` |
| `primitives/qhorus-thread.ts` | `channel-thread.ts` |
| `composites/qhorus-channel-feed.ts` | `channel-feed.ts` |
| `composites/qhorus-channel-nav.ts` | `channel-nav.ts` |
| `composites/qhorus-member-panel.ts` | `channel-member-panel.ts` |

---

### Task 1: Package Scaffold and Foundation Types

**Files:**
- Create: `components/channel-activity/package.json`
- Create: `components/channel-activity/tsconfig.json`
- Create: `components/channel-activity/tsconfig.build.json`
- Create: `components/channel-activity/vitest.config.ts`
- Create: `components/channel-activity/src/types.ts`
- Create: `components/channel-activity/src/markdown.ts`
- Create: `components/channel-activity/src/index.ts`
- Modify: `tsconfig.json` (root — already has channel-activity reference)
- Test: `components/channel-activity/src/types.test.ts`
- Test: `components/channel-activity/src/markdown.test.ts`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces: `QhorusMessage`, `QhorusChannel`, `MessageType`, `ActorType`,
  `CommitmentState`, `ChannelSemantic`, `ArtefactType`, `ArtefactRef`,
  `SelectionScope`, `ChatMessageRef`, `Reaction`, `ChannelMember`,
  `PresenceState`, `MESSAGE_TYPES`, `ACTOR_TYPES`, `COMMITMENT_STATES`,
  `CHANNEL_SEMANTICS`, `ARTEFACT_TYPES`, `isTerminalMessageType`,
  `isObligationCreating`, `messageTypeCategory`, `commitmentStateCategory`,
  `renderMarkdown`

- [ ] **Step 1: Create package.json**

```json
{
  "name": "@casehubio/blocks-ui-channel-activity",
  "version": "0.1.0",
  "description": "Qhorus channel activity — message feed, channel nav, member panel, speech-act badges",
  "repository": {
    "type": "git",
    "url": "https://github.com/casehubio/blocks-ui.git"
  },
  "publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "clean": "rimraf dist"
  },
  "dependencies": {
    "@casehubio/blocks-ui-core": "workspace:*",
    "dompurify": "^3.4.11",
    "emoji-picker-element": "^1.29.1",
    "lit": "^3.3.3",
    "marked": "^15.0.12"
  },
  "devDependencies": {
    "@types/dompurify": "^3.0.5",
    "jsdom": "^29.1.1",
    "rimraf": "^6.1.0",
    "typescript": "^5.6.0",
    "vitest": "^4.1.9"
  },
  "license": "Apache-2.0"
}
```

- [ ] **Step 2: Create tsconfig.json**

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "references": [
    { "path": "../../packages/blocks-ui-core" }
  ]
}
```

- [ ] **Step 3: Create tsconfig.build.json**

```json
{
  "extends": "./tsconfig.json",
  "exclude": ["src/**/*.test.ts"]
}
```

- [ ] **Step 4: Create vitest.config.ts**

Follow the trust-score-panel pattern with aliases for local workspace deps:

```typescript
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  resolve: {
    alias: [
      { find: '@casehubio/blocks-ui-core', replacement: path.resolve(__dirname, '../../packages/blocks-ui-core/src') },
      { find: '@casehubio/pages-ui-tokens', replacement: path.resolve(__dirname, '../../../pages/packages/pages-ui-tokens/src') },
      { find: /^@casehubio\/pages-component\/dist\/(.*)/, replacement: path.resolve(__dirname, '../../../pages/packages/pages-component/src/$1') },
      { find: '@casehubio/pages-component', replacement: path.resolve(__dirname, '../../../pages/packages/pages-component/src') },
      { find: /^@casehubio\/pages-data\/dist\/(.*)/, replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src/$1') },
      { find: '@casehubio/pages-data', replacement: path.resolve(__dirname, '../../../pages/packages/pages-data/src') },
    ],
  },
  esbuild: {
    target: 'es2022',
    tsconfigRaw: {
      compilerOptions: {
        experimentalDecorators: true,
        useDefineForClassFields: false,
      },
    },
  },
  test: {
    environment: 'jsdom',
    globals: true,
  },
});
```

- [ ] **Step 5: Copy and adapt types.ts from connectors**

Source: `connectors/chat-demo/src/main/webui/src/qhorus/types.ts`

Copy verbatim — no changes needed. The types are self-contained with no
imports from connectors code. All const arrays, union types, interfaces,
and helper functions transfer as-is.

- [ ] **Step 6: Copy and adapt markdown.ts from connectors**

Source: `connectors/chat-demo/src/main/webui/src/qhorus/markdown.ts`

Copy verbatim — the `renderMarkdown` function is self-contained with only
`marked` and `dompurify` as external deps (both in package.json).

- [ ] **Step 7: Create initial index.ts**

```typescript
export * from './types.js';
export { renderMarkdown } from './markdown.js';
```

- [ ] **Step 8: Copy and adapt types.test.ts from connectors**

Source: `connectors/chat-demo/src/main/webui/src/qhorus/types.test.ts`

Copy and update import paths from `'./types.js'` to `'./types.js'` (same
relative — no change needed since tests are co-located). Verify all tests
pass.

- [ ] **Step 9: Copy and adapt markdown.test.ts from connectors**

Source: `connectors/chat-demo/src/main/webui/src/qhorus/markdown.test.ts`

Copy and update import paths. Verify all tests pass.

- [ ] **Step 10: Install deps and run tests**

```bash
yarn install
yarn --cwd components/channel-activity test
```

Expected: all types and markdown tests pass.

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): package scaffold, types, and markdown (#39)"
```

---

### Task 2: Events Module

**Files:**
- Create: `components/channel-activity/src/events.ts`
- Modify: `components/channel-activity/src/index.ts`
- Test: `components/channel-activity/src/events.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage`, `MessageType`, `ArtefactRef` from Task 1
- Produces: `ChannelEventTopics`, `SendMessagePayload`, `ReactPayload`,
  `CreateChannelPayload`, `SelectChannelPayload`, `MessageSelectedPayload`,
  `DeleteChannelPayload`, `CursorActionPayload`

- [ ] **Step 1: Write failing test for event topics and emission**

Create `events.test.ts` — adapt from connectors' `events.test.ts` but:
- Replace `ChatEventTopics` → `ChannelEventTopics`
- Replace `chat:` topic prefix → `channel:`
- Replace `emitChatEvent` calls → `emitPagesEvent` from blocks-ui-core
- Drop unused topics (`SELECT_TOPIC`, `RESOLVE_TOPIC`)
- Add tests for new topics (`channel:cursor-catchup`, `channel:cursor-reload`)
- Add tests for new payload types (`CursorActionPayload`, `DeleteChannelPayload`)

- [ ] **Step 2: Run test to verify it fails**

```bash
yarn --cwd components/channel-activity test -- src/events.test.ts
```

Expected: FAIL — `events.ts` only has types.ts exports so far.

- [ ] **Step 3: Create events.ts**

```typescript
import type { QhorusMessage, MessageType, ArtefactRef } from './types.js';

export const ChannelEventTopics = {
  SEND_MESSAGE: 'channel:send-message',
  REACT: 'channel:react',
  UNREACT: 'channel:unreact',
  CREATE_CHANNEL: 'channel:create',
  DELETE_CHANNEL: 'channel:delete',
  SELECT_CHANNEL: 'channel:selected',
  MESSAGE_SELECTED: 'channel:message-selected',
  CURSOR_CATCHUP: 'channel:cursor-catchup',
  CURSOR_RELOAD: 'channel:cursor-reload',
} as const;

export interface SendMessagePayload {
  readonly channelId: string;
  readonly content: string;
  readonly topic?: string;
  readonly inReplyTo?: string;
  readonly speechAct?: MessageType;
  readonly artefactRefs?: readonly ArtefactRef[];
}

export interface ReactPayload {
  readonly messageId: string;
  readonly emoji: string;
}

export interface CreateChannelPayload {
  readonly name: string;
  readonly description?: string;
  readonly spaceId?: string;
  readonly semantic?: string;
}

export interface DeleteChannelPayload {
  readonly channelId: string;
}

export interface SelectChannelPayload {
  readonly channelId: string;
}

export interface MessageSelectedPayload {
  readonly message: QhorusMessage;
}

export interface CursorActionPayload {
  readonly channelId: string;
  readonly cursorId?: string;
}
```

- [ ] **Step 4: Update index.ts to export events**

Add: `export * from './events.js';`

- [ ] **Step 5: Run test to verify it passes**

```bash
yarn --cwd components/channel-activity test -- src/events.test.ts
```

Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add components/channel-activity/src/events.ts components/channel-activity/src/events.test.ts components/channel-activity/src/index.ts
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): events module with channel: topic prefix (#39)"
```

---

### Task 3: Message Primitive (`<channel-message>`)

**Files:**
- Create: `components/channel-activity/src/channel-message.ts`
- Modify: `components/channel-activity/src/index.ts`
- Test: `components/channel-activity/src/channel-message.test.ts`

**Interfaces:**
- Consumes: `QhorusMessage`, `CommitmentState`, `ActorType`, `MessageType`,
  `isTerminalMessageType`, `isObligationCreating`, `messageTypeCategory`,
  `commitmentStateCategory` from Task 1; `ChannelEventTopics`,
  `MessageSelectedPayload` from Task 2
- Produces: `<channel-message>` custom element with properties: `message`,
  `reactions`, `showSpeechAct`, `showActorBadge`, `commitmentState`,
  `parentMessage`, `channelName`, `formatSender`

- [ ] **Step 1: Write failing test**

Adapt from connectors' `qhorus-message.test.ts`:
- Replace tag name `qhorus-message` → `channel-message`
- Replace imports from connectors paths → `./channel-message.js`, `./types.js`
- Add test for `formatSender` callback: verify it's called with `(sender, actorType)` and the returned string is rendered
- Add test for terminal message dimming (class `ch-msg-terminal` applied when `isTerminalMessageType(msg.messageType)` is true)
- Verify `emitPagesEvent` is used (not `emitChatEvent`)

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Copy and adapt qhorus-message.ts**

Source: `connectors/chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message.ts`

Changes:
1. Rename class: `QhorusMessageElement` → `ChannelMessageElement`
2. Rename `customElements.define('qhorus-message', ...)` → `customElements.define('channel-message', ...)`
3. Replace `import { emitChatEvent, ChatEventTopics }` → `import { emitPagesEvent } from '@casehubio/blocks-ui-core'; import { ChannelEventTopics } from './events.js';`
4. Replace all `emitChatEvent(this, ChatEventTopics.X, payload)` → `emitPagesEvent(this, ChannelEventTopics.X, payload)`
5. Add `formatSender` property:
   ```typescript
   @property({ attribute: false })
   formatSender: (sender: string, actorType: ActorType) => string = (s) => s;
   ```
6. In render, replace `m.sender` display with `this.formatSender(m.sender, m.actorType)`
7. Update internal imports: `'../types.js'` → `'./types.js'`, `'../events.js'` → `'./events.js'`, `'../markdown.js'` → `'./markdown.js'`

- [ ] **Step 4: Update index.ts**

Add: `export { ChannelMessageElement } from './channel-message.js';`

- [ ] **Step 5: Run test to verify it passes**

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): channel-message primitive with formatSender (#39)"
```

---

### Task 4: Message Input Primitive (`<channel-input>`)

**Files:**
- Create: `components/channel-activity/src/channel-input.ts`
- Test: `components/channel-activity/src/channel-input.test.ts`

**Interfaces:**
- Consumes: `MessageType`, `MESSAGE_TYPES` from Task 1; `ChannelEventTopics`,
  `SendMessagePayload` from Task 2
- Produces: `<channel-input>` custom element with properties: `channelId`,
  `replyTo`, `showTypeSelector`, `messageTypes`, `allowedTypes`, `deniedTypes`,
  `renderError`

This task adds the extension points that close Gaps #1 and #4 from the spec.

- [ ] **Step 1: Write failing test**

Adapt from connectors' `qhorus-message-input.test.ts`:
- Replace tag name → `channel-input`
- Replace imports
- Add new tests:
  - **Type selector hidden by default** — verify no `<select>` rendered when `showTypeSelector` is false
  - **Type selector shown** — set `showTypeSelector: true`, verify `<select>` renders with all 9 types
  - **allowedTypes filtering** — set `allowedTypes: ['QUERY', 'COMMAND']`, verify only those options appear
  - **deniedTypes filtering** — set `deniedTypes: ['EVENT']`, verify EVENT is excluded
  - **deniedTypes precedence** — set `allowedTypes: ['QUERY', 'EVENT']` and `deniedTypes: ['EVENT']`, verify only QUERY appears
  - **speechAct included in payload** — verify `SendMessagePayload.speechAct` is set when type selector is visible
  - **renderError callback** — set `renderError`, trigger an error, verify custom template renders
  - **Error cleared on send** — verify error disappears after successful send event

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Copy and adapt qhorus-message-input.ts**

Source: `connectors/chat-demo/src/main/webui/src/qhorus/primitives/qhorus-message-input.ts`

Changes:
1. Rename class and tag: `channel-input`
2. Replace `emitChatEvent` → `emitPagesEvent`
3. Replace `ChatEventTopics` → `ChannelEventTopics`
4. Add new properties:
   ```typescript
   @property({ type: Boolean }) showTypeSelector = false;
   @property({ attribute: false }) messageTypes: MessageType[] = [...MESSAGE_TYPES];
   @property({ attribute: false }) allowedTypes?: MessageType[];
   @property({ attribute: false }) deniedTypes?: MessageType[];
   @property({ attribute: false }) renderError?: (error: string) => TemplateResult;
   ```
5. Add `@state() _error = '';`
6. Add type selector rendering (conditional on `showTypeSelector`):
   ```typescript
   private _computeAvailableTypes(): MessageType[] {
     let types = this.messageTypes;
     if (this.allowedTypes?.length) {
       types = types.filter(t => this.allowedTypes!.includes(t));
     }
     if (this.deniedTypes?.length) {
       types = types.filter(t => !this.deniedTypes!.includes(t));
     }
     return types;
   }
   ```
7. Render type `<select>` before the textarea when `showTypeSelector` is true
8. Include selected type as `speechAct` in `SendMessagePayload`
9. Render error slot: `this._error ? (this.renderError?.(this._error) ?? html\`<span class="ch-error">${this._error}</span>\`) : nothing`

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): channel-input with type selector and error callback (#39)"
```

---

### Task 5: Supporting Primitives (emoji-picker, reaction-bar, thread)

**Files:**
- Create: `components/channel-activity/src/channel-emoji-picker.ts`
- Create: `components/channel-activity/src/channel-reaction-bar.ts`
- Create: `components/channel-activity/src/channel-thread.ts`
- Test: `components/channel-activity/src/channel-emoji-picker.test.ts`
- Test: `components/channel-activity/src/channel-reaction-bar.test.ts`
- Test: `components/channel-activity/src/channel-thread.test.ts`

**Interfaces:**
- Consumes: types from Task 1, events from Task 2, `<channel-message>` from Task 3
- Produces: `<channel-emoji-picker>`, `<channel-reaction-bar>`, `<channel-thread>` custom elements

These are direct ports with rename only — no new extension points.

- [ ] **Step 1: Copy and adapt emoji-picker test + source**

Source tests: `connectors/.../qhorus-emoji-picker.test.ts`
Source component: `connectors/.../qhorus-emoji-picker.ts`

Changes for both: rename tag/class, update imports, replace `emitChatEvent` → `emitPagesEvent`, `ChatEventTopics` → `ChannelEventTopics`, flatten relative paths (`../types.js` → `./types.js`).

- [ ] **Step 2: Copy and adapt reaction-bar test + source**

Same rename pattern. The reaction-bar emits `channel:react` and `channel:unreact` events.
Add `currentActorId` property (per spec §Component Data Interface).

- [ ] **Step 3: Copy and adapt thread test + source**

Same rename pattern. Thread renders `<channel-message>` children (not `<qhorus-message>`).

- [ ] **Step 4: Update index.ts with all three exports**

- [ ] **Step 5: Run all tests**

```bash
yarn --cwd components/channel-activity test
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): emoji-picker, reaction-bar, thread primitives (#39)"
```

---

### Task 6: Channel Feed Composite (`<channel-feed>`)

**Files:**
- Create: `components/channel-activity/src/channel-feed.ts`
- Test: `components/channel-activity/src/channel-feed.test.ts`

**Interfaces:**
- Consumes: types from Task 1, events from Task 2, `<channel-message>` from Task 3, `<channel-thread>` from Task 5
- Produces: `<channel-feed>` custom element with properties: `messages`,
  `reactions`, `commitments`, `channelId`, `channelName`, `terminalDimming`,
  `eventStyling`, `autoScroll`, `staleCursorMinutes`, `renderContextHeader`

This task closes Gaps #2, #3, and #5 from the spec.

- [ ] **Step 1: Write failing test**

Adapt from connectors' `qhorus-channel-feed.test.ts`:
- Rename tag/imports
- Add tests for:
  - **renderContextHeader callback** — set callback, verify it renders above the feed
  - **renderContextHeader absent** — verify no header area when callback is undefined
  - **terminalDimming** — messages with terminal types get `opacity: 0.8` class; toggle off and verify no class
  - **eventStyling** — EVENT messages get italic + dimmed class; toggle off and verify
  - **autoScroll** — when feed is at bottom and new messages arrive, verify scrollTop advances; when scrolled up, verify no scroll change
  - **staleCursorMinutes** — set to 1, set a cursor timestamp older than 1 minute, select channel, verify stale prompt renders with "Catch up" and "Reload" buttons
  - **cursor-catchup event** — click "Catch up", verify `channel:cursor-catchup` event emitted with `{ channelId, cursorId }`
  - **cursor-reload event** — click "Reload", verify `channel:cursor-reload` event emitted with `{ channelId }`
  - **stale prompt clears on messages update** — after prompt shown, set new messages, verify prompt disappears

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Copy and adapt qhorus-channel-feed.ts**

Source: `connectors/.../qhorus-channel-feed.ts`

Changes:
1. Rename class/tag → `channel-feed`
2. Replace event imports
3. Add new properties:
   ```typescript
   @property({ type: Boolean }) terminalDimming = true;
   @property({ type: Boolean }) eventStyling = true;
   @property({ type: Boolean }) autoScroll = true;
   @property({ type: Number }) staleCursorMinutes = 30;
   @property({ attribute: false }) channelId = '';
   @property({ attribute: false }) renderContextHeader?: () => TemplateResult;
   ```
4. Add `@state() _showStalePrompt = false;`
5. Add `@state() _staleCursorId?: string;`
6. In render, add context header: `${this.renderContextHeader?.() ?? nothing}` before the feed div
7. In message rendering, add classes:
   - `.ch-msg-terminal { opacity: 0.8 }` when `terminalDimming && isTerminalMessageType(msg.messageType)`
   - `.ch-msg-event { opacity: 0.55; font-style: italic; }` when `eventStyling && msg.messageType === 'EVENT'`
8. In `updated()`, implement auto-scroll:
   ```typescript
   if (this.autoScroll && this.messages.length > this._prevMessageCount) {
     const feed = this.renderRoot.querySelector('.feed');
     if (feed) {
       const wasAtBottom = feed.scrollHeight - feed.scrollTop <= feed.clientHeight + 4;
       if (wasAtBottom) requestAnimationFrame(() => { feed.scrollTop = feed.scrollHeight; });
     }
   }
   this._prevMessageCount = this.messages.length;
   ```
9. Implement stale cursor detection:
   - On `channelId` change (via `willUpdate`), check sessionStorage for cursor
   - If cursor timestamp is older than `staleCursorMinutes * 60000`, show prompt
   - On `messages` property change, clear prompt
10. Render stale prompt when `_showStalePrompt`:
    ```html
    <div class="stale-prompt">
      <span>You were away for a while.</span>
      <button @click=${this._onCatchUp}>Catch up from where you left off</button>
      <button @click=${this._onReload}>Reload full history</button>
    </div>
    ```
11. Emit `channel:cursor-catchup` / `channel:cursor-reload` events

- [ ] **Step 4: Run test to verify it passes**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): channel-feed with context header, styling, auto-scroll, stale cursor (#39)"
```

---

### Task 7: Channel Nav and Member Panel Composites

**Files:**
- Create: `components/channel-activity/src/channel-nav.ts`
- Create: `components/channel-activity/src/channel-member-panel.ts`
- Test: `components/channel-activity/src/channel-nav.test.ts`
- Test: `components/channel-activity/src/channel-member-panel.test.ts`

**Interfaces:**
- Consumes: types from Task 1, events from Task 2
- Produces: `<channel-nav>`, `<channel-member-panel>` custom elements

Direct ports with rename — no new extension points.

- [ ] **Step 1: Copy and adapt channel-nav test + source**

Source: `connectors/.../qhorus-channel-nav.ts`, `...test.ts`

Changes: rename tag/class, replace events, flatten imports. Verify
keyboard navigation (ArrowUp, ArrowDown, Enter) tests transfer.

- [ ] **Step 2: Copy and adapt member-panel test + source**

Source: `connectors/.../qhorus-member-panel.ts`, `...test.ts`

Same rename pattern.

- [ ] **Step 3: Copy and adapt theme-variables.test.ts**

Source: `connectors/.../theme-variables.test.ts`

This test validates that all components use `--pages-*` CSS custom
properties consistently. Update component tag names in the test.

- [ ] **Step 4: Update index.ts with all exports**

Final index.ts should re-export everything:
```typescript
export * from './types.js';
export * from './events.js';
export { renderMarkdown } from './markdown.js';
export { ChannelMessageElement } from './channel-message.js';
export { ChannelInputElement } from './channel-input.js';
export { ChannelEmojiPickerElement } from './channel-emoji-picker.js';
export { ChannelReactionBarElement } from './channel-reaction-bar.js';
export { ChannelThreadElement } from './channel-thread.js';
export { ChannelFeedElement } from './channel-feed.js';
export { ChannelNavElement } from './channel-nav.js';
export { ChannelMemberPanelElement } from './channel-member-panel.js';
```

- [ ] **Step 5: Run full test suite**

```bash
yarn --cwd components/channel-activity test
```

Expected: all tests pass.

- [ ] **Step 6: Verify build**

```bash
yarn --cwd components/channel-activity build
yarn typecheck
```

Expected: clean build, no type errors.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "feat(channel-activity): channel-nav, member-panel, theme-variables, full exports (#39)"
```

---

### Task 8: Remove Stub and Clean Up

**Files:**
- Delete: `components/channel-activity/dist/index.d.ts` (old stub output)

**Interfaces:**
- Consumes: all prior tasks
- Produces: final clean package

- [ ] **Step 1: Delete old stub dist output**

The old stub's `dist/index.d.ts` (from the two-line placeholder) needs
removing — the build will regenerate it from the real source.

```bash
rm -f components/channel-activity/dist/index.d.ts
```

- [ ] **Step 2: Full rebuild and test**

```bash
yarn clean
yarn install
yarn build
yarn test
```

Expected: clean build, all tests pass across the entire monorepo.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui commit -m "chore(channel-activity): remove old stub, clean build (#39)"
```

---

## Post-Implementation Checklist

After all tasks complete:

- [ ] Update epic #39 description to reflect the design reversal (claudony → connectors as promotion base)
- [ ] File GitHub issue for chat-app module permanent home (deferred from spec)
- [ ] Verify `yarn build && yarn test && yarn typecheck` passes at repo root
- [ ] Verify all extension points are tested (formatSender, renderContextHeader, renderError, showTypeSelector, allowedTypes/deniedTypes filtering, terminalDimming, eventStyling, autoScroll, staleCursorMinutes)
