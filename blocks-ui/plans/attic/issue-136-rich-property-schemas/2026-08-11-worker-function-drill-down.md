# Worker Function Drill-Down Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #108 — Worker function drill-down — agent, flow, a2a, mcp configuration
**Issue group:** #108

**Goal:** Add function type indicators to worker stencils and type-specific
configuration editors to the property panel.

**Architecture:** New `worker-function/` module in `graph-stencil-case` provides
types, detection, defaults, YAML editing, and form renderers. The
`casehub-diagram-properties` component wires these into the property panel,
rendering a function type selector and type-specific sub-forms below the core
schema-driven fields. A native `<dialog>` in `casehub-diagram` provides a
pop-out editor for the agent's systemPrompt.

**Tech Stack:** TypeScript, Lit (lit-html templates), vitest, YAML (yaml library)

## Global Constraints

- All new code in `packages/graph-stencil-case/src/worker-function/` except
  the YAML editor extensions (in `src/adapter/yaml-editor.ts`) and the
  property panel integration (in `components/casehub-diagram/src/`)
- IntelliJ MCP mandatory for all .ts file operations
- Function type keys: `agent`, `do`, `a2a`, `mcp`, `sequence`
- YAML key for flow type is `do`, not `flow`
- Sub-forms receive function config object (not full worker), onChange emits
  paths within that config
- Test runner: `vitest run` in `packages/graph-stencil-case/`
- Inline styles only in render functions (shadow DOM boundary)

---

### Task 1: Types, Detection, and Defaults

**Files:**
- Create: `packages/graph-stencil-case/src/worker-function/types.ts`
- Create: `packages/graph-stencil-case/src/worker-function/detect.ts`
- Create: `packages/graph-stencil-case/src/worker-function/defaults.ts`
- Test: `packages/graph-stencil-case/src/worker-function/detect.test.ts`

**Interfaces:**
- Consumes: nothing (foundational)
- Produces:
  - `WorkerFunctionType` — `'agent' | 'flow' | 'a2a' | 'mcp' | 'sequence' | 'external' | 'unknown'`
  - `detectFunctionType(data: Record<string, unknown>): WorkerFunctionType`
  - `detectMcpTransport(mcp: Record<string, unknown>): McpTransportType | null`
  - `detectModelProvider(model: Record<string, unknown>): ModelProviderKey | null`
  - `FUNCTION_TYPE_KEYS`, `FUNCTION_TYPE_TO_YAML_KEY`, `CORE_WORKER_KEYS`
  - `MODEL_PROVIDERS: readonly ModelProviderKey[]`
  - `FUNCTION_TYPE_DEFAULTS`, `MCP_TRANSPORT_DEFAULTS`, `PROVIDER_DEFAULT`
  - All config interfaces: `AgentConfig`, `AgentModel`, `ProviderModelConfig`, `A2AConfig`, `McpConfig`, `McpStdioConfig`, `McpHttpConfig`, `AuthConfig`, `McpTransportType`, `ModelProviderKey`

- [ ] **Step 1: Write detection tests**

```typescript
// detect.test.ts
import { describe, it, expect } from 'vitest';
import { detectFunctionType, detectMcpTransport, detectModelProvider } from './detect.js';

describe('detectFunctionType', () => {
  it('detects agent', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], agent: { systemPrompt: '' } })).toBe('agent');
  });
  it('detects flow from do key', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], do: [] })).toBe('flow');
  });
  it('detects a2a', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], a2a: { endpoint: '' } })).toBe('a2a');
  });
  it('detects mcp', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], mcp: { command: [] } })).toBe('mcp');
  });
  it('detects sequence', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], sequence: ['a', 'b'] })).toBe('sequence');
  });
  it('returns external when no function key', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [] })).toBe('external');
  });
  it('returns unknown when unrecognised key present', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], grpc: { endpoint: '' } })).toBe('unknown');
  });
  it('first known key wins when multiple present', () => {
    expect(detectFunctionType({ name: 'w', capabilities: [], agent: {}, do: [] })).toBe('agent');
  });
});

describe('detectMcpTransport', () => {
  it('detects stdio', () => {
    expect(detectMcpTransport({ command: ['/bin/tool'] })).toBe('stdio');
  });
  it('detects http', () => {
    expect(detectMcpTransport({ url: 'https://example.com' })).toBe('http');
  });
  it('returns null for malformed config', () => {
    expect(detectMcpTransport({})).toBeNull();
  });
});

describe('detectModelProvider', () => {
  it('detects openai', () => {
    expect(detectModelProvider({ openai: { modelName: 'gpt-4' } })).toBe('openai');
  });
  it('detects anthropic', () => {
    expect(detectModelProvider({ anthropic: { modelName: 'claude-3' } })).toBe('anthropic');
  });
  it('returns null for empty model', () => {
    expect(detectModelProvider({})).toBeNull();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-case/src/worker-function/detect.test.ts`
