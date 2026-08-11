# Worker Function Drill-Down — Design Spec

**Issue:** casehubio/blocks-ui#108
**Branch:** issue-108-worker-function-drill-down
**Date:** 2026-08-11

## Problem

Workers in the CaseHub engine support 5 distinct function types (agent, flow/SWF,
A2A, MCP, sequence), each with its own configuration shape. The diagram currently
shows "Worker" as a single node type with no way to inspect or configure the
underlying function implementation. The property panel only shows core worker
fields (name, capabilities, description) because function-specific keys fall
under `additionalProperties: true` in the Worker schema — invisible to the
schema-driven property form.

The flow (`do:`) function type already has a thumbnail and drill-down to the SWF
diagram. No other function type has any visual presence or editing UI.

## Engine Worker Function Types

The engine's `WorkerFunctionProviderRegistry` treats function types as an open
set via the `WorkerFunctionProvider` SPI. Each provider detects its YAML key and
constructs the function. The UI handles the 5 known types with dedicated
renderers and falls back to raw JSON display for unrecognised keys.

| Type | YAML key | Detection | Configuration |
|------|----------|-----------|---------------|
| Agent | `agent:` | `data.agent` present | systemPrompt, inputProjection, outputProjection, userMessageTemplate, model (oneOf 5 providers) |
| Flow | `do:` | `data.do` present | Embedded SWF steps (existing thumbnail + drill-down) |
| A2A | `a2a:` | `data.a2a` present | endpoint, skill?, streaming?, auth? |
| MCP | `mcp:` | `data.mcp` present | stdio (command, env?) or HTTP (url, auth?) |
| Sequence | `sequence:` | `data.sequence` present | Ordered list of worker name references |
| External | none | No function key | No config — external worker |
| Unknown | unrecognised key | No known key match, but non-core keys present | Raw JSON fallback |

### Agent Model Providers

The `model` field is a discriminated union — exactly one provider key is present:

| Provider | Key | Fields |
|----------|-----|--------|
| OpenAI | `openai:` | modelName, apiKey?, temperature?, maxTokens?, topP? |
| Anthropic | `anthropic:` | modelName, apiKey?, temperature?, maxTokens?, topP? |
| Ollama | `ollama:` | modelName, temperature?, maxTokens?, topP? |
| Mistral AI | `mistralAi:` | modelName, apiKey?, temperature?, maxTokens?, topP? |
| Google Gemini | `googleAiGemini:` | modelName, apiKey?, temperature?, maxTokens?, topP? |

### Auth Config (shared by MCP-HTTP and A2A)

```yaml
auth:
  type: none | bearer | api-key
  tokenConfigKey: <config key for the secret>
```

### MCP Transport Variants

**Stdio:**
```yaml
mcp:
  command: ["/path/to/server"]
  env:
    KEY: value
```

**HTTP:**
```yaml
mcp:
  url: https://example.com/mcp
  auth:
    type: bearer
    tokenConfigKey: mcp.token
```

## Design

### Package Ownership

All new code lives in `graph-stencil-case` under `src/worker-function/`. This
package already owns worker rendering, YAML editing, and generated types. Function
types are consumed exclusively by the case diagram's worker stencil and property
panel.

### 1. Type Detection and TypeScript Types

**`src/worker-function/types.ts`**

TypeScript interfaces mirroring the engine's function type shapes. These are
handwritten because the engine's `additionalProperties: true` schema doesn't
generate them.

```typescript
export type WorkerFunctionType =
  | 'agent' | 'flow' | 'a2a' | 'mcp' | 'sequence' | 'external';

export interface AgentConfig {
  systemPrompt: string;
  inputProjection: string;
  outputProjection: string;
  userMessageTemplate?: string;
  model: AgentModel;
}

export type AgentModel =
  | { openai: ProviderModelConfig }
  | { anthropic: ProviderModelConfig }
  | { ollama: ProviderModelConfig }
  | { mistralAi: ProviderModelConfig }
  | { googleAiGemini: ProviderModelConfig };

export type ModelProviderKey =
  | 'openai' | 'anthropic' | 'ollama' | 'mistralAi' | 'googleAiGemini';

export const MODEL_PROVIDERS: readonly ModelProviderKey[] =
  ['openai', 'anthropic', 'ollama', 'mistralAi', 'googleAiGemini'];

export interface ProviderModelConfig {
  modelName: string;
  apiKey?: string;
  temperature?: number;
  maxTokens?: number;
  topP?: number;
}

export interface A2AConfig {
  endpoint: string;
  skill?: string;
  streaming?: boolean;
  auth?: AuthConfig;
}

export type McpConfig = McpStdioConfig | McpHttpConfig;

export interface McpStdioConfig {
  command: string[];
  env?: Record<string, string>;
}

export interface McpHttpConfig {
  url: string;
  auth?: AuthConfig;
}

export type McpTransportType = 'stdio' | 'http';

export interface AuthConfig {
  type: 'none' | 'bearer' | 'api-key';
  tokenConfigKey?: string;
}

export const FUNCTION_TYPE_KEYS = ['agent', 'do', 'a2a', 'mcp', 'sequence'] as const;

export const CORE_WORKER_KEYS = new Set([
  'name', 'description', 'capabilities', 'executionPolicy',
  'contextType', 'outputType',
]);
```

