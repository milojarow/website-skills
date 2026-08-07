# Verifying a PNG→SVG trace of line art — the obvious metric means nothing

When vectorizing a line-art logo (potrace / vtracer), the temptation is to
measure `symmetric difference / ink pixels` against the original mask. **On line
art that metric carries no signal.** Measured case: a perimeter of 41,076 px
against 161,549 px of area — nearly every ink pixel touches an edge, so a
PERFECT trace already spends ~4% on antialias halo alone. Every candidate scored
4.1–4.3% and it looked like none of them worked.

**The right question is not how much differs, but WHERE the difference falls.**

```python
ring   = dilate(ref) & ~erode(ref)      # 1 px band along the contour
outer  = (ref ^ got) & ~ring            # everything that is NOT halo
cores  = erode(outer).sum()             # solid blocks = real loss
```

Three independent measures, and the second one is the strong one:

| measure | what it catches | threshold used |
|---|---|---|
| difference OUTSIDE the 1 px ring | eaten stroke, deleted node, drifted curve | ~60 loose px |
| solid 3×3 cores in there | tells rounding apart from "a piece fell off" | **0** |
| max displacement (by dilation) | how far the extra/missing material sits | 2 px |

The good trace scored: 6,723 px of difference, **99.9% of it hugging the
contour**, 7 px outside, 0 cores. That is "identical", not "4% error".

## Two things this referee caught and the eye did not

1. **`potrace -a 1.334`** (more corner rounding) failed with **23 solid cores**:
   it was rounding the corners of some hexagons enough to count as real
   deviation. By eye it looked fine. `-a 1.0 -O 0.2` won.
2. **The referee itself failed first.** It rasterized black ink onto a black
   background (`rsvg-convert -b black`, and potrace emits `fill="#000000"`) → it
   measured 0 ink pixels ALWAYS and failed every SVG, good or bad, with a very
   convincing 100%. **Render onto WHITE and treat `< 128` as ink.** Mandatory
   control: run it against a deliberately mutilated SVG (one contour removed)
   and confirm it fails — the mutilated one scored 5,447 cores.

## Two more details that cost time

- **`--tight` plus baking the transform.** potrace emits
  `<g transform="translate(...) scale(0.05,-0.05)">` — **the Y is INVERTED**. If
  you later position anything inside the drawing (a glint, an anchor point),
  that flipped Y bills you for it. Bake the transform into the coordinates and
  re-verify with the same referee.
- **Relative coordinates and 2 decimals.** Absolute with 2 decimals doubled the
  path (69.5 KB vs 35 KB). Relative + 2 decimals: 42 KB / 15 KB gzip, with 7 px
  outside the contour. One decimal saves 1.8 KB gzip but triples the stray
  pixels (21). Simplifying with a high `-O` barely helps: potrace already lands
  near the optimum (6% between `-O 0.2` and `-O 2.0`).
- **The path travels ONCE:** `<defs><path id="x"/></defs>` in the root layout
  and `<use href="#x">` in each consumer. Otherwise it is repeated in the
  header, the footer and wherever else it is used. The `<defs>` does NOT get
  `display:none` (that breaks `<use>` in several browsers) — give it a 0×0 box
  out of flow instead.
