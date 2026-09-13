# blocks-ui prefix rename and cleanup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> executing-plans to implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural editing.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #94 — chore: rename all components to use blocks- prefix consistently
**Issue group:** #94, #93, #88

**Goal:** Fix the pages-modal duplicate registration crash, migrate channel-activity native HTML elements to pages-ui-components, and rename all 59 unprefixed component tags to use `blocks-` prefix.

**Architecture:** Three independent changes executed in order of increasing blast radius. #88 narrows 16 barrel imports to sub-path imports. #93 replaces native HTML elements with pages-ui-components in channel-activity (8 files). #94 adds `blocks-` prefix to all 59 custom element tag names and updates every HTML template reference, test, and example.

**Tech Stack:** Lit 3 (Web Components), TypeScript, Vitest, Yarn workspaces

## Global Constraints

- Pre-release: breaking changes are free — no backward compat shims
- npm package names (`@casehubio/blocks-ui-*`) do NOT change — already correctly prefixed
- TypeScript class names do NOT change — only the HTML tag names
- Import paths do NOT change — they reference npm packages, not tag names
- Example page tags (`*-page`) do NOT change — they're app-level, not library exports
- Test fixture elements (`test-*`) do NOT change
- Use `ide_replace_text_in_file` for all source file modifications
- Verify with `yarn test` after each task

---

### Task 1: #88 — Switch pages-primitives imports to sub-path

**Files:**
- Modify: `components/approval-gate/src/approval-gate.ts`
- Modify: `components/audit-trail-viewer/src/audit-trail-viewer.ts`
- Modify: `components/blocks-timeline/src/blocks-timeline.ts`
- Modify: `components/case-explorer/src/case-explorer.ts`
- Modify: `components/case-explorer/src/entity-command-bar.ts`
- Modify: `components/case-explorer/src/entity-detail.ts`
- Modify: `components/case-explorer/src/entity-list.ts`
- Modify: `components/case-explorer/src/entity-tree.ts`
- Modify: `components/detail-pane/src/detail-pane.ts`
- Modify: `components/list-pane/src/list-pane.ts`
- Modify: `components/notification-inbox/src/notification-bell.ts`
- Modify: `components/notification-inbox/src/notification-inbox.ts`
- Modify: `components/routing-rationale/src/routing-rationale.ts`
- Modify: `components/split-workbench/src/split-workbench.ts`
- Modify: `components/trust-score-panel/src/trust-score-panel.ts`
- Modify: `components/trust-workbench/src/trust-workbench.ts`
- Modify: `components/work-item-detail/src/work-item-detail.ts`
- Modify: `components/work-item-workbench/src/work-item-workbench.ts`

**Interfaces:**
- Consumes: `@casehubio/pages-primitives/a11y` sub-path export (LiveRegionMixin, FocusTrapMixin, KeyboardShortcutMixin)
- Produces: No interface change — same mixins, narrower import path

- [ ] **Step 1: Run existing tests to establish baseline**

Run: `yarn test`
Expected: All tests pass (establishes green baseline)

- [ ] **Step 2: Replace all 16 barrel imports with sub-path imports**

For each file listed above, use `ide_replace_text_in_file`:

```
searchText:  from '@casehubio/pages-primitives'
replaceText: from '@casehubio/pages-primitives/a11y'
```

The 16 files and their specific imports:

| File | Imports |
|------|---------|
| approval-gate.ts | FocusTrapMixin, LiveRegionMixin |
| audit-trail-viewer.ts | LiveRegionMixin |
| blocks-timeline.ts | LiveRegionMixin |
| case-explorer.ts | LiveRegionMixin |
| entity-command-bar.ts | LiveRegionMixin |
| entity-detail.ts | LiveRegionMixin |
| entity-list.ts | LiveRegionMixin |
| entity-tree.ts | LiveRegionMixin |
| detail-pane.ts | LiveRegionMixin |
| list-pane.ts | LiveRegionMixin |
| notification-bell.ts | FocusTrapMixin, KeyboardShortcutMixin |
| notification-inbox.ts | KeyboardShortcutMixin, LiveRegionMixin |
| routing-rationale.ts | LiveRegionMixin |
| split-workbench.ts | LiveRegionMixin |
| trust-score-panel.ts | LiveRegionMixin |
| trust-workbench.ts | LiveRegionMixin |
| work-item-detail.ts | FocusTrapMixin, LiveRegionMixin |
| work-item-workbench.ts | KeyboardShortcutMixin |

