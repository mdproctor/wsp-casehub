# Pluggable Theme System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #230 — feat: pluggable theme system — pipeline architecture for pages-ui-tokens
**Issue group:** #230

**Goal:** Replace pages-ui-tokens' monolithic runtime theme generation with a build-time pipeline of composable transforms, named presets, semantic role tokens, and a thin runtime API.

**Architecture:** Themes are ordered lists of `TokenMap → TokenMap` transforms declared in JSON presets. The pipeline runs at build time, producing DTCG JSON + CSS per preset. The browser runtime is thin: `applyTheme(name)` swaps pre-built CSS. A `<pages-theme-picker>` LitElement enables in-app theme switching.

**Tech Stack:** TypeScript 5, Vitest, OKLCH colour science, DTCG v2025.10, Lit ^3.3.3 (theme picker only)

## Global Constraints

- Package version: `0.3.0` (breaking — pre-release, no backward compat)
- CSS custom properties: `--pages-` prefix, lowercase, hyphen-separated (protocol PP-20260705-2ae91d)
- OKLCH 12-step scales: steps 1-12 with perceptual lightness mapping
- Token format: `oklch({L}% {C} {H})`
- All transforms are pure functions: `(tokens: TokenMap, params?) → TokenMap`
- Transforms compound: each receives the accumulated TokenMap from prior transforms
- `$extends` is append-only: child cannot remove/replace parent transforms
- Built-in presets: `default-light`, `default-dark`, `casehub-light`, `casehub-dark`
- Density variants orthogonal to theming (existing `.pages-density-compact` mechanism unchanged)
- IntelliJ MCP required for all code operations: `mcp__intellij-index__*` tools, project_path `/Users/mdproctor/claude/casehub/worktrees/33/pages`

---

### Task 1: Foundation — types, transform registry, pipeline runner

**Files:**
- Create: `packages/pages-ui-tokens/src/types.ts`
- Create: `packages/pages-ui-tokens/src/registry.ts`
- Create: `packages/pages-ui-tokens/src/pipeline.ts`
- Create: `packages/pages-ui-tokens/src/preset-loader.ts`
- Test: `packages/pages-ui-tokens/src/types.test.ts`
- Test: `packages/pages-ui-tokens/src/pipeline.test.ts`

**Interfaces:**
- Consumes: nothing (foundation)
- Produces:
  - `TokenLeaf` — `{ $value: string; $type: string }`
  - `TokenMap` — `{ [key: string]: TokenLeaf | TokenMap }`
  - `TransformFn` — `(tokens: TokenMap, params: Record<string, unknown>) => TokenMap`
  - `TransformDef` — `{ transform: string; params?: Record<string, unknown> }`
  - `PresetConfig` — `{ $name: string; $description?: string; $extends?: string; pipeline: TransformDef[] }`
  - `isTokenLeaf(v: unknown): v is TokenLeaf`
  - `registerTransform(name: string, fn: TransformFn): void`
  - `getTransform(name: string): TransformFn`
  - `runPipeline(preset: PresetConfig): TokenMap`
  - `loadPreset(nameOrPath: string): PresetConfig`
  - `resolvePresetChain(preset: PresetConfig): TransformDef[]`

- [ ] **Step 1: Write types.ts**

```typescript
// packages/pages-ui-tokens/src/types.ts
export type TokenLeaf = { readonly $value: string; readonly $type: string };
export type TokenMap = { readonly [key: string]: TokenLeaf | TokenMap };

export type TransformFn = (tokens: TokenMap, params: Record<string, unknown>) => TokenMap;

export interface TransformDef {
  readonly transform: string;
  readonly params?: Record<string, unknown>;
}

export interface PresetConfig {
  readonly $name: string;
  readonly $description?: string;
  readonly $extends?: string;
  readonly pipeline: readonly TransformDef[];
}

export function isTokenLeaf(v: unknown): v is TokenLeaf {
  return v !== null && typeof v === 'object' && '$value' in v && '$type' in v;
}
```

- [ ] **Step 2: Write type guard test**

```typescript
// packages/pages-ui-tokens/src/types.test.ts
import { describe, it, expect } from 'vitest';
import { isTokenLeaf } from './types.js';
import type { TokenMap } from './types.js';

describe('isTokenLeaf', () => {
  it('returns true for valid TokenLeaf', () => {
    expect(isTokenLeaf({ $value: 'oklch(50% 0.1 210)', $type: 'color' })).toBe(true);
  });

  it('returns false for TokenMap group', () => {
    expect(isTokenLeaf({ accent: { $value: 'x', $type: 'color' } })).toBe(false);
  });

  it('returns false for null', () => {
    expect(isTokenLeaf(null)).toBe(false);
  });

  it('returns false for primitives', () => {
    expect(isTokenLeaf('hello')).toBe(false);
    expect(isTokenLeaf(42)).toBe(false);
  });
});
```

- [ ] **Step 3: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: types.test.ts passes

- [ ] **Step 4: Write transform registry**

```typescript
// packages/pages-ui-tokens/src/registry.ts
import type { TransformFn } from './types.js';

const transforms = new Map<string, TransformFn>();

export function registerTransform(name: string, fn: TransformFn): void {
  transforms.set(name, fn);
}

export function getTransform(name: string): TransformFn {
  const fn = transforms.get(name);
  if (!fn) throw new Error(`Unknown transform: "${name}". Registered: ${[...transforms.keys()].join(', ')}`);
  return fn;
}

export function listTransforms(): string[] {
  return [...transforms.keys()];
}
```

- [ ] **Step 5: Write preset-loader.ts**

```typescript
// packages/pages-ui-tokens/src/preset-loader.ts
import type { PresetConfig, TransformDef } from './types.js';

const builtinPresets = new Map<string, PresetConfig>();

export function registerBuiltinPreset(preset: PresetConfig): void {
  builtinPresets.set(preset.$name, preset);
}

export function getBuiltinPreset(name: string): PresetConfig | undefined {
  return builtinPresets.get(name);
}

export function resolvePresetChain(preset: PresetConfig, basePath?: string): TransformDef[] {
  const visited = new Set<string>();
  return resolveChain(preset, visited, basePath);
}

function resolveChain(preset: PresetConfig, visited: Set<string>, basePath?: string): TransformDef[] {
  if (preset.$name && visited.has(preset.$name)) {
    throw new Error(`Circular $extends: ${[...visited, preset.$name].join(' → ')}`);
  }
  if (preset.$name) visited.add(preset.$name);

  if (!preset.$extends) return [...preset.pipeline];

  // Built-in name resolution first
  const parent = builtinPresets.get(preset.$extends);
  if (parent) {
    return [...resolveChain(parent, visited, basePath), ...preset.pipeline];
  }

  // File path resolution (relative paths for custom presets — Node.js only)
  if (basePath && (preset.$extends.startsWith('./') || preset.$extends.startsWith('../'))) {
    try {
      const { readFileSync } = await import('node:fs');
      const { resolve, dirname } = await import('node:path');
      const resolvedPath = resolve(basePath, preset.$extends);
      const raw = JSON.parse(readFileSync(resolvedPath, 'utf-8')) as PresetConfig;
      return [...resolveChain(raw, visited, dirname(resolvedPath)), ...preset.pipeline];
    } catch (e) {
      throw new Error(`Cannot resolve $extends file: "${preset.$extends}" from ${basePath}: ${e instanceof Error ? e.message : e}`);
    }
  }

  throw new Error(`Cannot resolve $extends: "${preset.$extends}". Available built-ins: ${[...builtinPresets.keys()].join(', ')}. Use a relative file path (./path.json) for custom presets.`);
}

// Note: resolveChain uses dynamic import for node:fs — this runs only in
// the CLI/build path, never in the browser runtime. The browser runtime
// only resolves built-in names from the registry.
```

- [ ] **Step 6: Write pipeline.ts**

```typescript
// packages/pages-ui-tokens/src/pipeline.ts
import type { TokenMap, PresetConfig } from './types.js';
import { getTransform } from './registry.js';
import { resolvePresetChain } from './preset-loader.js';
import { SPACING_SCALE, TYPOGRAPHY, MOTION, RADIUS } from './tokens.js';

function buildInitialTokenMap(): TokenMap {
  const tokens: Record<string, Record<string, { $value: string; $type: string }>> = {};

  tokens['spacing'] = {};
  for (const [key, value] of Object.entries(SPACING_SCALE)) {
    tokens['spacing'][key] = { $value: value, $type: 'dimension' };
  }

  tokens['font-size'] = {};
  for (const [key, value] of Object.entries(TYPOGRAPHY.sizes)) {
    tokens['font-size'][key] = { $value: value, $type: 'dimension' };
  }

  tokens['line-height'] = {};
  for (const [key, value] of Object.entries(TYPOGRAPHY.lineHeights)) {
    tokens['line-height'][key] = { $value: value, $type: 'dimension' };
  }

  tokens['font-weight'] = {};
  for (const [key, value] of Object.entries(TYPOGRAPHY.weights)) {
    tokens['font-weight'][key] = { $value: String(value), $type: 'fontWeight' };
  }

  tokens['font-family'] = { $value: TYPOGRAPHY.family, $type: 'fontFamily' } as unknown as Record<string, { $value: string; $type: string }>;

  tokens['duration'] = {};
  for (const [key, value] of Object.entries(MOTION.duration)) {
    tokens['duration'][key] = { $value: value, $type: 'duration' };
  }

  tokens['ease'] = {};
  for (const [key, value] of Object.entries(MOTION.easing)) {
    tokens['ease'][key] = { $value: value, $type: 'cubicBezier' };
  }

  tokens['radius'] = {};
  for (const [key, value] of Object.entries(RADIUS)) {
    tokens['radius'][key] = { $value: value, $type: 'dimension' };
  }

  return tokens as unknown as TokenMap;
}

export function runPipeline(preset: PresetConfig): TokenMap {
  const chain = resolvePresetChain(preset);
  let tokens: TokenMap = buildInitialTokenMap();

  for (const def of chain) {
    const fn = getTransform(def.transform);
    tokens = fn(tokens, def.params ?? {});
  }

  return tokens;
}

export { buildInitialTokenMap };
```

- [ ] **Step 7: Write pipeline tests**

```typescript
// packages/pages-ui-tokens/src/pipeline.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import type { TokenMap, PresetConfig, TransformFn } from './types.js';
import { registerTransform } from './registry.js';
import { registerBuiltinPreset, resolvePresetChain } from './preset-loader.js';
import { runPipeline, buildInitialTokenMap } from './pipeline.js';

describe('buildInitialTokenMap', () => {
  it('contains spacing tokens', () => {
    const tokens = buildInitialTokenMap();
    const spacing = tokens['spacing'] as TokenMap;
    expect(spacing['1']).toEqual({ $value: '4px', $type: 'dimension' });
    expect(spacing['0-5']).toEqual({ $value: '2px', $type: 'dimension' });
  });

  it('contains typography tokens', () => {
    const tokens = buildInitialTokenMap();
    const fontSize = tokens['font-size'] as TokenMap;
    expect(fontSize['base']).toEqual({ $value: '14px', $type: 'dimension' });
  });

  it('contains radius tokens', () => {
    const tokens = buildInitialTokenMap();
    const radius = tokens['radius'] as TokenMap;
    expect(radius['md']).toEqual({ $value: '6px', $type: 'dimension' });
  });
});

describe('resolvePresetChain', () => {
  beforeEach(() => {
    registerBuiltinPreset({
      $name: 'base',
      pipeline: [{ transform: 'a' }],
    });
    registerBuiltinPreset({
      $name: 'child',
      $extends: 'base',
      pipeline: [{ transform: 'b' }],
    });
  });

  it('returns own pipeline when no $extends', () => {
    const chain = resolvePresetChain({ $name: 'standalone', pipeline: [{ transform: 'x' }] });
    expect(chain).toEqual([{ transform: 'x' }]);
  });

  it('prepends parent pipeline for $extends', () => {
    const chain = resolvePresetChain({ $name: 'child', $extends: 'base', pipeline: [{ transform: 'b' }] });
    expect(chain).toEqual([{ transform: 'a' }, { transform: 'b' }]);
  });

  it('throws on circular $extends', () => {
    registerBuiltinPreset({ $name: 'loop-a', $extends: 'loop-b', pipeline: [] });
    registerBuiltinPreset({ $name: 'loop-b', $extends: 'loop-a', pipeline: [] });
    expect(() => resolvePresetChain({ $name: 'loop-a', $extends: 'loop-b', pipeline: [] }))
      .toThrow(/Circular/);
  });

  it('throws on unresolvable $extends', () => {
    expect(() => resolvePresetChain({ $name: 'x', $extends: 'nonexistent', pipeline: [] }))
      .toThrow(/Cannot resolve/);
  });
});

describe('runPipeline', () => {
  beforeEach(() => {
    registerTransform('add-color', (tokens: TokenMap) => ({
      ...tokens,
      color: { test: { $value: 'oklch(50% 0.1 210)', $type: 'color' } },
    }));
  });

  it('executes transforms in order and returns final TokenMap', () => {
    const preset: PresetConfig = {
      $name: 'test',
      pipeline: [{ transform: 'add-color' }],
    };
    const result = runPipeline(preset);
    const color = result['color'] as TokenMap;
    expect(color['test']).toEqual({ $value: 'oklch(50% 0.1 210)', $type: 'color' });
  });

  it('includes initial static tokens', () => {
    const result = runPipeline({ $name: 'test', pipeline: [] });
    expect(result['spacing']).toBeDefined();
    expect(result['radius']).toBeDefined();
  });

  it('throws on unknown transform', () => {
    expect(() => runPipeline({ $name: 'test', pipeline: [{ transform: 'nonexistent' }] }))
      .toThrow(/Unknown transform/);
  });
});
```

