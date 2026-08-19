# A fullPage screenshot CANNOT validate a scroll-driven reveal

## The defect it hides

The increasingly common way to reveal content on scroll **without JS or a library**:

```css
@supports (animation-timeline: view()) {
  .reveal {
    animation: rise 1s linear both;
    animation-timeline: view();
    animation-range: entry 5% cover 26%;
  }
}
```

The keyframe starts at `opacity: 0`. If an element **never reaches the range**, it
stays at opacity 0 **forever**: content present in the DOM, invisible on screen, not a
single console error. The natural candidates are the last block of the page and
anything inside a container whose scroller is not the document's.

## Why the screenshot says everything is fine

A `fullPage` capture over CDP (Playwright `fullPage: true`, DevTools "Capture full
size screenshot", any pixel-diff tool built on them) **resizes the viewport to the
full document height** before capturing. With that, *every* element is inside the
view, the `view()` timeline resolves to progress 1, and **everything renders at
opacity 1** — including what a real user would never see.

Textbook case of measuring the bug with the bug: the instrument changes exactly the
variable the defect depends on.

## The measurement that works

**Real** viewport, **no scrolling**, and read the computed opacity:

```js
// viewport pinned to 1440x900 (or 390x844), navigate, do NOT scroll
() => {
  const supported = CSS.supports('animation-timeline', 'view()');
  const stuck = [...document.querySelectorAll('.reveal')]
    .map((el) => {
      const r = el.getBoundingClientRect();
      return {
        txt: (el.textContent || '').trim().slice(0, 30),
        opacity: +getComputedStyle(el).opacity,
        onScreen: r.top < innerHeight && r.bottom > 0,
      };
    })
    .filter((x) => x.onScreen && x.opacity < 0.99);
  return { supported, stuck };   // empty stuck = pass
}
```

**Mandatory positive control, or the zero is worthless:** first confirm that
`CSS.supports('animation-timeline','view()')` returns `true`. If it returns `false`,
the rule lives inside the `@supports` block and was never applied — a `stuck: []`
there only proves the browser lacks the feature, not that the animation is correct.

Second pass, the one that catches the real case: scroll to the **end** of the document
and read again. An element at the foot that never reaches `cover 26%` sits at partial
opacity, and that is precisely the one a fullPage capture will never report.

## The rule

`@supports` around the rule is mandatory (a browser without support shows the content
as-is, never hidden), and all of it inside
`@media (prefers-reduced-motion: no-preference)`. But **neither guard protects against
the stuck element in a browser that does support it** — only reading the computed
opacity in a real viewport catches that.

Applies identically to `IntersectionObserver` + a `.visible` class: if the observer
never fires the element stays hidden, and the fullPage capture shows it visible
because everything came into view.

## While you are in there: measure the width in the same run

Read `documentElement.scrollWidth > clientWidth` on the same pass. A row that
overflows widens the whole **document**, and a screenshot at offset 0 looks identical
with and without the defect. Measure it, don't look at it — see
[overflowing-row-scroll-container.md](overflowing-row-scroll-container.md).
