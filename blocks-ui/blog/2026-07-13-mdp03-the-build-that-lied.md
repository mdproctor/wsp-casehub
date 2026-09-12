# The Build That Lied

The build broke and the error messages pointed at the wrong code.

Every component in blocks-ui reported the same thing: `Module '"@casehubio/blocks-ui-core"' has no exported member 'WorkIdentity'`. The fix seemed obvious — update the imports. Except WorkIdentity was still exported. The re-export chain was intact. The barrel files were correct.

The actual problem was three levels upstream. `blocks-ui-core` itself wouldn't compile — `exactOptionalPropertyTypes` violations in `fetch-source.ts` and `data-source-mixin.ts` meant TypeScript couldn't generate declaration files. With project references, downstream packages read from `.d.ts` outputs, not source. No declarations, no exports. 343 errors, all pointing at consumers, none pointing at the producer.

The tell was in the `dist/types/` directory: `.d.ts.map` files existed but the `.d.ts` files themselves were missing. A previous successful build left the maps; the current broken build couldn't overwrite the declarations because it couldn't produce them. That's the diagnostic — if you see map files without their declaration counterparts, the upstream package failed silently.

Three component tsconfigs were also missing `experimentalDecorators` and `useDefineForClassFields` — case-timeline, channel-activity, and trust-score-panel. Added after the base config was established, never given the Lit decorator flags. Another 117 errors that looked like decorator incompatibility but were really just missing config.

The batch also wired up audit-trail-viewer's row expansion. pages#172 shipped the `getRowDetail` callback on pages-table, so the fix was straightforward: pass a `getRowDetail` function that maps TypedRow back to the LedgerEntry and renders the detail inline, set `detailMode="single"`, bind `expandedDetailKeys`. The old code rendered detail sections as children of the table element, which pages-table silently ignored.

And #52 — the chat-app module's permanent home — landed in the obvious place: `examples/src/pages/channel-activity-page.ts`, alongside every other component demo. Not a standalone repo, not a connectors submodule. A demo page.
