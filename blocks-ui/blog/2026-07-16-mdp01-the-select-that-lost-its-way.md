---
layout: post
title: "The select that lost its way"
date: 2026-07-16
type: phase-update
entry_type: note
subtype: diary
projects: [CaseHub Blocks UI]
tags: [channel-activity, shadow-dom, web-components, claudony]
---

Claudony needed two extension points on channel-activity — a way to customise
message body rendering, and a compact channel selector for narrow panels. Both
turned out to be straightforward implementations with one genuinely surprising
detour.

## renderContent: the callback that was already there in spirit

`channel-message` already had `formatSender` — a callback that lets consumers
replace the default sender display. `renderContent` follows the same shape: an
optional function that receives the full `QhorusMessage` and returns either a
`TemplateResult` (replacing the markdown body) or `undefined` (falling back to
default rendering). One line in the render method:

```typescript
<div class="content">
  ${this.renderContent?.(m) ?? unsafeHTML(renderMarkdown(m.content))}
</div>
```

The interesting part wasn't the property itself — it was tracing the passthrough.
`channel-feed` and `channel-thread` both create `<channel-message>` elements
internally, so consumers using those containers couldn't reach the individual
messages. Neither `renderContent` nor `formatSender` had passthrough properties
on the parent components. We added both.

## The dropdown that appeared in the wrong place

Channel-nav gained three properties: `layout` (sidebar or dropdown), `showCreate`
and `showDelete` toggles, and a `messageCounts` map for badge display. The sidebar
mode was mechanical — conditional rendering around the existing create button and
delete buttons, plus a count badge chip.

The dropdown mode started as a native `<select>`. Clean, accessible, keyboard-
navigable out of the box. It rendered correctly in isolation. Then we embedded it
in the showcase — inside the shell's shadow root, inside the page component's
shadow root, inside channel-nav's shadow root, with a scrolled `overflow: auto`
ancestor — and the popup appeared somewhere in the middle of the page, nowhere
near the trigger.

The browser calculates native `<select>` popup positions relative to the viewport
but miscalculates when the element sits inside multiple nested shadow roots
combined with a scrolled container. The offset reflects the unscrolled document
position rather than the actual visual position.

We replaced it with a custom dropdown — a button trigger with an absolutely-
positioned panel using `position: relative` on the wrapper. Shadow DOM resolves
`position: absolute` correctly within a single shadow root, which is exactly
where the panel lives. Added ARIA attributes and keyboard handling to match
what the native `<select>` gave us for free.

The gotcha went into the garden. Anyone building compact selectors inside Web
Component shell architectures will hit the same thing — it only manifests with
the specific combination of nested shadow roots plus scrolled containers, which
is exactly the pattern every component showcase uses.
