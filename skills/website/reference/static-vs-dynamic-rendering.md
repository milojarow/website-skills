# Static vs dynamic rendering — one uncached fetch in the layout costs the whole site

Measured on Next.js 15.5 (App Router). Applies to any read that lives in `app/layout.jsx` or in a component the layout renders — header, footer, nav.

## The symptom

Adding one read to the site footer flips routes that never touch the backend — a login screen, a signup screen, an admin screen — from static to dynamic. The build's route table is where it shows:

```
BEFORE                      AFTER
○ /login     2.15 kB        ƒ /login     2.15 kB
○ /signup    1.14 kB        ƒ /signup    1.14 kB
```

The build says so plainly, if you read the full output instead of just the exit code:

```
Route /signup couldn't be rendered statically because it used
revalidate: 0 fetch https://api.example.com/...  /signup
```

## Why

The layout wraps **every** page. A `fetch` with `cache: 'no-store'` (or `revalidate: 0`) inside it marks every route dynamic, because Next cannot prerender a tree containing a read that declares itself always-fresh. It does not matter that the page itself never consumes that data.

The real cost is not the render mode — it is that a slow API now slows down the sign-in screen, which needed none of that data.

## The fix

Not removing the read from the layout — giving **that** read a cache policy matching what it represents. The deciding question is **how often the data actually changes**, not what policy the rest of the project uses:

```js
// The rest of the site reads with no-store: those are figures that change daily.
// This one is not: it is a list that changes about once a year.
const res = await fetch(`${API}/api/board-members`, { next: { revalidate: 3600 } });
```

With `revalidate`, the routes return to `○` and the build reports the window:

```
○ /login   2.15 kB   Revalidate 1h   Expire 1y
```

## The rule that stays

**When you add any read to the layout, diff the build's route table before and after.** A `○` that became `ƒ` is the entire symptom, and that table is the only place it appears: the page works the same, looks the same, and nothing fails.

Corollary for projects with a blanket "nothing is cached" policy: that policy usually has a concrete reason (freshly written data must be visible instantly). Before inheriting it for a new read, check whether the reason applies. If the data changes once a year it does not, and paying for it on every page of the site is pure cost.
