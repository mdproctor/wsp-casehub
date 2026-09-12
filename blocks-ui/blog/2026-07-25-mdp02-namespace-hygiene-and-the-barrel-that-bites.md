---
layout: post
title: "Namespace Hygiene and the Barrel That Bites"
date: 2026-07-25
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
series: issue-94-blocks-prefix-and-cleanup
---

# CaseHub Blocks UI — Namespace Hygiene and the Barrel That Bites

**Date:** 2026-07-25
**Type:** phase-update

Three issues closed on one branch, each touching a different layer of the same problem: platform components need clear ownership boundaries.

## The barrel that bites

The pages-modal crash (#88) looked like a bundler aliasing problem — esbuild resolving the same module through two paths. The garden had the root cause already (GE-20260720-96fab8): pages-primitives' barrel `index.ts` re-exports both a11y mixins and a modal component that calls `customElements.define()` at module scope. Every consumer that imports a mixin through the barrel triggers the modal registration as an unwanted side effect.

The fix on the pages side was already in — sub-path exports and a `sideEffects` array in `package.json`. What blocks-ui still needed: narrowing 17 imports from `'@casehubio/pages-primitives'` to `'@casehubio/pages-primitives/a11y'`. None of them use the modal. The barrel coupling was pure accident.

## The buttons that could migrate and the textarea that couldn't

Channel-activity had 19 native `<button>` elements across 8 files. Swapping to `<pages-button>` was mechanical — ghost variant for most, slot content preserved, CSS classes kept for layout. The tests query by class name, so they passed without changes.

The `<textarea>` and `<select>` in channel-input stayed native. Both already use `--pages-*` tokens for visual consistency. The problem is channel-input's `@query('textarea')` decorator — it reaches directly into the DOM for auto-resize and value manipulation. Wrapping it in `<pages-textarea>` puts the native element behind a shadow DOM boundary where `@query` can't reach it. The imperative API would break, and the visual gain is zero since the tokens are already applied.

## 59 tags, one prefix

The platform convention from casehub-pages#233: `pages-` for pages components, `blocks-` for blocks-ui. Two were already prefixed (`blocks-timeline`, `blocks-confirm-dialog`). The remaining 59 needed the `blocks-` prefix on their `@customElement` decorator, every HTML template reference, every querySelector in tests, and every HTMLElementTagNameMap entry. 136 files changed, and the replacement count was exactly symmetric — 697 insertions, 697 deletions.

The safe replacement pattern: search for `'old-tag'` with surrounding quotes. Import paths like `@casehubio/blocks-ui-old-tag` contain the old tag name as a substring, but never bounded by single quotes at the tag boundary. The quotes act as a natural delimiter that prevents false positives on package names.
