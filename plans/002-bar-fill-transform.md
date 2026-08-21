# 002 — Animate performance bar fills with transform:scaleX instead of width

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: MEDIUM
- **Category**: Performance
- **Estimated scope**: 1 file (`2026/index.html`), 2 CSS rules + 4 JS call sites

## Problem

The two "performance bar" components — the transposed compare-sheet `체감 비교` grid (`.feel-bar-fill`) and the string-detail `성능` infographic (`.perf-bar-fill`) — both animate the layout-triggering `width` property over 1 second, instead of a compositor-only `transform`.

Current code:

```css
/* 2026/index.html:427 */
.feel-cell .feel-bar-fill{height:100%;border-radius:999px;width:0;transition:width 1s var(--ease);}

/* 2026/index.html:436 */
.perf-bar-fill{height:100%;border-radius:999px;background:#3fae2a;width:0;transition:width 1s var(--ease);}
```

The fill divs are created with a `data-pct` attribute and their `style.width` is set by JS after a delay, at 4 call sites:

```js
// 2026/index.html:2252 (openSheet, string detail perf bars)
sheet.querySelectorAll('.perf-bar-fill').forEach(el=>{ el.style.width = el.dataset.pct+'%'; });

// 2026/index.html:2541 (openCompareSheet, feel-compare bars)
sheet.querySelectorAll('.feel-bar-fill').forEach(el=>{ el.style.width = el.dataset.pct+'%'; });

// 2026/index.html:2795 (finderShowDetail, string detail perf bars)
sheet.querySelectorAll('.perf-bar-fill').forEach(el=>{ el.style.width = el.dataset.pct+'%'; });

// 2026/index.html:3083 (stringFinderShowDetail, string detail perf bars)
sheet.querySelectorAll('.perf-bar-fill').forEach(el=>{ el.style.width = el.dataset.pct+'%'; });
```

The markup that creates each fill div (unchanged by this plan, shown for context):

```js
// 2026/index.html:2441 — perf-bar-fill markup, inline background + a per-row transition-delay
`<div class="perf-row"><div class="perf-label">${label}</div><div class="perf-bar-track"><div class="perf-bar-fill" data-pct="${pct}" style="background:${color};transition-delay:${i*70}ms"></div></div><div class="perf-val">${val}</div></div>`

// 2026/index.html:2452 — feel-bar-fill markup
`<div class="feel-cell"><div class="feel-bar-track"><div class="feel-bar-fill" data-pct="${pct}" style="background:${CMP_COLORS[i]}"></div></div></div>`
```

The compare sheet can render up to 5 metrics × 3 products = 15 `.feel-bar-fill` elements animating simultaneously, each forcing its own layout pass. `width`/`height`/`margin`/`padding`/`top`/`left` all trigger layout+paint+composite; `transform` and `opacity` are compositor-only.

## Target

Switch both fill elements to start at `transform:scaleX(0)` with `transform-origin:left`, transition `transform` instead of `width`, and set `style.width:100%` permanently (so the element's box always spans the full track — only its visual scale changes). The parent `.feel-bar-track`/`.perf-bar-track` already has `overflow:hidden`, so a scaled child never visually escapes it.

```css
/* target: 2026/index.html:427 */
.feel-cell .feel-bar-fill{height:100%;width:100%;border-radius:999px;transform:scaleX(0);transform-origin:left;transition:transform 1s var(--ease);}

/* target: 2026/index.html:436 */
.perf-bar-fill{height:100%;width:100%;border-radius:999px;background:#3fae2a;transform:scaleX(0);transform-origin:left;transition:transform 1s var(--ease);}
```

```js
// target: all 4 call sites — replace .style.width with .style.transform
sheet.querySelectorAll('.perf-bar-fill').forEach(el=>{ el.style.transform = 'scaleX(' + (el.dataset.pct/100) + ')'; });
sheet.querySelectorAll('.feel-bar-fill').forEach(el=>{ el.style.transform = 'scaleX(' + (el.dataset.pct/100) + ')'; });
```

## Repo conventions to follow

- `data-pct` stays exactly as-is — it is still read the same way, only the property it drives changes.
- The per-row `style="background:${color};transition-delay:${i*70}ms"` inline style in the markup at line 2441 is untouched by this plan; `transition-delay` still works identically when the transitioned property changes from `width` to `transform`.
- Match existing quoting/formatting style (single quotes for JS strings, no semicolon changes beyond what's shown).

## Steps

1. In `2026/index.html:427`, replace `.feel-cell .feel-bar-fill{height:100%;border-radius:999px;width:0;transition:width 1s var(--ease);}` with the target CSS shown above.
2. In `2026/index.html:436`, replace `.perf-bar-fill{height:100%;border-radius:999px;background:#3fae2a;width:0;transition:width 1s var(--ease);}` with the target CSS shown above.
3. In `2026/index.html:2252`, replace `el.style.width = el.dataset.pct+'%';` with `el.style.transform = 'scaleX(' + (el.dataset.pct/100) + ')';` inside that `.perf-bar-fill` `forEach`.
4. In `2026/index.html:2541`, do the same replacement for the `.feel-bar-fill` `forEach`.
5. In `2026/index.html:2795`, do the same replacement for the `.perf-bar-fill` `forEach`.
6. In `2026/index.html:3083`, do the same replacement for the `.perf-bar-fill` `forEach`.
7. Do not touch the markup template strings at lines 2441/2452 (the `data-pct`/inline-style attributes stay as they are).

## Boundaries

- Do NOT touch `.feel-bar-track`/`.perf-bar-track` (the `overflow:hidden` containers) — they already clip correctly and need no change.
- Do NOT change the 1s duration or `var(--ease)` curve — only the animated property changes, not the timing.
- Do NOT touch any other `.style.width` usage elsewhere in the file that is unrelated to these two bar-fill components.
- If plan 001 (reduced-motion) has already landed, do not add a `.feel-bar-fill`/`.perf-bar-fill` rule inside its `@media (prefers-reduced-motion: reduce)` block — that plan deliberately leaves these two components alone (see its Boundaries section); keep it that way.
- If the cited line numbers or code no longer match exactly (drift since commit `7c3b513`), STOP and report instead of guessing.

## Verification

- **Mechanical**: open `2026/index.html` in a browser via a local static server (e.g. `python3 -m http.server` from `2026/`), open DevTools console, confirm no JS errors when opening a string product detail sheet or a compare sheet with 2-3 racquets/strings.
- **Feel check**:
  - Open any badminton string's detail sheet (e.g. EXBOLT 68) — the 성능 bars (반발력/내구성/타구음/충격흡수력/컨트롤) should fill left-to-right exactly as before, each bar stopping at the same visual length as the old `width`-based version (spot-check one product's percentages against `dataset.pct` in DevTools to confirm the scale matches).
  - Open a compare sheet with 2-3 badminton racquets or strings — the 체감 비교 grid's bars should fill identically to before.
  - In DevTools Performance panel, record while opening a 3-product compare sheet and confirm the bar-fill frames show as `Composite Layers` only, with no `Layout` entries attributable to `.feel-bar-fill`/`.perf-bar-fill` (there will still be other layout work from the sheet's own entrance — only these specific elements need to be layout-free).
  - Set DevTools Animations panel playback to 10% and confirm each bar visually grows from the left edge (not from center, not by clipping — a true left-anchored horizontal scale).
- **Done when**: every bar-fill visually reaches the same final length as before the change, no `Layout` events are attributed to the fill elements in a Performance recording, and no console errors appear across all 4 call sites (product detail sheet, compare sheet, both finder result-detail flows).
