# Wire Pages Palettes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #140 — wire pages palette components and make all diagrams fully editable
**Issue group:** #140

**Goal:** Replace hand-built property forms and stencil palette with pages-property-palette and pages-diagram-palette. Implement EditPolicy for case and SWF diagrams. Remove old form utilities.

**Architecture:** DiagramBaseMixin bridges to pages-property-palette via PropertyPaletteSource getter and EditorResolver protected method. Stencil palette items derive from EditPolicy.getCreatableTypes(). Domain adapters handle YAML mutations. Virtual discriminators render via closure-captured render functions.

**Tech Stack:** TypeScript, Lit, pages-property-palette, pages-diagram-palette, graph-renderer EditPolicy SPI, vitest

## Global Constraints

- Stencil packages must not import from each other (PP-20260806-320d50)
- Component customisation via typed config + render callbacks (PP-20260713-8ea1af)
- Custom editors emit `change` events (not `value-changed`) per pages-property-palette contract
- `_editorResolver()` is a default method (returns `undefined`), NOT abstract — SWF has no custom editors
- `_editPolicy()` is a default method (returns `undefined`) — read-only diagrams have no policy
- YAML is the source of truth; GraphModel is derived, never reverse-serialised
- `subcase` is NOT a palette-creatable type — it's a binding target reference

---

## Batch 1: Dependencies + Property Palette Foundation

### Task 1: Publish pages SNAPSHOT and add palette dependencies

**Files:**
- Modify: `packages/diagram-core/package.json`
- Modify: `yarn.lock` (auto-generated)

**Interfaces:**
- Consumes: pages SNAPSHOT from `~/.m2/repository`
- Produces: `@casehubio/pages-property-palette` and `@casehubio/pages-diagram-palette` importable

- [ ] **Step 1: Build and install pages SNAPSHOT**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/pages build && mvn -f /Users/mdproctor/claude/casehub/pages/pom.xml install -DskipTests 2>&1 | tail -5`

- [ ] **Step 2: Add dependencies to diagram-core**

Add to `packages/diagram-core/package.json` dependencies:
```json
"@casehubio/pages-property-palette": "*",
"@casehubio/pages-diagram-palette": "*"
```

- [ ] **Step 3: Install and verify**

Run: `yarn --cwd /Users/mdproctor/claude/casehub/blocks-ui install`
Run: `ls node_modules/@casehubio/pages-property-palette node_modules/@casehubio/pages-diagram-palette`

- [ ] **Step 4: Commit**

### Task 2: Fix custom editor event contract

**Files:**
- Modify: `packages/diagram-core/src/editors/blocks-prompt-editor.ts`
- Modify: `packages/graph-stencil-case/src/editors/blocks-env-map-editor.ts`
- Modify: `packages/diagram-core/src/editors/editors.test.ts`
- Modify: `packages/graph-stencil-case/src/editors/editors.test.ts`

**Interfaces:**
- Consumes: existing editor components
- Produces: editors emit `change` event (not `value-changed`) matching pages-property-palette contract

- [ ] **Step 1: Write failing test**

```typescript
// editors.test.ts — add to blocks-prompt-editor describe block
it('emits change event on input', async () => {
  const el = new BlocksPromptEditorElement();
  document.body.appendChild(el);
  await el.updateComplete;
  const textarea = el.shadowRoot!.querySelector('textarea')!;
  const changePromise = new Promise<CustomEvent>(r => el.addEventListener('change', r as EventListener, { once: true }));
  textarea.value = 'test';
  textarea.dispatchEvent(new Event('input', { bubbles: true }));
  const event = await changePromise;
  expect(event.detail.value).toBe('test');
  el.remove();
});
```

- [ ] **Step 2: Update blocks-prompt-editor to emit `change` instead of `value-changed`**

Replace `new CustomEvent('value-changed', ...)` with `new CustomEvent('change', ...)`.

- [ ] **Step 3: Update blocks-env-map-editor to emit `change` instead of `value-changed`**

Same change.

- [ ] **Step 4: Run tests, commit**

### Task 3: PropertyPaletteSource adapter and _renderPropertyPanel in DiagramBaseMixin

**Files:**
- Modify: `packages/diagram-core/src/diagram-base-mixin.ts`
- Modify: `packages/diagram-core/src/index.ts`

**Interfaces:**
- Consumes: `PropertyPaletteSource` from `@casehubio/pages-property-palette`, `_selectedSchema`, `_selectedData`, `_applyPropertyEdit()`
- Produces: `_propertyPaletteSource` getter, `_onPropertyChange(field, value)`, `_editorResolver()`, `_renderPropertyPanel()`, `_editPolicy()`, `_addElement(type)`, `_paletteItems()`

- [ ] **Step 1: Write failing test for PropertyPaletteSource bridge**

```typescript
// diagram-base-mixin.test.ts or schema-registry.test.ts
import { registerPropertySchema, getPropertySchema } from './schema-registry.js';

