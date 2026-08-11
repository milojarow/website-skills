# An overflowing flex row widens the DOCUMENT, not itself

**Symptom the user reports:** "the whole page is deformed / everything is shifted."
A screenshot shows the browser's horizontal scrollbar, a full-bleed header sliced
off mid-viewport, and body copy running off the left edge. The actual culprit is
**one row of items** — a tab strip, a chip row, a filter bar.

**Why it escalates from a row to the page:** a `flex` container without
`flex-wrap` will not shrink below its content, so the items overflow the
container's box. `overflow: visible` is the default, so that overflow contributes
to the **document's** scroll width. One row too long = the entire page scrolls
sideways. (The `min-w-0` half of this is in
[mobile-design.md](mobile-design.md); this file is the containment + drag half.)

## The measurement that proves it (and that a screenshot cannot)

A screenshot at scroll offset 0 looks *fine* in both the broken and the fixed
version — the defect only appears once something scrolls. So measure:

```js
document.documentElement.scrollWidth > document.documentElement.clientWidth
```

Headless, with no MCP and no devtools session: build a page carrying the real
compiled stylesheet and the real class strings, write the numbers into a DOM node,
and read them back with `--dump-dom`:

```bash
brave --headless --disable-gpu --window-size=919,600 \
      --virtual-time-budget=1500 --dump-dom file://$PWD/case.html | grep -o 'MEDIDA[^<]*'
```

Real numbers from one case: **before, 1487px of content in a 904px viewport;
after, 904 == 904.** Do it for the before *and* after class strings from the same
generator, so both halves are measured by identical code.

## Fix — internal scroll container plus mouse drag

```html
<nav class="relative">                              <!-- anchor for the fades -->
  <div class="flex gap-2 overflow-x-auto overscroll-x-contain py-1
              [scrollbar-width:none] [&::-webkit-scrollbar]:hidden">
    <button class="shrink-0 …">…</button>
```

Five details that decide whether it feels finished:

1. **`shrink-0` on every item is mandatory.** Inside a scroll container without it,
   flex items compress down to their longest word instead of overflowing — so
   nothing ever scrolls and you just get cramped labels. This looks like the fix
   failed when it never engaged.
2. **Drag with the mouse; never supplant touch.** Native touch scrolling already
   has momentum and rubber-banding that hand-written JS will not match, and it
   matters most on the low-end phones that need it. Gate the handler on
   `e.pointerType === 'mouse'` and leave `touch-action` alone.
3. **Take `setPointerCapture` only *after* the drag threshold (~6px), never on
   `pointerdown`** — same rule as [draggable-card-strip.md](draggable-card-strip.md).
   Two things specific to a strip of *buttons* rather than links: cancel the click
   in the **capture phase on the container** (`preventDefault` + `stopPropagation`)
   instead of adding an `onClick` guard to every child, so the items need no
   changes at all; and **reset the travel distance on every `pointerdown`,
   including the ones you bail out of** — a drag that ends outside any item leaves
   a stale distance behind, and the next perfectly good click gets eaten by it.
4. **Setting one overflow axis to `auto` forces the other to `auto` too.** So a
   horizontal scroller also clips vertically and the focus ring of a focused item
   comes out shaved top and bottom (cousin of
   [focus-ring-clipped-by-rounded-parent.md](focus-ring-clipped-by-rounded-parent.md)).
   A little vertical padding *inside* the scroller buys the room. Do **not** add
   negative horizontal margins to bleed the strip past a centered container — that
   reintroduces the page-level overflow you just removed.
5. **Edge fades only on the side that actually has more.** A permanent veil over
   the first item does not read as "there is more", it reads as "this is broken".
   Track `scrollLeft > 1` and `scrollWidth - clientWidth - scrollLeft > 1`,
   recompute on `scroll` *and* through a `ResizeObserver`, and render the gradients
   absolutely so appearing and disappearing displaces nothing. The same condition
   gates `cursor-grab`: a grab cursor on a strip with nowhere to go is a lie.

## Bring the active item into view

When the strip mounts or the selection changes — otherwise a session restored to
the last tab points at something off-screen. Move **only** that strip:
`scrollIntoView` is allowed to scroll ancestors, including the page you just
finished containing. Compare `getBoundingClientRect()` of the item against the
container's and adjust `scrollLeft` by the difference; it is immune to whatever
`offsetParent` happens to be.
