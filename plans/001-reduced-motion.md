# 001 — Honor prefers-reduced-motion across all entrance/decorative animation

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: HIGH
- **Category**: Accessibility
- **Estimated scope**: 1 file (`2026/index.html`), one new `@media` block (~35 lines of CSS), no JS changes

## Problem

`2026/index.html` has zero occurrences of `prefers-reduced-motion` across its entire `<style>` block — verified via `grep -n "prefers-reduced-motion" 2026/index.html` (no matches). Every one of the 6 `@keyframes` blocks and every transform-based `transition` in the file runs unconditionally, including a full-viewport 100%-height slide-away and a continuous infinite bob. A user with `prefers-reduced-motion: reduce` set gets no accommodation anywhere in the app.

Current code (all in `2026/index.html`):

```css
/* line 45-47 — hero dismiss: full-viewport slide */
.hero{
  ...
  transition:transform .55s cubic-bezier(.65,0,.35,1), opacity .45s ease;
}
.hero.hide{transform:translateY(-100%);opacity:0;pointer-events:none;}

/* line 74-81 — continuous infinite bob on the scroll-hint chevron */
.hero-scroll{
  /* (surrounding declarations omitted, unchanged) */
  animation:bob 1.8s ease-in-out infinite;
}
@keyframes bob{0%,100%{transform:translate(-50%,0);}50%{transform:translate(-50%,7px);}}

/* line 84-89 — staggered hero landing text entrance */
@keyframes heroTextIn{from{opacity:0;transform:translateY(12px)}to{opacity:1;transform:translateY(0)}}
.hero-badge-row{animation:heroTextIn .85s var(--ease) both;}
.hero-title{animation:heroTextIn .85s var(--ease) .12s both;}
.hero-rule{animation:heroTextIn .85s var(--ease) .22s both;}
.hero-sub{animation:heroTextIn .85s var(--ease) .3s both;}
.hero-stats{animation:heroTextIn .85s var(--ease) .38s both;}

/* line 251-254 — product grid card entrance + hover lift */
.card.card-enter{animation:cardIn .85s var(--ease) both;}
@keyframes cardIn{from{opacity:0;transform:translateY(16px)}to{opacity:1;transform:translateY(0)}}
.card:hover{transform:translateY(-3px);box-shadow:0 2px 4px rgba(0,0,0,.06),0 16px 32px rgba(0,0,0,.1);}

/* line 305 — compare-toggle coachmark entrance */
@keyframes cmpCoachIn{from{opacity:0;transform:translateY(-4px)}to{opacity:1;transform:translateY(0)}}
/* line 304 — coachmark exit */
.cmp-coach.leaving{opacity:0;transform:translateY(-4px);}

/* line 352 — product/compare sheet entrance */
@keyframes up{from{transform:translateY(16px) scale(.99);opacity:0}to{transform:translateY(0) scale(1);opacity:1}}

/* line 355 — compare/finder panel entrance */
@keyframes panelIn{from{opacity:0;transform:translateY(5px)}to{opacity:1;transform:translateY(0)}}
```

## Target

Add one `@media (prefers-reduced-motion: reduce)` block, placed immediately before the closing `</style>` tag (end of the `<style>` block, so it wins the cascade over every rule above). Per AUDIT.md: reduced motion means fewer and gentler animations, **not zero** — keep the opacity feedback, drop position/scale movement. Small, non-directional feedback (press-scale on `:active`, the icon plus/check crossfade, the bar-fill reveals) is left untouched: it is brief, uniform, in-place motion that aids comprehension of state rather than disorienting movement, so it does not need gating.

```css
@media (prefers-reduced-motion: reduce) {
  /* hero dismiss: drop the full-viewport slide, keep the fade */
  .hero{ transition:opacity .45s ease; }
  .hero.hide{ transform:none; opacity:0; }

  /* kill the continuous decorative bob entirely — no opacity/color analog to preserve */
  .hero-scroll{ animation:none; }

  /* hero landing text: fade only, no rise */
  @keyframes heroTextIn{ from{opacity:0} to{opacity:1} }

  /* product grid card entrance: fade only, no rise */
  @keyframes cardIn{ from{opacity:0} to{opacity:1} }

  /* card hover lift: drop the translateY, keep the shadow cue */
  .card:hover{ transform:none; }

  /* compare-toggle coachmark: fade only, both directions */
  @keyframes cmpCoachIn{ from{opacity:0} to{opacity:1} }
  .cmp-coach.leaving{ transform:none; }

  /* product/compare sheet entrance: fade only, no rise/scale */
  @keyframes up{ from{opacity:0} to{opacity:1} }

  /* compare/finder panel entrance: fade only, no rise */
  @keyframes panelIn{ from{opacity:0} to{opacity:1} }
}
```