- [ ] **Step 8: Run all tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All new tests pass, all existing tests still pass

- [ ] **Step 9: Commit**

```
feat(#230): add pipeline foundation — types, registry, preset loader, runner
```

---

### Task 2: Core transforms — oklch-scale, light-mode, dark-mode

**Files:**
- Create: `packages/pages-ui-tokens/src/transforms/oklch-scale.ts`
- Create: `packages/pages-ui-tokens/src/transforms/light-mode.ts`
- Create: `packages/pages-ui-tokens/src/transforms/dark-mode.ts`
- Create: `packages/pages-ui-tokens/src/transforms/index.ts`
- Test: `packages/pages-ui-tokens/src/transforms/oklch-scale.test.ts`
- Test: `packages/pages-ui-tokens/src/transforms/mode.test.ts`

**Interfaces:**
- Consumes: `TokenMap`, `TransformFn`, `registerTransform` from Task 1; `generateScale` from `colours.ts`
- Produces:
  - `oklchScale(tokens, params)` — adds `{name}-{1..12}` colour token groups for each hue in `params.hues`
  - `lightMode(tokens)` — sets `$mode: light`, adds light elevation + surface tokens
  - `darkMode(tokens)` — sets `$mode: dark`, adds dark elevation + surface tokens
  - `registerCoreTransforms()` — registers all three with the transform registry

- [ ] **Step 1: Write oklch-scale transform**

```typescript
// packages/pages-ui-tokens/src/transforms/oklch-scale.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { generateScale } from '../colours.js';

export interface OklchScaleParams {
  readonly hues: Record<string, number>;
  readonly chroma?: number;
  readonly contrast?: number;
}

export function oklchScale(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const { hues, chroma = 0.12, contrast = 0.5 } = params as unknown as OklchScaleParams;
  const mode = (tokens['$mode'] as TokenLeaf | undefined)?.$value ?? 'light';
  const isDark = mode === 'dark';

  const result: Record<string, unknown> = { ...tokens };

  for (const [name, hue] of Object.entries(hues)) {
    const chromaVal = name === 'neutral' ? chroma * 0.15 : chroma;
    const scale = generateScale(hue, chromaVal, contrast, isDark);
    const group: Record<string, TokenLeaf> = {};
    for (const [step, value] of Object.entries(scale)) {
      group[step] = { $value: value, $type: 'color' };
    }
    result[name] = { ...(result[name] as Record<string, unknown> ?? {}), ...group };
  }

  return result as TokenMap;
}
```

- [ ] **Step 2: Write mode transforms**

```typescript
// packages/pages-ui-tokens/src/transforms/light-mode.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { ELEVATION_LIGHT } from '../tokens.js';

export function lightMode(tokens: TokenMap, _params: Record<string, unknown>): TokenMap {
  const result: Record<string, unknown> = { ...tokens };

  result['$mode'] = { $value: 'light', $type: 'meta' };

  const shadow: Record<string, TokenLeaf> = {};
  for (const [key, value] of Object.entries(ELEVATION_LIGHT.shadow)) {
    shadow[key] = { $value: value, $type: 'shadow' };
  }
  result['shadow'] = shadow;

  const surface: Record<string, TokenLeaf> = {};
  for (let i = 1; i <= 4; i++) {
    const opacity = 0.02 + (i * 0.02);
    surface[String(i)] = { $value: `oklch(0% 0 0 / ${opacity.toFixed(2)})`, $type: 'color' };
  }
  result['surface'] = surface;

  return result as TokenMap;
}
```

```typescript
// packages/pages-ui-tokens/src/transforms/dark-mode.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { ELEVATION_DARK } from '../tokens.js';

export function darkMode(tokens: TokenMap, _params: Record<string, unknown>): TokenMap {
  const result: Record<string, unknown> = { ...tokens };

  result['$mode'] = { $value: 'dark', $type: 'meta' };

  const shadow: Record<string, TokenLeaf> = {};
  for (const [key, value] of Object.entries(ELEVATION_DARK.shadow)) {
    shadow[key] = { $value: value, $type: 'shadow' };
  }
  result['shadow'] = shadow;

  const surface: Record<string, TokenLeaf> = {};
  for (let i = 1; i <= 4; i++) {
    const opacity = 0.05 + (i * 0.03);
    surface[String(i)] = { $value: `oklch(100% 0 0 / ${opacity.toFixed(2)})`, $type: 'color' };
  }
  result['surface'] = surface;

  return result as TokenMap;
}
```

- [ ] **Step 3: Write transforms/index.ts with registration**

```typescript
// packages/pages-ui-tokens/src/transforms/index.ts
import { registerTransform } from '../registry.js';
import { oklchScale } from './oklch-scale.js';
import { lightMode } from './light-mode.js';
import { darkMode } from './dark-mode.js';

export function registerCoreTransforms(): void {
  registerTransform('oklch-scale', oklchScale);
  registerTransform('light-mode', lightMode);
  registerTransform('dark-mode', darkMode);
}

export { oklchScale } from './oklch-scale.js';
export { lightMode } from './light-mode.js';
export { darkMode } from './dark-mode.js';
```

- [ ] **Step 4: Write oklch-scale test**

```typescript
// packages/pages-ui-tokens/src/transforms/oklch-scale.test.ts
import { describe, it, expect } from 'vitest';
import { oklchScale } from './oklch-scale.js';
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

describe('oklch-scale transform', () => {
  const lightTokens: TokenMap = { $mode: { $value: 'light', $type: 'meta' } };
  const darkTokens: TokenMap = { $mode: { $value: 'dark', $type: 'meta' } };

  it('generates 12-step scale for each hue', () => {
    const result = oklchScale(lightTokens, { hues: { accent: 245 } });
    const accent = result['accent'] as TokenMap;
    for (let i = 1; i <= 12; i++) {
      expect(isTokenLeaf(accent[String(i)])).toBe(true);
    }
  });

  it('generates multiple hue scales', () => {
    const result = oklchScale(lightTokens, { hues: { accent: 245, info: 210, success: 145 } });
    expect(result['accent']).toBeDefined();
    expect(result['info']).toBeDefined();
    expect(result['success']).toBeDefined();
  });

  it('applies 0.15 chroma multiplier for neutral', () => {
    const result = oklchScale(lightTokens, { hues: { neutral: 220 }, chroma: 0.12 });
    const neutral = result['neutral'] as TokenMap;
    const step6 = (neutral['6'] as TokenLeaf).$value;
    const chroma = parseFloat(step6.match(/oklch\(\d+\.\d+% (\d+\.\d+)/)![1]!);
    expect(chroma).toBeCloseTo(0.018, 3);
  });

  it('uses light steps when $mode is light', () => {
    const result = oklchScale(lightTokens, { hues: { accent: 245 } });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    const lightness = parseFloat(step1.match(/oklch\((\d+\.\d+)%/)![1]!);
    expect(lightness).toBeGreaterThan(97);
  });

  it('uses dark steps when $mode is dark', () => {
    const result = oklchScale(darkTokens, { hues: { accent: 245 } });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    const lightness = parseFloat(step1.match(/oklch\((\d+\.\d+)%/)![1]!);
    expect(lightness).toBeLessThan(10);
  });

  it('is additive — preserves existing tokens', () => {
    const existing: TokenMap = { ...lightTokens, existing: { $value: 'keep', $type: 'test' } };
    const result = oklchScale(existing, { hues: { accent: 245 } });
    expect(result['existing']).toEqual({ $value: 'keep', $type: 'test' });
  });

  it('defaults to light mode when $mode absent', () => {
    const result = oklchScale({}, { hues: { accent: 245 } });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    const lightness = parseFloat(step1.match(/oklch\((\d+\.\d+)%/)![1]!);
    expect(lightness).toBeGreaterThan(97);
  });
});
```

- [ ] **Step 5: Write mode transform tests**

```typescript
// packages/pages-ui-tokens/src/transforms/mode.test.ts
import { describe, it, expect } from 'vitest';
import { lightMode } from './light-mode.js';
import { darkMode } from './dark-mode.js';
import type { TokenMap, TokenLeaf } from '../types.js';

describe('lightMode transform', () => {
  it('sets $mode to light', () => {
    const result = lightMode({}, {});
    expect((result['$mode'] as TokenLeaf).$value).toBe('light');
  });

  it('adds light elevation shadows', () => {
    const result = lightMode({}, {});
    const shadow = result['shadow'] as TokenMap;
    expect(shadow['1']).toBeDefined();
    expect(shadow['4']).toBeDefined();
    expect((shadow['1'] as TokenLeaf).$value).toContain('0.05');
  });

  it('adds light surface overlays', () => {
    const result = lightMode({}, {});
    const surface = result['surface'] as TokenMap;
    expect((surface['1'] as TokenLeaf).$value).toContain('oklch(0%');
  });

  it('preserves existing tokens', () => {
    const result = lightMode({ existing: { $value: 'keep', $type: 'test' } });
    expect(result['existing']).toEqual({ $value: 'keep', $type: 'test' });
  });
});

describe('darkMode transform', () => {
  it('sets $mode to dark', () => {
    const result = darkMode({}, {});
    expect((result['$mode'] as TokenLeaf).$value).toBe('dark');
  });

  it('adds dark elevation shadows', () => {
    const result = darkMode({}, {});
    const shadow = result['shadow'] as TokenMap;
    expect((shadow['1'] as TokenLeaf).$value).toContain('0.3');
  });

  it('adds dark surface overlays with oklch(100%', () => {
    const result = darkMode({}, {});
    const surface = result['surface'] as TokenMap;
    expect((surface['1'] as TokenLeaf).$value).toContain('oklch(100%');
  });
});
```

- [ ] **Step 6: Run all tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All transform tests pass, all existing tests still pass

- [ ] **Step 7: Commit**

```
feat(#230): add core transforms — oklch-scale, light-mode, dark-mode
```

---

### Task 3: CSS/DTCG output generator

**Files:**
- Create: `packages/pages-ui-tokens/src/output.ts`
- Test: `packages/pages-ui-tokens/src/output.test.ts`

**Interfaces:**
- Consumes: `TokenMap`, `TokenLeaf`, `isTokenLeaf` from Task 1
- Produces:
  - `generateCSS(tokens: TokenMap, name: string): string` — TokenMap → CSS scoped under `.pages-theme-{name}`
  - `generateDTCG(tokens: TokenMap, name: string): object` — TokenMap → DTCG JSON object
  - `generateDensityCSS(): string` — density compact overrides (unchanged)

- [ ] **Step 1: Write failing tests for CSS output**

```typescript
// packages/pages-ui-tokens/src/output.test.ts
import { describe, it, expect } from 'vitest';
import { generateCSS, generateDTCG, generateDensityCSS } from './output.js';
import type { TokenMap } from './types.js';

const sampleTokens: TokenMap = {
  accent: {
    '1': { $value: 'oklch(98.5% 0.036 245)', $type: 'color' },
    '2': { $value: 'oklch(96.0% 0.072 245)', $type: 'color' },
  },
  shadow: {
    '1': { $value: '0 1px 2px oklch(0% 0 0 / 0.05)', $type: 'shadow' },
  },
  spacing: {
    '1': { $value: '4px', $type: 'dimension' },
  },
  'font-family': { $value: "'Inter', system-ui", $type: 'fontFamily' },
  $mode: { $value: 'dark', $type: 'meta' },
};

describe('generateCSS', () => {
  it('scopes under .pages-theme-{name}', () => {
    const css = generateCSS(sampleTokens, 'casehub-dark');
    expect(css).toContain('.pages-theme-casehub-dark {');
  });

  it('generates --pages- prefixed custom properties', () => {
    const css = generateCSS(sampleTokens, 'test');
    expect(css).toContain('--pages-accent-1: oklch(98.5% 0.036 245);');
    expect(css).toContain('--pages-accent-2: oklch(96.0% 0.072 245);');
  });

  it('generates shadow tokens', () => {
    const css = generateCSS(sampleTokens, 'test');
    expect(css).toContain('--pages-shadow-1:');
  });

  it('generates spacing tokens', () => {
    const css = generateCSS(sampleTokens, 'test');
    expect(css).toContain('--pages-space-1: 4px;');
  });

  it('generates font-family as flat token', () => {
    const css = generateCSS(sampleTokens, 'test');
    expect(css).toContain("--pages-font-family: 'Inter', system-ui;");
  });

  it('skips $-prefixed metadata keys', () => {
    const css = generateCSS(sampleTokens, 'test');
    expect(css).not.toContain('$mode');
  });
});

describe('generateDTCG', () => {
  it('includes $name in output', () => {
    const dtcg = generateDTCG(sampleTokens, 'casehub-dark');
    expect(dtcg['$name']).toBe('casehub-dark');
  });

  it('preserves token structure with $value and $type', () => {
    const dtcg = generateDTCG(sampleTokens, 'test') as Record<string, unknown>;
    const color = dtcg['color'] as Record<string, unknown>;
    const accent = color['accent'] as Record<string, unknown>;
    expect(accent['1']).toEqual({ $value: 'oklch(98.5% 0.036 245)', $type: 'color' });
  });

  it('excludes $-prefixed metadata', () => {
    const dtcg = generateDTCG(sampleTokens, 'test');
    expect(dtcg['$mode']).toBeUndefined();
  });
});

describe('generateDensityCSS', () => {
  it('generates .pages-density-compact class', () => {
    const css = generateDensityCSS();
    expect(css).toContain('.pages-density-compact {');
  });

  it('contains compact spacing overrides', () => {
    const css = generateDensityCSS();
    expect(css).toContain('--pages-space-1: 3px;');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: FAIL — module './output.js' not found

- [ ] **Step 3: Write output.ts**

```typescript
// packages/pages-ui-tokens/src/output.ts
import type { TokenMap } from './types.js';
import { isTokenLeaf } from './types.js';
import { DENSITY_COMPACT_OVERRIDES } from './tokens.js';

