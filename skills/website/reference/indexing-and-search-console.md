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

   **Derive it from the catalogue, never type it.** The sitemap must read the same array the
   routes read (the one feeding `generateStaticParams`), so a product or article created from
   the admin panel appears by itself. **A hand-written list starts lying at the first record
   added, and it lies silently — nobody opens a sitemap to review it.**

   Corollary of deriving it: **a category with zero items is not sent.** Today it is a page
   whose only content is "nothing here yet"; asking Google to index it is asking it to file a
   notice. With the derived sitemap it enters on its own the day it has a first item.

   What NOT to write, and why:

   - **`lastModified`** — only when a real per-page modification date exists. Using build time
     makes every URL claim the same date and all of them change together on every deploy: the
     pattern crawlers learn to ignore. **An invented datum is worth less than none.**
   - **`priority` and `changeFrequency`** — Google states publicly that it does not use them.
     Emitting them is noise somebody then has to keep up to date for nothing.

   A sitemap of nothing but `<loc>` entries is a correct sitemap.

3. **One canonical host constant**, used by the sitemap **and** by the root layout's
   `metadataBase: new URL(SITE)` + `alternates: { canonical: './' }`. The `'./'` means "this
   same path on the canonical host", so every page emits its own canonical without declaring
   one page by page.

   ⚠️ **`metadataBase` alone emits no canonical tag at all** — it only turns relative URLs
   absolute. The `alternates.canonical` is the part that emits the element. Verify on the
   served HTML, not in the source:

   ```bash
   curl -sS <url> | grep -o '<link rel="canonical"[^>]*>'
   ```

4. **⚠️ If the site ran with `noindex` during development, removing it is NOT enough.** The preview host (`*.vercel.app`) stays alive serving the same HTML: drop the `noindex` alone and you get two copies of the site competing for the same searches. Set **`alternates: { canonical: '/' }`** pointing at the real domain. A declarative tag can't drift out of sync; a per-host `noindex` behind a server conditional can.

5. **Verify against the deployed artifact, not the source**: `/robots.txt` and `/sitemap.xml` must return 200, and the sitemap must list the expected URLs. The one-command version is below.

6. **🔴 `robots.txt` does not prevent indexing — only crawling.** `Disallow` stops the engine
   from *opening* the page; **a blocked URL that somebody links from outside can still show up
   in results**, as a bare line with no description. What actually prevents indexing is
   `robots: { index: false, follow: false }` in that page's metadata.

   And listing private routes in `robots.txt` **is not security** — the file is public and it
   is precisely where you just pointed at them. Routes go there so they aren't crawled, not to
   hide them.

7. **If a route was renamed or removed after the sitemap was submitted**, add permanent `redirects()` in `next.config.js`. A 404 throws away whatever that URL had accumulated and teaches the crawler the site loses pages.

8. **Enable Web Analytics on the project** — same class of platform-side pending work as the two lists above, and it belongs in this same pass. Mounting `<Analytics />` does not enable it; until the project switch is flipped, the site records nothing and that history is unrecoverable. See [vercel-web-analytics.md](vercel-web-analytics.md).

---

## 🔴 Measure whether the apex AND the `www` both answer

```bash
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' https://example.com
curl -sS -o /dev/null -w '%{http_code} %{redirect_url}\n' https://www.example.com
```

Two `200`s with no `redirect_url` means **every page of the site lives at two addresses** —
duplicate content served by the site itself. It is easy to miss precisely because both "work".

The fix has two halves and only one of them is code:

- **Code:** the canonical host constant + `metadataBase` + `alternates.canonical` from the
  agent's list above. That declares which of the two is the real one.
- **Configuration:** a **308 redirect** from the extra host to the canonical one, in the
  hosting provider. Not code — and when the domain belongs to a client, it is not a change to
  make unilaterally; hand it over with the reason.

## Verifying a sitemap in one pass

```bash
curl -sS https://example.com/sitemap.xml > sm.xml
grep -o '<loc>' sm.xml | wc -l                                       # how many URLs
grep -o '<loc>[^<]*</loc>' sm.xml | grep -cE '/(admin|api|…)'        # no private routes
grep -o '<loc>[^<]*</loc>' sm.xml | grep -c 'www\.'                  # one consistent host
grep -o '<loc>[^<]*</loc>' sm.xml | sort | uniq -d | wc -l           # no duplicates
```

⚠️ **Grepping the whole document instead of the `<loc>` elements is a guaranteed false
positive:** the format's mandatory namespace is `http://www.sitemaps.org/schemas/sitemap/0.9`,
so *every* sitemap "contains www" and "contains http://". Measure inside the tag, never across
the document.

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

**Sitemap and robots are the NECESSARY condition, not the sufficient one.** For a local business, the deciding factor is the **Google Business Profile listing** — the map block sits above the ordinary results. Per Google's documentation, local results are ranked by *relevance, distance and prominence*, and *"If a customer doesn't share where they are, Google uses what it knows about their location"* — i.e. the search localizes itself even when no city is typed. Covering only the sitemap leaves out the biggest lever, and it's free. Send the operator to register the listing in the same step — and audit the listing itself: hours, pin, phone and website field decide whether it appears at all. See [google-business-profile.md](google-business-profile.md).
