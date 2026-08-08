---
name: website
description: Use when building or editing a website or web app on the DeployNow stack — Next.js App Router, React, Tailwind + DaisyUI, NextAuth, deployed on Vercel. Use when scaffolding a page/component/feature, naming files or components, deciding server vs client components, fixing a broken or cramped layout, optimizing images or Web Vitals, handling async errors, or deploying. Not for non-web code, native/mobile apps, or generic React unrelated to this stack.
---

# website

Build production-grade, distinctive Next.js websites the DeployNow way — server-first, layout-stable, deployed on Vercel.

> **🌐 ACTIVE-SKILL MARKER:** Prefix your reply with 🌐 **only on turns where the work touches the `website` domain** — the Next.js + Tailwind/DeployNow web stack — regardless of the layer/project (frontend, backend, a local script — all count); what matters is whether *this turn* touches the domain. On turns that do NOT touch it (typecheck, build, deploy, git ops, editing or curl in other domains), **omit 🌐** even if the skill loaded earlier in the session. If other active skills also apply to the same turn, **stack their emojis** in the prefix.

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
| Layout stability + the trace-upward debugging method; why `width:fit-content` on a flex row collapses instead of filling, and the `--n` count + `min()`/`calc()` fix | [reference/layout.md](reference/layout.md) |
| WebP images, lazy loading, Web Vitals (LCP/CLS/FID) | [reference/performance.md](reference/performance.md) |
| Static vs dynamic rendering: how one uncached fetch in the layout turns every route dynamic, reading the build's route table, choosing a cache policy per read | [reference/static-vs-dynamic-rendering.md](reference/static-vs-dynamic-rendering.md) |
| Vercel deploy, error handling, external-CDN / no double-hosting | [reference/deploy.md](reference/deploy.md) |
| Third-party scripts (pixels/analytics/widgets) under a strict CSP — silent dead-pixel trap + post-deploy verification | [reference/third-party-scripts-csp.md](reference/third-party-scripts-csp.md) |
| Getting indexed: `robots.js`/`sitemap.js`, canonical vs preview host, Search Console steps for the operator | [reference/indexing-and-search-console.md](reference/indexing-and-search-console.md) |
| Vercel Web Analytics: the per-project switch that `<Analytics />` does not flip, diagnosing it, free-tier event limits | [reference/vercel-web-analytics.md](reference/vercel-web-analytics.md) |
| Local business: auditing the Google Business Profile listing with the Places API, why declared hours gate the local block, owner vs manager access | [reference/google-business-profile.md](reference/google-business-profile.md) |
| Brand searches: getting the brand string into `<title>`/`<h1>`, the three distinct name constants, title character budget | [reference/brand-in-title.md](reference/brand-in-title.md) |
| Mobile-first: tab rows → scroll horizontal, grid shrink, tap targets, density | [reference/mobile-design.md](reference/mobile-design.md) |
| Client logo row from favicons: baked color plates vs transparent, round-badge normalization, wrap/centering, `alt` and the confidentiality question | [reference/client-logo-row.md](reference/client-logo-row.md) |
| Page-wide watermark: why a layer behind opaque sections disappears, the `mix-blend-multiply` overlay, z-index budget, measuring the opacity against WCAG contrast | [reference/page-watermark.md](reference/page-watermark.md) |
| PNG→SVG line-art tracing: why the diff-ratio metric is noise, the ring/cores referee, potrace flags, baking the inverted-Y transform, one `<defs>` + `<use>` | [reference/svg-line-art-tracing.md](reference/svg-line-art-tracing.md) |
| API boundary: sending identity fields explicitly, NFC vs NFD exact-match failures, typing the response you measured instead of the one that was documented | [reference/api-boundary-contracts.md](reference/api-boundary-contracts.md) |
| Dates: a Postgres `date` arriving as midnight UTC and shifting every row a day, the noon-UTC anchor, calendar days vs instants as separate types, ingesting an external export whose split date/time columns destroyed the offset | [reference/calendar-days-vs-instants.md](reference/calendar-days-vs-instants.md) |
| Money screens: never mixing currencies, summing in integer cents, committed vs collected, showing gaps, grouper vs entity, and captions read as assertions | [reference/money-dashboard.md](reference/money-dashboard.md) |

## Quick reference

- **Components:** PascalCase, prefixed by type — `ButtonAccount.jsx`, `CardAnalyticsMain.jsx`. Other files: kebab-case. Dirs: kebab-case.
- **Inside a component file:** exported component → subcomponents → helpers → static content.
- **Pure functions:** `function` keyword; concise conditionals without unnecessary braces.
- **Client code:** `'use client'` only for Web API access in small components — never for data fetching or state.
- **Text in JSX:** type real accented chars (á, é, í, ñ, ¿, ¡) directly — a backslash-u unicode escape renders as its literal characters between tags, not as the letter.
- **Builds:** never build locally — let Vercel build; read logs with `vercel inspect`.
- **Mobile tab rows:** una fila de tabs/pills que no cabe → una sola línea con scroll horizontal (`flex-nowrap overflow-x-auto min-w-0` + `shrink-0 whitespace-nowrap` en items), **nunca** `flex-wrap` apilado.

## Common mistakes