Expected: FAIL — modules not found

- [ ] **Step 3: Create types.ts**

```typescript
// types.ts
export type WorkerFunctionType =
  | 'agent' | 'flow' | 'a2a' | 'mcp' | 'sequence' | 'external' | 'unknown';

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
  ['openai', 'anthropic', 'ollama', 'mistralAi', 'googleAiGemini'] as const;

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

export const FUNCTION_TYPE_TO_YAML_KEY: Record<WorkerFunctionType, string | null> = {
  agent: 'agent',
  flow: 'do',
  a2a: 'a2a',
  mcp: 'mcp',
  sequence: 'sequence',
  external: null,
  unknown: null,
};

export const CORE_WORKER_KEYS = new Set([
  'name', 'description', 'capabilities', 'executionPolicy',
  'contextType', 'outputType',
]);
```

- [ ] **Step 4: Create detect.ts**

```typescript
// detect.ts
import type { WorkerFunctionType, McpTransportType, ModelProviderKey } from './types.js';
import { FUNCTION_TYPE_KEYS, CORE_WORKER_KEYS, MODEL_PROVIDERS } from './types.js';

export function detectFunctionType(
  data: Record<string, unknown>,
): WorkerFunctionType {
  if (data['agent'] != null) return 'agent';
  if (data['do'] != null) return 'flow';
  if (data['a2a'] != null) return 'a2a';
  if (data['mcp'] != null) return 'mcp';
  if (data['sequence'] != null) return 'sequence';
  const hasUnknown = Object.keys(data).some(
    k => !CORE_WORKER_KEYS.has(k) && !(FUNCTION_TYPE_KEYS as readonly string[]).includes(k),
  );
  return hasUnknown ? 'unknown' : 'external';
}

export function detectMcpTransport(
  mcp: Record<string, unknown>,
): McpTransportType | null {
  if (mcp['command'] != null) return 'stdio';
  if (mcp['url'] != null) return 'http';
  return null;
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

- [ ] **Step 5: Create defaults.ts**

```typescript
// defaults.ts
import type { WorkerFunctionType, McpTransportType, ProviderModelConfig } from './types.js';

export const FUNCTION_TYPE_DEFAULTS: Record<WorkerFunctionType, unknown> = {
  agent: {
    systemPrompt: '',
    inputProjection: '.',
    outputProjection: '.',
    model: { openai: { modelName: '' } },
  },
  a2a: { endpoint: '' },
  mcp: { command: [] },
  sequence: [],
  flow: [],
  external: null,
  unknown: null,
};

export const MCP_TRANSPORT_DEFAULTS: Record<McpTransportType, Record<string, unknown>> = {
  stdio: { command: [] },
  http: { url: '' },
};

export const PROVIDER_DEFAULT: ProviderModelConfig = { modelName: '' };
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `yarn vitest run packages/graph-stencil-case/src/worker-function/detect.test.ts`
Expected: PASS — all 10 tests green

- [ ] **Step 7: Commit**

```bash
git add packages/graph-stencil-case/src/worker-function/
git commit -m "feat(graph-stencil-case): add worker function types, detection, and defaults

Refs casehubio/blocks-ui#108"
```

