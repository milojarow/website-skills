# Getting indexed — sitemap, robots & Search Console

A site can be built, deployed and live, and still be **invisible to search** because nobody closed this loop. Close it **the day the domain is attached**, not months later.

The work splits into **two separate lists**, and keeping them separate is the point: the agent cannot enter Search Console (it needs the operator's Google account) and the operator should not be touching the repo. Merging both into one checklist is exactly what makes half of it get dropped.

---

## Agent's part — in the repo, nothing needed from the operator

1. **`app/robots.js`** — without it, `/robots.txt` returns 404. A 404 there blocks nothing (it reads as "go ahead"), but it's the only way to declare where the sitemap lives without someone registering it by hand.

   ```js
   export default function robots() {
     return { rules: { userAgent: '*', allow: '/' }, sitemap: `${SITE}/sitemap.xml` };
   }
   ```

2. **`app/sitemap.js`** — generates `/sitemap.xml`. Worth it **even with a single URL**: it's what gets handed to Search Console to request the first crawl, instead of waiting for the site to be discovered through a link that doesn't exist yet.

   - Use `lastModified: new Date()` (build time) rather than a hand-written date nobody will remember to update.
   - `priority` and `changeFrequency` are **crawl hints within the site itself**, not ranking signals. Not worth tuning.

3. **`metadataBase`** in the root layout. Without it, `alternates.canonical` and Open Graph URLs come out relative or broken.

4. **⚠️ If the site ran with `noindex` during development, removing it is NOT enough.** The preview host (`*.vercel.app`) stays alive serving the same HTML: drop the `noindex` alone and you get two copies of the site competing for the same searches. Set **`alternates: { canonical: '/' }`** pointing at the real domain. A declarative tag can't drift out of sync; a per-host `noindex` behind a server conditional can.

5. **Verify against the deployed artifact, not the source**: `/robots.txt` and `/sitemap.xml` must return 200, and the sitemap must list the expected URLs.

6. **If a route was renamed or removed after the sitemap was submitted**, add permanent `redirects()` in `next.config.js`. A 404 throws away whatever that URL had accumulated and teaches the crawler the site loses pages.

7. **Enable Web Analytics on the project** — same class of platform-side pending work as the two lists above, and it belongs in this same pass. Mounting `<Analytics />` does not enable it; until the project switch is flipped, the site records nothing and that history is unrecoverable. See [vercel-web-analytics.md](vercel-web-analytics.md).

---

## Operator's part — in Google Search Console

Hand these over as-is, no jargon:

1. Go to `search.google.com/search-console` → **Add property** → **Domain** (not "URL prefix") → type the bare domain, no `https://`.

2. **DNS verification.** Google asks for a TXT record `google-site-verification=...`.
   - If DNS is on Cloudflare, Google offers to sign in and **add the record itself**. That's the fast path.
   - Otherwise the operator passes the TXT value to the agent and the agent adds it to DNS.
   - **Don't delete that record afterwards** or verification is lost.

3. **Sitemaps** (left menu, under *Indexing*) → paste the **FULL URL**:

   ```
   https://example.com/sitemap.xml
   ```

   **🔴 The mistake that costs time:** on a **Domain** property there is no prefix to fill the gap, so entering the relative path `sitemap.xml` returns **"not a valid path"**. Google's help says it literally: *"copy the URL you tested… paste it into the Add a new sitemap box"*. On a *URL prefix* property the relative path does work — hence the confusion.

   Second detail from the same help page: **"Redirects are not followed"**. If `www` redirects to the apex, submit the **apex**, or the submission fails.

4. **URL Inspection** (top bar) → paste the homepage → **Request indexing**. That puts it in a priority queue instead of waiting its turn.

5. **Report back to the agent whatever "Google-selected canonical" says** in the *Page indexing* section of that same screen. If Google picked the `vercel.app` URL as canonical instead of the real domain, something needs fixing.

---

## Two method traps

**Never assert a site "has never been crawled" by deduction.** Reasoning from a `noindex` history plus zero clicks, and then presenting that as measured, is how you get contradicted by the tool itself reporting **"URL is on Google"** — already indexed. Search Console is the only thing that knows. Same with a favicon missing from the property list: it doesn't mean the favicon is served wrong, it means Google hasn't picked it up from its own crawl yet. Verify that `/favicon.ico`, `/icon.png` and `/apple-icon.png` return 200 — that's the boundary of what the site controls.

**Sitemap and robots are the NECESSARY condition, not the sufficient one.** For a local business, the deciding factor is the **Google Business Profile listing** — the map block sits above the ordinary results. Per Google's documentation, local results are ranked by *relevance, distance and prominence*, and *"If a customer doesn't share where they are, Google uses what it knows about their location"* — i.e. the search localizes itself even when no city is typed. Covering only the sitemap leaves out the biggest lever, and it's free. Send the operator to register the listing in the same step.