- Reaching for `'use client'` + `useState`/`useEffect` to fetch data → use an RSC + server fetch instead.
- Patching a leaf component to fix empty space or cramping → the cap is in an ancestor (page → layout → root → global CSS). Trace upward first.
- Sizing a flex row with `width:fit-content` (+ `max-width:100%`) so it "fills when there are many children" → with `min-width:0` children it collapses to about half the available width, and without the `max-width` it overflows and hides children behind a horizontal scrollbar. Pass the child count as a custom property and use `width: min(100%, calc(...))`. Verify by screenshotting at the target viewport width, not by reading the CSS. See [reference/layout.md](reference/layout.md).
- Committing `public/` while serving media from an external CDN → double-hosting. `public/` goes in `.gitignore` from the first scaffold.
- Using a backslash-u unicode escape in JSX text expecting an accented letter → it renders the raw escape characters; type the real char.
- Apilar tabs/pills con `flex-wrap` en móvil cuando no caben → usa una sola línea con scroll horizontal (`overflow-x-auto` + `min-w-0`). Ver [reference/mobile-design.md](reference/mobile-design.md).
- Adding a third-party pixel/analytics script to a site with a strict CSP without allow-listing its CDN → the loader is blocked *silently* (inline stub queues events that never send). Allow-list `script-src`/`connect-src` and verify the event reaches the vendor dashboard. See [reference/third-party-scripts-csp.md](reference/third-party-scripts-csp.md).
- Mounting `<Analytics />` and calling analytics done → the per-project switch is **off by default** and the component does not flip it; the site records 0 visitors and that history is unrecoverable. Enable it in the same step that takes the site to production. See [reference/vercel-web-analytics.md](reference/vercel-web-analytics.md).
- Ending every `<title>` in the brand's *abbreviation* and writing a purely descriptive `<h1>` → the full brand string then exists in no `<title>` and no `<h1>` on the site, and the short brand search is lost while the long one still works. Keep a `titleBrand` constant separate from `shortName`. See [reference/brand-in-title.md](reference/brand-in-title.md).
- Shipping a local-business site and calling search done at the sitemap → the Google Business Profile listing sits above the organic results, and a listing with **no declared hours** is dropped by the "Open now" filter entirely (reviews are not the gate; hours behave like it). Audit hours, pin, phone, website field and photos. See [reference/google-business-profile.md](reference/google-business-profile.md).
- Placing a page watermark behind the content with `z-index:-1` → every section with an opaque `bg-*` covers it and it looks intermittent. Use one fixed `mix-blend-multiply` layer above the content, and measure its opacity by subtracting two screenshots rather than eyeballing it. See [reference/page-watermark.md](reference/page-watermark.md).
- Judging a vectorized line-art logo by `difference / ink pixels` → a perfect trace already spends ~4% on antialias halo, so every candidate looks equally bad. Measure difference *outside* a 1 px contour ring, and count solid cores. Validate the referee against a deliberately broken SVG before trusting it. See [reference/svg-line-art-tracing.md](reference/svg-line-art-tracing.md).
- Dropping favicons straight into a client-logo row, or unifying them with a monochrome treatment → the ones with a color plate baked in read as blocks next to floating drawings, and monochrome turns them into solid discs. Split on corner alpha and normalize both cases into the same round badge. See [reference/client-logo-row.md](reference/client-logo-row.md).
- Omitting an identity field (`author`, `owner`) from a POST because you assume the backend derives it from the token → if it validates the field instead, 100% of requests 400. Sending it costs nothing and works in both worlds. And never hand-type or hardcode a name you compare by equality — accented values have two byte encodings (NFC/NFD) that look identical on screen; pass the session value verbatim. See [reference/api-boundary-contracts.md](reference/api-boundary-contracts.md).
- Passing a Postgres `date` value straight into `new Date()` and formatting it with a `timeZone` → it arrives as midnight UTC and every row renders one day early, with no error anywhere. Slice to 10 chars and anchor to noon UTC, and keep calendar days and instants in separate, differently-named functions. See [reference/calendar-days-vs-instants.md](reference/calendar-days-vs-instants.md).
- Ingesting the "for humans" export of an external system (separate `Fecha` / `Hora inicio` columns) instead of its ISO-with-offset export → a row whose offset was already wrong becomes **permanently** wrong, because the file kept the broken rendering and dropped the zone that proved it (measured: +13 h on a real appointment). Prefer the offset-bearing export, let it win any merge, compute the expected offset for that date, and crash on an unrecognized schema. See [reference/calendar-days-vs-instants.md](reference/calendar-days-vs-instants.md).
- Writing a TypeScript type from the written API contract instead of from a measured response → a field the endpoint never returns becomes a compiler-endorsed `undefined` at runtime, which is worse than a crash. Measure first. See [reference/api-boundary-contracts.md](reference/api-boundary-contracts.md).
- Totalling money in JavaScript floats, or collapsing "what customers should pay" and "what actually came in" into one figure → the header stops matching the table underneath it and the panel looks healthier than the business is. Sum in integer cents, send amounts as strings, and keep committed and collected as two numbers. See [reference/money-dashboard.md](reference/money-dashboard.md).
- Treating an explanatory caption on a money screen as decoration → the grouping code can be right while the note under it says something false, and no typecheck, linter, or unit test catches it. Read every caption as an assertion about the code in front of you. See [reference/money-dashboard.md](reference/money-dashboard.md).
- Adding a `no-store` / `revalidate: 0` fetch to `app/layout.jsx` (or to the header/footer it renders) → the layout wraps every page, so **every** route flips from static `○` to dynamic `ƒ`, and a slow API now slows down screens that never used that data. Give that read a `revalidate` matching how often the data really changes, and diff the build's route table before/after. See [reference/static-vs-dynamic-rendering.md](reference/static-vs-dynamic-rendering.md).
- Attaching the domain without shipping `app/robots.js` + `app/sitemap.js` and handing the operator the Search Console steps → the site is live and invisible. Also: dropping a development `noindex` without setting `alternates: { canonical: '/' }` leaves the `*.vercel.app` preview competing with the real domain. See [reference/indexing-and-search-console.md](reference/indexing-and-search-console.md).
