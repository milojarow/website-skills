# A "shine" sweep needs a surface, must sit UNDER the text, and must not cross light text

Three chained defects from mounting a diagonal light sweep (the 90s-anime gloss
reflection) over a tab bar. All three pass lint and typecheck.

## 1. With no resting background, the shine does not exist

White at 92% over a cream at 95% lightness raises it four points: on screen,
**nothing**. The element needs a surface of its own — a sunken tone, ~89% — for
the band to have somewhere to rise to.

Corollary for dark theme: the dark edge that helps in light theme is *surplus*
there — over a near-black surface a black edge is indistinguishable from the
background.

## 2. The shine goes at `z-index: -1` inside an isolated context, not on top

With the pseudo-element above the content, the band **erases the letters it
crosses** (a six-letter label rendered with its middle characters missing).

`isolation: isolate` on the parent + `z-index: -1` on the `::after` puts it above
the background and below the text, which is where a real reflection lives.

## 3. Don't run it over light text

In the hover state the chip is dark and the text near-white: where the white band
crosses the letters, contrast goes to zero. **Lowering the opacity is not
enough** — measured, white at 20% over a mid-green leaves the text at 3.24:1,
under the 4.5 floor. Turn the shine off in that state and move on; hover does not
need it.

## Measuring an animated sweep without guessing

Two separate jobs, two separate techniques:

- **To judge the SHAPE, freeze it** by planting the transform by hand:
  `animation: none !important; transform: translateX(200%) …`.
- **To prove it TRAVELS and STOPS**, correlate the computed `transform` against
  `anim.currentTime % duration` from the Web Animations API.

Sampling blindly every 200ms returns values that look like the animation is
stuck.

## And look at the screenshot at 200%

At real size all three items above look fine; the covered-text defect only
appeared on the enlarged crop.
