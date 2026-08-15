# PocketBase: a public form that must stay re-editable cannot depend on listing its own row

The shape: a single-row public page — a questionnaire the visitor answers a bit at a
time and can come back to edit. There is no account for that person, and putting a
login in front of a dozen questions is friction that kills the exercise. The correct
rules are therefore `create` and `update` public, with `list` / `view` / `delete`
closed to staff.

## The trap

The natural way to locate the row is a filter:

```
GET /api/collections/<col>/records?filter=key="<secret>"
```

That read **requires `list` to be open**. With `list` closed — which is the correct
setting — it is refused, the id is never obtained, and the symptom appears only on the
**second** visit: the first time, the `POST` creates the row and everything works; on
reload the id is gone, the code retries the `POST`, and the unique index on the key
field rejects it.

So the visitor can answer **once and never correct themselves** — precisely the inverse
of the requirement. It passes the build, and it passes a manual test done in a single
session.

## The way out: a deterministic id

PocketBase accepts an explicit `id` on create (15 alphanumeric characters). Derive it
from the key and burn it into the client:

```js
const ID = '<15-char-id>';   // fixed, known in advance
```

```
PATCH /records/<ID>   →  ok     saved
                      →  403    the rules are closed
                      →  404    the row does not exist yet → POST with that same ID
```

It never lists and never reads, so `list` and `view` can stay closed forever.

## 403 vs 404 is an instrument — but measure it per collection

Measured unauthenticated against a collection whose rules are NULL (superuser-only):

```
GET/PATCH/POST on an existing collection with closed rules → 403 "Only superusers..."
GET on a collection that does not exist                    → 404
```

The made-up-collection call is what gives the `403` its meaning: without that negative
control, "403" and "no such collection" are indistinguishable. With both, the front end
can say on screen **"not saving yet"** instead of pretending it saved.

⚠️ **This is not universal.** In other collections of the same project a denied `update`
answered **404** (PocketBase filters rather than denying), and a denied `list` answers
`200` with `items: []`. Measure the denial signal for the collection you are actually
calling; never inherit it from a sibling. See
[pocketbase-schema-and-rules.md](pocketbase-schema-and-rules.md).

## The product rule that wraps it

Save in two layers: `localStorage` on every keystroke, so the visitor never loses what
they typed, plus the backend on a debounce. And when the backend did **not** save, say
so on screen. A false "saved" is the worst available outcome — the person closes the
page convinced they answered, and nothing ever arrived on the other side. The error
indicator has to be as loud as the success one.