const CSS_PREFIX_MAP: Record<string, string> = {
  spacing: 'space',
  'font-size': 'font-size',
  'line-height': 'line-height',
  'font-weight': 'font-weight',
  'font-family': 'font-family',
  duration: 'duration',
  ease: 'ease',
  radius: 'radius',
  shadow: 'shadow',
  surface: 'surface',
};

function tokensToCSSLines(tokens: TokenMap, prefix: string = ''): string[] {
  const lines: string[] = [];
  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) continue;

    if (isTokenLeaf(value)) {
      const cssName = prefix ? `--pages-${prefix}-${key}` : `--pages-${key}`;
      lines.push(`  ${cssName}: ${value.$value};`);
    } else {
      const cssPrefix = CSS_PREFIX_MAP[key] ?? key;
      const nestedPrefix = prefix ? `${prefix}-${cssPrefix}` : cssPrefix;

      const group = value as TokenMap;
      const hasOnlyLeaves = Object.values(group).every(v => isTokenLeaf(v));
      if (hasOnlyLeaves) {
        for (const [subKey, subValue] of Object.entries(group)) {
          if (isTokenLeaf(subValue)) {
            lines.push(`  --pages-${nestedPrefix}-${subKey}: ${subValue.$value};`);
          }
        }
      } else {
        lines.push(...tokensToCSSLines(group, nestedPrefix));
      }
    }
  }
  return lines;
}

export function generateCSS(tokens: TokenMap, name: string): string {
  const lines = tokensToCSSLines(tokens);
  return `.pages-theme-${name} {\n${lines.join('\n')}\n}`;
}

export function generateDTCG(tokens: TokenMap, name: string): Record<string, unknown> {
  const result: Record<string, unknown> = { $name: name };

  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) continue;

    if (isTokenLeaf(value)) {
      const category = guessDTCGCategory(key);
      if (!result[category]) result[category] = {};
      (result[category] as Record<string, unknown>)[key] = { $value: value.$value, $type: value.$type };
    } else {
      const category = guessDTCGCategory(key);
      if (!result[category]) result[category] = {};
      const target = result[category] as Record<string, unknown>;
      target[key] = deepCopyTokens(value as TokenMap);
    }
  }

  return result;
}

function guessDTCGCategory(key: string): string {
  if (['accent', 'neutral', 'success', 'warning', 'danger', 'info', 'surface'].includes(key)) return 'color';
  if (['shadow'].includes(key)) return 'elevation';
  if (['spacing', 'radius'].includes(key)) return 'spacing';
  if (key.startsWith('font') || key.startsWith('line-height')) return 'typography';
  if (['duration', 'ease'].includes(key)) return 'motion';
  return key;
}

function deepCopyTokens(tokens: TokenMap): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(tokens)) {
    if (isTokenLeaf(value)) {
      result[key] = { $value: value.$value, $type: value.$type };
    } else {
      result[key] = deepCopyTokens(value as TokenMap);
    }
  }
  return result;
}

export function generateDensityCSS(): string {
  const lines = Object.entries(DENSITY_COMPACT_OVERRIDES)
    .map(([key, value]) => `  ${key}: ${value};`);
  return `.pages-density-compact {\n${lines.join('\n')}\n}`;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All output tests pass

- [ ] **Step 5: Commit**

```
feat(#230): add CSS and DTCG output generators
```

---

### Task 4: Default presets + backward compatibility regression

**Files:**
- Create: `packages/pages-ui-tokens/presets/default-light.json`
- Create: `packages/pages-ui-tokens/presets/default-dark.json`
- Modify: `packages/pages-ui-tokens/src/transforms/index.ts` — add preset registration
- Test: `packages/pages-ui-tokens/src/presets.test.ts`

**Interfaces:**
- Consumes: `runPipeline`, `generateCSS`, `generateDensityCSS`, `registerCoreTransforms`, `registerBuiltinPreset` from Tasks 1-3
- Produces:
  - Built-in preset JSON files for `default-light` and `default-dark`
  - `initPresets()` — loads and registers all built-in presets
  - Regression proof: pipeline CSS matches current `generateThemeCSS` output

- [ ] **Step 1: Create default-light.json**

```json
{
  "$name": "default-light",
  "$description": "Pages generic light theme",
  "pipeline": [
    { "transform": "light-mode" },
    {
      "transform": "oklch-scale",
      "params": {
        "hues": {
          "accent": 245,
          "neutral": 220,
          "success": 145,
          "warning": 55,
          "danger": 25,
          "info": 210
        },
        "chroma": 0.12,
        "contrast": 0.5
      }
    }
  ]
}
```

- [ ] **Step 2: Create default-dark.json**

```json
{
  "$name": "default-dark",
  "$description": "Pages generic dark theme",
  "pipeline": [
    { "transform": "dark-mode" },
    {
      "transform": "oklch-scale",
      "params": {
        "hues": {
          "accent": 245,
          "neutral": 220,
          "success": 145,
          "warning": 55,
          "danger": 25,
          "info": 210
        },
        "chroma": 0.12,
        "contrast": 0.5
      }
    }
  ]
}
```

- [ ] **Step 3: Add preset loading to transforms/index.ts**

Update `transforms/index.ts` to add an `initPresets()` function. Use inline preset constants (the JSON files in `presets/` serve as the human-readable source of truth; the code mirrors them):

```typescript
// Add to transforms/index.ts
import { registerBuiltinPreset } from '../preset-loader.js';
import type { PresetConfig } from '../types.js';

const DEFAULT_LIGHT_PRESET: PresetConfig = {
  $name: 'default-light',
  $description: 'Pages generic light theme',
  pipeline: [
    { transform: 'light-mode' },
    { transform: 'oklch-scale', params: { hues: { accent: 245, neutral: 220, success: 145, warning: 55, danger: 25, info: 210 }, chroma: 0.12, contrast: 0.5 } },
  ],
};

const DEFAULT_DARK_PRESET: PresetConfig = {
  $name: 'default-dark',
  $description: 'Pages generic dark theme',
  pipeline: [
    { transform: 'dark-mode' },
    { transform: 'oklch-scale', params: { hues: { accent: 245, neutral: 220, success: 145, warning: 55, danger: 25, info: 210 }, chroma: 0.12, contrast: 0.5 } },
  ],
};

export function initPresets(): void {
  registerCoreTransforms();
  registerBuiltinPreset(DEFAULT_LIGHT_PRESET);
  registerBuiltinPreset(DEFAULT_DARK_PRESET);
}
```

- [ ] **Step 4: Write backward compatibility regression test**

This is the critical test. The pipeline output must produce the same CSS custom property values as the current `generateThemeCSS`. The format may differ (class naming changes from `.pages-theme-light`/`.pages-theme-dark` to `.pages-theme-default-light`/`.pages-theme-default-dark`), but every `--pages-*` property value must be identical.

```typescript
// packages/pages-ui-tokens/src/presets.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { generateThemeCSS, DEFAULT_THEME } from './themes.js';
import { runPipeline } from './pipeline.js';
import { generateCSS, generateDensityCSS } from './output.js';
import { initPresets } from './transforms/index.js';
import type { PresetConfig } from './types.js';

function extractCSSProperties(css: string): Map<string, string> {
  const props = new Map<string, string>();
  const regex = /(--pages-[a-z0-9-]+):\s*([^;]+);/g;
  let match;
  while ((match = regex.exec(css)) !== null) {
    props.set(match[1]!, match[2]!.trim());
  }
  return props;
}

describe('backward compatibility — default presets match generateThemeCSS', () => {
  beforeAll(() => {
    initPresets();
  });

  it('default-light produces same colour tokens as generateThemeCSS light block', () => {
    const oldCSS = generateThemeCSS(DEFAULT_THEME);
    const lightBlock = oldCSS.match(/\.pages-theme-light \{([^}]+)\}/s)?.[1] ?? '';
    const oldProps = extractCSSProperties(lightBlock);

    const lightPreset: PresetConfig = { $name: 'default-light', $description: '', pipeline: [
      { transform: 'light-mode' },
      { transform: 'oklch-scale', params: { hues: { accent: 245, neutral: 220, success: 145, warning: 55, danger: 25, info: 210 }, chroma: 0.12, contrast: 0.5 } },
    ]};
    const tokens = runPipeline(lightPreset);
    const newCSS = generateCSS(tokens, 'default-light');
    const newProps = extractCSSProperties(newCSS);

    const hues = ['accent', 'neutral', 'success', 'warning', 'danger', 'info'];
    for (const hue of hues) {
      for (let step = 1; step <= 12; step++) {
        const key = `--pages-${hue}-${step}`;
        expect(newProps.get(key), `${key} missing in pipeline output`).toBeDefined();
        expect(newProps.get(key), `${key} value mismatch`).toBe(oldProps.get(key));
      }
    }
  });

  it('default-dark produces same colour tokens as generateThemeCSS dark block', () => {
    const oldCSS = generateThemeCSS(DEFAULT_THEME);
    const darkBlock = oldCSS.match(/\.pages-theme-dark \{([^}]+)\}/s)?.[1] ?? '';
    const oldProps = extractCSSProperties(darkBlock);

    const darkPreset: PresetConfig = { $name: 'default-dark', $description: '', pipeline: [
      { transform: 'dark-mode' },
      { transform: 'oklch-scale', params: { hues: { accent: 245, neutral: 220, success: 145, warning: 55, danger: 25, info: 210 }, chroma: 0.12, contrast: 0.5 } },
    ]};
    const tokens = runPipeline(darkPreset);
    const newCSS = generateCSS(tokens, 'default-dark');
    const newProps = extractCSSProperties(newCSS);

    const hues = ['accent', 'neutral', 'success', 'warning', 'danger', 'info'];
    for (const hue of hues) {
      for (let step = 1; step <= 12; step++) {
        const key = `--pages-${hue}-${step}`;
        expect(newProps.get(key), `${key} missing in pipeline output`).toBeDefined();
        expect(newProps.get(key), `${key} value mismatch`).toBe(oldProps.get(key));
      }
    }
  });

  it('shadow tokens match between old and new for both modes', () => {
    const oldCSS = generateThemeCSS(DEFAULT_THEME);

    for (const mode of ['light', 'dark'] as const) {
      const block = oldCSS.match(new RegExp(`\\.pages-theme-${mode} \\{([^}]+)\\}`, 's'))?.[1] ?? '';
      const oldProps = extractCSSProperties(block);

      const presetName = `default-${mode}`;
      const preset: PresetConfig = { $name: presetName, pipeline: [
        { transform: mode === 'dark' ? 'dark-mode' : 'light-mode' },
        { transform: 'oklch-scale', params: { hues: { accent: 245, neutral: 220, success: 145, warning: 55, danger: 25, info: 210 }, chroma: 0.12, contrast: 0.5 } },
      ]};
      const tokens = runPipeline(preset);
      const newCSS = generateCSS(tokens, presetName);
      const newProps = extractCSSProperties(newCSS);

      for (let i = 1; i <= 4; i++) {
        const shadowKey = `--pages-shadow-${i}`;
        expect(newProps.get(shadowKey), `${shadowKey} (${mode}) missing`).toBeDefined();
        expect(newProps.get(shadowKey), `${shadowKey} (${mode}) mismatch`).toBe(oldProps.get(shadowKey));

        const surfaceKey = `--pages-surface-${i}`;
        expect(newProps.get(surfaceKey), `${surfaceKey} (${mode}) missing`).toBeDefined();
        expect(newProps.get(surfaceKey), `${surfaceKey} (${mode}) mismatch`).toBe(oldProps.get(surfaceKey));
      }
    }
  });

  it('shared tokens (spacing, typography, motion, radius) present in pipeline output', () => {
    const preset: PresetConfig = { $name: 'default-light', pipeline: [
      { transform: 'light-mode' },
      { transform: 'oklch-scale', params: { hues: { accent: 245, neutral: 220, success: 145, warning: 55, danger: 25, info: 210 }, chroma: 0.12, contrast: 0.5 } },
    ]};
    const tokens = runPipeline(preset);
    const css = generateCSS(tokens, 'default-light');

    expect(css).toContain('--pages-space-1: 4px;');
    expect(css).toContain('--pages-font-size-base: 14px;');
    expect(css).toContain('--pages-duration-fast: 120ms;');
    expect(css).toContain('--pages-radius-md: 6px;');
  });

  it('density CSS is unchanged', () => {
    const oldCSS = generateThemeCSS(DEFAULT_THEME);
    const oldDensity = oldCSS.match(/\.pages-density-compact \{([^}]+)\}/s)?.[1] ?? '';
    const newDensity = generateDensityCSS().match(/\.pages-density-compact \{([^}]+)\}/s)?.[1] ?? '';

    const oldProps = extractCSSProperties(oldDensity);
    const newProps = extractCSSProperties(newDensity);

    for (const [key, value] of oldProps) {
      expect(newProps.get(key), `density ${key} missing`).toBe(value);
    }
  });
});
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All backward compatibility tests pass

- [ ] **Step 6: Commit**

```
feat(#230): add default presets with backward compatibility verification
```

---

### Task 5: Compositional transforms

**Files:**
- Create: `packages/pages-ui-tokens/src/transforms/lightness-shift.ts`
- Create: `packages/pages-ui-tokens/src/transforms/lightness-steps.ts`
- Create: `packages/pages-ui-tokens/src/transforms/chroma-curve.ts`
- Create: `packages/pages-ui-tokens/src/transforms/semantic-hues.ts`
- Create: `packages/pages-ui-tokens/src/transforms/override.ts`
- Test: `packages/pages-ui-tokens/src/transforms/compositional.test.ts`
- Modify: `packages/pages-ui-tokens/src/transforms/index.ts` — register new transforms

**Interfaces:**
- Consumes: `TokenMap`, `TokenLeaf`, `isTokenLeaf` from Task 1
- Produces:
  - `lightnessShift(tokens, { offset })` — uniform lightness offset on all colour tokens
  - `lightnessSteps(tokens, { steps })` — replace the 12-step lightness array
  - `chromaCurve(tokens, { curve, ...perHueMultipliers })` — control chroma distribution
  - `semanticHues(tokens, { success?, warning?, danger?, info? })` — override default semantic hue angles
  - `override(tokens, overrides)` — set specific tokens to exact values

- [ ] **Step 1: Write compositional transforms**

```typescript
// packages/pages-ui-tokens/src/transforms/lightness-shift.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

export function lightnessShift(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const offset = (params['offset'] as number) ?? 0;
  if (offset === 0) return tokens;
  return adjustColourTokens(tokens, (value) => {
    const match = value.match(/^oklch\((\d+\.?\d*)% (\d+\.?\d*) (\d+\.?\d*)\)$/);
    if (!match) return value;
    const l = Math.max(0, Math.min(100, parseFloat(match[1]!) + offset));
    return `oklch(${l.toFixed(1)}% ${match[2]} ${match[3]})`;
  });
}

function adjustColourTokens(tokens: TokenMap, adjust: (value: string) => string): TokenMap {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) { result[key] = value; continue; }
    if (isTokenLeaf(value)) {
      result[key] = value.$type === 'color'
        ? { $value: adjust(value.$value), $type: 'color' }
        : value;
    } else {
      result[key] = adjustColourTokens(value as TokenMap, adjust);
    }
  }
  return result as TokenMap;
}
```

```typescript
// packages/pages-ui-tokens/src/transforms/lightness-steps.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

