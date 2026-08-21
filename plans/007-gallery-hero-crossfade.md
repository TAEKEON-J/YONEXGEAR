# 007 — Crossfade the product sheet's hero photo when a gallery thumbnail is clicked

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: LOW (missed opportunity)
- **Category**: Missed opportunities
- **Estimated scope**: 1 file (`2026/index.html`), 1 markup edit + 1 function rewrite

## Problem

Inside an open product detail sheet, clicking a smaller gallery thumbnail swaps the large hero photo by setting `src` directly, with no transition — the image pops instantly from one photo to the next.

Current code:

```js
// 2026/index.html:1574-1582 — the hero <img> markup, created fresh each time openSheet() runs
function sheetHeroContent(p){
  const imgs = galleryList(p);
  const src = imgs[0];
  if(src){
    return `<img id="sheetHeroImg" src="${src}" alt="${p.name}" style="width:100%;height:100%;object-fit:cover;"
              onerror="this.replaceWith(Object.assign(document.createElement('div'),{innerHTML:iconFor('${p.s}'),style:'display:flex;align-items:center;justify-content:center;height:100%;'}))">`;
  }
  return iconFor(p.s);
}

// 2026/index.html:2218-2223 — called by each gallery thumbnail's onclick
function setHeroImage(src, thumbEl){
  const hero = document.getElementById('sheetHeroImg');
  if(hero) hero.src = src;
  document.querySelectorAll('.gallery-thumb').forEach(t=> t.style.border = '2px solid transparent');
  if(thumbEl) thumbEl.style.border = '2px solid var(--navy)';
}
```

This only affects products with 2+ gallery images (the thumbnail strip only renders when `galleryList(p).length >= 2`, per `galleryStripHtml()`), so it fires at "occasional" frequency — a handful of times per product view at most — squarely in the AUDIT.md budget for a standard, non-decorative transition.

## Target

Give `#sheetHeroImg` an opacity transition in its inline style, and make `setHeroImage()` fade the current photo out, swap the `src`, then fade the new photo in — a simple two-step timeout crossfade, no second `<img>` element needed since a full opaque swap mid-fade is imperceptible at this duration.

```js
// target: 2026/index.html:1574-1582 — add opacity:1;transition:opacity .15s ease; to the inline style
function sheetHeroContent(p){
  const imgs = galleryList(p);
  const src = imgs[0];
  if(src){
    return `<img id="sheetHeroImg" src="${src}" alt="${p.name}" style="width:100%;height:100%;object-fit:cover;opacity:1;transition:opacity .15s ease;"
              onerror="this.replaceWith(Object.assign(document.createElement('div'),{innerHTML:iconFor('${p.s}'),style:'display:flex;align-items:center;justify-content:center;height:100%;'}))">`;
  }
  return iconFor(p.s);
}

// target: 2026/index.html:2218-2223
function setHeroImage(src, thumbEl){
  const hero = document.getElementById('sheetHeroImg');
  if(hero){
    hero.style.opacity = '0';
    setTimeout(()=>{
      hero.src = src;
      hero.style.opacity = '1';
    }, 150);
  }
  document.querySelectorAll('.gallery-thumb').forEach(t=> t.style.border = '2px solid transparent');
  if(thumbEl) thumbEl.style.border = '2px solid var(--navy)';
}
```

## Repo conventions to follow

- `#sheetHeroImg`'s styling is entirely inline (no dedicated CSS class exists for it in the `<style>` block) — this plan keeps that pattern and adds the transition inline rather than introducing a new class, matching how the element is styled today.
- Plain `ease` (not a custom cubic-bezier) matches AUDIT.md's guidance for "hover / color change → `ease`"; an opacity crossfade on a static image swap falls in that same low-emphasis bucket, so no new easing token is needed here.
- `150ms` split into two 150ms halves (fade-out then swap-and-fade-in) keeps the whole crossfade inside AUDIT.md's small-popover/tooltip-adjacent budget (125–200ms) without being so slow it delays feeling responsive to the tap.

## Steps

1. In `2026/index.html:1578`, inside the `<img id="sheetHeroImg" ...>` template string, change the `style` attribute from `style="width:100%;height:100%;object-fit:cover;"` to `style="width:100%;height:100%;object-fit:cover;opacity:1;transition:opacity .15s ease;"`. Leave the `onerror` handler and every other attribute on that line untouched.
2. In `2026/index.html:2218-2223`, replace the body of `setHeroImage(src, thumbEl)` with the target version above: instead of `if(hero) hero.src = src;`, wrap the swap in a 150ms `setTimeout` that first sets `hero.style.opacity = '0'`, then inside the timeout sets both the new `src` and `hero.style.opacity = '1'`.
3. Leave the two `document.querySelectorAll('.gallery-thumb')...`/`thumbEl.style.border` lines exactly as they are — the thumbnail-selection border swap is instant and correct as-is; this plan only changes the large hero photo's transition.

## Boundaries

- Do NOT add a second `<img>` element or any crossfade-via-stacking technique — the single-element opacity-out/swap/opacity-in approach is sufficient here and matches the file's existing preference for minimal DOM (no other image transition in the file uses a stacked-image technique).
- Do NOT change the `onerror` fallback behavior (the broken-image → icon replacement) — if `onerror` fires after this change, the element is replaced entirely, which naturally short-circuits the fade; no special-casing is needed.
- Do NOT touch `galleryStripHtml()`, `galleryList()`, or the thumbnail markup/border-swap logic — only `sheetHeroContent()`'s inline style and `setHeroImage()`'s body change.
- If the cited line numbers or code no longer match exactly (drift since commit `7c3b513`), STOP and report instead of guessing.

## Verification

- **Mechanical**: open `2026/index.html` via a local static server, open a product with 2+ gallery images (e.g. any Astrox racquet with multiple angle photos), open DevTools console, confirm no JS errors when clicking through the gallery thumbnails.
- **Feel check**:
  - Click a second/third gallery thumbnail: the large hero photo should visibly fade out and back in around the swap, not pop instantly.
  - Click rapidly through 3+ thumbnails in quick succession: confirm no visual glitch (a flash of the wrong image stuck at 0 opacity, or the opacity failing to return to 1) — each click should cleanly restart the fade cycle.
  - Confirm the thumbnail-selection border (the navy outline on the active thumbnail) still updates instantly and independently of the hero photo's fade timing.
  - In DevTools Animations panel, set playback to 10% and confirm the opacity transition is smooth in both directions.
- **Done when**: every gallery thumbnail click produces a visible fade rather than an instant swap, rapid clicking doesn't leave the hero image stuck at a partial opacity, and the thumbnail border-selection behavior is unchanged.