describe('PropertyPaletteSource bridge', () => {
  it('_propertyPaletteSource returns undefined when no node selected', () => {
    // Test the getter via a mock subclass
  });
});
```

- [ ] **Step 2: Add imports and new methods to DiagramBaseMixin**

Add to `diagram-base-mixin.ts`:
- Import `PropertyPaletteSource`, `EditorResolver` from `@casehubio/pages-property-palette`
- Import `PaletteItem` from `@casehubio/pages-diagram-palette`
- Import `EditPolicy` from `@casehubio/graph-renderer`
- Add `_propertyPaletteSource` getter
- Add `_onPropertyChange(field, value)` method (wraps undo + `_applyPropertyEdit`)
- Add `_editorResolver()` default method returning `undefined`
- Add `_editPolicy()` default method returning `undefined`
- Add `_addElement(type: string)` abstract method
- Replace `_paletteTypes(): string[]` with `_paletteItems(): PaletteItem[]`
- Add `_handlePaletteSelect(item)` method
- Add `_renderPropertyPanel()` and `_renderStencilPalette()` template helpers

- [ ] **Step 3: Remove old abstract `_paletteTypes()`, add `_addElement()` abstract**

- [ ] **Step 4: Remove old `diagram-properties` imports from index.ts**

Remove exports: `DiagramProperties`, `renderPropertyForm`, `emitPropertyChange`, `fieldTypeFor`, `FieldType`, `validateField`, `FieldSchema`, `renderTriggerEditor`, `detectTriggerType`, `TriggerType`, `renderNestedGroup`.

- [ ] **Step 5: Run tests, commit**

## Batch 2: EditPolicy + Stencil Palette + YAML Mutations

### Task 4: CaseEditPolicy

**Files:**
- Create: `packages/graph-stencil-case/src/editing/case-edit-policy.ts`
- Create: `packages/graph-stencil-case/src/editing/case-edit-policy.test.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`

**Interfaces:**
- Consumes: `EditPolicy` from `@casehubio/graph-renderer`, `GraphModel`, `GraphNode`
- Produces: `CaseEditPolicy` implementing `EditPolicy` — `canConnect`, `getCreatableTypes` (4 types), `canDelete`, `getDeleteStrategy`

- [ ] **Step 1: Write failing tests**

```typescript
describe('CaseEditPolicy', () => {
  it('getCreatableTypes returns binding, worker, milestone, goal (not subcase)', () => { ... });
  it('canConnect returns true for binding → worker', () => { ... });
  it('canConnect returns false for worker → worker', () => { ... });
  it('canDelete returns false for subcase nodes', () => { ... });
  it('getDeleteStrategy returns auto-join for binding', () => { ... });
});
```

- [ ] **Step 2: Implement CaseEditPolicy**
- [ ] **Step 3: Export from index, run tests, commit**

### Task 5: SwfEditPolicy + addSwfTask

**Files:**
- Create: `packages/graph-stencil-swf/src/editing/swf-edit-policy.ts`
- Create: `packages/graph-stencil-swf/src/editing/swf-edit-policy.test.ts`
- Modify: `packages/graph-stencil-swf/src/adapter/swf-property-edit.ts` (add `addSwfTask`)
- Modify: `packages/graph-stencil-swf/src/index.ts`

**Interfaces:**
- Consumes: `EditPolicy` from `@casehubio/graph-renderer`
- Produces: `SwfEditPolicy`, `addSwfTask(yaml, taskType): string`

- [ ] **Step 1: Write failing tests for SwfEditPolicy and addSwfTask**
- [ ] **Step 2: Implement SwfEditPolicy**
- [ ] **Step 3: Implement addSwfTask with type-specific defaults**
- [ ] **Step 4: Export from index, run tests, commit**

### Task 6: switchTriggerType + migrate detectTriggerType

**Files:**
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.ts` (add `switchTriggerType`)
- Modify: `packages/graph-stencil-case/src/worker-function/detect.ts` (add `detectTriggerType`, `TriggerType`)
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`
- Modify: `packages/graph-stencil-case/src/index.ts`

**Interfaces:**
- Consumes: `yaml` library `parseDocument`
- Produces: `switchTriggerType(yaml, bindingPath, newType): string`, `detectTriggerType(data): TriggerType`

- [ ] **Step 1: Write failing test for switchTriggerType**

```typescript
it('switches from contextChange to cloudEvent, preserving other binding fields', () => {
  const yaml = `...binding with contextChange...`;
  const result = switchTriggerType(yaml, ['spec', 'bindings', 0], 'cloudEvent');
  expect(result).toContain('cloudEvent:');
  expect(result).not.toContain('contextChange:');
});
```

- [ ] **Step 2: Implement switchTriggerType**
- [ ] **Step 3: Migrate detectTriggerType from diagram-core trigger-editor.ts to graph-stencil-case detect.ts**
- [ ] **Step 4: Run tests, commit**

## Batch 3: Wire Diagrams + Remove Old Code

### Task 7: Wire casehub-diagram

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram.test.ts` (if exists)

