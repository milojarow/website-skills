# Draggable card strip: `setPointerCapture` steals the click from your `<a>`s

**Affected pattern:** any draggable carousel / strip / gallery whose cards are
`<Link>` or `<a>`.

**The misleading symptom:** the `href` is correct, the anchor is well-formed, the
destination works if you open it by hand — and clicking the card **navigates
nowhere**. Zero console errors, typecheck green, the anchor visible in the DOM.

## The cause

`setPointerCapture` does not only redirect `pointermove`/`pointerup` to the
capturing element: the browser also delivers the **mouse compatibility events**
to it, `click` included. If the capture is taken in the container's
`pointerdown`, the `click` is dispatched to the CONTAINER, not to the `<a>` under
the cursor. React/Next never see the navigation, and the card's own `onClick`
does not run either.

## Proving it — cheap and binary

Probe in the capture phase on `document`, then perform a **real click**. Not
`el.click()`: a synthetic click is dispatched straight at the anchor and
**passes the test with the bug in place**.

```js
document.addEventListener('click', (e) => {
  console.log(e.target.tagName, e.target.closest('a')?.getAttribute('href') ?? 'NONE');
}, true);
```

- With the bug: `DIV` + `NONE`, and `location.href` never moves.
- Control, one variable changed: `Element.prototype.setPointerCapture = function () {}`,
  then repeat the same click → it lands on the `<img>` inside the anchor and navigates.

## The fix: capture on threshold, never on press

Take the capture only once the travel has crossed the drag threshold:

```js
const onPointerDown = (e) => {
  drag.current = { active: true, fromX: e.clientX, base: pos.current, travel: 0, captured: false };
  // NO setPointerCapture here.
};

const onPointerMove = (e) => {
  const d = drag.current;
  if (!d.active) return;
  d.travel = Math.max(d.travel, Math.abs(e.clientX - d.fromX));
  if (!d.captured && d.travel > THRESHOLD_PX) {   // 6px works well
    e.currentTarget.setPointerCapture(e.pointerId);
    d.captured = true;
  }
  // …move the rail…
};
```

What this buys: the anchor keeps its **native** click — and with it **ctrl+click
and middle-click to open in a new tab**, which a hand-rolled `router.push()` in
`pointerup` would have killed. A real drag still captures, so the rail keeps
following the pointer even outside the strip, and its click is cancelled twice
over: by the threshold guard (`e.preventDefault()`) and by the capture's own
retargeting.

**General rule:** before reimplementing navigation by hand because "the `<Link>`
doesn't respond", check who holds the pointer capture. The anchor is almost never
the culprit.

## Sibling problem: in a strip of BUTTED cards, growing the image inside its frame is invisible

A `group-hover:scale-[1.05]` on the image, with the frame held still, reads as
nothing when the cards touch each other: **the border is the only reference the
eye has**, and the border did not move.

What does read is scaling the WHOLE card (~1.07) with a shadow, riding over its
neighbours. `scale` does not reflow, so nothing else shifts.

Two details that make or break it:

- A container with `overflow-hidden` needs **vertical padding** — at least half
  the growth. For a 352px card at 7% that is ~13px → `py-4`. Without it the grown
  card gets sliced and the effect reads as a crop.
- **`z-index` on hover** (`hover:z-20`, siblings at `z-0`), or the cards later in
  the DOM paint on top and the hovered card grows *underneath* them.
