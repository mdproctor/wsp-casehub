---
layout: post
title: "The CSS that looked like it worked"
date: 2026-07-17
type: phase-update
entry_type: note
subtype: diary
projects: [blocks-ui]
tags: [css, web-components, positioning]
---

The emoji picker in `channel-reaction-bar` overflows the viewport on narrow screens. The component positions it with `position: absolute; bottom: 100%; left: 0` — always above, always left-anchored. On a 375px phone screen, a 353px-wide picker has nowhere to go.

The interesting part: the CSS already had a `.flip` class.

```css
.picker-popover.flip {
  bottom: auto;
  top: 100%;
  margin-bottom: 0;
  margin-top: var(--pages-space-1, 4px);
}
```

Correct rules. Right properties. It would open downward instead of upward — exactly what you'd want when there's no space above. But nothing in the component ever added the class. The CSS was written anticipating the need, then the JS was never implemented to toggle it.

This is a specific kind of bug: the fix looks like it's already in place. You read the stylesheet, see `.flip`, think "vertical repositioning is handled," and look elsewhere for the overflow cause. The symptom — "picker overflows" — suggests missing CSS, not dead CSS with missing JS wiring. Investigation naturally targets the wrong layer.

The actual fix is a `_computePickerPosition()` method that runs before the picker opens. It measures the container's bounding rect (the container is always in the DOM, unlike the conditionally-rendered picker) and sets two boolean states: flip vertically if less than 400px above, align right if the picker width would overflow the viewport edge. The existing `.flip` class finally gets applied, and a new `.align-right` class handles the horizontal case.

The general pattern is worth watching for in any web component with popover positioning — CSS-only fallbacks that look complete but silently lack the JS to activate them.