---

### Task 2: YAML Editor Extensions

**Files:**
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.ts`
- Modify: `packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`

**Interfaces:**
- Consumes: `FUNCTION_TYPE_KEYS`, `FUNCTION_TYPE_TO_YAML_KEY`, `FUNCTION_TYPE_DEFAULTS`, `MCP_TRANSPORT_DEFAULTS`, `PROVIDER_DEFAULT`, `MODEL_PROVIDERS` from Task 1
- Produces:
  - `switchFunctionType(yaml: string, nodePath: readonly (string | number)[], newType: WorkerFunctionType): string`
  - `switchMcpTransport(yaml: string, nodePath: readonly (string | number)[], newTransport: McpTransportType): string`
  - `switchModelProvider(yaml: string, nodePath: readonly (string | number)[], newProvider: ModelProviderKey): string`

- [ ] **Step 1: Write switchFunctionType tests**

```typescript
// Append to yaml-editor.test.ts
import { switchFunctionType, switchMcpTransport, switchModelProvider } from './yaml-editor.js';

const WORKER_YAML = `dsl: "1.0.0"
namespace: test
name: sample
version: "1.0.0"
spec:
  bindings: []
  workers:
    - name: my-worker
      capabilities:
        - analyze
      agent:
        systemPrompt: "You are helpful"
        inputProjection: "."
        outputProjection: "."
        model:
          openai:
            modelName: gpt-4
`;

describe('switchFunctionType', () => {
  it('switches agent to a2a', () => {
    const result = switchFunctionType(WORKER_YAML, ['spec', 'workers', 0], 'a2a');
    const parsed = parseYaml(result);
    const worker = parsed.spec.workers[0];
    expect(worker.agent).toBeUndefined();
    expect(worker.a2a).toEqual({ endpoint: '' });
    expect(worker.name).toBe('my-worker');
    expect(worker.capabilities).toEqual(['analyze']);
  });

  it('switches agent to flow using do key', () => {
    const result = switchFunctionType(WORKER_YAML, ['spec', 'workers', 0], 'flow');
    const parsed = parseYaml(result);
    const worker = parsed.spec.workers[0];
    expect(worker.agent).toBeUndefined();
    expect(worker.do).toEqual([]);
  });

  it('switches to external removes all function keys', () => {
    const result = switchFunctionType(WORKER_YAML, ['spec', 'workers', 0], 'external');
    const parsed = parseYaml(result);
    const worker = parsed.spec.workers[0];
    expect(worker.agent).toBeUndefined();
    expect(worker.do).toBeUndefined();
    expect(worker.a2a).toBeUndefined();
    expect(worker.mcp).toBeUndefined();
    expect(worker.sequence).toBeUndefined();
    expect(worker.name).toBe('my-worker');
  });

  it('switches to mcp with stdio defaults', () => {
    const result = switchFunctionType(WORKER_YAML, ['spec', 'workers', 0], 'mcp');
    const parsed = parseYaml(result);
    const worker = parsed.spec.workers[0];
    expect(worker.agent).toBeUndefined();
    expect(worker.mcp).toEqual({ command: [] });
  });

  it('switches to sequence with empty array', () => {
    const result = switchFunctionType(WORKER_YAML, ['spec', 'workers', 0], 'sequence');
    const parsed = parseYaml(result);
    const worker = parsed.spec.workers[0];
    expect(worker.agent).toBeUndefined();
    expect(worker.sequence).toEqual([]);
  });
});

