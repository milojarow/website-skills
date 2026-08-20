# A categorical palette validated for MARKS fails as small TEXT — and reusing it looks reasonable

A project already carries six categorical colors in its CSS tokens, generated and validated
for **charts on a light surface**. A new feature needs "a color per user" to sign names in the
UI. Reusing the six is the obvious move: they exist, they are validated, and the token is
labelled *validated*.

Measured before proposing them: **four of the six failed.**

## The floor depends on the ROLE of the color, not on the color

| role | WCAG floor |
|---|---|
| chart mark (bar, dot, line) | 3.0 |
| small text (< 18 px, or < 14 px bold) | 4.5 |
| large text | 3.0 |

A username at 13–14 px is small text. Real measurement of a categorical palette that **did**
pass the chart validator, used as a text color:

```
color          on #fafbf9   #f3f6f2   #ecf0ec
verdigris          3.81       3.63      3.43   ← fails
amber              3.65       3.48      3.29   ← fails
blue               4.70       4.48      4.24   ← fails on 2 of 3
olive              4.70       4.48      4.24   ← fails on 2 of 3
terracotta         4.88       4.65      4.40
violet             6.18       5.89      5.57
```

The detail that multiplies the error: **measure against the DARKEST surface the color can land
on**, not against the page background. Blue and olive passed on the page and failed on the
raised surfaces — the label would have been correct in one place and wrong in another, which is
worse than being wrong everywhere, because the screen that proves it is the one nobody opens.

## Deriving the text variant without losing the color's identity

**Darken by MULTIPLYING the channel, never by subtracting.** Subtracting desaturates, and a
washed amber stops reading as amber; multiplying preserves the ratio between channels, i.e. the
hue.

```python
def darken(hex_src, target=4.60, bg=DARKEST_SURFACE):
    base = hx2rgb(hex_src); k = 1.0
    while k > 0.05:
        cand = tuple(c * k for c in base)
        if ratio(cand, bg) >= target:
            return rgb2hx(cand)
        k -= 0.005
```

On the palette above all six landed at 4.59–5.57 against the darkest surface, with the **hue
preserved exactly** (worst drift 171°→172°, which is rounding) and the closest pair still ΔE
27.8 apart (practical floor ≈ 15).

## Two rules worth writing next to the values

1. **The contrast table goes as a comment beside the hexes.** A value with no measurement next
   to it invites the next person to nudge it by eye. And the line to write is not the number
   alone — it is **why the floor is 4.5 and not 3.0**: because this is text, not a mark. That is
   precisely the mistake the next arrival is going to make.
2. **A user/category color is not stored as a hex in the database — store the SLUG**, and keep
   the hex list in exactly one place. A copy in the front to paint and another in the backend to
   validate are two truths about the same datum and they drift. Corollary: whoever validates the
   slug rejects anything off the list with `400` instead of storing free text — an unknown slug
   leaves the consumer with no hex to paint, and the result reads as a render bug.

## Bonus: a free color picker breaks the UI's semantic code

If the interface already means red = danger and amber = warning, a free picker lets someone
choose a name that reads as an error on every row. Use a closed palette, and leave out the hues
that already mean something.
