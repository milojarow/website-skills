# Compositing a real product photo into a generated interior — perspective is the easy half, light is the one that kills it

For a catalogue or a store hero you often need a real product (the one actually sold) standing in
an attractive interior that does not exist. The generation part is cheap and controllable; the
part that decides whether it ships is the light in the **source photograph**, and that is fixed
before you start.

---

## The technique that works: generate the room AROUND the product

Feed the model the product's **own photograph** and ask it to build the room around it, instead
of generating an interior blind and pasting a cut-out on top. The generated room then inherits
the photo's camera height and vanishing point, and real details from the shot (a door, the floor
pattern) survive into it.

Two refinements that matter:

- **Ask for the room EMPTY.** Generating the room *with* the product inside produces a beautiful
  image in which the model has **redrawn the merchandise** — different proportions, invented
  props. For a catalogue that means publishing a product that is not for sale. Empty room + the
  real cut-out on top keeps 100% of the product pixels authentic.
- **The cut-out does not register pixel-for-pixel with the original** if the background remover
  trims to the bounding box (measured: 1048×1400 out of a 1200×1600 source). Check the dimensions
  before planning an overlay that depends on registration.

---

## The failure mode that kills it anyway

**Perspective is not what betrays a composite. Light is.**

A product photographed in a warehouse carries hard overhead light and its own white balance. The
generated room carries different light. Even with the geometry correct — the piece standing on
the right floor, at the right scale, at the right camera height — the eye reads "pasted" without
being able to name why, and that is exactly the note a client sends back.

There is no prompt fix. Fixing it means **relighting the product photograph**, which is redrawing
it, which is the thing the empty-room trick existed to avoid.

> If the product photos are not studio-lit, do not composite them into generated interiors.

---

## Two dead ends worth not repeating

**A fake contact shadow built from the alpha channel.** Blurring the whole silhouette and
multiplying it under the piece produces a black halo around the *entire* object, including the
parts standing against the wall. It looks worse than no shadow. A real contact shadow needs the
lowest opaque pixel **per column**, not a blurred silhouette.

**An object in FRONT of the scene needs no shadow at all.** In a poster composition — a
photographic panel clipped by a hard diagonal, the product overflowing past the clip onto flat
colour — the product is not *in* the room, it is in front of it. Nobody expects a foreground
object to cast onto a background photograph. That composition sidesteps the shadow problem
entirely.

---

## The cheap arbiter for this class of work

Three steps, and step 3 is a look, not a metric:

1. cut out the product,
2. generate the room from the product's photo,
3. **look at the composite and give it a go / no-go**, then repeat step 1 or 2 for the failures.

Expect a real failure rate at step 3, for reasons that live in the SOURCE photo and no generation
fixes:

- a piece **shot from above** cannot stand in a room seen at eye level — the two perspectives
  cannot coexist;
- a piece carrying a **burned-in watermark or seal** printed on its surface reads as a sticker at
  hero size.

**Check the source photos for camera angle and burned-in marks before spending generations.**

---

## Cost anchor

An image model via OpenRouter runs ~$0.07 per generated room at ~2s; background removal via
`recraft/remove-background` is 1 credit and 2-3s. Ten operations for a multi-slide hero stay
under $0.50 — **cost is not the constraint here, the light mismatch is.**
