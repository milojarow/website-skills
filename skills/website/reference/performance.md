# Performance & images

## Images

- **Format:** WebP
- **Always include size data** (width / height) to prevent layout shift
- **Lazy loading** for below-the-fold images

## Web Vitals

Optimize for:

- **LCP** (Largest Contentful Paint) — server-render the hero, prioritize its image, avoid render-blocking resources.
- **CLS** (Cumulative Layout Shift) — reserve space for images, embeds, and dynamic content (see [layout.md](layout.md)).
- **FID / INP** (input responsiveness) — keep the main thread free; minimize client JS (see [server-first.md](server-first.md)).

Server-first rendering + sized images + reserved layout space are the three levers that move all three metrics at once — they are not separate projects.
