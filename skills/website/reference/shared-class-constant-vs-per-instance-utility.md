# A shared class constant with `w-full` beats the width you add next to it

Measured on a Next + Tailwind 3 form. The project had the usual constant shared by
every field:

```js
const input = 'w-full rounded-xl border border-line px-4 py-3 …';
```

and a row with two controls — a country selector plus a phone number:

```jsx
<div className="flex gap-2">
  <select className={`${input} w-28 shrink-0`}>…</select>
  <input  className={input} />
</div>
```

**The `w-28` does not win.** Both classes sit in the attribute, and which one
applies is decided by the **order of the generated stylesheet**, not by the order
you typed them. `w-full` wins, the selector eats the whole row, and the second
control is pushed off screen — clipped, with no horizontal scrollbar to betray it
and no console error.

Measurement: `getBoundingClientRect()` on the input returned `right: 1050` with
`innerWidth: 960`. That is the honest check, because by eye the field simply "is
not there" and the first suspect is conditional rendering, not width.

## The good fix is a wrapping DIV, not `!important`

```jsx
<div className="flex gap-2">
  <div className="w-28 shrink-0"><select className={input}>…</select></div>
  <div className="min-w-0 flex-1"><input className={input} /></div>
</div>
```

The width lives on the container, which competes with nothing, and the shared
class stays intact for every other field. `!w-28` also works, but it leaves a
hidden exception inside a class list somebody will copy to another field without
understanding why it carries the `!`.

## The generalizable rule

When a class constant is shared across many components, **any utility from the
same family (`w-`, `p-`, `text-`) added per instance is a bet on CSS order.** If
the override is layout, put it on the parent. If it is style, pull that property
out of the constant and pass it as a parameter.

Sibling of
[an opacity modifier off the scale](tailwind-opacity-modifier-scale.md): both
share the signature — **the class is written in the DOM and does nothing** — and
both are diagnosed the same way, by reading the computed value off the live
element instead of judging the screen. Related, but a different mechanism:
[an unlayered class beating the whole utilities layer](css-cascade-layers-vs-tailwind.md).
