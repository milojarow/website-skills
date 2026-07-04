# Third-party scripts under a strict CSP — the silent dead-pixel trap

When the site ships a strict Content Security Policy, **any third-party script whose CDN domain is not allow-listed is blocked by the browser — silently.** Pixels, analytics, and chat widgets are the classic victims: the code looks live in production, but no events ever leave the page.

## Why it's silent (the treacherous part)

Almost every analytics/pixel snippet has **two parts**:

1. **Inline bootstrap** — defines a stub (`window.fbq`, `window.dataLayer.push`, `window.gtag`, …) that *enqueues* calls. It runs fine, because `'unsafe-inline'` in `script-src` allows it.
2. **External loader** (`fbevents.js`, `gtm.js`, `analytics.js`, …) — the script that drains the queue and actually sends to the vendor. **This** is what a CSP without the vendor's CDN domain blocks.

Result: the stub exists, `fbq('track', 'Lead')` doesn't throw, app code believes everything works, events pile up in the queue… and die there. **Zero errors in app code.** The only signals are the external request showing `[pending]`/blocked in Network, and (sometimes) a CSP violation in the console.

## Checklist when adding ANY third-party script (pixel, analytics, chat widget)

1. **Before writing code**, check for an existing CSP:
   `curl -sI https://the-site.example | grep -i content-security-policy`
   If a CSP is present, the new script needs its domains explicitly allowed.
2. **Domains a pixel/tag typically needs:**
   - `script-src` — the CDN that serves the script (e.g. `https://connect.facebook.net`, `https://www.googletagmanager.com`).
   - `connect-src` — where it *sends* events, since delivery goes through `sendBeacon`/`fetch`, which `connect-src` governs (e.g. `https://www.facebook.com`, `https://www.google-analytics.com`).
   - `img-src` — 1×1 image fallback (a generic `https:` usually covers it).
3. **Post-deploy verification — NEVER skip it** (real or headless browser):
   - Network: the external script returns **200** (not `[pending]`, not blocked).
   - The event actually leaves: an outgoing request to the vendor's collector (e.g. `facebook.com/tr?...`, GA `/collect`).
   - **Meta Pixel in-page probe:**
     ```js
     ({
       callMethodDefined: typeof window.fbq?.callMethod === 'function', // true = real script loaded
       queueLength: window.fbq?.queue?.length  // >0 after load = events STUCK, script blocked
     })
     ```
     `callMethod` is only defined by the real `fbevents.js`; if it's `undefined` and the queue has entries, the CSP (or something else) is blocking the loader.
4. **"The code is in production" ≠ "the pixel works."** Verification is the event arriving at the vendor dashboard (Meta Events Manager, GA Realtime), or at minimum the outgoing request visible in Network.

## CSP fix shape (`next.config.mjs` headers)

```js
"script-src 'self' 'unsafe-inline' ... https://connect.facebook.net",
"connect-src 'self' ... https://connect.facebook.net https://www.facebook.com",
```

## Independent chains verify independently

A lead reaching the CRM does **not** prove the pixel fired, and vice versa — they are separate delivery chains. Marking a "verify pixel in Events Manager" checkpoint as done because a *different* chain (CRM/email) was validated is how a dead pixel survives for weeks. One independent chain = one independent verification, each confirmed at its own destination.