## Repo conventions to follow

- The file has exactly one `<style>` block (starts at `2026/index.html:16`, closes later in the `<head>`); this new block is the last rule in it, so no specificity fights are needed — a later same-media rule always wins over an earlier one for the same selector/keyframe name.
- `@keyframes` redeclared inside a `@media` block is standard CSS and is what makes this pattern work: when the media query matches, the later-declared keyframes definition (the one inside `@media`) replaces the base one for any element using `animation-name` that matches, with no JS required.
- Match the existing code style exactly: no spaces after selectors before `{`, declarations on one line where the surrounding file does that (see any rule above for the pattern), 2-space indent inside `@media`.

## Steps

1. Open `2026/index.html`. Find the end of the `<style>` block — search for the first `</style>` tag after line 16 (the `:root` declaration). Confirm you are still inside the same `<style>` element that contains `.hero`, `.card`, `@keyframes cardIn`, etc. (all in this one block, roughly lines 16–630 based on the commit stamp above — re-verify with the running line numbers if the file has drifted).
2. Insert the full `@media (prefers-reduced-motion: reduce) { ... }` block from **Target** above as the last rule before that closing `</style>` tag, exactly as written (all 8 sub-rules, in that order).
3. Do not modify any of the base (non-media) rules — this is purely additive.

## Boundaries

- Do NOT touch `:active` press-scale rules (`.card:active`, `.cmp-toggle:active`, `.finder-btn:active`, etc.) — these are brief, in-place, non-directional feedback and are intentionally left alone per the Problem/Target reasoning above.
- Do NOT touch the icon-crossfade rules (`.cmp-icon-plus`/`.cmp-icon-check`, `.fo-check svg`) — these are the app's state-indication mechanism (plus → checkmark), not decorative entrance motion.
- Do NOT touch `.feel-bar-fill`/`.perf-bar-fill` width/transform transitions — a separate plan (002) changes their implementation; this plan only adds the reduced-motion media query and must not conflict with it. If plan 002 has already landed when you run this plan, do not re-touch those selectors here.
- Do NOT add any JavaScript (no `matchMedia` listener, no `useReducedMotion` — this is a static CSS file, the media query alone is sufficient).
- If any of the 8 cited selectors/keyframe names no longer exist verbatim at their cited lines (drift since commit `7c3b513`), STOP and report instead of guessing which rule replaced them.

## Verification

- **Mechanical**: no build step exists for this static file. Open `2026/index.html` directly in a browser (e.g. `python3 -m http.server` from the `2026/` directory, then visit `http://localhost:PORT/index.html`) and confirm the page loads with no console errors.
- **Feel check**:
  - In Chrome/Edge DevTools, open the Rendering panel (Cmd/Ctrl+Shift+P → "Show Rendering"), set "Emulate CSS media feature prefers-reduced-motion" to `reduce`.
  - Reload the page: the hero landing text should still fade in but not rise; the scroll-hint chevron should sit still (no bob).
  - Click "카탈로그 보기" to dismiss the hero: it should fade out in place, not slide up off-screen.
  - Open any product card: the sheet should fade in without the slide-up/scale.
  - Switch category/filter tabs: grid cards should fade in without rising.
  - Hover a card with a mouse: it should NOT lift (no `translateY`), but should still gain the stronger box-shadow.
  - Trigger the compare coachmark (first visit to 배드민턴 → 라켓, cleared `localStorage`): it should fade in/out without the 4px vertical shift.
  - Turn the emulation back to "No emulation" and repeat: all the original motion (slide, bob, rise, lift) should be back exactly as before — this block must be fully inert when the preference is off.
- **Done when**: every check above passes in both the `reduce` and default (no-preference) emulation states, and no rule outside the new `@media` block was modified.
