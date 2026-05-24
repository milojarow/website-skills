# Server-first React

Default to React Server Components (RSC) and Next.js SSR. The client bundle is a cost; everything that can render on the server should.

## The discipline

- **Minimize** `'use client'`, `useState`, `useEffect`. They are exceptions you justify, not defaults you reach for.
- `'use client'` **only** for Web API access (`window`, `localStorage`, event listeners) in **small** components — never for data fetching or state management.
- **Data fetching:** server-side in RSCs. Use `swr` for client-side fetching **only when absolutely necessary**.
- **Suspense:** wrap client components in `<Suspense>` with a fallback.
- **Dynamic loading:** `next/dynamic` for non-critical components so they don't block the initial render.

## Why

Server-first cuts the JS shipped to the browser, improves LCP/TTFB, and keeps secrets server-side. Each `'use client'` boundary pulls its whole subtree (and its dependencies) into the client bundle — so push the boundary as far **down** the tree as possible. A single `'use client'` at the top of a page is the costly anti-pattern.

## Patterns

**Isolate a Web-API read into a small client leaf** — keep the chart (and everything else) server-rendered:

```jsx
// page.jsx — Server Component; the chart stays on the server
export default function Page() {
  return (
    <>
      <SalesChart />     {/* server component, no client JS */}
      <ThemeToggle />    {/* small client leaf, below */}
    </>
  )
}
```
```jsx
// ThemeToggle.jsx — the ONLY client part, because it touches localStorage
'use client'
import { useEffect, useState } from 'react'
export function ThemeToggle() {
  const [theme, setTheme] = useState(null)
  useEffect(() => setTheme(localStorage.getItem('theme')), [])
  // ...
}
```
The rule: only the sliver that needs a Web API opts into the client; never lift `'use client'` up to the page to "make it work."

**Fetch data in the RSC itself** — no `useEffect`, no client fetch:

```jsx
// page.jsx — Server Component, fetch runs on the server
export default async function DashboardPage() {
  const data = await getMetrics()         // server-side, secrets safe
  return <CardAnalyticsMain data={data} />
}
```
Reach for `swr` only when the data must refresh client-side after the initial load.