**Interfaces:**
- Consumes: `CaseEditPolicy`, `PropertyPaletteSource`, `EditorResolver`, `PaletteItem`, `_renderPropertyPanel()`, `_renderStencilPalette()`, `_addElement()`, discriminator detection functions
- Produces: fully wired casehub-diagram with pages-property-palette, pages-diagram-palette, CaseEditPolicy, EditorResolver with discriminator rendering

- [ ] **Step 1: Override `_editPolicy()` to return CaseEditPolicy**
- [ ] **Step 2: Override `_editorResolver()` with x-editor-component and x-discriminator support**
- [ ] **Step 3: Implement `_renderDiscriminator`, `_renderBranchPalette`, `_renderTargetSelector`, `_renderTriggerDiscriminator`, `_renderProviderDiscriminator`, `_renderTransportDiscriminator`**
- [ ] **Step 4: Override `_addElement(type)` delegating to `addElement()`**
- [ ] **Step 5: Remove old casehub-diagram-palette usage, casehub-diagram-properties usage, prompt dialog, old event handlers**
- [ ] **Step 6: Run tests, commit**

### Task 8: Wire swf-diagram

**Files:**
- Modify: `components/swf-diagram/src/swf-diagram.ts`

**Interfaces:**
- Consumes: `SwfEditPolicy`, `addSwfTask`, `_renderPropertyPanel()`, `_renderStencilPalette()`
- Produces: swf-diagram with pages-property-palette (no custom editors), pages-diagram-palette, SwfEditPolicy

- [ ] **Step 1: Override `_editPolicy()` to return SwfEditPolicy**
- [ ] **Step 2: Override `_addElement(type)` delegating to `addSwfTask()`**
- [ ] **Step 3: Add palette and property panel to render**
- [ ] **Step 4: Run tests, commit**

### Task 9: Remove old code

