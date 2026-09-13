---
title: "Sparklines and the Mixin Question"
date: 2026-07-10
project: casehub-blocks-ui
branch: issue-45-trust-score-trend-sparkline
issue: 45
type: diary
---

The trust-score-panel had a placeholder where the trend line should be. The issue was straightforward — replace it with an SVG sparkline fed by the pages-data pipeline. What made it interesting was the architectural question underneath: should the second data source be an ad-hoc adapter on the panel, or a mixin that sets a direction?

I started with the ad-hoc approach as the recommendation. It's the minimal change — one class field, wire it up, done. But the question "does this set a direction?" is the right one to ask. The pattern — primary data source plus time-series trend — shows up in kpi-metric-row (static sparklines today), SLA indicators (breach trends), anything that shows a current value with history. A mixin standardises it once. Pre-release is when you establish these conventions.

The design review earned its keep. Round 1 raised 11 issues, several genuinely important: the extraction function was using `TypedRow.number()` which throws on NULL cells (a single malformed row would crash the component), and the sparkline auto-scaled Y-axis in a way that exaggerated minor score fluctuations. Round 3 caught a subtler problem — a single cache field for the precedence logic creates a race when both `trendData` and `trendSource` are active. Two separate cache fields, slicing in the getter. Four rounds, $14, everything resolved.

The shared `renderSparkline` also fixed a pre-existing gradient ID collision in kpi-metric-row — the hardcoded `id="spark-fill"` meant only the first sparkline on a page got its gradient fill. The kind of bug that sits there quietly until someone renders two cards side by side.
