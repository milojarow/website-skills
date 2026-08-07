# Page watermark: `z-index:-1` almost never works — `mix-blend-multiply` on top does

**The misleading symptom:** you put a fixed watermark behind the content with
`z-index:-1`, it looks perfect in one section and vanishes in the rest. It looks
intermittent or broken. It is neither: every section with an OPAQUE `bg-*`
covers it. On a typical landing page that is almost every section.

**30-second check, BEFORE choosing the technique** (not after mounting it):

```bash
for f in components/Section*.tsx; do
  printf "%-30s %s\n" "$(basename $f)" \
    "$(grep -oE 'className="[^"]*\bbg-[a-z-]+' "$f" | head -1 | grep -oE 'bg-[a-z-]+')"
done
```

If most of them carry a background, a layer behind the content is useless.

## The way out, when the surfaces are nearly the same color

In light-theme design systems it is normal for `--bg`, `--sunken`, `--surface`
and white to sit a few RGB units apart. When that holds, **a single fixed layer
with `mix-blend-mode: multiply` ON TOP of the content tints them all equally**:

```jsx
<div aria-hidden className="pointer-events-none fixed inset-0 z-[1]
                            overflow-hidden mix-blend-multiply">
  <Mark className="… opacity-[0.06]" />
</div>
```

Multiply darkens proportionally: the paper takes the tint and the text, already
dark, barely moves. One element, zero changes to the sections.

**The z-index is the part to think about.** Sections usually declare no
z-index, so a fixed `z-1` covers them. The header (`z-50`) and a floating chat
widget paint afterwards and stay OUTSIDE the tint — which is what you want: a
watermark that darkens the header logo and the CTA button reads as a filter over
the site, not as paper.

**And if the hero has a background video, hide it there:** multiply over video
dirties it. Use an `IntersectionObserver` on the hero, not a scroll listener.

## Verifying the opacity — measure it, don't eyeball it

The referee is the WCAG contrast of the text on top, and the worst case is body
text (the lightest color actually used for reading) landing right ON a stroke of
the mark — not an average. In one measured case: at opacity 0.09 it scored
**4.49:1**, one hundredth below the 4.5 floor. One hundredth below is below. At
0.06 it scored 4.67:1.

**How to measure it without being lied to: two screenshots of the SAME frame,
layer on and layer off, then subtract.** That isolates the mark's exact
contribution.

An earlier attempt sampled "the paper" as the lightest percentile of a region
and "the mark" as the darkest — and picked up **card borders and section
dividers** as if they were tinted paper. It reported 3.85:1 and claimed opacity
0.09 and 0.06 were nearly identical, which is impossible, and that was the clue
that the meter was wrong, not the design. **If two different settings produce
the same number, the suspect is the measurement.**

```python
d = without_layer.astype(float) - with_layer.astype(float)   # isolated contribution
worst_delta = d.reshape(-1, 3)[d.mean(axis=2).reshape(-1).argmax()]
# then: contrast(theme_paper - worst_delta, text_color) >= 4.5
```
