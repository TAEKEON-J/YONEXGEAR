# 006 — Give the three trigger-anchored popovers a scale+fade entrance from their trigger

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: MEDIUM (missed opportunity)
- **Category**: Physicality & origin / Missed opportunities
- **Estimated scope**: 1 file (`2026/index.html`), 3 CSS rules + 1 shared rule + 6 JS functions

## Problem

Three popovers — the tech-glossary tooltip (`#techPopover`), the recommended-strings popover (`#recoPopover`), and the add-to-compare search popover (`#cmpAddPopover`) — are all positioned relative to the button that opened them, but all three appear and disappear by toggling `style.display` between `'none'` and `'block'` directly, with no transition and no `transform-origin`. They teleport into existence instead of visibly emerging from their trigger.

Current CSS (base rules, unchanged parts omitted with `...`):

```css
/* 2026/index.html:611-615 */
#techPopover{
  position:fixed;z-index:200;max-width:260px;background:var(--navy-deep);color:#fff;
  padding:10px 12px;border-radius:10px;font-size:12.5px;line-height:1.5;
  box-shadow:0 8px 20px rgba(0,0,0,.3);display:none;
}

/* 2026/index.html:446-451 */
#recoPopover{
  position:fixed;z-index:200;width:236px;background:#fff;color:var(--ink);
  padding:18px;border-radius:16px;font-size:13px;line-height:1.5;
  box-shadow:0 20px 48px rgba(0,0,0,.16),0 2px 10px rgba(0,0,0,.06);
  border:1px solid var(--line);display:none;
}

/* 2026/index.html:473-478 */
#cmpAddPopover{
  position:fixed;z-index:200;width:260px;background:#fff;color:var(--ink);
  padding:14px;border-radius:16px;font-size:13px;
  box-shadow:0 20px 48px rgba(0,0,0,.16),0 2px 10px rgba(0,0,0,.06);
  border:1px solid var(--line);display:none;
}
```

Current JS (all three show/close pairs):

```js
// 2026/index.html:1858-1876 — techPopover
function showTechPopover(evt, idx){
  evt.stopPropagation();
  const entry = TECH_GLOSSARY[idx];
  const pop = document.getElementById('techPopover');
  document.getElementById('techPopoverTitle').textContent = entry.label;
  document.getElementById('techPopoverDesc').textContent = entry.desc;
  const rect = evt.target.getBoundingClientRect();
  const popW = 260;
  let left = rect.left - popW/2 + rect.width/2;
  left = Math.max(10, Math.min(left, window.innerWidth - popW - 10));
  let top = rect.bottom + 8;
  if(top > window.innerHeight - 100) top = rect.top - 90;
  pop.style.left = left + 'px';
  pop.style.top = top + 'px';
  pop.style.display = 'block';
}
function closeTechPopover(){
  document.getElementById('techPopover').style.display = 'none';
}

// 2026/index.html:2856-2895 — recoPopover (unrelated body-building code omitted, unchanged by this plan)
  const rect = evt.target.getBoundingClientRect();
  const sheetRect = document.getElementById('sheet').getBoundingClientRect();
  const popW = 236;
  pop.style.visibility = 'hidden';
  pop.style.display = 'block';
  const popH = pop.offsetHeight;
  pop.style.visibility = '';
  const rightSpace = window.innerWidth - sheetRect.right;
  let left, top;
  if(rightSpace >= popW + 28){
    left = sheetRect.right + 16;
    top = sheetRect.bottom - popH;
    top = Math.max(16, Math.min(top, window.innerHeight - popH - 16));
  } else {
    left = Math.max(10, Math.min(rect.left, window.innerWidth - popW - 10));
    top = rect.bottom + 8;
    if(top + popH > window.innerHeight - 10) top = Math.max(10, rect.top - popH - 8);
  }
  pop.style.left = left + 'px';
  pop.style.top = top + 'px';
  pop.style.display = 'block';
}
function closeRecoPopover(){
  document.getElementById('recoPopover').style.display = 'none';
}

// 2026/index.html:2902-2923 — cmpAddPopover
function showCmpAddPopover(evt){
  evt.stopPropagation();
  const pop = document.getElementById('cmpAddPopover');
  document.getElementById('cmpAddSearch').value = '';
  renderCmpAddList('');
  const rect = evt.target.getBoundingClientRect();
  const popW = 260;
  pop.style.visibility = 'hidden';
  pop.style.display = 'block';
  const popH = pop.offsetHeight;
  pop.style.visibility = '';
  let left = Math.max(10, Math.min(rect.left, window.innerWidth - popW - 10));
  let top = rect.bottom + 8;
  if(top + popH > window.innerHeight - 10) top = Math.max(10, rect.top - popH - 8);
  pop.style.left = left + 'px';
  pop.style.top = top + 'px';
  pop.style.display = 'block';
  document.getElementById('cmpAddSearch').focus();
}
function closeCmpAddPopover(){
  document.getElementById('cmpAddPopover').style.display = 'none';
}
```

