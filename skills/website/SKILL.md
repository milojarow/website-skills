---
name: website
description: Use when building or editing a website or web app on the DeployNow stack — Next.js App Router, React, Tailwind + DaisyUI, NextAuth, deployed on Vercel. Use when scaffolding a page/component/feature, naming files or components, deciding server vs client components, fixing a broken or cramped layout, optimizing images or Web Vitals, handling async errors, or deploying. Not for non-web code, native/mobile apps, or generic React unrelated to this stack.
---

# website

Build production-grade, distinctive Next.js websites the DeployNow way — server-first, layout-stable, deployed on Vercel.

> **🌐 ACTIVE-SKILL MARKER:** While `website` is active, begin every reply with 🌐 so the operator sees at a glance that this skill is engaged. Do not omit it.

## Overview

The stack is **DeployNow**: JavaScript, Node.js, React, Next.js App Router, Tailwind CSS + DaisyUI, NextAuth, deployed on Vercel. Two principles dominate everything else:

- **Server-first.** Default to React Server Components and SSR. `'use client'`, `useState`, `useEffect` are exceptions you justify, not defaults you reach for. See [reference/server-first.md](reference/server-first.md).
- **Layout stability.** Dynamic content must never displace its siblings, and a broken layout is almost always an *ancestor* constraint — fix the parent, never patch the leaf. See [reference/layout.md](reference/layout.md).

Design is **production-grade and distinctive** — never cookie-cutter. Mobile-first, responsive, ready to ship.

## When to use

- Building a page, component, or feature for a website / web app.
- Naming files or components, or laying out the file structure.
- Deciding server vs client components, data fetching, Suspense.
- Fixing a broken, cramped, or misaligned layout.
- Optimizing images or Web Vitals; handling async errors; deploying to Vercel.

**Not for:** non-web code, native/mobile apps, or generic React questions unrelated to this stack.

## Where things live

| Task | Reference |
|---|---|
| Stack, file structure, file/component naming, pure functions, JSX unicode gotcha | [reference/stack-and-conventions.md](reference/stack-and-conventions.md) |
| Server vs client, RSC/SSR, `'use client'` rules, `swr`, Suspense, code skeletons | [reference/server-first.md](reference/server-first.md) |
| Layout stability + the trace-upward debugging method | [reference/layout.md](reference/layout.md) |
| WebP images, lazy loading, Web Vitals (LCP/CLS/FID) | [reference/performance.md](reference/performance.md) |
| Vercel deploy, error handling, external-CDN / no double-hosting | [reference/deploy.md](reference/deploy.md) |

## Quick reference

- **Components:** PascalCase, prefixed by type — `ButtonAccount.jsx`, `CardAnalyticsMain.jsx`. Other files: kebab-case. Dirs: kebab-case.
- **Inside a component file:** exported component → subcomponents → helpers → static content.
- **Pure functions:** `function` keyword; concise conditionals without unnecessary braces.
- **Client code:** `'use client'` only for Web API access in small components — never for data fetching or state.
- **Text in JSX:** type real accented chars (á, é, í, ñ, ¿, ¡) directly — a backslash-u unicode escape renders as its literal characters between tags, not as the letter.
- **Builds:** never build locally — let Vercel build; read logs with `vercel inspect`.

## Common mistakes

- Reaching for `'use client'` + `useState`/`useEffect` to fetch data → use an RSC + server fetch instead.
- Patching a leaf component to fix empty space or cramping → the cap is in an ancestor (page → layout → root → global CSS). Trace upward first.
- Committing `public/` while serving media from an external CDN → double-hosting. `public/` goes in `.gitignore` from the first scaffold.
- Using a backslash-u unicode escape in JSX text expecting an accented letter → it renders the raw escape characters; type the real char.
