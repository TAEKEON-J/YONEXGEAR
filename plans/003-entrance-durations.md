# 003 — Bring sheet/card/panel entrance durations inside the 200–500ms modal budget

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: MEDIUM
- **Category**: Easing & duration
- **Estimated scope**: 1 file (`2026/index.html`), 3 one-line CSS edits

## Problem

Per AUDIT.md's duration budget table, modals/drawers should animate in 200–500ms; everything else on-screen UI should stay under 300ms. Three entrance animations in `2026/index.html` exceed that:

```css
/* line 348 — product/compare sheet entrance: 700ms */
.sheet{
  background:#fff;width:100%;max-width:520px;border-radius:20px 20px 0 0;
  max-height:88vh;overflow-y:auto;padding:0 0 24px;position:relative;
  animation:up .7s var(--ease);
}

/* line 251 — product grid card entrance: 850ms, runs on every category/filter switch */
.card.card-enter{animation:cardIn .85s var(--ease) both;}

/* line 353 — compare/finder panel entrance: 600ms */
.finder-wrap,.cmp-wrap{animation:panelIn .6s var(--ease);}
```

`cardIn` is the most frequent of the three: `renderGrid()` (`2026/index.html:2152-2155`) staggers it across every card in the grid on each category/filter/series change, one of the most common state changes in the app — see `grid.querySelectorAll('.card, .series-divider').forEach((el,i)=>{ el.style.animationDelay = Math.min(i*42,400)+'ms'; el.classList.add('card-enter'); });`. At 850ms per card plus a stagger delay capped at 400ms, the last card in a full grid doesn't finish animating until 1.25s after the trigger.

## Target

Shorten each animation's declared duration only — keep `var(--ease)` (`cubic-bezier(.22,1,.36,1)`) unchanged, keep the stagger delay logic in `renderGrid()` unchanged, keep every other property on these rules unchanged.

```css
/* target: line 348 */
.sheet{
  background:#fff;width:100%;max-width:520px;border-radius:20px 20px 0 0;
  max-height:88vh;overflow-y:auto;padding:0 0 24px;position:relative;
  animation:up .45s var(--ease);
}

/* target: line 251 */
.card.card-enter{animation:cardIn .4s var(--ease) both;}

/* target: line 353 */
.finder-wrap,.cmp-wrap{animation:panelIn .35s var(--ease);}
```

## Repo conventions to follow

- `var(--ease)` (`2026/index.html:22`, `cubic-bezier(.22,1,.36,1)`) is the project's one shared easing token for entrances — every rule above already uses it and must keep using it; do not introduce a different curve.
- Keyframe bodies (`@keyframes up`, `@keyframes cardIn`, `@keyframes panelIn`) are untouched by this plan — only the duration value in the `animation:` shorthand on each selector changes.

## Steps

1. In `2026/index.html:348`, inside the `.sheet{...}` rule, change `animation:up .7s var(--ease);` to `animation:up .45s var(--ease);`. Leave every other declaration in that rule untouched.
2. In `2026/index.html:251`, change `.card.card-enter{animation:cardIn .85s var(--ease) both;}` to `.card.card-enter{animation:cardIn .4s var(--ease) both;}`.
3. In `2026/index.html:353`, change `.finder-wrap,.cmp-wrap{animation:panelIn .6s var(--ease);}` to `.finder-wrap,.cmp-wrap{animation:panelIn .35s var(--ease);}`.
4. Do not touch `@keyframes up`, `@keyframes cardIn`, or `@keyframes panelIn` themselves (lines 352, 252, 355) — their `from`/`to` values are correct and unrelated to duration.
5. Do not touch the stagger-delay logic in `renderGrid()` (`2026/index.html:2152-2155`, the `Math.min(i*42,400)` calculation) — it composes correctly with the new shorter duration without changes.

## Boundaries

- Do NOT change `.sheet-reveal-2{animation:panelIn .5s var(--ease) .1s both;}` (line 354) — it is a separate selector sharing the same keyframe name but a distinct, already-reasonable duration; this plan does not touch it.
- Do NOT change `@keyframes cmpCoachIn` or its `.25s` duration (line 300) — already inside budget, out of scope.
- Do NOT change any hover/press-feedback transition durations (`.card:hover`, `.card:active`, etc.) — this plan is entrance-only.
- If the three cited durations (`.7s`, `.85s`, `.6s`) don't match what's currently in the file at commit `7c3b513`, STOP and report instead of guessing which value to shorten.

## Verification

- **Mechanical**: open `2026/index.html` via a local static server, confirm no console errors on load.
- **Feel check**:
  - Click any product card: the sheet should slide up and settle noticeably faster than before, but still readable as a deliberate motion, not an instant snap.
  - Switch between 라켓/스트링 subcategory tabs a few times: the card grid should finish its stagger-and-fade-in within roughly 800ms total (400ms max stagger delay + 400ms duration) instead of the previous ~1.25s, and should not feel abrupt.
  - Open the Racquet Finder ("시작하기"): each quiz step swap and the results panel should feel snappier without looking like it skipped a frame.
  - In DevTools Animations panel, set playback to 10% and confirm each of the three re-timed animations (`up`, `cardIn`, `panelIn`) plays smoothly start-to-finish at its new duration with no visual pop or jump-cut at the end.
- **Done when**: all three durations read `.45s`, `.4s`, and `.35s` respectively at their cited selectors, no other rule in the file was modified, and the feel-check items above all read as smooth rather than abrupt.