All 18 files (16 unique components, some have multiple files per component) use the same replacement pattern. The import specifiers (`LiveRegionMixin`, etc.) stay the same — only the module path changes.

- [ ] **Step 3: Verify typecheck passes**

Run: `yarn typecheck`
Expected: Clean — pages-primitives/a11y exports the same symbols as the barrel

- [ ] **Step 4: Run tests**

Run: `yarn test`
Expected: All tests pass — behaviour unchanged

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add -A
git commit -m "fix(#88): narrow pages-primitives imports to /a11y sub-path

Eliminates transitive pages-modal registration from barrel import.
All 16 components import only a11y mixins — sub-path avoids pulling
in modal side-effect that causes duplicate CustomElementRegistry crash
in aliased bundler setups.

Closes #88"
```

---

### Task 2: #93 — Migrate channel-activity native HTML to pages-ui-components

**Files:**
- Modify: `components/channel-activity/package.json` (add dependency)
- Modify: `components/channel-activity/src/channel-artifact-panel.ts` (3 buttons)
- Modify: `components/channel-activity/src/channel-feed.ts` (2 buttons)
- Modify: `components/channel-activity/src/channel-input.ts` (2 buttons + select + textarea)
- Modify: `components/channel-activity/src/channel-message.ts` (2 buttons)
- Modify: `components/channel-activity/src/channel-nav.ts` (3 buttons + 2 hardcoded colors)
- Modify: `components/channel-activity/src/channel-reaction-bar.ts` (2 buttons)
- Modify: `components/channel-activity/src/channel-thread.ts` (1 button)
- Modify: `components/channel-activity/src/channel-topic-bar.ts` (4 buttons)

**Interfaces:**
- Consumes: `@casehubio/pages-ui-components` — `<pages-button>`, `<pages-select>`, `<pages-textarea>`
- Produces: No interface change — same component behaviour, delegated rendering

- [ ] **Step 1: Add pages-ui-components dependency**

Add to `components/channel-activity/package.json` dependencies:
```json
"@casehubio/pages-ui-components": "file:../../../pages/packages/pages-ui-components"
```

Then add the side-effect import to each file that uses pages-ui-components elements:
```typescript
import '@casehubio/pages-ui-components';
```

- [ ] **Step 2: Run existing tests to establish baseline**

Run: `yarn --cwd components/channel-activity test`
Expected: All tests pass

- [ ] **Step 3: Migrate channel-artifact-panel.ts (3 buttons)**

Read the file first. Replace each `<button ...>` with `<pages-button ...>` and `</button>` with `</pages-button>`. Preserve all attributes and event handlers. Use `ide_replace_text_in_file` for each replacement.

The button styling currently uses inline CSS — pages-button handles styling via tokens, so remove inline `background`, `color`, `border`, `padding`, `cursor` styles that pages-button provides by default. Keep layout styles (`margin`, `position`).

- [ ] **Step 4: Migrate channel-feed.ts (2 buttons)**

Same pattern: `<button>` → `<pages-button>`, remove redundant inline styles.

- [ ] **Step 5: Migrate channel-input.ts (2 buttons + select + textarea)**

Replace:
- `<button>` → `<pages-button>`
- `<select>` → `<pages-select>` (verify pages-select supports same attributes)
- `<textarea>` → `<pages-textarea>` (verify pages-textarea supports same attributes)

Read pages-ui-components source first to understand the element APIs.

- [ ] **Step 6: Migrate channel-message.ts (2 buttons)**

Same button pattern.

- [ ] **Step 7: Migrate channel-nav.ts (3 buttons + fix hardcoded colors)**

Replace 3 buttons. Additionally fix:
- `color: #fff` → `color: var(--pages-neutral-1, #fff)`
- `box-shadow: 0 4px 12px rgba(0,0,0,0.1)` → `box-shadow: var(--pages-shadow-3, 0 4px 12px rgba(0,0,0,0.1))`

