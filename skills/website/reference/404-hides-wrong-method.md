# A 404 doesn't distinguish "route doesn't exist" from "exists, wrong method"

Probing whether an API route is deployed by hitting it with `GET` for convenience conflates two
different facts. Many routers (Hono among them) answer **404 for a method that doesn't match a
registered handler**, not 405 — so a positive control (a route you know doesn't exist, confirmed
404) only proves the *instrument* can find things; it does not prove you asked the *right
question* on the routes that also came back 404.

**Rule: probe existence with the contract's own method, not `GET` for convenience.** Decode the
result like this:

    401 / 403  → route EXISTS (reached its auth layer)
    200        → exists, no session required
    404        → doesn't exist  OR  exists with a different method  ← ambiguous, retry the other verb
    405        → exists, wrong verb (only if the server sends it — many don't)

This is a sound-instrument failure, not a broken one: every control passes, and the wrong
conclusion comes from asking the right tool the wrong question — harder to catch than a broken
instrument because nothing looks off.

See also [permission-without-a-screen.md](permission-without-a-screen.md) for the 401-vs-404
existence probe used to measure permission debt (with its own wrong-verb anecdote), and
[api-boundary-contracts.md](api-boundary-contracts.md) for other API-boundary traps.