**`src/worker-function/detect.ts`**

```typescript
export function detectFunctionType(
  data: Record<string, unknown>,
): WorkerFunctionType {
  if (data['agent'] != null) return 'agent';
  if (data['do'] != null) return 'flow';
  if (data['a2a'] != null) return 'a2a';
  if (data['mcp'] != null) return 'mcp';
  if (data['sequence'] != null) return 'sequence';
  return 'external';
}

export function hasUnknownFunctionKey(
  data: Record<string, unknown>,
): boolean {
  return Object.keys(data).some(
    k => !CORE_WORKER_KEYS.has(k) && !FUNCTION_TYPE_KEYS.includes(k),
  );
}

export function detectMcpTransport(
  mcp: Record<string, unknown>,
): McpTransportType {
  return mcp['command'] != null ? 'stdio' : 'http';
}

export function detectModelProvider(
  model: Record<string, unknown>,
): ModelProviderKey | null {
  for (const key of MODEL_PROVIDERS) {
    if (model[key] != null) return key;
  }
  return null;
}
```

### 2. Worker Stencil Badge

In `src/stencils/worker.ts`, add a coloured pill after the worker name showing
the detected function type. The pill is a plain inline-styled `<span>` — not the
StatusBadge component, which is for dynamic runtime state. Function type labels
are static type indicators.

```
┌──────────────────────────────────┐
│ worker-name          [agent]     │  ← coloured pill
│ capability1, capability2         │
│ description text...              │
│ ┌────────────────────────┐       │  ← SWF thumbnail (flow only, existing)
│ │  (thumbnail)           │       │
│ └────────────────────────┘       │
└──────────────────────────────────┘
```

Badge colours (CSS custom properties with fallbacks):

| Type | Label | Colour |
|------|-------|--------|
| agent | agent | purple |
| flow | flow | blue |
| a2a | a2a | teal |
| mcp | mcp | orange |
| sequence | seq | grey |
| external | ext | light grey |

### 3. Property Panel — Function Type Section

The `casehub-diagram-properties` component gains a function type section below
the existing `renderPropertyForm()` output:

```
┌─ Properties ─────────────────────┐
│ worker-name                      │  ← panel header
│                                  │
│ [Core fields via renderPropertyForm]
│ Name: [________]                 │
│ Capabilities: [________]         │
│ Description: [________]          │
│                                  │
│ ── Function ─────────────────    │  ← new section
│ Type: [agent ▾]                  │  ← dropdown selector
│                                  │
│ [Type-specific sub-form]         │
│ System Prompt: [____] [⤢]       │  ← pop-out button
│ Input Projection: [________]     │
│ Output Projection: [________]    │
│ Provider: [openai ▾]            │
│   Model Name: [________]        │
│   Temperature: [0.7]            │
│   Max Tokens: [1024]            │
│                                  │
└──────────────────────────────────┘
```

#### Function Type Selector

Dropdown at the top of the function section. Options: Agent, Flow, A2A, MCP,
Sequence, External (none). Changing the type calls `switchFunctionType()` in
the YAML editor, which removes the old function key and inserts the new one
with type-appropriate defaults. The undo stack captures the full YAML string
before the switch, so the operation is atomically undoable.

The dropdown also shows "Unknown" (disabled) when unrecognised function keys
are present — prevents switching away from a config the UI can't reproduce.

#### Type-Specific Sub-Forms

Each function type gets a dedicated render function in
`src/worker-function/forms/`:

**`render-agent-form.ts`** — `renderAgentForm(data, readonly, onChange)`
- systemPrompt: textarea with pop-out button (see §5)
- inputProjection: text input
- outputProjection: text input
- userMessageTemplate: textarea (optional)
- Provider selector: dropdown of 5 providers
- Provider fields: modelName (text), apiKey (text, optional), temperature
  (number, 0-2), maxTokens (number), topP (number, 0-1, optional)
- Switching provider removes old provider key, creates new with defaults

**`render-a2a-form.ts`** — `renderA2AForm(data, readonly, onChange)`
- endpoint: text input (required)
- skill: text input (optional)
- streaming: checkbox
- auth: shared auth sub-form (see below)

**`render-mcp-form.ts`** — `renderMcpForm(data, readonly, onChange)`
- Transport selector: radio buttons (stdio / HTTP)
- Stdio fields: command (string-array, one arg per line), env (textarea, KEY=VALUE per line, parsed to/from Record<string,string>)
- HTTP fields: url (text input), auth (shared auth sub-form)
- Switching transport clears the old transport fields

**`render-sequence-form.ts`** — `renderSequenceForm(data, readonly, onChange, workerNames)`
- Ordered list of worker name references
- Add button: dropdown of available worker names (filtered to exclude self
  and already-listed names)
