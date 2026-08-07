# Client logo row built from favicons — normalize them or the block looks broken

Favicons are the easiest source of client logos: every project has one, and it
almost always *is* the logo. But **a favicon is built for a browser tab, not for
a row**, and one detail ruins the block:

**Some of them ship a COLOR PLATE baked into the file** (logo on a black square,
on a yellow square, …) while the rest are transparent. Mixed into one row they
read as color blocks sitting next to floating drawings. In one measured set it
was 3 files out of 11.

## Monochrome does NOT solve this

The general rule — *N logos that fight each other get unified by ONE monochrome
treatment* — fails here: the alpha-channel silhouette of a logo with its own
plate is **a solid disc**. Those three would become blobs.

## What works: a round badge, with the case decided by corner alpha

Sample the corners of the file — exactly where a loose logo never has ink and a
color plate always does:

```python
corner = np.mean([a[2, 2], a[2, w-3], a[h-3, 2], a[h-3, w-3]])   # alpha channel
plated = corner > 200
```

- **`plated`** (ships its own plate) → its plate **is** the badge: `-resize
  400x400^` (cover) plus a circular mask. The black square becomes a black
  circle, which reads as a brand seal.
- **transparent** → padded to 75% over white paper, then the same mask.

Without that split, scaling everything "to fit" leaves the plated ones as
**octagons** — a square with clipped corners, which looks worse than the
original problem.

```bash
magick logo.png -background white -alpha remove -alpha off \
  -resize 400x400^ -gravity center -extent 400x400 \
  \( -size 400x400 xc:none -fill white -draw "circle 200,200 200,2" -alpha copy \) \
  -compose CopyOpacity -composite badge.png
```

## Mounting details

- **Centered `flex-wrap`, not a fixed-column grid.** With 11 items any column
  count leaves a lame last row; wrapping and centering puts the incomplete row
  in the middle.
- **No written name** (the logo carries the identity) **but the name DOES go in
  `alt`** — an image with no alternative text does not exist for a screen
  reader.
- **At 128 px, converted to WebP, they weigh ~3 KB each** (vs ~13 KB as PNG). At
  that size, re-optimizing them through `next/image` adds nothing.

## The part that isn't technical — ask first

A logo identifies a client **as much as or more than** their written name. If a
site generalizes client names for confidentiality and at the same time shows
their logos, the coherent position is: **the relationship is public, the
client's problem is not.** And if any copy says "names are withheld because the
name belongs to them", that sentence becomes false the day the logos land —
rewrite it in the same change.
