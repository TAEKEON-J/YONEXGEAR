# 008 — Crossfade the SERIES MAP/TECHNOLOGY tab body on switch

- **Status**: DONE
- **Commit**: 587369a
- **Severity**: MEDIUM
- **Category**: Missed opportunity / Interruptibility
- **Estimated scope**: 1 file (`2026/index.html`), 1 CSS rule + 1 template edit

## Problem

`openTennisSeriesMapSheet()` (`2026/index.html:7162-7188`) renders the SERIES MAP/TECHNOLOGY segmented tab added this session. Every tab click does a full `sheet.innerHTML` replace with zero transition on the body content:

```js
// 2026/index.html:7162 — current
function openTennisSeriesMapSheet(seriesName, tab){
  ...
  const bodyHtml = activeTab==='tech' ? tennisRacquetSeriesTechContentHtml(seriesName) : tennisSeriesMapContentHtml(seriesName);
  ...
  sheet.innerHTML = `
    <div class="smap-wrap">
      <div class="smap-head">...</div>
      ${tabsHtml}
      ${bodyHtml}
    </div>
  `;
  ...
}
```

The bubble map and the technology infographic are visually unrelated content — swapping between them with no transition is exactly the "state change that teleports" AUDIT.md category 8 calls out. Every other content-swap-within-an-open-sheet in this codebase (`.finder-wrap`, `.cmp-wrap`) already gets a `panelIn` fade; this one was added after that convention existed and never got it.

## Target

```css
/* target — 2026/index.html, next to the existing .finder-wrap,.cmp-wrap rule at :471 */
.smap-tab-body{animation:panelIn .35s var(--ease);}
```

```js
// target — 2026/index.html:7162, sheet.innerHTML body
sheet.innerHTML = `
  <div class="smap-wrap">
    <div class="smap-head">...</div>
    ${tabsHtml}
    <div class="smap-tab-body">${bodyHtml}</div>
  </div>
`;
```

## Repo conventions to follow

- The `panelIn` keyframe and `var(--ease)` token already exist (`2026/index.html:472` `@keyframes panelIn{from{opacity:0;transform:translateY(5px)}to{opacity:1;transform:translateY(0)}}`; `:22` `--ease:cubic-bezier(.22,1,.36,1)`). Do not invent a new keyframe or curve.
- Exemplar: `.finder-wrap,.cmp-wrap{animation:panelIn .35s var(--ease);}` at `2026/index.html:470` — same "content swapped inside an already-open sheet" situation, same duration, no delay. Copy this pattern exactly rather than the `.sheet-reveal-2` variant (`.5s ... .1s both`), which is for a sheet's *initial* reveal, not a same-sheet tab switch.
- Reduced-motion: `@media(prefers-reduced-motion:reduce)` already redefines the `panelIn` keyframe globally to fade-only (`2026/index.html:901-902`, `@keyframes panelIn{ from{opacity:0} to{opacity:1} }`). Because that override targets the keyframe name, not a selector, `.smap-tab-body` inherits the reduced-motion-safe version automatically — no new media-query rule needed.

## Steps

1. In the `<style>` block, right after the `.finder-wrap,.cmp-wrap{animation:panelIn .35s var(--ease);}` rule (`2026/index.html:471`), add:
   ```css
   .smap-tab-body{animation:panelIn .35s var(--ease);}
   ```
2. In `openTennisSeriesMapSheet()` (`2026/index.html:7162`), change `${bodyHtml}` to `<div class="smap-tab-body">${bodyHtml}</div>` inside the `sheet.innerHTML` template.

## Boundaries

- Do NOT touch `tennisSeriesMapContentHtml()` or `tennisRacquetSeriesTechContentHtml()` themselves — only the wrapper in `openTennisSeriesMapSheet()`.
- Do NOT touch `.smap-tabs`/`.smap-head` — this plan is scoped to the body content only.
- Do NOT add a new easing token or keyframe; reuse `panelIn`/`var(--ease)` exactly as they exist.
- If `openTennisSeriesMapSheet()`'s template literal has drifted from the excerpt above (drift since commit `587369a`), stop and report instead of guessing where to insert the wrapper.

## Verification

- **Mechanical**: open `2026/index.html` in a browser, no console errors on load.
- **Feel check**: open a tennis racquet series with a TECHNOLOGY tab (e.g. VCORE → SERIES MAP → tap TECHNOLOGY), and confirm:
  - The bubble map and the infographic no longer swap instantly — the incoming content fades up over ~350ms.
  - Tapping back and forth between the two tabs rapidly never shows a stuck/half-faded frame (each click restarts a fresh keyframe animation on newly-inserted DOM, which is fine here since neither view keeps the other visible mid-animation).
  - In DevTools Animations panel, slow playback to 10% and confirm only `opacity`/`transform` animate, no layout shift.
  - Toggle `prefers-reduced-motion` (Rendering panel) and confirm the swap becomes a plain opacity fade with no rise.
- **Done when**: switching SERIES MAP ↔ TECHNOLOGY visibly crossfades instead of teleporting, on both a series with the tab (VCORE/PERCEPT/EZONE/ASTREL) and — no-op check — a series without it (should be unaffected, since `tabsHtml`/the tech branch don't render at all there).
