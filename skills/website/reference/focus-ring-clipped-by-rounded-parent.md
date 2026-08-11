# A focus ring inside `overflow-hidden` + `rounded-*` loses its four corners

**Symptom the user reports:** "the rounded corners look broken / there are four
loose lines around the card." A card with a rounded silhouette shows a
square-cornered ring whose corners are missing, so it reads as four floating
straight segments instead of a ring.

**The markup that produces it** — the very common disclosure-row pattern:

```html
<li class="overflow-hidden rounded-xl bg-…">   <!-- rounded clip -->
  <button class="w-full px-4 py-3 …">          <!-- NO radius, fills the parent -->
```

## Mechanism (measured in Chromium, not deduced)

The native `:focus-visible` ring (`outline: -webkit-focus-ring-color auto 1px`) is
painted **straddling** the button's border box — part inside, part outside. The
button has no radius, so the ring is a square. `overflow: hidden` on the parent
clips **descendants**, corners included. Along the straight edges the square ring
and the rounded clip coincide, so those segments survive; at each corner the
radius carves the ring away. Result: four disconnected lines.

## The trap when fixing it

The obvious move is to give the child the same radius as the parent. That makes it
**worse**: a plain `outline: 2px solid` is painted entirely *outside* the border
box, and since the child fills the parent exactly, the whole ring falls outside the
clip and **disappears**. Verified on a real control card: zero visible ring. *A
control with no visible focus indicator is an a11y regression that looks like a
successful fix.*

## The fix — hang the ring on the rounded container

An element's own `overflow` does not clip its own outline, so put the ring on the
element that owns the radius:

```html
<li class="overflow-hidden rounded-xl
           has-[:focus-visible]:outline-2
           has-[:focus-visible]:outline-offset-2
           has-[:focus-visible]:outline-<color>">
  <button class="… focus-visible:outline-none">
```

Tailwind v4 notes:

- `outline-none` now means `outline-style: none` — the old transparent-ring
  behavior moved to `outline-hidden`.
- `outline-<n>` resolves style through
  `@property --tw-outline-style { initial-value: solid; inherits: false }`, so the
  child's `none` cannot leak up to the parent.
- Confirm the utilities actually **compiled** by grepping the served CSS chunk: a
  typo'd arbitrary variant fails silently.

**When you need the ring on the child exactly** (the child is only part of the
card): use an **inset** ring (`inset-ring-*` / an inset box-shadow), which is
painted inside the border box and therefore survives the clip — but then the child
still needs the matching radius, or you are back to the square-ring defect.

## Cheap arbiter for this class of defect

The visual is the only thing that can judge it. Render a before/after in **one**
page — old class string and new class string side by side, against the app's real
compiled stylesheet — and screenshot it headless:

```bash
brave --headless --disable-gpu --screenshot=out.png --window-size=900,700 file://$PWD/case.html
```

Same measuring code for both halves, and do it in **light and dark**: a ring color
that clears contrast on one surface can die on the other.
