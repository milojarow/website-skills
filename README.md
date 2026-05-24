# website-skills

Milo's house conventions for building production-grade **Next.js + Tailwind** websites on the **DeployNow** stack, packaged as an installable Claude Code skill.

## What it covers

One skill, `website`, with progressive-disclosure reference files:

- **stack-and-conventions** — DeployNow stack, file structure, file/component naming, pure functions.
- **server-first** — RSC/SSR default, `'use client'` discipline, `swr`, Suspense, dynamic loading, the JSX unicode gotcha.
- **layout** — layout stability + the trace-upward debugging method.
- **performance** — WebP images, lazy loading, Web Vitals (LCP/CLS/FID).
- **deploy** — Vercel builds, async error handling, external-CDN / no double-hosting.

## Install

```
/plugin marketplace add milojarow/website-skills
/plugin install website-skills
```

## Active-skill marker

While `website` is engaged, replies are prefixed with 🌐.

## License

MIT
