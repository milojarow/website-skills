# Shipping the front ahead of its backend — the capability gate

Recurring whenever the front and the backend are **two separate deployments** (the site on
Vercel, the API somewhere else): the front is finished and the backend is waiting on a
maintenance window, an approval, or simply another person. The temptation is to hold both.
Waiting has a real cost — the operator cannot see their own feature.

You *can* ship the front first, but **"it doesn't crash" is not the bar**. The expensive
failure mode is not an error, it is **a control that lies about what it saved**.

Measured: a post composer let the author pick a type (`proposal` · `notice` · `job`). Against a
backend that did not yet know the column, the author picked "Proposal", the request returned
**200**, the post appeared on screen — and it was stored as "job". **The only thing that failed
was the only thing the author cared about**, with no error anywhere: no console, no toast, no
log. Same family as [an instrument that answers a different question](client-clock-vs-server-deadline.md):
all green, wrong data.

## The pattern: a flag derived from the RESPONSE

Not from an environment variable, not from a date — from what the server actually sent back:

```js
// in the data client
newContract: hasKey(body, "reactions")
```

and every new control hangs off it:

```jsx
{withTypes && <TypeSelector … />}
...
body: JSON.stringify({
  ...theUsual,
  ...(withTypes ? { type, idem_key } : {}),   // the new fields do NOT travel
})
```

With the flag `false`, the old backend receives **exactly** what it received before the feature
existed. Zero new surface.

## 🔴 Probe the PRESENCE of the key, never `.length`

This is the nuance that makes it work, and it is easy to get wrong:

- `response.reactions.length > 0` → **switches itself off** the day someone removes the last
  reaction, and the empty list is the normal state at launch. The feature would vanish for no
  apparent reason.
- `"reactions" in response` → an honest binary. **Empty** means "nobody has reacted";
  **absent** means "this server does not know what a reaction is."

After the response is flattened the two look identical, so **the check must happen BEFORE
normalizing**.

Also confirm with the backend that the key **always** travels: if that route can return a
partial object, the signal is garbage. In the measured case all the queries went out in a
single `Promise.all` and any failure took the whole route down with a 500 — never half an
object.

## Three rules that complete it

1. **Say on screen what is missing, in user language.** Removing a control without explaining
   it reads as "the feature I was promised does not exist." One line is enough: *"for now
   everything is filed as X; marking it as Y arrives when the server is updated — until then we
   don't offer it, so you don't pick something that won't be saved."*
2. **The gate turns itself on.** It is not a to-do and not a flag someone has to remember to
   remove: the backend's own deploy makes the signal true. A `TODO: remove this` survives for
   years; a condition does not.
3. **A server error is painted with the SERVER'S TEXT.** If the backend answers
   `400 {error, values}`, that `error` goes straight to the screen. A homemade sentence
   describing the other side's rule goes stale the moment the rule changes, and a stale
   explanation is worse than none. See [user-facing-error-copy.md](user-facing-error-copy.md).

## Distinguish 401 from 404 on any route that may not exist yet

Corollary of the same deploy, and an expensive silent one. An `if (!res.ok) return null` over a
route that is not deployed yet makes the screen say **"your session expired"**: the user signs
out, retypes their password, signs in fine, and sees the exact same message. An error that
blames them for something they did not do, in a loop.

```js
if (res.ok)             return { source: "backend", … };
if (res.status === 401) return { source: "no-session" };   // fixed by signing in again
return { source: "no-backend" };                           // not fixable from the browser
```

The two cases are worded differently **because the user can do something different about each
one**.