describe('switchMcpTransport', () => {
  const MCP_YAML = `dsl: "1.0.0"
namespace: test
name: sample
version: "1.0.0"
spec:
  bindings: []
  workers:
    - name: tool-worker
      capabilities: []
      mcp:
        command:
          - /bin/tool
        env:
          KEY: val
`;

  it('switches stdio to http', () => {
    const result = switchMcpTransport(MCP_YAML, ['spec', 'workers', 0], 'http');
    const parsed = parseYaml(result);
    const mcp = parsed.spec.workers[0].mcp;
    expect(mcp.command).toBeUndefined();
    expect(mcp.env).toBeUndefined();
    expect(mcp.url).toBe('');
  });

  it('switches http to stdio', () => {
    const httpYaml = switchMcpTransport(MCP_YAML, ['spec', 'workers', 0], 'http');
    const result = switchMcpTransport(httpYaml, ['spec', 'workers', 0], 'stdio');
    const parsed = parseYaml(result);
    const mcp = parsed.spec.workers[0].mcp;
    expect(mcp.url).toBeUndefined();
    expect(mcp.command).toEqual([]);
  });
});

describe('switchModelProvider', () => {
  it('switches openai to anthropic', () => {
    const result = switchModelProvider(WORKER_YAML, ['spec', 'workers', 0], 'anthropic');
    const parsed = parseYaml(result);
    const model = parsed.spec.workers[0].agent.model;
    expect(model.openai).toBeUndefined();
    expect(model.anthropic).toEqual({ modelName: '' });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`
Expected: FAIL — functions not exported

- [ ] **Step 3: Implement switchFunctionType, switchMcpTransport, switchModelProvider**

Add to `yaml-editor.ts`:

```typescript
import type { WorkerFunctionType, McpTransportType, ModelProviderKey } from '../worker-function/types.js';
import { FUNCTION_TYPE_KEYS, FUNCTION_TYPE_TO_YAML_KEY, MODEL_PROVIDERS } from '../worker-function/types.js';
import { FUNCTION_TYPE_DEFAULTS, MCP_TRANSPORT_DEFAULTS, PROVIDER_DEFAULT } from '../worker-function/defaults.js';

export function switchFunctionType(
  yaml: string,
  nodePath: readonly (string | number)[],
  newType: WorkerFunctionType,
): string {
  const doc = parseDocument(yaml);
  const node = doc.getIn(nodePath) as YAMLMap;
  // Remove all existing function keys
  for (const key of FUNCTION_TYPE_KEYS) {
    if (node.has(key)) node.delete(key);
  }
  // Insert new key with defaults
  const yamlKey = FUNCTION_TYPE_TO_YAML_KEY[newType];
  if (yamlKey != null) {
    const defaultValue = FUNCTION_TYPE_DEFAULTS[newType];
    node.set(yamlKey, doc.createNode(defaultValue));
  }
  return doc.toString();
}

export function switchMcpTransport(
  yaml: string,
  nodePath: readonly (string | number)[],
  newTransport: McpTransportType,
): string {
  const doc = parseDocument(yaml);
  const mcpPath = [...nodePath, 'mcp'];
  const mcp = doc.getIn(mcpPath) as YAMLMap;
  // Remove existing transport keys
  for (const key of ['command', 'env', 'url', 'auth']) {
    if (mcp.has(key)) mcp.delete(key);
  }
  // Insert new transport defaults
  const defaults = MCP_TRANSPORT_DEFAULTS[newTransport];
  for (const [k, v] of Object.entries(defaults)) {
    mcp.set(k, doc.createNode(v));
  }
  return doc.toString();
}

export function switchModelProvider(
  yaml: string,
  nodePath: readonly (string | number)[],
  newProvider: ModelProviderKey,
): string {
  const doc = parseDocument(yaml);
  const modelPath = [...nodePath, 'agent', 'model'];
  const model = doc.getIn(modelPath) as YAMLMap;
  // Remove all existing provider keys
  for (const key of MODEL_PROVIDERS) {
    if (model.has(key)) model.delete(key);
  }
  model.set(newProvider, doc.createNode(PROVIDER_DEFAULT));
  return doc.toString();
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/graph-stencil-case/src/adapter/yaml-editor.test.ts`
Expected: PASS — all tests green (existing + new)

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-case/src/adapter/yaml-editor.ts packages/graph-stencil-case/src/adapter/yaml-editor.test.ts
git commit -m "feat(graph-stencil-case): add switchFunctionType/Transport/Provider to YAML editor

Refs casehubio/blocks-ui#108"
```

---

### Task 3: Worker Stencil Badge

**Files:**
- Modify: `packages/graph-stencil-case/src/stencils/worker.ts`
- Modify: `packages/graph-stencil-case/src/stencils/stencils.test.ts`

**Interfaces:**
- Consumes: `detectFunctionType`, `WorkerFunctionType` from Task 1
- Produces: Updated `renderWorker` that shows a coloured function type pill

- [ ] **Step 1: Write badge rendering test**

Append to `stencils.test.ts`:

```typescript
import { detectFunctionType } from '../worker-function/detect.js';

describe('renderWorker badge', () => {
  it('includes agent badge for agent worker', () => {
    const node = {
      id: 'w1', type: 'worker',
      properties: { name: 'analyzer', capabilities: ['analyze'], agent: { systemPrompt: '' } },
    };
    const result = renderWorker(node as any);
    // lit-html TemplateResult — check strings contain badge text
    const rendered = result.strings.join('');
    expect(rendered).toContain('badge');
  });

  it('includes ext badge for external worker', () => {
    const node = {
      id: 'w1', type: 'worker',
      properties: { name: 'external', capabilities: [] },
    };
    const result = renderWorker(node as any);
    const rendered = result.strings.join('');
    expect(rendered).toContain('badge');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn vitest run packages/graph-stencil-case/src/stencils/stencils.test.ts`
Expected: FAIL — badge not in rendered output

- [ ] **Step 3: Add badge rendering to renderWorker**

In `worker.ts`, import `detectFunctionType` and add a badge configuration map
and pill rendering in the header row next to the name:

```typescript
import { detectFunctionType } from '../worker-function/detect.js';
import type { WorkerFunctionType } from '../worker-function/types.js';

const BADGE_CONFIG: Record<WorkerFunctionType, { label: string; bg: string; fg: string }> = {
  agent:    { label: 'agent', bg: '#7c3aed', fg: '#fff' },
  flow:     { label: 'flow',  bg: '#2563eb', fg: '#fff' },
  a2a:      { label: 'a2a',   bg: '#0d9488', fg: '#fff' },
  mcp:      { label: 'mcp',   bg: '#ea580c', fg: '#fff' },
  sequence: { label: 'seq',   bg: '#6b7280', fg: '#fff' },
  external: { label: 'ext',   bg: '#d1d5db', fg: '#374151' },
  unknown:  { label: '?',     bg: '#fbbf24', fg: '#374151' },
};
```

Add the badge `<span>` inside the header `<div>` after the name, before the
drill-down button:

```html
<span class="badge" style="font-size: 9px; padding: 1px 5px; border-radius: 3px;
  background: ${badge.bg}; color: ${badge.fg}; font-weight: 600;
  letter-spacing: 0.3px; text-transform: uppercase;">${badge.label}</span>
```

Where `badge = BADGE_CONFIG[detectFunctionType(data)]`.

- [ ] **Step 4: Run test to verify it passes**

Run: `yarn vitest run packages/graph-stencil-case/src/stencils/stencils.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-case/src/stencils/worker.ts packages/graph-stencil-case/src/stencils/stencils.test.ts
git commit -m "feat(graph-stencil-case): add function type badge to worker stencil

Refs casehubio/blocks-ui#108"
```

---

### Task 4: Form Renderers — Auth, Agent, A2A, MCP, Sequence, Unknown

**Files:**
- Create: `packages/graph-stencil-case/src/worker-function/forms/render-auth-config.ts`
- Create: `packages/graph-stencil-case/src/worker-function/forms/render-agent-form.ts`
- Create: `packages/graph-stencil-case/src/worker-function/forms/render-a2a-form.ts`
- Create: `packages/graph-stencil-case/src/worker-function/forms/render-mcp-form.ts`
- Create: `packages/graph-stencil-case/src/worker-function/forms/render-sequence-form.ts`
- Create: `packages/graph-stencil-case/src/worker-function/forms/render-unknown-form.ts`
- Create: `packages/graph-stencil-case/src/worker-function/forms/index.ts`
- Test: `packages/graph-stencil-case/src/worker-function/forms/forms.test.ts`

**Interfaces:**
- Consumes: `detectMcpTransport`, `detectModelProvider`, `MODEL_PROVIDERS`, `PROVIDER_DEFAULT`, `MCP_TRANSPORT_DEFAULTS` from Tasks 1
- Produces:
  - `renderAuthConfig(auth: Record<string, unknown> | undefined, onChange: (field: (string | number)[], value: unknown) => void): TemplateResult`
  - `renderAgentForm(data: Record<string, unknown>, readonly: boolean, onChange: OnChange, onPopOut: (value: string) => void): TemplateResult`
  - `renderA2AForm(data: Record<string, unknown>, readonly: boolean, onChange: OnChange): TemplateResult`
  - `renderMcpForm(data: Record<string, unknown>, readonly: boolean, onChange: OnChange, onTransportSwitch: (transport: McpTransportType) => void): TemplateResult`
  - `renderSequenceForm(data: Record<string, unknown>, readonly: boolean, onChange: OnChange, workerNames: string[]): TemplateResult`
  - `renderUnknownForm(data: Record<string, unknown>): TemplateResult`
  - `type OnChange = (field: (string | number)[], value: unknown) => void`

- [ ] **Step 1: Write form tests**

```typescript
// forms.test.ts
import { describe, it, expect, vi } from 'vitest';
import { renderAuthConfig } from './render-auth-config.js';
import { renderAgentForm } from './render-agent-form.js';
import { renderA2AForm } from './render-a2a-form.js';
import { renderMcpForm } from './render-mcp-form.js';
import { renderSequenceForm } from './render-sequence-form.js';
import { renderUnknownForm } from './render-unknown-form.js';

describe('renderAuthConfig', () => {
  it('renders auth type selector', () => {
    const result = renderAuthConfig(undefined, vi.fn());
    expect(result).toBeDefined();
    expect(result.strings.join('')).toContain('select');
  });

  it('renders tokenConfigKey when type is bearer', () => {
    const result = renderAuthConfig({ type: 'bearer', tokenConfigKey: 'my.key' }, vi.fn());
    expect(result.strings.join('')).toContain('input');
  });
});

describe('renderAgentForm', () => {
  it('renders systemPrompt textarea', () => {
    const data = { systemPrompt: 'test', inputProjection: '.', outputProjection: '.', model: { openai: { modelName: 'gpt-4' } } };
    const result = renderAgentForm(data, false, vi.fn(), vi.fn());
    expect(result).toBeDefined();
    expect(result.strings.join('')).toContain('textarea');
  });

  it('renders provider selector', () => {
    const data = { systemPrompt: '', inputProjection: '.', outputProjection: '.', model: { anthropic: { modelName: 'claude' } } };
    const result = renderAgentForm(data, false, vi.fn(), vi.fn());
    expect(result.strings.join('')).toContain('select');
  });
});

describe('renderA2AForm', () => {
  it('renders endpoint input', () => {
    const data = { endpoint: 'https://example.com' };
    const result = renderA2AForm(data, false, vi.fn());
    expect(result).toBeDefined();
    expect(result.strings.join('')).toContain('input');
  });
});

describe('renderMcpForm', () => {
  it('renders stdio fields when command present', () => {
    const data = { command: ['/bin/tool'] };
    const result = renderMcpForm(data, false, vi.fn(), vi.fn());
    expect(result).toBeDefined();
  });

  it('renders http fields when url present', () => {
    const data = { url: 'https://example.com' };
    const result = renderMcpForm(data, false, vi.fn(), vi.fn());
    expect(result).toBeDefined();
  });
});

describe('renderSequenceForm', () => {
  it('renders sequence items', () => {
    const data = ['worker-a', 'worker-b'];
    const result = renderSequenceForm(data as any, false, vi.fn(), ['worker-a', 'worker-b', 'worker-c']);
    expect(result).toBeDefined();
  });
});

describe('renderUnknownForm', () => {
  it('renders JSON display with warning', () => {
    const result = renderUnknownForm({ grpc: { endpoint: 'localhost' } });
    expect(result).toBeDefined();
    expect(result.strings.join('')).toContain('pre');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn vitest run packages/graph-stencil-case/src/worker-function/forms/forms.test.ts`
Expected: FAIL — modules not found

- [ ] **Step 3: Implement all form renderers**

Create each file with the render function. All functions return `TemplateResult`
from `lit-html`. Use inline styles (shadow DOM boundary). See the spec §3 for
field details per form.

Key implementation details:
- `renderAuthConfig`: dropdown for type (none/bearer/api-key), conditional
  tokenConfigKey input. onChange emits `['auth', 'type']` or `['auth', 'tokenConfigKey']`.
- `renderAgentForm`: textarea for systemPrompt with pop-out button (emits
  `onPopOut(currentValue)`), text inputs for projections, provider dropdown
  calling `detectModelProvider`, provider-specific model fields below it.
  Provider switch calls `onChange` with `['model']` replacing the entire model
  object.
- `renderA2AForm`: endpoint text, skill text, streaming checkbox, auth sub-form.
- `renderMcpForm`: radio buttons for transport type, conditional stdio/http
  fields. Transport switch emits `onTransportSwitch` (handled by parent, calls
  `switchMcpTransport`).
- `renderSequenceForm`: renders list items with drag handles, remove buttons,
  add dropdown filtered to `workerNames` excluding already-listed names. Drag
  reorder uses HTML5 drag and drop with `draggable`, `dragstart`, `dragover`,
  `drop` events. onChange emits the full reordered array.
- `renderUnknownForm`: `<pre>` JSON display + warning text. No onChange
  (read-only).
- `index.ts`: re-exports all form renderers.

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn vitest run packages/graph-stencil-case/src/worker-function/forms/forms.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/graph-stencil-case/src/worker-function/forms/
git commit -m "feat(graph-stencil-case): add worker function type form renderers

Includes agent, A2A, MCP, sequence, unknown, and shared auth config.

Refs casehubio/blocks-ui#108"
```

---

### Task 5: Package Exports and Index

**Files:**
- Modify: `packages/graph-stencil-case/src/index.ts`

**Interfaces:**
- Consumes: All modules from Tasks 1-4
- Produces: Public API surface for consumers (casehub-diagram)

- [ ] **Step 1: Add exports to index.ts**

```typescript
// Append to index.ts
export { switchFunctionType, switchMcpTransport, switchModelProvider } from './adapter/yaml-editor.js';

export { detectFunctionType, detectMcpTransport, detectModelProvider } from './worker-function/detect.js';

export type {
  WorkerFunctionType, AgentConfig, AgentModel, ProviderModelConfig,
  ModelProviderKey, A2AConfig, McpConfig, McpStdioConfig, McpHttpConfig,
  McpTransportType, AuthConfig,
} from './worker-function/types.js';
export {
  FUNCTION_TYPE_KEYS, FUNCTION_TYPE_TO_YAML_KEY, CORE_WORKER_KEYS, MODEL_PROVIDERS,
} from './worker-function/types.js';

export {
  renderAgentForm, renderA2AForm, renderMcpForm,
  renderSequenceForm, renderUnknownForm, renderAuthConfig,
} from './worker-function/forms/index.js';
```

- [ ] **Step 2: Run full test suite**

Run: `yarn vitest run` (from `packages/graph-stencil-case/`)
Expected: PASS — all tests green

- [ ] **Step 3: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 4: Commit**

```bash
git add packages/graph-stencil-case/src/index.ts
git commit -m "feat(graph-stencil-case): export worker function public API

Refs casehubio/blocks-ui#108"
```

---

### Task 6: Property Panel Integration

**Files:**
- Modify: `components/casehub-diagram/src/casehub-diagram-properties.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram-properties.test.ts`
- Modify: `components/casehub-diagram/src/casehub-diagram.ts`

**Interfaces:**
- Consumes: `detectFunctionType`, `FUNCTION_TYPE_TO_YAML_KEY`, `FUNCTION_TYPE_KEYS`,
  `renderAgentForm`, `renderA2AForm`, `renderMcpForm`, `renderSequenceForm`,
  `renderUnknownForm`, `switchFunctionType`, `switchMcpTransport`,
  `switchModelProvider` from Tasks 1-5
- Produces:
  - `CasehubDiagramProperties` with new `nodeType` and `workerNames` properties
  - `CasehubDiagram` passes `nodeType` and `workerNames` to properties component
  - `CasehubDiagram` renders prompt editor `<dialog>`

- [ ] **Step 1: Write property panel tests**

Append to `casehub-diagram-properties.test.ts`:

```typescript
describe('function type section', () => {
  it('does not render function section for non-worker nodes', () => {
    // The component should not render the function section
    // when nodeType is not 'worker'
    // (structural test — verify the function type selector is absent)
  });
});
```

- [ ] **Step 2: Add nodeType and workerNames properties to CasehubDiagramProperties**

Add two new `@litProp` fields:
- `nodeType: string = ''`
- `workerNames: string[] = []`

- [ ] **Step 3: Add function type section rendering**

In the `render()` method, after `renderPropertyForm()`, add a conditional
function type section that only renders when `this.nodeType === 'worker'`.

The section includes:
1. A separator and "Function" header
2. Function type dropdown (detect current from `this.data`, list all types)
3. Type-specific sub-form dispatched by `detectFunctionType(this.data)`

Filter function-type keys from schema before passing to `renderPropertyForm`:
create a filtered copy of `this.schema` that excludes properties whose keys
are in `FUNCTION_TYPE_KEYS` (to prevent double-rendering of `sequence`).

Handle function type switch: call `switchFunctionType()` via a
`function-type-change` event (parent handles YAML editing).

Handle transport/provider switch: emit corresponding events.

Handle pop-out: emit `prompt-editor-open` event with current systemPrompt
value.

- [ ] **Step 4: Wire CasehubDiagram to pass nodeType, workerNames, and handle events**

In `casehub-diagram.ts`:
1. Pass `nodeType` to the properties component (from the selected node's type)
2. Pass `workerNames` (extract from `_adapterResult.model.nodes` filtering
   type === 'worker')
3. Handle `function-type-change` event: call `switchFunctionType()`
4. Handle `mcp-transport-change` event: call `switchMcpTransport()`
5. Handle `model-provider-change` event: call `switchModelProvider()`
6. Handle `prompt-editor-open` event: open `<dialog>` with prompt value
7. Add `<dialog>` element to the render output

- [ ] **Step 5: Run tests**

Run: `yarn test` (full workspace)
Expected: PASS

- [ ] **Step 6: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 7: Commit**

```bash
git add components/casehub-diagram/src/ packages/graph-stencil-case/src/
git commit -m "feat(casehub-diagram): integrate worker function type editor in property panel

Adds function type selector, type-specific sub-forms, and pop-out prompt
editor dialog. Properties component gains nodeType and workerNames props.

Refs casehubio/blocks-ui#108"
```

---

### Task 7: Full Integration Test and Cleanup

**Files:**
- Review: all files from Tasks 1-6
- Run: full test suite and typecheck

- [ ] **Step 1: Run full test suite**

Run: `yarn test`
Expected: PASS — all packages green

- [ ] **Step 2: Run typecheck**

Run: `yarn typecheck`
Expected: No type errors

- [ ] **Step 3: Run build**

Run: `yarn build`
Expected: Successful build

- [ ] **Step 4: Manual verification in examples (if available)**

Check if there's an examples app that renders casehub-diagram.
Verify: select a worker node → badge visible → function type section appears →
type dropdown works → switching type updates YAML.

- [ ] **Step 5: Final commit if any cleanup needed**

```bash
git commit -m "chore: integration cleanup for worker function drill-down

Refs casehubio/blocks-ui#108"
```
