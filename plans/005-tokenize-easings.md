# 005 — Tokenize the hand-typed cubic-bezier curves alongside the existing --ease token

- **Status**: TODO
- **Commit**: 7c3b513
- **Severity**: LOW
- **Category**: Cohesion & tokens
- **Estimated scope**: 1 file (`2026/index.html`), 3 new `:root` tokens + 4 call sites updated

## Problem

The project already tokenizes one easing curve (`--ease`) but three other distinct cubic-beziers are hand-typed inline instead of being declared as reusable custom properties:

```css
/* 2026/index.html:17-31 — the one existing token */
:root{
  --navy:#111111;
  --navy-deep:#000000;
  --blue:#1656c9;
  --green:#8cff5c;
  --ease:cubic-bezier(.22,1,.36,1);
  --red:#e01130;
  --ink:#0a0a0a;
  --paper:#ffffff;
  --card:#ffffff;
  --line:#e6e6e6;
  --muted:#6f6f6f;
  --radius:8px;
  font-family:'Archivo',-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Noto Sans KR",sans-serif;
}

/* 2026/index.html:45 — hero dismiss slide, used once */
.hero{
  ...
  transition:transform .55s cubic-bezier(.65,0,.35,1), opacity .45s ease;
}

/* 2026/index.html:249 — card hover, used once */
.card{
  ...
  transition:transform .18s cubic-bezier(.2,.8,.2,1), box-shadow .18s ease;
}

/* 2026/index.html:290 — compare-toggle icon crossfade, used once */
.cmp-icon{
  ...
  transition:opacity .2s cubic-bezier(.2,0,0,1),transform .2s cubic-bezier(.2,0,0,1),filter .2s cubic-bezier(.2,0,0,1);
}

/* 2026/index.html:554 — finder-option checkmark icon crossfade, used once (same curve as .cmp-icon, independently typed) */
.fo-check svg{
  ...
  transition:opacity .2s cubic-bezier(.2,0,0,1),transform .2s cubic-bezier(.2,0,0,1),filter .2s cubic-bezier(.2,0,0,1);
}
```

`cubic-bezier(.2,0,0,1)` is already duplicated verbatim across two components (lines 290 and 554) with no shared source of truth — a future change to one is easy to make without noticing the other.

## Target

Add three new custom properties to `:root`, named for what they do (matching how `--ease` is already named for its role, not its shape), then point all four call sites at the tokens instead of the literal values.

```css
/* target: 2026/index.html:17-31 — three new lines added to :root, --ease unchanged */
:root{
  --navy:#111111;
  --navy-deep:#000000;
  --blue:#1656c9;
  --green:#8cff5c;
  --ease:cubic-bezier(.22,1,.36,1);
  --ease-hover:cubic-bezier(.2,.8,.2,1);
  --ease-icon:cubic-bezier(.2,0,0,1);
  --ease-slide:cubic-bezier(.65,0,.35,1);
  --red:#e01130;
  --ink:#0a0a0a;
  --paper:#ffffff;
  --card:#ffffff;
  --line:#e6e6e6;
  --muted:#6f6f6f;
  --radius:8px;
  font-family:'Archivo',-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Noto Sans KR",sans-serif;
}
```

```css
/* target: line 45 */
.hero{
  ...
  transition:transform .55s var(--ease-slide), opacity .45s ease;
}

/* target: line 249 */
.card{
  ...
  transition:transform .18s var(--ease-hover), box-shadow .18s ease;
}

/* target: line 290 */
.cmp-icon{
  ...
  transition:opacity .2s var(--ease-icon),transform .2s var(--ease-icon),filter .2s var(--ease-icon);
}

/* target: line 554 */
.fo-check svg{
  ...
  transition:opacity .2s var(--ease-icon),transform .2s var(--ease-icon),filter .2s var(--ease-icon);
}
```

## Repo conventions to follow

- Name new tokens for their role, matching `--ease`'s own naming (a job-based name, not a shape-based one like `--ease-strong` or a value-based one like `--ease-2801`): `--ease-hover` (card hover response), `--ease-icon` (icon state crossfade), `--ease-slide` (the hero's full-panel slide).
- Keep the exact insertion point inside `:root` right after the existing `--ease:` line, so all easing tokens stay grouped together (matches the file's existing habit of grouping related tokens — color tokens are already adjacent to each other).
- Do not reformat any surrounding `:root` line; only insert the three new lines.

## Steps

1. In `2026/index.html:22`, immediately after the line `--ease:cubic-bezier(.22,1,.36,1);`, insert three new lines:
   ```css
   --ease-hover:cubic-bezier(.2,.8,.2,1);
   --ease-icon:cubic-bezier(.2,0,0,1);
   --ease-slide:cubic-bezier(.65,0,.35,1);
   ```
2. In `2026/index.html:45`, inside `.hero{...}`, change `transition:transform .55s cubic-bezier(.65,0,.35,1), opacity .45s ease;` to `transition:transform .55s var(--ease-slide), opacity .45s ease;`.
3. In `2026/index.html:249`, inside `.card{...}`, change `transition:transform .18s cubic-bezier(.2,.8,.2,1), box-shadow .18s ease;` to `transition:transform .18s var(--ease-hover), box-shadow .18s ease;`.
4. In `2026/index.html:290`, inside `.cmp-icon{...}`, change `transition:opacity .2s cubic-bezier(.2,0,0,1),transform .2s cubic-bezier(.2,0,0,1),filter .2s cubic-bezier(.2,0,0,1);` to `transition:opacity .2s var(--ease-icon),transform .2s var(--ease-icon),filter .2s var(--ease-icon);`.
5. In `2026/index.html:554`, inside `.fo-check svg{...}`, apply the identical substitution as step 4.

## Boundaries

- Do NOT touch `var(--ease)` itself or any of its existing call sites — this plan only adds new tokens and re-points the 4 literal-cubic-bezier sites above.
- Do NOT touch the plain `ease`/`ease-out`/`linear` keyword usages elsewhere in the file (e.g. `.card:hover` transitions using `ease` for box-shadow, `.cmp-coach{animation:cmpCoachIn .25s ease-out}`) — those are CSS built-in keywords, not hand-typed cubic-beziers, and are out of scope for this plan.
- Do NOT change any duration values — only the easing function reference changes, from a literal to a `var()`.
- If any of the four cited `cubic-bezier(...)` literals don't match exactly what's in the file (drift since commit `7c3b513`), STOP and report rather than guessing which value moved.

## Verification

- **Mechanical**: open `2026/index.html` via a local static server, open DevTools console, confirm no CSS parse errors and no `var()` resolution warnings (Chrome DevTools' Elements > Styles panel will show a strikethrough or warning icon on any custom property that fails to resolve — confirm all four transitions show a resolved computed value, not the raw `var(...)`).
- **Feel check**:
  - Dismiss the hero ("카탈로그 보기"): the slide-away motion should look pixel-identical to before (same curve, just referenced by token now).
  - Hover a product card with a mouse: the lift/shadow transition should feel identical to before.
  - Tap a compare-toggle "+" button on any card: the plus-to-checkmark crossfade should look identical to before.
  - Pick an answer in the Racquet/String Finder quiz: the checkmark crossfade inside the picked option should look identical to before.
  - In DevTools Elements panel, inspect `:root` and confirm `--ease-hover`, `--ease-icon`, `--ease-slide` all appear with their correct values.
- **Done when**: all four transitions render with visually identical timing to before the change, the three new tokens are present and correctly resolved in `:root`, and no other transition/animation in the file references a `cubic-bezier(...)` literal that duplicates one of these three new tokens.
