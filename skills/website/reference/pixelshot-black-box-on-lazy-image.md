# A screenshot tool painting an image black is not evidence the image failed to load

Measured against a Next.js site using `next/image` with `fill` + `loading="lazy"`, served
through the Vercel image optimizer. After a deploy, a headless screenshot tool returned a grid
section with a few cells rendered in near-pure black (measured brightness ~6/255) while every
neighboring cell in the same grid rendered correctly. Cell size and caption were both correct —
only the image was missing. This reads exactly like "the new images aren't loading" and sends
you to debug the site.

## Cold cache is the wrong hypothesis, and it costs a full detour to rule out

The intuitive read — "the freshly-deployed variants are cold" — is false and worth ruling out
fast:

- A genuinely never-requested variant answers `200` with a cache-miss header and real image
  bytes. The optimizer does not 404 on a cold variant: it blocks and serves.
- Warming every default device-size variant for the affected images (all of them landing `200`)
  and re-capturing still reproduced the same black cells.

## The control that actually resolves it: ask the real browser for the `<img>` element's state

```js
// evaluate against the live page
const figs = [...document.querySelectorAll('#section figure')];
figs.map(f => { const i = f.querySelector('img');
  return { complete: i?.complete, nat: i?.naturalWidth + 'x' + i?.naturalHeight,
           src: (i?.currentSrc || '').slice(-70) }; });
```

This returned `complete: true` and a real, non-zero `naturalWidth`/`naturalHeight` for **every**
image, including the ones the screenshot tool painted black. Downloaded, decoded, and a real
browser screenshot shows them fully rendered.

**The defect is in the capture tool, not the site.**

## Rule

A black tile in an automated screenshot is not, by itself, evidence that an image failed to
load. Before touching any code:

1. `curl` the optimizer URL directly and confirm `200` + an `image/*` content type.
2. Ask the browser for `img.complete` and `img.naturalWidth` on the element in question. Both
   truthy means the site is fine and the capture tool lied.

And the inverse, so a real defect doesn't get waved away: if `naturalWidth` comes back `0`,
that's a genuine bug worth fixing.

## A broken instrument can also manufacture a phantom 404 — same lesson, different shell trap

In the same investigation, a probe loop written as `for pair in ...; do set -- $pair; curl
"$url/$2"; done` in zsh produced a 404 on all three test images — including one that visibly
renders. Unlike POSIX `sh`/bash, **zsh does not word-split an unquoted parameter**, so `$2`
arrived empty and the request hit a URL with a missing path segment. The 404 was self-inflicted,
not the site's.

**General alarm:** if a measurement calls something broken that clearly works on screen, suspect
the instrument before the target. In zsh, split a string into words explicitly (`${=var}`) or
use an array — never rely on unquoted word-splitting the way you would in bash.
