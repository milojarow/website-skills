# An opacity modifier off Tailwind's scale generates NO class — and nothing warns

Measured on a Next + Tailwind 3 app, against `getComputedStyle` of the live
element (not read from the docs):

    text-fg/45   → rgba(58, 15, 31, 0.45)   ✅
    text-fg/55   → rgba(58, 15, 31, 0.55)   ✅
    bg-brand/10  → rgba(110, 27, 52, 0.1)   ✅
    bg-brand/12  → rgba(0, 0, 0, 0)         ❌ fully transparent

Tailwind 3's default `opacity` scale steps by 5 (`0, 5, 10, 15, … 100`). Any value
in between — `/12`, `/13`, `/47` — **is not on the scale, so the class is simply
never generated**. The `class` attribute still carries it, there is no warning, no
build error, and the element paints with its default: a completely transparent
fill.

## Why it deceives more than other Tailwind mistakes

A misspelled class (`bg-brnad/10`) looks the same on screen, but you suspect the
name. Here the color **exists**, the pattern is right, the siblings at `/45` and
`/55` do work, and the only sin is that 12 is not a multiple of 5. Real symptom:
avatar circles rendering "invisible" in a list — the letter showed, the fill did
not — and the screen read as unfinished rather than broken.

**Prerequisite for the modifier to work at all:** the color must be declared with
`<alpha-value>` in the config —
`brand: 'rgb(var(--c-brand) / <alpha-value>)'`. Declared as a plain hex, the
opacity modifier does not apply either.

## The two ways out

- Stay on the scale: `/10`, `/15`, `/45`.
- Use the bracket syntax, which accepts any value: `bg-brand/[12%]`,
  `bg-brand/[0.12]`.

## Verifying it in five seconds instead of squinting at the screen

Find the element by its exact class and read the computed color:

```js
const el = [...document.querySelectorAll('*')].find(
  e => typeof e.className === 'string' && e.className.split(/\s+/).includes('bg-brand/12'));
getComputedStyle(el).backgroundColor   // 'rgba(0, 0, 0, 0)' = the class does not exist
```

An `rgba(0,0,0,0)` on an element you gave a background to is **not** "the fill is
very faint" — it is "there is no class". Always read a positive control beside it
(a sibling class that does work), or a zero reads as "the theme didn't load".

Same signature as
[a shared class constant beating a per-instance utility](shared-class-constant-vs-per-instance-utility.md):
the class is written in the DOM and does nothing. Both are diagnosed the same way
— read the computed value off the live element, never the screen.