- Remove button per item
- Drag handles for reordering
- `workerNames` parameter: list of all worker names in the current case
  definition, passed from the diagram component

**`render-unknown-form.ts`** — `renderUnknownForm(data)`
- Read-only JSON display of all non-core keys
- Warning text: "Unrecognised function configuration"

**`render-auth-config.ts`** — `renderAuthConfig(auth, onChange)`
- Auth type: dropdown (none / bearer / api-key)
- tokenConfigKey: text input (shown when type is not 'none')
- Used by both MCP-HTTP and A2A forms

#### onChange Contract

All sub-form onChange callbacks emit `(field: (string | number)[], value)` where
`field` is the path relative to the function key. For example, editing the
agent's systemPrompt emits `onChange(['agent', 'systemPrompt'], 'new value')`.
The properties component prepends the worker's YAML path and calls
`applyPropertyEdit()`.

### 4. YAML Editor Extension

In `src/adapter/yaml-editor.ts`, add:

```typescript
export function switchFunctionType(
  yaml: string,
  nodePath: readonly (string | number)[],
  newType: WorkerFunctionType,
): string
```

Removes any existing function key (`agent`, `do`, `a2a`, `mcp`, `sequence`)
from the worker node and inserts the new key with type-appropriate defaults:

| Type | Default value |
|------|---------------|
| agent | `{ systemPrompt: '', inputProjection: '.', outputProjection: '.', model: { openai: { modelName: '' } } }` |
| a2a | `{ endpoint: '' }` |
| mcp | `{ command: [] }` (stdio default) |
| sequence | `[]` |
| flow | `[]` (empty do block) |
| external | (remove all function keys, no new key) |

Also add:

```typescript
export function switchMcpTransport(
  yaml: string,
  nodePath: readonly (string | number)[],
  newTransport: McpTransportType,
): string
```

Removes existing MCP transport fields and inserts the new transport's defaults.

```typescript
export function switchModelProvider(
  yaml: string,
  nodePath: readonly (string | number)[],
  newProvider: ModelProviderKey,
): string
```

Removes the existing provider key and inserts the new one with
`{ modelName: '' }` as default.

### 5. Pop-Out Prompt Editor

A modal overlay for editing the agent's systemPrompt. Not a new component —
rendered inline in the properties panel when the pop-out button is clicked.

- Full-width overlay anchored to the diagram container (covers canvas + properties panel)
- Larger textarea (monospace, ~20 rows)
- Save and Cancel buttons
- Save writes the value via onChange and closes the overlay
- Cancel discards changes and closes
- Escape key cancels

### 6. Exports

`graph-stencil-case/src/index.ts` exports:
- `detectFunctionType`, `hasUnknownFunctionKey`, `detectMcpTransport`, `detectModelProvider`
- All type interfaces (`WorkerFunctionType`, `AgentConfig`, etc.)
- `switchFunctionType`, `switchMcpTransport`, `switchModelProvider`
- Form render functions (`renderAgentForm`, `renderA2AForm`, `renderMcpForm`,
  `renderSequenceForm`, `renderUnknownForm`, `renderAuthConfig`)

The `casehub-diagram-properties` component in `components/casehub-diagram/`
imports these and wires them into the property panel.

### 7. File Structure

```
packages/graph-stencil-case/src/
  worker-function/
    types.ts                    — TypeScript interfaces
    detect.ts                   — function type detection
    defaults.ts                 — default values per type
    forms/
      render-agent-form.ts      — agent config sub-form
      render-a2a-form.ts        — A2A config sub-form
      render-mcp-form.ts        — MCP config sub-form (stdio/HTTP)
      render-sequence-form.ts   — sequence editor with drag-reorder
      render-unknown-form.ts    — raw JSON fallback
      render-auth-config.ts     — shared auth config sub-form
      index.ts                  — re-exports
  adapter/
    yaml-editor.ts              — + switchFunctionType, switchMcpTransport, switchModelProvider

components/casehub-diagram/src/
  casehub-diagram-properties.ts — + function type section
```

## Testing

- **Detection:** Unit tests for `detectFunctionType` covering all types,
  multi-key, no-key, and unknown-key cases
- **YAML editing:** Unit tests for `switchFunctionType`, `switchMcpTransport`,
  `switchModelProvider` — verify key removal, default insertion, and
  preservation of other worker properties
- **Forms:** Unit tests for each render function — verify correct fields render,
  onChange emits correct paths, readonly disables inputs
- **Integration:** Test the full flow in casehub-diagram: select worker →
  see badge and function section → switch type → verify YAML updates → undo

## Scope Boundaries

**In scope:**
- Function type badge on worker stencil
- Function type selector and sub-forms in property panel
- YAML editor extensions for type switching
- Pop-out prompt editor for agent systemPrompt
- Fallback renderer for unknown function types

**Out of scope:**
- Flow drill-down improvements (existing, unchanged)
- Worker function discovery/runtime behaviour
- Schema generation for function types (handwritten types are sufficient)
- Property form generic discriminated union support (deferred to future work)