- [ ] **Step 8: Migrate remaining files**

- channel-reaction-bar.ts: 2 buttons
- channel-thread.ts: 1 button
- channel-topic-bar.ts: 4 buttons

Same pattern for all.

- [ ] **Step 9: Run tests**

Run: `yarn --cwd components/channel-activity test`
Expected: All tests pass — behaviour unchanged

- [ ] **Step 10: Run full typecheck**

Run: `yarn typecheck`
Expected: Clean

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add -A
git commit -m "feat(#93): migrate channel-activity native HTML to pages-ui-components

Replace 19 <button>, 1 <select>, 1 <textarea> with pages-ui-components
equivalents. Fix 2 hardcoded colors in channel-nav to use --pages-* tokens.

Closes #93"
```

---

### Task 3: #94 — Rename all 59 components to blocks- prefix

**Complete tag name mapping (59 renames):**

| Old tag | New tag | Source file |
|---------|---------|-------------|
| approval-gate | blocks-approval-gate | components/approval-gate/src/approval-gate.ts |
| audit-trail-viewer | blocks-audit-trail-viewer | components/audit-trail-viewer/src/audit-trail-viewer.ts |
| case-definition-browser | blocks-case-definition-browser | components/case-explorer/src/convenience/case-definition-browser.ts |
| case-detail-panel | blocks-case-detail-panel | components/case-explorer/src/convenience/case-detail-panel.ts |
| case-explorer | blocks-case-explorer | components/case-explorer/src/case-explorer.ts |
| case-instance-list | blocks-case-instance-list | components/case-explorer/src/convenience/case-instance-list.ts |
| channel-artifact-panel | blocks-channel-artifact-panel | components/channel-activity/src/channel-artifact-panel.ts |
| channel-correlation-panel | blocks-channel-correlation-panel | components/channel-activity/src/channel-correlation-panel.ts |
| channel-emoji-picker | blocks-channel-emoji-picker | components/channel-activity/src/channel-emoji-picker.ts |
| channel-feed | blocks-channel-feed | components/channel-activity/src/channel-feed.ts |
| channel-input | blocks-channel-input | components/channel-activity/src/channel-input.ts |
| channel-member-panel | blocks-channel-member-panel | components/channel-activity/src/channel-member-panel.ts |
| channel-message | blocks-channel-message | components/channel-activity/src/channel-message.ts |
| channel-nav | blocks-channel-nav | components/channel-activity/src/channel-nav.ts |
| channel-preferences | blocks-channel-preferences | components/notification-inbox/src/channel-preferences.ts |
| channel-reaction-bar | blocks-channel-reaction-bar | components/channel-activity/src/channel-reaction-bar.ts |
| channel-task-panel | blocks-channel-task-panel | components/channel-activity/src/channel-task-panel.ts |
| channel-thread | blocks-channel-thread | components/channel-activity/src/channel-thread.ts |
| channel-topic-bar | blocks-channel-topic-bar | components/channel-activity/src/channel-topic-bar.ts |
| compliance-summary | blocks-compliance-summary | components/compliance-summary/src/compliance-summary.ts |
| detail-action-bar | blocks-detail-action-bar | components/work-item-detail/src/detail-action-bar.ts |
| detail-activity-tab | blocks-detail-activity-tab | components/work-item-detail/src/detail-activity-tab.ts |
| detail-pane | blocks-detail-pane | components/detail-pane/src/detail-pane.ts |
| detail-relations-tab | blocks-detail-relations-tab | components/work-item-detail/src/detail-relations-tab.ts |
| entity-command-bar | blocks-entity-command-bar | components/case-explorer/src/entity-command-bar.ts |
| entity-detail | blocks-entity-detail | components/case-explorer/src/entity-detail.ts |
| entity-list | blocks-entity-list | components/case-explorer/src/entity-list.ts |
| entity-tree | blocks-entity-tree | components/case-explorer/src/entity-tree.ts |
| gdpr-erasure-action | blocks-gdpr-erasure-action | components/gdpr-erasure-action/src/gdpr-erasure-action.ts |
| grouped-data-view | blocks-grouped-data-view | components/grouped-data-view/src/grouped-data-view.ts |
| inbox-filter-bar | blocks-inbox-filter-bar | components/work-item-inbox/src/inbox-filter-bar.ts |
| inbox-summary-bar | blocks-inbox-summary-bar | components/work-item-inbox/src/inbox-summary-bar.ts |
| kpi-metric-row | blocks-kpi-metric-row | components/kpi-metric-row/src/kpi-metric-row.ts |
| list-pane | blocks-list-pane | components/list-pane/src/list-pane.ts |
| mute-list | blocks-mute-list | components/notification-inbox/src/mute-list.ts |
| notification-bell | blocks-notification-bell | components/notification-inbox/src/notification-bell.ts |
| notification-inbox | blocks-notification-inbox | components/notification-inbox/src/notification-inbox.ts |
| notification-preferences | blocks-notification-preferences | components/notification-inbox/src/notification-preferences.ts |
| preferences-editor | blocks-preferences-editor | components/preferences-editor/src/preferences-editor.ts |
| queue-pill-bar | blocks-queue-pill-bar | components/work-item-inbox/src/queue-pill-bar.ts |
| routing-rationale | blocks-routing-rationale | components/routing-rationale/src/routing-rationale.ts |
| scope-context-bar | blocks-scope-context-bar | components/work-item-inbox/src/scope-context-bar.ts |
| similarity-panel | blocks-similarity-panel | components/similarity-panel/src/similarity-panel.ts |
| sla-breach-policy | blocks-sla-breach-policy | components/sla-breach-policy/src/sla-breach-policy.ts |
| sla-indicator | blocks-sla-indicator | components/sla-indicator/src/sla-indicator.ts |
| snooze-control | blocks-snooze-control | components/notification-inbox/src/snooze-control.ts |
| split-workbench | blocks-split-workbench | components/split-workbench/src/split-workbench.ts |
| subscription-editor | blocks-subscription-editor | components/notification-inbox/src/subscription-editor.ts |
| subscription-list | blocks-subscription-list | components/notification-inbox/src/subscription-list.ts |
| trust-feedback-display | blocks-trust-feedback-display | components/trust-feedback-display/src/trust-feedback-display.ts |
| trust-score-panel | blocks-trust-score-panel | components/trust-score-panel/src/trust-score-panel.ts |
| trust-workbench | blocks-trust-workbench | components/trust-workbench/src/trust-workbench.ts |
| value-editor | blocks-value-editor | components/preferences-editor/src/value-editor.ts |
| work-item-detail | blocks-work-item-detail | components/work-item-detail/src/work-item-detail.ts |
| work-item-inbox | blocks-work-item-inbox | components/work-item-inbox/src/work-item-inbox.ts |
| work-item-row | blocks-work-item-row | components/work-item-row/src/work-item-row.ts |
| work-item-workbench | blocks-work-item-workbench | components/work-item-workbench/src/work-item-workbench.ts |
| worker-detail-panel | blocks-worker-detail-panel | components/case-explorer/src/convenience/worker-detail-panel.ts |
| worker-list | blocks-worker-list | components/case-explorer/src/convenience/worker-list.ts |

**Already prefixed (no change):** blocks-timeline, blocks-confirm-dialog

**Replacement algorithm per tag name:**

For each `OLD` → `blocks-OLD` in the mapping:

1. **Decorator** in source file: `ide_replace_text_in_file` with `searchText: "@customElement('OLD')"`, `replaceText: "@customElement('blocks-OLD')"`

2. **HTML opening tags** across all .ts files: `ide_search_text` for `<OLD` in `*.ts` files, then for each hit: `ide_replace_text_in_file` with `searchText: "<OLD"`, `replaceText: "<blocks-OLD"` — but ONLY if the match is an HTML tag (inside a template literal), not an import path. Safe because `<OLD` only appears in HTML contexts — npm package names use `@casehubio/blocks-ui-OLD`, never `<OLD`.

3. **HTML closing tags**: `ide_replace_text_in_file` with `searchText: "</OLD>"`, `replaceText: "</blocks-OLD>"`

4. **Tag name strings in tests**: `querySelector('OLD')` → `querySelector('blocks-OLD')`, `createElement('OLD')` → `createElement('blocks-OLD')`

**What NOT to replace:**
- `@casehubio/blocks-ui-OLD` (npm package name) — contains `-OLD` not `<OLD`, safe
- `blocks-ui-OLD` (directory name) — same, no `<` prefix
- Already-prefixed tags (blocks-timeline, blocks-confirm-dialog) — not in the mapping

- [ ] **Step 1: Run existing tests to establish baseline**

Run: `yarn test`
Expected: All tests pass

- [ ] **Step 2: Rename all 59 decorators**

For each entry in the mapping table, run `ide_replace_text_in_file` on the source file:
```
file: <source file from table>
searchText: @customElement('<old tag>')
replaceText: @customElement('blocks-<old tag>')
```

Process all 59 files. Batch by package for efficiency (all files in a package directory in sequence).

- [ ] **Step 3: Update all internal HTML template cross-references**

For each old tag name, search across all `.ts` files for HTML usage:
```
ide_search_text(query: "<OLD", filePattern: "*.ts")
```

For each file found, apply two replacements:
```
ide_replace_text_in_file(file, searchText: "<OLD", replaceText: "<blocks-OLD")
ide_replace_text_in_file(file, searchText: "</OLD>", replaceText: "</blocks-OLD>")
```

Known internal cross-references (components that render other components in their templates):
- approval-gate renders `<sla-indicator>`
- work-item-inbox renders `<inbox-filter-bar>`, `<inbox-summary-bar>`, `<queue-pill-bar>`, `<scope-context-bar>`
- work-item-detail renders `<detail-action-bar>`, `<detail-activity-tab>`, `<detail-relations-tab>`
- work-item-workbench renders `<work-item-inbox>`, `<work-item-detail>`, `<split-workbench>`
- trust-workbench renders `<trust-score-panel>`, `<list-pane>`, `<routing-rationale>`, `<trust-feedback-display>`, `<split-workbench>`
- case-explorer renders `<entity-list>`, `<entity-detail>`, `<entity-tree>`, `<entity-command-bar>`, `<split-workbench>`, `<detail-pane>`
- notification-inbox renders `<notification-bell>`, `<subscription-list>`, `<subscription-editor>`, `<channel-preferences>`, `<mute-list>`, `<snooze-control>`, `<notification-preferences>`
- channel-activity components render sibling channel-* components
- sla-breach-policy renders `<sla-indicator>`
- grouped-data-view renders `<list-pane>`

- [ ] **Step 4: Update all test files**

Search test files for old tag names in string contexts:
```
ide_search_text(query: "OLD", filePattern: "*.test.ts")
```

Replace in:
- `querySelector('OLD')` → `querySelector('blocks-OLD')`
- `createElement('OLD')` → `createElement('blocks-OLD')`
- Template literals containing `<OLD` and `</OLD>`
- Any fixture HTML strings

- [ ] **Step 5: Update all example pages**

Search example files for old tag names:
```
ide_search_text(query: "OLD", filePattern: "*.ts", in examples/src/)
```

Replace HTML template references. Example page tag names themselves (`approval-gate-page` etc.) do NOT change.

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: Clean

- [ ] **Step 7: Run full test suite**

Run: `yarn test`
Expected: All tests pass with new tag names

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/blocks-ui add -A
git commit -m "feat(#94): rename all components to blocks- prefix

59 custom element tags renamed from bare names to blocks-* prefix
for platform namespace consistency (pages- for pages, blocks- for
blocks-ui). All internal template references, tests, and examples
updated.

Closes #94"
```

---

## Post-implementation

- [ ] **Update CLAUDE.md** — Key Directories section: add `blocks-` prefix to component descriptions where tag names are mentioned
- [ ] **Run work-end** — code review, squash, push
