# 009 — Repoint the to-top/compare-bar reveal onto the shared `--ease` token

- **Status**: DONE
- **Commit**: 587369a
- **Severity**: LOW
- **Category**: Cohesion & tokens
- **Estimated scope**: 1 file (`2026/index.html`), 2 selectors

## Problem

`.to-top` and `.compare-bar` (`2026/index.html:125-146`) are the two floating action elements that reveal on scroll (scroll-to-top button, comparison tray). Both use the bare built-in `ease` instead of this project's own `--ease` token, even though every other reveal/overlay transition in the file (`.overlay`, `.sheet`, `.cmp-coach`, `.card-enter`) already uses `var(--ease)`:

```css
/* 2026/index.html:125-134 — current */
.to-top{
  ...
  opacity:0;transform:translateY(10px);pointer-events:none;
  transition:opacity .2s ease, transform .2s ease;
}
.to-top.show{opacity:1;transform:translateY(0);pointer-events:auto;}
...

/* 2026/index.html:139-147 — current */
.compare-bar{
  ...
  opacity:0;transform:translateY(10px);pointer-events:none;
  transition:opacity .2s ease, transform .2s ease;
}
.compare-bar.show{opacity:1;transform:translateY(0);pointer-events:auto;}
```

Plan `005-tokenize-easings.md` (already DONE) consolidated the hand-typed cubic-beziers elsewhere in this file but pre-dates these two selectors' current form and missed them — they're the only remaining `translateY(10px)+opacity` reveal pair still on the plain built-in easing.

## Target

```css
/* target */
.to-top{
  ...
  transition:opacity .2s var(--ease), transform .2s var(--ease);
}
.compare-bar{
  ...
  transition:opacity .2s var(--ease), transform .2s var(--ease);
}
```

## Repo conventions to follow

- `--ease:cubic-bezier(.22,1,.36,1)` is defined once in `:root` at `2026/index.html:22` — do not add a second token or a new curve.
- Exemplar of the same reveal shape already using the token correctly: `.overlay{transition:opacity .55s var(--ease),visibility 0s linear .55s;}` at `2026/index.html:458`.

## Steps

1. At `2026/index.html:131`, change `transition:opacity .2s ease, transform .2s ease;` (inside `.to-top`) to `transition:opacity .2s var(--ease), transform .2s var(--ease);`.
2. At `2026/index.html:145`, change the identical line inside `.compare-bar` the same way.

## Boundaries

- Do NOT change the `.2s` duration, the `translateY(10px)` distance, or any other property on either selector — this plan is the easing keyword only.
- Do NOT touch `.to-top:hover`/`.to-top:active`/`.compare-bar .cb-*` rules — out of scope.
- If either selector's transition line has drifted from the excerpt above, stop and report rather than guessing which line to change.

## Verification

- **Mechanical**: no build step; confirm the file still parses (open in a browser, no console errors).
- **Feel check**: scroll the catalog page down past the fold and confirm the to-top button rises in; add two products to compare and confirm the compare bar rises in the same way. Both should feel identical in character to the sheet/overlay open motion elsewhere in the app — a touch snappier at the start, not a linear crawl. In DevTools Animations panel, slow playback to 10% and confirm the curve is `cubic-bezier(0.22, 1, 0.36, 1)`, not linear-ish `ease`.
- **Done when**: both selectors' `transition` lines reference `var(--ease)` and no other property changed.