export function lightnessSteps(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const steps = params['steps'] as number[];
  if (!steps || steps.length !== 12) throw new Error('lightness-steps requires exactly 12 step values');

  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) { result[key] = value; continue; }
    if (isTokenLeaf(value)) { result[key] = value; continue; }

    const group = value as TokenMap;
    const isColourScale = Object.keys(group).some(k => /^\d+$/.test(k) && isTokenLeaf(group[k]));
    if (!isColourScale) { result[key] = value; continue; }

    const newGroup: Record<string, unknown> = {};
    for (const [step, leaf] of Object.entries(group)) {
      if (!isTokenLeaf(leaf) || leaf.$type !== 'color') { newGroup[step] = leaf; continue; }
      const idx = parseInt(step, 10) - 1;
      if (idx < 0 || idx >= 12) { newGroup[step] = leaf; continue; }
      const match = leaf.$value.match(/^oklch\(\d+\.?\d*% (\d+\.?\d*) (\d+\.?\d*)\)$/);
      if (!match) { newGroup[step] = leaf; continue; }
      const l = Math.max(0, Math.min(100, steps[idx]!));
      newGroup[step] = { $value: `oklch(${l.toFixed(1)}% ${match[1]} ${match[2]})`, $type: 'color' };
    }
    result[key] = newGroup;
  }
  return result as TokenMap;
}
```

```typescript
// packages/pages-ui-tokens/src/transforms/chroma-curve.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

export function chromaCurve(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const curve = (params['curve'] as string) ?? 'flat';
  const perHue = params as Record<string, unknown>;

  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) { result[key] = value; continue; }
    if (isTokenLeaf(value)) { result[key] = value; continue; }

    const group = value as TokenMap;
    const isColourScale = Object.keys(group).some(k => /^\d+$/.test(k) && isTokenLeaf(group[k]));
    if (!isColourScale) { result[key] = value; continue; }

    const hueMultiplier = typeof perHue[key] === 'number' ? perHue[key] as number : 1;

    const newGroup: Record<string, unknown> = {};
    for (const [step, leaf] of Object.entries(group)) {
      if (!isTokenLeaf(leaf) || leaf.$type !== 'color') { newGroup[step] = leaf; continue; }
      const match = leaf.$value.match(/^oklch\((\d+\.?\d*)% (\d+\.?\d*) (\d+\.?\d*)\)$/);
      if (!match) { newGroup[step] = leaf; continue; }
      const idx = parseInt(step, 10) - 1;
      const curveMultiplier = curveWeight(idx, curve);
      const newChroma = parseFloat(match[2]!) * hueMultiplier * curveMultiplier;
      newGroup[step] = { $value: `oklch(${match[1]}% ${newChroma.toFixed(3)} ${match[3]})`, $type: 'color' };
    }
    result[key] = newGroup;
  }
  return result as TokenMap;
}

function curveWeight(stepIndex: number, curve: string): number {
  const t = stepIndex / 11;
  switch (curve) {
    case 'gaussian': {
      const center = 5.5;
      const sigma = 3;
      return Math.exp(-0.5 * Math.pow((stepIndex - center) / sigma, 2));
    }
    case 'bezier': {
      // Cubic bezier easing — peaks at mid-range, tapers at extremes
      const p = 2 * t - 1; // [-1, 1]
      return 1 - p * p; // Inverted parabola, max at t=0.5
    }
    default: return 1; // 'flat'
  }
}
```

```typescript
// packages/pages-ui-tokens/src/transforms/semantic-hues.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';
import { generateScale } from '../colours.js';

