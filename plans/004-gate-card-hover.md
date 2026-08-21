# 004 — Gate the product card hover-lift behind a hover-capable media query

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: MEDIUM
- **Category**: Accessibility
- **Estimated scope**: 1 file (`2026/index.html`), 1 CSS rule moved into a media query

## Problem

`.card:hover` is the only motion-bearing (not merely color-changing) `:hover` rule in the entire file, and it is ungated:

```css
/* 2026/index.html:254 */
.card:hover{transform:translateY(-3px);box-shadow:0 2px 4px rgba(0,0,0,.06),0 16px 32px rgba(0,0,0,.1);}
```

This is the primary tap target of the entire product grid (verified: `.card` is generated once per product, up to ~193 times, in `renderGrid()`). Many mobile browsers fire `:hover` on tap and only clear it on a subsequent tap elsewhere on the page, which can leave a tapped card visually "lifted" (translated -3px with an oversized shadow) until the user taps something else — a touch-first retail catalog is exactly the context where this bug surfaces most.

## Target

Wrap the transform+shadow hover rule in `@media (hover:hover) and (pointer:fine)`, so it only applies on devices with a real, precise hovering pointer (mice, trackpads) and never fires from a touch tap.

```css
/* target: replace the rule at 2026/index.html:254 */
@media (hover:hover) and (pointer:fine) {
  .card:hover{transform:translateY(-3px);box-shadow:0 2px 4px rgba(0,0,0,.06),0 16px 32px rgba(0,0,0,.1);}
}
```

## Repo conventions to follow

- This is the first `@media (hover:hover)` block in the file — there is no existing exemplar to match, so place it immediately after the `.card` rule block (after line 254, before `.card .thumb{` at line 255) to keep it visually adjacent to the selector it modifies, consistent with how other state variants (`.card.card-enter`, `.card:active`) are kept next to `.card` in the source.
- Do not rename or restructure `.card:active{transform:translateY(0) scale(.96);}` (line 253) — it is unrelated (press feedback, not hover) and must keep firing on both touch and mouse.

## Steps

1. In `2026/index.html`, delete the standalone line `.card:hover{transform:translateY(-3px);box-shadow:0 2px 4px rgba(0,0,0,.06),0 16px 32px rgba(0,0,0,.1);}` (line 254).
2. In its place, insert:
   ```css
   @media (hover:hover) and (pointer:fine) {
     .card:hover{transform:translateY(-3px);box-shadow:0 2px 4px rgba(0,0,0,.06),0 16px 32px rgba(0,0,0,.1);}
   }
   ```
3. Confirm the surrounding rules (`.card.card-enter` above, `.card:active` above, `.card .thumb` below) are unchanged and still read in the same order.

## Boundaries

- Do NOT touch `.card:active` (line 253) — press feedback must remain available on touch devices.
- Do NOT add `(hover:hover)` gating to any other selector in the file — the audit found this is the only motion-bearing hover rule; color-only `:hover` rules (`.chip:hover`, `.cmp-add-btn:hover`, `.finder-result-card:hover`, etc.) are out of scope and should not be touched.
- Do NOT change the transform or box-shadow values themselves.
- If line 254 no longer matches exactly the code quoted above (drift since commit `7c3b513`), STOP and report instead of guessing.

## Verification

- **Mechanical**: open `2026/index.html` via a local static server, confirm no CSS syntax errors (page still renders, no broken layout) and no console errors.
- **Feel check**:
  - With a real mouse/trackpad (or Chrome DevTools with device emulation OFF), hover a product card: it should still lift -3px with the larger shadow, exactly as before.
  - In Chrome DevTools, open the Device Toolbar (Ctrm/Cmd+Shift+M) to emulate a touch device (e.g. iPhone), then tap a card: it should navigate to the product sheet without any visible "stuck lift" on the card underneath when you navigate back.
  - In DevTools, use the Rendering panel's "Emulate CSS media feature `hover`" set to `none` (and `pointer` to `coarse`): confirm hovering a card with the mouse in this emulated state produces NO lift/shadow change (proving the gate works), then set both back to default and confirm the hover returns.
- **Done when**: the hover lift still works with a real pointer, is confirmed absent under `hover:none`/`pointer:coarse` emulation, and no other selector in the file was modified.
