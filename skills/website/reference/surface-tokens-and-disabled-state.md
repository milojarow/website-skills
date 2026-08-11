# A disabled control painted with a surface token disappears on the page that *uses* that surface

**Symptom the user reports:** "this text is out of place" / "why is that sentence
floating there?" They are looking at a disabled button whose fill happens to equal
the page background. It is not dimmed — it is **gone**, and the only thing left on
screen is its label, reading as stray body copy in the middle of a form.

## Mechanism

Design systems name surfaces (`surface`, `surface-raised`, `surface-sunken`). A
disabled style written as `disabled:bg-<surface-sunken>` is implicitly assuming the
page is `surface`. The moment another screen in the same app uses `surface-sunken`
as its **page** background, every disabled control on it renders at **1.00:1**
against the page. Measured on a real product: light theme and dark theme both,
identical hex for fill and page.

Same failure as a fixed brand green for "money in" that reads 8:1 on a light card
and 1.6:1 on a dark one — **a fixed value is only ever correct relative to a
surface someone assumed.**

## Fix

Give the disabled state its own token, chosen so it differs from *every* page
surface in the system, and pair it with a text token:

```css
--control-apagado:        /* a step off every page surface */;   /* light */
--control-apagado-texto:  var(--text-body);                      /* resolves per theme */
```

Targets that worked:

- **~1.3–1.6:1 fill against the page** — a visible slab that does not shout "enabled".
- **≥7:1 for the label.** The label contrast matters more than it looks: on a
  disabled control the label is often the thing explaining *why* it cannot be
  pressed, so `--text-muted` on a muted fill (measured 2.6–3.5:1) is the wrong pair.

Only the **fill** needs a dark-theme value; a `-texto` token defined as
`var(--text-body)` resolves per theme on its own.

**Do not sweep every instance.** A control on a *raised card* with a deliberate
`opacity-40` is a different intent on a different surface — leave it. Swap only the
ones you measured against the page that broke.

## The harness trap that makes you measure the wrong thing

Building a standalone page to compare before/after, putting the page background on
`<body>` with a utility class loses to the design system's own **unlayered**
`body { background: var(--surface) }` (see
[css-cascade-layers-vs-tailwind.md](css-cascade-layers-vs-tailwind.md)). The
"before" button then renders against the *other* screen's background and looks
perfectly visible — contradicting the real screenshot.

1. **Reproduce the real DOM structure, not just the real classes.** If the app
   paints its background on a wrapping `<div>` rather than on `body`, copying only
   the class strings reproduces the styling and not the cascade.
2. **When the harness disagrees with a measurement taken off the user's actual
   screen, the harness is the suspect.** Sampling the user's screenshot gave the
   same hex at four points — inside the control, below it, and outside it. That is
   evidence from an independent path; the mock is not. Fix the mock until its page
   pixel equals the one sampled from reality, *then* trust its before/after.

Sampling pixels straight out of a headless screenshot and computing the ratio in a
few lines is enough — no devtools session required, and it catches "the element is
not there at all", which no screenshot at rest will tell you and no typechecker can.
