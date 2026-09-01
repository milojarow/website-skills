# Layout: stability + debugging

## Layout stability — siblings never move

Dynamic content (tooltips, descriptions, expanding panels, spawned elements) must **never** displace sibling components. Reserve the space up front:

- Fixed heights or `min-h-*`
- Absolute positioning for overlays
- Reserved / placeholder space

Other components stay immovable regardless of what spawns nearby. A panel that pushes its neighbors when it opens is a bug, not a behavior.

## Layout debugging — trace upward first

When a layout is broken (elements cramped, empty space, wrong alignment), **do NOT patch the leaf component.** The constraint is almost always in an **ancestor**.

Read upward, in order, looking for `maxWidth`, `flex`, `width`, `padding` that caps the child's space budget:

1. The page file
2. Layout files
3. Root layout
4. Global CSS

Only modify the leaf after confirming **every parent in the chain is clean.** Patching symptoms while a parent constraint is still active wastes iterations and looks like flailing.

**Screenshot clue:** when the user marks empty space in red, that geometry means an ancestor refuses to use that space → **fix the ancestor, not the child.**

## `width: fit-content` on a flex container does NOT resolve to the available width

Measured in headless Chromium at a 1440 px viewport, building a weekly-calendar grid: N day columns plus a fixed hour gutter.

**The goal:** a 6-column week fills the available width; a 3-column week stays at its natural size without stretching.

**What fails:**

```css
.grid { display:flex; width:fit-content; max-width:100%; }
.col  { flex:1 1 268px; max-width:268px; min-width:0; }
```

On paper `fit-content` = `min(max(min-content, available), max-content)`, and with max-content = 74 + 6×268 = 1682 > 1360 available it *should* give 1360. **It doesn't** — it resolved to ~660 px, barely half, leaving 700 px of white space on the right. `min-width:0` on the children sinks their min-content contribution, so shrink-to-fit collapses **downward** instead of clamping upward.

Dropping the `max-width` gives the opposite defect: the grid overflows and **two days sit behind a horizontal scrollbar** — and a hidden day in a calendar reads as "nothing scheduled that day", which is precisely the false statement.

**What works — take the count out of the layout and do the arithmetic explicitly:**

```html
<div class="grid" style="--n:6">   <!-- column count, from the generator -->
```

```css
.grid { display:flex; width: min(100%, calc(74px + var(--n) * 268px)); }
.col  { flex:1 1 0; min-width:0; }
```

- 6 columns → `min(100%, 1682px)` = 100% → each column (1360−74)/6 = 214 px. Fills.
- 3 columns → `min(100%, 878px)` = 878 px → each column 268 px. Doesn't stretch.

**The rule:** when the desired width depends on **how many** children there are, pass the count in as a custom property and compute it with `min()`/`calc()`. `fit-content` guesses, and it guesses wrong as soon as the children carry `min-width:0` — which they must, for text to truncate.

**How it was caught:** by rendering the page to an image and *looking at it*, not by reading the CSS. Both defects — overflowing and shrinking — are invisible in the source and obvious in the pixels. Two runs of a headless screenshot tool at an explicit `--viewport-width 1440` cost three minutes; at the tool's default (~875 px, tablet) **neither case manifests**, so always pin the viewport to the width where the layout is supposed to work.

## A flex-column child can collapse to zero height with no error at all

In a fixed-height flex column — `.page { height: 11in; overflow: hidden; display: flex; flex-direction: column; }`, the shape of a printed page or a `h-screen` card — this is a common failure mode:

```css
.page { height: 11in; overflow: hidden; display: flex; flex-direction: column; }
```

When the combined content exceeds the fixed height, the flex algorithm shrinks children to fit. The default is `flex-shrink: 1`, and `min-height: auto` does **not** protect a child whose content is compressible. A whole block can get squashed to ~0 height, and `overflow: hidden` swallows the result — so there's no overflow, no console error, no warning. The block is just gone from the render, and the symptom ("a section is missing") points at the markup or the template, which is not where the bug is.

**The fix:**

```css
.page > * { flex-shrink: 0; }
```

This forces real overflow instead of a silent collapse — the failure becomes visible (content spills past the edge) and therefore diagnosable, instead of a block quietly evaporating.

This bites hardest in anything with a **fixed** container height — generated PDFs, slides, fixed-height cards — because on an ordinary web page the container just grows and the shrink algorithm rarely has to fight for space.

**Method point, and the one that generalizes furthest: rendered output must be looked at, not just executed.** A render command exiting 0 and producing a full-size output file says only that the renderer didn't crash — it says nothing about whether the content is complete. Convert the output to images and open them:

```bash
pdftoppm -r 110 -png output.pdf /tmp/pg     # one image per page
```

Two classes of defect only show up this way: content clipped past the container edge (fixed height + `overflow: hidden` hides it with no trace) and content that collapsed to zero size and vanished. Neither appears in a log or an exit code.

**Related fixed-height-print combo, for reuse:**

- `@page { size: 8.5in 11in; margin: 0 }` plus the page element owning its own width/height (padding for margin) avoids fighting the print engine's own page-box.
- `.page + .page { page-break-before: always }` to split pages.
- A footer pinned with `margin-top: auto` inside the flex column is the *correct* use of flex here, and coexists fine with `flex-shrink: 0` on its siblings.
- Inline images as `data:` URIs so the render has no filesystem/network dependency.
- A generous virtual-time budget on the headless render so webfonts finish loading before capture — otherwise the output silently falls back to the system font, another failure with no error.
