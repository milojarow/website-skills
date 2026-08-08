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
