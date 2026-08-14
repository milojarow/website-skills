# PocketBase: a `number` cannot say "unknown", and a denied rule does not say "denied"

Measured against the DDL PocketBase 0.38 generates. The theme is the same in every
section: a value the schema cannot express, or a refusal the API does not signal,
arrives at the front end looking exactly like ordinary data.

## A `number` field is `NOT NULL DEFAULT 0`

```sql
CREATE TABLE ... `duration_min` NUMERIC DEFAULT 0 NOT NULL
```

There is no schema option that makes it nullable. **"No value captured" and "zero" are
the same byte on disk**, and `0` is what travels over the wire. Two measured
consequences:

- A price that was never captured serves as `0` and the page renders **"$0"** — worse
  than rendering nothing, because it reads as a decision.
- A duration that was never captured serves as `0`, and **a zero-length duration
  collides with nothing**: a scheduler accepts it and stacks bookings on top of each
  other without a complaint.

## Two fixes, chosen by what the field has to hold

**Discrete value set → use `select`, not `number`.** An empty select is storable and
serializes as `''`, so "unknown" becomes expressible *and* distinguishable:

```json
{"duration_min": ""}     // unknown
{"duration_min": "60"}   // sixty minutes
```

The price you pay is that it travels as a **string** and the consumer needs `parseInt`.
Cheap next to not being able to say "I don't know". Bonus: it restricts the column to
valid values, so anything outside the set bounces instead of being accepted silently.

**Has to be numeric (a price) → guard in a hook, not in the type.** The useful
invariant is not "the price cannot be 0", it is **"a VISIBLE record cannot exist
without a price"**. Hook it on `create` **and** on `update` — the update is the one that
matters, because the realistic path is someone flipping an old, priceless record to
visible. No `try`/`catch` around it: there the `throw` *is* the functionality.

The payoff is structural, not cosmetic: the consumer may now assume that everything the
public API returns has `price > 0`, which makes loading the rest of the catalogue with
`visible=false` a safe operation instead of a bet.

## Sibling gotchas from the same schema

- **PocketBase has no per-field defaults.** A "default" is implemented as an
  `onRecordCreate` hook. It runs *before* validation, so it can legitimately fill fields
  marked `required`.
- 🔴 **That hook treats `0` as absent if it tests `if (!value)`** — `0` is falsy in JS, so
  `duration_min: 0` is not rejected, it is silently replaced by the default. A `min:5`
  floor only fires on a truthy out-of-range value. Send `null` or omit the field
  entirely; never send `0` to mean "unset".
- **A denied rule does not return 403.** `list` denied → **`200` with `items: []`**;
  `view` denied → **`404`**. So an **empty** collection looks identical to a **gated**
  one, and "the API returned nothing" is not evidence that the lock exists. To prove it,
  seed one record and check that a superuser sees 1 and an anonymous caller sees 0.
- **The factory `users` collection ships `createRule = ''`** — public open signup. That
  is the product default, not somebody's oversight, but it has to be closed on any site
  that does not offer accounts.