**Files:**
- Delete: `packages/diagram-core/src/form/field-renderer.ts`
- Delete: `packages/diagram-core/src/form/validation.ts`
- Delete: `packages/diagram-core/src/form/trigger-editor.ts`
- Delete: `packages/diagram-core/src/form/nested-group.ts`
- Delete: `packages/diagram-core/src/form/property-form.ts`
- Delete: `packages/diagram-core/src/diagram-properties.ts`
- Delete: `components/casehub-diagram/src/casehub-diagram-palette.ts`
- Delete: `components/casehub-diagram/src/casehub-diagram-palette.test.ts`
- Delete: `components/casehub-diagram/src/casehub-diagram-properties.ts`
- Delete: `components/casehub-diagram/src/casehub-diagram-properties.test.ts`
- Modify: `packages/diagram-core/src/index.ts` (remove exports)
- Delete: `packages/graph-stencil-case/src/worker-function/forms/` directory (sub-form renderers)

**Interfaces:**
- Consumes: nothing (all consumers already migrated in Tasks 7-8)
- Produces: clean codebase with no dead code

- [ ] **Step 1: Delete form/ directory files**
- [ ] **Step 2: Delete diagram-properties.ts**
- [ ] **Step 3: Delete casehub-diagram-palette.ts and its test**
- [ ] **Step 4: Delete casehub-diagram-properties.ts and its test**
- [ ] **Step 5: Delete worker-function/forms/ directory**
- [ ] **Step 6: Clean up index.ts exports**
- [ ] **Step 7: Run full test suite across all 4 packages, fix any broken imports**
- [ ] **Step 8: Commit**

## Batch 4: Showcase Gallery

### Task 10: Update showcase pages

**Files:**
- Modify: `examples/src/pages/casehub-diagram-page.ts`
- Modify: `examples/src/pages/swf-diagram-page.ts`
- Modify: `examples/src/pages/diagram-workbench-page.ts`

**Interfaces:**
- Consumes: `<casehub-diagram>` with palette and property panel, `<swf-diagram>` with palette
- Produces: showcase pages demonstrating property palette on node selection, stencil palette with click-to-add

- [ ] **Step 1: Update casehub-diagram-page — add property palette sidebar visible on selection**
- [ ] **Step 2: Update swf-diagram-page — add stencil palette and property panel**
- [ ] **Step 3: Update diagram-workbench-page — both palettes in split layout**
- [ ] **Step 4: Run dev server, verify pages render correctly**
- [ ] **Step 5: Commit**

## References

- specs/issue-140-wire-pages-palettes/2026-08-27-wire-pages-palettes-design.md — design spec
- packages/diagram-core/src/diagram-base-mixin.ts — DiagramBaseMixin (abstract methods, _updateSelectedNode, _handlePropertyChange)
- packages/diagram-core/src/form/ — old form utilities (to be removed)
- packages/diagram-core/src/diagram-properties.ts — old property panel (to be removed)
- components/casehub-diagram/src/casehub-diagram.ts — main case diagram component
- components/casehub-diagram/src/casehub-diagram-palette.ts — old palette (to be removed)
- components/casehub-diagram/src/casehub-diagram-properties.ts — old properties (to be removed)
- components/swf-diagram/src/swf-diagram.ts — SWF diagram component
- packages/graph-stencil-case/src/adapter/yaml-editor.ts — addElement, switchFunctionType, switchBindingTarget
- packages/graph-stencil-case/src/worker-function/types.ts — WorkerFunctionType, FUNCTION_TYPE_TO_YAML_KEY
- packages/graph-stencil-case/src/worker-function/detect.ts — detectFunctionType
- packages/graph-stencil-swf/src/adapter/swf-property-edit.ts — applySwfPropertyEdit
- pages/packages/pages-property-palette/src/types.ts — PropertyPaletteSource, EditorResolver
- pages/packages/pages-diagram-palette/src/types.ts — PaletteItem, PaletteSelectDetail
- pages/packages/graph-renderer/src/editing/types.ts — EditPolicy, GraphEdit
- PP-20260806-320d50 — stencil package isolation
- PP-20260713-8ea1af — component customisation pattern
- casehubio/blocks-ui#136 — property schemas
- casehubio/blocks-ui#140