All three outside-click handlers check the raw display value and must keep working unmodified:

```js
// 2026/index.html:1893-1898
document.addEventListener('click', (e)=>{
  const pop = document.getElementById('techPopover');
  if(pop.style.display === 'block' && !pop.contains(e.target) && !e.target.classList.contains('tech-info')){
    closeTechPopover();
  }
});
// (2026/index.html:2896-2901 and 2951-2956 follow the identical pattern for recoPopover/cmpAddPopover — same structure, different id/trigger class)
```

## Target

Add a shared `opacity`/`transform`/`transition` treatment to all three base rules (scale `.96→1` + opacity `0→1`, 160ms, matching AUDIT.md's tooltip/small-popover budget of 125–200ms and its physicality target of `scale(0.9–0.97)`), plus a `.pop-open` modifier class. Set `transform-origin` in JS at open time, based on which branch of each function's existing above/below/side-panel positioning logic fired, so each popover visibly grows from the edge nearest its trigger. Keep every positioning calculation exactly as it is today — only add origin-setting and class-toggling around it.

```css
/* target: 2026/index.html:611-615 */
#techPopover{
  position:fixed;z-index:200;max-width:260px;background:var(--navy-deep);color:#fff;
  padding:10px 12px;border-radius:10px;font-size:12.5px;line-height:1.5;
  box-shadow:0 8px 20px rgba(0,0,0,.3);display:none;
  opacity:0;transform:scale(.96);
  transition:opacity .16s cubic-bezier(.2,0,0,1),transform .16s cubic-bezier(.2,0,0,1);
}
#techPopover.pop-open{opacity:1;transform:scale(1);}

/* target: 2026/index.html:446-451 */
#recoPopover{
  position:fixed;z-index:200;width:236px;background:#fff;color:var(--ink);
  padding:18px;border-radius:16px;font-size:13px;line-height:1.5;
  box-shadow:0 20px 48px rgba(0,0,0,.16),0 2px 10px rgba(0,0,0,.06);
  border:1px solid var(--line);display:none;
  opacity:0;transform:scale(.96);
  transition:opacity .16s cubic-bezier(.2,0,0,1),transform .16s cubic-bezier(.2,0,0,1);
}
#recoPopover.pop-open{opacity:1;transform:scale(1);}

/* target: 2026/index.html:473-478 */
#cmpAddPopover{
  position:fixed;z-index:200;width:260px;background:#fff;color:var(--ink);
  padding:14px;border-radius:16px;font-size:13px;
  box-shadow:0 20px 48px rgba(0,0,0,.16),0 2px 10px rgba(0,0,0,.06);
  border:1px solid var(--line);display:none;
  opacity:0;transform:scale(.96);
  transition:opacity .16s cubic-bezier(.2,0,0,1),transform .16s cubic-bezier(.2,0,0,1);
}
#cmpAddPopover.pop-open{opacity:1;transform:scale(1);}
```

```js
// target: techPopover
function showTechPopover(evt, idx){
  evt.stopPropagation();
  const entry = TECH_GLOSSARY[idx];
  const pop = document.getElementById('techPopover');
  document.getElementById('techPopoverTitle').textContent = entry.label;
  document.getElementById('techPopoverDesc').textContent = entry.desc;
  const rect = evt.target.getBoundingClientRect();
  const popW = 260;
  let left = rect.left - popW/2 + rect.width/2;
  left = Math.max(10, Math.min(left, window.innerWidth - popW - 10));
  let top = rect.bottom + 8;
  let placedAbove = false;
  if(top > window.innerHeight - 100){ top = rect.top - 90; placedAbove = true; }
  pop.style.left = left + 'px';
  pop.style.top = top + 'px';
  pop.style.transformOrigin = placedAbove ? 'bottom center' : 'top center';
  pop.style.display = 'block';
  void pop.offsetHeight;
  pop.classList.add('pop-open');
}
function closeTechPopover(){
  const pop = document.getElementById('techPopover');
  pop.classList.remove('pop-open');
  setTimeout(()=>{ pop.style.display = 'none'; }, 160);
}

// target: recoPopover (only the positioning tail shown — body-building code above it is unchanged)
  const rect = evt.target.getBoundingClientRect();
  const sheetRect = document.getElementById('sheet').getBoundingClientRect();
  const popW = 236;
  pop.style.visibility = 'hidden';
  pop.style.display = 'block';
  const popH = pop.offsetHeight;
  pop.style.visibility = '';
  const rightSpace = window.innerWidth - sheetRect.right;
  let left, top, origin;
  if(rightSpace >= popW + 28){
    left = sheetRect.right + 16;
    top = sheetRect.bottom - popH;
    top = Math.max(16, Math.min(top, window.innerHeight - popH - 16));
    origin = 'left center';
  } else {
    left = Math.max(10, Math.min(rect.left, window.innerWidth - popW - 10));
    top = rect.bottom + 8;
    origin = 'top center';
    if(top + popH > window.innerHeight - 10){ top = Math.max(10, rect.top - popH - 8); origin = 'bottom center'; }
  }
  pop.style.left = left + 'px';
  pop.style.top = top + 'px';
  pop.style.transformOrigin = origin;
  pop.style.display = 'block';
  pop.classList.add('pop-open');
}
function closeRecoPopover(){
  const pop = document.getElementById('recoPopover');
  pop.classList.remove('pop-open');
  setTimeout(()=>{ pop.style.display = 'none'; }, 160);
}

// target: cmpAddPopover
function showCmpAddPopover(evt){
  evt.stopPropagation();
  const pop = document.getElementById('cmpAddPopover');
  document.getElementById('cmpAddSearch').value = '';
  renderCmpAddList('');
  const rect = evt.target.getBoundingClientRect();
  const popW = 260;
  pop.style.visibility = 'hidden';
  pop.style.display = 'block';
  const popH = pop.offsetHeight;
  pop.style.visibility = '';
  let left = Math.max(10, Math.min(rect.left, window.innerWidth - popW - 10));
  let top = rect.bottom + 8;
  let origin = 'top center';
  if(top + popH > window.innerHeight - 10){ top = Math.max(10, rect.top - popH - 8); origin = 'bottom center'; }
  pop.style.left = left + 'px';
  pop.style.top = top + 'px';
  pop.style.transformOrigin = origin;
  pop.style.display = 'block';
  pop.classList.add('pop-open');
  document.getElementById('cmpAddSearch').focus();
}
function closeCmpAddPopover(){
  const pop = document.getElementById('cmpAddPopover');
  pop.classList.remove('pop-open');
  setTimeout(()=>{ pop.style.display = 'none'; }, 160);
}
```

## Repo conventions to follow

- This exact "class toggle for the transition + `setTimeout` to flip `display:none` after the transition completes" pattern is already established in the file for `.cmp-coach`'s exit: see `dismissCmpCoach()` (`2026/index.html:2310-2317`), which does `coach.classList.add('leaving'); setTimeout(()=>{ coach.style.display='none'; coach.classList.remove('leaving'); }, 180);`. This plan's `closeXPopover()` functions follow the same shape (add exit intent → delay → hide), scaled to the 160ms duration used here.
- `recoPopover`/`cmpAddPopover` already force a reflow via the `pop.style.visibility='hidden';pop.style.display='block';const popH=pop.offsetHeight;pop.style.visibility='';` sequence — that existing reflow is sufficient to make the later `classList.add('pop-open')` transition reliably, so no extra `void pop.offsetHeight` is needed for those two. `techPopover` has no such measurement step today, so this plan adds one explicit `void pop.offsetHeight;` line for it alone, immediately before `pop.classList.add('pop-open')`.
- `cubic-bezier(.2,0,0,1)` is the same curve already used twice in this file for the compare-toggle and finder-checkmark icon crossfades (`2026/index.html:290,554`) — reusing it here for a third small-UI-element transition is consistent with existing practice. (If plan 005 has already landed and added `--ease-icon:cubic-bezier(.2,0,0,1)` to `:root`, you may reference `var(--ease-icon)` instead of the literal in the three new `transition:` lines — either is correct; do not block on plan 005 if it hasn't landed.)

## Steps

1. In `2026/index.html:611-615`, add the four new lines (`opacity:0;transform:scale(.96);transition:...;`) inside the existing `#techPopover{...}` rule and add the new `#techPopover.pop-open{opacity:1;transform:scale(1);}` rule immediately after it, per **Target**.
2. Repeat step 1 for `#recoPopover` (`2026/index.html:446-451`) and `#cmpAddPopover` (`2026/index.html:473-478`), per **Target**.
3. Replace `showTechPopover`/`closeTechPopover` (`2026/index.html:1858-1876`) with the target versions above.
4. Replace the positioning tail of the reco-popover show function and `closeRecoPopover` (`2026/index.html:2871-2895`) with the target versions above — leave the earlier body-building code (the `groups`/`items` HTML construction, lines 2856-2870) untouched.
5. Replace `showCmpAddPopover`/`closeCmpAddPopover` (`2026/index.html:2902-2923`) with the target versions above.
6. Leave all three outside-click handlers (`2026/index.html:1893-1898`, and the equivalent blocks for reco/cmpAdd) completely untouched — they already check `pop.style.display === 'block'`, which this plan continues to set identically, so they keep working with zero changes.

## Boundaries

- Do NOT change any of the positioning math (the `left`/`top`/`popW`/`popH`/clamping calculations) — only add `origin`/`transformOrigin` assignment alongside the existing branches, and the class/display choreography around them.
- Do NOT change the recoPopover body-building code (the `groups`/`items` template literals) or `renderCmpAddList`/`filterCmpAddList` — out of scope.
- Do NOT create a `@media (prefers-reduced-motion: reduce)` block yourself. If plan 001 has already landed (its media block exists near the end of the `<style>` section), you may add `#techPopover.pop-open,#recoPopover.pop-open,#cmpAddPopover.pop-open{transform:none}` inside that existing block; if it has not landed, skip this — do not create the block.
- Do NOT touch `#techPopoverTitle`, `#techPopoverDesc`, `#recoPopoverBody`, `#cmpAddList`, `#cmpAddSearch`, or any other content-population logic.
- If any of the cited functions or CSS rules don't match the code shown above (drift since commit `7c3b513`), STOP and report instead of improvising a merge.

## Verification

- **Mechanical**: open `2026/index.html` via a local static server, open DevTools console, confirm no JS errors when opening/closing each of the three popovers repeatedly, including rapid open-close-open cycles (click a `tech-info` glossary term, then immediately click another one before the first finishes closing).
- **Feel check**:
  - Click any `ⓘ` tech-glossary term in a product spec table: the popover should visibly grow from the edge nearest the term (top-anchored if it appears below, bottom-anchored if it appears above a term near the bottom of the viewport) rather than snapping into view.
  - On a product with `reco` data (Astrox 99 Pro), click "추천 스트링": on a wide viewport it should grow from its left edge (nearest the sheet); on a narrow viewport where it falls back to the below/above-button placement, it should grow from the matching top/bottom edge.
  - Click "+ 상품 추가" in an open compare sheet: the search popover should grow from the edge nearest the button.
  - Click outside any open popover: it should fade/scale out over ~160ms, not vanish instantly, and must be fully gone (not just invisible-but-present) by the time you can interact with whatever is underneath it — confirm via DevTools that `display` reads `none` after the transition (Elements panel, ~200ms after clicking outside).
  - In DevTools Animations panel, set playback to 10% for each popover's open transition and confirm the scale animates from `.96` to `1` smoothly with no jump.
- **Done when**: all three popovers scale+fade in from a trigger-relative edge, scale+fade out on close without an instant pop, the outside-click-to-close behavior for all three still works unmodified, and no positioning calculation changed from its original values.
