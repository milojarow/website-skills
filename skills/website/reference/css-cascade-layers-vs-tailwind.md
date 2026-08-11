# A custom class in `globals.css` BEATS Tailwind utilities — silently, and only halfway

Tailwind 4 emits its utilities inside `@layer utilities`. A rule written
**outside any layer** — any bare `.my-class { … }` in `globals.css` — wins the
cascade against the ENTIRE layer, regardless of specificity or source order.
That is how `@layer` is defined; it is not a bug.

**The failure mode is worse than "my style didn't apply": it applies HALFWAY.**

Measured case — a tab bar whose active state came from the JSX:

```jsx
className={`my-tab ... ${active ? 'bg-brand-700 text-surface-50' : 'text-[var(--text-body)]'}`}
```

…while the custom class declared **only** `background-color`:

- `bg-brand-700` (in-layer) → **lost** against `.my-tab`'s base background.
- `text-surface-50` (in-layer) → **won**, because `.my-tab` declared no `color`.

Result: near-white text on the light resting background. **The tab for the page
the user was actually on became the only illegible one in the bar.** No tool
catches this: `tsc` green, `eslint` green, the class present in the DOM, the rule
present in the CSS.

## The rule

If you write a component class outside a layer, **that class owns ALL of the
component's visual states** — resting, `:hover`, `:focus-visible`, active — and
the corresponding color utilities leave the JSX. One source of truth. Mixing the
two is the defect.

```css
.my-tab { background-color: var(--tab-bg); color: var(--text-body); }
.my-tab:hover, .my-tab:focus-visible { background-color: var(--tab-hover-bg); color: var(--tab-hover-fg); }
/* AFTER the hover rule: same specificity (0,2,0), last one wins — so hovering the
   current tab doesn't disguise it as something else. */
.my-tab[aria-current='page'] { background-color: var(--tab-active-bg); color: var(--tab-active-fg); }
```

## If you genuinely want utilities to win

Put your rules in `@layer components` (Tailwind orders `theme, base, components,
utilities`). But then *any* utility overrides them, which is usually worse than
the problem you were solving.

## What to check when a color "doesn't apply"

Don't reach for `!important` and don't bump specificity. Ask first: **is the
competing rule inside a layer?** An unlayered rule and a layered one are not
comparable by specificity at all — the layer loses before specificity is
consulted.
