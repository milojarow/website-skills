# Deploy, errors & CDN

## Builds — never local

Don't run local production builds. **Let Vercel build.** Read build logs with `vercel inspect`. Local builds drift from the Vercel environment and waste time chasing differences that won't exist in production.

## Error handling

For async operations, use `try / catch / finally`:

- **catch:** show a user-friendly message; hide technical details from the UI.
- **finally:** reset loading states here — so a thrown error never leaves a spinner stuck spinning.

## A gitignored file still ships in `vercel deploy` — only `.vercelignore` excludes it

`vercel deploy` from the CLI uploads the **working directory**, not the git tree.
`.gitignore` takes no part in that decision. The only list that removes files from the
upload is **`.vercelignore`**, and when that file does not exist, **everything on disk is
published** — including App Router routes that never entered git.

Measured: a gitignored `app/<route>/` holding a `route.js` and a JSON file of operational
data answered **HTTP 200 publicly** on the deployment domain for a week. It was invisible
to every normal review of the repo, because the three usual reflexes fail *together*:

- `git status` is clean → nothing looks loose.
- `git ls-files` does not list it → it looks like it does not exist.
- Nothing links to the route → it feels private.

And `<meta robots noindex>` / `x-robots-tag` are **not protection**: they stop crawlers,
not somebody holding the URL.

### Two traps while fixing it

- **Adding `.vercelignore` is not neutral hygiene — it IS the decision to take the route
  down**, deferred to the next deploy. If someone asks for "just the `.vercelignore`, to
  be safe" without meaning to remove the route yet, say so: same effect, different name.
- **If the files are still on disk, deploying RE-PUBLISHES them.** An "I already removed
  it" that never touched the checkout turns the deploy into the act that leaks again
  rather than the act that fixes it. Verify the checkout yourself.

### Procedure

1. Back the file up **outside the repo** first — it may be the only copy.
2. Remove the files from the working dir (to the trash, not `rm`).
3. Create/update `.vercelignore`.
4. Rebuild from clean and confirm **the route is absent from the build output**
   (`ls .next/server/app/<route>`) — not that "I deleted it".
5. Sweep the build for the sensitive strings **before** uploading, not after.
6. Deploy.
7. Objective control: the URL must return **404**.

Step 5 is the load-bearing one: a control that runs after the deploy detects the failure
but **does not undo the publication**. For irreversible actions the control goes first,
on the artifact about to be uploaded.

### The leak detector needs its own control

A "ten consecutive digits" regex for leaked phone numbers produces false positives that
read exactly like a real leak — a long numeric id from a third-party URL split into two
halves of ten, a certification number, the business's own public phone.

**Counting matches is not measuring; reading the context of each one is.** The better
control is to take the concrete strings from the sensitive source (the actual names and
numbers in the original file) and search for *those*. It returns 0, or it returns the
exact record — no interpretation in between.

## External CDN — no double hosting

If the project serves media from an external CDN (e.g. `cdn.example.com`), put **`public/` in `.gitignore` from the initial scaffold.**

Committing local copies of CDN-served assets creates **double hosting**: Vercel's CDN ships duplicates while the real CDN serves the actual assets — wasted bandwidth, drift between the two copies, and confusion about the source of truth. The external CDN is the source of truth; the repo must not carry copies.

Note the interaction with the section above: `.gitignore` keeps those copies out of the
**repo**, not out of a **CLI deploy**. If the project is deployed with `vercel deploy`
rather than through the git integration, list them in `.vercelignore` too or the local
copies are uploaded anyway.

If a project already double-hosts, the cleanup is: add `public/` to `.gitignore`, purge the tracked copies from history, force-push, and GC. (Detection + full cleanup procedure live in the operator's notes — this is a heavy, history-rewriting operation, so confirm before running it.)

### The second door: `next/image` re-hosts the CDN's own bytes, and no diff shows it

The check above only catches copies checked into the repo (`git ls-files public/` returns
zero). It misses the more common case in a modern Next.js app: **`next/image` downloads the
file from the external CDN, re-compresses it, and serves it from Vercel's own domain**
(`/_next/image?url=…`). Two hosts end up serving the same image, and one of them is metered.
There is no copy anywhere in the repo — the documented audit comes back clean and the project
is still double-hosting.

It fails **asymmetrically**, which is what makes it expensive to diagnose: old images keep
rendering because they are already in the optimizer's cache; **new ones break**. The exact
work someone just did — uploading a fresh photo — is what shows a broken-image icon, while the
rest of the site looks perfect. The symptom points at the upload, not at the double hosting.

    402 Payment Required
    OPTIMIZED_IMAGE_REQUEST_PAYMENT_REQUIRED

It is not a cold-cache problem, and that hypothesis costs a full detour to rule out: a
genuinely never-requested variant answers `200` with `x-vercel-cache: MISS` and real image
bytes — the optimizer does not 404 on a cold hit, it blocks and serves. Warming every default
`deviceSizes` width for the affected images can come back 100% `200` and the same tiles will
still render black/broken.

Detect it per surface:

    # does any surface request images through the optimizer?
    curl -s https://SITE/PATH | grep -c "_next/image"

    # what does the optimizer answer for one of them?
    curl -s -o /dev/null -w "%{http_code}\n" "https://SITE/_next/image?url=<urlencoded>&w=256&q=75"

A `402` there is the diagnosis. Always compare against the same URL requested straight from
the CDN: if the CDN answers `200` and the optimizer answers `402`, the file itself is fine and
the problem is the double hosting, not the upload.

Fix:

```ts
// next.config.ts
images: {
  unoptimized: true,          // the CDN serves; Vercel does not touch media
  remotePatterns: [ /* … */ ] // keep these — they declare which hosts are allowed
}
```

Free of quality cost when the CDN already serves WebP (check with `curl -sI <url> | grep
content-type`) — the optimizer was just re-compressing an already-compressed file. The real
cost is bandwidth on thumbnails, which is the price of images always rendering and of a third
party losing its vote over whether a client's own merchandise shows up.

The audit question is not "are there copies in the repo?" but **"which host serves the bytes
the browser actually requests?"** With an external CDN the answer must be *only the CDN* — and
there are at least two ways to break that: one visible in `git ls-files`, one visible only in a
rendered `<img>`'s `src`. Both belong in the checklist, with the `grep -c "_next/image"` sweep
run against every deployed surface.