export function semanticHues(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const mode = (tokens['$mode'] as TokenLeaf | undefined)?.$value ?? 'light';
  const isDark = mode === 'dark';
  const result: Record<string, unknown> = { ...tokens };

  for (const [name, hue] of Object.entries(params)) {
    if (typeof hue !== 'number') continue;
    if (!['success', 'warning', 'danger', 'info'].includes(name)) continue;

    const existingGroup = tokens[name] as TokenMap | undefined;
    if (!existingGroup) continue;

    const firstLeaf = Object.values(existingGroup).find(v => isTokenLeaf(v)) as TokenLeaf | undefined;
    if (!firstLeaf) continue;
    const oldChroma = parseFloat(firstLeaf.$value.match(/oklch\(\d+\.?\d*% (\d+\.?\d*)/)![1]!);

    const scale = generateScale(hue, oldChroma, 0.5, isDark);
    const group: Record<string, { $value: string; $type: string }> = {};
    for (const [step, value] of Object.entries(scale)) {
      group[step] = { $value: value, $type: 'color' };
    }
    result[name] = group;
  }

  return result as TokenMap;
}
```

```typescript
// packages/pages-ui-tokens/src/transforms/override.ts
import type { TokenMap } from '../types.js';

export function override(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const result: Record<string, unknown> = { ...tokens };
  for (const [path, value] of Object.entries(params)) {
    if (typeof value !== 'string') continue;
    const parts = path.split('.');
    if (parts.length === 1) {
      result[parts[0]!] = { $value: value, $type: 'color' };
    } else if (parts.length === 2) {
      const [group, key] = parts;
      if (!result[group!] || typeof result[group!] !== 'object') result[group!] = {};
      (result[group!] as Record<string, unknown>)[key!] = { $value: value, $type: 'color' };
    }
  }
  return result as TokenMap;
}
```

- [ ] **Step 2: Register new transforms in index.ts**

Update `transforms/index.ts` to register all five new transforms:

```typescript
import { lightnessShift } from './lightness-shift.js';
import { lightnessSteps } from './lightness-steps.js';
import { chromaCurve } from './chroma-curve.js';
import { semanticHues } from './semantic-hues.js';
import { override } from './override.js';

// Add to registerCoreTransforms():
registerTransform('lightness-shift', lightnessShift);
registerTransform('lightness-steps', lightnessSteps);
registerTransform('chroma-curve', chromaCurve);
registerTransform('semantic-hues', semanticHues);
registerTransform('override', override);
```

- [ ] **Step 3: Write tests**

```typescript
// packages/pages-ui-tokens/src/transforms/compositional.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { lightnessShift } from './lightness-shift.js';
import { lightnessSteps } from './lightness-steps.js';
import { chromaCurve } from './chroma-curve.js';
import { semanticHues } from './semantic-hues.js';
import { override } from './override.js';
import type { TokenMap, TokenLeaf } from '../types.js';

const baseTokens: TokenMap = {
  $mode: { $value: 'dark', $type: 'meta' },
  accent: {
    '1': { $value: 'oklch(8.0% 0.036 245)', $type: 'color' },
    '6': { $value: 'oklch(34.0% 0.120 245)', $type: 'color' },
    '12': { $value: 'oklch(93.0% 0.036 245)', $type: 'color' },
  },
  neutral: {
    '1': { $value: 'oklch(8.0% 0.018 220)', $type: 'color' },
    '6': { $value: 'oklch(34.0% 0.018 220)', $type: 'color' },
  },
  success: {
    '1': { $value: 'oklch(8.0% 0.120 145)', $type: 'color' },
    '6': { $value: 'oklch(34.0% 0.120 145)', $type: 'color' },
    '9': { $value: 'oklch(55.0% 0.120 145)', $type: 'color' },
  },
  spacing: { '1': { $value: '4px', $type: 'dimension' } },
};

describe('lightnessShift', () => {
  it('shifts lightness of all colour tokens by offset', () => {
    const result = lightnessShift(baseTokens, { offset: 10 });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    expect(step1).toBe('oklch(18.0% 0.036 245)');
  });

  it('clamps to 0-100 range', () => {
    const result = lightnessShift(baseTokens, { offset: -20 });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    const lightness = parseFloat(step1.match(/oklch\((\d+\.?\d*)%/)![1]!);
    expect(lightness).toBeGreaterThanOrEqual(0);
  });

  it('preserves non-colour tokens', () => {
    const result = lightnessShift(baseTokens, { offset: 10 });
    expect((result['spacing'] as TokenMap)['1']).toEqual({ $value: '4px', $type: 'dimension' });
  });

  it('is a no-op for offset=0', () => {
    const result = lightnessShift(baseTokens, { offset: 0 });
    expect(((result['accent'] as TokenMap)['1'] as TokenLeaf).$value).toBe('oklch(8.0% 0.036 245)');
  });
});

describe('lightnessSteps', () => {
  it('replaces lightness values with provided steps', () => {
    const steps = [10, 15, 20, 25, 30, 40, 50, 55, 60, 70, 80, 95];
    const result = lightnessSteps(baseTokens, { steps });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    expect(step1).toBe('oklch(10.0% 0.036 245)');
    const step6 = ((result['accent'] as TokenMap)['6'] as TokenLeaf).$value;
    expect(step6).toBe('oklch(40.0% 0.120 245)');
  });

  it('throws if steps array is not length 12', () => {
    expect(() => lightnessSteps(baseTokens, { steps: [1, 2, 3] })).toThrow(/12/);
  });
});

describe('chromaCurve', () => {
  it('applies per-hue multiplier', () => {
    const result = chromaCurve(baseTokens, { curve: 'flat', neutral: 0.02 });
    const neutral6 = ((result['neutral'] as TokenMap)['6'] as TokenLeaf).$value;
    const chroma = parseFloat(neutral6.match(/oklch\(\d+\.?\d*% (\d+\.?\d*)/)![1]!);
    expect(chroma).toBeCloseTo(0.018 * 0.02, 4);
  });

  it('applies gaussian curve shape', () => {
    const result = chromaCurve(baseTokens, { curve: 'gaussian' });
    const step1 = ((result['accent'] as TokenMap)['1'] as TokenLeaf).$value;
    const step6 = ((result['accent'] as TokenMap)['6'] as TokenLeaf).$value;
    const chroma1 = parseFloat(step1.match(/oklch\(\d+\.?\d*% (\d+\.?\d*)/)![1]!);
    const chroma6 = parseFloat(step6.match(/oklch\(\d+\.?\d*% (\d+\.?\d*)/)![1]!);
    expect(chroma1).toBeLessThan(chroma6);
  });
});

describe('semanticHues', () => {
  it('replaces success hue while preserving scale structure', () => {
    const result = semanticHues(baseTokens, { success: 175 });
    const step9 = ((result['success'] as TokenMap)['9'] as TokenLeaf).$value;
    expect(step9).toContain('175');
  });

  it('does not modify non-semantic hues', () => {
    const result = semanticHues(baseTokens, { success: 175 });
    expect(((result['accent'] as TokenMap)['1'] as TokenLeaf).$value).toBe('oklch(8.0% 0.036 245)');
  });
});

describe('override', () => {
  it('overrides a specific token value', () => {
    const result = override(baseTokens, { 'accent.1': 'oklch(20% 0.05 240)' });
    expect(((result['accent'] as TokenMap)['1'] as TokenLeaf).$value).toBe('oklch(20% 0.05 240)');
  });

  it('creates group if not present', () => {
    const result = override(baseTokens, { 'brand.primary': 'oklch(50% 0.2 270)' });
    expect(((result['brand'] as TokenMap)['primary'] as TokenLeaf).$value).toBe('oklch(50% 0.2 270)');
  });
});
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All compositional transform tests pass

- [ ] **Step 5: Commit**

```
feat(#230): add compositional transforms — lightness, chroma, semantic-hues, override
```

---

### Task 6: Semantic-map transform — role tokens

**Files:**
- Create: `packages/pages-ui-tokens/src/transforms/semantic-map.ts`
- Test: `packages/pages-ui-tokens/src/transforms/semantic-map.test.ts`
- Modify: `packages/pages-ui-tokens/src/transforms/index.ts` — register semantic-map

**Interfaces:**
- Consumes: `TokenMap`, `TokenLeaf`, `isTokenLeaf` from Task 1
- Produces:
  - `semanticMap(tokens, params)` — generates semantic role tokens from primitive scales
  - Default mappings: surface-primary→neutral-1, text-primary→neutral-12, interactive→accent-9, etc.

- [ ] **Step 1: Write semantic-map transform**

```typescript
// packages/pages-ui-tokens/src/transforms/semantic-map.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

const DEFAULT_ROLE_MAPPINGS: Record<string, string> = {
  'surface-primary': 'neutral.1',
  'surface-secondary': 'neutral.2',
  'surface-tertiary': 'neutral.3',
  'surface-hover': 'neutral.3',
  'surface-selected': 'accent.2',

  'border-subtle': 'neutral.4',
  'border-default': 'neutral.6',
  'border-strong': 'neutral.8',

  'text-primary': 'neutral.12',
  'text-secondary': 'neutral.11',
  'text-muted': 'neutral.8',
  'text-disabled': 'neutral.6',

  'interactive': 'accent.9',
  'interactive-hover': 'accent.10',
  'interactive-active': 'accent.11',
  'focus-ring': 'accent.8',

  'status-success': 'success.9',
  'status-warning': 'warning.9',
  'status-danger': 'danger.9',
  'status-info': 'info.9',
};

function resolveRef(tokens: TokenMap, ref: string): TokenLeaf | undefined {
  const [group, key] = ref.split('.');
  if (!group || !key) return undefined;
  const g = tokens[group];
  if (!g || isTokenLeaf(g)) return undefined;
  const leaf = (g as TokenMap)[key];
  return isTokenLeaf(leaf) ? leaf : undefined;
}

export function semanticMap(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const mappings = { ...DEFAULT_ROLE_MAPPINGS };

  if (params['mappings'] && typeof params['mappings'] === 'object') {
    Object.assign(mappings, params['mappings']);
  }

  const result: Record<string, unknown> = { ...tokens };
  const roles: Record<string, TokenLeaf> = {};

  for (const [roleName, ref] of Object.entries(mappings)) {
    const resolved = resolveRef(tokens, ref);
    if (resolved) {
      roles[roleName] = { $value: `var(--pages-${ref.replace('.', '-')})`, $type: 'color' };
    }
  }

  result['role'] = roles;
  return result as TokenMap;
}
```

- [ ] **Step 2: Register in transforms/index.ts**

```typescript
import { semanticMap } from './semantic-map.js';
// Add to registerCoreTransforms():
registerTransform('semantic-map', semanticMap);
```

- [ ] **Step 3: Write tests**

```typescript
// packages/pages-ui-tokens/src/transforms/semantic-map.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { semanticMap } from './semantic-map.js';
import type { TokenMap, TokenLeaf } from '../types.js';

const baseTokens: TokenMap = {
  neutral: {
    '1': { $value: 'oklch(8% 0.002 260)', $type: 'color' },
    '2': { $value: 'oklch(12% 0.003 260)', $type: 'color' },
    '3': { $value: 'oklch(17% 0.004 260)', $type: 'color' },
    '4': { $value: 'oklch(22% 0.005 260)', $type: 'color' },
    '6': { $value: 'oklch(34% 0.007 260)', $type: 'color' },
    '8': { $value: 'oklch(47% 0.009 260)', $type: 'color' },
    '9': { $value: 'oklch(55% 0.010 260)', $type: 'color' },
    '11': { $value: 'oklch(78% 0.012 260)', $type: 'color' },
    '12': { $value: 'oklch(93% 0.014 260)', $type: 'color' },
  },
  accent: {
    '2': { $value: 'oklch(12% 0.036 245)', $type: 'color' },
    '8': { $value: 'oklch(47% 0.120 245)', $type: 'color' },
    '9': { $value: 'oklch(55% 0.120 245)', $type: 'color' },
    '10': { $value: 'oklch(65% 0.120 245)', $type: 'color' },
    '11': { $value: 'oklch(78% 0.120 245)', $type: 'color' },
  },
  success: { '9': { $value: 'oklch(55% 0.120 145)', $type: 'color' } },
  warning: { '9': { $value: 'oklch(55% 0.120 55)', $type: 'color' } },
  danger: { '9': { $value: 'oklch(55% 0.120 25)', $type: 'color' } },
  info: { '9': { $value: 'oklch(55% 0.120 210)', $type: 'color' } },
};

describe('semanticMap transform', () => {
  it('generates surface role tokens', () => {
    const result = semanticMap(baseTokens, {});
    const roles = result['role'] as TokenMap;
    expect((roles['surface-primary'] as TokenLeaf).$value).toBe('var(--pages-neutral-1)');
    expect((roles['surface-secondary'] as TokenLeaf).$value).toBe('var(--pages-neutral-2)');
    expect((roles['surface-tertiary'] as TokenLeaf).$value).toBe('var(--pages-neutral-3)');
  });

  it('generates text role tokens', () => {
    const result = semanticMap(baseTokens, {});
    const roles = result['role'] as TokenMap;
    expect((roles['text-primary'] as TokenLeaf).$value).toBe('var(--pages-neutral-12)');
    expect((roles['text-muted'] as TokenLeaf).$value).toBe('var(--pages-neutral-8)');
  });

  it('generates interactive role tokens', () => {
    const result = semanticMap(baseTokens, {});
    const roles = result['role'] as TokenMap;
    expect((roles['interactive'] as TokenLeaf).$value).toBe('var(--pages-accent-9)');
    expect((roles['focus-ring'] as TokenLeaf).$value).toBe('var(--pages-accent-8)');
  });

  it('generates status role tokens', () => {
    const result = semanticMap(baseTokens, {});
    const roles = result['role'] as TokenMap;
    expect((roles['status-success'] as TokenLeaf).$value).toBe('var(--pages-success-9)');
    expect((roles['status-danger'] as TokenLeaf).$value).toBe('var(--pages-danger-9)');
  });

  it('allows custom mapping overrides', () => {
    const result = semanticMap(baseTokens, {
      mappings: { 'surface-primary': 'neutral.2' },
    });
    const roles = result['role'] as TokenMap;
    expect((roles['surface-primary'] as TokenLeaf).$value).toBe('var(--pages-neutral-2)');
  });

  it('supports custom role names', () => {
    const tokens: TokenMap = {
      ...baseTokens,
      violet: { '9': { $value: 'oklch(55% 0.120 270)', $type: 'color' } },
    };
    const result = semanticMap(tokens, {
      mappings: { 'brand-primary': 'violet.9' },
    });
    const roles = result['role'] as TokenMap;
    expect((roles['brand-primary'] as TokenLeaf).$value).toBe('var(--pages-violet-9)');
  });

  it('preserves existing tokens', () => {
    const result = semanticMap(baseTokens, {});
    expect(result['neutral']).toBeDefined();
    expect(result['accent']).toBeDefined();
  });
});
```

- [ ] **Step 4: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All semantic-map tests pass

- [ ] **Step 5: Commit**

```
feat(#230): add semantic-map transform — role tokens for theme-portable components
```

---

### Task 7: Contrast-check and gamut-clamp transforms

**Files:**
- Create: `packages/pages-ui-tokens/src/transforms/contrast-check.ts`
- Create: `packages/pages-ui-tokens/src/transforms/gamut-clamp.ts`
- Test: `packages/pages-ui-tokens/src/transforms/validation.test.ts`
- Modify: `packages/pages-ui-tokens/src/transforms/index.ts` — register new transforms

**Interfaces:**
- Consumes: `TokenMap`, `TokenLeaf`, `isTokenLeaf` from Task 1
- Produces:
  - `contrastCheck(tokens, { minContrast, fix })` — validate APCA contrast ratios
  - `gamutClamp(tokens)` — clamp out-of-gamut OKLCH values to sRGB

- [ ] **Step 1: Write contrast-check transform**

```typescript
// packages/pages-ui-tokens/src/transforms/contrast-check.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

function oklchToApproxY(value: string): number | null {
  const match = value.match(/^oklch\((\d+\.?\d*)%/);
  if (!match) return null;
  return parseFloat(match[1]!) / 100;
}

function apcaContrast(textY: number, bgY: number): number {
  const sapc = textY > bgY
    ? Math.pow(bgY, 0.56) - Math.pow(textY, 0.57)
    : Math.pow(bgY, 0.65) - Math.pow(textY, 0.62);
  return Math.abs(sapc * 100);
}

interface ContrastViolation {
  readonly textToken: string;
  readonly bgToken: string;
  readonly contrast: number;
  readonly required: number;
}

export function contrastCheck(tokens: TokenMap, params: Record<string, unknown>): TokenMap {
  const minContrast = (params['minContrast'] as number) ?? 60;
  const fix = (params['fix'] as boolean) ?? false;

  const roles = tokens['role'] as TokenMap | undefined;
  if (!roles) return tokens;

  const textBgPairs: [string, string][] = [
    ['text-primary', 'surface-primary'],
    ['text-secondary', 'surface-primary'],
    ['text-muted', 'surface-primary'],
    ['interactive', 'surface-primary'],
  ];

  const violations: ContrastViolation[] = [];

  for (const [textRole, bgRole] of textBgPairs) {
    const textLeaf = roles[textRole] as TokenLeaf | undefined;
    const bgLeaf = roles[bgRole] as TokenLeaf | undefined;
    if (!textLeaf || !bgLeaf) continue;

    const textRef = resolveVarRef(textLeaf.$value, tokens);
    const bgRef = resolveVarRef(bgLeaf.$value, tokens);
    if (!textRef || !bgRef) continue;

    const textY = oklchToApproxY(textRef);
    const bgY = oklchToApproxY(bgRef);
    if (textY === null || bgY === null) continue;

    const contrast = apcaContrast(textY, bgY);
    if (contrast < minContrast) {
      violations.push({ textToken: textRole, bgToken: bgRole, contrast, required: minContrast });
    }
  }

  if (violations.length === 0) return tokens;

  if (!fix) {
    const details = violations.map(v =>
      `${v.textToken} on ${v.bgToken}: APCA ${v.contrast.toFixed(1)} < ${v.required}`
    ).join('\n');
    throw new Error(`Contrast violations:\n${details}`);
  }

  // fix: true — adjust lightness to meet contrast threshold
  let adjusted = { ...tokens } as Record<string, unknown>;
  for (const v of violations) {
    const textRef = (roles[v.textToken] as TokenLeaf)?.$value;
    const match = textRef?.match(/^var\(--pages-([a-z]+)-(\d+)\)$/);
    if (!match) continue;
    const [, group, step] = match;
    const g = adjusted[group!] as Record<string, unknown> | undefined;
    if (!g) continue;
    const leaf = g[step!] as TokenLeaf | undefined;
    if (!leaf) continue;
    const parsed = leaf.$value.match(/^oklch\((\d+\.?\d*)% (\d+\.?\d*) (\d+\.?\d*)\)$/);
    if (!parsed) continue;
    const bgRef = resolveVarRef((roles[v.bgToken] as TokenLeaf)?.$value ?? '', tokens);
    const bgY = bgRef ? oklchToApproxY(bgRef) : null;
    if (bgY === null) continue;
    // Binary search for minimum lightness that meets contrast
    let lo = parseFloat(parsed[1]!);
    let hi = lo > bgY * 100 ? 100 : 0;
    if (hi < lo) [lo, hi] = [hi, lo];
    for (let i = 0; i < 20; i++) {
      const mid = (lo + hi) / 2;
      const contrast = apcaContrast(mid / 100, bgY);
      if (contrast >= minContrast) hi = mid;
      else lo = mid;
    }
    const fixedL = (lo + hi) / 2;
    const fixedChroma = Math.min(parseFloat(parsed[2]!), 0.2);
    g[step!] = { $value: `oklch(${fixedL.toFixed(1)}% ${fixedChroma.toFixed(3)} ${parsed[3]})`, $type: 'color' };
    console.warn(`contrast-check fix: ${group}-${step} lightness ${parsed[1]}→${fixedL.toFixed(1)}`);
  }

  return adjusted as TokenMap;
}

function resolveVarRef(value: string, tokens: TokenMap): string | null {
  const match = value.match(/^var\(--pages-([a-z]+-\d+)\)$/);
  if (!match) return value;
  const [group, step] = match[1]!.split('-');
  const g = tokens[group!] as TokenMap | undefined;
  if (!g) return null;
  const leaf = g[step!] as TokenLeaf | undefined;
  return leaf?.$value ?? null;
}
```

- [ ] **Step 2: Write gamut-clamp transform**

```typescript
// packages/pages-ui-tokens/src/transforms/gamut-clamp.ts
import type { TokenMap, TokenLeaf } from '../types.js';
import { isTokenLeaf } from '../types.js';

function oklchInGamut(L: number, C: number, H: number): boolean {
  const l = L / 100;
  const a = C * Math.cos(H * Math.PI / 180);
  const b = C * Math.sin(H * Math.PI / 180);

  const l_ = l + 0.3963377774 * a + 0.2158037573 * b;
  const m_ = l - 0.1055613458 * a - 0.0638541728 * b;
  const s_ = l - 0.0894841775 * a - 1.2914855480 * b;

  const lr = l_ * l_ * l_;
  const mr = m_ * m_ * m_;
  const sr = s_ * s_ * s_;

  const r = +4.0767416621 * lr - 3.3077115913 * mr + 0.2309699292 * sr;
  const g = -1.2684380046 * lr + 2.6097574011 * mr - 0.3413193965 * sr;
  const bv = -0.0041960863 * lr - 0.7034186147 * mr + 1.7076147010 * sr;

  const eps = 0.001;
  return r >= -eps && r <= 1 + eps && g >= -eps && g <= 1 + eps && bv >= -eps && bv <= 1 + eps;
}

function clampToGamut(L: number, C: number, H: number): [number, number, number] {
  if (oklchInGamut(L, C, H)) return [L, C, H];

  let lo = 0;
  let hi = C;
  while (hi - lo > 0.0001) {
    const mid = (lo + hi) / 2;
    if (oklchInGamut(L, mid, H)) lo = mid;
    else hi = mid;
  }
  return [L, lo, H];
}

export function gamutClamp(tokens: TokenMap): TokenMap {
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) { result[key] = value; continue; }
    if (isTokenLeaf(value)) {
      result[key] = value.$type === 'color' ? clampLeaf(value) : value;
    } else {
      result[key] = gamutClamp(value as TokenMap);
    }
  }
  return result as TokenMap;
}

function clampLeaf(leaf: TokenLeaf): TokenLeaf {
  const match = leaf.$value.match(/^oklch\((\d+\.?\d*)% (\d+\.?\d*) (\d+\.?\d*)\)$/);
  if (!match) return leaf;
  const [L, C, H] = [parseFloat(match[1]!), parseFloat(match[2]!), parseFloat(match[3]!)];
  const [cL, cC, cH] = clampToGamut(L, C, H);
  if (cC === C) return leaf;
  return { $value: `oklch(${cL.toFixed(1)}% ${cC.toFixed(3)} ${cH})`, $type: 'color' };
}
```

- [ ] **Step 3: Register in transforms/index.ts**

```typescript
import { contrastCheck } from './contrast-check.js';
import { gamutClamp } from './gamut-clamp.js';
// Add to registerCoreTransforms():
registerTransform('contrast-check', contrastCheck);
registerTransform('gamut-clamp', gamutClamp);
```

- [ ] **Step 4: Write tests**

```typescript
// packages/pages-ui-tokens/src/transforms/validation.test.ts
import { describe, it, expect } from 'vitest';
import { contrastCheck } from './contrast-check.js';
import { gamutClamp } from './gamut-clamp.js';
import type { TokenMap, TokenLeaf } from '../types.js';

describe('contrastCheck', () => {
  const goodTokens: TokenMap = {
    neutral: {
      '1': { $value: 'oklch(8% 0.002 260)', $type: 'color' },
      '12': { $value: 'oklch(93% 0.014 260)', $type: 'color' },
    },
    accent: { '9': { $value: 'oklch(55% 0.120 245)', $type: 'color' } },
    role: {
      'text-primary': { $value: 'var(--pages-neutral-12)', $type: 'color' },
      'surface-primary': { $value: 'var(--pages-neutral-1)', $type: 'color' },
      'interactive': { $value: 'var(--pages-accent-9)', $type: 'color' },
    },
  };

  it('passes when contrast is sufficient', () => {
    expect(() => contrastCheck(goodTokens, { minContrast: 30 })).not.toThrow();
  });

  it('throws on violations when fix=false', () => {
    const lowContrastTokens: TokenMap = {
      neutral: {
        '1': { $value: 'oklch(50% 0.002 260)', $type: 'color' },
        '8': { $value: 'oklch(52% 0.009 260)', $type: 'color' },
      },
      role: {
        'text-muted': { $value: 'var(--pages-neutral-8)', $type: 'color' },
        'surface-primary': { $value: 'var(--pages-neutral-1)', $type: 'color' },
      },
    };
    expect(() => contrastCheck(lowContrastTokens, { minContrast: 60 })).toThrow(/Contrast/);
  });

  it('returns tokens unchanged when no role tokens present', () => {
    const noRoles: TokenMap = { neutral: { '1': { $value: 'oklch(8% 0.002 260)', $type: 'color' } } };
    expect(contrastCheck(noRoles, {})).toEqual(noRoles);
  });
});

describe('gamutClamp', () => {
  it('passes through in-gamut values unchanged', () => {
    const tokens: TokenMap = {
      accent: { '6': { $value: 'oklch(50.0% 0.100 245)', $type: 'color' } },
    };
    const result = gamutClamp(tokens);
    expect(((result['accent'] as TokenMap)['6'] as TokenLeaf).$value).toBe('oklch(50.0% 0.100 245)');
  });

  it('reduces chroma for out-of-gamut values', () => {
    const tokens: TokenMap = {
      accent: { '6': { $value: 'oklch(50.0% 0.500 245)', $type: 'color' } },
    };
    const result = gamutClamp(tokens);
    const chroma = parseFloat(((result['accent'] as TokenMap)['6'] as TokenLeaf).$value.match(/oklch\(\d+\.?\d*% (\d+\.?\d*)/)![1]!);
    expect(chroma).toBeLessThan(0.5);
  });

  it('preserves non-colour tokens', () => {
    const tokens: TokenMap = { spacing: { '1': { $value: '4px', $type: 'dimension' } } };
    const result = gamutClamp(tokens);
    expect(result).toEqual(tokens);
  });

  it('skips metadata keys', () => {
    const tokens: TokenMap = { $mode: { $value: 'dark', $type: 'meta' } };
    const result = gamutClamp(tokens);
    expect(result).toEqual(tokens);
  });
});
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All validation transform tests pass

- [ ] **Step 6: Commit**

```
feat(#230): add contrast-check and gamut-clamp validation transforms
```

---

### Task 8: CaseHub presets

**Files:**
- Create: `packages/pages-ui-tokens/presets/casehub-dark.json`
- Create: `packages/pages-ui-tokens/presets/casehub-light.json`
- Modify: `packages/pages-ui-tokens/src/transforms/index.ts` — register CaseHub presets
- Test: `packages/pages-ui-tokens/src/casehub-presets.test.ts`

**Interfaces:**
- Consumes: `runPipeline`, `initPresets`, `generateCSS` from Tasks 1-4
- Produces: Built-in CaseHub brand presets reproducing the Claudony/casehub.org palette

- [ ] **Step 1: Create casehub-dark.json**

```json
{
  "$name": "casehub-dark",
  "$description": "CaseHub brand dark theme — Claudony / casehub.org look",
  "$extends": "default-dark",
  "pipeline": [
    {
      "transform": "oklch-scale",
      "params": {
        "hues": {
          "violet": 270,
          "green": 160,
          "magenta": 320
        }
      }
    },
    {
      "transform": "lightness-shift",
      "params": { "offset": 10 }
    },
    {
      "transform": "chroma-curve",
      "params": {
        "curve": "gaussian",
        "neutral": 0.02
      }
    },
    {
      "transform": "semantic-hues",
      "params": {
        "success": 175,
        "warning": 100
      }
    },
    {
      "transform": "semantic-map",
      "params": {}
    },
    {
      "transform": "gamut-clamp"
    }
  ]
}
```

- [ ] **Step 2: Create casehub-light.json**

```json
{
  "$name": "casehub-light",
  "$description": "CaseHub brand light theme",
  "$extends": "default-light",
  "pipeline": [
    {
      "transform": "oklch-scale",
      "params": {
        "hues": {
          "violet": 270,
          "green": 160,
          "magenta": 320
        }
      }
    },
    {
      "transform": "chroma-curve",
      "params": {
        "curve": "gaussian",
        "neutral": 0.02
      }
    },
    {
      "transform": "semantic-hues",
      "params": {
        "success": 175,
        "warning": 100
      }
    },
    {
      "transform": "semantic-map",
      "params": {}
    },
    {
      "transform": "gamut-clamp"
    }
  ]
}
```

- [ ] **Step 3: Register CaseHub presets in transforms/index.ts**

Add preset imports and registration alongside the default presets in `initPresets()`.

- [ ] **Step 4: Write tests**

```typescript
// packages/pages-ui-tokens/src/casehub-presets.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { runPipeline } from './pipeline.js';
import { generateCSS } from './output.js';
import { initPresets } from './transforms/index.js';
import { getBuiltinPreset } from './preset-loader.js';
import type { TokenMap, TokenLeaf } from './types.js';

describe('casehub-dark preset', () => {
  beforeAll(() => { initPresets(); });

  it('resolves from built-in registry', () => {
    expect(getBuiltinPreset('casehub-dark')).toBeDefined();
  });

  it('extends default-dark', () => {
    expect(getBuiltinPreset('casehub-dark')?.$extends).toBe('default-dark');
  });

  it('generates brand hue scales (violet, green, magenta)', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-dark')!);
    expect(tokens['violet']).toBeDefined();
    expect(tokens['green']).toBeDefined();
    expect(tokens['magenta']).toBeDefined();
  });

  it('preserves parent hues alongside brand hues ($extends is additive)', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-dark')!);
    // Parent (default-dark) hues must survive child's oklch-scale
    expect(tokens['accent']).toBeDefined();
    expect(tokens['neutral']).toBeDefined();
    expect(tokens['success']).toBeDefined();
    expect(tokens['warning']).toBeDefined();
    expect(tokens['danger']).toBeDefined();
    expect(tokens['info']).toBeDefined();
    // Child brand hues
    expect(tokens['violet']).toBeDefined();
    expect(tokens['green']).toBeDefined();
    expect(tokens['magenta']).toBeDefined();
  });

  it('produces near-achromatic neutrals', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-dark')!);
    const neutral6 = ((tokens['neutral'] as TokenMap)['6'] as TokenLeaf).$value;
    const chroma = parseFloat(neutral6.match(/oklch\(\d+\.?\d*% (\d+\.?\d*)/)![1]!);
    expect(chroma).toBeLessThan(0.01);
  });

  it('uses shifted success hue (175 = teal)', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-dark')!);
    const success9 = ((tokens['success'] as TokenMap)['9'] as TokenLeaf).$value;
    expect(success9).toContain('175');
  });

  it('generates semantic role tokens', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-dark')!);
    const roles = tokens['role'] as TokenMap;
    expect(roles['surface-primary']).toBeDefined();
    expect(roles['text-primary']).toBeDefined();
    expect(roles['interactive']).toBeDefined();
  });

  it('produces valid CSS', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-dark')!);
    const css = generateCSS(tokens, 'casehub-dark');
    expect(css).toContain('.pages-theme-casehub-dark');
    expect(css).not.toContain('NaN');
    expect(css).not.toContain('undefined');
  });
});

describe('casehub-light preset', () => {
  beforeAll(() => { initPresets(); });

  it('extends default-light', () => {
    expect(getBuiltinPreset('casehub-light')?.$extends).toBe('default-light');
  });

  it('generates brand hue scales', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-light')!);
    expect(tokens['violet']).toBeDefined();
    expect(tokens['green']).toBeDefined();
  });

  it('uses light mode steps (step 1 near-white)', () => {
    const tokens = runPipeline(getBuiltinPreset('casehub-light')!);
    const neutral1 = ((tokens['neutral'] as TokenMap)['1'] as TokenLeaf).$value;
    const lightness = parseFloat(neutral1.match(/oklch\((\d+\.?\d*)%/)![1]!);
    expect(lightness).toBeGreaterThan(90);
  });
});
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All CaseHub preset tests pass

- [ ] **Step 6: Commit**

```
feat(#230): add casehub-dark and casehub-light presets — brand palette via pipeline
```

---

### Task 9: CLI + build integration

**Files:**
- Create: `packages/pages-ui-tokens/src/cli.ts`
- Create: `packages/pages-ui-tokens/src/build.ts`
- Modify: `packages/pages-ui-tokens/package.json` — add bin entry, build script
- Test: `packages/pages-ui-tokens/src/cli.test.ts`

**Interfaces:**
- Consumes: `runPipeline`, `initPresets`, `generateCSS`, `generateDTCG`, `generateDensityCSS`, `getBuiltinPreset` from Tasks 1-8
- Produces:
  - `pages-tokens build [preset]` CLI command
  - `pages-tokens validate [presets]` CLI command
  - `pages-tokens debug [preset]` CLI command
  - `build.ts` — build-only exports for programmatic use
  - Generated `dist/*.css` and `dist/*.tokens.json` files

- [ ] **Step 1: Write build.ts — build-only exports**

```typescript
// packages/pages-ui-tokens/src/build.ts
export { generateThemeCSS } from './themes.js';
export { runPipeline } from './pipeline.js';
export { generateCSS, generateDTCG, generateDensityCSS } from './output.js';
export { initPresets } from './transforms/index.js';
export { registerTransform, getTransform, listTransforms } from './registry.js';
export { registerBuiltinPreset, getBuiltinPreset, resolvePresetChain } from './preset-loader.js';
export type { TokenMap, TokenLeaf, TransformFn, TransformDef, PresetConfig } from './types.js';
export { isTokenLeaf } from './types.js';
```

- [ ] **Step 2: Write cli.ts**

```typescript
// packages/pages-ui-tokens/src/cli.ts
import { writeFileSync, mkdirSync } from 'node:fs';
import { join, resolve } from 'node:path';
import { initPresets } from './transforms/index.js';
import { runPipeline } from './pipeline.js';
import { generateCSS, generateDTCG, generateDensityCSS } from './output.js';
import { getBuiltinPreset, resolvePresetChain } from './preset-loader.js';
import type { TokenMap } from './types.js';
import { isTokenLeaf } from './types.js';
import { getTransform } from './registry.js';
import { buildInitialTokenMap } from './pipeline.js';

initPresets();

const [,, command, ...args] = process.argv;

function getAllPresetNames(): string[] {
  return ['default-light', 'default-dark', 'casehub-light', 'casehub-dark'];
}

function buildPreset(name: string, outDir: string): void {
  const preset = getBuiltinPreset(name);
  if (!preset) throw new Error(`Unknown preset: ${name}`);

  const tokens = runPipeline(preset);
  const css = generateCSS(tokens, name);
  const dtcg = generateDTCG(tokens, name);

  mkdirSync(outDir, { recursive: true });
  writeFileSync(join(outDir, `${name}.css`), css + '\n\n' + generateDensityCSS() + '\n');
  writeFileSync(join(outDir, `${name}.tokens.json`), JSON.stringify(dtcg, null, 2) + '\n');
  console.log(`  ✓ ${name}.css + ${name}.tokens.json`);
}

function debugPreset(name: string): void {
  const preset = getBuiltinPreset(name);
  if (!preset) throw new Error(`Unknown preset: ${name}`);

  const chain = resolvePresetChain(preset);

  console.log(`\nPreset: ${name}`);
  if (preset.$extends) console.log(`Extends: ${preset.$extends}`);
  console.log(`Pipeline (${chain.length} transforms):\n`);

  let tokens: TokenMap = buildInitialTokenMap();

  for (const def of chain) {
    const fn = getTransform(def.transform);
    tokens = fn(tokens, def.params ?? {});
    const count = countLeaves(tokens);
    console.log(`  [${def.transform}] → ${count} tokens`);
    if (def.params) console.log(`    params: ${JSON.stringify(def.params)}`);
  }
}

function countLeaves(tokens: TokenMap): number {
  let count = 0;
  for (const [key, value] of Object.entries(tokens)) {
    if (key.startsWith('$')) continue;
    if (isTokenLeaf(value)) count++;
    else count += countLeaves(value as TokenMap);
  }
  return count;
}

switch (command) {
  case 'build': {
    const outDir = resolve(args.find(a => a.startsWith('--out='))?.slice(6) ?? 'dist/themes');
    const presetArg = args.find(a => !a.startsWith('--'));
    const names = presetArg ? [presetArg] : getAllPresetNames();
    console.log(`Building ${names.length} preset(s)...`);
    for (const name of names) buildPreset(name, outDir);
    console.log('Done.');
    break;
  }
  case 'validate': {
    const names = args.length > 0 ? args : getAllPresetNames();
    console.log(`Validating ${names.length} preset(s)...`);
    let failed = false;
    for (const name of names) {
      try {
        const preset = getBuiltinPreset(name);
        if (!preset) throw new Error(`Unknown preset: ${name}`);
        runPipeline(preset);
        console.log(`  ✓ ${name}`);
      } catch (e) {
        console.error(`  ✗ ${name}: ${e instanceof Error ? e.message : e}`);
        failed = true;
      }
    }
    if (failed) process.exit(1);
    break;
  }
  case 'debug': {
    const name = args[0];
    if (!name) { console.error('Usage: pages-tokens debug <preset-name>'); process.exit(1); }
    debugPreset(name);
    break;
  }
  default:
    console.error('Usage: pages-tokens <build|validate|debug> [preset] [--out=dir]');
    process.exit(1);
}
```

All imports are static — no dynamic imports needed since the CLI always runs in Node.

- [ ] **Step 3: Update package.json**

Add `bin` entry and update build script:

```json
{
  "bin": {
    "pages-tokens": "dist/cli.js"
  },
  "scripts": {
    "build": "tsc -p tsconfig.build.json && node dist/cli.js build",
    "build:tokens": "node dist/cli.js build",
    "validate:tokens": "node dist/cli.js validate"
  }
}
```

Also add `"resolveJsonModule": true` to `tsconfig.json` if not already present, for JSON preset imports.

- [ ] **Step 4: Write CLI tests**

```typescript
// packages/pages-ui-tokens/src/cli.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { initPresets } from './transforms/index.js';
import { runPipeline } from './pipeline.js';
import { generateCSS, generateDTCG, generateDensityCSS } from './output.js';
import { getBuiltinPreset } from './preset-loader.js';

describe('CLI build integration', () => {
  beforeAll(() => { initPresets(); });

  it('builds all four built-in presets without errors', () => {
    for (const name of ['default-light', 'default-dark', 'casehub-light', 'casehub-dark']) {
      const preset = getBuiltinPreset(name);
      expect(preset, `preset ${name} not found`).toBeDefined();
      const tokens = runPipeline(preset!);
      const css = generateCSS(tokens, name);
      const dtcg = generateDTCG(tokens, name);
      expect(css).toContain(`.pages-theme-${name}`);
      expect(dtcg['$name']).toBe(name);
    }
  });

  it('density CSS is independent of preset', () => {
    const css = generateDensityCSS();
    expect(css).toContain('.pages-density-compact');
    expect(css).toContain('--pages-space-1: 3px;');
  });

  it('CSS output contains no NaN or undefined for any preset', () => {
    for (const name of ['default-light', 'default-dark', 'casehub-light', 'casehub-dark']) {
      const tokens = runPipeline(getBuiltinPreset(name)!);
      const css = generateCSS(tokens, name);
      expect(css, `${name} contains NaN`).not.toContain('NaN');
      expect(css, `${name} contains undefined`).not.toContain('undefined');
    }
  });
});
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All CLI tests pass

- [ ] **Step 6: Verify build produces output files**

Run: `yarn workspace @casehubio/pages-ui-tokens run build`
Expected: `dist/themes/` contains `default-light.css`, `default-dark.css`, `casehub-light.css`, `casehub-dark.css`, and corresponding `.tokens.json` files

- [ ] **Step 7: Commit**

```
feat(#230): add pages-tokens CLI and build integration — generates CSS/DTCG per preset
```

---

### Task 10: Runtime API

**Files:**
- Create: `packages/pages-ui-tokens/src/runtime.ts`
- Modify: `packages/pages-ui-tokens/src/index.ts` — export new runtime API, remove old runtime exports
- Test: `packages/pages-ui-tokens/src/runtime.test.ts`

**Interfaces:**
- Consumes: Generated CSS files from Task 9 (bundled as string imports)
- Produces:
  - `applyTheme(name: string, target?: HTMLElement): void`
  - `registerTheme(name: string, css: string): void`
  - `getTheme(target?: HTMLElement): string`
  - `listThemes(): string[]`

- [ ] **Step 1: Write failing tests for runtime API**

```typescript
// packages/pages-ui-tokens/src/runtime.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { applyTheme, registerTheme, getTheme, listThemes } from './runtime.js';

describe('runtime API', () => {
  beforeEach(() => {
    document.documentElement.innerHTML = '';
    document.documentElement.className = '';
  });

  describe('registerTheme + listThemes', () => {
    it('lists registered themes', () => {
      registerTheme('test-light', '.pages-theme-test-light { --pages-accent-1: red; }');
      expect(listThemes()).toContain('test-light');
    });

    it('registers multiple themes', () => {
      registerTheme('a', '.pages-theme-a {}');
      registerTheme('b', '.pages-theme-b {}');
      const themes = listThemes();
      expect(themes).toContain('a');
      expect(themes).toContain('b');
    });
  });

  describe('applyTheme', () => {
    it('sets theme class on document.documentElement by default', () => {
      registerTheme('dark', '.pages-theme-dark { --pages-accent-1: blue; }');
      applyTheme('dark');
      expect(document.documentElement.classList.contains('pages-theme-dark')).toBe(true);
    });

    it('injects style element with data-pages-theme attribute', () => {
      registerTheme('dark', '.pages-theme-dark {}');
      applyTheme('dark');
      const style = document.querySelector('style[data-pages-theme]');
      expect(style).not.toBeNull();
    });

    it('removes previous theme class when switching', () => {
      registerTheme('light', '.pages-theme-light {}');
      registerTheme('dark', '.pages-theme-dark {}');
      applyTheme('light');
      applyTheme('dark');
      expect(document.documentElement.classList.contains('pages-theme-light')).toBe(false);
      expect(document.documentElement.classList.contains('pages-theme-dark')).toBe(true);
    });

    it('replaces existing style element on reapply', () => {
      registerTheme('dark', '.pages-theme-dark {}');
      applyTheme('dark');
      applyTheme('dark');
      const styles = document.querySelectorAll('style[data-pages-theme]');
      expect(styles.length).toBe(1);
    });

    it('applies to specific target element', () => {
      const el = document.createElement('div');
      document.body.appendChild(el);
      registerTheme('dark', '.pages-theme-dark {}');
      applyTheme('dark', el);
      expect(el.classList.contains('pages-theme-dark')).toBe(true);
      expect(document.documentElement.classList.contains('pages-theme-dark')).toBe(false);
    });

    it('throws on unknown theme', () => {
      expect(() => applyTheme('nonexistent')).toThrow(/Unknown theme/);
    });
  });

  describe('getTheme', () => {
    it('returns current theme name', () => {
      registerTheme('dark', '.pages-theme-dark {}');
      applyTheme('dark');
      expect(getTheme()).toBe('dark');
    });

    it('returns empty string when no theme applied', () => {
      expect(getTheme()).toBe('');
    });

    it('returns theme for specific target', () => {
      const el = document.createElement('div');
      registerTheme('dark', '.pages-theme-dark {}');
      applyTheme('dark', el);
      expect(getTheme(el)).toBe('dark');
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: FAIL — module './runtime.js' not found

- [ ] **Step 3: Write runtime.ts**

```typescript
// packages/pages-ui-tokens/src/runtime.ts

const themeRegistry = new Map<string, string>();
const appliedThemes = new WeakMap<Element, string>();

export function registerTheme(name: string, css: string): void {
  themeRegistry.set(name, css);
}

export function applyTheme(name: string, target: HTMLElement = document.documentElement): void {
  const css = themeRegistry.get(name);
  if (!css) throw new Error(`Unknown theme: "${name}". Available: ${listThemes().join(', ')}`);

  const currentTheme = appliedThemes.get(target);
  if (currentTheme) {
    target.classList.remove(`pages-theme-${currentTheme}`);
  }

  const root = target === document.documentElement ? document.head : target;
  const existing = root.querySelector('style[data-pages-theme]');
  if (existing) existing.remove();

  const style = document.createElement('style');
  style.setAttribute('data-pages-theme', name);
  style.textContent = css;
  root.prepend(style);

  target.classList.add(`pages-theme-${name}`);
  appliedThemes.set(target, name);
}

export function getTheme(target: HTMLElement = document.documentElement): string {
  return appliedThemes.get(target) ?? '';
}

export function listThemes(): string[] {
  return [...themeRegistry.keys()];
}
```

- [ ] **Step 4: Update index.ts — new runtime exports**

Replace the old runtime exports with the new API:

```typescript
// packages/pages-ui-tokens/src/index.ts
export { generateScale } from './colours.js';
export { SPACING_SCALE, TYPOGRAPHY, MOTION, RADIUS, ELEVATION_LIGHT, ELEVATION_DARK, DENSITY_COMPACT_OVERRIDES } from './tokens.js';
export { applyTheme, registerTheme, getTheme, listThemes } from './runtime.js';

// Build-only — kept for backward compat during migration, removed in next version
export { generateThemeCSS, injectTheme, applyThemeMode, DEFAULT_THEME, type ThemeConfig } from './themes.js';
```

Note: The old exports (`generateThemeCSS`, `injectTheme`, `applyThemeMode`, `DEFAULT_THEME`, `ThemeConfig`) are kept temporarily so consumers can migrate incrementally. They will be removed once all consumers have migrated (Task 11).

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: All runtime tests pass, all existing tests still pass

- [ ] **Step 6: Commit**

```
feat(#230): add runtime API — applyTheme, registerTheme, getTheme, listThemes
```

---

### Task 11: Consumer migration — pages-runtime + examples

**Files:**
- Modify: `packages/pages-runtime/src/site.ts` — replace ThemeConfig/injectTheme/applyThemeMode with applyTheme
- Modify: `examples/src/casehub-entry.ts` — simplified theme setup
- Modify: `examples/fixtures/table-spanning-test.html` — update inline theme calls
- Test: existing pages-runtime tests must pass

**Interfaces:**
- Consumes: `applyTheme`, `registerTheme` from Task 10
- Produces:
  - `SiteOptions.themeName?: string` (replaces `themeConfig?: ThemeConfig`)
  - `LiveSite.setTheme(mode)` maps mode to theme name

- [ ] **Step 1: Migrate pages-runtime/site.ts**

Use `ide_edit_member` and `ide_replace_member` for structural edits. Key changes:

**Import changes (line ~30-31):**
Replace:
```typescript
import type {ThemeConfig} from "@casehubio/pages-ui-tokens";
import {applyThemeMode, DEFAULT_THEME, injectTheme} from "@casehubio/pages-ui-tokens";
```
With:
```typescript
import {applyTheme} from "@casehubio/pages-ui-tokens";
```

**SiteOptions interface (line ~129):**
Replace `themeConfig?: ThemeConfig` with `themeName?: string`.

**loadSite function (line ~147):**
Replace:
```typescript
applyThemeMode(target, isDark ? "dark" : "light");
```
With:
```typescript
const themeBaseName = options?.themeName ?? 'default';
applyTheme(`${themeBaseName}-${isDark ? 'dark' : 'light'}`, target);
```

**Theme injection (line ~991):**
Remove:
```typescript
injectTheme(options?.themeConfig ?? DEFAULT_THEME, target);
```
(The `applyTheme` call above already handles injection.)

**LiveSite.setTheme (around line ~1035):**
Replace:
```typescript
setTheme(mode: "light" | "dark"): void {
  applyThemeMode(target, mode);
  ...
}
```
With:
```typescript
setTheme(mode: "light" | "dark"): void {
  const currentThemeName = options?.themeName ?? 'default';
  applyTheme(`${currentThemeName}-${mode}`, target);
  const echartsThemeName = mode === "dark" ? "dark" : "";
  for (const [, entry] of registry) {
    const vizEl = entry.vizElement;
    if (vizEl && "buildOption" in vizEl && "theme" in vizEl) {
      (vizEl as unknown as { theme: string }).theme = echartsThemeName;
    }
  }
}
```

- [ ] **Step 2: Migrate examples/src/casehub-entry.ts**

Replace:
```typescript
import { injectTheme, applyThemeMode, DEFAULT_THEME } from "@casehubio/pages-ui-tokens";

injectTheme(DEFAULT_THEME);
applyThemeMode(document.documentElement, "light");

export { loadSite, injectTheme, applyThemeMode, DEFAULT_THEME };
```

With:
```typescript
import { applyTheme } from "@casehubio/pages-ui-tokens";

applyTheme('default-light');

export { loadSite, applyTheme };
```

- [ ] **Step 3: Migrate examples/fixtures/table-spanning-test.html**

Replace:
```javascript
casehubPages.injectTheme(casehubPages.DEFAULT_THEME);
casehubPages.applyThemeMode(document.documentElement, 'light');
```

With:
```javascript
casehubPages.applyTheme('default-light');
```

- [ ] **Step 4: Remove old exports from index.ts**

Now that consumers are migrated, remove the backward compat exports:

```typescript
// packages/pages-ui-tokens/src/index.ts
export { generateScale } from './colours.js';
export { SPACING_SCALE, TYPOGRAPHY, MOTION, RADIUS, ELEVATION_LIGHT, ELEVATION_DARK, DENSITY_COMPACT_OVERRIDES } from './tokens.js';
export { applyTheme, registerTheme, getTheme, listThemes } from './runtime.js';
```

The old `generateThemeCSS`, `injectTheme`, `applyThemeMode` are now only available via `import from '@casehubio/pages-ui-tokens/build'`.

- [ ] **Step 5: Auto-register built-in themes**

The runtime needs built-in themes to be auto-registered at import time. The build step (Task 9) generates CSS files for each preset. At import time, these pre-built CSS strings are registered — no pipeline execution in the browser.

Add a build script (`scripts/bundle-themes.ts` or inline in the CLI build step) that writes a generated module:

```typescript
// packages/pages-ui-tokens/src/builtin-themes.generated.ts  (generated by build)
// DO NOT EDIT — generated by pages-tokens build
export const BUILTIN_THEMES: Record<string, string> = {
  'default-light': `.pages-theme-default-light {\n  --pages-neutral-1: ...\n}\n\n.pages-density-compact { ... }`,
  'default-dark': `...`,
  'casehub-light': `...`,
  'casehub-dark': `...`,
};
```

Then the init module simply registers the pre-built strings:

```typescript
// packages/pages-ui-tokens/src/init.ts
import { registerTheme } from './runtime.js';
import { BUILTIN_THEMES } from './builtin-themes.generated.js';

for (const [name, css] of Object.entries(BUILTIN_THEMES)) {
  registerTheme(name, css);
}
```

Add to index.ts as a side-effect import:
```typescript
import './init.js';
```

The build script in `package.json` must run `pages-tokens build` before `tsc` to generate the CSS, then a small script writes `builtin-themes.generated.ts` from the generated CSS files. This ensures zero pipeline execution at import time — only string registration.

Update the `build` script in `package.json`:
```json
"build": "node scripts/generate-builtin-themes.mjs && tsc -p tsconfig.build.json"
```

The `scripts/generate-builtin-themes.mjs` script reads `dist/themes/*.css` and writes `src/builtin-themes.generated.ts`.

- [ ] **Step 6: Run full test suite**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Run: `yarn typecheck`
Expected: All tests pass, type checking passes

- [ ] **Step 7: Build and verify**

Run: `yarn build:packages`
Expected: All packages build successfully

- [ ] **Step 8: Commit**

```
feat(#230): migrate consumers to applyTheme API — pages-runtime, examples

BREAKING CHANGE: generateThemeCSS, injectTheme, applyThemeMode removed from
runtime exports. Use applyTheme('theme-name') instead.
SiteOptions.themeConfig replaced by SiteOptions.themeName.
```

---

### Task 12: Theme picker component

**Files:**
- Create: `packages/pages-ui-tokens/src/theme-picker.ts`
- Modify: `packages/pages-ui-tokens/package.json` — add `lit` dependency
- Modify: `packages/pages-ui-tokens/src/index.ts` — export PagesThemePickerElement
- Test: `packages/pages-ui-tokens/src/theme-picker.test.ts`

**Interfaces:**
- Consumes: `applyTheme`, `getTheme`, `listThemes` from Task 10
- Produces:
  - `PagesThemePickerElement` — LitElement custom element `<pages-theme-picker>`
  - Properties: `target: HTMLElement`, `compact: boolean`

- [ ] **Step 1: Add lit dependency**

Update `packages/pages-ui-tokens/package.json`:
```json
{
  "dependencies": {
    "lit": "^3.3.3"
  }
}
```

Run: `yarn install`

- [ ] **Step 2: Write failing tests**

```typescript
// packages/pages-ui-tokens/src/theme-picker.test.ts
import { describe, it, expect, beforeEach, beforeAll } from 'vitest';
import { registerTheme, applyTheme } from './runtime.js';

// Register test themes before importing component
beforeAll(async () => {
  registerTheme('default-light', '.pages-theme-default-light {}');
  registerTheme('default-dark', '.pages-theme-default-dark {}');
  registerTheme('casehub-light', '.pages-theme-casehub-light {}');
  registerTheme('casehub-dark', '.pages-theme-casehub-dark {}');
  applyTheme('default-dark');
  await import('./theme-picker.js');
});

describe('pages-theme-picker', () => {
  let picker: HTMLElement;

  beforeEach(async () => {
    document.body.innerHTML = '';
    picker = document.createElement('pages-theme-picker');
    document.body.appendChild(picker);
    await (picker as any).updateComplete;
  });

  it('is a defined custom element', () => {
    expect(customElements.get('pages-theme-picker')).toBeDefined();
  });

  it('renders a shadow root', () => {
    expect(picker.shadowRoot).not.toBeNull();
  });

  it('groups themes by family', () => {
    const select = picker.shadowRoot?.querySelector('select');
    expect(select).not.toBeNull();
    const options = Array.from(select?.querySelectorAll('option') ?? []);
    const labels = options.map(o => o.textContent);
    expect(labels).toContain('Default');
    expect(labels).toContain('CaseHub');
  });

  it('has light/dark mode toggle buttons', () => {
    const buttons = picker.shadowRoot?.querySelectorAll('button');
    expect(buttons?.length).toBeGreaterThanOrEqual(2);
  });
});
```

- [ ] **Step 3: Write theme-picker.ts**

```typescript
// packages/pages-ui-tokens/src/theme-picker.ts
import { LitElement, html, css } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { applyTheme, getTheme, listThemes } from './runtime.js';

interface ThemeFamily {
  readonly name: string;
  readonly displayName: string;
  readonly hasLight: boolean;
  readonly hasDark: boolean;
}

function extractFamilies(themes: string[]): ThemeFamily[] {
  const familyMap = new Map<string, { hasLight: boolean; hasDark: boolean }>();
  for (const theme of themes) {
    const lightMatch = theme.match(/^(.+)-light$/);
    const darkMatch = theme.match(/^(.+)-dark$/);
    const family = lightMatch?.[1] ?? darkMatch?.[1] ?? theme;
    const entry = familyMap.get(family) ?? { hasLight: false, hasDark: false };
    if (lightMatch) entry.hasLight = true;
    if (darkMatch) entry.hasDark = true;
    if (!lightMatch && !darkMatch) { entry.hasLight = true; entry.hasDark = true; }
    familyMap.set(family, entry);
  }
  return [...familyMap.entries()].map(([name, v]) => ({
    name,
    displayName: name.split('-').map(w => w[0]?.toUpperCase() + w.slice(1)).join(' '),
    ...v,
  }));
}

function parseCurrentTheme(current: string): { family: string; mode: 'light' | 'dark' } {
  const lightMatch = current.match(/^(.+)-light$/);
  const darkMatch = current.match(/^(.+)-dark$/);
  return {
    family: lightMatch?.[1] ?? darkMatch?.[1] ?? current,
    mode: darkMatch ? 'dark' : 'light',
  };
}

@customElement('pages-theme-picker')
export class PagesThemePickerElement extends LitElement {
  static override styles = css`
    :host { display: inline-flex; align-items: center; gap: 8px; }
    select {
      background: var(--pages-surface-secondary, #222);
      color: var(--pages-text-secondary, #ccc);
      border: 1px solid var(--pages-border-default, #444);
      border-radius: var(--pages-radius-sm, 4px);
      padding: 4px 8px;
      font: inherit;
    }
    .mode-toggle { display: inline-flex; gap: 0; }
    .mode-toggle button {
      background: var(--pages-surface-secondary, #222);
      color: var(--pages-text-secondary, #ccc);
      border: 1px solid var(--pages-border-default, #444);
      padding: 4px 12px;
      cursor: pointer;
      font: inherit;
    }
    .mode-toggle button:first-child { border-radius: var(--pages-radius-sm, 4px) 0 0 var(--pages-radius-sm, 4px); }
    .mode-toggle button:last-child { border-radius: 0 var(--pages-radius-sm, 4px) var(--pages-radius-sm, 4px) 0; border-left: none; }
    .mode-toggle button[aria-pressed="true"] {
      background: var(--pages-interactive, #4a9eff);
      color: var(--pages-surface-primary, #111);
    }
  `;

  @property({ attribute: false }) target: HTMLElement = document.documentElement;
  @property({ type: Boolean }) compact = false;

  @state() private _family = '';
  @state() private _mode: 'light' | 'dark' = 'dark';
  @state() private _families: ThemeFamily[] = [];

  override connectedCallback(): void {
    super.connectedCallback();
    this._families = extractFamilies(listThemes());
    const current = getTheme(this.target);
    if (current) {
      const parsed = parseCurrentTheme(current);
      this._family = parsed.family;
      this._mode = parsed.mode;
    } else if (this._families.length > 0) {
      this._family = this._families[0]!.name;
    }
  }

  override render() {
    if (this.compact) return this._renderCompact();
    return html`
      <select @change=${this._onFamilyChange}>
        ${this._families.map(f => html`
          <option value=${f.name} ?selected=${f.name === this._family}>${f.displayName}</option>
        `)}
      </select>
      <div class="mode-toggle">
        <button aria-pressed=${this._mode === 'light'} @click=${() => this._setMode('light')}>Light</button>
        <button aria-pressed=${this._mode === 'dark'} @click=${() => this._setMode('dark')}>Dark</button>
      </div>
    `;
  }

  private _renderCompact() {
    return html`
      <div class="mode-toggle">
        <button aria-pressed=${this._mode === 'light'} @click=${() => this._setMode('light')} title="Light mode">☀</button>
        <button aria-pressed=${this._mode === 'dark'} @click=${() => this._setMode('dark')} title="Dark mode">☾</button>
      </div>
    `;
  }

  private _onFamilyChange(e: Event): void {
    this._family = (e.target as HTMLSelectElement).value;
    this._apply();
  }

  private _setMode(mode: 'light' | 'dark'): void {
    this._mode = mode;
    this._apply();
  }

  private _apply(): void {
    const themeName = `${this._family}-${this._mode}`;
    const available = listThemes();
    if (available.includes(themeName)) {
      applyTheme(themeName, this.target);
    }
  }
}
```

- [ ] **Step 4: Export from index.ts**

```typescript
export { PagesThemePickerElement } from './theme-picker.js';
```

- [ ] **Step 5: Run tests**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Expected: Theme picker tests pass (may need lit SSR shims in vitest config — add `@lit/reactive-element` to vitest setup if needed)

- [ ] **Step 6: Commit**

```
feat(#230): add <pages-theme-picker> LitElement — family dropdown + mode toggle
```

---

### Task 13: Protocol update and cleanup

**Files:**
- Modify: `packages/pages-ui-tokens/package.json` — version bump to 0.3.0
- Modify: `docs/protocols/casehub/css-design-tokens.md` — add semantic role tokens
- Modify: `packages/pages-ui-tokens/src/index.ts` — final cleanup of old exports
- Test: full test suite + typecheck + build

**Interfaces:**
- Consumes: everything from Tasks 1-12
- Produces: Updated protocol, clean package version

- [ ] **Step 1: Bump version to 0.3.0**

Update `packages/pages-ui-tokens/package.json`:
```json
{ "version": "0.3.0" }
```

- [ ] **Step 2: Update css-design-tokens.md protocol**

Add new sections for semantic role tokens and theme class naming:

```markdown
## Rule 4 — Semantic role tokens (tier 2)

Components consume role tokens instead of primitive step numbers for
theme-portable styling. Role tokens use `var()` references to primitives:

| Category | Pattern | Examples |
|----------|---------|----------|
| Surface | `--pages-surface-{role}` | primary, secondary, tertiary, hover, selected |
| Border | `--pages-border-{role}` | subtle, default, strong |
| Text | `--pages-text-{role}` | primary, secondary, muted, disabled |
| Interactive | `--pages-interactive[-{state}]` | (base), hover, active |
| Focus | `--pages-focus-ring` | — |
| Status | `--pages-status-{severity}` | success, warning, danger, info |

Role tokens are generated by the `semantic-map` pipeline transform.
Primitives (`--pages-{hue}-{1-12}`) remain stable and valid. Migration
to role tokens is incremental — not blocking.

## Rule 5 — Theme class naming

Theme CSS is scoped under `.pages-theme-{$name}`:
`pages-theme-default-light`, `.pages-theme-casehub-dark`, etc.

The old `.pages-theme-light` / `.pages-theme-dark` class names are replaced
by the named equivalents. `applyTheme(name)` sets the class.
```

- [ ] **Step 3: Final cleanup — remove old themes.ts runtime exports if not yet removed**

Verify that `themes.ts` exports (`generateThemeCSS`, `injectTheme`, `applyThemeMode`, `DEFAULT_THEME`, `ThemeConfig`) are no longer in `index.ts`. If they remain, remove them now. Keep `themes.ts` itself — it's still importable directly for build-only use.

- [ ] **Step 4: Run full verification**

Run: `yarn workspace @casehubio/pages-ui-tokens run test -- --reporter=verbose`
Run: `yarn typecheck`
Run: `yarn build`
Expected: Everything passes, builds successfully

- [ ] **Step 5: Commit**

```
feat(#230): bump pages-ui-tokens to 0.3.0, update css-design-tokens protocol

Refs #230
```
